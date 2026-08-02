const express = require('express');
const http = require('http');
const path = require('path');
const { Server } = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = new Server(server);

const PORT = process.env.PORT || 3000;

app.use(express.static(path.join(__dirname, 'public')));

// In-memory store: { [roomCode]: { messages: [...], users: Set } }
const rooms = {};
const MAX_MESSAGES_PER_ROOM = 300;

function getRoom(code) {
  if (!rooms[code]) {
    rooms[code] = { messages: [], users: new Map() };
  }
  return rooms[code];
}

io.on('connection', (socket) => {
  let currentRoom = null;
  let currentName = null;

  socket.on('join', ({ code, name }) => {
    if (!code || !name) return;
    code = String(code).trim().toUpperCase().slice(0, 12);
    name = String(name).trim().slice(0, 20);
    if (!code || !name) return;

    currentRoom = code;
    currentName = name;
    socket.join(code);

    const room = getRoom(code);
    room.users.set(socket.id, name);

    // Send history to the joining client
    socket.emit('history', room.messages);

    // Let the room know someone joined
    const joinMsg = {
      system: true,
      text: `${name} joined the frequency`,
      ts: Date.now()
    };
    room.messages.push(joinMsg);
    if (room.messages.length > MAX_MESSAGES_PER_ROOM) room.messages.shift();
    io.to(code).emit('message', joinMsg);

    io.to(code).emit('presence', { count: room.users.size });
  });

  socket.on('message', ({ text }) => {
    if (!currentRoom || !currentName) return;
    if (!text || !String(text).trim()) return;
    const room = getRoom(currentRoom);
    const msg = {
      user: currentName,
      text: String(text).trim().slice(0, 500),
      ts: Date.now()
    };
    room.messages.push(msg);
    if (room.messages.length > MAX_MESSAGES_PER_ROOM) room.messages.shift();
    io.to(currentRoom).emit('message', msg);
  });

  socket.on('disconnect', () => {
    if (!currentRoom) return;
    const room = rooms[currentRoom];
    if (!room) return;
    room.users.delete(socket.id);

    const leaveMsg = {
      system: true,
      text: `${currentName} left the frequency`,
      ts: Date.now()
    };
    room.messages.push(leaveMsg);
    if (room.messages.length > MAX_MESSAGES_PER_ROOM) room.messages.shift();
    io.to(currentRoom).emit('message', leaveMsg);
    io.to(currentRoom).emit('presence', { count: room.users.size });

    // Clean up empty rooms after a while to avoid memory buildup
    if (room.users.size === 0) {
      setTimeout(() => {
        if (rooms[currentRoom] && rooms[currentRoom].users.size === 0) {
          delete rooms[currentRoom];
        }
      }, 10 * 60 * 1000); // 10 minutes
    }
  });
});

server.listen(PORT, () => {
  console.log(`Frequency chat running on port ${PORT}`);
});
