# bookeasy — Backend High-Level Design

> The source code is private (proprietary SaaS). This document provides an architectural overview and demonstrates the system design.
---

## Contents

1. [Description](#1-description)
2. [Technology Stack](#2-technology-stack)
3. [Architecture](#3-architecture)
4. [Project Structure](#4-project-structure)
5. [Domain Diagram](#5-domain-diagram)
6. [Anonymous Booking Flow](#6-anonymous-booking-flow)
7. [Architecture Decisions (ADR)](#7-architecture-decisions-adr)

---

## 1. Description

**BOOKEASY** is a SaaS platform for automating client appointments for beauty industry professionals. The system provides a professional with tools to manage their schedule, services, and client base, supporting online booking, automated reminders, and SaaS billing.

---

## 2. Technology Stack

| Category | Technology |
| --- | --- |
| Runtime | Python 3.12 |
| Web & Real-Time APIs | FastAPI, Socket.IO |
| Persistence & Storage | PostgreSQL, Cloudflare R2, SQLAlchemy 2.0 |
| Cache, Broker, Event Bus | Redis |
| Task Queue | Celery |
| Bot Framework | aiogram 3 |
| Payment Gateway | WayForPay |
| Observability | Sentry SDK |

---

## 3. Architecture

The project follows Clean Architecture and Domain-Driven Design (DDD).

```
┌────────────────────────────────────────┐
│          Presentation Layer            │
│   REST API (FastAPI) | Telegram Bot    │
│         WebSocket (Socket.IO)          │
└──────────────┬─────────────────────────┘
               │ 
┌──────────────▼─────────────────────────┐
│          Application Layer             │
│  Interactors · Event Handlers          │
│  Interfaces (Ports) · DTOs             │
└──────────────┬─────────────────────────┘
               │
┌──────────────▼─────────────────────────┐
│            Domain Layer                │
│  Entities · Value Objects · Events     │
│  Repository Interfaces · Exceptions    │
└────────────────────────────────────────┘
               ▲
┌──────────────┴─────────────────────────┐
│         Infrastructure Layer           │
│  PostgreSQL · Redis · Celery           │
│  WayForPay · Notifiers (TG/SMS/WS/PUSH)     │
│  transports/http · Google OAuth        │
│  JWT · Argon2 · Sentry                 │
└────────────────────────────────────────┘
               ▲
┌──────────────┴─────────────────────────┐
│           Bootstrap Layer              │
│  Dishka IoC · Event Subscriptions      │
│  ASGI App · Bot Runner · Celery Worker │
└────────────────────────────────────────┘
```

---

## 4. Project Structure

```text
src/
├── core/
├── domain/                    
│   ├── entities/
│   ├── value_objects/
│   ├── events/               
│   ├── exceptions/
│   ├── repositories/
│   └── enums/
├── application/    
│   ├── use_cases/                                  
│   ├── event_handlers/       
│   ├── services/              
│   ├── interfaces/            
│   └── dto/                   
├── infrastructure/            
│   ├── persistence/
│   │   ├── object_storage/
│   │   ├── postgres/          
│   │   └── redis/             
│   ├── messaging/
│   │   ├── event_bus/         
│   │   ├── notifiers/         
│   │   └── queues/celery/     
│   ├── transports/http/       
│   ├── billing/               
│   ├── security/      
│   ├── media/             
│   └── telemetry/             
├── presentation/              
│   ├── api/                   
│   └── bot/                   
└── bootstrap/                 
    ├── container/             
    ├── entrypoints/           
    ├── setup/
    └── subscriptions.py
```

---

## 5. Domain Diagram

```mermaid
classDiagram
    direction TB

    %% ── Identity ────────────────────────────────
    class Identity { <<abstract>> }
    class TelegramIdentity
    class GoogleIdentity
    class EmailIdentity
    class AnonymousIdentity
    
    Identity <|-- TelegramIdentity
    Identity <|-- GoogleIdentity
    Identity <|-- EmailIdentity
    Identity <|-- AnonymousIdentity

    %% ── User Aggregate ──────────────────────────
    class User {
        +add_identity(identity) void
        +revoke_identity(provider) void
        +has_identity(provider) bool
        +rename(value) void
    }
    User *-- Identity : identities

    class Master {
        +set_notifications(channel) void
        +reschedule_reminder(minutes) void
        +change_contact_phone_number(value) void
        +change_contact_telegram_link(value) void
        +change_contact_instagram_link(value) void
    }
    class Client {
        +assign_master(master_id) void
        +add_instagram(instagram) void
    }
    
    User <|-- Master
    User <|-- Client
    Client --> Master : assigned to

    %% ── Master Configs ──────────────────────────
    class Contact {
        +change_phone_number(value) void
        +change_telegram_link(value) void
        +change_instagram_link(value) void
        +change_address(value) void
    }
    class ProfileConfig {
        +reschedule_reminder(minutes) void
        +set_notifications(channel) void
    }
    
    Master *-- Contact : has
    Master *-- ProfileConfig : configures

    %% ── Portfolio ────────────────────────────────
    class Photo {
        +shift_position(position) void
    }
    Master *-- "N" Photo : showcases

    %% ── Expenses Aggregate ───────────────────────
    class Expense {
        +rename(title) void
        +adjust_amount(amount) void
        +shift_date(dt) void
        +reclassify(category) void
    }
    Master *-- "N" Expense : incurs

    %% ── Scheduling & Services ───────────────────
    class Service {
        +reprice(price) void
        +rename(title) void
        +reschedule(duration) void
        +redescribe(description) void
        +recolor(color) void
        +archive() void
    }
    class TimeSlot {
        +book() void
        +release() void
        +waste() void
        +resize(duration) void
        +can_book(now) bool
        +is_due(now) bool
        +is_started(now) bool
        +overlaps(other) bool
    }

    Master *-- "N" Service : manages
    Master *-- "N" TimeSlot : owns

    %% ── Booking Aggregate ────────────────────────
    class Booking {
        +confirm() void
        +cancel() void
        +complete() void
        +set_notes(notes) void
        +should_schedule_reminder(now) bool
    }

    Booking --> Service : for
    Booking --> TimeSlot : occupies
    Booking --> Master : belongs to
    Booking --> Client : placed by

    %% ── Billing Aggregate ────────────────────────
    class Plan {
        +is_free() bool
        +has_feature(name) bool
        +has_quota(name) bool
        +get_quota(name) Quota
    }
    class Subscription {
        +activate() void
        +assign_token(token) void
        +renew(period: BillingPeriod) void
        +cancel() void
        +is_active() bool
        +switch_plan(new_plan) void
        +has_access(now) bool
    }
    class Invoice {
        +mark_as_paid() void
        +void() void
        +is_paid() bool
        +is_open() bool
    }
    class Payment {
        +assign_payment_link(link: PaymentLink) void
        +mark_success() void
        +mark_failed() void
        +is_expired(now) bool
    }

    Subscription --> Plan : subscribed to
    Master *-- "N" Subscription : owns
    Subscription *-- "N" Invoice : generates
    Invoice *-- Payment : fulfilled by
```

---

## 6. Anonymous Booking Flow

A client without an account books an appointment via the master's public link.

```mermaid
sequenceDiagram
    autonumber
    actor C as Anonymous Client
    participant P as Presentation Layer
    participant A as Application Layer
    participant D as Domain
    participant DB as PostgreSQL
    participant Cache as Redis
    participant Bus as Event Bus
    participant Q as Celery Worker
    participant N as Notifier

    C->>P: POST /api/v1/bookings/anonymous
    P->>A: MakeAnonymousBookingInteractor.execute(command)

    A->>DB: Resolve master · client (upsert) · slot · service
    DB-->>A: entities | Failure → 404 / 409

    A->>D: Booking.confirm()
    note over D: Validates slot availability<br/>and booking deadline

    alt BookingError
        D-->>A: Failure(UNPROCESSABLE_ENTITY) → 422
        A-->>P: Failure
        P-->>C: 422
    else OK
        A->>DB: Persist booking (optimistic lock on TimeSlot)
        alt Version conflict
            DB-->>A: Failure — slot already booked → 422
            A-->>P: Failure
            P-->>C: 422
        else Commit OK
            A->>Bus: publish(BookingCreated)
            A->>Cache: store hash(claim_token) TTL 30m
            A-->>P: Success(booking, claim_token)
            P-->>C: 201 Created
        end
    end

    Bus->>Q: BookingCreatedHandler
    Q->>Bus: publish(BookingDeactivated)
    Q->>N: Notify master
    Q->>Q: Schedule reminders

    Bus->>Q: BookingDeactivationHandler → Schedule auto-complete
```

The `claim_token` allows an anonymous client to view or cancel the booking within 30 minutes. A hash is stored in Redis — the original is not persisted.

---

## 7. Architecture Decisions (ADR)

### ADR-001: Unified Redis for Broker, Cache, Session, and Event Bus

- **Decision:** Use Redis as a single infra dependency for Celery broker, cache/session storage, and event bus transport.
- **Rationale:** Reduces operational surface and speeds delivery at the current product stage.
- **Trade-off:** Part of runtime messaging is non-durable by design; acceptable at this stage.

### ADR-002: Async-to-Sync Bridge for Celery Runtime

- **Decision:** Execute async interactors and async DI scope inside Celery via a long-lived event loop per worker process.
- **Rationale:** Keeps application/services async-first without rewriting worker integration to sync code.
- **Trade-off:** Additional lifecycle complexity in worker bootstrap/shutdown.

### ADR-003: Booking Consistency via Optimistic Concurrency on Time Slots

- **Decision:** Enforce slot state transitions through optimistic locking (`version`) during persistence.
- **Rationale:** Prevents double booking under concurrent writes while preserving throughput.
- **Trade-off:** Conflict handling is pushed to application flow (`422`/retry paths).

### ADR-004: Ephemeral Anonymous Claim Token (Hashed + TTL)

- **Decision:** Store only hashed anonymous claim tokens in Redis with strict TTL.
- **Rationale:** Enables temporary ownership/cancel flow without persisting raw bearer tokens.
- **Trade-off:** Claim is intentionally short-lived and non-recoverable after expiration.

### ADR-005: Best-Effort Internal Event Handling with Cross-Process Dedup

- **Decision:** Use Redis Pub/Sub for internal events with deduplication guard by `event_id`.
- **Rationale:** Lightweight event propagation between app processes and worker flows.
- **Trade-off:** Best-effort / at-most-once delivery semantics (no replay log); consumers must stay idempotent and tolerant to delivery timing.

### ADR-006: Single-Use Ticket Pattern for WebSocket Authentication

- **Decision:** Authenticate WebSocket connections via short-lived single-use tickets with TTL requested via `POST /api/v1/auth/ws-ticket` and consumed atomically (`GETDEL`) in Redis during handshake.
- **Rationale:** Reuses the existing HTTP authentication and permission pipeline, avoiding duplicated auth logic in the WebSocket transport.
- **Trade-off:** Requires an initial HTTP POST roundtrip before opening the socket connection.
