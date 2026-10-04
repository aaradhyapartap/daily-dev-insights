# 📌 Event-driven architecture
*October 04, 2026 · Daily Dev Insight*

## 🧠 Overview

Event-driven architecture (EDA) is a design paradigm where components communicate by producing and consuming events rather than making direct calls to each other. Think of it like a sophisticated pub-sub notification system where services don't need to know about each other's existence—they just react to things happening in the system. When a user places an order, instead of the order service directly calling inventory, payments, and shipping services, it simply emits an "OrderPlaced" event, and interested services respond independently.

This decoupling is EDA's superpower. Services become autonomous units that can be deployed, scaled, and failed independently. Your payment service going down doesn't break order creation—it just processes payments when it comes back online. The trade-off? You're exchanging the simplicity of direct function calls for eventual consistency and the complexity of distributed message handling. Your system becomes more resilient but also harder to debug, since a single user action might trigger a cascade of events across dozens of services.

The key mental shift is moving from "what should I call next?" to "what just happened that others might care about?" This inversion of control makes systems more flexible but requires disciplined event schema management and careful monitoring. When done right, EDA enables true microservices independence; when done wrong, you get an unmaintainable mess of mystery events flying around.

## 💡 Key Concepts

- **Events are immutable facts**: Events represent things that have already happened ("OrderCreated", "PaymentProcessed"), not commands. They're historical records that should never change once published.

- **Producers and consumers are decoupled**: Event emitters don't know or care who's listening. New consumers can subscribe without modifying producers, enabling easy feature additions.

- **Eventual consistency over immediate consistency**: Accept that your system's state won't be synchronized instantly. An order might show as "pending payment" for a few seconds while events propagate.

- **Event stores as source of truth**: Modern EDA often uses event sourcing, where events themselves become the database. You can rebuild any state by replaying events from the beginning.

- **Dead letter queues are mandatory**: When event processing fails, you need a place to park failed events for investigation and retry, otherwise you'll lose data silently.

## �🐍 Python Example

```python
import asyncio
from dataclasses import dataclass
from typing import Callable, List
from datetime import datetime

@dataclass
class Event:
    """Immutable event structure"""
    event_type: str
    payload: dict
    timestamp: datetime = None
    
    def __post_init__(self):
        self.timestamp = self.timestamp or datetime.now()

class EventBus:
    """Simple in-memory event bus for demonstration"""
    
    def __init__(self):
        self._subscribers: dict[str, List[Callable]] = {}
    
    def subscribe(self, event_type: str, handler: Callable):
        """Register a handler for specific event type"""
        if event_type not in self._subscribers:
            self._subscribers[event_type] = []
        self._subscribers[event_type].append(handler)
        print(f"📌 Subscribed {handler.__name__} to {event_type}")
    
    async def publish(self, event: Event):
        """Publish event to all subscribers"""
        print(f"📢 Publishing: {event.event_type}")
        
        if event.event_type in self._subscribers:
            # Process all handlers concurrently
            tasks = [
                handler(event) 
                for handler in self._subscribers[event.event_type]
            ]
            await asyncio.gather(*tasks)

# Example: E-commerce order flow
bus = EventBus()

async def send_confirmation_email(event: Event):
    await asyncio.sleep(0.5)  # Simulate email API call
    print(f"✉️  Email sent to {event.payload['email']}")

async def reserve_inventory(event: Event):
    await asyncio.sleep(0.3)  # Simulate database operation
    print(f"📦 Reserved {event.payload['items']} from inventory")

async def process_payment(event: Event):
    await asyncio.sleep(0.7)  # Simulate payment gateway
    print(f"💳 Charged ${event.payload['total']}")
    # Payment success triggers another event
    await bus.publish(Event("PaymentProcessed", {"order_id": event.payload['order_id']}))

async def ship_order(event: Event):
    print(f"🚚 Shipping order {event.payload['order_id']}")

# Wire up event handlers
bus.subscribe("OrderPlaced", send_confirmation_email)
bus.subscribe("OrderPlaced", reserve_inventory)
bus.subscribe("OrderPlaced", process_payment)
bus.subscribe("PaymentProcessed", ship_order)

# Simulate order placement
async def main():
    order_event = Event(
        "OrderPlaced",
        {
            "order_id": "ORD-123",
            "email": "customer@example.com",
            "items": ["laptop", "mouse"],
            "total": 1299.99
        }
    )
    await bus.publish(order_event)

asyncio.run(main())
```

## 🟨 JavaScript Example

```javascript
const EventEmitter = require('events');

// Event-driven order processing system
class OrderEventBus extends EventEmitter {
  constructor() {
    super();
    this.setMaxListeners(20); // Allow many subscribers
  }

  publishEvent(eventType, payload) {
    const event = {
      type: eventType,
      payload,
      timestamp: new Date().toISOString(),
      id: Math.random().toString(36).substr(2, 9)
    };
    
    console.log(`📢 Event published: ${eventType} [${event.id}]`);
    this.emit(eventType, event);
    
    // Also emit to wildcard listeners
    this.emit('*', event);
  }
}

const bus = new OrderEventBus();

// Service 1: Inventory management
bus.on('OrderPlaced', (event) => {
  console.log(`📦 [Inventory] Reserving items:`, event.payload.items);
  
  setTimeout(() => {
    bus.publishEvent('InventoryReserved', {
      orderId: event.payload.orderId,
      items: event.payload.items
    });
  }, 300);
});

// Service 2: Payment processor
bus.on('InventoryReserved', (event) => {
  console.log(`💳 [Payment] Processing payment for order ${event.payload.orderId}`);
  
  setTimeout(() => {
    const success = Math.random() > 0.2; // 80% success rate
    
    if (success) {
      bus.publishEvent('PaymentCompleted', { orderId: event.payload.orderId });
    } else {
      bus.publishEvent('PaymentFailed', { 
        orderId: event.payload.orderId,
        reason: 'Insufficient funds'
      });
    }
  }, 500);
});

// Service 3: Shipping
bus.on('PaymentCompleted', (event) => {
  console.log(`🚚 [Shipping] Creating shipment for ${event.payload.orderId}`);
});

// Service 4: Notifications
bus.on('PaymentFailed', (event) => {
  console.log(`⚠️  [Notification] Alerting customer about payment failure`);
});

// Audit logger - subscribes to ALL events
bus.on('*', (event) => {
  console.log(`📝 [Audit] Logged: ${event.type} at ${event.timestamp}`);
});

// Trigger the workflow
bus.publishEvent('OrderPlaced', {
  orderId: 'ORD-456',
  customerId: 'CUST-789',
  items: ['keyboard', 'monitor'],
  total: 599.99
});
```

## ⚖️ When To Use / When To Avoid

**✅ Use Event-Driven Architecture When:**
- You need to scale services independently based on different