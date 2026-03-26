# SDS Toolbox: Real-Time Communication with WebSockets in Node.js

## Introduction

The Software Design School (SDS) Toolbox is a collection of guides and resources to help you get started with the essential tools and technologies used in modern software engineering.

In this guide, we will explore the fundamentals of **real-time, bidirectional communication** using WebSockets in a Node.js environment. By the end of this tutorial, you will understand how WebSockets differ from standard web traffic and how to build a real-time chat application from scratch.

![chat](/images/chat.png)

---

## HTTP vs. WebSockets

To understand why WebSockets are so powerful, we first need to understand the limitations of standard web traffic.

### The Limitation of HTTP (Half-Duplex)

Standard HTTP communication is **Half-Duplex**. Think of it like a walkie-talkie: only one side can speak at a time.

1. The client (your browser) sends a request to the server.
2. The server processes it and sends a response.
3. The connection is immediately closed.

The server cannot "push" data to the client unless the client asks for it first. If you want live updates using standard HTTP, the client has to constantly poll the server, which wastes a massive amount of server resources and bandwidth.

### The Power of WebSockets (Full-Duplex)

WebSockets are **Full-Duplex**. Think of it like a phone call:

* **Bidirectional:** Both the client and the server can send data to each other simultaneously.
* **Persistent:** Once the initial connection is made, the line stays open. This removes the heavy overhead of establishing a new connection and sending HTTP headers for every single message.
* **Real-time:** Because the line is always open, the server can instantly push data to the client the moment it becomes available.

### When to Use (and Not Use) WebSockets

**Perfect for:**

* **Real-time collaboration:** Shared documents (Google Docs), live whiteboards.
* **Live Dashboards:** Stock tickers, sports scores, live analytics.
* **Communication:** Chat apps, instant messaging.
* **Multiplayer Games:** Fast-paced interaction between players.

**Not ideal for:**

* **Standard Web Pages:** If you just need to load an article or a profile page once.
* **Simple API Calls:** Use standard REST APIs (GET/POST) for basic database operations.
* **Large File Transfers:** WebSockets are not optimized for heavy binary data like video streaming or large uploads; use standard HTTP streams instead.

---

## The Handshake

A WebSocket connection begins with an HTTP request containing an `Upgrade` header.

If the server supports it, it responds with a `101 Switching Protocols` status code.

From that exact moment, the HTTP protocol is abandoned. The connection stays open, and the two machines begin speaking a lightweight, binary/text WebSocket protocol over the same TCP connection.


![Upgrade HTTP to Websocket](/images/websocket_upgrade.png)
*Source: [Mohan Manavalan via Dev.to](https://dev.to/_mohanmurali/comparison-of-http-and-websocket-protocols-55pd)*

---

## Native WebSockets vs. Socket.IO

In the JavaScript ecosystem, you'll encounter two main ways to build real-time apps:

### 1. Native WebSockets

* **The Browser API:** Native support in all modern browsers (`new WebSocket()`). No frontend library is needed.

* **Node.js `ws` library:** A lightweight, bare-bones backend implementation that follows the official WebSocket protocol strictly. It is highly performant and great for learning the fundamentals.

### 2. Socket.IO

* A massively popular library that uses WebSockets but adds extra, convenient features on top, such as:

  * Automatic reconnection if the internet drops.
  * "Rooms" and "Namespaces" for easily grouping users (e.g., a specific chat room).
  * Fallback to HTTP polling if a strict corporate firewall blocks WebSockets.

> **Note:** A native browser WebSocket cannot connect directly to a Socket.IO server because Socket.IO adds its own custom language layer on top of the standard protocol. You must use the Socket.IO client library to talk to a Socket.IO server.

---

## Build a Simple Chat Application

This project is a monorepo consisting of a Node.js server and a React client. We use the native `ws` library on the backend and the browser's `WebSocket` API on the frontend.

### Prerequisites

- [Node.js LTS](https://nodejs.org/en/download)

### Initialize the WebSocket Server

On the server, we use the `ws` package. First, we create an HTTP server and then attach a `WebSocketServer` to it. This allows the server to handle both standard HTTP requests and WebSocket upgrades on the same port.

"server/server.js"

`TODO: [Websocket Server]`

```javascript
const server = http.createServer(app);
const wss = new WebSocketServer({ server });
```

### Step 2: Handle Connections and Broadcasting

The server's job is to listen for new connections and "broadcast" any received messages to every connected client.

"server/server.js"

`TODO [Function to broadcast]`

```javascript
const broadcastUserCount = () => {
  const payload = JSON.stringify({
      type: "user_count",
      count: wss.clients.size,
      timestamp: new Date().toISOString(),
    });

    wss.clients.forEach((client) => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(payload);
      }
    });
}
```

`TODO [On message]`
```javascript
wss.on("connection", (ws) => {
  // rest of the code...

  ws.on("message", (data) => {
    try {
      const message = JSON.parse(data);
      console.log("Received:", message);

      // Add server-side timestamp and broadcast the message
      const payload = JSON.stringify({
        ...message,
        timestamp: new Date().toISOString(),
      });

      wss.clients.forEach((client) => {
        if (client.readyState === WebSocket.OPEN) {
          client.send(payload);
        }
      });
    } catch (err) {
      console.error("Failed to parse message:", err);
    }
  });
})
```

`TODO [On close]`
```javascript
wss.on("connection", (ws) => {
  // rest of the code...

  ws.on("close", () => {
    console.log("User disconnected. Total:", wss.clients.size);
    // Broadcast the updated count on disconnect
    broadcastUserCount();
  });
})
```

### Step 3: Connect from the Frontend

In React, we manage the WebSocket lifecycle using a [custom hook](https://react.dev/learn/reusing-logic-with-custom-hooks).
<explain what a custom hook is>
We establish the connection when the user "joins" and clean it up when the component unmounts.

"client/src/hooks/useWebSocket.js"

`TODO [Set count and message]`
```javascript
useEffect(() => {
  // rest of the code...

  socket.onmessage = (event) => {
      const data = JSON.parse(event.data);

      // Handle different message types
      if (data.type === 'user_count') {
        setUserCount(data.count);
      } else {
        // Chat message
        setMessages((prev) => [...prev, data]);
      }
    };

  // rest of the code...

}, [url]);
```

---

### Running the App

1. **Install Dependencies:**
   Run the following in the root directory:
   ```bash
   npm install
   ```

2. **Configure Environment:**
   Create a `.env` file in the `server` directory:
   ```env
   PORT=8080
   NODE_ENV=development
   ```

3. **Run in Development Mode:**
   ```bash
   npm run dev
   ```

   *   **Client:** [http://localhost:5173](http://localhost:5173)
   *   **Server:** [ws://localhost:8080](ws://localhost:8080)

![Home](/images/home.png)
If you see home screen way to go!

Open simultaneous windows to mock users and explore!
Ensure to click leave room to cleanly terminate

Observe logs to see messages and user counts

### Clean up
<how to clean up resources>

---

## References

* **MDN Web Docs: The WebSocket API** - [developer.mozilla.org/en-US/docs/Web/API/WebSocket_API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket_API)
* **`ws` Library Repository** - [github.com/websockets/ws](https://github.com/websockets/ws)
* **RFC 6455: The WebSocket Protocol** - [datatracker.ietf.org/doc/html/rfc6455](https://datatracker.ietf.org/doc/html/rfc6455)
