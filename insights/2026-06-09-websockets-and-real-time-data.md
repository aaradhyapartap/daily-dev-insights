# 📌 WebSockets and real-time data
*June 09, 2026 · Daily Dev Insight*

## 🧠 Overview

WebSockets represent one of those paradigm shifts that fundamentally changed how we think about web communication. Unlike traditional HTTP's request-response cycle, WebSockets establish a persistent, full-duplex connection that lets both client and server send data whenever they need to. This isn't just about technical elegance—it's about enabling experiences that feel genuinely real-time.

The beauty of WebSockets lies in their simplicity and efficiency. Once the initial handshake completes (which happens over HTTP), you get a raw TCP connection with minimal overhead. No more polling every few seconds hoping for updates, no more long-polling hacks that make your server cry. When data changes on the server, it flows instantly to connected clients. When a user takes an action, the server knows immediately.

What makes WebSockets particularly powerful in 2026 is how they've become the backbone of modern collaborative applications. Every time you see live cursors in Figma, real-time document editing in Notion, or instant messaging that just works, there's likely a WebSocket connection doing the heavy lifting behind the scenes.

## 💡 Key Concepts

• **Persistent Connection**: Unlike HTTP, the connection stays open, eliminating the overhead of establishing new connections for each data exchange
• **Bidirectional Communication**: Both client and server can initiate data transmission, enabling true push notifications and reactive updates
• **Low Latency**: Messages can be delivered in milliseconds since there's no connection establishment delay or HTTP header overhead
• **Connection Management**: Handling disconnections, reconnections, and connection health becomes critical for robust real-time applications
• **Scaling Considerations**: Each WebSocket connection consumes server resources; horizontal scaling requires message broadcasting strategies

## 🐍 Python Example

```python
import asyncio
import websockets
import json
from datetime import datetime

class ChatRoom:
    def __init__(self):
        self.clients = set()
        self.message_history = []
    
    async def register_client(self, websocket):
        """Add a new client to the chat room"""
        self.clients.add(websocket)
        # Send recent message history to new client
        for message in self.message_history[-10:]:
            await websocket.send(json.dumps(message))
        print(f"Client connected. Total: {len(self.clients)}")
    
    async def unregister_client(self, websocket):
        """Remove client from chat room"""
        self.clients.discard(websocket)
        print(f"Client disconnected. Total: {len(self.clients)}")
    
    async def broadcast_message(self, message_data):
        """Send message to all connected clients"""
        if not self.clients:
            return
        
        # Add timestamp and store in history
        message_data['timestamp'] = datetime.now().isoformat()
        self.message_history.append(message_data)
        
        # Keep only last 100 messages
        if len(self.message_history) > 100:
            self.message_history.pop(0)
        
        # Broadcast to all clients
        message = json.dumps(message_data)
        disconnected = set()
        
        for client in self.clients:
            try:
                await client.send(message)
            except websockets.exceptions.ConnectionClosed:
                disconnected.add(client)
        
        # Clean up disconnected clients
        for client in disconnected:
            self.clients.discard(client)

# Global chat room instance
chat_room = ChatRoom()

async def handle_client(websocket, path):
    """Handle individual WebSocket connections"""
    await chat_room.register_client(websocket)
    
    try:
        async for message in websocket:
            try:
                data = json.loads(message)
                await chat_room.broadcast_message({
                    'user': data.get('user', 'Anonymous'),
                    'content': data.get('content', ''),
                    'type': 'message'
                })
            except json.JSONDecodeError:
                await websocket.send(json.dumps({
                    'type': 'error',
                    'message': 'Invalid JSON format'
                }))
    except websockets.exceptions.ConnectionClosed:
        pass
    finally:
        await chat_room.unregister_client(websocket)

# Start the WebSocket server
start_server = websockets.serve(handle_client, "localhost", 8765)
print("Chat server starting on ws://localhost:8765")
asyncio.get_event_loop().run_until_complete(start_server)
asyncio.get_event_loop().run_forever()
```

## 🟨 JavaScript Example

```javascript
// Client-side WebSocket chat implementation
class ChatClient {
    constructor(serverUrl) {
        this.serverUrl = serverUrl;
        this.socket = null;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 5;
        this.reconnectDelay = 1000;
        this.messageHandlers = new Map();
    }
    
    connect() {
        try {
            this.socket = new WebSocket(this.serverUrl);
            
            this.socket.onopen = (event) => {
                console.log('Connected to chat server');
                this.reconnectAttempts = 0;
                this.onConnectionStatus('connected');
            };
            
            this.socket.onmessage = (event) => {
                try {
                    const message = JSON.parse(event.data);
                    this.handleMessage(message);
                } catch (error) {
                    console.error('Failed to parse message:', error);
                }
            };
            
            this.socket.onclose = (event) => {
                console.log('Disconnected from chat server');
                this.onConnectionStatus('disconnected');
                this.attemptReconnect();
            };
            
            this.socket.onerror = (error) => {
                console.error('WebSocket error:', error);
                this.onConnectionStatus('error');
            };
            
        } catch (error) {
            console.error('Failed to create WebSocket connection:', error);
            this.attemptReconnect();
        }
    }
    
    attemptReconnect() {
        if (this.reconnectAttempts < this.maxReconnectAttempts) {
            this.reconnectAttempts++;
            console.log(`Reconnecting... Attempt ${this.reconnectAttempts}`);
            
            setTimeout(() => {
                this.connect();
            }, this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1)); // Exponential backoff
        } else {
            console.error('Max reconnection attempts reached');
            this.onConnectionStatus('failed');
        }
    }
    
    sendMessage(user, content) {
        if (this.socket && this.socket.readyState === WebSocket.OPEN) {
            this.socket.send(JSON.stringify({
                user: user,
                content: content
            }));
        } else {
            console.error('Cannot send message: WebSocket not connected');
        }
    }
    
    handleMessage(message) {
        const handler = this.messageHandlers.get(message.type) || this.onMessage;
        if (handler) {
            handler(message);
        }
    }
    
    // Override these methods to handle events
    onMessage(message) {
        console.log('Received message:', message);
    }
    
    onConnectionStatus(status) {
        console.log('Connection status:', status);
    }
    
    disconnect() {
        if (this.socket) {
            this.socket.close();
        }
    }
}

// Usage example
const chat = new ChatClient('ws://localhost:8765');

chat.onMessage = (message) => {
    const messagesDiv = document.getElementById('messages');
    const messageElement = document.createElement('div');
    messageElement.innerHTML = `<strong>${message.user}:</strong> ${message.content}`;
    messagesDiv.appendChild(messageElement);
    messagesDiv