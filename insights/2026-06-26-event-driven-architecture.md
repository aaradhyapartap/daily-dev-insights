# 📌 Event-driven architecture
*June 26, 2026 · Daily Dev Insight*

## 🧠 Overview

Event-driven architecture (EDA) is a design paradigm where components communicate by emitting and responding to events rather than making direct, synchronous calls to each other. Think of it like a newsroom: reporters don't personally call every subscriber when a story breaks—they publish the story, and interested subscribers receive it. This decoupling is powerful because producers don't need to know who's consuming their events, and consumers can react to events without the producer waiting around for them.

The beauty of EDA lies in its scalability and flexibility. When your e-commerce app processes an order, you might need to update inventory, charge a credit card, send confirmation emails, trigger fulfillment, and update analytics. In a traditional architecture, that's a brittle chain of synchronous calls. With events, the order service simply publishes an "OrderPlaced" event, and each downstream service independently reacts. If you add a new feature like fraud detection tomorrow, you just add another listener—no changes to existing code.

However, EDA isn't a silver bullet. The distributed nature introduces complexity: debugging becomes harder when data flows through asynchronous event streams, eventual consistency replaces immediate consistency, and you need robust infrastructure for event delivery guarantees. Use it when the benefits of decoupling outweigh the operational overhead.

## 💡 Key Concepts

- **Events vs Messages**: Events announce something that happened (past tense: "UserRegistered"), while messages are commands telling a service to do something. Events are facts; they can have multiple consumers who each interpret them differently.

- **Event Brokers**: Infrastructure components (like Kafka, RabbitMQ, or AWS EventBridge) that receive events from producers and route them to consumers. They provide durability, ordering guarantees, and replay capabilities.

- **Eventual Consistency**: In EDA, data across services becomes consistent *eventually*, not immediately. Your order service might confirm an order before the email service sends the receipt—and that's okay.

- **Event Sourcing**: An advanced pattern where you store every state change as an event, making the event log the source of truth. You can rebuild any state by replaying events, which is invaluable for auditing and debugging.

- **Choreography vs Orchestration**: Choreography means services independently react to events (decentralized), while orchestration has a central coordinator directing the workflow. Most EDA uses choreography for better decoupling.

## 🐍 Python Example

```python
import asyncio
from typing import Callable, Dict, List
from dataclasses import dataclass
from datetime import datetime

@dataclass
class Event:
    """Base event with metadata"""
    event_type: str
    data: Dict
    timestamp: datetime = None
    
    def __post_init__(self):
        self.timestamp = self.timestamp or datetime.now()

class EventBus:
    """Simple in-memory event bus implementation"""
    
    def __init__(self):
        self._subscribers: Dict[str, List[Callable]] = {}
    
    def subscribe(self, event_type: str, handler: Callable):
        """Register a handler for an event type"""
        if event_type not in self._subscribers:
            self._subscribers[event_type] = []
        self._subscribers[event_type].append(handler)
    
    async def publish(self, event: Event):
        """Publish an event to all subscribers"""
        handlers = self._subscribers.get(event.event_type, [])
        print(f"📢 Publishing {event.event_type} to {len(handlers)} handlers")
        
        # Run handlers concurrently
        await asyncio.gather(*[handler(event) for handler in handlers])

# Example application: order processing system
async def send_confirmation_email(event: Event):
    """Email service handler"""
    await asyncio.sleep(0.5)  # Simulate email sending
    print(f"✉️  Email sent to {event.data['user_email']}")

async def update_inventory(event: Event):
    """Inventory service handler"""
    await asyncio.sleep(0.3)
    print(f"📦 Inventory reduced for order {event.data['order_id']}")

async def trigger_analytics(event: Event):
    """Analytics service handler"""
    print(f"📊 Analytics recorded: ${event.data['amount']}")

async def main():
    bus = EventBus()
    
    # Services register their interest in events
    bus.subscribe("OrderPlaced", send_confirmation_email)
    bus.subscribe("OrderPlaced", update_inventory)
    bus.subscribe("OrderPlaced", trigger_analytics)
    
    # Order service publishes event without knowing who's listening
    order_event = Event(
        event_type="OrderPlaced",
        data={
            "order_id": "ORD-12345",
            "user_email": "customer@example.com",
            "amount": 99.99
        }
    )
    
    await bus.publish(order_event)
    print("✅ Order service is done (handlers run independently)")

if __name__ == "__main__":
    asyncio.run(main())
```

## 🟨 JavaScript Example

```javascript
const EventEmitter = require('events');

/**
 * Domain-specific event bus for order processing
 */
class OrderEventBus extends EventEmitter {
  constructor() {
    super();
    this.setMaxListeners(50); // Allow many subscribers
  }

  /**
   * Type-safe event publishing with logging
   */
  publishOrderEvent(eventType, data) {
    const event = {
      type: eventType,
      data,
      timestamp: new Date().toISOString(),
      eventId: Math.random().toString(36).substr(2, 9)
    };
    
    console.log(`📢 Publishing ${eventType} [${event.eventId}]`);
    this.emit(eventType, event);
    
    // Also publish to wildcard listeners
    this.emit('*', event);
    
    return event.eventId;
  }
}

// Initialize the event bus
const orderBus = new OrderEventBus();

// Service 1: Email notification service
orderBus.on('order.placed', async (event) => {
  console.log(`✉️  [Email Service] Sending confirmation to ${event.data.email}`);
  // Simulate async email sending
  await new Promise(resolve => setTimeout(resolve, 500));
  console.log(`✉️  [Email Service] Confirmation sent for order ${event.data.orderId}`);
});

// Service 2: Warehouse fulfillment service
orderBus.on('order.placed', (event) => {
  console.log(`📦 [Warehouse] Processing order ${event.data.orderId}`);
  console.log(`📦 [Warehouse] Items: ${event.data.items.join(', ')}`);
});

// Service 3: Fraud detection (added later, no changes to existing code!)
orderBus.on('order.placed', (event) => {
  const riskScore = Math.random();
  if (riskScore > 0.8) {
    console.log(`⚠️  [Fraud] High-risk order detected: ${event.data.orderId}`);
    orderBus.publishOrderEvent('order.flagged', { 
      orderId: event.data.orderId,
      reason: 'high_risk_score' 
    });
  }
});

// Audit logger subscribes to ALL events
orderBus.on('*', (event) => {
  console.log(`📝 [Audit] Logged event: ${event.type} at ${event.timestamp}`);
});

// Simulate an order being placed
console.log('🚀 Application started\n');

orderBus.publishOrderEvent('order.placed', {
  orderId: 'ORD-98765',
  email: 'alice@example.com',
  items: ['Laptop', 'Mouse', 'Keyboard'],
  total: 1299.99
});

console.log('\n✅ Order service returned immediately (handlers still processing)');
```

## ⚖️ When To Use / When To Avoid

**✅ Use Event-Driven Architecture When:**