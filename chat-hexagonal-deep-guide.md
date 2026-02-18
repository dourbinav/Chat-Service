# Chat Service -- Deep Hexagonal Architecture Guide

This document explains in depth:

1. What each folder is responsible for
2. What each folder MUST NOT contain
3. Where to write specific types of code
4. How to swap PostgreSQL with MySQL or MongoDB by changing code in
   only ONE place

---

# 1. High-Level Philosophy

Hexagonal Architecture (Ports & Adapters) separates:

- Core Business Logic (Domain)
- Interfaces (Ports)
- Infrastructure Implementations (Adapters)
- Initialization (Infra)
- Dependency Wiring (App / Container)

Core must NEVER depend on database, Kafka, gRPC, WebSocket, or
frameworks.

---

# 2. Final Folder Structure

chat-service/ ├── cmd/server/main.go ├── internal/ │ ├── core/ │ │ ├──
domain/ │ │ ├── ports/ │ │ └── service/ │ │ │ ├── adapters/ │ │ ├──
inbound/ │ │ └── outbound/ │ │ │ ├── infra/ │ ├── app/ │ └── config/

---

# 3. Folder-by-Folder Deep Explanation

## CORE LAYER (Pure Business Logic)

### internal/core/domain/

**What goes here:** - Domain models (Message, Room, User) - Domain
validation rules - Business invariants

**What must NOT go here:** - SQL queries - Kafka imports - HTTP or
WebSocket logic - gRPC client calls

Responsibility: Define what a Message is, not how it is stored.

---

### internal/core/ports/

**What goes here:** - Interfaces that define dependencies - Repository
interfaces - Event publisher interfaces - External service interfaces

Example:

type MessageRepository interface { Save(ctx context.Context, msg
\*Message) error }

These are contracts that allow swapping implementations later.

---

### internal/core/service/

**What goes here:** - Business logic - Use cases - Application rules

Example responsibilities: - Validate message content - Call
repository.Save() - Call publisher.Publish()

**Must NOT contain:** - sql.DB - kafka.Writer - grpc.ClientConn -
http.Request

This layer must be fully testable with mocks.

---

## ADAPTERS LAYER (Implements Ports)

### internal/adapters/inbound/

**What goes here:** - WebSocket handlers - HTTP handlers - gRPC server
handlers

Responsibilities: - Convert external request into domain objects - Call
core services - Return response

Must NOT: - Contain business rules - Contain direct SQL queries

---

### internal/adapters/outbound/

**What goes here:** - Postgres repository implementation - Kafka
publisher implementation - gRPC client implementation

Responsibilities: - Implement core ports - Translate domain models into
DB queries or events

---

## INFRA LAYER (Initialization Only)

### internal/infra/

This layer builds infrastructure clients.

**What goes here:** - Create database connections - Create Kafka
producers - Create gRPC connections - Start HTTP server - Graceful
shutdown logic

Important: No business logic here.

---

## APP LAYER (Dependency Wiring)

### internal/app/container.go

This is the only place where everything is connected.

Responsibilities: - Initialize infra - Create adapter implementations -
Inject dependencies into core services - Return ready-to-use handlers

This is where swapping databases becomes powerful.

---

## CONFIG LAYER

### internal/config/

Load environment variables. Define configuration structs. No business
logic.

---

# 4. How Database Swapping Works

Core depends ONLY on:

type MessageRepository interface

The core does NOT care about database type.

Currently you may have:

adapters/outbound/postgres/message_repository.go

If tomorrow you want MySQL:

adapters/outbound/mysql/message_repository.go

If you want MongoDB:

adapters/outbound/mongo/message_repository.go

Each must implement:

MessageRepository interface

---

# 5. The ONLY Place You Change Code

You change code in:

internal/app/container.go

Example:

Before (Postgres):

messageRepo := postgres.NewMessageRepository(db)

After (MySQL):

messageRepo := mysql.NewMessageRepository(mysqlDB)

After (Mongo):

messageRepo := mongo.NewMessageRepository(mongoClient)

Core does not change. Services do not change. WebSocket does not change.
Kafka does not change.

Only one line changes.

---

# 6. Strict Rules

1. Core must not import adapters.
2. Core must not import infra.
3. Adapters must implement core ports.
4. Only app/container wires dependencies.
5. main.go must stay minimal.

---

# 7. Mental Model

Core = Brain
Ports = Contracts
Adapters = Plug Types
Infra = Tools
App = Wiring Board
Main = Power Button

---

# 8. Testing Strategy

To test core:

- Mock MessageRepository
- Mock EventPublisher

No DB needed. No Kafka needed. No gRPC needed.

---

# 9. Final Result

With this architecture:

✔ You can swap databases easily
✔ You can swap Kafka with RabbitMQ
✔ You can switch WebSocket to gRPC streaming
✔ You protect business logic
✔ You gain long-term maintainability

This is production-grade architecture used in scalable systems.
