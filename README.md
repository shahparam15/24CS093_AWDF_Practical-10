# CHARUSAT ITUE301 — Practical 10: Asynchronous Processing with Event-Driven Architecture

**Student Roll Number:** 24CS093  
**Course:** Advanced Web Development Frameworks (ITUE301)  
**Subject:** Practical 10 — Asynchronous Processing with Event-Driven Architecture (Node.js EventEmitter)  

---

## 📌 Event Architecture & Asynchronous Flow

Practical 10 implements an in-process event-driven architecture using Node.js's built-in `EventEmitter` module to decouple slow notification side-effects (e.g., email dispatch, logging, webhooks) from critical synchronous HTTP request-response cycles.

```text
HTTP Client (Postman / Browser)
       │
       │ POST /tasks { title: "..." }
       ▼
Express API Handler
       │
       ├── 1. Validate Request Body
       ├── 2. Save Document to MongoDB
       ├── 3. Invalidate Relevant Caches
       ├── 4. Emit 'task-created' Event
       │      │
       │      ├──► Emit triggers background listener (non-blocking)
       │      │        ├── Timer initiated (2000 ms simulated delay)
       │      │        └── Notification dispatched asynchronously
       │      │
       └── 5. Return HTTP 201 Response IMMEDIATELY (30 ms)
```

---

## 🕒 Asynchronous Execution Proof: Timestamp Demonstration

To conclusively demonstrate that the HTTP request completes **before** slow background processing finishes, a controlled asynchronous delay of 2000ms was incorporated into the `task-created` listener.

### Real Measurement Timings

| Execution Step | Timestamp | Elapsed | Status |
| :--- | :--- | :---: | :--- |
| **Client Sent Request** | `2026-10-12T10:10:12.247Z` | 0 ms | Request in flight |
| **API Response Dispatched** | `2026-10-12T10:10:12.272Z` | +25 ms | HTTP 201 Created |
| **Client Received Response** | `2026-10-12T10:10:12.277Z` | **30.68 ms** | Response available to user |
| **Event Listener Started** | `2026-10-12T10:10:12.272Z` | +0 ms | Started in same tick |
| **Event Listener Completed** | `2026-10-12T10:10:14.280Z` | **+2003 ms** | Finished 2 seconds later |

> **Key Observation:**
> The client received the HTTP 201 response in **30.68 ms**, while the background notification handler finished **2003 ms later**. The client is never blocked waiting for slow notifications.

---

## 🛡️ Robust Error Handling & Fault Tolerance

1. **Global Error Listener**: An explicit `taskEvents.on("error", ...)` listener is registered on the shared EventEmitter instance. In Node.js, emitting an unhandled `'error'` event causes the process to throw and terminate; the global error handler safely logs the event without crashing the server.
2. **Listener Try/Catch Enclosure**: All synchronous and asynchronous logic within listeners is enclosed in `try...catch` blocks to prevent unhandled promise rejections.
3. **Database Write Isolation**: Events are emitted **strictly after** MongoDB confirms a successful write. Failed validations or database errors do not emit phantom events.
4. **Client Protection**: Notification failures never alter the HTTP response or expose internal server stack traces to clients.

---

## ⚠️ Limitations of Node.js EventEmitter

While `EventEmitter` is ideal for lightweight in-process decoupling, it has distinct architectural limitations in production systems:

1. **In-Memory & Process-Bound**: Events exist solely within the heap of the active Node.js process. They cannot be shared across multiple cluster worker processes or container replicas.
2. **Lack of Durability**: If the server crashes or restarts before a deferred listener executes, all in-flight events are permanently lost.
3. **No Automatic Retry / Backoff**: EventEmitter does not provide persistent retries, exponential backoff, or dead-letter queues.
4. *Production Recommendation*: For mission-critical distributed systems requiring durability and horizontal scalability, distributed message brokers such as **RabbitMQ**, **Redis Streams / BullMQ**, **Apache Kafka**, or **AWS SQS** should be used.

---

## 🚀 How to Run and Test

### 1. Run Automated Tests
```bash
npm run test:backend
# Or: cd backend && npm test
```
All **6 test suites pass**, verifying:
- Rapid HTTP 201 response with background listener emission
- Prevention of event emission on failed database operations
- `task-deleted` event handling on task deletion
- Event execution history via `GET /tasks/events/logs`
- Global error listener handling of simulated errors without process crashes

### 2. Run the Development Server
```bash
npm run dev:backend
# Starts Express server on http://localhost:5000
```
