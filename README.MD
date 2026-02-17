# Chat Application -- Hexagonal Architecture

## 🚀 Overview

This project is a scalable chat application built using:

-   WebSocket → Real-time messaging
-   Kafka → Event-driven notification processing
-   gRPC → Inter-service communication
-   PostgreSQL → Message persistence
-   Redis (optional) → Horizontal scaling & presence
-   Hexagonal Architecture (Ports & Adapters) → Clean separation of
    concerns

------------------------------------------------------------------------

# 🏗 Architecture Overview

Client (Web / Mobile) │ ▼ WebSocket Adapter (Chat Service) │ ▼ Core
Business Logic (Hexagonal Core) │ ├── Save Message → Postgres Adapter
├── Publish Event → Kafka Adapter └── gRPC Calls → Other Services │ ▼
Notification Service (Kafka Consumer)

------------------------------------------------------------------------

# 🧱 Hexagonal Architecture Structure

## Chat Service

chat-service/ ├── cmd/ │ └── main.go ├── internal/ │ ├── core/ │ │ ├──
domain/ │ │ │ └── message.go │ │ ├── ports/ │ │ │ ├──
message_repository.go │ │ │ ├── event_publisher.go │ │ │ └──
user_service.go │ │ └── service/ │ │ └── chat_service.go │ ├── adapters/
│ │ ├── websocket/ │ │ ├── postgres/ │ │ ├── kafka/ │ │ └── grpc/ │ └──
config/ ├── go.mod └── README.md

------------------------------------------------------------------------

# 🧠 Core Layer Example

## Domain Model

type Message struct { ID string SenderID string RoomID string Content
string CreatedAt time.Time }

------------------------------------------------------------------------

## Ports (Interfaces)

type MessageRepository interface { Save(msg \*Message) error }

type EventPublisher interface { PublishMessageSent(msg \*Message) error
}

type UserService interface { GetUserDevices(userID string) (\[\]string,
error) }

------------------------------------------------------------------------

## Core Service

type ChatService struct { repo MessageRepository publisher
EventPublisher }

func (s *ChatService) SendMessage(msg *Message) error { if msg.Content
== "" { return errors.New("message cannot be empty") }

    if err := s.repo.Save(msg); err != nil {
        return err
    }

    return s.publisher.PublishMessageSent(msg)

}

------------------------------------------------------------------------

# 🔔 Notification Service Structure

notification-service/ ├── core/ │ ├── domain/ │ ├── ports/ │ └──
service/ ├── adapters/ │ ├── kafka/ │ ├── grpc/ │ └── push/

------------------------------------------------------------------------

# 📡 Message Flow

1.  User sends message via WebSocket
2.  Chat Service validates and stores message
3.  Event is published to Kafka
4.  Notification Service consumes event
5.  Notification sent via Push/Email/SMS

------------------------------------------------------------------------

# ⚖ Why Hexagonal?

-   Infrastructure independent core
-   Easy unit testing
-   Replace Kafka without changing business logic
-   Replace DB without changing business logic
-   Clean separation of concerns
-   Microservice friendly

------------------------------------------------------------------------

# 🚀 Scaling Strategy

-   Multiple Chat Service instances
-   Redis Pub/Sub for cross-instance broadcasting
-   Kafka for durable async events
-   gRPC for sync service communication

------------------------------------------------------------------------

# 🏁 Stack Summary

WebSocket + Kafka + gRPC + PostgreSQL + Redis\
Hexagonal Architecture (Ports & Adapters)

Production-ready. Scalable. Testable.

