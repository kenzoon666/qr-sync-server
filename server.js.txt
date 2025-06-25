const express = require('express');
const { createServer } = require('http');
const { WebSocketServer } = require('ws');
const { v4: uuidv4 } = require('uuid');
const cors = require('cors');

const app = express();
app.use(cors());

const server = createServer(app);
const wss = new WebSocketServer({ server });

const sessions = new Map();

app.get('/session', (req, res) => {
  const id = uuidv4();
  sessions.set(id, { clients: [], text: '' });
  setTimeout(() => sessions.delete(id), 10 * 60 * 1000); // 10 минут
  res.json({ sessionId: id });
});

wss.on('connection', (ws) => {
  let currentSession = null;

  ws.on('message', msg => {
    const { type, sessionId, text } = JSON.parse(msg);
    if (type === 'join') {
      const session = sessions.get(sessionId);
      if (session) {
        session.clients.push(ws);
        currentSession = sessionId;
        ws.send(JSON.stringify({ type: 'init', text: session.text }));
      } else {
        ws.send(JSON.stringify({ type: 'error', message: 'Invalid session ID' }));
      }
    } else if (type === 'update' && currentSession) {
      const session = sessions.get(currentSession);
      if (session) {
        session.text = text;
        session.clients.forEach(client => {
          if (client !== ws && client.readyState === 1) {
            client.send(JSON.stringify({ type: 'update', text }));
          }
        });
      }
    }
  });

  ws.on('close', () => {
    if (currentSession) {
      const session = sessions.get(currentSession);
      if (session) {
        session.clients = session.clients.filter(c => c !== ws);
      }
    }
  });
});

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => console.log(`Server running on ${PORT}`));
