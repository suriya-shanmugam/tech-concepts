SSE stands for **Server-Sent Events**.

It is a web standard that lets a **server push updates to the client over a single long-lived HTTP connection**.

The key idea:

* the **client opens** a normal HTTP request
* the **server keeps that response open**
* the server keeps sending new events down that same response stream
* the client receives them as they arrive

Unlike WebSocket, SSE is **one-way**:

* **server → client** only for the event stream
* if the client wants to send data, it still uses normal HTTP requests

## Mental model

* **HTTP**: one request, one response
* **SSE**: one request, one response stream that stays open
* **WebSocket**: one upgraded connection, both sides can send anytime

## How it works

The browser creates an `EventSource` connection:

```javascript
const es = new EventSource("/events");

es.onmessage = (event) => {
  console.log("new message:", event.data);
};
```

The browser sends a normal HTTP GET request to `/events`.

The server responds with:

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

Then it keeps sending lines in a special format like:

```text
data: hello

data: another update

```

Each event ends with a blank line.

The browser parses those events and fires callbacks.

## Event format

A server can send fields like these:

```text
event: priceUpdate
id: 42
data: {"symbol":"AAPL","price":212.44}

```

Meaning:

* `event:` optional custom event name
* `id:` event ID for reconnection
* `data:` actual payload
* blank line ends the event

On the client:

```javascript
const es = new EventSource("/events");

es.addEventListener("priceUpdate", (event) => {
  const data = JSON.parse(event.data);
  console.log(data.symbol, data.price);
});
```

## Why people use SSE

SSE is great when the server needs to continuously push updates to the browser, such as:

* live notifications
* news feeds
* stock tickers
* progress updates
* monitoring dashboards
* log streaming

## Why it is simpler than WebSocket

SSE is often easier because:

* it uses regular HTTP
* browsers support it directly with `EventSource`
* automatic reconnection is built in
* text-based event format is simple

## Automatic reconnection

One nice feature of SSE is that the browser usually reconnects automatically if the connection drops.

The server can also send a retry hint:

```text
retry: 5000
data: reconnect in 5 seconds if disconnected

```

And with `id:`, the browser can resume from the last received event by sending `Last-Event-ID`.

That helps with missed messages.

## Example server behavior

Imagine a progress endpoint:

Server sends:

```text
data: {"progress":10}

data: {"progress":45}

data: {"progress":80}

data: {"progress":100,"status":"done"}

```

The client updates a progress bar each time an event arrives.

## SSE vs WebSocket

### SSE

* one-way: server to client
* built on plain HTTP
* easy for streaming updates
* text only
* auto-reconnect built in

### WebSocket

* two-way: client and server
* starts as HTTP, then upgrades
* better for chat, games, collaboration
* can send text or binary
* you manage more of the connection behavior yourself

## SSE vs polling

With polling, the client keeps asking:

* “any updates?”
* “any updates now?”
* “what about now?”

With SSE, the client asks once, and the server pushes updates whenever they happen.

That reduces wasted requests and gives lower-latency updates.

## Limitations of SSE

* client cannot send messages on the same stream
* mostly text-based payloads
* some proxies/load balancers need proper config for long-lived connections
* not ideal for fully interactive bidirectional apps

## Simple end-to-end example

Client:

```javascript
const es = new EventSource("/events");

es.onmessage = (event) => {
  console.log("message:", event.data);
};

es.onerror = () => {
  console.log("connection issue, browser may reconnect automatically");
};
```

Server stream:

```text
data: connected

event: notification
data: {"title":"New message","count":3}

event: notification
data: {"title":"Build complete","status":"success"}

```

## Rule of thumb

Use **SSE** when:

* the server mainly pushes updates
* the client does not need a full duplex channel
* you want something simpler than WebSocket

Use **WebSocket** when:

* both sides need to send data frequently
* latency is critical
* you are building chat, games, or collaborative editing


