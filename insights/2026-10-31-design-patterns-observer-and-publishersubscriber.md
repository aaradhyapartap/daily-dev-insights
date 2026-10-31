# 📌 Design patterns: Observer and Publisher/Subscriber
*October 31, 2026 · Daily Dev Insight*

## 🧠 Overview

At first glance, Observer and Pub/Sub patterns seem identical—both handle one-to-many relationships where changes in one object trigger updates in others. But here's the kicker: the difference between them has profound architectural implications. The Observer pattern creates a *direct* coupling between subjects and observers, while Pub/Sub introduces a message broker or event channel that completely decouples publishers from subscribers.

Think of Observer like a newsletter where the publisher maintains a list of subscriber email addresses—they know exactly who's listening. Pub/Sub is more like posting on Twitter: you shout into the void (event bus), and whoever's listening (subscribed to that topic) hears it. You don't know who or how many. This distinction matters enormously when you're building distributed systems, microservices, or even just trying to keep your frontend components loosely coupled.

The Observer pattern shines in scenarios where you need tight control and direct relationships—think UI frameworks where a data model needs to notify specific view components. Pub/Sub excels when you need scalability, flexibility, and the ability to add/remove listeners without modifying the source. Modern reactive frameworks like RxJS blur these lines beautifully, but understanding the fundamentals helps you make better architectural decisions.

## 💡 Key Concepts

- **Coupling matters**: Observer creates compile-time dependencies between subject and observers; Pub/Sub allows runtime-only relationships through an event bus or broker
- **One-to-many notification**: Both patterns solve the same fundamental problem—broadcasting state changes to multiple interested parties efficiently
- **Push vs Pull**: Observers typically receive data pushed to them, but can also pull additional context from the subject they're directly coupled to
- **Event channels/topics**: Pub/Sub systems organize events by topic/channel, allowing selective subscription without the publisher knowing the subscriber exists
- **Language support**: Many modern languages provide built-in observable primitives (Python's `weakref`, JavaScript's `EventEmitter`, C#'s events), but Pub/Sub usually requires a library or external broker

## 🐍 Python Example

```python
from typing import Protocol, List
from dataclasses import dataclass

# Observer Pattern - Direct coupling
class Observer(Protocol):
    def update(self, temperature: float, humidity: float) -> None:
        ...

class WeatherStation:
    """Subject that maintains observer list"""
    def __init__(self):
        self._observers: List[Observer] = []
        self._temperature: float = 0.0
        self._humidity: float = 0.0
    
    def attach(self, observer: Observer) -> None:
        self._observers.append(observer)
    
    def detach(self, observer: Observer) -> None:
        self._observers.remove(observer)
    
    def notify(self) -> None:
        for observer in self._observers:
            observer.update(self._temperature, self._humidity)
    
    def set_measurements(self, temp: float, humidity: float) -> None:
        self._temperature = temp
        self._humidity = humidity
        self.notify()

class PhoneDisplay:
    def update(self, temperature: float, humidity: float) -> None:
        print(f"📱 Phone: {temperature}°C, {humidity}% humidity")

class DesktopDisplay:
    def update(self, temperature: float, humidity: float) -> None:
        print(f"🖥️  Desktop: Current conditions: {temperature}°C")

# Usage
station = WeatherStation()
phone = PhoneDisplay()
desktop = DesktopDisplay()

station.attach(phone)
station.attach(desktop)
station.set_measurements(25.5, 65)  # Both displays update
```

## 🟨 JavaScript Example

```javascript
// Pub/Sub Pattern - Decoupled via event bus
class EventBus {
  constructor() {
    this.subscribers = new Map();
  }

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

  publish(event, data) {
    if (!this.subscribers.has(event)) return;
    this.subscribers.get(event).forEach(callback => callback(data));
  }
}

// Create global event bus
const eventBus = new EventBus();

// Publishers don't know about subscribers
class PaymentProcessor {
  processPayment(amount, userId) {
    // Process payment logic...
    console.log(`💳 Processing $${amount} for user ${userId}`);
    
    // Just publish - don't care who's listening
    eventBus.publish('payment:completed', { amount, userId, timestamp: Date.now() });
  }
}

// Subscribers register independently
const unsubEmail = eventBus.subscribe('payment:completed', (data) => {
  console.log(`📧 Sending receipt email for $${data.amount}`);
});

eventBus.subscribe('payment:completed', (data) => {
  console.log(`📊 Analytics: Payment event recorded`);
});

// Usage
const processor = new PaymentProcessor();
processor.processPayment(99.99, 'user_123');
// Both subscribers receive the event, processor has no idea
```

## ⚖️ When To Use / When To Avoid

**Use Observer when:**
- You need tight coupling and direct relationships (UI components ↔ models)
- Performance is critical and you want zero intermediary overhead
- You're working within a single process/module boundary
- The number of observers is relatively stable and known

**Use Pub/Sub when:**
- Building microservices or distributed systems
- You want to add/remove listeners without modifying publishers
- Cross-cutting concerns (logging, analytics) need to tap into events
- You need message queuing, persistence, or delivery guarantees

**Avoid both when:**
- Simple callbacks or promises suffice for one-to-one relationships
- You're creating deeply nested event chains that become impossible to debug
- Synchronous execution order is critical (events make this unpredictable)

## 📚 Further Reading

- [Observer Pattern - Refactoring Guru](https://refactoring.guru/design-patterns/observer) · Excellent visual explanations with multiple language examples
- [Python `weakref` for Observers](https://docs.python.org/3/library/weakref.html) · Prevent memory leaks when observers hold references to subjects
- [Node.js EventEmitter Documentation](https://nodejs.org/api/events.html) · The built-in Pub/Sub mechanism in Node
- [RxJS Observables Guide](https://rxjs.dev/guide/observable) · Modern reactive programming that extends these patterns
- [AWS EventBridge](https://docs.aws.amazon.com/eventbridge/) · Enterprise Pub/Sub for distributed systems

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*