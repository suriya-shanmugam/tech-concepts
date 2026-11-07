WebSocket is a way for a client and server to keep one **long-lived, two-way connection** open.

With normal HTTP, the client sends a request and the server sends back a response. Then that exchange is over. If the server wants to send new data later, the client usually has to ask again.

With WebSocket, after an initial HTTP-based setup, both sides can send messages to each other at any time over the same connection.

Here is the basic flow:

1. **Client starts with an HTTP request**
   The browser or app sends a special request saying it wants to upgrade the connection to WebSocket.

2. **Server agrees to upgrade**
   If the server supports it, it responds with an upgrade confirmation.

3. **Connection switches protocols**
   After that, it is no longer normal request-response HTTP traffic. It becomes a persistent WebSocket connection.

4. **Both sides exchange messages freely**
   The client can send messages whenever it wants.
   The server can also push messages whenever it wants.
   This is called **full-duplex** communication.

A simple mental model:

* HTTP is like mailing letters back and forth
* WebSocket is like staying on a phone call

Why it is useful:

* chat apps
* live notifications
* multiplayer games
* collaborative editors
* stock tickers
* dashboards with real-time updates

What happens under the hood:

* The connection usually begins as HTTP on the same port
* Then it performs a handshake using headers like `Upgrade: websocket`
* After the handshake, data is sent in small units called **frames**
* The connection stays open until one side closes it or the network drops

Benefits:

* low latency
* fewer repeated HTTP requests
* real-time server push
* efficient for frequent small updates

Things to watch out for:

* you need to manage reconnects
* long-lived connections consume server resources
* load balancers and proxies must support it
* authentication and authorization still matter

Tiny example:

Browser:

```javascript
const ws = new WebSocket("wss://example.com/socket");

ws.onopen = () => {
  ws.send("hello server");
};

ws.onmessage = (event) => {
  console.log("got:", event.data);
};
```

Server behavior:

* accept connection
* keep it open
* read incoming messages
* send responses or unsolicited updates

Also note:

* `ws://` is plain WebSocket
* `wss://` is secure WebSocket, like HTTPS

-----
WebSocket has two main parts:

1. the **handshake** that upgrades HTTP into WebSocket
2. the **message exchange pattern** after the connection is open

## 1) Handshake: how HTTP becomes WebSocket

A WebSocket connection starts as a normal HTTP request.

### Client request

The client sends an HTTP request like this:

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://example.com
```

### What those headers mean

* `Upgrade: websocket` says “switch this connection to WebSocket”
* `Connection: Upgrade` says the `Upgrade` header applies to this request
* `Sec-WebSocket-Key` is a random base64 value from the client
* `Sec-WebSocket-Version: 13` is the standard WebSocket version
* `Origin` helps the server check where the request came from

### Server response

If the server accepts, it replies:

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### Why `Sec-WebSocket-Accept` exists

The server takes the client’s `Sec-WebSocket-Key`, appends a fixed GUID, hashes it with SHA-1, then base64-encodes the result.

This proves the server understood the WebSocket upgrade instead of treating it like ordinary HTTP.

### After `101 Switching Protocols`

At that point, HTTP is done. The TCP connection stays open, and both sides start speaking the WebSocket protocol.

---

## 2) What communication looks like after the handshake

Once connected, communication is **full-duplex**:

* client can send anytime
* server can send anytime
* neither side has to wait for a request-response cycle

Data is sent in **frames**, usually representing text or binary messages.

Common frame types:

* **text frames** for strings like JSON
* **binary frames** for bytes, files, protobuf, etc.
* **ping**
* **pong**
* **close**

---

## 3) Common communication patterns

### A. Request-response over WebSocket

Even though WebSocket is not HTTP, you can still build request-response behavior on top of it.

Client sends:

```json
{ "id": 1, "action": "getProfile", "userId": 42 }
```

Server replies:

```json
{ "id": 1, "status": "ok", "data": { "name": "Ava", "role": "admin" } }
```

The `id` lets the client match the reply to the request.

Use this when:

* the client asks for something specific
* you want fewer HTTP calls
* you still want structured responses

---

### B. Server push / pub-sub

This is one of WebSocket’s biggest strengths.

Client subscribes:

```json
{ "action": "subscribe", "topic": "stock:AAPL" }
```

Server pushes updates whenever data changes:

```json
{ "topic": "stock:AAPL", "price": 212.44, "time": "2026-04-28T10:15:00Z" }
```

Use this for:

* stock prices
* live sports
* notifications
* dashboards

---

### C. Broadcast / fan-out

A server receives one event and sends it to many connected clients.

Example chat room:

User sends:

```json
{ "action": "sendMessage", "room": "general", "text": "hello everyone" }
```

Server broadcasts to all users in `general`:

```json
{ "room": "general", "from": "maya", "text": "hello everyone" }
```

Use this for:

* chat rooms
* multiplayer lobbies
* collaborative apps

---

### D. Streaming updates

Instead of sending one big response, the server sends partial data over time.

Example progress updates:

```json
{ "jobId": "abc123", "progress": 10 }
{ "jobId": "abc123", "progress": 35 }
{ "jobId": "abc123", "progress": 80 }
{ "jobId": "abc123", "progress": 100, "status": "done" }
```

Use this for:

* file processing
* AI generation progress
* long-running tasks

---

### E. Bidirectional event exchange

Both sides continuously send events, not just one side.

Example online game:

Client sends movement:

```json
{ "type": "move", "x": 120, "y": 80 }
```

Server sends world updates:

```json
{ "type": "state", "players": [{ "id": "p1", "x": 120, "y": 80 }] }
```

Use this for:

* games
* whiteboards
* collaborative editing

---

## 4) Message format patterns

WebSocket only gives you the pipe. You choose the message format.

### Simple event envelope

A common pattern is:

```json
{
  "type": "eventName",
  "payload": {}
}
```

Example:

```json
{
  "type": "chat.message",
  "payload": {
    "room": "general",
    "text": "hi"
  }
}
```

### Envelope with correlation ID

Useful for request-response and debugging:

```json
{
  "id": "req-42",
  "type": "getProfile",
  "payload": {
    "userId": 7
  }
}
```

Server:

```json
{
  "id": "req-42",
  "type": "getProfile.result",
  "payload": {
    "name": "Sam"
  }
}
```

### Event + error format

```json
{
  "type": "error",
  "code": "UNAUTHORIZED",
  "message": "Token expired"
}
```

---

## 5) Example: browser client

```javascript
const ws = new WebSocket("wss://example.com/ws");

ws.onopen = () => {
  console.log("connected");

  ws.send(JSON.stringify({
    id: "1",
    type: "subscribe",
    payload: { topic: "news" }
  }));
};

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  console.log("received:", msg);
};

ws.onclose = () => {
  console.log("connection closed");
};

ws.onerror = (err) => {
  console.error("socket error", err);
};
```

---

## 6) Example: Node.js server idea

Using a WebSocket library, the server usually does this:

```javascript
wss.on("connection", (socket) => {
  socket.on("message", (raw) => {
    const msg = JSON.parse(raw);

    if (msg.type === "subscribe") {
      socket.send(JSON.stringify({
        type: "subscribe.ok",
        payload: { topic: msg.payload.topic }
      }));
    }
  });

  socket.send(JSON.stringify({
    type: "welcome",
    payload: { message: "connected to server" }
  }));
});
```

---

## 7) Important protocol details

### Masking

Client-to-server frames are masked. Server-to-client frames are not.

This is a protocol rule that browsers follow automatically. You usually do not handle it manually unless you are implementing the protocol itself.

### Fragmentation

A large message can be split across multiple frames, then reassembled.

### Ping/pong

Used to keep connections alive and detect dead peers.

Example:

* server sends `ping`
* client answers `pong`

### Close handshake

One side sends a close frame, the other replies with a close frame, then the TCP connection closes.

---

## 8) Typical lifecycle in a real app

### Chat app example

1. Browser opens `wss://chat.example.com/ws`
2. HTTP handshake upgrades to WebSocket
3. Server authenticates the session or token
4. Client sends `joinRoom`
5. Server confirms
6. Users send messages
7. Server broadcasts messages to room members
8. Ping/pong keeps the connection healthy
9. On disconnect, client reconnects and re-subscribes

---

## 9) Reconnection pattern

WebSockets can drop due to:

* network changes
* idle timeouts
* server restarts
* proxy/load balancer issues

So clients often use reconnect logic:

```javascript
function connect() {
  const ws = new WebSocket("wss://example.com/ws");

  ws.onopen = () => console.log("connected");

  ws.onclose = () => {
    console.log("reconnecting...");
    setTimeout(connect, 2000);
  };
}

connect();
```

In production, add:

* exponential backoff
* auth token refresh
* re-subscribe after reconnect
* message replay or recovery if needed

---

## 10) WebSocket vs HTTP thinking

A useful mental model:

* **HTTP**: “give me this resource”
* **WebSocket**: “keep this pipe open and we’ll exchange events”

So WebSocket is best when:

* updates happen frequently
* latency matters
* the server needs to push data
* both sides need to speak continuously

---

## 11) Practical design tips

A solid message design often includes:

* `type` for routing
* `id` for matching replies
* `payload` for data
* `error` format for failures
* heartbeat handling
* reconnect strategy
* authentication at connect time
* authorization per action, not just per connection

Example robust message:

```json
{
  "id": "req-101",
  "type": "chat.send",
  "payload": {
    "roomId": "general",
    "text": "hello"
  }
}
```

Server response:

```json
{
  "id": "req-101",
  "type": "chat.send.ack",
  "payload": {
    "messageId": "m-555"
  }
}
```

Or error:

```json
{
  "id": "req-101",
  "type": "error",
  "error": {
    "code": "ROOM_NOT_FOUND",
    "message": "Room does not exist"
  }
}
```

## 12) One compact end-to-end example

### Handshake

Client asks to upgrade HTTP to WebSocket.

### After upgrade

Client sends:

```json
{ "id": "7", "type": "orderBook.subscribe", "payload": { "symbol": "BTC-USD" } }
```

Server replies:

```json
{ "id": "7", "type": "orderBook.subscribe.ok", "payload": { "symbol": "BTC-USD" } }
```

Then server pushes ongoing updates:

```json
{ "type": "orderBook.update", "payload": { "symbol": "BTC-USD", "bid": 62100, "ask": 62110 } }
```

That is the essence of WebSocket communication:

* one HTTP upgrade
* one persistent connection
* many bidirectional messages

