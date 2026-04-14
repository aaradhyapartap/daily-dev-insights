# 📌 Design patterns: Observer and Publisher/Subscriber
*April 14, 2026 · Daily Dev Insight*

## 🧠 Overview

The Observer and Publisher/Subscriber patterns are often confused because they both deal with notification systems, but they solve different architectural problems. The Observer pattern creates a direct relationship between subjects and observers, while Pub/Sub introduces a message broker that decouples publishers from subscribers entirely. Think of Observer as a direct phone call—you know exactly who you're talking to. Pub/Sub is more like posting on social media—you broadcast a message, and anyone interested can tune in.

In practice, Observer works brilliantly for tight-knit systems where components need immediate, synchronous updates (like UI frameworks). Pub/Sub shines in distributed systems where you want loose coupling and don't care who's listening. The key insight? Observer optimizes for simplicity and direct communication, while Pub/Sub optimizes for scalability and independence.

## 💡 Key Concepts

• **Coupling vs Decoupling**: Observer creates 1:N relationships with direct references, while Pub/Sub uses an intermediary broker for complete decoupling
• **Synchronous vs Asynchronous**: Observer typically operates synchronously, while Pub/Sub naturally supports async messaging patterns  
• **Error Propagation**: In Observer, one failing observer can crash the subject; Pub/Sub isolates failures through the broker
• **Discovery**: Observer requires subjects to know their observers; Pub/Sub allows dynamic subscription without prior knowledge
• **Scalability**: Pub/Sub scales better across network boundaries and supports message queuing, filtering, and persistence

## 🐍 Python Example

```python
from abc import ABC, abstractmethod
from typing import List, Dict, Any
import asyncio
from collections import defaultdict

# Observer Pattern - Direct notification
class NewsAgency:
    def __init__(self):
        self._observers: List[NewsChannel] = []
        self._news = ""
    
    def attach(self, observer):
        self._observers.append(observer)
    
    def detach(self, observer):
        self._observers.remove(observer)
    
    def set_news(self, news: str):
        self._news = news
        self._notify_all()
    
    def _notify_all(self):
        for observer in self._observers:
            observer.update(self._news)

class NewsChannel:
    def __init__(self, name: str):
        self.name = name
    
    def update(self, news: str):
        print(f"{self.name} received: {news}")

# Pub/Sub Pattern - Broker-mediated
class EventBroker:
    def __init__(self):
        self._subscribers: Dict[str, List] = defaultdict(list)
    
    async def subscribe(self, topic: str, callback):
        self._subscribers[topic].append(callback)
    
    async def publish(self, topic: str, data: Any):
        for callback in self._subscribers[topic]:
            try:
                await callback(data)  # Async and error-isolated
            except Exception as e:
                print(f"Subscriber error: {e}")

# Usage example
async def main():
    # Observer pattern usage
    agency = NewsAgency()
    cnn = NewsChannel("CNN")
    bbc = NewsChannel("BBC")
    
    agency.attach(cnn)
    agency.attach(bbc)
    agency.set_news("Breaking: New Python version released!")
    
    # Pub/Sub pattern usage
    broker = EventBroker()
    
    async def user_signup_handler(user_data):
        print(f"Sending welcome email to {user_data['email']}")
    
    async def analytics_handler(user_data):
        print(f"Tracking signup event for user {user_data['id']}")
    
    await broker.subscribe("user.signup", user_signup_handler)
    await broker.subscribe("user.signup", analytics_handler)
    await broker.publish("user.signup", {"id": 123, "email": "user@example.com"})

# asyncio.run(main())
```

## 🟨 JavaScript Example

```javascript
// Observer Pattern - Traditional implementation
class Stock {
  constructor(symbol, price) {
    this.symbol = symbol;
    this.price = price;
    this.investors = [];
  }
  
  attach(investor) {
    this.investors.push(investor);
  }
  
  detach(investor) {
    this.investors = this.investors.filter(inv => inv !== investor);
  }
  
  notify() {
    this.investors.forEach(investor => investor.update(this));
  }
  
  setPrice(price) {
    this.price = price;
    this.notify();
  }
}

class Investor {
  constructor(name) {
    this.name = name;
  }
  
  update(stock) {
    console.log(`${this.name} notified: ${stock.symbol} is now $${stock.price}`);
  }
}

// Pub/Sub Pattern - Event-driven with EventEmitter
const EventEmitter = require('events');

class MessageBus extends EventEmitter {
  constructor() {
    super();
    this.setMaxListeners(100); // Prevent memory leak warnings
  }
  
  subscribe(event, handler) {
    this.on(event, handler);
  }
  
  unsubscribe(event, handler) {
    this.off(event, handler);
  }
  
  publish(event, data) {
    this.emit(event, data);
  }
}

// Real-world usage example
const messageBus = new MessageBus();

// Multiple services can subscribe independently
messageBus.subscribe('order.created', (order) => {
  console.log(`Inventory: Reserving items for order ${order.id}`);
});

messageBus.subscribe('order.created', (order) => {
  console.log(`Billing: Processing payment for $${order.total}`);
});

messageBus.subscribe('order.created', (order) => {
  console.log(`Notifications: Sending confirmation to ${order.customerEmail}`);
});

// Observer example
const tesla = new Stock('TSLA', 800);
const warren = new Investor('Warren');
const cathie = new Investor('Cathie');

tesla.attach(warren);
tesla.attach(cathie);
tesla.setPrice(850); // Both investors get notified immediately

// Pub/Sub example
messageBus.publish('order.created', {
  id: 'ORD-001',
  total: 299.99,
  customerEmail: 'customer@example.com'
});
```

## ⚖️ When To Use / When To Avoid

**Use Observer When:**
• Building UI components that need immediate updates (React state, Vue reactivity)
• Working within a single process/application boundary
• You need guaranteed synchronous execution order
• Simple notification chains with known participants

**Use Pub/Sub When:**
• Building microservices that shouldn't know about each other
• Implementing event sourcing or CQRS patterns
• You need message persistence, filtering, or replay capabilities
• Handling high-volume events that can be processed asynchronously

**Avoid Observer When:**
• You have many observers that could fail and crash the system
• Network boundaries are involved (use Pub/Sub instead)

**Avoid Pub/Sub When:**
• You need immediate, synchronous responses
• The overhead of a message broker isn't justified
• Simple, direct communication is sufficient

## 📚 Further Reading

• [MDN Event-driven Programming Guide](https://developer.mozilla.org/en-US/docs/Web/Events) - Excellent coverage of DOM events and custom event patterns
• [Python asyncio Documentation](https://docs.python.org/3/library/asyncio.html) - Deep dive into async patterns that work well with Pub/Sub
• [Martin Fowler on Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html) - Architectural patterns and trade-offs
• [Redis Pub/Sub Documentation](https://redis.io/docs/manual/pubsub/) - Production-ready message broker implementation
• [RxJS Observable Patterns](https://rxjs.dev