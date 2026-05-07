# 📌 Event-driven architecture
*May 07, 2026 · Daily Dev Insight*

## 🧠 Overview

Event-driven architecture (EDA) is a design pattern where components communicate through the production and consumption of events rather than direct method calls. Think of it as a pub-sub system where services announce what happened (events) without caring who's listening. This creates loosely coupled systems where adding new features often means just adding new event handlers, rather than modifying existing code.

The real power of EDA emerges in complex systems where multiple services need to react to the same business event. When a user places an order, you might need to update inventory, send confirmation emails, trigger shipping workflows, and update analytics—all without the order service knowing about these downstream concerns. This architectural style has become increasingly popular with the rise of microservices and cloud-native applications.

What makes EDA particularly compelling is its natural fit for modern business requirements: real-time updates, scalable processing, and the ability to add new features without touching existing systems. However, it comes with trade-offs in complexity and debugging that every team should carefully consider.

## 💡 Key Concepts

• **Event Producers and Consumers**: Services that emit events (producers) are decoupled from services that react to them (consumers). A single event can have zero, one, or many consumers.

• **Event Store/Bus**: A central mechanism for routing events from producers to consumers. This could be a message queue (RabbitMQ), a streaming platform (Apache Kafka), or cloud services (AWS EventBridge).

• **Event Sourcing**: Instead of storing current state, store the sequence of events that led to that state. This provides complete audit trails and the ability to replay events.

• **Eventual Consistency**: Unlike synchronous systems, EDA accepts that different parts of the system may be temporarily out of sync, with consistency achieved over time.

• **Idempotency**: Event handlers must be designed to handle duplicate events gracefully, as event delivery guarantees often involve "at least once" semantics.

## 🐍 Python Example

```python
import asyncio
from dataclasses import dataclass, asdict
from typing import List, Callable, Any
import json
from datetime import datetime

@dataclass
class Event:
    type: str
    data: dict
    timestamp: datetime = None
    
    def __post_init__(self):
        if not self.timestamp:
            self.timestamp = datetime.now()

class EventBus:
    def __init__(self):
        self._handlers: dict[str, List[Callable]] = {}
    
    def subscribe(self, event_type: str, handler: Callable):
        """Subscribe a handler to an event type"""
        if event_type not in self._handlers:
            self._handlers[event_type] = []
        self._handlers[event_type].append(handler)
    
    async def publish(self, event: Event):
        """Publish an event to all subscribers"""
        handlers = self._handlers.get(event.type, [])
        if handlers:
            # Run all handlers concurrently
            await asyncio.gather(*[handler(event) for handler in handlers])
        print(f"📡 Published event: {event.type}")

# Example application: E-commerce order system
class OrderService:
    def __init__(self, event_bus: EventBus):
        self.event_bus = event_bus
    
    async def place_order(self, user_id: str, items: List[dict]):
        order_id = f"order_{datetime.now().timestamp()}"
        
        # Emit order created event
        event = Event(
            type="order.created",
            data={"order_id": order_id, "user_id": user_id, "items": items}
        )
        await self.event_bus.publish(event)
        return order_id

# Event handlers
async def send_confirmation_email(event: Event):
    print(f"📧 Sending confirmation email for order {event.data['order_id']}")

async def update_inventory(event: Event):
    print(f"📦 Updating inventory for {len(event.data['items'])} items")

async def trigger_shipping(event: Event):
    print(f"🚚 Creating shipping label for order {event.data['order_id']}")

# Wire it all together
async def main():
    bus = EventBus()
    
    # Subscribe handlers to events
    bus.subscribe("order.created", send_confirmation_email)
    bus.subscribe("order.created", update_inventory)
    bus.subscribe("order.created", trigger_shipping)
    
    # Create order service and process an order
    order_service = OrderService(bus)
    order_id = await order_service.place_order("user123", [{"sku": "ABC", "qty": 2}])
    print(f"✅ Order {order_id} processed")

# asyncio.run(main())
```

## 🟨 JavaScript Example

```javascript
const EventEmitter = require('events');

class EventBus extends EventEmitter {
    constructor() {
        super();
        this.eventLog = [];
    }
    
    publish(eventType, data) {
        const event = {
            type: eventType,
            data,
            timestamp: new Date().toISOString(),
            id: Math.random().toString(36).substr(2, 9)
        };
        
        this.eventLog.push(event);
        this.emit(eventType, event);
        console.log(`📡 Published: ${eventType} (${event.id})`);
    }
    
    subscribe(eventType, handler) {
        this.on(eventType, handler);
        console.log(`🎧 Subscribed to ${eventType}`);
    }
}

// Domain services
class UserService {
    constructor(eventBus) {
        this.eventBus = eventBus;
        this.setupEventHandlers();
    }
    
    setupEventHandlers() {
        this.eventBus.subscribe('user.registered', this.sendWelcomeEmail.bind(this));
        this.eventBus.subscribe('user.registered', this.createUserProfile.bind(this));
    }
    
    registerUser(email, name) {
        const userId = `user_${Date.now()}`;
        console.log(`👤 Registering user: ${email}`);
        
        // Emit user registered event
        this.eventBus.publish('user.registered', {
            userId,
            email,
            name,
            registrationSource: 'web'
        });
        
        return userId;
    }
    
    async sendWelcomeEmail(event) {
        const { email, name } = event.data;
        // Simulate async email sending
        setTimeout(() => {
            console.log(`📧 Welcome email sent to ${email}`);
        }, 100);
    }
    
    async createUserProfile(event) {
        const { userId, name } = event.data;
        console.log(`📝 Created profile for ${name} (${userId})`);
    }
}

class AnalyticsService {
    constructor(eventBus) {
        this.eventBus = eventBus;
        this.setupEventHandlers();
    }
    
    setupEventHandlers() {
        // Listen to multiple event types
        this.eventBus.subscribe('user.registered', this.trackUserRegistration.bind(this));
        this.eventBus.subscribe('order.created', this.trackOrderCreated.bind(this));
    }
    
    trackUserRegistration(event) {
        console.log(`📊 Analytics: New user from ${event.data.registrationSource}`);
    }
    
    trackOrderCreated(event) {
        console.log(`📊 Analytics: Order value $${event.data.total}`);
    }
}

// Example usage
const eventBus = new EventBus();
const userService = new UserService(eventBus);
const analyticsService = new AnalyticsService(eventBus);

// Simulate user registration
userService.registerUser('jane@example.com', 'Jane Doe');

// Simulate order (analytics service will pick this up too)
eventBus.publish('order.created', {