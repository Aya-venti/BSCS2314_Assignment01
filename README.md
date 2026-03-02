# BSCS2314_Assignment01

# Publish-Subscribe System Implementation

**Name:** [Syeda Ayesha Gillani]  
**Student ID:** [BSCS23124]  
**Platform:** Windows (Winsock)

## Project Overview

This project implements a distributed publish-subscribe system with three components:
- **Broker**: Central server that manages topic subscriptions and routes messages
- **Publisher**: Clients that send messages to specific topics
- **Subscriber**: Clients that receive messages from topics they subscribe to

## Build Instructions

### Prerequisites
- Windows operating system
- Visual Studio 2019 or later (with C++ development tools)
- OR MinGW-w64 compiler
- OR CMake (optional)


### Method: Using CMake

mkdir build
cd build
cmake ..
cmake --build . --config Release


## Running the System

### Step 1: Start the Broker
Open a terminal/command prompt and run:
broker.exe 8080
The broker will start listening on port 8080.

### Step 2: Start Subscribers
Open new terminals for each subscriber:
subscriber.exe 127.0.0.1 8080 sports
subscriber.exe 127.0.0.1 8080 news weather
This creates subscribers for different topics.

### Step 3: Start Publishers
Open new terminals for publishers:
publisher.exe 127.0.0.1 8080

### Step 4: Publish Messages
In the publisher terminal, type messages in format:
sports Lakers win 120-115
news Breaking: Major announcement today
weather Sunny with a high of 25°C

Subscribers will receive messages for topics they've subscribed to.

### Shutdown
- Press `Ctrl+C` in the broker terminal to shut down the server
- Type `quit` in publisher terminals to exit
- Press any key in subscriber terminals to exit

## Design Choices and Thread Safety Implementation

### Thread Safety for Subscription Map

The subscription map (`std::map<std::string, std::vector<SOCKET>>`) is a critical shared resource that multiple client threads access concurrently. I implemented thread safety using the following approach:

1. **Mutex Protection**: A single `std::mutex` (`subscriptions_mutex`) protects all operations on the subscription map. Every read or write operation must acquire this lock using `std::lock_guard` for RAII-style exception-safe locking.

2. **Minimized Lock Duration**: The lock is held only during critical sections. For example, when publishing a message, the lock is released after copying the subscriber list, allowing the actual message forwarding to occur without holding the lock.

3. **Cleanup Operations**: When a client disconnects, its socket must be removed from all topic lists. This operation is performed under lock to prevent race conditions where a message might be sent to a disconnected client.

### Communication Protocol

The system uses a simple text-based protocol with newline-terminated messages:
- `SUBSCRIBE <topic>` - Client subscribes to a topic
- `PUBLISH <topic> <message>` - Publisher sends a message
- `MESSAGE <topic> <content>` - Broker forwards to subscribers

This design choice simplifies debugging and makes the system extensible for future features.

## Discussion: Scalability and Dependability

### Scalability Analysis

The centralized broker architecture presents several scalability challenges. The broker is a single point of bottleneck as it must handle all client connections, process all subscriptions, and route all messages. With the current threading model, each client gets a dedicated thread, which can lead to resource exhaustion with thousands of concurrent connections. The subscription map mutex becomes a contention point when many threads simultaneously try to access it during heavy publish/subscribe operations.

To improve scalability, several approaches could be considered:
- Implement a thread pool instead of per-client threads to better manage system resources
- Use non-blocking I/O with a reactor pattern (like `select()` or `IOCP` on Windows)
- Partition topics across multiple broker instances (topic-based sharding)
- Implement message batching to reduce lock contention

### Dependability Considerations

The current implementation has dependability concerns, primarily the single point of failure represented by the broker. If the broker crashes, all publishers and subscribers lose connectivity, and the system becomes completely unavailable. The protocol lacks acknowledgments, so message delivery is not guaranteed—if a subscriber disconnects temporarily, it misses messages published during that period.

To enhance dependability, we could implement:
- **Broker replication**: Use a primary-backup or multi-master replication scheme where multiple broker instances share subscription state
- **Message persistence**: Store messages in a durable log to allow recovery after crashes
- **Client acknowledgment**: Add ACK messages to confirm delivery, enabling at-least-once delivery semantics
- **Heartbeat mechanism**: Detect failed clients and brokers more quickly to clean up resources
- **Reconnection with state recovery**: Allow subscribers to reconnect and receive missed messages based on sequence numbers

The trade-off is that these dependability improvements add complexity and can impact performance, requiring careful design to maintain acceptable throughput.

