# showcase-the-spork-clone

> A distributed restaurant reservation and discovery platform engineered as an event-driven Nx monorepo, decoupling API gateway edge routing from asynchronous domain microservices via RabbitMQ and polyglot persistence.

**Inception (First Commit):** `2025-01-22` • **Batch / Horizon:** `Batch 2024–2025` • **Architecture Domain:** `Full-Stack Web Architecture / Domain Clone`

---

## 📌 Architecture & Overview

**The-Spork** is a full-stack distributed ecosystem inspired by restaurant reservation systems (such as *TheFork*), built with an event-driven architecture (EDA) designed around an enterprise-grade **Nx workspace**. The platform separates edge client ingress, user identity, catalog discovery, table bookings, and rating aggregation into domain-bounded contexts coordinated asynchronously via RabbitMQ topic exchanges and synchronously via tRPC/REST.

* **Context:** Architectural proof-of-concept and scalable reference implementation demonstrating multi-tenant restaurant discovery, concurrent reservation orchestration, and eventual consistency across relational and document stores.
* **Key Patterns:**
  * **Nx Monorepo Modularization:** Distinct separation between application entry points (`apps/`) and reusable cross-cutting libraries (`libs/`) enforcing clean dependency boundaries.
  * **Event-Driven Architecture (EDA):** Asynchronous pub/sub event bus coordinating state synchronization between services without synchronous tight coupling.
  * **Polyglot Persistence:** Dual-engine storage strategy selecting PostgreSQL for transactional, relational identity datasets and MongoDB for hierarchical restaurant menus, reviews, and booking records.
  * **Edge Gateway Facade:** API Gateway abstracting internal service topology, executing JWT authentication, payload validation, and client routing.
  * **Eventual Consistency:** Asynchronous rating recalculation and reservation seat adjustment driven by domain event consumption.

### Workspace Architecture Map

```
The-Spork/
├── apps/
│   ├── api-gateway/            # NestJS API Gateway: client ingress, auth routing, request validation
│   └── api-gateway-e2e/        # End-to-end integration test suite powered by Jest & Axios
├── libs/
│   ├── shared/                 # Domain DTOs (Reservation, Restaurant, Review, User, Hours)
│   ├── config/                 # Centralized configuration services (@nestjs/config wrappers)
│   ├── messaging/              # Broker transport interfaces & RabbitMQ publisher/consumer abstractions
│   ├── mongo/                  # MongoDB database connection providers & schema abstractions
│   ├── sql/                    # PostgreSQL database connection providers & TypeORM entities
│   └── validation/             # Reusable validation pipes & constraint decorators
├── docker/
│   ├── docker-compose.yaml     # Container definitions (PostgreSQL, pgAdmin, MongoDB)
│   └── db.env                  # Database runtime environment credentials
└── docs/                       # High-level system design, sequence diagrams & service analysis
```

### Event-Driven Topology

```mermaid
flowchart TD
    Client(["HTTP / Web Clients"]) -->|REST / JSON| Gateway["API Gateway (:3000)"]
    
    Gateway -.->|Sync Dispatch / tRPC| UserService["User Service (PostgreSQL)"]
    Gateway -.->|Sync Dispatch / tRPC| RestService["Restaurant Service (MongoDB)"]
    Gateway -.->|Sync Dispatch / tRPC| ResService["Reservation Service (MongoDB)"]
    Gateway -.->|Sync Dispatch / tRPC| RevService["Review Service (MongoDB)"]

    subgraph MessageBroker ["RabbitMQ Message Broker"]
        ExUser["user_exchange"]
        ExRest["restaurant_exchange"]
        ExRes["reservation_exchange"]
        ExRev["review_exchange"]
    end

    UserService -->|Publishes: user.registered, user.deleted| ExUser
    RestService -->|Publishes: restaurant.created, restaurant.deleted| ExRest
    ResService -->|Publishes: reservation.created, reservation.updated| ExRes
    RevService -->|Publishes: review.created, review.deleted| ExRev

    ExUser -->|Consumes| ResService
    ExUser -->|Consumes| RevService
    ExRest -->|Consumes| ResService
    ExRev -->|Consumes: Recalculate Rating| RestService
    ExRes -->|Consumes: Adjust Capacity| RestService
```

---

## 🛠 Tech Stack

* **Workspace & Tooling:** Nx 20.4, pnpm 9, TypeScript 5.7, Webpack 5, SWC compiler
* **Runtime & Framework:** Node.js (>= 18), NestJS 10.0, RxJS 7.8
* **Data & Storage:** 
  * PostgreSQL 16 (Relational storage for user authentication and identities)
  * MongoDB 7 (Document store for flexible menus, reservations, and reviews)
  * pgAdmin 4 (Database management UI)
* **Messaging & Transport:** RabbitMQ (AMQP topic exchanges & durable task queues), tRPC, REST
* **Validation & Serialization:** `class-validator` 0.14, `class-transformer` 0.5
* **Containerization & CI:** Docker Compose v2, GitHub Actions (`.github/workflows/ci.yml`)
* **Testing:** Jest 29.7, `ts-jest`, Supertest, Axios

---

## ✨ Features & Endpoints

* **API Gateway Authentication:** Edge validation and ingress parsing for user registration and role verification.
* **Restaurant Directory:** Hierarchical catalogue with geolocation coordinates, dynamic operating schedules, and multi-cuisine filtering.
* **Reservation Lifecycle:** Slot booking with automated status management (`PENDING`, `APPROVED`, `REJECTED`).
* **Review Aggregation:** Asynchronous rating updates recalculated upon review publication and deletion.
* **Shared Contract Library:** Strongly typed Data Transfer Objects (`@the-spork/shared`) sharing schemas across gateway and microservices.

### Key API Routes & Messaging Interfaces

| Protocol / Layer | Target / Endpoint | Description |
|---|---|---|
| `POST` | `/api/auth/register` | User account registration with DTO payload validation |
| `POST` | `/api/auth/login` | User authentication returning scoped JWT bearer token |
| `GET` | `/api/users/me` | Fetch active user session profile |
| `GET` | `/api/restaurants` | List restaurants with city and cuisine filtering |
| `POST` | `/api/restaurants` | Provision new restaurant entity (Admin authorized) |
| `POST` | `/api/reservations` | Create reservation slot booking |
| `GET` | `/api/restaurants/:id/reviews` | Retrieve paginated reviews for a specific restaurant |
| `AMQP TOPIC` | `user.registered` | Broadcast on `user_exchange` when an account is created |
| `AMQP TOPIC` | `restaurant.created` | Broadcast on `restaurant_exchange` to prime booking engine |
| `AMQP TOPIC` | `reservation.created` | Broadcast on `reservation_exchange` to update slot availability |
| `AMQP TOPIC` | `review.created` | Broadcast on `review_exchange` to trigger rating recalculation |

---

## 🚀 Getting Started

### Prerequisites

* **Node.js** `>= 18.16.9` (Node.js 20 LTS recommended)
* **pnpm** `>= 9.0.0`
* **Docker** & **Docker Compose**

### Installation

```bash
# Clone repository
git clone https://github.com/fabogit/showcase-the-spork-clone.git
cd showcase-the-spork-clone/the-spork

# Install dependencies using pnpm
pnpm install
```

### Environment Configuration

Configure database settings in `docker/db.env` or create an application `.env`:

```env
# Application Server
PORT=3000
NODE_ENV=development

# PostgreSQL Configuration
POSTGRES_HOST=localhost
POSTGRES_PORT=5433
POSTGRES_USER=postgres
POSTGRES_PASSWORD=password
POSTGRES_DB=the_spork

# MongoDB Configuration
MONGODB_URI="mongodb://localhost:27018/the_spork"

# RabbitMQ Configuration
RABBITMQ_URI="amqp://guest:guest@localhost:5672"
```

### Running External Dependencies

Start database containers (PostgreSQL, pgAdmin, and MongoDB):

```bash
# From repository root
docker compose -f docker/docker-compose.yaml up -d
```

* **pgAdmin Web UI:** Accessible at `http://localhost:5050` (Login: `admin@example.com` / `password`)
* **PostgreSQL:** Running on `localhost:5433`
* **MongoDB:** Running on `localhost:27018`

### Running the Application

Execute tasks using the **Nx CLI**:

```bash
# Start API Gateway in development mode
npx nx serve api-gateway

# Build API Gateway bundle
npx nx build api-gateway --configuration=production

# Show dependency graph
npx nx graph
```

### Running Tests & Linting

```bash
# Run linting across affected projects
npx nx affected -t lint

# Run unit tests across affected libraries
npx nx affected -t test

# Run End-to-End tests
npx nx run api-gateway-e2e:e2e
```

---

## 📄 License & Author

Developed by [@fabogit](https://github.com/fabogit).
Licensed under the MIT License.
