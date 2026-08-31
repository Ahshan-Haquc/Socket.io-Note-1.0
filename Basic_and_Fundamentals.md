# Socket.IO Fundamentals — সম্পূর্ণ বাংলা গাইড (MERN Stack Real-Time Chat App প্রস্তুতি)

এই note-টি লেখা হয়েছে তাদের জন্য যারা ইতিমধ্যে JavaScript, React.js, Node.js, Express.js, REST API, JWT Authentication এবং MongoDB জানেন, কিন্তু Socket.IO দিয়ে **real-time communication** শুরু করতে চান। লক্ষ্য — এই note পড়ে শেষ করার পর আপনি একটি basic real-time chat application বানানোর জন্য প্রস্তুত থাকবেন।

---

## 📑 সূচিপত্র

1. [Socket.IO কী এবং কেন ব্যবহার করা হয়](#1-socketio-কী-এবং-কেন-ব্যবহার-করা-হয়)
2. [WebSocket কী এবং Socket.IO-এর সাথে সম্পর্ক](#2-websocket-কী-এবং-socketio-এর-সাথে-সম্পর্ক)
3. [HTTP/REST API বনাম Real-time Communication](#3-httprest-api-বনাম-real-time-communication)
4. [Socket.IO কীভাবে কাজ করে — Client ↔ Server Flow](#4-socketio-কীভাবে-কাজ-করে--client--server-flow)
5. [Socket.IO Install ও Basic Setup](#5-socketio-install-ও-basic-setup)
6. [Node.js + Express.js Server-এ Setup](#6-nodejs--expressjs-server-এ-setup)
7. [React Client-এ Setup](#7-react-client-এ-setup)
8. [`io`, `socket` এবং `socket.io-client` কী](#8-io-socket-এবং-socketio-client-কী)
9. [Connection এবং Disconnection](#9-connection-এবং-disconnection)
10. [Event এবং Event-Driven Communication](#10-event-এবং-event-driven-communication)
11. [`emit()` এবং `on()` বিস্তারিতভাবে](#11-emit-এবং-on-বিস্তারিতভাবে)
12. [Client → Server Communication](#12-client--server-communication)
13. [Server → Client Communication](#13-server--client-communication)
14. [Server → All Clients Broadcasting](#14-server--all-clients-broadcasting)
15. [`socket.broadcast.emit()`](#15-socketbroadcastemit)
16. [Acknowledgement / Callback](#16-acknowledgement--callback)
17. [Multiple Custom Events Organize করা](#17-multiple-custom-events-organize-করা)
18. [Socket ID কী](#18-socket-id-কী)
19. [Rooms কী এবং কেন দরকার](#19-rooms-কী-এবং-কেন-দরকার)
20. [Room-এ Join করা](#20-room-এ-join-করা)
21. [Specific Room-এ Message পাঠানো](#21-specific-room-এ-message-পাঠানো)
22. [Room থেকে Leave করা](#22-room-থেকে-leave-করা)
23. [Namespace কী](#23-namespace-কী)
24. [Basic Error Handling](#24-basic-error-handling)
25. [JWT Authentication Integration](#25-jwt-authentication-integration)
26. [Common Mistakes এবং Solution](#26-common-mistakes-এবং-solution)
27. [Project Folder Structure](#27-project-folder-structure)
28. [Socket.IO Cheat Sheet](#28-socketio-cheat-sheet)
29. [Practice Tasks](#29-practice-tasks)
30. [Final Project Exercise — Real-time Messaging Server](#30-final-project-exercise--real-time-messaging-server)

---

## 1. Socket.IO কী এবং কেন ব্যবহার করা হয়

**Socket.IO** একটি JavaScript library যা browser (client) এবং server-এর মধ্যে **real-time, bidirectional (দুই দিকেই), event-based communication** সম্ভব করে।

সাধারণ REST API-তে client সবসময় server-কে **request** পাঠায়, আর server **response** দেয়। Server নিজে থেকে client-কে কিছু "push" করতে পারে না। কিন্তু chat app, notification system, live tracking, collaborative editor — এসব জায়গায় server-কে *নিজে থেকেই* client-কে data পাঠাতে হয়, কোনো request ছাড়াই। এটাই Socket.IO সমাধান করে।

**কেন দরকার:**
- Real-time chat app (message instant delivery)
- Live notification (কেউ like/comment করলেই সাথে সাথে জানানো)
- Online/offline user status
- Live tracking (delivery app-এর মতো)
- Collaborative tools (Google Docs-এর মতো একসাথে editing)
- Live dashboard/analytics

Socket.IO শুধু একটা library নয়, এটার **client library** (`socket.io-client`) আর **server library** (`socket.io`) — দুটোই আছে, আর এরা নিজেদের মধ্যে একটা নির্দিষ্ট protocol মেনে কথা বলে।

---

## 2. WebSocket কী এবং Socket.IO-এর সাথে সম্পর্ক

**WebSocket** হলো একটা low-level communication protocol (`ws://` বা `wss://`) যেটা client আর server-এর মধ্যে একটা **persistent (স্থায়ী) connection** তৈরি করে। একবার connection হয়ে গেলে, দুই পক্ষই যেকোনো সময় একে অপরকে data পাঠাতে পারে — নতুন করে request করার দরকার হয় না।

**Socket.IO আসলে WebSocket-এর wrapper নয়, বরং একটা বড় library যা WebSocket ব্যবহার করে (যখন সম্ভব)।**

সম্পর্কটা এভাবে বোঝা যায়:

| WebSocket | Socket.IO |
|---|---|
| Raw protocol, নিজে থেকে reconnect করে না | Automatic reconnection আছে |
| Event system নেই, শুধু raw message পাঠায় | Built-in event system (`emit`, `on`) |
| Fallback নেই | পুরনো ব্রাউজার/network-এ HTTP long-polling-এ fallback করে |
| Room/Namespace-এর concept নেই | Room ও Namespace built-in |
| Browser compatibility নিজে সামলাতে হয় | সব browser-এ কাজ করার ব্যবস্থা আছে |

সহজ কথায়: **Socket.IO = WebSocket + অনেক extra সুবিধা (reconnection, fallback, rooms, acknowledgement ইত্যাদি) একসাথে প্যাকেজ করা।**

> ⚠️ Note: Socket.IO-এর client আর plain WebSocket client একে অপরের সাথে সরাসরি compatible না। মানে আপনি `socket.io-client` দিয়ে একটা raw WebSocket server-এ connect করতে পারবেন না, এবং উল্টোটাও না। দুই পাশেই Socket.IO থাকতে হবে।

---

## 3. HTTP/REST API বনাম Real-time Communication

| বিষয় | HTTP/REST API | Real-time (Socket.IO) |
|---|---|---|
| Communication direction | One-way (Client → Server → Response) | Two-way (যেকোনো সময় যেকোনো দিকে) |
| Connection | প্রতি request-এ নতুন connection (বা short-lived) | একবার connect হলে সেটা persistent থাকে |
| Server initiative | Server নিজে থেকে data পাঠাতে পারে না | Server যেকোনো সময় client-কে push করতে পারে |
| Use case | CRUD operations, data fetch, form submit | Chat, notification, live update, tracking |
| Overhead | প্রতি request-এ HTTP header ইত্যাদি overhead | Connection একবার হলে overhead কম |

**উদাহরণ দিয়ে বোঝা যাক:**

REST API দিয়ে chat বানালে client-কে বারবার poll করতে হতো — "নতুন message আছে কিনা?" প্রতি ২ সেকেন্ড পরপর জিজ্ঞেস করা (এটাকে বলে **polling**), যেটা অনেক inefficient এবং delay তৈরি করে।

Socket.IO দিয়ে server নিজে থেকেই বলে দেয় — "এই নাও নতুন message" — সাথে সাথে, কোনো delay ছাড়া।

> 💡 বাস্তব প্রজেক্টে REST API আর Socket.IO **একসাথেই** ব্যবহার হয়। যেমন: message history load করতে REST API (একবার fetch), কিন্তু নতুন message আসলে সেটা real-time-এ দেখাতে Socket.IO।

---

## 4. Socket.IO কীভাবে কাজ করে — Client ↔ Server Flow

Socket.IO-এর কাজের ধারা সহজভাবে এভাবে বোঝা যায়:

```
1. Client browser-এ socket.io-client দিয়ে server-এর সাথে connection request পাঠায়
2. Server (socket.io) সেই request accept করে একটা persistent connection তৈরি করে
3. এখন client আর server দুজনেই একে অপরকে "event" পাঠাতে (emit) এবং শুনতে (on/listen) পারে
4. Connection চলতে থাকে যতক্ষণ না browser বন্ধ হয়, network কাটে, বা কেউ disconnect করে
5. Connection কাটলে socket.io-client automatic ভাবে reconnect করার চেষ্টা করে
```

Flow diagram (text আকারে):

```
[React Client]                         [Node.js + Express Server]
     |                                            |
     |------ connect (handshake) -------------->  |
     |  <----------- connection established ----- |
     |                                            |
     |------ socket.emit("sendMessage") ------->  |  (client পাঠাচ্ছে)
     |                                            |  server তা receive করে,
     |                                            |  process করে
     |  <----- io.emit("receiveMessage") -------- |  (server broadcast করছে)
     |                                            |
     |------ socket.disconnect() -------------->  |
     |  <----------- "disconnect" event fired --- |
```

প্রতিটা connected client-এর জন্য server-এ একটা আলাদা **socket instance** তৈরি হয়, যার একটা unique `id` থাকে। এই socket instance দিয়েই server নির্দিষ্ট একজন client-কে, বা সবাইকে, বা একটা group-কে (room) message পাঠাতে পারে।

---

## 5. Socket.IO Install ও Basic Setup

দুই জায়গায় আলাদা package লাগবে — **server side** আর **client side**।

**Server side (Node.js/Express project-এ):**

```bash
npm install socket.io
```

**Client side (React project-এ):**

```bash
npm install socket.io-client
```

> ⚠️ খেয়াল রাখবেন — server-এ `socket.io` install হবে, client-এ `socket.io-client`। দুটো আলাদা package, নাম একটু মিলে যায় বলে অনেকে ভুল করে।

**Version compatibility:** Socket.IO-এর client আর server version মোটামুটি কাছাকাছি থাকা ভালো (major version মিলিয়ে রাখুন)। খুব পুরনো client নতুন server-এর সাথে অথবা উল্টোটা করলে connection সমস্যা হতে পারে।

---

## 6. Node.js + Express.js Server-এ Setup

Socket.IO সরাসরি Express app-এর সাথে attach হয় না — এটা **raw HTTP server**-এর সাথে attach হয়, আর Express সেই HTTP server-এর মধ্যেই চলে। তাই setup-টা একটু আলাদাভাবে করতে হয়:

```js
// server.js
const express = require("express");
const http = require("http");
const { Server } = require("socket.io");

const app = express();

// Express app দিয়ে raw HTTP server তৈরি করা হচ্ছে
const server = http.createServer(app);

// সেই HTTP server-এর সাথে Socket.IO attach করা হচ্ছে
const io = new Server(server, {
  cors: {
    origin: "http://localhost:5173", // React app-এর URL
    methods: ["GET", "POST"],
  },
});

// Client connect করলে এই callback চলবে
io.on("connection", (socket) => {
  console.log("একজন user connect করেছে, socket id:", socket.id);

  socket.on("disconnect", () => {
    console.log("User disconnect করেছে:", socket.id);
  });
});

// app.listen() না করে server.listen() করতে হবে
server.listen(5000, () => {
  console.log("Server চলছে port 5000-এ");
});
```

**কেন `app.listen()` না করে `server.listen()`?**
কারণ Socket.IO-কে raw HTTP server-এর reference দিতে হয়েছে (`http.createServer(app)`)। Express-এর `app.listen()` আসলে ভিতরে ভিতরে নিজেই একটা HTTP server তৈরি করে ফেলে, যেটার সাথে আমাদের `io` instance-টা attach নেই। তাই আমাদের নিজে থেকে HTTP server বানিয়ে সেটাকেই listen করাতে হয় — যাতে Express আর Socket.IO দুজনেই একই server share করে।

**CORS কেন লাগে?** React app সাধারণত আলাদা port-এ (যেমন 5173) চলে, আর backend আলাদা port-এ (যেমন 5000)। Browser security policy অনুযায়ী আলাদা origin থেকে connection করতে হলে CORS allow করে দিতে হয়, নাহলে browser connection block করে দেবে।

---

## 7. React Client-এ Setup

React-এ সাধারণত একটা centralized socket instance তৈরি করে সেটা পুরো app-এ ব্যবহার করা ভালো practice, যাতে বারবার নতুন connection তৈরি না হয়।

```js
// socket.js
import { io } from "socket.io-client";

const socket = io("http://localhost:5000", {
  autoConnect: false, // চাইলে manually connect করার জন্য
});

export default socket;
```

```jsx
// App.jsx
import { useEffect, useState } from "react";
import socket from "./socket";

function App() {
  const [connected, setConnected] = useState(false);

  useEffect(() => {
    socket.connect(); // component mount হলে connect

    socket.on("connect", () => {
      setConnected(true);
      console.log("Server-এর সাথে connected, id:", socket.id);
    });

    socket.on("disconnect", () => {
      setConnected(false);
    });

    // Cleanup — component unmount হলে listener remove করা জরুরি
    return () => {
      socket.off("connect");
      socket.off("disconnect");
      socket.disconnect();
    };
  }, []);

  return <div>{connected ? "Connected ✅" : "Disconnected ❌"}</div>;
}

export default App;
```

> 💡 `useEffect`-এর **cleanup function**-এ `socket.off()` করা খুব গুরুত্বপূর্ণ। React development mode-এ component দুইবার mount হয় (Strict Mode-এর কারণে), তাই cleanup না করলে duplicate event listener তৈরি হয়ে একই message বারবার handle হতে পারে — এটা একটা খুব কমন bug।

---

## 8. `io`, `socket` এবং `socket.io-client` কী

এই তিনটা term নিয়ে অনেকে confuse হয়। পার্থক্যটা পরিষ্কার করে নেওয়া যাক:

- **`io` (server-side):** এটা পুরো Socket.IO **server instance**। এটা দিয়ে আপনি সব connected client-কে একসাথে message পাঠাতে পারেন (broadcast), বা নতুন connection listen করতে পারেন। এক app-এ সাধারণত একটাই `io` instance থাকে।

- **`socket` (server-side, প্রতি connection-এর জন্য):** যখনই একজন client connect করে, `io.on("connection", (socket) => {...})`-এর ভিতরে যে `socket` object পাওয়া যায়, সেটা **সেই একজন specific client**-কে represent করে। এই `socket` object দিয়ে আপনি শুধু ওই একজন client-এর সাথে communicate করতে পারবেন।

- **`socket` (client-side):** React/browser-এ `socket.io-client`-এর `io()` function কল করে যে object পাওয়া যায়, সেটাও `socket` — এটা represent করে **client নিজে server-এর সাথে যে connection বানিয়েছে সেটা**।

- **`socket.io-client`:** এটা একটা npm package, যেটা browser/React app-এ install করে server-এর সাথে connect করার জন্য ব্যবহার হয়।

সহজভাবে মনে রাখুন:

```
Server side:  io  = পুরো server (সবার সাথে যোগাযোগের মাধ্যম)
              socket = একজন নির্দিষ্ট client-এর সাথে connection

Client side:  socket = client নিজের connection (server-এর সাথে)
```

---

## 9. Connection এবং Disconnection

**Server side:**

```js
io.on("connection", (socket) => {
  console.log("নতুন client connected:", socket.id);

  socket.on("disconnect", (reason) => {
    console.log(`Client ${socket.id} disconnect করেছে। কারণ: ${reason}`);
  });
});
```

`disconnect` event-এর একটা `reason` parameter থাকে যেটা বলে কেন disconnect হলো — যেমন `"io client disconnect"` (client নিজে disconnect করেছে), `"transport close"` (network issue), `"ping timeout"` ইত্যাদি।

**Client side:**

```js
socket.on("connect", () => {
  console.log("Server-এর সাথে সংযুক্ত হয়েছি!");
});

socket.on("disconnect", () => {
  console.log("Server থেকে সংযোগ বিচ্ছিন্ন হয়েছে।");
});

socket.on("connect_error", (err) => {
  console.log("Connection করতে সমস্যা হয়েছে:", err.message);
});
```

**ব্যবহারিক ক্ষেত্রে:** Chat app-এ `connection` event-এ user-কে online list-এ যোগ করা হয়, আর `disconnect` event-এ সেই user-কে offline দেখানো হয় — এটাই online/offline status দেখানোর মূল ভিত্তি।

---

## 10. Event এবং Event-Driven Communication

Socket.IO পুরোপুরি **event-driven** — মানে সবকিছুই ঘটে "event" পাঠানো আর "event" শোনার মাধ্যমে। কোনো নির্দিষ্ট URL/route নেই যেমন REST API-তে থাকে (`/api/users`), বরং আছে **event নাম** (যেমন `"sendMessage"`, `"userTyping"`)।

**Event-driven মানে কী?** কোনো কিছু "ঘটলে" (event fire হলে), যে সেটা শুনছে (listener) সে automatically react করে। যেমন — client একটা `"sendMessage"` event emit করল, server সেটা শুনছিল (`socket.on("sendMessage", ...)`), তাই server সাথে সাথে react করল।

Socket.IO-তে কিছু **built-in/reserved event** আছে (এগুলো নাম আপনি custom event-এর জন্য ব্যবহার করবেন না):

- `connect` — connection স্থাপিত হলে
- `disconnect` — connection বিচ্ছিন্ন হলে
- `connect_error` — connection করতে ব্যর্থ হলে
- `error` — কোনো error হলে

এছাড়া বাকি সব event নাম **আপনার নিজের তৈরি (custom)** — যেমন `"sendMessage"`, `"newUser"`, `"typing"`, `"joinRoom"` — আপনি যা খুশি নাম দিতে পারবেন, শুধু client আর server-এ নামটা মিলতে হবে।

---

## 11. `emit()` এবং `on()` বিস্তারিতভাবে

এই দুটোই Socket.IO-এর সবচেয়ে গুরুত্বপূর্ণ method — পুরো Socket.IO আসলে এই দুইয়ের উপরেই দাঁড়িয়ে।

- **`emit(eventName, data)`** — একটা event পাঠানো (trigger করা), সাথে data attach করে।
- **`on(eventName, callback)`** — একটা event শোনা (listen করা); সেই event এলে callback function চলবে।

```js
// পাঠানো (emit)
socket.emit("greeting", { message: "Hello Server!" });

// শোনা (on)
socket.on("greeting", (data) => {
  console.log(data.message); // "Hello Server!"
});
```

**গুরুত্বপূর্ণ নিয়ম:**
1. যে পাঠাচ্ছে সেটা `emit`, যে শুনছে সেটা `on` — **event নাম দুইপাশে একদম হুবহু মিলতে হবে** (case-sensitive)।
2. `emit`-এর সাথে যেকোনো ধরনের data পাঠানো যায় — string, number, object, array, এমনকি একাধিক argument।
3. `on` অবশ্যই `emit` করার **আগে** register করতে হবে (listener setup না থাকলে event miss হয়ে যাবে)।

```js
// একাধিক data পাঠানো
socket.emit("userJoined", "Ahsan", { room: "general" }, Date.now());

socket.on("userJoined", (name, roomInfo, timestamp) => {
  console.log(name, roomInfo, timestamp);
});
```

---

## 12. Client → Server Communication

Client থেকে server-কে data পাঠানোর জন্য client `emit` করে, server সেই event `on` দিয়ে শোনে।

**Client (React):**

```js
function sendMessage(text) {
  socket.emit("sendMessage", { text, sender: "Ahsan" });
}
```

**Server (Node.js):**

```js
io.on("connection", (socket) => {
  socket.on("sendMessage", (data) => {
    console.log("Message পাওয়া গেছে:", data.text, "from:", data.sender);
    // এখানে চাইলে DB-তে save করা যায়, বা অন্য client-দের কাছে পাঠানো যায়
  });
});
```

---

## 13. Server → Client Communication

Server যখন **একজন নির্দিষ্ট client**-কে message পাঠাতে চায়, তখন সেই client-এর `socket` object দিয়ে `emit` করে (broadcast না করে)।

```js
io.on("connection", (socket) => {
  // এই client শুধু connect করার সাথে সাথে welcome message পাবে,
  // অন্য কেউ এটা পাবে না
  socket.emit("welcome", { message: "স্বাগতম! আপনি এখন connected।" });
});
```

**Client side listener:**

```js
socket.on("welcome", (data) => {
  console.log(data.message);
});
```

এটা ব্যবহার হয় যখন কোনো information শুধু একজন specific ইউজারের জন্য প্রাসঙ্গিক — যেমন "আপনার নতুন private message আছে" notification।

---

## 14. Server → All Clients Broadcasting

সব connected client-কে (নিজেসহ) একসাথে message পাঠাতে চাইলে `io.emit()` ব্যবহার হয়।

```js
io.on("connection", (socket) => {
  socket.on("sendMessage", (data) => {
    // যে পাঠিয়েছে তাকেসহ সব client-এর কাছে message যাবে
    io.emit("receiveMessage", data);
  });
});
```

এটা সাধারণত group chat-এ ব্যবহার হয় যেখানে সবাই সব message দেখতে পাবে (নিজেরটাও UI-তে দেখানো লাগে বলে)।

---

## 15. `socket.broadcast.emit()`

কখনো কখনো আপনি চাইবেন **যে পাঠিয়েছে তাকে বাদ দিয়ে বাকি সবাইকে** পাঠাতে — সেই কাজে `socket.broadcast.emit()` ব্যবহার হয়।

```js
io.on("connection", (socket) => {
  socket.on("sendMessage", (data) => {
    // পাঠানো ব্যক্তি ছাড়া বাকি সবাই পাবে
    socket.broadcast.emit("receiveMessage", data);
  });
});
```

**`io.emit()` বনাম `socket.broadcast.emit()`:**

| Method | কে message পায় |
|---|---|
| `io.emit()` | connected সবাই, sender সহ |
| `socket.emit()` | শুধু ওই একজন specific client |
| `socket.broadcast.emit()` | sender বাদে বাকি সবাই |

**Practical use case:** ধরুন আপনার UI client side-এ নিজের পাঠানো message আলাদা style-এ (optimistic UI দিয়ে) সাথে সাথেই দেখিয়ে দেয়, তাহলে সেই একই message আবার server থেকে ফিরিয়ে পাঠানোর দরকার নেই — তাই এই ক্ষেত্রে `socket.broadcast.emit()` ব্যবহার করে শুধু অন্যদের পাঠানো ভালো।

---

## 16. Acknowledgement / Callback

সাধারণ `emit()` হলো "fire and forget" — পাঠিয়ে দিলাম, কিন্তু জানার উপায় নেই যে অন্য পাশ সেটা receive করেছে কিনা বা কী response দিলো। এই সমস্যা সমাধানের জন্য Socket.IO-তে **acknowledgement (callback)** সাপোর্ট আছে — অনেকটা REST API-র response-এর মতো।

**কখন ব্যবহার করবেন:** যখন client-এর জানা দরকার যে server তার পাঠানো event ঠিকভাবে process করেছে কিনা — যেমন message সফলভাবে DB-তে save হলো কিনা।

**Client side (callback পাঠানো):**

```js
socket.emit("sendMessage", { text: "Hi there" }, (response) => {
  // এই callback তখনই চলবে যখন server acknowledgement পাঠাবে
  if (response.success) {
    console.log("Message সফলভাবে পাঠানো হয়েছে!");
  } else {
    console.log("Error:", response.error);
  }
});
```

**Server side (callback কল করা):**

```js
socket.on("sendMessage", (data, callback) => {
  // এখানে চাইলে DB-তে save করার লজিক থাকবে
  console.log("Message পাওয়া গেছে:", data.text);

  // Server client-কে জানাচ্ছে যে কাজ সম্পন্ন হয়েছে
  callback({ success: true, receivedAt: Date.now() });
});
```

`emit()`-এর সর্বশেষ argument যদি একটা function হয়, Socket.IO automatically সেটাকে acknowledgement callback হিসেবে treat করে।

---

## 17. Multiple Custom Events Organize করা

একটা বাস্তব app-এ অনেকগুলো event থাকে (`sendMessage`, `typing`, `joinRoom`, `leaveRoom`, `userOnline` ইত্যাদি)। সব একসাথে একটা ফাইলে লিখলে code messy হয়ে যায়। তাই organize করার কিছু ভালো practice:

**১. Event নামগুলো constant হিসেবে রাখা (typo এড়াতে):**

```js
// events.js
module.exports = {
  SEND_MESSAGE: "sendMessage",
  RECEIVE_MESSAGE: "receiveMessage",
  JOIN_ROOM: "joinRoom",
  LEAVE_ROOM: "leaveRoom",
  TYPING: "typing",
  STOP_TYPING: "stopTyping",
};
```

```js
const { SEND_MESSAGE, RECEIVE_MESSAGE } = require("./events");

socket.on(SEND_MESSAGE, (data) => {
  io.emit(RECEIVE_MESSAGE, data);
});
```

**২. প্রতিটা connection handler-কে আলাদা function/file-এ ভাগ করা:**

```js
// socketHandlers/messageHandler.js
function messageHandler(io, socket) {
  socket.on("sendMessage", (data) => {
    io.emit("receiveMessage", data);
  });
}

module.exports = messageHandler;
```

```js
// socketHandlers/roomHandler.js
function roomHandler(io, socket) {
  socket.on("joinRoom", (roomId) => {
    socket.join(roomId);
  });

  socket.on("leaveRoom", (roomId) => {
    socket.leave(roomId);
  });
}

module.exports = roomHandler;
```

```js
// server.js
const messageHandler = require("./socketHandlers/messageHandler");
const roomHandler = require("./socketHandlers/roomHandler");

io.on("connection", (socket) => {
  messageHandler(io, socket);
  roomHandler(io, socket);
});
```

এভাবে প্রতিটা feature আলাদা module-এ থাকলে code readable আর maintainable থাকে — বড় প্রজেক্টে এটা খুবই দরকারি।

---

## 18. Socket ID কী

প্রতিটা client connect করলে Socket.IO তাকে একটা **unique string ID** দেয় (যেমন `"a1b2c3d4-..."`), যেটা `socket.id`-তে পাওয়া যায়। এই ID দিয়ে server নির্দিষ্ট করে বলতে পারে "এই client-টা কে"।

```js
io.on("connection", (socket) => {
  console.log("নতুন socket id:", socket.id);
});
```

**গুরুত্বপূর্ণ বিষয়:**
- এই ID **user-এর permanent identity না** — প্রতিবার নতুন connection (যেমন page reload) হলে নতুন ID তৈরি হয়।
- তাই কোনো user-কে permanently চিনতে চাইলে (যেমন "userId ৫ কোন socket-এ আছে") আপনাকে নিজে থেকে একটা mapping বানাতে হবে — `userId → socket.id` — সাধারণত JWT authentication-এর সাথে মিলিয়ে (২৫ নং section দেখুন)।

```js
// একটা সাধারণ mapping example
const onlineUsers = new Map(); // userId -> socket.id

io.on("connection", (socket) => {
  socket.on("registerUser", (userId) => {
    onlineUsers.set(userId, socket.id);
  });

  socket.on("disconnect", () => {
    // disconnect হলে mapping থেকে remove করা
    for (const [userId, sockId] of onlineUsers.entries()) {
      if (sockId === socket.id) onlineUsers.delete(userId);
    }
  });
});
```

---

## 19. Rooms কী এবং কেন দরকার

**Room** হলো একটা logical grouping — একদল socket-কে একসাথে একটা নামের ভিতরে রাখা, যাতে তাদের কাছে আলাদাভাবে message পাঠানো যায়, বাকি সবার কাছে না।

**কেন দরকার?** ধরুন আপনার chat app-এ একাধিক conversation চলছে — Ahsan আর Karim-এর মধ্যে একটা chat, আবার আলাদা একটা group chat। যদি সব message `io.emit()` দিয়ে সবাইকে পাঠানো হয়, তাহলে ভুল মানুষও message পেয়ে যাবে। **Room** দিয়ে এই সমস্যার সমাধান হয় — একটা নির্দিষ্ট conversation-এর সব সদস্যকে একটা room-এ রাখা হয়, আর message শুধু সেই room-এই পাঠানো হয়।

Room কোনো আলাদা তৈরি করার জিনিস না (যেমন database table না) — এটা শুধু Socket.IO-এর internal ভাবে socket-দের group করে রাখার একটা mechanism। একজন socket একসাথে একাধিক room-এ থাকতে পারে।

**বাস্তব ব্যবহার:**
- 1-to-1 private chat (২ জনের জন্য একটা unique room)
- Group chat / channel (একাধিক member-এর room)
- Specific project/document-এর collaborators

---

## 20. Room-এ Join করা

`socket.join(roomName)` দিয়ে একটা socket-কে room-এ যোগ করা হয়।

```js
io.on("connection", (socket) => {
  socket.on("joinRoom", (roomId) => {
    socket.join(roomId);
    console.log(`${socket.id} room "${roomId}"-এ join করেছে`);

    // ঐ room-এর বাকি member-দের জানানো যেতে পারে
    socket.to(roomId).emit("userJoinedRoom", { socketId: socket.id });
  });
});
```

**Client side:**

```js
socket.emit("joinRoom", "room123");
```

`roomId` যেকোনো string হতে পারে — সাধারণত এটা একটা conversation-এর MongoDB `_id`, অথবা দুইজন user-এর id একসাথে জোড়া লাগিয়ে বানানো unique string (যেমন 1-to-1 chat-এর জন্য `sortedUserIds.join("_")`)।

---

## 21. Specific Room-এ Message পাঠানো

Room-এর ভিতরের সবাইকে message পাঠাতে `io.to(roomId).emit()` অথবা `socket.to(roomId).emit()` ব্যবহার হয়।

```js
io.on("connection", (socket) => {
  socket.on("sendRoomMessage", ({ roomId, text, sender }) => {
    // roomId-তে থাকা সবাইকে message যাবে (sender সহ)
    io.to(roomId).emit("newRoomMessage", { text, sender });
  });
});
```

**`io.to()` বনাম `socket.to()`:**

| Method | কারা পায় |
|---|---|
| `io.to(roomId).emit()` | room-এর সবাই, sender সহ |
| `socket.to(roomId).emit()` | room-এর সবাই, sender **বাদে** |

```js
// শুধু room-এর অন্যদের পাঠানো (নিজে বাদে) — broadcast-এর মতোই কিন্তু room-scoped
socket.to(roomId).emit("newRoomMessage", { text, sender });
```

---

## 22. Room থেকে Leave করা

`socket.leave(roomName)` দিয়ে একটা socket room থেকে বের হয়ে যায়।

```js
io.on("connection", (socket) => {
  socket.on("leaveRoom", (roomId) => {
    socket.leave(roomId);
    console.log(`${socket.id} room "${roomId}" থেকে leave করেছে`);

    socket.to(roomId).emit("userLeftRoom", { socketId: socket.id });
  });

  // Client disconnect হলে সে automatically সব room থেকে বের হয়ে যায়,
  // এটা manually করার দরকার নেই
  socket.on("disconnect", () => {
    console.log(`${socket.id} disconnect করার সাথে সাথে সব room থেকে বের হয়ে গেছে`);
  });
});
```

> 💡 মনে রাখবেন: client disconnect (browser বন্ধ করলে বা tab close করলে) হলে Socket.IO automatically সেই socket-কে সব room থেকে বের করে দেয় — এর জন্য আলাদা কিছু করতে হয় না।

---

## 23. Namespace কী

**Namespace** হলো একটা আলাদা communication channel, একই server-এর ভিতরে — যেটা দিয়ে আপনি পুরো application-কে logical ভাবে ভাগ করতে পারেন। Default namespace হলো `/`।

**Room বনাম Namespace-এর পার্থক্য বোঝা জরুরি:**

| বিষয় | Namespace | Room |
|---|---|---|
| তৈরি হয় কীভাবে | Server code-এ define করে (`io.of("/chat")`) | Runtime-এ dynamically (`socket.join()`) |
| Scope | বড়, পুরো feature/module আলাদা করতে | ছোট, একটা group/conversation আলাদা করতে |
| Client-কে জানতে হয় | হ্যাঁ, connect করার সময়ই namespace বলে দিতে হয় | না, connect হওয়ার পর যেকোনো সময় join করা যায় |

**উদাহরণ:** ধরুন আপনার app-এ চ্যাট ফিচার আর নোটিফিকেশন ফিচার — দুটোর জন্যই Socket.IO দরকার, কিন্তু আপনি তাদের সম্পূর্ণ আলাদা রাখতে চান।

**Server side:**

```js
const chatNamespace = io.of("/chat");
const notificationNamespace = io.of("/notifications");

chatNamespace.on("connection", (socket) => {
  console.log("Chat namespace-এ connected:", socket.id);

  socket.on("sendMessage", (data) => {
    chatNamespace.emit("receiveMessage", data);
  });
});

notificationNamespace.on("connection", (socket) => {
  console.log("Notification namespace-এ connected:", socket.id);
});
```

**Client side:**

```js
import { io } from "socket.io-client";

const chatSocket = io("http://localhost:5000/chat");
const notifSocket = io("http://localhost:5000/notifications");
```

> 📌 Beginner হিসেবে basic chat app বানানোর সময় namespace-এর দরকার সাধারণত হয় না — এটা তখনই দরকার হয় যখন আপনার app বড় হয়ে একাধিক সম্পূর্ণ আলাদা real-time feature নিয়ে কাজ করে। শুরুতে শুধু concept-টা জেনে রাখাই যথেষ্ট।

---

## 24. Basic Error Handling

Socket.IO-তে error handle করার কয়েকটা গুরুত্বপূর্ণ জায়গা:

**১. Connection-level error (client side):**

```js
socket.on("connect_error", (err) => {
  console.log("Connection করতে ব্যর্থ:", err.message);
  // যেমন: সার্ভার down, অথবা auth token invalid
});
```

**২. নিজের তৈরি event-এর ভিতরে try-catch দিয়ে handle করা (server side):**

```js
socket.on("sendMessage", async (data, callback) => {
  try {
    if (!data.text || data.text.trim() === "") {
      throw new Error("Message empty হতে পারবে না");
    }

    // ধরুন এখানে DB save করার কাজ হচ্ছে
    // await Message.create(data);

    io.emit("receiveMessage", data);
    callback({ success: true });
  } catch (error) {
    console.log("sendMessage-এ error:", error.message);
    callback({ success: false, error: error.message });
  }
});
```

**৩. `error` event listen করা (custom errors emit করার জন্য):**

```js
// Server side — কোনো unauthorized action হলে
socket.emit("error", { message: "আপনি এই room-এ access করার অনুমতি নেই" });
```

```js
// Client side
socket.on("error", (err) => {
  console.log("Server থেকে error:", err.message);
});
```

**মূল নিয়ম:** REST API-তে যেমন `try-catch` আর proper status code দিয়ে error handle করেন, Socket.IO-তেও প্রতিটা event handler-এর ভিতরে সেরকমই `try-catch` ব্যবহার করা উচিত, যাতে একটা unexpected error পুরো server crash না করে ফেলে।

---

## 25. JWT Authentication Integration

যেহেতু আপনি ইতিমধ্যে JWT জানেন, এই অংশটা সহজেই বুঝবেন। মূল ধারণা: **connection স্থাপন হওয়ার আগেই** token verify করে ফেলা, যাতে unauthorized কেউ connect-ই করতে না পারে।

Socket.IO-এর **middleware** system দিয়ে এটা করা হয় — অনেকটা Express middleware-এর মতোই।

**Server side:**

```js
const jwt = require("jsonwebtoken");

io.use((socket, next) => {
  // Client handshake-এর সাথে token পাঠাবে
  const token = socket.handshake.auth?.token;

  if (!token) {
    return next(new Error("Authentication token পাওয়া যায়নি"));
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    socket.user = decoded; // পরবর্তী handler-এ socket.user দিয়ে user info পাওয়া যাবে
    next(); // সব ঠিক থাকলে connection allow
  } catch (error) {
    next(new Error("Invalid বা Expired token"));
  }
});

io.on("connection", (socket) => {
  console.log("Authenticated user connected:", socket.user.id);
});
```

**Client side (token পাঠানো handshake-এর সাথে):**

```js
import { io } from "socket.io-client";

const token = localStorage.getItem("accessToken"); // login করার পর যেভাবে save করেছেন

const socket = io("http://localhost:5000", {
  auth: {
    token: token,
  },
});
```

**Connection ব্যর্থ হলে client-এ handle করা:**

```js
socket.on("connect_error", (err) => {
  console.log("Authentication ব্যর্থ:", err.message);
  // চাইলে user-কে login page-এ redirect করা যায়
});
```

> 💡 এভাবে করলে `socket.user`-এ decoded JWT payload (যেমন `userId`) সবসময় available থাকবে, যেটা দিয়ে আপনি message পাঠানোর সময় sender কে যাচাই করতে পারবেন (client থেকে পাঠানো sender info-কে অন্ধভাবে বিশ্বাস না করে)।

---

## 26. Common Mistakes এবং তাদের Solution

| ভুল | সমস্যা | সমাধান |
|---|---|---|
| `app.listen()` ব্যবহার করা Socket.IO সহ project-এ | Socket.IO কাজ করবে না, কারণ `io` আলাদা HTTP server-এর সাথে attach করা | `http.createServer(app)` বানিয়ে `server.listen()` ব্যবহার করুন |
| `useEffect`-এ cleanup না করা | Component re-render/remount-এ duplicate listener তৈরি হয়, একই message বারবার আসে | `return () => { socket.off(...) }` দিয়ে cleanup করুন |
| প্রতি render-এ নতুন `io()` কল করা | বারবার নতুন connection তৈরি হয়ে যায় | একটা আলাদা `socket.js` ফাইলে single instance বানিয়ে export করুন |
| Event নামে typo (client আর server-এ ভিন্ন নাম) | Event কখনো পৌঁছায় না, কোনো error-ও দেখায় না | Event নামগুলো constant/shared file-এ রাখুন |
| CORS configure না করা | Browser connection block করে দেয় | Server-এ সঠিক `cors: { origin: ... }` সেট করুন |
| Client-থেকে পাঠানো sender/userId-কে অন্ধভাবে বিশ্বাস করা | যে কেউ নিজের নাম পরিবর্তন করে অন্য কারো নামে message পাঠাতে পারবে | JWT verify করে `socket.user`-এর তথ্য ব্যবহার করুন, client-এর পাঠানো data নয় |
| Room join করার আগে message পাঠানো | Message হারিয়ে যায়, কারণ ঐ socket তখনো room-এর সদস্য না | নিশ্চিত করুন join সফল হওয়ার পরই message পাঠানো হচ্ছে |
| `io.emit()` আর `socket.broadcast.emit()`-এর পার্থক্য না বোঝা | নিজের পাঠানো message নিজের কাছে দুইবার আসা, অথবা একদমই না আসা | কোনটা কখন লাগবে সেটা ভালোভাবে বুঝে ব্যবহার করুন (§14, §15 দেখুন) |
| Server restart হলে room membership হারিয়ে যাওয়া নিয়ে না ভাবা | Development-এ nodemon restart হলে সব client disconnect হয়ে যায় | Client side-এ reconnect হলে আবার room join করার logic রাখুন |

---

## 27. Project Folder Structure

একটা basic Socket.IO + Express + React chat project-এর সাধারণ folder structure এরকম হতে পারে:

```
chat-app/
│
├── server/                         # Backend (Node.js + Express)
│   ├── config/
│   │   └── db.js                   # MongoDB connection
│   │
│   ├── models/
│   │   ├── User.js
│   │   └── Message.js
│   │
│   ├── routes/
│   │   ├── authRoutes.js           # REST API — login/register
│   │   └── messageRoutes.js        # REST API — message history fetch
│   │
│   ├── middleware/
│   │   └── verifyJWT.js            # REST API-এর জন্য JWT middleware
│   │
│   ├── socket/
│   │   ├── index.js                # io.on("connection") মূল entry point
│   │   ├── authMiddleware.js       # Socket.IO-এর জন্য JWT middleware (io.use)
│   │   ├── messageHandler.js       # message সম্পর্কিত সব event
│   │   └── roomHandler.js          # room join/leave সম্পর্কিত event
│   │
│   ├── .env
│   ├── server.js                   # Main entry — Express + http server + io setup
│   └── package.json
│
└── client/                         # Frontend (React)
    ├── src/
    │   ├── socket.js                # Centralized socket.io-client instance
    │   ├── components/
    │   │   ├── ChatWindow.jsx
    │   │   ├── MessageInput.jsx
    │   │   └── OnlineUsers.jsx
    │   ├── pages/
    │   │   └── ChatPage.jsx
    │   ├── context/
    │   │   └── SocketContext.jsx    # socket instance app-জুড়ে share করার জন্য (optional)
    │   ├── App.jsx
    │   └── main.jsx
    ├── .env
    └── package.json
```

**মূল কথা:** REST API আর Socket.IO logic আলাদা folder-এ রাখুন (`routes/` বনাম `socket/`), যাতে code গোছানো থাকে এবং কোনটা কোন কাজ করছে সেটা সহজে বোঝা যায়।

---

## 28. Socket.IO Cheat Sheet

### Server Side

```js
const { Server } = require("socket.io");
const io = new Server(server, { cors: { origin: "..." } });

io.use((socket, next) => { ... next(); });     // middleware (auth)
io.on("connection", (socket) => { ... });      // নতুন connection

socket.id                                       // unique socket id
socket.emit(event, data);                       // শুধু এই client-কে পাঠানো
socket.on(event, callback);                      // event শোনা
socket.broadcast.emit(event, data);              // sender বাদে সবাইকে
io.emit(event, data);                            // sender সহ সবাইকে

socket.join(room);                               // room-এ join
socket.leave(room);                              // room থেকে leave
io.to(room).emit(event, data);                   // room-এর সবাইকে (sender সহ)
socket.to(room).emit(event, data);               // room-এর সবাইকে (sender বাদে)

io.of("/namespace");                             // namespace তৈরি

socket.on("disconnect", (reason) => { ... });    // disconnect হ্যান্ডল
```

### Client Side

```js
import { io } from "socket.io-client";

const socket = io("http://localhost:5000", {
  auth: { token: "..." },
});

socket.emit(event, data);                        // event পাঠানো
socket.emit(event, data, (response) => { ... }); // acknowledgement সহ

socket.on(event, callback);                       // event শোনা
socket.off(event);                                // listener remove করা (cleanup)

socket.on("connect", () => { ... });
socket.on("disconnect", () => { ... });
socket.on("connect_error", (err) => { ... });

socket.connect();                                 // manually connect
socket.disconnect();                              // manually disconnect
```

---

## 29. Practice Tasks

নিজে নিজে চর্চা করার জন্য এই কাজগুলো ধাপে ধাপে করুন:

1. **Basic Connection:** একটা Express server আর React app বানান যেখানে connect হলে console-এ "connected" আর disconnect হলে "disconnected" print হবে।
2. **Simple Echo:** Client থেকে একটা text পাঠান, server সেটা receive করে uppercase করে ফেরত পাঠাবে।
3. **Broadcast Chat:** সব client একসাথে message পাঠাতে পারবে এবং সবাই সব message দেখতে পাবে (কোনো room ছাড়াই)।
4. **Online User Count:** যতজন client connected আছে, সেই সংখ্যা সব client-এর কাছে real-time দেখান (কেউ যোগ/বিয়োগ হলেই আপডেট)।
5. **Typing Indicator:** একজন user টাইপ করলে অন্যদের "... typing" দেখান, টাইপ বন্ধ করলে সেটা সরিয়ে দিন (`typing` আর `stopTyping` event দিয়ে)।
6. **Room-based Chat:** ব্যবহারকারী একটা room name লিখে join করবে, শুধু সেই room-এর member-রাই একে অপরের message দেখবে।
7. **Acknowledgement:** Message পাঠানোর পর client-এ একটা ✔️ (delivered) চিহ্ন দেখান, যেটা server-এর callback আসলে দেখাবে।
8. **JWT-protected Socket:** শুধুমাত্র valid JWT token থাকা user-ই connect করতে পারবে — বাকিদের connection reject হবে।

---

## 30. Final Project Exercise — Real-time Messaging Server

এখন উপরের সব concept একসাথে করে একটা ছোট কিন্তু বাস্তবিক **real-time messaging server** বানানোর পালা। এটা এখনো পুরো chat app না (database integration, UI polish বাদ), বরং **core real-time engine**-টা ঠিকভাবে কাজ করানো লক্ষ্য।

### লক্ষ্য (Requirements)

আপনার Node.js + Express.js + Socket.IO দিয়ে এমন একটা server বানাতে হবে যেখানে:

1. Server শুরু হলে `http://localhost:5000`-এ Socket.IO চালু থাকবে।
2. একজন client connect করলে server console-এ তার `socket.id` print হবে।
3. Client `"joinRoom"` event দিয়ে একটা `roomId` পাঠিয়ে সেই room-এ join করবে।
4. Client join করলে সেই room-এর বাকি সবাই `"userJoined"` event পাবে (নিজে বাদে)।
5. Client `"sendMessage"` event দিয়ে `{ roomId, text, sender }` পাঠালে, শুধু সেই room-এর সবাই (sender সহ) `"newMessage"` event-এ সেটা পাবে।
6. Server acknowledgement callback দিয়ে client-কে জানাবে message সফলভাবে পাঠানো হয়েছে কিনা।
7. Client `"leaveRoom"` event দিয়ে room থেকে বের হতে পারবে, তখন room-এর বাকিরা `"userLeft"` event পাবে।
8. Client disconnect হলে console-এ সেটা log হবে।
9. সব event handler-এ basic `try-catch` error handling থাকবে (empty message allow করবে না)।

### শুরু করার জন্য কাঠামো (Starter Structure)

```js
// server.js
const express = require("express");
const http = require("http");
const { Server } = require("socket.io");

const app = express();
const server = http.createServer(app);
const io = new Server(server, {
  cors: { origin: "*" }, // development-এর জন্য, production-এ specific origin দেবেন
});

io.on("connection", (socket) => {
  console.log("Connected:", socket.id);

  socket.on("joinRoom", (roomId) => {
    // TODO: এখানে socket.join() করুন
    // TODO: room-এর বাকিদের "userJoined" জানান
  });

  socket.on("sendMessage", ({ roomId, text, sender }, callback) => {
    // TODO: validation করুন (text খালি কিনা)
    // TODO: শুধু ঐ room-এ "newMessage" পাঠান
    // TODO: callback দিয়ে success/failure জানান
  });

  socket.on("leaveRoom", (roomId) => {
    // TODO: socket.leave() করুন
    // TODO: room-এর বাকিদের "userLeft" জানান
  });

  socket.on("disconnect", () => {
    console.log("Disconnected:", socket.id);
  });
});

server.listen(5000, () => console.log("Server running on port 5000"));
```

### পরবর্তী ধাপ (এই exercise শেষ করার পর)

এই সার্ভারটা ঠিকভাবে কাজ করানোর পর, আপনি প্রস্তুত থাকবেন:

- এই একই server-এর সাথে একটা React frontend (§7-এ দেখানো পদ্ধতিতে) connect করতে।
- MongoDB-তে প্রতিটা message `Message` model হিসেবে save করতে (REST API দিয়ে history load, Socket.IO দিয়ে live update)।
- JWT authentication middleware (§25) যোগ করে শুধু logged-in user-দের access দিতে।
- এরপর ধীরে ধীরে online/offline status, read-receipts, একাধিক conversation — এসব ফিচার যোগ করতে।

---

**শুভকামনা! 🚀** এই note-এর প্রতিটা concept ভালোভাবে practice করলে আপনি একটা সম্পূর্ণ MERN Stack real-time chat application বানানোর জন্য প্রস্তুত হয়ে যাবেন।
