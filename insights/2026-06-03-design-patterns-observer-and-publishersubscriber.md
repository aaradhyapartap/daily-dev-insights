# 📌 Design patterns: Observer and Publisher/Subscriber
*June 03, 2026 · Daily Dev Insight*

## 🧠 Overview

The Observer and Publisher/Subscriber patterns are two closely related behavioral patterns that solve the fundamental problem of loose coupling between components that need to communicate. While often confused, they have subtle but important differences. The Observer pattern creates a direct one-to-many dependency where subjects maintain a list of observers and notify them directly. Publisher/Subscriber (Pub/Sub) introduces a message broker or event bus that decouples publishers from subscribers entirely.

In modern software development, these patterns are everywhere—from React's state management to Node.js EventEmitter, from database triggers to microservice architectures. The key insight is that both patterns promote the Open/Closed principle: your system becomes open for extension (new observers/subscribers) but closed for modification (existing code doesn't change). The Pub/Sub variant is particularly powerful in distributed systems where you want zero knowledge between components.

## 💡 Key Concepts

- **Loose Coupling**: Publishers/subjects don't need to know who's listening, enabling better modularity and testability
- **Dynamic Subscription**: Observers can subscribe/unsubscribe at runtime without affecting the publisher's core logic
- **Broadcast Communication**: One event can trigger multiple handlers, perfect for cross-cutting concerns like logging, analytics, and notifications
- **Event-Driven Architecture**: Both patterns enable reactive programming where components respond to state changes rather than polling
- **Separation of Concerns**: Business logic stays focused while side effects (UI updates, persistence, etc.) are handled separately

## 🐍 Python Example

```python
from abc import ABC, abstractmethod
from typing import Dict, List, Any
import threading

class EventBus:
    """Publisher/Subscriber implementation with thread safety"""
    
    def __init__(self):
        self._subscribers: Dict[str, List[callable]] = {}
        self._lock = threading.RLock()
    
    def subscribe(self, event_type: str, callback: callable):
        """Subscribe to an event type with a callback function"""
        with self._lock:
            if event_type not in self._subscribers:
                self._subscribers[event_type] = []
            self._subscribers[event_type].append(callback)
    
    def unsubscribe(self, event_type: str, callback: callable):
        """Remove a specific callback from an event type"""
        with self._lock:
            if event_type in self._subscribers:
                self._subscribers[event_type].remove(callback)
    
    def publish(self, event_type: str, data: Any = None):
        """Publish an event to all subscribers"""
        with self._lock:
            subscribers = self._subscribers.get(event_type, [])
            for callback in subscribers.copy():  # Copy to avoid modification during iteration
                try:
                    callback(data)
                except Exception as e:
                    print(f"Error in subscriber {callback.__name__}: {e}")

# Usage example with a user management system
bus = EventBus()

# Different services subscribe to user events
def send_welcome_email(user_data):
    print(f"📧 Sending welcome email to {user_data['email']}")

def update_analytics(user_data):
    print(f"📊 Recording new user signup: {user_data['id']}")

def setup_user_profile(user_data):
    print(f"👤 Creating profile for {user_data['username']}")

# Subscribe various handlers to user registration
bus.subscribe('user.registered', send_welcome_email)
bus.subscribe('user.registered', update_analytics)
bus.subscribe('user.registered', setup_user_profile)

# Publisher publishes without knowing about subscribers
bus.publish('user.registered', {
    'id': 12345,
    'username': 'newuser',
    'email': 'user@example.com'
})
```

## 🟨 JavaScript Example

```javascript
// Modern Observer pattern using ES6 classes and async operations
class StockPriceMonitor {
    constructor() {
        this.observers = new Map(); // Use Map for better performance
        this.prices = new Map();
    }
    
    // Subscribe to price changes for a specific symbol
    subscribe(symbol, observer) {
        if (!this.observers.has(symbol)) {
            this.observers.set(symbol, new Set());
        }
        this.observers.get(symbol).add(observer);
        
        // Send current price if available
        if (this.prices.has(symbol)) {
            observer.onPriceUpdate(symbol, this.prices.get(symbol));
        }
    }
    
    unsubscribe(symbol, observer) {
        const symbolObservers = this.observers.get(symbol);
        if (symbolObservers) {
            symbolObservers.delete(observer);
        }
    }
    
    async updatePrice(symbol, newPrice) {
        const oldPrice = this.prices.get(symbol);
        this.prices.set(symbol, newPrice);
        
        // Notify all observers for this symbol
        const symbolObservers = this.observers.get(symbol);
        if (symbolObservers) {
            const notifications = Array.from(symbolObservers).map(observer => 
                Promise.resolve(observer.onPriceUpdate(symbol, newPrice, oldPrice))
                    .catch(err => console.error(`Observer error:`, err))
            );
            await Promise.allSettled(notifications);
        }
    }
}

// Different observer implementations
class PortfolioTracker {
    constructor(name) {
        this.name = name;
    }
    
    onPriceUpdate(symbol, price, oldPrice) {
        const change = oldPrice ? ((price - oldPrice) / oldPrice * 100).toFixed(2) : 0;
        console.log(`📈 ${this.name}: ${symbol} = $${price} (${change}%)`);
    }
}

class AlertSystem {
    constructor(threshold) {
        this.threshold = threshold;
    }
    
    onPriceUpdate(symbol, price) {
        if (price > this.threshold) {
            console.log(`🚨 ALERT: ${symbol} exceeded threshold at $${price}`);
        }
    }
}

// Usage
const monitor = new StockPriceMonitor();
const portfolio = new PortfolioTracker('My Portfolio');
const alerts = new AlertSystem(150);

monitor.subscribe('AAPL', portfolio);
monitor.subscribe('AAPL', alerts);

// Simulate price updates
await monitor.updatePrice('AAPL', 145.50);
await monitor.updatePrice('AAPL', 152.30);
```

## ⚖️ When To Use / When To Avoid

**✅ Use When:**
- You need to decouple components that communicate frequently
- Multiple parts of your system need to react to the same events
- You're building event-driven or reactive architectures
- You want to add new behaviors without modifying existing code
- You're implementing MV* patterns (Model-View-Controller/MVVM)

**❌ Avoid When:**
- You have simple, direct relationships between two components
- Performance is critical and you can't afford the indirection overhead
- Your team struggles with debugging event-driven flows
- You're dealing with complex event ordering or transactional requirements
- The observer chain might create memory leaks (dangling references)

## 📚 Further Reading

- [MDN EventTarget and Custom Events](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget) - Browser-native observer patterns
- [Python's asyncio and Observer patterns](https://docs.python.org/3/library/asyncio-eventloop.html) - Async event handling in Python
- [Node.js EventEmitter documentation](https://nodejs.org/api/events.html) - Built-in Pub/Sub implementation
- [Reactive Extensions (RxJS) documentation](https://rxjs.dev/guide/overview) - Advanced observable patterns for complex async flows
- [Martin Fowler on Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html) - Architectural patterns using Observer/Pub-Sub

---
*Auto-generated by [Daily Dev