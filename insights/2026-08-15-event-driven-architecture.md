# 📌 Event-driven architecture
*August 15, 2026 · Daily Dev Insight*

## 🧠 Overview

Event-driven architecture (EDA) is a design paradigm where components communicate by emitting and responding to events rather than directly calling each other. Think of it as a pub-sub model on steroids: producers fire events when something interesting happens, and consumers react independently without the producer knowing or caring who's listening. This decoupling is EDA's superpower—it lets you build systems that are flexible, scalable, and easier to evolve over time.

The beauty of EDA shines when you're dealing with complex workflows, microservices, or real-time data processing. Instead of tightly coupling service A to service B with synchronous API calls, you let them communicate through events. When a user places an order, you emit an "OrderPlaced" event. The inventory service, notification service, and analytics service can all react to that event independently, at their own pace, without creating a brittle dependency chain.

However, EDA isn't free lunch. You're trading the complexity of tight coupling for the complexity of distributed state management, eventual consistency, and debugging flows that span multiple services. The key is knowing when this trade-off makes sense—which is more often than you might think in modern cloud-native applications.

## 💡 Key Concepts

- **Event Emitters & Listeners**: Producers emit events when state changes occur; consumers subscribe to event types they care about without direct knowledge of each other
- **Asynchronous Communication**: Events are typically processed asynchronously, allowing services to remain responsive and handle varying loads gracefully
- **Event Store/Log**: A durable record of all events (like Kafka, RabbitMQ, or AWS EventBridge) serves as the backbone, enabling replay, audit trails, and recovery
- **Eventual Consistency**: Since events propagate asynchronously, different parts of your system may have slightly stale views of the world—this is a feature, not a bug
- **Choreography vs Orchestration**: Events enable choreography (services react independently) as opposed to orchestration (a central coordinator directs the workflow)

## 🐍 Python Example

```python
from dataclasses import dataclass
from typing import Callable, Dict, List
from datetime import datetime
import json

# Simple in-memory event bus for demonstration
class EventBus:
    def __init__(self):
        self._subscribers: Dict[str, List[Callable]] = {}
    
    def subscribe(self, event_type: str, handler: Callable):
        """Register a handler for a specific event type"""
        if event_type not in self._subscribers:
            self._subscribers[event_type] = []
        self._subscribers[event_type].append(handler)
    
    def publish(self, event_type: str, data: dict):
        """Publish an event to all registered handlers"""
        event = {
            "type": event_type,
            "timestamp": datetime.utcnow().isoformat(),
            "data": data
        }
        print(f"📤 Publishing: {event_type}")
        
        for handler in self._subscribers.get(event_type, []):
            try:
                handler(event)
            except Exception as e:
                print(f"❌ Handler error: {e}")

# Example services
def send_confirmation_email(event):
    """Email service reacts to order events"""
    order_id = event["data"]["order_id"]
    print(f"📧 Sending confirmation email for order #{order_id}")

def update_inventory(event):
    """Inventory service decrements stock"""
    items = event["data"]["items"]
    print(f"📦 Updating inventory: {items}")

def track_analytics(event):
    """Analytics service tracks metrics"""
    total = event["data"]["total"]
    print(f"📊 Recording sale: ${total}")

# Wire up the event-driven system
bus = EventBus()
bus.subscribe("order.placed", send_confirmation_email)
bus.subscribe("order.placed", update_inventory)
bus.subscribe("order.placed", track_analytics)

# Simulate an order being placed
bus.publish("order.placed", {
    "order_id": "ORD-12345",
    "items": ["laptop", "mouse"],
    "total": 1299.99
})
```

## 🟨 JavaScript Example

```javascript
const EventEmitter = require('events');

// Domain event classes
class OrderPlacedEvent {
  constructor(orderId, customerId, total) {
    this.orderId = orderId;
    this.customerId = customerId;
    this.total = total;
    this.timestamp = new Date();
  }
}

// Event-driven order service
class OrderService extends EventEmitter {
  placeOrder(customerId, items) {
    const orderId = `ORD-${Date.now()}`;
    const total = items.reduce((sum, item) => sum + item.price, 0);
    
    console.log(`💳 Processing order ${orderId}...`);
    
    // Emit event instead of directly calling other services
    this.emit('order.placed', new OrderPlacedEvent(orderId, customerId, total));
    
    return orderId;
  }
}

// Independent service listeners
class NotificationService {
  handleOrderPlaced(event) {
    console.log(`📧 Sending notification to customer ${event.customerId}`);
    // Async notification logic here
  }
}

class InventoryService {
  handleOrderPlaced(event) {
    console.log(`📦 Reserving inventory for order ${event.orderId}`);
    // Async inventory update here
  }
}

class LoyaltyService {
  handleOrderPlaced(event) {
    const points = Math.floor(event.total / 10);
    console.log(`⭐ Adding ${points} loyalty points`);
    // Async points calculation here
  }
}

// Wire everything together
const orderService = new OrderService();
const notificationService = new NotificationService();
const inventoryService = new InventoryService();
const loyaltyService = new LoyaltyService();

orderService.on('order.placed', (e) => notificationService.handleOrderPlaced(e));
orderService.on('order.placed', (e) => inventoryService.handleOrderPlaced(e));
orderService.on('order.placed', (e) => loyaltyService.handleOrderPlaced(e));

// Place an order - watch the magic happen
orderService.placeOrder('CUST-789', [
  { name: 'Keyboard', price: 129.99 },
  { name: 'Monitor', price: 399.99 }
]);
```

## ⚖️ When To Use / When To Avoid

**✅ Use Event-Driven Architecture When:**
- You need to decouple microservices and reduce inter-service dependencies
- Multiple downstream systems need to react to the same state changes
- You're building real-time features (notifications, dashboards, live updates)
- You need audit trails and event replay capabilities
- Scalability and resilience are critical requirements

**❌ Avoid Event-Driven Architecture When:**
- You have simple CRUD operations with straightforward request-response flows
- Strong consistency is non-negotiable (banking transactions, inventory counts)
- Your team lacks experience with distributed systems debugging
- You're building an MVP and need to ship fast with minimal infrastructure
- End-to-end transaction tracing and debugging are already pain points

## 📚 Further Reading

- [AWS Event-Driven Architecture Patterns](https://aws.amazon.com/event-driven-architecture/) - Comprehensive guide to EDA patterns in cloud environments
- [Martin Fowler: Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html) - Classic overview from the godfather of software patterns
- [Kafka Documentation: Event Streaming Fundamentals](https://kafka.apache.org/documentation/) - Deep dive into the most popular event streaming platform
- [Node.js EventEmitter Documentation](https://nodejs.org/api/events.html) - Official docs for Node's built-in event system
- [Python asyncio Events](https://docs.python.org/3/library/asyncio-sync.html#asyncio.Event) - Python's async event primitives for