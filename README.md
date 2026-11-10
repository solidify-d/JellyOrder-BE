# Jelly Order 🍧

**A lightweight, open-source table-order and POS system for small restaurants.**

Jelly Order helps small business owners cut costs and closes the digital gap for older and less tech-savvy customers: anyone can download, self-host, and customize it for free.

> 🏆 **1st Place**, Hanwha Systems BEYOND SW Bootcamp (2nd team project, Sep 2025)

### <img width="50" height="50" alt="Team logo" src="https://github.com/user-attachments/assets/2c140acc-54f4-4b42-b0ba-a3bde1cf5743" /> Team Comet Impact
> *Dust, rock, and ice came together to form a comet, and we are colliding with the world at full speed.*

<table>
  <tr>
    <td align="center">
      <img src="https://avatars.githubusercontent.com/u/98812214?s=120" width="120" height="120" alt="JeaPple"/><br />
      <sub><b><a href="https://github.com/JeaPple" target="_blank" rel="noreferrer">@JeaPple</a></b></sub>
    </td>
    <td align="center">
      <img src="https://avatars.githubusercontent.com/u/194198612?s=120" width="120" height="120" alt="ifunhy"/><br />
      <sub><b><a href="https://github.com/ifunhy" target="_blank" rel="noreferrer">@ifunhy</a></b></sub>
    </td>
    <td align="center">
      <img src="https://avatars.githubusercontent.com/u/149755774?s=120" width="120" height="120" alt="jinnn12"/><br />
      <sub><b><a href="https://github.com/jinnn12" target="_blank" rel="noreferrer">@jinnn12</a></b></sub>
    </td>
    <td align="center">
      <img src="https://avatars.githubusercontent.com/u/171583314?s=120" width="120" height="120" alt="Hyeseong Bak"/><br />
      <sub><b><a href="https://github.com/solidify-d" target="_blank" rel="noreferrer">Hyeseong Bak</a></b></sub>
    </td>
  </tr>
</table>

> This repository is a fork of the team's backend. The frontend lives in a separate team repository.

## 🙋 My role — Hyeseong Bak

I was the **top contributor to the backend (194 of 556 non-merge commits)** and owned the **menu, category, option, and ingredient (inventory) domains** that power the owner console, plus the **real-time menu-status pipeline** to the table tablets.

**Menu catalog and options**
- Designed the menu domain end to end: menus, categories, and a **main option → sub option** model with four selection types (`REQUIRED_SINGLE`, `REQUIRED_MULTIPLE`, `OPTIONAL_SINGLE`, `OPTIONAL_MULTIPLE`) validated whenever a menu is created or edited.
- Built create / read / update / delete APIs for menus and categories, with separate **owner (admin) and customer views** of the same menu.
- Rewrote option syncing on menu edit as an **upsert + soft delete**, so editing a menu never breaks past orders that reference old options.
- Implemented multipart menu updates with **S3 image replacement** (UUID-based object names, upload and delete).

**Ingredient-linked inventory and automatic sold-out**
- Modeled ingredients and a `MenuIngredient` join entity (composite key) so each menu knows exactly which ingredients it depends on.
- Ingredients move through `SUFFICIENT → INSUFFICIENT → EXHAUSTED`. When one becomes `EXHAUSTED`, every menu that uses it is **automatically marked `OUT_OF_STOCK`** and the change is pushed to tablets.
- Restoring is careful: a menu goes back `ON_SALE` only if **no other ingredient it needs is exhausted**, it has not hit its sales limit, and the owner did not mark it sold out manually (`SOLD_OUT_MANUAL`).
- Deleting an ingredient returns the list of menus it affects, so the owner sees the impact before it happens.
- Added **sales limits**: once a menu hits its limit during ordering it flips to `OUT_OF_STOCK` and the change is broadcast.
- Applied **soft delete** to menus, categories, and options, and blocked deletions while a table that ordered the item is still eating (`EATING`).

**Real-time menu status (Redis Pub/Sub → SSE)**
- Menu status changes are published only **after the database transaction commits** (`@TransactionalEventListener(AFTER_COMMIT)`), so tablets never see a state that was rolled back.
- Events go to a per-store Redis channel (`tablet-menu-status:{storeId}`), and every backend instance relays them to its connected tablets over **SSE**.
- Built a **per-store SSE hub** (emitter registry, room broadcast) with a dedicated **heartbeat scheduler** that keeps idle connections alive.

**Foundations and security**
- Created the shared UUID-based base entities (`BaseIdEntity`, `BaseIdAndTimeEntity`) used across the codebase.
- Moved the menu, category, and ingredient APIs from client-sent `storeId` to the **store ID inside the JWT**, so an owner can only touch their own store's data.
- Unified duplicate-resource and invalid-UUID error handling into common exception handlers.
- Set up the team's PR template and wrote the project README and feature test documentation.


## 🔍 How it works

A tour of the backend for engineers: how a single order travels through the system, and the decisions behind it.

### Architecture

```mermaid
flowchart LR
    T[Table tablet<br/>Vue] -- "STOMP orders" --> API
    O[Owner console<br/>Vue] -- "REST + SSE" --> API
    subgraph EKS["Amazon EKS (HPA 1–3 pods)"]
        API[Spring Boot API<br/>JWT · REST · STOMP · SSE]
    end
    API <--> R[(Redis<br/>Pub/Sub · tokens · counters)]
    API --> DB[(MariaDB<br/>RDS / Aurora)]
    API --> S3[(S3 + CloudFront<br/>menu images)]
    API --> KP[[KakaoPay API]]
    API --> NTS[[ODcloud business<br/>registration API]]
    API --> SMTP[[Gmail SMTP]]
```

The backend is one Spring Boot service, but it is built to run as **several pods behind a load balancer**: every real-time message goes through Redis Pub/Sub, so a tablet connected to pod A still hears about an order that pod B processed.

### The life of an order

```mermaid
sequenceDiagram
    autonumber
    participant Tab as Table tablet
    participant Pod as API pod (any)
    participant DB as MariaDB
    participant Redis as Redis Pub/Sub
    participant All as Every API pod
    participant Own as Owner console

    Tab->>Pod: STOMP SEND /publish/{storeId}/orders
    Pod->>DB: Pass 1 — validate every item (sold out? enough left under the sales limit?)
    alt any item fails
        Pod-->>Tab: /topic/{tableId} ack {ok:false, all problems listed at once}
    else all items OK
        Pod->>DB: Pass 2 — save order, add to the table's running bill, bump salesToday
        Pod->>Redis: INCR per-store order number
        Pod-->>Tab: /topic/{tableId} ack {ok:true, order}
        Pod->>Redis: PUBLISH "order"
        Redis-->>All: message
        All-->>Own: /topic/{storeId} new order
        opt a menu just hit its sales limit
            Pod->>Redis: PUBLISH tablet-menu-status:{storeId} (after commit)
            Redis-->>All: message
            All-->>Tab: SSE menu-status → item greys out on every tablet
        end
    end
```

Details that matter:

- **Two-pass validation.** The first pass only checks; it collects *every* sold-out item and every item without enough remaining quantity, then rejects the order with one clear message ("Sold out: A, B / Not enough left: C (2 remaining)"). Nothing is written until the whole cart is valid, so a customer never gets a half-placed order.
- **Running bill per table.** Each order (`UnitOrder`) is attached to the table's open `TotalOrder`, whose totals are updated as items are added; the table switches to `EATING` on its first order.
- **Per-table acknowledgements.** Each tablet subscribes to its own `/topic/{storeTableId}`, so success and failure messages go only to the device that ordered, while the owner console listens on `/topic/{storeId}`.
- **Authenticated sockets.** A `ChannelInterceptor` verifies the JWT on the STOMP `CONNECT` frame and binds the tablet's table ID as the session principal.
- **Order numbers from Redis.** Daily ticket numbers come from an atomic `INCR` on a per-store key, so two pods can never hand out the same number. The key is deleted at closing time and the next `INCR` starts again from 1.

### Inventory and sold-out logic

A menu's availability is decided by three independent reasons, and the code is careful never to let one reason undo another:

```mermaid
stateDiagram-v2
    [*] --> ON_SALE
    ON_SALE --> OUT_OF_STOCK: a required ingredient becomes EXHAUSTED
    ON_SALE --> OUT_OF_STOCK: salesToday reaches salesLimit
    ON_SALE --> SOLD_OUT_MANUAL: owner marks it sold out
    OUT_OF_STOCK --> ON_SALE: all ingredients available and under the sales limit
    SOLD_OUT_MANUAL --> ON_SALE: owner puts it back on sale
```

- Ingredients move through `SUFFICIENT → INSUFFICIENT → EXHAUSTED`, and each menu is linked to the ingredients it needs (`MenuIngredient`).
- When an ingredient comes back, a menu is restored only if **no other** required ingredient is still exhausted, it has not sold out its daily limit, and the owner did not turn it off by hand.
- Every change is published **after the transaction commits** (`@TransactionalEventListener(AFTER_COMMIT)`), so tablets never show a state that was rolled back.

### Real-time channels at a glance

| Channel | Transport | Direction | Used for |
|---|---|---|---|
| `/publish/{storeId}/orders` | STOMP over WebSocket | tablet → server | placing orders |
| `/topic/{storeTableId}` | STOMP | server → one tablet | order accepted / rejected |
| `/topic/{storeId}` | STOMP (fanned out via Redis `order`) | server → owner console | new orders |
| `tablet-menu-status:{storeId}` | Redis Pub/Sub → SSE | server → tablets | sold-out / back-on-sale |
| `request-channel` | Redis Pub/Sub → SSE | tablet → owner console | customer requests (water, cutlery, …) |

We used **WebSocket** where the client must send data and get an answer (ordering), and **SSE** where the server only pushes updates (menu status, requests). SSE connections are kept alive with a dedicated heartbeat scheduler.

### Redis, split by responsibility

Instead of one keyspace for everything, each concern gets its own logical database and its own `RedisTemplate`, which keeps keys from colliding and makes each concern easy to inspect or flush on its own.

| DB | Purpose |
|---|---|
| 0 | default (primary `RedisTemplate`) |
| 1 | store (owner) refresh tokens |
| 2 | table-tablet refresh tokens |
| 3 | password-reset verification codes |
| 4 | per-store order-number counters |
| 7 | SSE Pub/Sub (menu status, requests) |
| 8 | order-table cache (Spring Cache) |

### Two kinds of users, two kinds of tokens

- **Owners** (`STORE`) sign up with a business registration number that is **verified against the National Tax Service data via the ODcloud API** before the account is created, then log in for access and refresh tokens.
- **Table tablets** (`STORE_TABLE`) log in as a specific table and get their own token pair, with the table's UUID embedded in the claims.
- Refresh tokens live in Redis (separate DBs per user type) and are validated on refresh; password reset uses a code sent by email and stored in Redis.
- All owner-side APIs read the store ID **from the token**, not from the request, so one store cannot read or change another store's data.

### Running the business day

- **Open / close:** `POST /open-close/open` starts a business day; `close` is **refused while any payment is still `PENDING` or any order is still in progress**, so the day can't be closed with money in limbo. Closing resets the order-number counters and returns a summary of the day.
- **Tables:** tables are grouped into zones and move through `STANDBY → EATING → PAY_DONE`; a zone (or a menu) cannot be deleted while a table in it is still eating.
- **Payments:** KakaoPay QR (ready → QR scan → approval callback → redirect) or counter payment; each sale is `PENDING`, `COMPLETED`, or `CANCELLED`.
- **Settlement dashboard:** today / last 7 days / this month gross sales, each compared with the previous period, plus daily, weekly, and monthly breakdowns.

### Deployment

| Piece | Details |
|---|---|
| CI/CD | GitHub Actions builds a Docker image, pushes it to **Amazon ECR**, and rolls out the deployment on **Amazon EKS** on every push to `main` |
| Kubernetes | Deployment + Service, **Horizontal Pod Autoscaler** (1–3 replicas), Ingress, in-cluster Redis |
| TLS | cert-manager `ClusterIssuer` + `Certificate` for HTTPS |
| AWS | EKS, ECR, S3, CloudFront, RDS/Aurora, Route 53, Certificate Manager, IAM, VPC |

Manifests live in [`k8s/`](k8s) and the pipeline in [`.github/workflows/`](.github/workflows).

### Design decisions

- **Validate everything, then write.** Two-pass order processing gives customers one complete error message and keeps the database consistent without partial orders.
- **Publish after commit.** Real-time events are bridged from Spring application events to Redis only after the transaction commits, so the UI and the database never disagree.
- **Never let one sold-out reason erase another.** Ingredient exhaustion, sales limits, and manual sold-out are tracked separately, and a menu comes back only when all three allow it.
- **Redis for anything shared across pods.** Order numbers, refresh tokens, and real-time fan-out all live in Redis, which is what makes horizontal scaling on EKS safe.
- **Trust the token, not the request.** Store and table identities come from JWT claims, never from IDs the client sends.

---

## Table of contents

- [My role](#-my-role--hyeseong-bak)
- [How it works](#-how-it-works)
0. [Overview](#0-overview)
1. [Project proposal](#1-project-proposal)
2. [Tech stack](#2-tech-stack)
3. [Analysis and design](#3-analysis-and-design)
4. [Core technologies and implementation](#4-core-technologies-and-implementation)
5. [Feature test results](#5-feature-test-results)

---

## 0. Overview
![img.png](img.png)

<p align="center">
  Jelly Order is an open-source, lightweight table-order and POS system that any small business can adopt and run easily,
</p>
<p align="center">
  built to tackle two problems: the cost burden on small business owners and the digital divide.
</p>

---

## 1. Project proposal

<details>
<summary>Project summary</summary>

Jelly Order is a lightweight, open-source table-order and POS system for small businesses. We defined two core problems: the **cost barrier** of commercial solutions (installation fees, monthly subscriptions, and payment-gateway (PG) fees) and **complex UIs** that are hard for older adults and non-technical users. In response, Jelly Order offers an intuitive, large-font UI/UX and a lock-in-free architecture centered on VAN, cash, and standard card terminals. Owners get a simpler POS experience, customers get a faster and easier way to order, and because the code and documentation are open source, anyone can install and modify it for free. The design assumes, as field reports show, that cost and the digital divide are the main barriers to adoption.

</details>

---

<details>
<summary>Background</summary>

<p align="center">
  <img width="800" alt="Background" src="https://github.com/user-attachments/assets/832ac568-cf29-438f-b520-9b87d1e6ef6e" />
</p>

### 1) Cost barriers and opaque fee structures
Table ordering spread quickly as a way to save on labor, but many reports describe cases where **high payment fees from PG integration (usually 0.8–2.5% or more)** plus **monthly subscriptions, network fees, and hardware costs** ended up costing more than the labor they saved. Some businesses also face contract risks such as **mandatory usage periods and cancellation penalties**. Industry and press reports point out that routing payments through a PG instead of a VAN can add **hundreds of thousands of won per month, or millions of won per year**, and that **PG fees are not capped or standardized** by law. [YTN](https://www.ytn.co.kr/_ln/0102_202410181024214328) · [Seoul Economic Daily](https://www.sedaily.com/NewsView/2DFEDQEVRN) · [Nate News](https://news.nate.com/view/20240108n02405)

> Example: one report describes a shop with ₩20M in monthly sales paying PG 3.3% + service 0.6% ≈ 3.9%, about ₩780K per month in fees. Even when the government subsidized installation, the ongoing fees fell on the small business owner. (Nate News)

Government agencies now publish **guidance on payment-device fees and what to check before signing a contract**, but **optimizing costs after adoption** is still left largely to owners. [Sobaek24](https://www.sbiz.or.kr/smst/bbs/view.do?bbsSn=6840&key=211130603916)

---

### 2) Usability for older and non-technical users
Field visits and surveys repeatedly show that older adults **give up on digital ordering or rely on staff** because of **small text, complex screen flows, and difficulty recovering from payment errors**. A Seoul city survey also found **low device usage and psychological burden**, while **training, large text, and voice guidance** helped people adapt quickly. This suggests that an intuitive, accessible UI (readability, fewer steps, simpler wording) decides whether adoption succeeds. [Bravo My Life](https://bravo.etoday.co.kr/view/atc_view/13547)

---

### 3) The gap between the need for digital transformation and reality
Academic, policy, and press sources agree that smart technologies such as unmanned ordering, smart ordering, and kiosks **improve small-business efficiency**, but they also name **adoption and operating costs** and the **age-based digital divide** as bottlenecks. Jelly Order aims to close this gap with **open-source distribution, low-cost adoption, and an age-friendly UI**. [KCI](https://www.kci.go.kr/kciportal/ci/sereArticleSearch/ciSereArtiView.kci?sereArticleSearchBean.artiId=ART002995983)

</details>

---

<details>
<summary>Goals and objectives</summary>

### 1) Mission
- **Remove the cost barrier:** release as open source so owners can self-host or install locally **without upfront costs or subscriptions**, and minimize fees by **avoiding PG lock-in** and prioritizing **VAN, existing card terminals, and cash**. [Sobaek24](https://www.sbiz.or.kr/smst/bbs/view.do?bbsSn=6840&key=211130603916)
- **Narrow the digital divide:** large fonts, high contrast, plain wording, and minimal steps so that **older and non-technical users** can order and pay on their own. [Bravo My Life](https://bravo.etoday.co.kr/view/atc_view/13547)

---

### 2) Goals by stakeholder
- **Owners (stores)**
  - A **lightweight POS console** that is easy to install, learn, and run, with minimal-click flows for menus, inventory, sold-out items, and seats/tables.
- **Customers**
  - A senior-friendly mode (larger text and buttons, voice hints, guided error recovery) to reduce abandoned orders.
  - **Ordering in 5 clicks or fewer** (category → menu → options → quantity → pay) to keep cognitive load low. [Bravo My Life](https://bravo.etoday.co.kr/view/atc_view/13547)

---

### 3) Target KPIs (initial proposal)
- **Lower adoption cost:** **80%+ less upfront CAPEX** than commercial systems (open source on off-the-shelf tablets; owners pay only for hardware). Commercial systems are widely reported to accumulate costs through **subscriptions + PG fees**. [YTN](https://www.ytn.co.kr/_ln/0102_202410181024214328) · [Seoul Economic Daily](https://www.sedaily.com/NewsView/2DFEDQEVRN)
- **Lower fees:** aim for **zero to minimal extra PG fees** by preferring **VAN / direct settlement** (depends on each store's contracts). [Seoul Economic Daily](https://www.sedaily.com/NewsView/2DFEDQEVRN)
- **Order success rate:** **30% fewer abandoned orders** (drop-off before payment) in stores using the senior-friendly UI. Surveys show high abandonment and psychological burden among older users. [Bravo My Life](https://bravo.etoday.co.kr/view/atc_view/13547)
- **Learning curve:** owners can run the basics (register menus, mark sold-out items, check receipts) **within 30 minutes of onboarding**.

---

### 4) Core feature scope
- **Owner console:** menus, options, and prices; tables and seats; inventory and sold-out status linked to ingredients; live orders; settlement reports (including a fee simulator).
- **Customer UI:** **large fonts and simple layouts**, high-contrast theme, **error-recovery guidance**, and as few payment steps as possible. [Bravo My Life](https://bravo.etoday.co.kr/view/atc_view/13547)
- **Payments:** **VAN / existing card terminals / cash** by default, with **optional PG integration** (with fee warnings and comparisons). [Seoul Economic Daily](https://www.sedaily.com/NewsView/2DFEDQEVRN)
- **Deployment and operations:** open-source release, **Docker and self-hosting templates**, install automation scripts, and an operations guide / security checklist.

</details>

---

<details>
<summary>Problem statement (with sources)</summary>

- **Fees and operating costs** often cancel out labor savings, and there is **no cap on PG fees** plus contract risk → the cost barrier must go. [YTN](https://www.ytn.co.kr/_ln/0102_202410181024214328) · [Seoul Economic Daily](https://www.sedaily.com/NewsView/2DFEDQEVRN) · [Nate News](https://news.nate.com/view/20240108n02405)
- **Usability limits for older and non-technical users** block adoption, while **large text, voice, and fewer steps** improve it → accessibility-first design. [Bravo My Life](https://bravo.etoday.co.kr/view/atc_view/13547)
- Research recognizes the **value of smart technology** but names **adoption and investment costs** as the bottleneck → an **open-source, low-cost, self-installed** model has public value. [KCI](https://www.kci.go.kr/kciportal/ci/sereArticleSearch/ciSereArtiView.kci?sereArticleSearchBean.artiId=ART002995983)

</details>

<br>

## 2. Tech stack

### Backend
![Java 17](https://img.shields.io/badge/Java%2017-007396?style=for-the-badge&logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot%203.5-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)
![Spring Security](https://img.shields.io/badge/Spring%20Security-6DB33F?style=for-the-badge&logo=springsecurity&logoColor=white)
![Spring Data JPA](https://img.shields.io/badge/Spring%20Data%20JPA-59666C?style=for-the-badge&logo=hibernate&logoColor=white)
![Hibernate](https://img.shields.io/badge/Hibernate-59666C?style=for-the-badge&logo=hibernate&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)
![WebSocket](https://img.shields.io/badge/WebSocket-000000?style=for-the-badge&logo=websocket&logoColor=white)
![MariaDB](https://img.shields.io/badge/MariaDB-003545?style=for-the-badge&logo=mariadb&logoColor=white)
![SSE](https://img.shields.io/badge/SSE%20(Server--Sent%20Events)-000000?style=for-the-badge&logo=fastapi&logoColor=white)
![JWT](https://img.shields.io/badge/JWT-black?style=for-the-badge&logo=jsonwebtokens&logoColor=white)

### Frontend
![Vue.js](https://img.shields.io/badge/Vue.js%203-4FC08D?style=for-the-badge&logo=vue.js&logoColor=white)
![Vuetify](https://img.shields.io/badge/Vuetify-1867C0?style=for-the-badge&logo=vuetify&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black)
![HTML5](https://img.shields.io/badge/HTML5-E34F26?style=for-the-badge&logo=html5&logoColor=white)
![CSS3](https://img.shields.io/badge/CSS3-1572B6?style=for-the-badge&logo=css3&logoColor=white)

### ☁️ Infra & Cloud
![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![AWS S3](https://img.shields.io/badge/AWS%20S3-569A31?style=for-the-badge&logo=amazons3&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazonaws&logoColor=white)
![CloudFront](https://img.shields.io/badge/CloudFront-945CF1?style=for-the-badge&logo=CloudFront&logoColor=white)
![Aurora-and-RDS](https://img.shields.io/badge/Aurora--and--RDS-4561E3?style=for-the-badge&logo=Aurora-and-RDS&logoColor=white)
![Route53](https://img.shields.io/badge/Route53-945CF2?style=for-the-badge&logo=Route53&logoColor=white)
![Certificate Manager](https://img.shields.io/badge/Certificate--Manager-F54749?style=for-the-badge&logo=Certificate--Manager&logoColor=white)
![IAM](https://img.shields.io/badge/IAM-F54648?style=for-the-badge&logo=IAM&logoColor=white)
![VPC](https://img.shields.io/badge/VPC-5B31B6?style=for-the-badge&logo=VPC&logoColor=white)

### External API & Integration
![RestTemplate](https://img.shields.io/badge/RestTemplate-6DB33F?style=for-the-badge&logo=spring&logoColor=white)
![KakaoPay](https://img.shields.io/badge/KakaoPay-FFCD00?style=for-the-badge&logo=kakaotalk&logoColor=black)
![ODcloud](https://img.shields.io/badge/ODcloud_API-005BAC?style=for-the-badge&logo=databricks&logoColor=white)
![Gmail SMTP](https://img.shields.io/badge/Gmail%20SMTP-EA4335?style=for-the-badge&logo=gmail&logoColor=white)

### Test & Docs
![Swagger UI](https://img.shields.io/badge/Swagger-85EA2D?style=for-the-badge&logo=swagger&logoColor=black)
![Postman](https://img.shields.io/badge/Postman-FF6C37?style=for-the-badge&logo=postman&logoColor=white)
![JMeter](https://img.shields.io/badge/Apache%20JMeter-D22128?style=for-the-badge&logo=apachejmeter&logoColor=white)

### Tools & Collaboration
![Git](https://img.shields.io/badge/Git-F05032?style=for-the-badge&logo=git&logoColor=white)
![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)
![Notion](https://img.shields.io/badge/Notion-000000?style=for-the-badge&logo=notion&logoColor=white)
![Discord](https://img.shields.io/badge/Discord-5865F2?style=for-the-badge&logo=discord&logoColor=white)
![Figma](https://img.shields.io/badge/Figma-F24E1E?style=for-the-badge&logo=figma&logoColor=white)

<br>

## 3. Analysis and design

> The linked design documents are written in Korean.

### Requirements specification - [View](https://docs.google.com/spreadsheets/d/1lV6OYcvnEEBzQFvDyBVyoR1MRkYB52oj-mvlcYLXNRM/edit?gid=1096538882#gid=1096538882)
<details>
  <summary><b>Requirements specification</b></summary>
<img width="1714" height="1863" alt="Requirements specification" src="https://github.com/user-attachments/assets/c850eac9-e2e1-4006-bd64-32953ab91beb" />
</details>

### Screen design - [View in Figma](https://www.figma.com/design/WuBdoUWSz5n3gdSHdMN2qo/be16-4th-team?node-id=0-1&t=5D9WbMwwOTOxQaVj-1)
<details>
  <summary><b>Owner console</b></summary>
<img width="800" alt="Owner console design" src="https://github.com/user-attachments/assets/50a33d51-f4e6-4848-b90b-70fbc4c9f666" />
</details>
<details>
  <summary><b>Table-order tablet</b></summary>
<img width="800" alt="Table-order design" src="https://github.com/user-attachments/assets/c0bb2ccf-a485-407f-874e-c51e33b41423" />
</details>

### ERD - [View in ERDCloud](https://www.erdcloud.com/d/fAJgKBWBde3CPAkgW)
<details>
  <summary><b>ERD</b></summary>
  <img width="3380" height="1629" alt="Jelly Order ERD" src="https://github.com/user-attachments/assets/6230dd70-a6f5-42a7-8c24-b7d5026507a5" />
</details>

### WBS - [View](https://docs.google.com/spreadsheets/d/1lV6OYcvnEEBzQFvDyBVyoR1MRkYB52oj-mvlcYLXNRM/edit?gid=0#gid=0)
<details>
  <summary><b>WBS</b></summary>
<img width="1420" height="648" alt="WBS" src="https://github.com/user-attachments/assets/bcfe1f43-44c1-418a-a2a6-12862316e928" />
</details>

### API specification - [View](https://docs.google.com/spreadsheets/d/1lV6OYcvnEEBzQFvDyBVyoR1MRkYB52oj-mvlcYLXNRM/edit?gid=394720298#gid=394720298)
<details>
  <summary><b>API specification</b></summary>
<img width="1665" height="1389" alt="API specification" src="https://github.com/user-attachments/assets/b51c1360-2863-459b-a3d7-f35f32df07f0" />
</details>
<br>

---

## 4. Core technologies and implementation

<table>
  <thead>
    <tr>
      <th>Technology</th>
      <th>What we built</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><b>JWT authentication and authorization with Spring Security</b></td>
      <td>
        We configured a JWT filter in Spring Security to define authentication and authorization rules, and issue an <code>access token</code> and a <code>refresh token</code> when a store (STORE) or a table device (STORE_TABLE) logs in. Because JWT is stateless, refresh tokens are stored in Redis for fast lookup and revocation. The token payload is assembled per login type, and a dedicated <code>@Bean</code> exposes values such as the UUID-based table ID across the service layer. Auto-login separates <code>sessionStorage</code> and <code>localStorage</code> by use case. Owner sign-up also verifies the business registration number through the Korean public data portal (ODcloud) API.
      </td>
    </tr>
    <tr>
      <td><b>Real-time messaging with Redis Pub/Sub</b></td>
      <td>
        When an owner updates or deletes an ingredient, or a customer tablet sends a request or an order, the service layer computes the state change immediately and publishes an event to a <code>Redis Pub/Sub</code> channel. Every subscriber (table devices and dashboards) receives it at the same time, keeping devices in sync even when the backend runs on multiple instances.
      </td>
    </tr>
    <tr>
      <td><b>State sync with Server-Sent Events (SSE)</b></td>
      <td>
        Menu and request changes on the backend are streamed to devices over SSE, so the Vue frontends (table tablets and the owner dashboard) always show the latest state without polling.
      </td>
    </tr>
    <tr>
      <td><b>Real-time ordering with WebSocket (STOMP)</b></td>
      <td>
        Order events are sent in real time over <code>STOMP WebSocket</code>. The store dashboard subscribes to a <code>storeId</code>-based topic and each table device subscribes to a <code>tableId</code>-based queue, separating their roles. The Vue clients apply new orders automatically, so the order screen is always up to date.
      </td>
    </tr>
    <tr>
      <td><b>KakaoPay QR payment integration</b></td>
      <td>
        We integrated the KakaoPay payment API to support QR payments, with a flow of <code>payment ready → QR code generation/scan → approval callback → redirect to the result page</code>.
      </td>
    </tr>
  </tbody>
</table>

---

## 5. Feature test results

> We verified the core features of the owner console and the table-order tablet. Each result is in a collapsible section with screenshots.

### 📌 Owner console
<details>
  <summary><b>Sign-up and login</b></summary>
  
  - **Owner sign-up**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/43680d12-01ba-4f90-8d4f-7e6965704932" alt="screenshot" width="720">
    </p>
  
  - **Owner login**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/b431146e-4368-4559-8c06-44291ab59692" alt="screenshot" width="720">
    </p>
  
  - **Owner auto-login**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/3a68f0f8-9e06-4513-abcc-a30a9631e2a7" alt="screenshot" width="720">
    </p>

  - **Find username**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/99220b56-a342-4283-b39d-7465954cd054" alt="screenshot" width="720">
    </p>

  - **Reset password**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/f0d92778-05ff-4068-afbc-3727e21502e2" alt="screenshot" width="720">
    </p>
</details>

<details>
  <summary><b>Table status</b></summary>

  - **View ordering tables by zone**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/0db521d3-c686-4b13-9538-68711bf03a52" alt="screenshot" width="720">
    </p>

  - **Table status change after QR payment**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/eee7ec00-0810-4648-bc99-dcb33b79cb1a" alt="screenshot" width="720">
    </p>

  - **Counter payment**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/674e6a6a-7e93-455e-b80c-343aef914e7e" alt="screenshot" width="720">
    </p>

  - **Clear a table**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/d5e6c6f5-a588-4372-adf4-bbb2d0876af8" alt="screenshot" width="720">
    </p>

  - **Edit an order**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/14d64ee4-fe28-418b-9345-a51e7153f3e0" alt="screenshot" width="720">
    </p>
</details>

<details>
  <summary><b>Order status</b></summary>

  - **Order queue (Received tab)**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/f06d9dee-5260-4e89-9a31-bec36bb6c273" width="720">
    </p>
    
  - **Order queue (Completed tab)**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/92b9335d-69ac-4f8f-b4a5-461c91bd1202" width="720">
    </p>

  - **Order queue (Cancelled tab)**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/0c85ad12-07e4-4f37-9824-1bd5233de9cd" width="720">
    </p>
    
  - **Real-time order check**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/3f330879-6b42-4359-b4f9-647f251f6872" width="720">
    </p>

  - **Mark as cooked**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/5d7d0036-7118-4266-a042-a58685159562" width="720">
    </p>

  - **Cancel cooking**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/ac051629-bcb0-4246-b293-cd525c17eb9d" width="720">
    </p>
</details>

<details>
  <summary><b>Menu management</b></summary>

  - **Menu page**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/4a7618f2-1817-49aa-8d80-322b85c4300e" width="720">
    </p>

  - **Create menu (add options)**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/4040d775-de6d-4cac-8dbf-2165de76ba2c" width="720">
    </p>

  - **Create menu (link ingredients)**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/661784de-255c-4c98-b093-6dea3d34c9f7" width="720">
    </p>

  - **Edit menu (auto sold-out when ingredients run out)**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/100132f8-5a50-4664-a66c-6ea66690a2da" width="720">
    </p>

  - **Delete menu**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/56d18fdf-eebc-4cdd-bc8f-1ab9e50b1d1d" width="720">
    </p>

  - **Search menu**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/c65e3f9e-0b37-4385-81ac-196dec89f4ad" width="720">
    </p>
    
  - **Browse menu by category**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/04735dfc-0fa5-4d4f-aa4a-63b1a8705cac" width="720">
    </p>
        
  - **Create category (no duplicates)**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/5909c887-f8b1-47e5-a632-da3fc2b27148" width="720">
    </p>
</details>

<details>
  <summary><b>Ingredient management</b></summary>
  
  - **Ingredient management page**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/8b0c4cb0-c68d-4bb2-993e-6dc62ec5e91b" width="720">
    </p>

  - **Search ingredients**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/d077f4a8-feed-40b4-a6ce-bfc5eb4e4556" width="720">
    </p>
    
  - **Add ingredient**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/1418257f-615a-472d-8541-067a7336605d" width="720">
    </p>

  - **Edit ingredient**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/8bd52f97-1c99-43a8-8327-15b7f78ad1cf" width="720">
    </p>

  - **Delete ingredient**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/16e3bcbf-af83-4015-a762-4d284ee905dc" width="720">
    </p>
  

</details>

---

### 📌 Table-order tablet
<details>
  <summary><b>Main features</b></summary>

  - **Idle screen**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/afd3565a-5b1e-44b3-9c58-1b97094320f1" width="720">
    </p>

  - **Choose a category**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/07a70800-979e-41e6-b517-fd40b81b0f63" width="720">
    </p>
    
  - **Switch layout**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/a7f501bb-f8f8-42cc-b21e-fff3e00abf24" width="720">
    </p>

  - **Send a request to staff**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/0540c3af-c6b0-45b3-9976-1980a0e20637" width="720">
    </p>

  - **Menu detail + add to cart**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/40bb14c4-01f6-469f-af61-9758ec436699" width="720">
    </p>

  - **Select options by type**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/1f3f30d9-a050-4c56-9066-249a8e628a8e" width="720">
    </p>
    
  - **Change cart quantity**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/df401c5c-590f-440e-9ccb-57ba28d2489b" width="720">
    </p>
    
  - **View sold-out menu**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/592b79e4-dccf-47b3-9fe1-8e90bca6a213" alt="screenshot" width="720">
    </p>

  - **Try to order a sold-out menu**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/992cf2b5-7e2b-4895-be24-741a5c7f7fb6" alt="screenshot" width="720">
    </p>
    
  - **Order a limited-quantity menu**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/d22f5bb0-0362-463f-9123-dfd217a83598" alt="screenshot" width="720">
    </p>
        
  - **Place an order**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/537c4ed4-81d5-4167-94be-30de2f9ec02c" alt="screenshot" width="720">
    </p>
    
  - **View order history**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/d6ccd45f-038d-44a6-b252-cb9482ea43e4" alt="screenshot" width="720">
    </p>

  - **QR payment**  
    <p align="center">
      <img src="https://github.com/user-attachments/assets/5c60bfee-ff61-4d4b-9a89-5f57f87d24e2" alt="screenshot" width="720">
      <img src="https://github.com/user-attachments/assets/f114d151-7382-4a0e-8bea-089697a3ee11" alt="screenshot" width="720">
    </p>
</details>

