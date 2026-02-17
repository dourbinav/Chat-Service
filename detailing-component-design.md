# Deep Dive into Chat System Component Design

This document provides a detailed look into the design of the core components of a scalable chat system, based on the principles outlined in "System Design Interview - An Insider's Guide."

## 1. Chat Service

The Chat Service is the backbone of the real-time communication system. It is a stateful service responsible for managing persistent connections with clients and routing messages between them.

### Core Responsibilities:

*   **Connection Management**: Establishes and maintains a WebSocket connection for every online user. This allows for low-latency, bidirectional communication.
*   **Message Routing**: Receives messages from a sender and forwards them to the intended recipient.
*   **User-to-Server Mapping**: Keeps track of which user is connected to which instance of the Chat Service.

### Detailed Architecture:

1.  **Connection Gateway**: This is the entry point for clients. A load balancer distributes incoming WebSocket connection requests across the available Chat Service instances.

2.  **User-to-Server Mapping**: To route messages, the system needs to know which server a user is connected to. This mapping is stored in a distributed cache like Redis.
    *   **Key**: `userID`
    *   **Value**: `serverID` (or server's IP address)

    When a user connects, the Chat Service instance handling the connection updates this mapping in Redis.

3.  **Message Flow**:
    1.  User A (sender) sends a message to User B (recipient) over their WebSocket connection to Chat Server 1.
    2.  Chat Server 1 receives the message and retrieves User B's server location from the Redis cache (`serverID` for `userID_B`).
    3.  If User B is online and connected to Chat Server 2, Chat Server 1 forwards the message to Chat Server 2.
    4.  Chat Server 2 then sends the message to User B over their WebSocket connection.
    5.  If User B is offline, the message is stored in the database and delivered when User B next comes online.

### Pseudo-code for Message Handling:

```
function onMessage(senderId, recipientId, messageContent) {
  // 1. Get recipient's server from cache
  recipientServer = redis.get(recipientId);

  if (recipientServer) {
    // 2. Forward message to the recipient's server
    forwardMessage(recipientServer, senderId, recipientId, messageContent);
  } else {
    // 3. Store message for later delivery
    storeOfflineMessage(recipientId, messageContent);
  }
}
```

## 2. Presence Service

The Presence Service is responsible for tracking the online/offline status of users.

### Core Responsibilities:

*   **Status Tracking**: Maintains the current status (online, offline, away) for all users.
*   **Status Updates**: Broadcasts status updates to a user's contacts (friends).

### Detailed Architecture:

1.  **Heartbeat Mechanism**: When a client establishes a WebSocket connection with the Chat Service, it starts sending periodic heartbeat signals.
2.  **Status Updates**:
    *   **On Connect**: When a user connects, the Presence Service is notified, and the user's status is set to "online" in a distributed cache (like Redis). A "last_seen" timestamp is also recorded.
    *   **On Disconnect**: If a user disconnects gracefully, their status is set to "offline". If the connection is lost unexpectedly, the absence of heartbeats for a certain period triggers a timeout, and the user is marked as "offline".
3.  **Publish/Subscribe Model**:
    *   When a user (e.g., User A) comes online, the Presence Service publishes this event to a "presence" topic.
    *   All of User A's contacts who are currently online are subscribed to this topic and receive the status update. This allows for the "green dot" next to a user's name to be updated in real-time.

## 3. Message Storage

A robust and scalable storage solution is critical for persisting chat history.

### Database Choice:

*   A **NoSQL database like Apache Cassandra** is an excellent choice for the following reasons:
    *   **High Write Throughput**: Chat applications are write-heavy.
    *   **Scalability**: Cassandra can scale horizontally to handle a massive number of messages.
    *   **Fault Tolerance**: It has built-in replication and no single point of failure.

### Data Model:

A common approach is to model conversations and messages in separate tables.

**Conversations Table:**
*   This table stores metadata about each conversation.
*   **Primary Key**: `conversation_id`
*   **Columns**: `participants` (a list of userIDs), `created_at`, `last_message_timestamp`.

**Messages Table:**
*   This is where the actual message content is stored.
*   **Primary Key**: `(conversation_id, message_id)` where `message_id` is a time-based UUID (like a `TimeUUID` in Cassandra) to ensure chronological order.
*   **Columns**: `sender_id`, `content`, `timestamp`.

### Example Schema (Cassandra CQL):

```cql
CREATE TABLE messages (
    conversation_id uuid,
    message_id timeuuid,
    sender_id uuid,
    content text,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

This schema allows for efficient retrieval of the latest messages for a given conversation, which is a very common access pattern.
