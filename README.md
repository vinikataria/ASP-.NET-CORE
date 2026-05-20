# 🚀 ASP.NET Core Microservices Reference Project

A production-grade reference implementation of a microservices architecture using ASP.NET Core 8, demonstrating advanced patterns including Saga orchestration, gRPC, YARP, and MassTransit.

---

## 📐 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        React Frontend                           │
│          (Login · Order Dashboard · Real-time Status)           │
└────────────────────────────┬────────────────────────────────────┘
                             │ REST (HTTP/JSON)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  API Gateway  (YARP)  :5000                     │
│  • JWT validation (central)                                     │
│  • Routes: /api/auth → Identity  /api/orders → Order           │
│            /api/payments → Payment                              │
└──────────┬──────────────────┬──────────────────┬───────────────┘
           │ REST             │ REST             │ REST
           ▼                  ▼                  ▼
 ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
 │  Identity API    │ │   Order API      │ │  Payment API     │
 │  :5100 / :5101   │ │   :5200          │ │  :5300           │
 │                  │ │                  │ │                  │
 │  • Registration  │ │  • Create Order  │ │  • Process Cmd   │
 │  • JWT issuance  │ │  • Saga SM       │ │  • Publish result│
 │  • gRPC server   │◄│  • gRPC client   │ │                  │
 │  • Identity DB   │ │  • Order DB      │ │  • Payment DB    │
 └──────────────────┘ └────────┬─────────┘ └────────┬─────────┘
                               │                     │
                               └──────┬──────────────┘
                                      │  AMQP (MassTransit)
                                      ▼
                             ┌─────────────────┐
                             │   RabbitMQ      │
                             │  :5672 / :15672 │
                             └─────────────────┘
```

---

## 🗂 Project Structure

```
/
├── src/
│   ├── Shared/
│   │   └── Contracts/                  # MassTransit message contracts
│   │       └── Messages/OrderMessages.cs
│   └── Services/
│       ├── Identity/Identity.API/      # Auth + JWT + gRPC server
│       ├── Order/Order.API/            # Orders + Saga state machine
│       ├── Payment/Payment.API/        # Payment processing consumer
│       └── Gateway/ApiGateway/         # YARP reverse proxy
├── frontend/                           # React 18 + Vite SPA
├── docker-compose.yml
└── README.md
```

---

## 🔄 Saga State Machine — Order Lifecycle

The **Choreography is replaced by Orchestration**: the `OrderStateMachine` (MassTransit) coordinates the entire order→payment flow.

### Happy Path

```
sequenceDiagram
    actor User
    participant GW   as API Gateway (YARP)
    participant ORDS  as Order Service
    participant RMQ  as RabbitMQ
    participant SAGA as Order Saga (MassTransit SM)
    participant PAY  as Payment Service
    participant ID   as Identity Service (gRPC)

    User->>GW: POST /api/orders  [JWT]
    GW->>ORDS: Forward (JWT validated at gateway)
    ORDS->>ID: gRPC ValidateUser(userId)
    ID-->>ORDS: { isValid: true }
    ORDS->>ORDS: INSERT Order (status=Submitted)
    ORDS->>RMQ: Publish OrderSubmitted

    RMQ->>SAGA: Consume OrderSubmitted
    SAGA->>SAGA: Transition → WaitingForPayment
    SAGA->>RMQ: Send ProcessPayment (command)

    RMQ->>PAY: Consume ProcessPayment
    PAY->>PAY: INSERT Payment (status=Pending)
    PAY->>PAY: Call payment gateway ✅
    PAY->>PAY: UPDATE Payment (status=Succeeded)
    PAY->>RMQ: Publish PaymentCompleted

    RMQ->>SAGA: Consume PaymentCompleted
    SAGA->>SAGA: Transition → Completed (Finalized)
    SAGA->>RMQ: Publish OrderConfirmed

    RMQ->>ORDS: Consume OrderConfirmed
    ORDS->>ORDS: UPDATE Order (status=Confirmed)
    User->>GW: GET /api/orders/{id}  → status: "Confirmed"
```

### ❌ Failure Path — Compensating Transaction

```
sequenceDiagram
    participant PAY  as Payment Service
    participant RMQ  as RabbitMQ
    participant SAGA as Order Saga
    participant ORDS  as Order Service

    PAY->>PAY: Call payment gateway ❌ (declined / limit exceeded)
    PAY->>PAY: UPDATE Payment (status=Failed)
    PAY->>RMQ: Publish PaymentFailed(reason="Payment declined")

    RMQ->>SAGA: Consume PaymentFailed
    SAGA->>SAGA: Store FailureReason
    SAGA->>SAGA: Transition → Failed

    Note over SAGA: COMPENSATING TRANSACTION
    SAGA->>RMQ: Publish OrderCancelled(reason)

    RMQ->>ORDS: Consume OrderCancelled (OrderCancelledConsumer)
    ORDS->>ORDS: UPDATE Order (status=Cancelled, cancellationReason=reason)

    Note over ORDS: Order is now visibly Cancelled in the dashboard
```

**Key insight**: No 2-phase commit is used. Instead, each service owns its own database and the Saga publishes compensating events to undo partial work if any step fails.

---

## 🛡 Security Model

| Layer | Mechanism |
|---|---|
| External → Gateway | JWT Bearer (`Authorization: Bearer <token>`) |
| Gateway → Services | JWT forwarded in headers; services re-validate |
| Service → Identity | gRPC over internal Docker network (mTLS in production) |
| Token storage | In-memory (React context) — see BFF note below |

> **BFF Pattern Note**: For production, implement a Backend-for-Frontend server that issues tokens into `HttpOnly`, `SameSite=Strict` cookies so JavaScript never touches the raw JWT. The current in-memory approach is appropriate for reference/demo purposes.

---

## 🚦 Communication Patterns

| Pattern | Used For | Technology |
|---|---|---|
| REST (synchronous) | Client ↔ Gateway, Gateway ↔ Services | ASP.NET Core + YARP |
| gRPC (synchronous) | Order Service → Identity Service (ValidateUser) | Grpc.AspNetCore / Grpc.Net.Client |
| Async Messaging | Saga coordination, Payment events | MassTransit 8 + RabbitMQ |
| Compensating Tx | Payment failure rollback | MassTransit Saga + OrderCancelled event |

---

## 🏃 Quick Start

### Prerequisites
- Docker Desktop ≥ 4.x
- .NET 8 SDK (for local development)
- Node.js 20+ (for frontend development)

### Run with Docker Compose

```bash
docker-compose up --build
```

| Service | URL |
|---|---|
| React Frontend | http://localhost:3000 |
| API Gateway | http://localhost:5000 |
| Identity Swagger | http://localhost:5100/swagger |
| Order Swagger | http://localhost:5200/swagger |
| Payment Swagger | http://localhost:5300/swagger |
| RabbitMQ UI | http://localhost:15672 (guest/guest) |

### Local Development (without Docker)

```bash
# 1. Start infrastructure only
docker-compose up rabbitmq identity-db order-db payment-db -d

# 2. Run each service (separate terminals)
cd src/Services/Identity/Identity.API  && dotnet run
cd src/Services/Order/Order.API        && dotnet run
cd src/Services/Payment/Payment.API    && dotnet run
cd src/Services/Gateway/ApiGateway     && dotnet run

# 3. Run the frontend
cd frontend && npm install && npm run dev
# → http://localhost:5173
```

### EF Core Migrations (first time)

```bash
# Identity
dotnet ef migrations add InitialCreate \
  --project src/Services/Identity/Identity.API \
  --startup-project src/Services/Identity/Identity.API

# Order
dotnet ef migrations add InitialCreate \
  --project src/Services/Order/Order.API \
  --startup-project src/Services/Order/Order.API

# Payment
dotnet ef migrations add InitialCreate \
  --project src/Services/Payment/Payment.API \
  --startup-project src/Services/Payment/Payment.API
```

> All services auto-migrate on startup in Docker (`db.Database.MigrateAsync()`).

---

## 🧪 Testing the Saga Failure Path

The Payment Service simulates an 85% success rate. To **force a failure**, place an order with a total exceeding **$10,000** — the consumer will decline it and the Saga will trigger the compensating transaction.

```bash
# Register
curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"firstName":"Jane","lastName":"Doe","email":"jane@demo.com","password":"Password1!"}'

# Login → get token
TOKEN=$(curl -s -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"jane@demo.com","password":"Password1!"}' | jq -r .accessToken)

# Place a failing order (amount > $10,000)
curl -X POST http://localhost:5000/api/orders \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"items":[{"productId":"'$(uuidgen)'","productName":"Luxury Item","quantity":1,"unitPrice":15000}]}'
```

Watch the order status transition to **Cancelled** within seconds.

---

## 📊 Cross-Cutting Concerns

| Concern | Implementation |
|---|---|
| Structured Logging | Serilog → Console (JSON-friendly; swap sink for Seq/ELK) |
| Health Checks | `/health` on every service (DB + RabbitMQ probes) |
| Error Handling | `ErrorHandlingMiddleware` on every service |
| DI / IoC | ASP.NET Core built-in DI throughout |
| Database isolation | PostgreSQL database-per-service |
| Saga persistence | MassTransit EF Core saga repository (OrderSagaStates table) |

---

## ⚙️ Key Configuration

All secrets are passed via **environment variables** in docker-compose.yml.  
For production, use Docker Secrets, Azure Key Vault, or HashiCorp Vault.

The single shared JWT secret key must be **identical** across Identity, Order, Payment, and Gateway services so each can independently validate tokens without a round-trip to Identity.

---

## 📦 Technology Stack

| Component | Technology |
|---|---|
| Backend framework | ASP.NET Core 8 |
| ORM | Entity Framework Core 8 + Npgsql |
| Saga / Messaging | MassTransit 8.3 + RabbitMQ |
| gRPC | Grpc.AspNetCore 2.67 / Grpc.Net.Client |
| Reverse Proxy | YARP 2.3 |
| Logging | Serilog 8 |
| Database | PostgreSQL 16 |
| Frontend | React 18 + Vite 5 + React Router 6 + Axios |
| Containerisation | Docker + Docker Compose |
