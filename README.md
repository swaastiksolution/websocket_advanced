# websocket_advanced

Building upon the foundational knowledge of WebSockets, let's delve into more advanced concepts to enhance the robustness and scalability of your real-time applications using React on the frontend and Node.js on the backend.
----
## 1. Enhancing the Backend with Node.js
### a. Structuring the WebSocket Server
Instead of a monolithic server file, structure your server to handle multiple concerns:

- **Modularize Handlers:** Separate connection, message, and error handling into distinct modules.
- **Manage Client Connections:** Maintain a registry of connected clients to facilitate targeted messaging and efficient resource management.
#### Example:
````js
// server.js
const WebSocket = require('ws');
const { handleConnection } = require('./handlers/connectionHandler');

const wss = new WebSocket.Server({ port: 8080 });
console.log('WebSocket server running on ws://localhost:8080');

wss.on('connection', (ws) => {
  handleConnection(ws, wss);
});

````

````js
// handlers/connectionHandler.js
const { handleMessage } = require('./messageHandler');

const clients = new Map();

function handleConnection(ws, wss) {
  const id = generateUniqueId();
  clients.set(id, ws);
  console.log(`Client ${id} connected`);

  ws.on('message', (message) => handleMessage(message, ws, wss, id));
  ws.on('close', () => {
    clients.delete(id);
    console.log(`Client ${id} disconnected`);
  });
}

function generateUniqueId() {
  return Math.random().toString(36).substr(2, 9);
}

module.exports = { handleConnection };
````

### b. Implementing Heartbeat Mechanism
To ensure active connections and detect disconnected clients:
````js
// server.js
function heartbeat() {
  this.isAlive = true;
}

wss.on('connection', (ws) => {
  ws.isAlive = true;
  ws.on('pong', heartbeat);
  // ...existing connection handling...
});

const interval = setInterval(() => {
  wss.clients.forEach((ws) => {
    if (!ws.isAlive) return ws.terminate();
    ws.isAlive = false;
    ws.ping();
  });
}, 30000);
````

### c. Handling Different Message Types
Define a protocol for various message types to manage different functionalities:
````js
// handlers/messageHandler.js
function handleMessage(message, ws, wss, clientId) {
  const parsedMessage = JSON.parse(message);
  switch (parsedMessage.type) {
    case 'chat':
      broadcastMessage(parsedMessage.content, wss, clientId);
      break;
    case 'notification':
      sendNotification(parsedMessage.content, ws);
      break;
    // Handle other message types...
  }
}

function broadcastMessage(content, wss, senderId) {
  wss.clients.forEach((client) => {
    if (client.readyState === WebSocket.OPEN && client !== senderId) {
      client.send(JSON.stringify({ type: 'chat', content }));
    }
  });
}

function sendNotification(content, ws) {
  ws.send(JSON.stringify({ type: 'notification', content }));
}

module.exports = { handleMessage };

````
----

## 2. Advancing the Frontend with React
### a. Custom WebSocket Hook
Create a custom React hook to manage WebSocket connections and encapsulate related logic:
````js
// useWebSocket.js
import { useEffect, useRef } from 'react';

export const useWebSocket = (url, onMessage) => {
  const ws = useRef(null);

  useEffect(() => {
    ws.current = new WebSocket(url);

    ws.current.onopen = () => console.log('Connected to WebSocket server');
    ws.current.onmessage = (event) => onMessage(event);
    ws.current.onclose = () => console.log('Disconnected from WebSocket server');

    return () => ws.current.close();
  }, [url, onMessage]);

  const sendMessage = (message) => {
    if (ws.current.readyState === WebSocket.OPEN) {
      ws.current.send(JSON.stringify(message));
    }
  };

  return sendMessage;
};
````
### b. Handling Reconnection Logic
Implement reconnection strategies to maintain connectivity:
````js
// useWebSocket.js
import { useEffect, useRef } from 'react';

export const useWebSocket = (url, onMessage, onError) => {
  const ws = useRef(null);
  const reconnectInterval = useRef(null);

  const connect = () => {
    ws.current = new WebSocket(url);

    ws.current.onopen = () => {
      console.log('Connected to WebSocket server');
      if (reconnectInterval.current) {
        clearInterval(reconnectInterval.current);
        reconnectInterval.current = null;
      }
    };

    ws.current.onmessage = (event) => onMessage(event);

    ws.current.onclose = () => {
      console.log('Disconnected from WebSocket server');
      if (!reconnectInterval.current) {
        reconnectInterval.current = setInterval(connect, 5000);
      }
    };

    ws.current.onerror = (error) => {
      console.error('WebSocket error:', error);
      onError && onError(error);
    };
  };

  useEffect(() => {
    connect();
    return () => {
      if (reconnectInterval.current) {
        clearInterval(reconnectInterval.current);
      }
      ws.current && ws.current.close();
    };
  }, [url]);

  const sendMessage = (message) => {
    if (ws.current.readyState === WebSocket.OPEN) {
      ws.current.send(JSON.stringify(message));
    }
  };

  return sendMessage;
};
````
### c. Optimizing Performance
- **Batch Messages:** Accumulate messages and send them in batches to reduce network overhead.
- **Throttling:** Limit the rate of message sending to prevent overwhelming the server.
**Example**
````js
  // useWebSocket.js
import { useEffect, useRef } from 'react';
import { throttle } from 'lodash';

export const useWebSocket = (url, onMessage) => {
  const ws = useRef(null);
  const messageQueue = useRef([]);

  useEffect(() => {
    ws.current = new WebSocket(url);

    ws.current.onopen = () => {
      console.log('Connected to WebSocket server');
      // Send any queued messages
      messageQueue.current.forEach((msg) => ws.current.send(msg));
      messageQueue.current = [];
    };
    
    ws.current.onmessage = (event) => onMessage(event);
    ws.current.onclose = () => console.log('Disconnected from WebSocket server');
    
    return () => ws.current.close();
  }, [url, onMessage]);
  
  const sendMessage = throttle((message) => {
    const msg = JSON.stringify(message);
    if (ws.current.readyState === WebSocket.OPEN) {
      ws.current.send(msg);
    } else {
      messageQueue.current.push(msg);
    }
  }, 1000); // Throttle messages to one per second

  return sendMessage;
};

````


## 3. Security Considerations
- **Data Validation:** Always validate incoming messages to prevent injection attacks.
- **Authentication:** Implement token-based authentication to ensure only authorized clients can connect.
- **Encryption:** Use Secure WebSockets `(wss://)` to encrypt data in transit.
