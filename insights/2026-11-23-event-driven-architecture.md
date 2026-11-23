# 📌 Event-driven architecture
*November 23, 2026 · Daily Dev Insight*

## 🧠 Overview

Event-driven architecture (EDA) is a design paradigm where system components communicate by producing and consuming events rather than directly calling each other. Think of it like a newsroom: reporters publish stories (events), and various departments—social media, print, web—react to those stories independently. No one waits for everyone else to finish their work, and new departments can start consuming stories without the reporters changing how they work.

The beauty of EDA lies in its decoupling. When Service A emits an "OrderPlaced" event, it doesn't know or care who's listening. Maybe it's the inventory service, the email service, and the analytics service. Maybe tomorrow you'll add a fraud detection service. Service A remains blissfully unaware, and that's exactly the point. This loose coupling makes systems more resilient, scalable, and easier to evolve over time.

However, EDA isn't free lunch. You're trading direct, synchronous complexity for distributed, asynchronous complexity. Debugging becomes harder when a request flows through five services via events. You need solid observability, idempotent handlers (because events might arrive twice), and careful thinking about event schemas and versioning. But when you need to scale independent concerns or build truly pluggable systems, EDA is often worth the investment.

## 💡 Key Concepts

- **Events vs Commands**: Events represent something that *happened* ("OrderPlaced"), while commands represent something you *want* to happen ("PlaceOrder"). Events are past-tense facts that multiple consumers can react to independently.

- **Event Bus/Broker**: The middleware (RabbitMQ, Kafka, Redis Streams, AWS EventBridge) that handles event routing. It decouples producers from consumers and handles delivery guarantees, ordering, and backpressure.

- **At-least-once delivery**: Most event systems guarantee events arrive at least once, meaning duplicates are possible. Your handlers must be idempotent—processing the same event twice should have the same effect as processing it once.

- **Event sourcing vs Event-driven**: Event-driven is about communication patterns. Event sourcing is a storage pattern where you store state changes as events. They're complementary but distinct concepts.

- **Choreography vs Orchestration**: Choreography means services react to events autonomously (distributed decision-making). Orchestration means a central service coordinates the workflow. Both have their place; choreography scales better but is harder to reason about.

## 🐍 Python Example

```python
import asyncio
from dataclasses import dataclass
from typing import Callable, List
import json
from datetime import datetime

@dataclass
class Event:
    """Represents a domain event"""
    event_type: str
    payload: dict
    timestamp: str = None
    
    def __post_init__(self):
        self.timestamp = self.timestamp or datetime.utcnow().isoformat()

class EventBus:
    """Simple in-memory event bus for demonstration"""
    def __init__(self):
        self._subscribers: dict[str, List[Callable]] = {}
    
    def subscribe(self, event_type: str, handler: Callable):
        """Register a handler for specific event type"""
        if event_type not in self._subscribers:
            self._subscribers[event_type] = []
        self._subscribers[event_type].append(handler)
        print(f"✓ Subscribed {handler.__name__} to {event_type}")
    
    async def publish(self, event: Event):
        """Publish event to all subscribers"""
        print(f"\n📤 Publishing: {event.event_type}")
        handlers = self._subscribers.get(event.event_type, [])
        
        # Execute all handlers concurrently
        tasks = [handler(event) for handler in handlers]
        await asyncio.gather(*tasks, return_exceptions=True)

# Example usage: Order processing system
bus = EventBus()

async def send_confirmation_email(event: Event):
    """Email service handler"""
    await asyncio.sleep(0.5)  # Simulate I/O
    print(f"  📧 Email sent to {event.payload['customer_email']}")

async def update_inventory(event: Event):
    """Inventory service handler"""
    await asyncio.sleep(0.3)
    print(f"  📦 Inventory reduced for order {event.payload['order_id']}")

async def track_analytics(event: Event):
    """Analytics service handler"""
    print(f"  📊 Analytics tracked: ${event.payload['amount']}")

# Wire up subscribers
bus.subscribe("order.placed", send_confirmation_email)
bus.subscribe("order.placed", update_inventory)
bus.subscribe("order.placed", track_analytics)

# Simulate order placement
async def main():
    order_event = Event(
        event_type="order.placed",
        payload={
            "order_id": "ORD-123",
            "customer_email": "user@example.com",
            "amount": 99.99
        }
    )
    await bus.publish(order_event)

# asyncio.run(main())  # Uncomment to run
```

## 🟨 JavaScript Example

```javascript
// Simple event-driven system using Node.js EventEmitter
const EventEmitter = require('events');

class OrderEventBus extends EventEmitter {
  constructor() {
    super();
    this.eventLog = [];
  }

  publishEvent(eventType, payload) {
    const event = {
      eventType,
      payload,
      timestamp: new Date().toISOString(),
      eventId: Math.random().toString(36).substring(7)
    };
    
    this.eventLog.push(event);
    console.log(`\n📤 Publishing: ${eventType} [${event.eventId}]`);
    
    // Emit with both specific and wildcard listeners
    this.emit(eventType, event);
    this.emit('*', event); // Catch-all listener
    
    return event;
  }
}

// Initialize event bus
const orderBus = new OrderEventBus();

// Service 1: Email notifications
orderBus.on('order.placed', async (event) => {
  // Simulate async operation
  setTimeout(() => {
    console.log(`  📧 Email sent to ${event.payload.customerEmail}`);
  }, 500);
});

// Service 2: Payment processing
orderBus.on('order.placed', async (event) => {
  setTimeout(() => {
    console.log(`  💳 Payment processed: $${event.payload.amount}`);
    
    // Emit follow-up event
    orderBus.publishEvent('payment.completed', {
      orderId: event.payload.orderId,
      transactionId: 'TXN-' + Date.now()
    });
  }, 300);
});

// Service 3: Shipping preparation
orderBus.on('payment.completed', (event) => {
  console.log(`  📦 Shipping label created for ${event.payload.orderId}`);
});

// Audit logger: catches all events
orderBus.on('*', (event) => {
  console.log(`  📝 [AUDIT] ${event.eventType} logged`);
});

// Example usage
orderBus.publishEvent('order.placed', {
  orderId: 'ORD-456',
  customerEmail: 'customer@example.com',
  amount: 149.99,
  items: ['Widget A', 'Widget B']
});

// Demonstrate event log
setTimeout(() => {
  console.log(`\n📊 Total events processed: ${orderBus.eventLog.length}`);
}, 1000);
```

## ⚖️ When To Use / When To Avoid

**Use Event-Driven Architecture When:**
- You need to scale independent services differently (email service vs payment service)
- Multiple services need to react to the same business event
- You want to add new functionality without modifying existing services
- You're building notification systems, audit logs, or real-time dashboards
- You need resilience—services can catch up on events after downtime

**Avoid Event-Driven Architecture When:**
- You need immediate, synchronous responses (use direct API calls)