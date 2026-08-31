# Build a Basic MERN Chat Application with Socket.IO — সম্পূর্ণ বাংলা Study Note

এটা আপনার আগের **Socket.IO Fundamentals** note-এর continuation। এখন আমরা সেই concept-গুলো ব্যবহার করে একটা বাস্তব **MERN Stack (MongoDB, Express, React, Node.js) one-to-one real-time chat application** ধাপে ধাপে বানাব — সম্পূর্ণ runnable code সহ।

---

## 📑 সূচিপত্র

- [Project Architecture](#project-architecture)
- [সম্পূর্ণ Message Flow (খুব গুরুত্বপূর্ণ)](#সম্পূর্ণ-message-flow-খুব-গুরুত্বপূর্ণ)
1. [Chat Application-এর Requirements ও Features](#1-chat-application-এর-requirements-ও-features)
2. [Frontend এবং Backend-এর Responsibilities](#2-frontend-এবং-backend-এর-responsibilities)
3. [Database-এ User এবং Message Model Design](#3-database-এ-user-এবং-message-model-design)
4. [Message-এর জন্য প্রয়োজনীয় Fields](#4-message-এর-জন্য-প্রয়োজনীয়-fields)
5. [REST API এবং Socket.IO — কোন কাজ কোথায়](#5-rest-api-এবং-socketio--কোন-কাজ-কোথায়)
6. [Backend Project Setup](#6-backend-project-setup)
7. [Express + Socket.IO Server Setup](#7-express--socketio-server-setup)
8. [MongoDB Connection](#8-mongodb-connection)
9. [User Authentication-এর Basic Structure](#9-user-authentication-এর-basic-structure)
10. [Socket Connection-এর সময় Authenticated User Identify করা](#10-socket-connection-এর-সময়-authenticated-user-identify-করা)
11. [React Application-এ Socket.IO Client Setup](#11-react-application-এ-socketio-client-setup)
12. [Socket Connection Manage করা](#12-socket-connection-manage-করা)
13. [User-to-User Private Chat-এর Concept](#13-user-to-user-private-chat-এর-concept)
14. [একটি Conversation/Chat Room তৈরি করা](#14-একটি-conversationchat-room-তৈরি-করা)
15. [দুইজন User-এর জন্য Unique Room কীভাবে তৈরি করা যায়](#15-দুইজন-user-এর-জন্য-unique-room-কীভাবে-তৈরি-করা-যায়)
16. [User Room-এ Join করবে কীভাবে](#16-user-room-এ-join-করবে-কীভাবে)
17. [Message Send করার Complete Flow](#17-message-send-করার-complete-flow)
18. [Message Database-এ Save করা](#18-message-database-এ-save-করা)
19. [Socket.IO দিয়ে Receiver-এর কাছে Message পাঠানো](#19-socketio-দিয়ে-receiver-এর-কাছে-message-পাঠানো)
20. [React-এ Received Message UI-তে দেখানো](#20-react-এ-received-message-ui-তে-দেখানো)
21. [Existing Messages API থেকে Load করা](#21-existing-messages-api-থেকে-load-করা)
22. [নতুন Message এবং Old Messages একসাথে Manage করা](#22-নতুন-message-এবং-old-messages-একসাথে-manage-করা)
23. [React State কীভাবে Manage করব](#23-react-state-কীভাবে-manage-করব)
24. [Socket Event Listener Properly Setup ও Cleanup করা](#24-socket-event-listener-properly-setup-ও-cleanup-করা)
25. [Duplicate Message কেন হতে পারে এবং কীভাবে Prevent করব](#25-duplicate-message-কেন-হতে-পারে-এবং-কীভাবে-prevent-করব)
26. [Component Unmount হলে Socket Listener Cleanup](#26-component-unmount-হলে-socket-listener-cleanup)
27. [Basic Error Handling](#27-basic-error-handling)
28. [Basic Loading State](#28-basic-loading-state)
29. [Complete Project Folder Structure](#29-complete-project-folder-structure)
- [Backend Complete Code Flow](#backend-complete-code-flow)
- [Frontend Complete Code Flow](#frontend-complete-code-flow)
- [Socket Events Table](#socket-events-table)
- [REST API বনাম Socket.IO Responsibility Table](#rest-api-বনাম-socketio-responsibility-table)
- [Common Bugs এবং Debugging Techniques](#common-bugs-এবং-debugging-techniques)
- [Final Checklist](#final-checklist)
- [Practice Assignment](#practice-assignment)

---

## Project Architecture

```
   ┌───────────────────┐
   │  React Frontend   │
   │  (Vite/CRA app)   │
   └─────────┬─────────┘
             │
     ┌───────┴────────┐
     │                │
 REST API call   Socket.IO Client
 (axios, JWT)     (socket.io-client)
     │                │
     ▼                ▼
   ┌───────────────────────────┐
   │   Node.js + Express.js    │
   │   (একই HTTP server)        │
   │  ┌──────────────────────┐ │
   │  │   Express REST Routes │ │  → auth, user list, message history
   │  ├──────────────────────┤ │
   │  │   Socket.IO Server    │ │  → real-time message relay, room join
   │  └──────────────────────┘ │
   └─────────────┬──────────────┘
                 │  (Mongoose)
                 ▼
          ┌──────────────┐
          │   MongoDB    │
          │ Users, Msgs  │
          └──────────────┘
```

**মূল ধারণা:** React frontend দুইভাবে backend-এর সাথে কথা বলে —

1. **REST API (HTTP request/response)** — যেসব কাজ "একবার করে ফলাফল নেওয়া" ধরনের, যেমন login করা, user list আনা, পুরনো message history load করা।
2. **Socket.IO (persistent connection)** — যেসব কাজ "real-time, event-based" ধরনের, যেমন নতুন message পাঠানো/receive করা, room join করা।

দুটোই একই Node.js + Express server-এর ভিতরে চলে (একই HTTP server-এর সাথে Socket.IO attach করা থাকে — যেটা আগের note-এর §6-এ দেখানো হয়েছিল), আর দুটোই একই MongoDB database ব্যবহার করে।

---

## সম্পূর্ণ Message Flow (খুব গুরুত্বপূর্ণ)

পুরো chat application-এর হৃদয় হলো এই flow-টা। নিচে ধাপে ধাপে দেখানো হলো একটা message পাঠানো থেকে শুরু করে অন্য user-এর screen-এ দেখানো পর্যন্ত ঠিক কী কী ঘটে। প্রতিটা ধাপের সাথে কোন code file দায়ী সেটাও উল্লেখ করা আছে — নিচের section-গুলোতে প্রতিটা ধাপ বিস্তারিতভাবে দেখানো হবে।

```
১. User A ChatWindow-এ টাইপ করে "Send" চাপে
        │
        ▼  (React state → function call)
২. React-এর sendMessage() function চলে (src/hooks/useChatSocket.js)
        │
        ▼  (socket.emit)
৩. Socket.IO Client "sendMessage" event emit করে,
   সাথে { receiverId, text } data এবং একটা acknowledgement callback পাঠায়
        │
        ▼  (network — persistent WebSocket connection দিয়ে)
৪. Socket.IO Server-এ "sendMessage" event handler চলে (server/socket/index.js)
        │
        ▼
৫. Server validation করে (text খালি কিনা), তারপর roomId তৈরি করে
   (generateRoomId দিয়ে, দুইজনের userId sort করে বানানো)
        │
        ▼  (Mongoose)
৬. Message MongoDB-তে save হয় (Message.create())
        │
        ▼
৭. Server সেই roomId-তে io.to(roomId).emit("newMessage", payload) করে —
   অর্থাৎ ঐ room-এ join করা সবাইকে (User A এবং User B, দুজনকেই) পাঠায়
        │
        ├──────────────────────────────┐
        ▼                              ▼
৮ক. User A-এর browser-এ            ৮খ. User B-এর browser-এ
    "newMessage" event receive হয়      "newMessage" event receive হয়
        │                              │
        ▼                              ▼
৯. দুই পাশেই React-এর useChatSocket hook-এর
   handleNewMessage() চলে, message state-এ যোগ হয়
        │
        ▼
১০. ChatWindow.jsx re-render হয়, নতুন message UI-তে দেখা যায়
        │
        ▼
১১. Server acknowledgement callback-ও call করে, User A-এর UI-তে
    "message সফলভাবে পাঠানো হয়েছে" নিশ্চিত হয়
```

**একটা গুরুত্বপূর্ণ design সিদ্ধান্ত:** আমরা sender (User A)-কেও `io.to(roomId).emit()` দিয়ে message পাঠাচ্ছি, sender-কে বাদ দিয়ে (`socket.broadcast`) না। কারণ — sender যখন message পাঠায়, তখন তার নিজের UI-তে সেই message দেখানোর দরকার, আর যদি server-ই সেই message ফিরিয়ে দেয়, তাহলে client-এ আলাদা করে "নিজে পাঠানো message নিজে state-এ push করা" লজিকের দরকার নেই — এতে duplicate message হওয়ার ঝুঁকি অনেক কমে যায় (§25-এ বিস্তারিত)।

---

## 1. Chat Application-এর Requirements ও Features

এই application-টা **intentionally basic** রাখা হয়েছে — মূল focus হলো Socket.IO-এর core concept ঠিকভাবে বোঝা এবং implement করা, advanced feature-এ না গিয়ে।

### ✅ যা থাকবে (In Scope)

- User registration ও login (JWT-based)
- সব user-এর একটা list (নিজেকে বাদ দিয়ে)
- একজন user select করে তার সাথে one-to-one chat
- আগের message history load করা (REST API দিয়ে)
- নতুন message পাঠানো ও real-time receive করা (Socket.IO দিয়ে)
- Chat room concept (দুইজন user-এর জন্য একটা unique room)
- MongoDB-তে message persist করা (permanent save)
- Socket connection/disconnection ঠিকভাবে handle করা
- Basic error handling ও loading state

### ❌ যা ইচ্ছাকৃতভাবে বাদ দেওয়া হয়েছে (Out of Scope)

এগুলো পরবর্তীতে নিজে থেকে শেখার/যোগ করার জন্য রাখা হলো, কিন্তু এই note-এ থাকবে না যাতে core concept থেকে focus না সরে:

- Online/offline status
- Typing indicator
- Read/delivery receipt (✔️✔️)
- File/image sharing
- Voice/video calling
- Redis, horizontal scaling, clustering
- Microservices architecture
- Advanced push notifications

---

## 2. Frontend এবং Backend-এর Responsibilities

| Layer | দায়িত্ব |
|---|---|
| **React Frontend** | UI render করা, form handle করা, REST API call করা (login/register/user-list/history), Socket.IO client দিয়ে connect/emit/listen করা, local UI state manage করা |
| **Express REST API** | Authentication (register/login), user list serve করা, message history serve করা, JWT verify করা |
| **Socket.IO Server** | Real-time connection-এর জন্য JWT verify করা, room join manage করা, নতুন message receive করে DB-তে save করা এবং সংশ্লিষ্ট client-দের কাছে relay করা |
| **MongoDB** | User এবং Message data permanently store করা |

মূল নীতি: **যা "state" (স্থায়ী তথ্য) সেটা DB + REST API দিয়ে যায়, যা "event" (মুহূর্তের ঘটনা) সেটা Socket.IO দিয়ে যায়।**

---

## 3. Database-এ User এবং Message Model Design

**`server/models/User.js`**

```js
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema(
  {
    username: { type: String, required: true, unique: true, trim: true },
    email: { type: String, required: true, unique: true, trim: true, lowercase: true },
    password: { type: String, required: true }, // bcrypt দিয়ে hashed থাকবে
  },
  { timestamps: true }
);

module.exports = mongoose.model("User", userSchema);
```

**`server/models/Message.js`**

```js
const mongoose = require("mongoose");

const messageSchema = new mongoose.Schema(
  {
    roomId: { type: String, required: true, index: true },
    sender: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true },
    receiver: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true },
    text: { type: String, required: true, trim: true },
  },
  { timestamps: true }
);

module.exports = mongoose.model("Message", messageSchema);
```

`roomId`-তে `index: true` দেওয়া হয়েছে কারণ আমরা বারবার একটা নির্দিষ্ট `roomId` দিয়ে message query করব (history load করার সময়) — index থাকলে সেটা দ্রুত হয়।

---

## 4. Message-এর জন্য প্রয়োজনীয় Fields

| Field | Type | কেন দরকার |
|---|---|---|
| `roomId` | String | কোন conversation-এর message সেটা চিহ্নিত করা; history query-এর ভিত্তি |
| `sender` | ObjectId (ref User) | কে পাঠিয়েছে — UI-তে message alignment (নিজের/অন্যের) ঠিক করতে লাগে |
| `receiver` | ObjectId (ref User) | কাকে পাঠানো হয়েছে — permission/validation-এর জন্য দরকার হতে পারে |
| `text` | String | Message-এর actual content |
| `createdAt` | Date (timestamps থেকে auto) | Message-এর সময় দেখানো, ঠিক sequence-এ sort করা |

> 📌 এই basic version-এ আমরা `status` (sent/delivered/read) field রাখছি না, কারণ read-receipt feature আমাদের scope-এর বাইরে। পরবর্তীতে এই feature যোগ করতে চাইলে এখানে `status: { type: String, enum: ["sent", "delivered", "read"], default: "sent" }` যোগ করলেই যথেষ্ট হবে — schema design-টা future-এর জন্য extend-friendly রাখা হয়েছে।

---

## 5. REST API এবং Socket.IO — কোন কাজ কোথায়

| কাজ | মাধ্যম | কেন |
|---|---|---|
| Register / Login | REST API | একবারের request-response, persistent connection দরকার নেই |
| User list fetch করা | REST API | Page load-এ একবার আনলেই যথেষ্ট |
| পুরনো message history load | REST API | বড় ডেটা একসাথে fetch করা REST-এ সহজ ও standard |
| নতুন message পাঠানো/receive করা | Socket.IO | Real-time push দরকার, request-response model এখানে অচল |
| Room join | Socket.IO | এটা connection-এর একটা state, session-based |
| Connection/disconnection track করা | Socket.IO | এটা inherently persistent-connection-এর বিষয় |

(এই table-এর পূর্ণাঙ্গ, endpoint/event নাম-সহ ভার্সন একদম শেষে ["REST API বনাম Socket.IO Responsibility Table"](#rest-api-বনাম-socketio-responsibility-table) অংশে দেওয়া আছে।)

---

## 6. Backend Project Setup

```bash
mkdir server && cd server
npm init -y

npm install express mongoose socket.io cors dotenv jsonwebtoken bcryptjs
npm install --save-dev nodemon
```

`package.json`-এ একটা script যোগ করুন:

```json
"scripts": {
  "dev": "nodemon server.js",
  "start": "node server.js"
}
```

**`.env`**

```
PORT=5000
MONGO_URI=mongodb://localhost:27017/mern_chat
JWT_SECRET=your_super_secret_jwt_key
CLIENT_URL=http://localhost:5173
```

> ⚠️ `JWT_SECRET` একটাই রাখুন এবং REST API-এর token verify আর Socket.IO-এর token verify — দুই জায়গাতেই এই **একই** secret ব্যবহার করবেন। আলাদা secret ব্যবহার করলে REST-এ ঠিকভাবে login করা token দিয়েও Socket connect করার সময় "invalid token" error আসবে (এটা একটা কমন bug — নিচে debugging table-এও আছে)।

---

## 7. Express + Socket.IO Server Setup

**`server/server.js`**

```js
require("dotenv").config();
const express = require("express");
const http = require("http");
const cors = require("cors");
const { Server } = require("socket.io");

const connectDB = require("./config/db");
const authRoutes = require("./routes/authRoutes");
const userRoutes = require("./routes/userRoutes");
const messageRoutes = require("./routes/messageRoutes");
const initSocket = require("./socket");

connectDB();

const app = express();
app.use(cors({ origin: process.env.CLIENT_URL || "http://localhost:5173" }));
app.use(express.json());

// ---- REST API routes ----
app.use("/api/auth", authRoutes);
app.use("/api/users", userRoutes);
app.use("/api/messages", messageRoutes);

// ---- Socket.IO-এর জন্য raw HTTP server তৈরি (Express-এর app.listen() না করে) ----
const server = http.createServer(app);

const io = new Server(server, {
  cors: {
    origin: process.env.CLIENT_URL || "http://localhost:5173",
    methods: ["GET", "POST"],
  },
});

initSocket(io);

const PORT = process.env.PORT || 5000;
server.listen(PORT, () => console.log(`Server চলছে port ${PORT}-এ`));
```

লক্ষ্য করুন — একই `server` (raw HTTP server) দিয়ে Express app আর Socket.IO দুটোই চলছে, ঠিক আগের note-এর §6-এর নিয়ম মেনে।

---

## 8. MongoDB Connection

**`server/config/db.js`**

```js
const mongoose = require("mongoose");

async function connectDB() {
  try {
    await mongoose.connect(process.env.MONGO_URI);
    console.log("MongoDB connected");
  } catch (error) {
    console.error("MongoDB connection failed:", error.message);
    process.exit(1);
  }
}

module.exports = connectDB;
```

---

## 9. User Authentication-এর Basic Structure

**`server/middleware/verifyJWT.js`** (REST API রুট protect করার জন্য)

```js
const jwt = require("jsonwebtoken");

function verifyJWT(req, res, next) {
  const authHeader = req.headers.authorization;
  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    return res.status(401).json({ message: "Token পাওয়া যায়নি" });
  }

  const token = authHeader.split(" ")[1];

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded; // { id, username }
    next();
  } catch (error) {
    return res.status(401).json({ message: "Invalid বা Expired token" });
  }
}

module.exports = verifyJWT;
```

**`server/controllers/authController.js`**

```js
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");
const User = require("../models/User");

function generateToken(user) {
  return jwt.sign({ id: user._id, username: user.username }, process.env.JWT_SECRET, {
    expiresIn: "7d",
  });
}

exports.register = async (req, res) => {
  try {
    const { username, email, password } = req.body;

    if (!username || !email || !password) {
      return res.status(400).json({ message: "সব field পূরণ করুন" });
    }

    const existingUser = await User.findOne({ $or: [{ email }, { username }] });
    if (existingUser) {
      return res.status(400).json({ message: "এই email/username দিয়ে আগেই account আছে" });
    }

    const hashedPassword = await bcrypt.hash(password, 10);
    const user = await User.create({ username, email, password: hashedPassword });

    const token = generateToken(user);

    res.status(201).json({
      token,
      user: { id: user._id, username: user.username, email: user.email },
    });
  } catch (error) {
    res.status(500).json({ message: "Server error", error: error.message });
  }
};

exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;

    const user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ message: "ভুল email বা password" });
    }

    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(400).json({ message: "ভুল email বা password" });
    }

    const token = generateToken(user);

    res.json({
      token,
      user: { id: user._id, username: user.username, email: user.email },
    });
  } catch (error) {
    res.status(500).json({ message: "Server error", error: error.message });
  }
};
```

**`server/routes/authRoutes.js`**

```js
const router = require("express").Router();
const { register, login } = require("../controllers/authController");

router.post("/register", register);
router.post("/login", login);

module.exports = router;
```

**Bonus — User List API** (feature list-এ থাকা "User list" ফিচারের জন্য দরকার):

**`server/controllers/userController.js`**

```js
const User = require("../models/User");

exports.getAllUsers = async (req, res) => {
  try {
    // নিজেকে বাদ দিয়ে বাকি সব user-এর list
    const users = await User.find({ _id: { $ne: req.user.id } }).select("username email");
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: "Server error", error: error.message });
  }
};
```

**`server/routes/userRoutes.js`**

```js
const router = require("express").Router();
const verifyJWT = require("../middleware/verifyJWT");
const { getAllUsers } = require("../controllers/userController");

router.get("/", verifyJWT, getAllUsers);

module.exports = router;
```

---

## 10. Socket Connection-এর সময় Authenticated User Identify করা

REST API-তে যেমন `verifyJWT` middleware দিয়ে প্রতিটা request-এ user identify করি, Socket.IO-তেও একইভাবে **connection স্থাপনের আগেই** middleware (`io.use`) দিয়ে JWT verify করা হয় — যাতে unauthorized কেউ connect-ই করতে না পারে।

```js
// server/socket/index.js — এই অংশটুকু (io.use middleware)
const jwt = require("jsonwebtoken");

io.use((socket, next) => {
  const token = socket.handshake.auth?.token;
  if (!token) return next(new Error("Authentication token পাওয়া যায়নি"));

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    socket.user = decoded; // { id, username } — এখন থেকে socket.user দিয়ে user info পাওয়া যাবে
    next();
  } catch (error) {
    next(new Error("Invalid token"));
  }
});
```

এই middleware পাস হয়ে গেলে, connection-এর পুরো lifetime জুড়ে `socket.user` object-এ ঐ user-এর `id` আর `username` available থাকবে — এটাই আমরা পরের সব event handler-এ ব্যবহার করব (client থেকে পাঠানো কোনো "আমি কে" তথ্যকে বিশ্বাস না করে, কারণ সেটা spoof করা সম্ভব)।

---

## 11. React Application-এ Socket.IO Client Setup

প্রথমে REST API call-এর জন্য একটা axios instance বানাই, কারণ login/register/user-list — এসব REST দিয়েই হবে।

```bash
cd client
npm install axios react-router-dom socket.io-client
```

**`client/src/api/axios.js`**

```js
import axios from "axios";

const api = axios.create({
  baseURL: "http://localhost:5000/api",
});

// প্রতিটা request-এর সাথে automatically JWT token যোগ করা হচ্ছে
api.interceptors.request.use((config) => {
  const token = localStorage.getItem("token");
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export default api;
```

এবার Socket.IO client setup — কিন্তু এখানে একটা গুরুত্বপূর্ণ পার্থক্য আগের basic note থেকে: **socket connection তখনই তৈরি করা যাবে যখন আমাদের কাছে JWT token আছে** (মানে login করার পর)। তাই সরাসরি module-level-এ `io()` না ডেকে, একটা function দিয়ে on-demand তৈরি করব।

**`client/src/socket.js`**

```js
import { io } from "socket.io-client";

let socket = null;

export function initSocket(token) {
  socket = io("http://localhost:5000", {
    auth: { token },
    autoConnect: true,
  });
  return socket;
}

export function getSocket() {
  return socket;
}

export function disconnectSocket() {
  if (socket) {
    socket.disconnect();
    socket = null;
  }
}
```

`socket` কে module-level variable-এ রাখা হয়েছে, যাতে app-এর যেকোনো component `getSocket()` কল করে একই connection ব্যবহার করতে পারে — বারবার নতুন connection তৈরি না হয়।

---

## 12. Socket Connection Manage করা

Login সফল হলেই `initSocket(token)` কল করতে হবে, আর logout করলে `disconnectSocket()`। এই লজিকটা একটা centralized `AuthContext`-এ রাখা সবচেয়ে পরিষ্কার approach।

**`client/src/context/AuthContext.jsx`**

```jsx
import { createContext, useContext, useState } from "react";
import { initSocket, disconnectSocket } from "../socket";

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(() => {
    const saved = localStorage.getItem("user");
    return saved ? JSON.parse(saved) : null;
  });

  function login(userData, token) {
    localStorage.setItem("token", token);
    localStorage.setItem("user", JSON.stringify(userData));
    setUser(userData);
    initSocket(token); // login হওয়ার সাথে সাথে socket connect
  }

  function logout() {
    localStorage.removeItem("token");
    localStorage.removeItem("user");
    setUser(null);
    disconnectSocket(); // logout হওয়ার সাথে সাথে socket disconnect
  }

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  return useContext(AuthContext);
}
```

**`client/src/pages/LoginPage.jsx`**

```jsx
import { useState } from "react";
import api from "../api/axios";
import { useAuth } from "../context/AuthContext";
import { useNavigate } from "react-router-dom";

function LoginPage() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);
  const { login } = useAuth();
  const navigate = useNavigate();

  async function handleSubmit(e) {
    e.preventDefault();
    setError("");
    setLoading(true);
    try {
      const res = await api.post("/auth/login", { email, password });
      login(res.data.user, res.data.token);
      navigate("/chat");
    } catch (err) {
      setError(err.response?.data?.message || "Login ব্যর্থ হয়েছে");
    } finally {
      setLoading(false);
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <h2>Login</h2>
      {error && <p style={{ color: "red" }}>{error}</p>}
      <input type="email" placeholder="Email" value={email} onChange={(e) => setEmail(e.target.value)} required />
      <input type="password" placeholder="Password" value={password} onChange={(e) => setPassword(e.target.value)} required />
      <button type="submit" disabled={loading}>{loading ? "Loading..." : "Login"}</button>
    </form>
  );
}

export default LoginPage;
```

**`client/src/pages/RegisterPage.jsx`**

```jsx
import { useState } from "react";
import api from "../api/axios";
import { useAuth } from "../context/AuthContext";
import { useNavigate } from "react-router-dom";

function RegisterPage() {
  const [form, setForm] = useState({ username: "", email: "", password: "" });
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);
  const { login } = useAuth();
  const navigate = useNavigate();

  function handleChange(e) {
    setForm({ ...form, [e.target.name]: e.target.value });
  }

  async function handleSubmit(e) {
    e.preventDefault();
    setError("");
    setLoading(true);
    try {
      const res = await api.post("/auth/register", form);
      login(res.data.user, res.data.token);
      navigate("/chat");
    } catch (err) {
      setError(err.response?.data?.message || "Registration ব্যর্থ হয়েছে");
    } finally {
      setLoading(false);
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <h2>Register</h2>
      {error && <p style={{ color: "red" }}>{error}</p>}
      <input name="username" placeholder="Username" value={form.username} onChange={handleChange} required />
      <input name="email" type="email" placeholder="Email" value={form.email} onChange={handleChange} required />
      <input name="password" type="password" placeholder="Password" value={form.password} onChange={handleChange} required />
      <button type="submit" disabled={loading}>{loading ? "Loading..." : "Register"}</button>
    </form>
  );
}

export default RegisterPage;
```

**`client/src/App.jsx`** — routing setup

```jsx
import { BrowserRouter, Routes, Route, Navigate } from "react-router-dom";
import { AuthProvider, useAuth } from "./context/AuthContext";
import LoginPage from "./pages/LoginPage";
import RegisterPage from "./pages/RegisterPage";
import ChatPage from "./pages/ChatPage";

function PrivateRoute({ children }) {
  const { user } = useAuth();
  return user ? children : <Navigate to="/login" />;
}

function AppRoutes() {
  return (
    <Routes>
      <Route path="/login" element={<LoginPage />} />
      <Route path="/register" element={<RegisterPage />} />
      <Route
        path="/chat"
        element={
          <PrivateRoute>
            <ChatPage />
          </PrivateRoute>
        }
      />
      <Route path="*" element={<Navigate to="/chat" />} />
    </Routes>
  );
}

function App() {
  return (
    <AuthProvider>
      <BrowserRouter>
        <AppRoutes />
      </BrowserRouter>
    </AuthProvider>
  );
}

export default App;
```

---

## 13. User-to-User Private Chat-এর Concept

One-to-one chat মানে — কোনো message শুধুমাত্র দুইজন নির্দিষ্ট ব্যক্তির মধ্যেই আদান-প্রদান হবে, বাকি কেউ সেটা দেখতে পাবে না। এর জন্য প্রথম ধাপ হলো — user একজন **নির্দিষ্ট ব্যক্তি select করবে**, যার সাথে সে chat করতে চায়।

**`client/src/components/UserList.jsx`**

```jsx
import { useEffect, useState } from "react";
import api from "../api/axios";

function UserList({ selectedUser, onSelectUser }) {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    async function fetchUsers() {
      try {
        const res = await api.get("/users");
        setUsers(res.data);
      } catch (error) {
        console.log("User list load করতে সমস্যা:", error.message);
      } finally {
        setLoading(false);
      }
    }
    fetchUsers();
  }, []);

  if (loading) return <p>User list loading...</p>;

  return (
    <ul>
      {users.map((u) => (
        <li
          key={u._id}
          onClick={() => onSelectUser(u)}
          style={{ fontWeight: selectedUser?._id === u._id ? "bold" : "normal", cursor: "pointer" }}
        >
          {u.username}
        </li>
      ))}
    </ul>
  );
}

export default UserList;
```

একজন user select হলে, সেটাই হবে "কার সাথে chat করছি" — এই select করা user-এর `_id` দিয়েই পরের ধাপে room তৈরি হবে।

---

## 14. একটি Conversation/Chat Room তৈরি করা

গুরুত্বপূর্ণ একটা বিষয় বোঝা দরকার: **আমরা room-কে আলাদা করে MongoDB-তে কোনো collection/model হিসেবে store করছি না।** Room এখানে শুধুই Socket.IO-এর একটা runtime concept (আগের note-এর §19 মনে করুন) — দুইজন user connect করলে তারা একটা নির্দিষ্ট নামের room-এ join করে, আর সেই নামটা **প্রতিবারই একই নিয়মে হিসাব করে বের করা হয়**, কোথাও save করার দরকার হয় না।

Message-এ যে `roomId` field আছে (§3-৪ দেখুন), সেটাই আসলে conversation-এর identifier হিসেবে কাজ করে — নতুন কোনো "Conversation" model বানানোর দরকার নেই এই basic version-এ।

---

## 15. দুইজন User-এর জন্য Unique Room কীভাবে তৈরি করা যায়

দুইজন user-এর জন্য একটা **deterministic** (মানে সবসময় একই ফলাফল দেয় এমন) roomId দরকার — যাতে User A থেকে User B-কে message পাঠানো হোক বা User B থেকে User A-কে, দুই ক্ষেত্রেই একই roomId তৈরি হয়।

**`server/utils/generateRoomId.js`**

```js
function generateRoomId(userId1, userId2) {
  // দুইজন user-এর id sort করে জোড়া লাগানো হচ্ছে,
  // যাতে (A, B) আর (B, A) — দুই ক্রমেই একই roomId পাওয়া যায়
  return [userId1.toString(), userId2.toString()].sort().join("_");
}

module.exports = generateRoomId;
```

**কেন sort করা জরুরি?** ধরুন sort না করলে User A → B পাঠালে roomId হতো `"A_B"`, আর User B → A পাঠালে হতো `"B_A"` — এই দুটো **আলাদা string**, ফলে দুইজন আসলে দুইটা আলাদা room-এ থাকত এবং একে অপরের message পেত না। Sort করার ফলে দুই ক্ষেত্রেই সবসময় একই string তৈরি হয়।

> 🔒 **নিরাপত্তা টিপ:** এই `roomId` সবসময় **server নিজে হিসাব করবে** (sender আর receiver-এর id দিয়ে), client থেকে সরাসরি roomId পাঠানো বিশ্বাস করা ঠিক না — নাহলে কেউ ইচ্ছাকৃতভাবে অন্য কারো roomId পাঠিয়ে অনধিকার প্রবেশ করতে পারত।

---

## 16. User Room-এ Join করবে কীভাবে

একজন user যখন কারো সাথে chat window খোলে, client `"joinRoom"` event emit করে (শুধু receiver-এর `userId` পাঠায়, roomId নিজে হিসাব করে পাঠায় না — সেটা server করবে):

**Client side (React hook-এর ভিতরের অংশ, পুরোটা §23-এ দেখুন):**

```js
socket.emit("joinRoom", { otherUserId: otherUser._id });
```

**Server side:**

```js
// server/socket/index.js — এই অংশটুকু
socket.on("joinRoom", ({ otherUserId }) => {
  const roomId = generateRoomId(socket.user.id, otherUserId);
  socket.join(roomId);
  console.log(`${socket.user.username} room "${roomId}"-এ join করেছে`);
});
```

দুইজন user-ই (A এবং B) যখন একে অপরের সাথে chat window খুলবে, তখন দুইজনই একই roomId-তে join করবে — কারণ `generateRoomId` উভয় দিক থেকেই একই ফলাফল দেয়।

---

## 17. Message Send করার Complete Flow

এখন সম্পূর্ণ `socket/index.js` ফাইলটা দেখা যাক — এতে auth middleware, `joinRoom`, `sendMessage`, এবং `disconnect` — সবগুলো handler একসাথে আছে।

**`server/socket/index.js`**

```js
const jwt = require("jsonwebtoken");
const Message = require("../models/Message");
const generateRoomId = require("../utils/generateRoomId");

function initSocket(io) {
  // Connection স্থাপনের আগেই JWT verify করা হচ্ছে (§10)
  io.use((socket, next) => {
    const token = socket.handshake.auth?.token;
    if (!token) return next(new Error("Authentication token পাওয়া যায়নি"));

    try {
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      socket.user = decoded; // { id, username }
      next();
    } catch (error) {
      next(new Error("Invalid token"));
    }
  });

  io.on("connection", (socket) => {
    console.log(`${socket.user.username} connected (socket: ${socket.id})`);

    // ---- Room join (§16) ----
    socket.on("joinRoom", ({ otherUserId }) => {
      const roomId = generateRoomId(socket.user.id, otherUserId);
      socket.join(roomId);
      console.log(`${socket.user.username} room "${roomId}"-এ join করেছে`);
    });

    // ---- Message পাঠানো (§18-১৯) ----
    socket.on("sendMessage", async ({ receiverId, text }, callback) => {
      try {
        if (!text || text.trim() === "") {
          throw new Error("Message খালি হতে পারবে না");
        }

        const roomId = generateRoomId(socket.user.id, receiverId);

        // ১. Database-এ save করা
        const message = await Message.create({
          roomId,
          sender: socket.user.id,
          receiver: receiverId,
          text: text.trim(),
        });

        const payload = {
          _id: message._id,
          roomId,
          sender: socket.user.id,
          receiver: receiverId,
          text: message.text,
          createdAt: message.createdAt,
        };

        // ২. ঐ room-এ থাকা সবাইকে (sender + receiver) পাঠানো
        io.to(roomId).emit("newMessage", payload);

        // ৩. Sender-কে acknowledgement callback দিয়ে জানানো
        callback({ success: true, message: payload });
      } catch (error) {
        callback({ success: false, error: error.message });
      }
    });

    socket.on("disconnect", () => {
      console.log(`${socket.user.username} disconnected`);
    });
  });
}

module.exports = initSocket;
```

এটাই §"সম্পূর্ণ Message Flow" section-এ দেখানো ধাপ ৩ থেকে ৭ পর্যন্তের backend implementation।

---

## 18. Message Database-এ Save করা

উপরের কোডে এই অংশটাই DB save করছে:

```js
const message = await Message.create({
  roomId,
  sender: socket.user.id,
  receiver: receiverId,
  text: text.trim(),
});
```

খেয়াল করুন — `sender` হিসেবে client-এর পাঠানো কোনো ডেটা না নিয়ে, `socket.user.id` (যেটা JWT থেকে verify করে পাওয়া, §10) ব্যবহার করা হচ্ছে। এতে কেউ অন্য কারো নাম ব্যবহার করে message পাঠাতে পারবে না।

---

## 19. Socket.IO দিয়ে Receiver-এর কাছে Message পাঠানো

Save হওয়ার পরপরই এই লাইনটা message-টাকে real-time পৌঁছে দেয়:

```js
io.to(roomId).emit("newMessage", payload);
```

`roomId`-তে যেহেতু sender আর receiver — দুজনেই join করা থাকে (যদি দুজনেই chat window খোলা রাখে), তাই **দুজনেই** এই `"newMessage"` event পাবে। Receiver যদি ওই মুহূর্তে chat window বন্ধ রাখে (তাই room-এ join করা নেই), সে event পাবে না — কিন্তু message DB-তে save হয়েই আছে, তাই সে পরে chat window খুললে REST API দিয়ে history load করার সময় (§21) সেটা দেখতে পাবে।

---

## 20. React-এ Received Message UI-তে দেখানো

**`client/src/components/ChatWindow.jsx`**

```jsx
import { useRef, useEffect } from "react";
import useChatSocket from "../hooks/useChatSocket";

function ChatWindow({ selectedUser, currentUser }) {
  const { messages, loading, error, sendMessage } = useChatSocket(selectedUser, currentUser.id);
  const bottomRef = useRef(null);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages]);

  if (!selectedUser) return <p>চ্যাট শুরু করতে বাম পাশ থেকে একজন user select করুন</p>;

  return (
    <div>
      <h3>{selectedUser.username}-এর সাথে চ্যাট</h3>

      {loading && <p>Messages loading...</p>}
      {error && <p style={{ color: "red" }}>{error}</p>}

      <div style={{ height: "400px", overflowY: "auto", border: "1px solid #ccc", padding: "8px" }}>
        {messages.map((msg) => (
          <div
            key={msg._id}
            style={{ textAlign: msg.sender === currentUser.id ? "right" : "left", margin: "4px 0" }}
          >
            <span style={{ background: "#eee", padding: "4px 8px", borderRadius: "8px", display: "inline-block" }}>
              {msg.text}
            </span>
          </div>
        ))}
        <div ref={bottomRef} />
      </div>

      <MessageForm onSend={sendMessage} />
    </div>
  );
}

function MessageForm({ onSend }) {
  const inputRef = useRef(null);

  function handleSubmit(e) {
    e.preventDefault();
    const text = inputRef.current.value;
    if (!text.trim()) return;
    onSend(text);
    inputRef.current.value = "";
  }

  return (
    <form onSubmit={handleSubmit}>
      <input ref={inputRef} type="text" placeholder="Message লিখুন..." />
      <button type="submit">Send</button>
    </form>
  );
}

export default ChatWindow;
```

`messages` array-টা `useChatSocket` hook থেকে আসছে (পুরো hook-টা §23-এ দেখুন) — সেটাই `.map()` করে প্রতিটা message bubble হিসেবে দেখানো হচ্ছে। নিজের পাঠানো message ডান পাশে, অন্যেরটা বাম পাশে — এটা `msg.sender === currentUser.id` দিয়ে ঠিক করা হচ্ছে।

**`client/src/pages/ChatPage.jsx`**

```jsx
import { useState } from "react";
import UserList from "../components/UserList";
import ChatWindow from "../components/ChatWindow";
import { useAuth } from "../context/AuthContext";

function ChatPage() {
  const [selectedUser, setSelectedUser] = useState(null);
  const { user } = useAuth();

  return (
    <div style={{ display: "flex", gap: "16px" }}>
      <div style={{ width: "200px" }}>
        <h3>Users</h3>
        <UserList selectedUser={selectedUser} onSelectUser={setSelectedUser} />
      </div>
      <div style={{ flex: 1 }}>
        <ChatWindow selectedUser={selectedUser} currentUser={user} />
      </div>
    </div>
  );
}

export default ChatPage;
```

---

## 21. Existing Messages API থেকে Load করা

Chat window খোলার সাথে সাথে আগের সব message দেখানো দরকার — এটা REST API দিয়ে হয় (Socket.IO দিয়ে না, কারণ এটা "একবার fetch করা" ধরনের কাজ, §5 মনে করুন)।

**`server/controllers/messageController.js`**

```js
const Message = require("../models/Message");
const generateRoomId = require("../utils/generateRoomId");

exports.getMessages = async (req, res) => {
  try {
    const myId = req.user.id;
    const otherUserId = req.params.userId;

    const roomId = generateRoomId(myId, otherUserId);

    const messages = await Message.find({ roomId }).sort({ createdAt: 1 });

    res.json({ roomId, messages });
  } catch (error) {
    res.status(500).json({ message: "Server error", error: error.message });
  }
};
```

**`server/routes/messageRoutes.js`**

```js
const router = require("express").Router();
const verifyJWT = require("../middleware/verifyJWT");
const { getMessages } = require("../controllers/messageController");

router.get("/:userId", verifyJWT, getMessages);

module.exports = router;
```

`.sort({ createdAt: 1 })` — পুরনো থেকে নতুন ক্রমে sort করা হচ্ছে (`1` = ascending), যাতে chat UI-তে উপর থেকে নিচে ঠিক ক্রমে message দেখা যায়।

---

## 22. নতুন Message এবং Old Messages একসাথে Manage করা

আমাদের দুই ধরনের message আছে —

1. **পুরনো message** — component mount হওয়ার সাথে সাথে **একবার** REST API দিয়ে load হয়।
2. **নতুন message** — Socket.IO-এর `"newMessage"` event দিয়ে **যখনই আসে তখনই** যোগ হয়।

দুটোকেই একই React state array-তে রাখতে হবে, যাতে UI একটাই ধারাবাহিক list দেখাতে পারে। Strategy: প্রথমে REST দিয়ে initial list set করা, তারপর socket থেকে আসা প্রতিটা নতুন message সেই array-এর শেষে **append** করা (§23-এর hook-এ এটাই হচ্ছে)।

---

## 23. React State কীভাবে Manage করব

সব logic — history load, socket listen, duplicate prevention, send — একটা custom hook-এ একসাথে রাখা হয়েছে, যাতে `ChatWindow` component পরিষ্কার থাকে এবং logic পুনর্ব্যবহারযোগ্য হয়।

**`client/src/hooks/useChatSocket.js`**

```js
import { useEffect, useState, useCallback, useRef } from "react";
import api from "../api/axios";
import { getSocket } from "../socket";

function useChatSocket(otherUser, currentUserId) {
  const [messages, setMessages] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState("");
  const seenIds = useRef(new Set()); // duplicate message ঠেকানোর জন্য

  // ১. পুরনো message history REST API দিয়ে load করা
  useEffect(() => {
    if (!otherUser) return;

    async function loadHistory() {
      setLoading(true);
      setError("");
      try {
        const res = await api.get(`/messages/${otherUser._id}`);
        setMessages(res.data.messages);
        seenIds.current = new Set(res.data.messages.map((m) => m._id));
      } catch (err) {
        setError("Message history load করতে সমস্যা হয়েছে");
      } finally {
        setLoading(false);
      }
    }

    loadHistory();
  }, [otherUser]);

  // ২. Room join + নতুন message listen করা
  useEffect(() => {
    if (!otherUser) return;

    const socket = getSocket();
    if (!socket) return;

    socket.emit("joinRoom", { otherUserId: otherUser._id });

    function handleNewMessage(payload) {
      // শুধু বর্তমান খোলা conversation-এর message হলেই যোগ করব
      const isRelevant =
        (payload.sender === otherUser._id && payload.receiver === currentUserId) ||
        (payload.sender === currentUserId && payload.receiver === otherUser._id);

      if (!isRelevant) return;

      // Duplicate prevention — একই _id দুইবার state-এ যোগ হবে না
      if (seenIds.current.has(payload._id)) return;
      seenIds.current.add(payload._id);

      setMessages((prev) => [...prev, payload]);
    }

    socket.on("newMessage", handleNewMessage);

    // Cleanup — user পাল্টালে বা component unmount হলে listener সরিয়ে ফেলা
    return () => {
      socket.off("newMessage", handleNewMessage);
    };
  }, [otherUser, currentUserId]);

  // ৩. Message পাঠানোর function
  const sendMessage = useCallback(
    (text) => {
      if (!text.trim() || !otherUser) return;

      const socket = getSocket();
      socket.emit("sendMessage", { receiverId: otherUser._id, text }, (response) => {
        if (!response.success) {
          setError(response.error || "Message পাঠানো যায়নি");
        }
        // নিজের পাঠানো message-ও "newMessage" event দিয়েই ফিরে আসবে (§17, §25),
        // তাই এখানে আলাদা করে state-এ push করার দরকার নেই
      });
    },
    [otherUser]
  );

  return { messages, loading, error, sendMessage };
}

export default useChatSocket;
```

**State-এর design সিদ্ধান্তগুলো:**
- `messages` — একটাই array, history আর real-time দুই ধরনের message-ই এখানে মিলেমিশে থাকে।
- `seenIds` (একটা `Set`, `useRef` দিয়ে রাখা) — কোন message ইতিমধ্যে দেখানো হয়ে গেছে সেটা track করে; `useRef` ব্যবহার করা হয়েছে কারণ এটা re-render trigger করার দরকার নেই, শুধু ভিতরে ভিতরে track করলেই চলে।
- `otherUser` পাল্টালে (মানে অন্য user select করলে) `useEffect`-এর dependency array-এর কারণে পুরনো listener cleanup হয়ে নতুন করে সব setup হয় — প্রতিটা conversation-এর জন্য আলাদা fresh state।

---

## 24. Socket Event Listener Properly Setup ও Cleanup করা

উপরের hook-এর এই অংশটাই listener setup আর cleanup-এর নিয়ম মেনে চলে:

```js
socket.on("newMessage", handleNewMessage);

return () => {
  socket.off("newMessage", handleNewMessage);
};
```

**নিয়ম:** `useEffect`-এর ভিতরে যেকোনো `socket.on()` করলে, সেই একই `useEffect`-এর return function-এ অবশ্যই মিলিয়ে `socket.off()` করতে হবে, **একই function reference** দিয়ে (`handleNewMessage` — named function হিসেবে রাখা হয়েছে ঠিক এই কারণেই, inline arrow function দিলে `off()` দিয়ে ঠিক সেটাকে target করা যায় না)।

---

## 25. Duplicate Message কেন হতে পারে এবং কীভাবে Prevent করব

**Duplicate message হওয়ার সাধারণ কারণগুলো:**

1. **Listener cleanup না করা** — `otherUser` পাল্টালে বা component re-render হলে পুরনো `socket.on("newMessage", ...)` remove না করলে, একই event-এর জন্য একাধিক listener জমে যায় — ফলে একটা message আসলে সেটা দুই/তিনবার handle হয়।
2. **React Strict Mode-এ double effect run** — development mode-এ `useEffect` দুইবার চলে; cleanup ঠিকভাবে না থাকলে duplicate listener তৈরি হয়ে যায়।
3. **Optimistic UI + server echo — দুইবার যোগ করা** — অনেকে message পাঠানোর সাথে সাথেই client-এ নিজে থেকে state-এ push করে ("optimistic update"), তারপর server থেকে সেই একই message আবার `"newMessage"` event দিয়ে ফিরে আসলে সেটাও যোগ করে ফেলে — ফলে একই message দুইবার দেখা যায়।

**আমাদের implementation-এ যেভাবে এটা prevent করা হয়েছে:**

- **`seenIds` Set** — প্রতিটা message-এর `_id` track করা হয়, একই `_id` দ্বিতীয়বার আসলে সেটা ignore করা হয় (§23-এর `handleNewMessage`-এ দেখুন)।
- **Optimistic update না করা** — `sendMessage` function-এ message পাঠানোর সাথে সাথেই client-এ push করা হয় না; বরং server থেকে `"newMessage"` event ফিরে আসা পর্যন্ত অপেক্ষা করা হয় (§17-এর design decision অনুযায়ী sender-ও room-এ থাকে বলে সে নিজেও এই event পায়)। এতে "নিজে একবার যোগ করলাম, তারপর server থেকেও একবার এলো" — এই সমস্যাটাই তৈরি হয় না।
- **প্রতিটা `useEffect`-এ proper cleanup** (§24 অনুযায়ী)।

---

## 26. Component Unmount হলে Socket Listener Cleanup

`useChatSocket` hook-টা যে component-এ ব্যবহার হচ্ছে (`ChatWindow`), সেটা unmount হলে (যেমন user logout করলে বা অন্য page-এ চলে গেলে), `useEffect`-এর cleanup function automatically React নিজেই call করে — যেটা আগেই দেখানো `socket.off("newMessage", handleNewMessage)`।

**Cleanup না করলে কী হয়:**
- Memory leak — পুরনো, আর দরকার নেই এমন listener browser-এর memory-তে থেকে যায়।
- Stale closure bug — পুরনো listener-টা পুরনো `otherUser`/`currentUserId` value ধরে রাখে (JavaScript closure-এর কারণে), ফলে ভুল conversation-এ message যোগ হওয়ার সম্ভাবনা তৈরি হয়।
- Duplicate handling (§25-এ আলোচিত)।

> 💡 এই hook pattern-এর সবচেয়ে বড় সুবিধা — cleanup logic একটা জায়গায় (hook-এর ভিতরে) কেন্দ্রীভূত থাকে, তাই যে component-ই এই hook ব্যবহার করুক না কেন, cleanup সবসময় ঠিকভাবে হয়ে যায়।

---

## 27. Basic Error Handling

**Backend (§17-এর `sendMessage` handler-এ ইতিমধ্যে দেখানো হয়েছে):**

```js
socket.on("sendMessage", async ({ receiverId, text }, callback) => {
  try {
    if (!text || text.trim() === "") {
      throw new Error("Message খালি হতে পারবে না");
    }
    // ... বাকি logic
    callback({ success: true, message: payload });
  } catch (error) {
    callback({ success: false, error: error.message });
  }
});
```

**Frontend — REST API call-এর error (LoginPage/RegisterPage-এ ইতিমধ্যে দেখানো):**

```js
try {
  const res = await api.post("/auth/login", { email, password });
  // ...
} catch (err) {
  setError(err.response?.data?.message || "Login ব্যর্থ হয়েছে");
}
```

**Frontend — Socket acknowledgement-এর error (§23-এর `sendMessage` function-এ):**

```js
socket.emit("sendMessage", { receiverId: otherUser._id, text }, (response) => {
  if (!response.success) {
    setError(response.error || "Message পাঠানো যায়নি");
  }
});
```

**Frontend — Connection-level error:**

```js
// এটা AuthContext বা App.jsx-এ, socket তৈরি হওয়ার পর যোগ করা যায়
const socket = getSocket();
socket?.on("connect_error", (err) => {
  console.log("Socket connection ব্যর্থ:", err.message);
});
```

**মূল নিয়ম:** REST endpoint-এর মতোই প্রতিটা socket event handler-এ `try-catch` রাখুন, আর client-এ প্রতিটা async call (REST বা socket acknowledgement) থেকে আসা error-কে state-এ রেখে user-কে দেখান — silent failure এড়িয়ে চলুন।

---

## 28. Basic Loading State

Loading state তিনটা জায়গায় দরকার — user list load, message history load, এবং form submit।

- **`UserList.jsx`** — `loading` state true থাকা অবস্থায় "User list loading..." দেখায় (§13-এ দেখানো হয়েছে)।
- **`useChatSocket.js`** — `loadHistory()` চলাকালীন `loading` true, শেষ হলে false (§23-এ দেখানো হয়েছে); এটা `ChatWindow.jsx`-এ "Messages loading..." হিসেবে দেখানো হয় (§20)।
- **`LoginPage.jsx` / `RegisterPage.jsx`** — submit button `disabled={loading}` করে রাখা হয়েছে, যাতে একই request দুইবার না পাঠানো যায় (§12)।

সাধারণ pattern সবসময় একই — `try { setLoading(true); await someAsyncCall(); } finally { setLoading(false); }`।

---

## 29. Complete Project Folder Structure

```
mern-chat-app/
│
├── server/
│   ├── config/
│   │   └── db.js
│   │
│   ├── models/
│   │   ├── User.js
│   │   └── Message.js
│   │
│   ├── controllers/
│   │   ├── authController.js
│   │   ├── userController.js
│   │   └── messageController.js
│   │
│   ├── routes/
│   │   ├── authRoutes.js
│   │   ├── userRoutes.js
│   │   └── messageRoutes.js
│   │
│   ├── middleware/
│   │   └── verifyJWT.js
│   │
│   ├── socket/
│   │   └── index.js
│   │
│   ├── utils/
│   │   └── generateRoomId.js
│   │
│   ├── .env
│   ├── server.js
│   └── package.json
│
└── client/
    ├── src/
    │   ├── api/
    │   │   └── axios.js
    │   │
    │   ├── context/
    │   │   └── AuthContext.jsx
    │   │
    │   ├── hooks/
    │   │   └── useChatSocket.js
    │   │
    │   ├── components/
    │   │   ├── UserList.jsx
    │   │   └── ChatWindow.jsx
    │   │
    │   ├── pages/
    │   │   ├── LoginPage.jsx
    │   │   ├── RegisterPage.jsx
    │   │   └── ChatPage.jsx
    │   │
    │   ├── socket.js
    │   ├── App.jsx
    │   └── main.jsx
    │
    ├── .env
    └── package.json
```

---

## Backend Complete Code Flow

একটা request/event ঠিক কোন ফাইলগুলোর ভিতর দিয়ে যায়, সেই সারসংক্ষেপ:

- **Registration/Login:** `server.js` → `authRoutes.js` → `authController.js` → `models/User.js` (bcrypt hash + JWT sign) → response client-এ ফেরত।
- **User list:** `server.js` → `verifyJWT.js` middleware (token check) → `userRoutes.js` → `userController.js` → `models/User.js` থেকে query।
- **Message history:** একইভাবে `verifyJWT.js` → `messageRoutes.js` → `messageController.js` → `generateRoomId.js` দিয়ে roomId হিসাব → `models/Message.js` থেকে query।
- **Socket connection:** `server.js`-এ `initSocket(io)` কল হয় → `socket/index.js`-এর `io.use()` middleware token verify করে → `io.on("connection")` চলে।
- **Real-time message:** `socket/index.js`-এর `"sendMessage"` handler → `generateRoomId.js` → `models/Message.js` (save) → `io.to(roomId).emit()` (relay) → acknowledgement callback।

---

## Frontend Complete Code Flow

- **App শুরু:** `main.jsx` → `App.jsx` (`AuthProvider` + routing) → `localStorage`-এ token থাকলে `user` state restore হয় (কিন্তু socket তখনও connect হয় না — শুধু `login()` কল হলেই হয়, তাই সরাসরি reload-এর পর socket পুনরায় connect করার logic আলাদাভাবে যোগ করতে হবে — নিচে checklist-এ উল্লেখ আছে)।
- **Login/Register:** `LoginPage.jsx`/`RegisterPage.jsx` → `api/axios.js` দিয়ে REST call → success হলে `AuthContext`-এর `login()` → `socket.js`-এর `initSocket()` কল হয়ে connection তৈরি হয়।
- **Chat page:** `ChatPage.jsx` → `UserList.jsx` (REST দিয়ে user list) + `ChatWindow.jsx`।
- **Conversation খোলা:** `ChatWindow.jsx` → `useChatSocket.js` hook চলে → REST দিয়ে history load + socket দিয়ে `"joinRoom"` emit + `"newMessage"` listen শুরু।
- **Message পাঠানো:** `MessageForm` submit → hook-এর `sendMessage()` → `socket.emit("sendMessage", ..., callback)` → response আসলে UI update/error দেখানো।
- **Message receive:** Socket-এর `"newMessage"` event → hook-এর `handleNewMessage()` → `seenIds` check → `messages` state আপডেট → `ChatWindow.jsx` re-render।

---

## Socket Events Table

| Event | Sender | Receiver | Purpose |
|---|---|---|---|
| `connection` *(built-in)* | — | Server | নতুন client connect হলে fire হয় |
| `joinRoom` | Client | Server | দুইজন user-এর private room-এ join করা |
| `sendMessage` | Client | Server | নতুন message পাঠানো (acknowledgement callback সহ) |
| `newMessage` | Server | Room-এর সব member (sender + receiver) | নতুন message real-time সবাইকে জানানো |
| `disconnect` *(built-in)* | — | Server | Client সংযোগ বিচ্ছিন্ন হলে fire হয় |
| `connect_error` *(built-in)* | — | Client | Authentication বা connection ব্যর্থ হলে |

---

## REST API বনাম Socket.IO Responsibility Table

| Method | Endpoint/Event | মাধ্যম | Auth লাগে? | কাজ |
|---|---|---|---|---|
| POST | `/api/auth/register` | REST | না | নতুন user registration |
| POST | `/api/auth/login` | REST | না | Login, JWT token issue |
| GET | `/api/users` | REST | হ্যাঁ | সব user-এর list (নিজেকে বাদে) |
| GET | `/api/messages/:userId` | REST | হ্যাঁ | নির্দিষ্ট user-এর সাথে conversation history |
| — | `joinRoom` | Socket.IO | হ্যাঁ (connection-এই) | Private room-এ join |
| — | `sendMessage` | Socket.IO | হ্যাঁ | নতুন message পাঠানো + DB save |
| — | `newMessage` | Socket.IO | হ্যাঁ | Real-time message delivery |

---

## Common Bugs এবং Debugging Techniques

| সমস্যা | সম্ভাব্য কারণ | Debug/Fix করার উপায় |
|---|---|---|
| Socket connect হচ্ছে না, `connect_error` আসছে | `JWT_SECRET` REST আর Socket-এ আলাদা, অথবা token expired/missing | দুই জায়গায় একই `.env`-এর `JWT_SECRET` ব্যবহার হচ্ছে কিনা check করুন; `socket.handshake.auth.token`-এ ঠিক token যাচ্ছে কিনা console log করুন |
| Login করা token কাজ করে কিন্তু user list আসে না (401) | Frontend axios interceptor token attach করছে না | `api/axios.js`-এর interceptor ঠিকভাবে `localStorage`-এ token খুঁজছে কিনা, header ঠিক format-এ (`Bearer <token>`) যাচ্ছে কিনা দেখুন |
| দুইজন user একে অপরের message পাচ্ছে না | দুইজনের roomId আলাদা হয়ে যাচ্ছে | `generateRoomId` function-এ sort হচ্ছে কিনা নিশ্চিত করুন; দুই পাশেই একই function ব্যবহার হচ্ছে কিনা (client roomId হিসাব করলে সেটা বাদ দিয়ে server-কে করতে দিন) |
| Message পাঠানোর সাথে সাথে দেখা যাচ্ছে না, refresh করলে দেখা যাচ্ছে | `joinRoom` emit না হওয়া, অথবা `"newMessage"` listener register-ই হয়নি | Chat window খোলার সাথে সাথেই `joinRoom` emit হচ্ছে কিনা console log করুন; `useEffect`-এর dependency array ঠিক আছে কিনা দেখুন |
| একই message দুই/তিনবার দেখাচ্ছে | Listener cleanup হচ্ছে না, বা duplicate listener register হচ্ছে | §25-অনুযায়ী `seenIds` Set ঠিকভাবে কাজ করছে কিনা, `useEffect` cleanup আছে কিনা check করুন |
| Message history load-এ সব message উল্টো ক্রমে আসছে | MongoDB query-তে `.sort()` মিসিং | `Message.find({ roomId }).sort({ createdAt: 1 })` ঠিকভাবে আছে কিনা দেখুন |
| CORS error browser console-এ | `CLIENT_URL` মিলছে না, বা Socket.IO-এর cors config আলাদা | Express-এর `cors()` আর `new Server(server, { cors: {...} })` — দুই জায়গাতেই একই origin দিন |
| Page reload করলে chat page-এ ঢুকলেও message পাঠানো যায় না | Reload-এর পর socket পুনরায় connect হচ্ছে না (কারণ `initSocket` শুধু `login()`-এর ভিতরে কল হয়) | `App.jsx`/`AuthProvider`-এ mount হওয়ার সময় `localStorage`-এ token থাকলে সেখান থেকেও `initSocket(token)` কল করার logic যোগ করুন |
| Server console-এ "Cannot read properties of undefined (reading 'id')" | `socket.user` set হয়নি (auth middleware fail করেছে কিন্তু connection তবু হয়ে গেছে ধরে নেওয়া হয়েছে) | `io.use()` middleware ঠিকভাবে `next(new Error(...))` কল করছে কিনা, এবং client-side `connect_error` handle করছে কিনা যাচাই করুন |

---

## Final Checklist

- [ ] Register/Login কাজ করছে, JWT token `localStorage`-এ save হচ্ছে
- [ ] User list load হচ্ছে (নিজেকে বাদে)
- [ ] একজন user select করলে তার সাথের পুরনো message history দেখা যাচ্ছে
- [ ] Login-এর পরপরই socket connect হচ্ছে (valid token সহ)
- [ ] Chat window খুললে `joinRoom` emit হচ্ছে
- [ ] Message পাঠালে সেটা DB-তে save হচ্ছে এবং `newMessage` দিয়ে broadcast হচ্ছে
- [ ] দুইটা আলাদা browser (বা একটা normal + একটা incognito) দিয়ে দুইজন user দিয়ে টেস্ট করলে real-time-এ message আদান-প্রদান হচ্ছে
- [ ] বারবার message পাঠালে বা re-render হলেও duplicate message দেখা যাচ্ছে না
- [ ] Page reload করলেও history হারিয়ে যাচ্ছে না (REST দিয়ে reload হচ্ছে)
- [ ] Logout করলে socket ঠিকভাবে disconnect হচ্ছে
- [ ] Empty message পাঠানো block হচ্ছে, error দেখানো হচ্ছে

---

## Practice Assignment

উপরের পুরো code নিজে হাতে লিখে, দেখে দেখে নয় বরং প্রতিটা অংশ বুঝে বুঝে বানান। Assignment সম্পন্ন ধরা হবে যখন নিচের সব acceptance criteria পূরণ হবে:

### কাজ

1. একটা নতুন `mern-chat-app` project থেকে শুরু করে, উপরে দেখানো folder structure অনুসরণ করে সম্পূর্ণ backend আর frontend বানান।
2. দুইজন test user register করুন (যেমন "ahsan" আর "karim")।
3. দুইটা আলাদা browser session (একটাতে ahsan হিসেবে login, আরেকটাতে karim হিসেবে login) দিয়ে একে অপরকে message পাঠিয়ে যাচাই করুন।

### Acceptance Criteria

- [ ] দুইজন user register/login করতে পারছে
- [ ] প্রতিটা user login করার পর বাকি সব user-এর list দেখতে পাচ্ছে
- [ ] একজন user অন্যজনকে select করলে তাদের আগের কোনো message থাকলে তা load হচ্ছে (প্রথমবার খালি থাকবে, এটাই স্বাভাবিক)
- [ ] ahsan থেকে message পাঠালে karim-এর screen-এ **এক সেকেন্ডের মধ্যে**, কোনো page reload ছাড়াই সেই message দেখা যাচ্ছে
- [ ] karim থেকে reply করলে ahsan-এর screen-এও একইভাবে real-time দেখা যাচ্ছে
- [ ] MongoDB compass/shell দিয়ে চেক করলে `messages` collection-এ সব message ঠিকভাবে save হয়ে আছে, প্রতিটাতে সঠিক `roomId`, `sender`, `receiver` আছে
- [ ] Page reload করলে conversation history অক্ষত থাকছে
- [ ] একই message কোনো অবস্থাতেই দুইবার দেখাচ্ছে না (দ্রুত কয়েকবার message পাঠিয়ে টেস্ট করুন)
- [ ] Network tab-এ গিয়ে REST call (login, users, messages history) আর Socket.IO-এর WebSocket connection — দুটোই আলাদাভাবে চিহ্নিত করে দেখাতে পারছেন সেগুলো ঠিক কোথায় ব্যবহার হচ্ছে
- [ ] Console-এ কোনো unhandled error/warning আসছে না (React key warning, socket duplicate listener warning ইত্যাদি)

### Bonus (Optional, স্কোপের বাইরে কিন্তু চেষ্টা করতে পারেন)

এগুলো এই note-এ ইচ্ছাকৃতভাবে বাদ দেওয়া হয়েছিল — core concept-এ comfortable হয়ে গেলে নিজে থেকে explore করে দেখতে পারেন:

- Online/offline status একটা simple `Map` দিয়ে track করা (userId → socket.id)
- Typing indicator (`typing`/`stopTyping` event)
- Message-এ delivered/read status যোগ করা

---

এই note শেষ করার পর আপনি নিজে থেকে একটা basic MERN real-time chat application-এর প্রতিটা অংশ — architecture, database design, REST/Socket.IO split, room-based private messaging, এবং state management — বুঝে implement করতে পারবেন, কোনো tutorial copy না করেই। শুভকামনা! 🚀
