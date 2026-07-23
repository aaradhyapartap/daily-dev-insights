# 📌 Design patterns: Observer and Publisher/Subscriber
*July 23, 2026 · Daily Dev Insight*

## 🧠 Overview

The Observer and Pub/Sub patterns are often confused—and for good reason. They both solve the same fundamental problem: decoupling components that need to react to state changes. But here's the key distinction that'll save you from architectural headaches: **Observer is a direct relationship, while Pub/Sub introduces a middleman (message broker/event bus)**.

In the Observer pattern, subjects maintain a list of observers and notify them directly when state changes. Think of it like a newsletter where the publisher knows exactly who their subscribers are and emails them personally. It's tight coupling with a polite interface. Pub/Sub, on the other hand, is like posting on a bulletin board—publishers don't know or care who's listening, and subscribers don't know who's posting. This extra layer of indirection is both its superpower and its complexity tax.

The real world rarely presents you with a textbook choice between the two. Modern frameworks like Redux, RxJS, and even the DOM's event system blur these lines. What matters is understanding the trade-offs: Observer gives you simplicity and directness at the cost of coupling; Pub/Sub gives you ultimate flexibility at the cost of harder debugging and potential message delivery guarantees to worry about.

## 💡 Key Concepts

- **Direct vs. Indirect Coupling**: Observer maintains references between subject and observers; Pub/Sub uses an event channel that neither publisher nor subscriber knows about each other
- **Synchronous vs. Asynchronous**: Observer typically notifies synchronously; Pub/Sub often (but not always) implies async message delivery
- **Filtering & Routing**: Pub/Sub naturally supports topic-based subscriptions and message filtering; Observer usually receives all notifications
- **Scalability**: Pub/Sub shines in distributed systems where you might have multiple publishers and dynamic subscribers across services
- **Debugging Complexity**: Observer has a clear call stack; Pub/Sub can feel like "action at a distance" when tracing event flows

## 🐍 Python Example

```python
from abc import ABC, abstractmethod
from typing import List, Dict, Any

# Observer Pattern Implementation
class Observer(ABC):
    @abstractmethod
    def update(self, data: Dict[str, Any]) -> None:
        pass

class StockTicker:
    """Subject that maintains and notifies observers directly"""
    def __init__(self, symbol: str):
        self.symbol = symbol
        self._price = 0.0
        self._observers: List[Observer] = []
    
    def attach(self, observer: Observer) -> None:
        self._observers.append(observer)
    
    def detach(self, observer: Observer) -> None:
        self._observers.remove(observer)
    
    def set_price(self, price: float) -> None:
        self._price = price
        self._notify()
    
    def _notify(self) -> None:
        # Direct, synchronous notification
        for observer in self._observers:
            observer.update({"symbol": self.symbol, "price": self._price})

class PriceAlert(Observer):
    def __init__(self, threshold: float):
        self.threshold = threshold
    
    def update(self, data: Dict[str, Any]) -> None:
        if data["price"] > self.threshold:
            print(f"🚨 Alert! {data['symbol']} hit ${data['price']}")

class PriceLogger(Observer):
    def update(self, data: Dict[str, Any]) -> None:
        print(f"📊 Logged: {data['symbol']} = ${data['price']}")

# Usage
ticker = StockTicker("AAPL")
alert = PriceAlert(threshold=150.0)
logger = PriceLogger()

ticker.attach(alert)
ticker.attach(logger)
ticker.set_price(155.0)  # Both observers notified directly
```

## 🟨 JavaScript Example

```javascript
// Pub/Sub Pattern Implementation with Event Bus
class EventBus {
  constructor() {
    this.subscribers = new Map();
  }

  // Publishers don't know who's listening
  publish(event, data) {
    if (!this.subscribers.has(event)) return;
    
    // Async notification - decoupled execution
    this.subscribers.get(event).forEach(callback => {
      setTimeout(() => callback(data), 0);
    });
  }

  // Subscribers don't know who's publishing
  subscribe(event, callback) {
    if (!this.subscribers.has(event)) {
      this.subscribers.set(event, []);
    }
    this.subscribers.get(event).push(callback);
    
    // Return unsubscribe function
    return () => {
      const callbacks = this.subscribers.get(event);
      const index = callbacks.indexOf(callback);
      if (index > -1) callbacks.splice(index, 1);
    };
  }
}

// Usage - complete decoupling
const eventBus = new EventBus();

// Subscriber 1 - doesn't know about other subscribers
const unsubscribe = eventBus.subscribe('user:login', (user) => {
  console.log(`📧 Sending welcome email to ${user.email}`);
});

// Subscriber 2 - independent concern
eventBus.subscribe('user:login', (user) => {
  console.log(`📊 Analytics: User ${user.id} logged in`);
});

// Publisher - doesn't know who's listening
function loginUser(credentials) {
  const user = { id: 123, email: 'dev@example.com' };
  eventBus.publish('user:login', user);
}

loginUser({ username: 'developer' });
// Both handlers execute asynchronously, fully decoupled
```

## ⚖️ When To Use / When To Avoid

**Use Observer when:**
- You have a small, known set of observers
- Synchronous notification is acceptable/desired
- You're working within a single process/module
- You want simple, traceable call stacks

**Use Pub/Sub when:**
- You need true decoupling across modules/services
- Dynamic subscription/unsubscription is important
- You're building event-driven architectures
- Multiple publishers might emit the same event type

**Avoid both when:**
- Simple callbacks or promises suffice (don't over-architect!)
- You need guaranteed delivery order (use queues instead)
- Debugging difficulty outweighs decoupling benefits
- Your team isn't comfortable with event-driven thinking

## 📚 Further Reading

- [Observer Pattern - Refactoring Guru](https://refactoring.guru/design-patterns/observer) — Excellent visual explanations and multi-language examples
- [The Many Faces of Publish/Subscribe](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget) — MDN's EventTarget API, the web's built-in pub/sub
- [Python's Built-in Observer: weakref callbacks](https://docs.python.org/3/library/weakref.html) — How Python handles memory-safe observation
- [RxJS Documentation](https://rxjs.dev/guide/overview) — The ultimate reactive programming library that extends these patterns
- [Event-Driven Architecture on AWS](https://aws.amazon.com/event-driven-architecture/) — Pub/Sub at scale with real infrastructure

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*