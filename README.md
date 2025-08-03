# Basic-Redis-Chat-App-Demo-Node.js-
Welcome to the Redis Chat App Demo! This project shows you how to build a basic, real-time chat application using Node.js, Socket.IO, and Redis. It leverages Redis' Pub/Sub and data storage features to keep things fast, reliable, and scalable.
It's a great demo for learning how to:

Build a real-time chat system 🗨️

Use Redis for both data storage and real-time communication 🚀

Manage chat rooms, private messages, and online users 🌐


🧰 Tech Stack
Layer	Tools
Frontend	React, Socket.IO
Backend	Node.js, Express
Realtime	Socket.IO, Redis Pub/Sub
Database	Redis

🔄 How It Works
Let’s break down what happens behind the scenes.

🧱 Initialization
When the app starts, we initialize demo users and rooms in Redis if they don't already exist.

✅ Check for existing users:
redis
Copy
Edit
EXISTS total_users
🧍 Create users:
Generate a new ID → INCR total_users

Map the username → SET username:nick user:1

Store user details →

redis
Copy
Edit
HSET user:1 username "nick" password "bcrypt_hashed_password"
🏠 Add user to the “General” room:
redis
Copy
Edit
SADD user:1:rooms "0"
🔒 Create private rooms:
A private room between two users (e.g., user 1 and user 2) gets a room ID like 1:2. Each user is added to that room:

redis
Copy
Edit
SADD user:1:rooms 1:2  
SADD user:2:rooms 1:2
Then, add messages:

redis
Copy
Edit
ZADD room:1:2 1615480369 '{"from": 1, "date": 1615480369, "message": "Hello", "roomId": "1:2"}'
💬 Add messages to “General” room:
redis
Copy
Edit
ZADD room:0 1615480370 '{"from": 2, "message": "Hey!", "roomId": "0"}'
📝 Registration Flow
When a user signs up:

We hash the password

Store it in a Redis hash set

Add them to the “General” room by default

✅ Code Example
js
Copy
Edit
const hashedPassword = await bcrypt.hash(password, 10);
const nextId = await incr("total_users");
const userKey = `user:${nextId}`;
await set(`username:${username}`, userKey);
await hmset(userKey, ["username", username, "password", hashedPassword]);
await sadd(`user:${nextId}:rooms`, "0");
📦 Redis Data Structure
👤 Users
Hash set:
Key: user:{userId}
Fields: username, password

Username lookup:
Key: username:{username} → Returns userId

💬 Rooms
Each user has a set of room IDs:
Key: user:{id}:rooms

Room name is stored in:

redis
Copy
Edit
SET room:{roomId}:name "General"
Messages are stored as sorted sets:

redis
Copy
Edit
ZADD room:{roomId} {timestamp} {message JSON}
🧠 Code: Get All My Rooms
Here’s how we list all rooms a user is part of (private + public):

js
Copy
Edit
const rooms = [];
for (let roomId of roomIds) {
    let name = await get(`room:${roomId}:name`);
    if (!name) {
        // Possibly a private room
        const exists = await exists(`room:${roomId}`);
        if (!exists) continue;

        const userIds = roomId.split(":");
        const usernames = await Promise.all([
            hmget(`user:${userIds[0]}`, "username"),
            hmget(`user:${userIds[1]}`, "username")
        ]);
        rooms.push({ id: roomId, names: usernames });
    } else {
        rooms.push({ id: roomId, names: [name] });
    }
}
📬 Messaging
🔄 Pub/Sub
We use Redis’ Pub/Sub to broadcast messages across all connected servers:

redis
Copy
Edit
SUBSCRIBE message
When a message is published to the message channel, every connected server receives it and forwards it to WebSocket clients.

This allows multi-server communication without worrying about how they’re implemented — just listen and broadcast!

✏️ Code Example: Send Message
js
Copy
Edit
const messageString = JSON.stringify(message);
const roomKey = `room:${message.roomId}`;

// Mark sender online
await sadd("online_users", message.from);

// If it's a new private room, notify both users
if (!(await exists(`${roomKey}:name`)) && !(await exists(roomKey))) {
    const ids = message.roomId.split(":");
    const usernames = await Promise.all([
        hmget(`user:${ids[0]}`, "username"),
        hmget(`user:${ids[1]}`, "username")
    ]);
    publish("show.room", { id: message.roomId, names: usernames });
    socket.broadcast.emit("show.room", { id: message.roomId, names: usernames });
}

// Store and broadcast the message
await zadd(roomKey, "" + message.date, messageString);
publish("message", message);
io.to(roomKey).emit("message", message);
👥 Session Handling
Redis also handles sessions and online states.

🟢 Connect
Store user ID in session

Add them to the online_users set:

redis
Copy
Edit
SADD online_users 1
Broadcast user joined

🔴 Disconnect
Remove from online_users:

redis
Copy
Edit
SREM online_users 1
Broadcast user left

⚙️ Session Config (Node.js)
js
Copy
Edit
const session = require("express-session");
const RedisStore = require("connect-redis")(session);

const sessionMiddleware = session({
  store: new RedisStore({ client: redisClient }),
  secret: "keyboard cat",
  saveUninitialized: true,
  resave: true,
});
🧪 Pub/Sub: Behind the Scenes
Redis lets multiple app instances stay in sync by subscribing to the same message channel. A message looks like this:

json
Copy
Edit
{
  "serverId": 4132,
  "type": "message",
  "data": {
    "from": 1,
    "date": 1615480369,
    "message": "Hello",
    "roomId": "1:2"
  }
}
type = kind of event (connect, message, disconnect)

serverId = helps avoid re-sending a message to itself

data = actual message

🧪 How to Run Locally
Set your Redis credentials (via .env or Docker):

env
Copy
Edit
REDIS_ENDPOINT_URL = your_redis_server_url
REDIS_PASSWORD = your_redis_password
▶️ Start the Frontend
bash
Copy
Edit
cd client
yarn install
yarn start
▶️ Start the Backend
bash
Copy
Edit
cd server
yarn install
yarn start
