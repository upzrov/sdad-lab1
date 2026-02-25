# 🧪 Laboratory Work 1: Variant 3 — Offline Message Delivery

## 🧠 System Concept

To handle users who might be offline for days or weeks, we rely on a robust message broker. By utilizing RabbitMQ with Durable Exchanges and Lazy Queues, we can safely buffer messages on disk without exhausting system memory. The database transitions to a secondary role as a historical archive, while RabbitMQ handles the active delivery state.

---

## 🧱 Part 1 — Component Diagram

### Design Decisions

1. Message Broker (RabbitMQ): Acts as the central nervous system. Each user has a dedicated durable queue. Messages sit safely on disk in RabbitMQ until explicitly acknowledged by the client.

2. Delivery/Routing Worker: A backend service that processes incoming messages, routes them to the correct RabbitMQ queue, and triggers push notifications if the user's WebSocket connection is absent.

3. Archive DB: The database is no longer the "Inbox." It simply acts as a cold-storage archive for chat history, written to asynchronously.

```mermaid
graph TD
    Client[Client App Mobile/Web]
    
    subgraph Backend_Infrastructure [Backend Infrastructure]
        API[API Gateway / WebSocket Server]
        Auth[Auth Service]
        MsgService[Message Ingestion Service]
        Worker[Routing & Notification Worker]
    end
    
    subgraph Message_Broker [Message Broker]
        RMQ[[RabbitMQ]]
        Exchange((Direct Exchange))
        Q_UserA>Queue: User A]
        Q_UserB>Queue: User B]
    end
    
    subgraph Persistence_Layer [Persistence Layer]
        DB[(Archive Database)]
        Cache[(Session Cache)]
    end
    
    ExternalPush[External Push Provider FCM/APNS]

    %% Flows
    Client -->|1. Send Msg via WS/HTTP| API
    API --> Auth
    API --> MsgService
    
    MsgService -->|Publish| Exchange
    Exchange -->|Route Key: user.B| Q_UserB
    
    Worker -.->|Consume & Monitor| Q_UserB
    Worker -->|Archive Msg| DB
    Worker -->|Check Session| Cache
    Worker -->|If offline, trigger| ExternalPush
    ExternalPush -.->|Push Notification| Client
    
    Client -->|2. Reconnect WS| API
    API -->|Subscribe| Q_UserB
    Q_UserB -->|Push Pending Msgs| API
    API -->|Deliver| Client
```

---

## 🔁 Part 2 — Sequence Diagram

### Scenario: User Comeback (Synchronization)

This scenario details the moment User B comes back online. Instead of the server querying a database, the WebSocket server simply binds to User B's RabbitMQ queue. RabbitMQ automatically pushes the buffered messages, and the system waits for the client to confirm receipt before deleting them.

```mermaid
sequenceDiagram
    participant UserB as User B (Client)
    participant API as API/WebSocket Node
    participant RMQ as RabbitMQ (User B Queue)
    participant Cache as Session Cache

    Note over UserB, RMQ: User B comes online after 2 days. Messages are waiting in RabbitMQ.

    UserB->>API: CONNECT (WebSocket)
    API->>Cache: Update Status = ONLINE
    API-->>UserB: Connected
    
    Note right of API: API Node subscribes to<br/>User B's durable queue

    API->>RMQ: basic.consume(queue='user.b')
    RMQ-->>API: Deliver Msg 1051
    RMQ-->>API: Deliver Msg 1052
    RMQ-->>API: Deliver Msg 1053
    
    API-->>UserB: Push [Msg 1051, 1052, 1053] over WebSocket

    Note right of UserB: Client processes messages<br/>and updates local UI

    UserB->>API: ACK {ids: [1051, 1052, 1053]}
    API->>RMQ: basic.ack (multiple=true)
    
    Note over RMQ: RabbitMQ removes messages<br/>from the queue disk

```

---

## 🔄 Part 3 — State Diagram

### Object: `MessageDeliveryLifecycle`

With RabbitMQ, the state is primarily managed by the queue itself (Message sits in queue vs. Message is acknowledged).

```mermaid
stateDiagram-v2
    [*] --> Published : Ingested by API
    
    Published --> Queued : Routed to User's Durable Queue
    
    state "Queue / Buffer State" as BufferBlock {
        Queued --> NotifyOffline : Worker detects user offline
        NotifyOffline --> WaitingInQueue : Push Notification Sent
        WaitingInQueue --> Queued : Sits on disk (Lazy Queue)
    }
    
    Queued --> Unacknowledged : Pushed to Connected Consumer (WS)
    
    Unacknowledged --> Delivered : Client sends ACK
    Unacknowledged --> Queued : WS Drops (NACK / Requeue)
    
    Delivered --> [*] : Removed from RabbitMQ

```

---

## 📚 Part 4 — ADR (Architecture Decision Record)

### ADR-001: Message Broker Strategy for Offline Bufferin

## Status

Accepted

## Context

In "Variant 3: Offline Message Delivery," users may be offline for extended periods (weeks). We need a reliable mechanism to buffer these messages. Previously, database polling (the Inbox pattern) was considered to avoid overloading memory with millions of idle queues. However, database polling introduces high read latency, database locking issues, and complex cursor-management logic on the client.

## Decision

We will use RabbitMQ with Durable, Lazy Queues as the primary buffer and delivery mechanism.

Every user is assigned a specific routing key and a dedicated Durable Queue in RabbitMQ.

Queues are configured as Lazy Queues (queue.declare with x-queue-mode=lazy). This forces RabbitMQ to write messages immediately to disk rather than keeping them in RAM, perfectly suiting users who are offline for weeks.

We rely on RabbitMQ Consumer Acknowledgements (ACKs). Messages are not removed from the queue until the client explicitly confirms receipt.

The database is strictly used for asynchronous archiving, not active delivery.

## Alternatives

Database-First Buffering (Inbox Pattern): Rejected. High IOPS during "thundering herd" reconnections. Requires complex pagination and cursor state management.

In-Memory Redis Lists: Rejected. High risk of data loss on server restart and prohibitively expensive RAM usage for long-term offline buffering.

## Consequences

Reliability: Messages are safely persisted to disk via RabbitMQ. If a WebSocket connection drops mid-delivery, the un-ACKed messages are safely requeued.

Simplicity: The backend logic is heavily simplified. The server no longer tracks "what the user missed"; it simply subscribes to the queue, and RabbitMQ handles the cursor and delivery state.

Infrastructure Overhead: We must manage and monitor a RabbitMQ cluster, ensuring disk space is sufficient for the lazy queues and that connection limits are tuned for many idle queues.
