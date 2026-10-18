#SystemDesign #WebCommunication #WebSockets #SignalR #ServerSentEvents #Webhooks #WebRTC #DotNet #Architecture

> [!abstract] Executive Summary
> While the **Synchronous HTTP Request/Response cycle** powers roughly 80% of standard web applications, modern software often breaks down under its constraints. When systems require **sub-millisecond real-time updates, AI token streaming, long-running background tasks, peer-to-peer media, or offline collaboration**, architects reach for alternative communication paradigms.
> 
> This guide covers the **complete spectrum of modern web communication models**, explaining how each works under the hood, their trade-offs, real-world .NET implementations, and an architectural decision tree to pick the right tool for the job.

---

# 🗺️ 1. The Global Taxonomy of Web Communication

```
                               ┌─────────────────────────────────────────┐
                               │   WEB COMMUNICATION PARADIGM SPECTRUM   │
                               └────────────────────┬────────────────────┘
                                                    │
         ┌──────────────────────────┬───────────────┴───────────────┬──────────────────────────┐
         ▼                          ▼                               ▼                          ▼
 ┌───────────────┐          ┌───────────────┐               ┌───────────────┐          ┌───────────────┐
 │ 1. BI-DIRECT. │          │ 2. STREAMING  │               │ 3. DECOUPLED  │          │ 4. DIRECT /   │
 │  FULL-DUPLEX  │          │  (ONE-WAY)    │               │  ASYNCHRONOUS │          │  SPECIALIZED  │
 ├───────────────┤          ├───────────────┤               ├───────────────┤          ├───────────────┤
 │ • WebSockets  │          │ • Server-Sent │               │ • Webhooks    │          │ • WebRTC (P2P)│
 │ • SignalR     │          │   Events (SSE)│               │ • Queue RPC   │          │ • OS Push     │
 │               │          │               │               │ • Long-Polling│          │ • CRDTs (Sync)│
 └───────────────┘          └───────────────┘               └───────────────┘          └───────────────┘
```

---

# ⚡ 2. The Paradigms Deep Dive

---

## 1. Bi-Directional Full-Duplex: WebSockets & SignalR

### How it Works
1. **The Handshake:** The browser begins with a standard HTTP GET request carrying two headers:
   `Connection: Upgrade` and `Upgrade: websocket`.
2. **The 101 Protocol Switch:** The server responds with `HTTP/1.1 101 Switching Protocols`.
3. **The Persistent Socket:** The TCP connection is kept open indefinitely. The HTTP protocol is stripped away; communication transitions to **raw WebSocket binary/text frames**.
4. **Full-Duplex:** Either client or server can transmit packets at any millisecond with as little as **2 bytes of frame overhead** (compared to ~1,000 bytes of HTTP headers per request).

```
[ Client ] ──(HTTP GET Upgrade)──► [ Server ]
[ Client ] ◄──(101 Switching)────── [ Server ]
══════════════ [ Persistent TCP Connection Open ] ══════════════
[ Client ] ──────(2-byte Frame)───► [ Server ]  (Client pushes state)
[ Client ] ◄─────(2-byte Frame)──── [ Server ]  (Server pushes state)
```

### Best Use Cases
* High-frequency bidirectional data: **Multiplayer gaming, live chat applications, financial trading orderbooks, collaborative whiteboards (Figma/Miro)**.

### The Trade-Offs
* ✅ **Pros:** True zero-latency push; virtually zero packet overhead; bidirectional.
* ❌ **Cons:** Stateful connections consume server RAM (100,000 open connections = 100,000 open sockets); difficult to scale horizontally across servers without a shared backplane; corporate firewalls sometimes block non-HTTP/80 traffic.

### C# / .NET Reality: ASP.NET Core SignalR
In .NET, you rarely write raw WebSocket handlers. You use **SignalR**, which provides high-level RPC Hubs, automatic transport fallback (WebSockets $\rightarrow$ SSE $\rightarrow$ Long Polling), and horizontal scaling via a **Redis Backplane**:

```csharp
// 1. The Hub (Server)
public class ChatHub : Hub
{
    public async Task SendMessage(string user, string message) =>
        await Clients.All.SendAsync("ReceiveMessage", user, message);
}

// 2. Program.cs
builder.Services.AddSignalR().AddStackExchangeRedis("redis-connection-string");
app.MapHub<ChatHub>("/chathub");
```

---

## 2. Unidirectional Server-Push: Server-Sent Events (SSE)

### How it Works
1. The client opens a standard HTTP request with `Accept: text/event-stream`.
2. The server responds with `Content-Type: text/event-stream` and `Transfer-Encoding: chunked`.
3. **The stream stays open forever:** The server periodically writes UTF-8 text lines formatted as `data: {...}\n\n`.
4. The browser's native `EventSource` JavaScript API automatically reconnects if the connection drops.

```
[ Client ] ───(GET /events, Accept: text/event-stream)───► [ Server ]
[ Client ] ◄──(200 OK, Content-Type: text/event-stream)─── [ Server ]
[ Client ] ◄──(data: {"token": "Hello"}\n\n)────────────── [ Server ]
[ Client ] ◄──(data: {"token": " World"}\n\n)───────────── [ Server ]
```

### WebSockets vs. SSE: The Critical Comparison
* **Direction:** WebSockets is **Bi-directional**; SSE is **Server-to-Client only** (the client cannot push data up the stream; it sends regular HTTP POSTs for client mutations).
* **Transport:** WebSockets replaces HTTP with a custom TCP socket protocol; SSE is **100% standard HTTP/2 or HTTP/3**. It multiplexes seamlessly over existing HTTP/2 connections with zero firewall issues.
* **Resilience:** Browsers have built-in automatic reconnect and message ID tracking for SSE out of the box.

### Best Use Cases
* **AI Text Generation (ChatGPT / Claude token-by-token streaming)**.
* Live sports score tickers, server CPU monitoring dashboards, real-time social notification feeds.

### C# / .NET Reality
Implemented in ASP.NET Core using `IAsyncEnumerable<T>` or chunked HTTP responses:
```csharp
app.MapGet("/ai/stream", async (HttpResponse response, CancellationToken ct) =>
{
    response.Headers.ContentType = "text/event-stream";
    foreach (var token in new[] { "Modern ", "System ", "Design ", "in ", ".NET" })
    {
        await response.WriteAsync($"data: {token}\n\n", ct);
        await response.Body.FlushAsync(ct);
        await Task.Delay(200, ct);
    }
});
```

---

## 3. Asynchronous Callbacks: The Webhook Pattern

### How it Works
When a task takes 15 seconds to 2 hours (e.g., credit card clearing, video transcoding, AI image generation), holding an HTTP connection open is an anti-pattern (it exhausts web server thread pools and triggers client timeouts).

Instead, systems use the **"Don't call us, we'll call you"** model:
1. Client requests a job $\rightarrow$ Server accepts it, stores a `JobId`, and immediately returns **`202 Accepted`**.
2. A background worker executes the job asynchronously.
3. Upon completion, the server makes an **outbound HTTP POST request** to a callback URL provided by the client (the **Webhook**).

```
[ Client ] ────(POST /exports, CallbackUrl: "https://client.com/hook")────► [ Server ]
[ Client ] ◄───(HTTP 202 Accepted, JobId: "123")─────────────────────────── [ Server ]
                                                                                │
                                                                                ▼ (Background Worker runs 5 mins)
[ Client ] ◄───(POST https://client.com/hook, Body: {Status: "Done"})───────── [ Server ]
[ Client ] ────(HTTP 200 OK)──────────────────────────────────────────────► [ Server ]
```

### The 3 Production Rules for Webhooks
1. **Cryptographic Signatures:** The sender signs the payload using an HMAC SHA-256 secret key (`X-Signature: sha256=...`). The receiver verifies this to prevent attackers from sending fake payment webhooks.
2. **Retries with Exponential Backoff:** If the receiver returns a `500` or times out, the sender must retry (e.g., after 5s, 1m, 15m, 1hr) before sending to a Dead-Letter Queue.
3. **Receiver Idempotency:** The receiver must check whether the `EventId` was already processed to avoid processing duplicate deliveries twice.

---

## 4. Peer-to-Peer (P2P): WebRTC (Web Real-Time Communication)

### How it Works
WebRTC enables browsers to stream high-bandwidth video, audio, and binary data **directly to each other without touching the application server**.

1. **Signaling Phase:** Browser A and Browser B talk to an intermediary server (via WebSocket) to exchange metadata (SDP - Session Description Protocol: supported codecs, resolution, IP addresses).
2. **NAT Traversal (STUN / TURN):** STUN servers discover each browser's public IP address behind residential home routers and NATs. If symmetric NATs block direct connections, a TURN relay server relays traffic.
3. **Direct P2P Data Channel:** Once connected, raw media and data travel directly over encrypted UDP (SRTP / SCTP) between the two devices.

```
                  ┌──────────────────────────────┐
                  │ Signaling Server (WebSocket) │
                  └──────┬────────────────┬──────┘
       Exchange SDP & IP │                │ Exchange SDP & IP
                         ▼                ▼
                  [ Browser A ] ◄──────► [ Browser B ]
                      ▲                     ▲
                      └──────(Direct P2P)───┘
                       Encrypted Media over UDP
```

### Best Use Cases
* **Google Meet, Zoom, Microsoft Teams video calls**, Discord voice channels, browser-to-browser torrenting.

---

## 5. OS-Level Device Wakeup: Push Notifications (APNs & FCM)

### How it Works
When a mobile app or browser tab is **completely closed and asleep**, WebSockets and HTTP connections are dead. You cannot send an HTTP request directly to a phone because its IP changes constantly and firewalls block incoming connections.

1. The device registers with the OS vendor: **Apple Push Notification service (APNs)** for iOS or **Firebase Cloud Messaging (FCM)** for Android, receiving a unique `DeviceToken`.
2. The phone sends its `DeviceToken` to your backend database.
3. When an event occurs (e.g., "New Message"), your backend sends an HTTP request to **Apple/Google's servers**.
4. Apple/Google uses their low-power, persistent operating system connection to wake the device's screen.

```
[ Your Backend ] ──(POST to Apple/Google API)──► [ Apple APNs / Google FCM ]
                                                           │
                                                           ▼ (System-Level Socket)
                                                 [ Locked Mobile Phone ] ──► "You have a new message!"
```

---

## 6. Asynchronous Queue RPC (Request-Reply over Message Brokers)

### How it Works
In distributed backend microservices, services often need to communicate asynchronously without coupling to each other's network addresses or risking cascading HTTP timeouts:
1. **Service A** drops a command message into a shared RabbitMQ/Kafka queue.
2. The message contains two routing metadata headers:
   * `ReplyTo: "temp-reply-queue-serviceA"`
   * `CorrelationId: "guid-456"`
3. **Service B** consumes the message, runs the calculations, and publishes the result into the specified `ReplyTo` queue stamped with the same `CorrelationId`.
4. **Service A** listens on its reply queue, matches the `CorrelationId`, and resumes execution.

```
[ Service A ] ──(Command, CorrelationId: 123, ReplyTo: temp-q)──► [ Work Queue ]
                                                                         │
                                                                         ▼
[ Service A ] ◄─(Result, CorrelationId: 123)── [ temp-q ] ◄──── [ Service B ]
```

---

## 7. The "Hanging Request": HTTP Long-Polling

### How it Works
The predecessor to WebSockets and SSE. Used as an automatic fallback when strict enterprise firewalls or proxies block WebSocket upgrades:
1. Browser sends a standard `GET /notifications`.
2. The server **deliberately does not respond immediately**. It holds the connection open ("hangs") for up to 30–60 seconds.
3. The moment new data arrives, the server writes the response and closes the connection.
4. The client's JavaScript immediately fires another hanging request to start listening again.

---

## 8. Local-First with CRDTs (Conflict-Free Replicated Data Types)

### How it Works
Used in modern collaborative apps (like **Figma, Linear, and Notion**):
1. State mutations are committed **immediately to local device memory / IndexedDB** in 0 milliseconds (zero network wait).
2. The mutation is expressed as a mathematical CRDT operation (e.g., state-based or operation-based delta).
3. The client broadcasts delta packets in the background.
4. When concurrent offline edits arrive from multiple users, the mathematical rules of CRDTs guarantee that **all nodes merge the edits into the exact same deterministic state** without manual merge conflict dialogues or server rollbacks.

---

# 📊 3. Master Communication Paradigms Comparison Matrix

| Paradigm | Transport Protocol | Directionality | Connection Lifecycle | Network Overhead | Best Real-World Use Case | Primary Failure Mode |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Standard HTTP (Note 01)** | TCP (HTTP/1.1 - 3) | Request $\rightarrow$ Response | Ephemeral (ms) | High (~1KB headers) | Standard CRUD, public REST APIs | Cascading latency on slow DB calls |
| **WebSockets (SignalR)** | TCP (Upgraded) | **Bi-directional** | Persistent (hours/days) | Ultra-Low (2 bytes/frame) | Multiplayer games, live financial orderbooks | Socket exhaustion on server RAM |
| **Server-Sent Events (SSE)** | HTTP/2 / HTTP/3 | **Server $\rightarrow$ Client** | Long-lived stream | Low (Text chunks) | **AI token streaming (ChatGPT)**, live scoreboards | Unidirectional only; HTTP/1.1 connection limits |
| **Webhooks** | HTTP POST | Server $\rightarrow$ Client | Asynchronous | Medium (Standard HTTP) | Payment confirmations (Stripe), CI/CD events | Receiver downtime & duplicate deliveries |
| **WebRTC** | UDP (SRTP / SCTP) | **Peer-to-Peer** | Persistent direct session | Extremely Low | Video/Voice calls (Zoom, Google Meet), P2P files | Symmetric NAT blocks (requires TURN server) |
| **OS Push Notifications** | Vendor TCP Socket | OS $\rightarrow$ Device | Intermittent Wakeup | Negligible | Mobile alerts when app is closed (Uber, WhatsApp) | Message delivery not 100% guaranteed |
| **Queue RPC** | AMQP / Kafka | Asynchronous Bi-directional| Decoupled queues | Low (Broker binary) | High-volume backend microservice RPC | Reply queue orchestration complexity |
| **HTTP Long-Polling** | Standard HTTP | Client pulls (Hangs) | 30s timeout cycles | High (Repeated HTTP handshakes) | Fallback for restricted enterprise firewalls | Server thread starvation under load |
| **Local-First (CRDTs)** | Local DB + Async Sync | Peer-to-Peer / Hub-Spoke | Eventual background sync | Mathematical deltas | Real-time collaboration (Figma, Linear) | Memory overhead for complex CRDT tombstones |

---

# 🌳 4. The Senior Architectural Decision Tree

When choosing how a client and server should communicate, run your requirements through this algorithm:

```
Do you need real-time data flow?
 │
 ├── NO  ──► Is the operation fast (<5s)?
 │            ├── YES ──► Use Standard REST / GraphQL HTTP Request-Response (Note 01)
 │            └── NO  ──► Use Asynchronous Webhook / 202 Accepted Job Polling
 │
 └── YES ──► Does the client need to send high-frequency data BACK to the server over the same pipe?
              │
              ├── YES ──► Is it direct audio/video streaming between users?
              │            ├── YES ──► Use WebRTC (P2P)
              │            └── NO  ──► Use WebSockets / ASP.NET Core SignalR
              │
              └── NO  ──► Does the data flow ONE-WAY from server to client (e.g. AI tokens, stock prices)?
                           ├── YES ──► Use Server-Sent Events (SSE)
                           └── Is the device closed/asleep? ──► Use Mobile Push Notifications (APNs/FCM)
```

---

# 🔗 Related Notes in this Vault
- [[01_The Lifecycle of an HTTP Request_ From Frontend to Database and Back|01. The Lifecycle of an HTTP Request: Frontend to DB and Back]]
- [[API/01_API Design Paradigms_ REST, GraphQL, gRPC|01. API Design Paradigms (REST, GraphQL, gRPC)]]
- [[API/03_Idempotency_ Building Safe and Resilient APIs|03. Idempotency: Building Safe & Resilient APIs]]
- [[Application/00_Application Architecture Styles_ Decision Quadrant & Patterns|00. Application Architecture Styles: Decision Quadrant & Patterns]]
- [[Application/02_The 5 Hidden Architectural Realities Before Starting a Major Project|02. The 5 Hidden Architectural Realities]]
