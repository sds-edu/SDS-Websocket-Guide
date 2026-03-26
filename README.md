# SDS Toolbox: Real-Time Communication with WebSockets

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

This project consiss of a Node.js server and a React client. We use the native `ws` library on the backend and the browser's `WebSocket` API on the frontend.

## Getting Started

Follow these steps to get your development environment up and running.

### Prerequisites

* **Node.js:** Ensure you have the [LTS version](https://nodejs.org/en/download) installed.
* **Starter Repository:** Clone the [SDS Kit Chat App](https://github.com/sds-edu/SDS-Kit-Chat-App) repository.

---

#### 1. Clone & Navigate

First, pull the code to your local machine and move into the project directory:

```bash
git clone https://github.com/sds-edu/SDS-Kit-Chat-App
cd SDS-Kit-Chat-App
```

#### 2. Install Dependencies

Install the required packages for the project.

```bash
npm install
```

#### 3. Configure Environment Variables

The server requires specific configuration to run. Create a `.env` file inside the `server` directory:

**Path:** `server/.env`

```env
PORT=8080
NODE_ENV=development
```

#### 4. Launch the App

Once configured, you can start the development server:

```bash
npm run dev
```

---

### Initialize the WebSocket Server

On the server, we use the `ws` package. First, we create an HTTP server and then attach a `WebSocketServer` to it. This allows the server to handle both standard HTTP requests and WebSocket upgrades on the same port.

In the repository, find the file `server/server.js`. Locate the `TODO: [Websocket Server]` block. Copy the code snippet below to initialize an HTTP and WebSocket server.

```javascript
const server = http.createServer(app);
const wss = new WebSocketServer({ server });
```

---

### Connections and Broadcasting

#### Update Number of Online Users

The server must listen for new connections and "broadcast" the updated number of online users to every connected client.

The `wss.clients` property is a Set containing all currently connected WebSocket instances. We iterate through this Set and send the user count only to clients whose connection is currently `OPEN`.

Locate the `TODO [Function to broadcast]` section in the file and copy the code below.

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

* `wss.clients.forEach((client) => { ... })`: Iterates through the server's internal set of all currently connected client instances.
* `client.readyState === WebSocket.OPEN`: Acts as a safety check to ensure the server only attempts to send data to clients with fully active, open connections (ignoring those in the process of closing).

#### Manage Incoming Text Messages

When a client sends a message, the server acts as a relay. It parses the incoming JSON, adds a server-side timestamp to ensure consistency, and broadcasts the enriched payload to everyone (including the sender).

Locate the `TODO [Message]` section in the file and copy the code below.

```javascript
wss.on("connection", (ws) => {
  // ...

  ws.on("message", (data) => {
    try {
      const message = JSON.parse(data);
      console.log("Received:", message);

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

* `wss.on("connection", (ws) => { ... })`: Listens for a new client connecting to the WebSocket server.
* `ws.on("message", (data) => { ... })`: Sets up a listener on the individual client's connection to capture any data they send to the server.
* `JSON.stringify({ ...message, timestamp: new Date().toISOString() })`: Takes the parsed message, injects a server-generated timestamp for consistency, and converts the whole object back into a string so it can be transmitted.

#### Handling Disconnections

It's critical to clean up when a user leaves. When the `close` event fires, we log the event and trigger another broadcast so all remaining users see the updated participant count.

Locate the `TODO [Disconnect]` section in the file and copy the code below.

```javascript
wss.on("connection", (ws) => {
  // ...

  ws.on("close", () => {
    console.log("User disconnected. Total:", wss.clients.size);

    // Broadcast the updated count on disconnect
    broadcastUserCount();
  });
})
```

* `ws.on("close", () => { ... })`: Sets up a listener that triggers automatically the moment this specific client disconnects.

---

### Connect from the Frontend

In React, to manage a persistent, external connection like a WebSocket we use React Hooks.

Hooks are special built-in JavaScript functions that allow you to extract complex, behind-the-scenes logic into modular, reusable functions, keeping your main UI components clean and focused entirely on rendering what the user sees.

In this application, we've created a custom hook called `useWebSocket`. It encapsulates all the complex logic of opening a connection, listening for incoming messages, and updating the chat state.

Inside our custom hook, we rely heavily on React's built-in `useEffect` hook. We use it to automatically establish the WebSocket connection the exact moment our chat component "mounts" (appears on the screen).

Find the file `client/src/hooks/useWebSocket.js`. Locate the `TODO [Set user count and message]` section and copy the code below.

```javascript
useEffect(() => {
  // ...

  socket.onmessage = (event) => {
    const data = JSON.parse(event.data);

    // Handle different message types
    if (data.type === 'user_count') {
      setUserCount(data.count);
    } else {
      // Chat message: append to the existing list
      setMessages((prev) => [...prev, data]);
    }
  };

  socket.onclose = () => {
      console.log('WebSocket Disconnected');
      setIsConnected(false);
      setUserCount(0);
    };

    socket.onerror = (error) => {
      console.error('WebSocket Error:', error);
    };

  // ...

}, [url, currentUsername]);
```

* `socket.onmessage = (event) => { ... }`: Sets up a listener that triggers automatically every time the client receives new data from the server.
* `if (data.type === 'user_count') { setUserCount(data.count); }`: Checks the payload to see if the server is sending a system update. If it's a user count update, it updates the React state to display the current number of participants.
* `setMessages((prev) => [...prev, data])`: It takes the previous list of messages and creates a new array with the incoming message appended to the end, triggering React to re-render the chat window.
* `socket.onclose = () => { ... }`: Listens for the connection severing - whether the server went down, the network dropped, or the connection was manually closed.
* `socket.onerror = (error) => { ... }`: Catches any unexpected connection failures and logs them to the browser's console.

---

### Running the App

Execute the following command in your terminal.

```bash
npm run dev
```

* Client: [http://localhost:5173](http://localhost:5173)
* Server: [ws://localhost:8080](ws://localhost:8080)

If you see the home screen, great job! You've successfully built a real-time server.

![Home](/images/home.png)

#### Observe

* Open simultaneous browser windows (or incognito tabs) to mock multiple users.
* Ensure you click "Leave Room" to cleanly terminate connections.
* Keep an eye on your terminal logs to watch the messages route in real-time and observe the user count fluctuate!

---

### Terminating

Properly closing connections is essential to prevent memory leaks and ensure the server has an accurate user count.

**Client-Side Cleanup:**
In our `useWebSocket` hook, the `useEffect` returns a cleanup function:

```javascript
return () => {
  socket.close();
};
```

When a user clicks "Leave" or closes the browser tab, the component unmounts, and `socket.close()` is called. This immediately notifies the server so it can update the participant list for everyone else.

The server is configured to handle "graceful shutdowns." If you stop the server, it will first notify all connected clients that the server is closing before shutting down the underlying HTTP listener.

```javascript
const shutdown = () => {
  wss.close(() => {
    server.close(() => {
      process.exit(0);
    });
  });
};

process.on("SIGINT", shutdown);
process.on("SIGTERM", shutdown);
```

Woohoo! 🥳 You have just implemented your first chat application.

---

## References

* [MDN Web Docs: The WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket_API)
* [ws Library Repository](https://github.com/websockets/ws)
* [RFC 6455: The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455)

## AI Declaration

Some parts of this code were structured and generated with the assistance of `Gemini 3.1 Pro` . All code snippets used were reviewed, implemented, and tested by the teaching team to ensure accuracy and functionality.
