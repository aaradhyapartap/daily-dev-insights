# 📌 SOLID principles with examples
*November 07, 2026 · Daily Dev Insight*

## 🧠 Overview

SOLID principles aren't just academic concepts from a dusty OOP textbook—they're battle-tested guidelines that separate maintainable codebases from those that make developers cry during 2 AM production incidents. Coined by Robert C. Martin (Uncle Bob), these five principles form a foundation for writing code that's easier to understand, extend, and refactor. The beauty of SOLID is that it's not about writing more code; it's about writing smarter code that welcomes change instead of fighting it.

Here's the reality: most codebases don't fail because of performance issues or scalability bottlenecks. They fail because they become impossible to change safely. SOLID principles address this by promoting loose coupling, high cohesion, and proper abstractions. When applied thoughtfully (not dogmatically), they create systems where adding a new feature doesn't require archeological excavations through layers of tangled dependencies.

Think of SOLID as guardrails, not handcuffs. The goal isn't to follow every principle religiously in every line of code—it's to recognize when you're violating them and understand the technical debt you're accepting. Sometimes you *should* break these rules, but you should do it consciously, not accidentally.

## 💡 Key Concepts

- **Single Responsibility Principle (SRP)**: A class should have one, and only one, reason to change. Each module should do one thing well, making it easier to understand, test, and modify without unexpected side effects.

- **Open/Closed Principle (OCP)**: Software entities should be open for extension but closed for modification. Use abstractions and polymorphism to add new functionality without changing existing, tested code.

- **Liskov Substitution Principle (LSP)**: Subtypes must be substitutable for their base types without breaking the program. Your inheritance hierarchies should make semantic sense, not just reuse code.

- **Interface Segregation Principle (ISP)**: Clients shouldn't be forced to depend on interfaces they don't use. Many small, focused interfaces beat one bloated "god interface."

- **Dependency Inversion Principle (DIP)**: Depend on abstractions, not concretions. High-level modules shouldn't depend on low-level modules; both should depend on abstractions.

## 🐍 Python Example

```python
from abc import ABC, abstractmethod
from typing import List

# DIP & ISP: Define focused abstractions
class MessageSender(ABC):
    @abstractmethod
    def send(self, message: str, recipient: str) -> bool:
        pass

# OCP: Easy to add new implementations without changing existing code
class EmailSender(MessageSender):
    def send(self, message: str, recipient: str) -> bool:
        print(f"Sending email to {recipient}: {message}")
        return True

class SMSSender(MessageSender):
    def send(self, message: str, recipient: str) -> bool:
        print(f"Sending SMS to {recipient}: {message}")
        return True

# SRP: NotificationService has ONE job - orchestrate notifications
class NotificationService:
    def __init__(self, sender: MessageSender):
        self._sender = sender  # DIP: Depend on abstraction
    
    def notify(self, recipient: str, message: str) -> bool:
        # Single responsibility: coordinate, don't implement sending logic
        return self._sender.send(message, recipient)

# SRP: User management separate from notifications
class User:
    def __init__(self, name: str, contact: str):
        self.name = name
        self.contact = contact

# Usage demonstrates flexibility from following SOLID
if __name__ == "__main__":
    user = User("Alice", "alice@example.com")
    
    # Easy to swap implementations (OCP, DIP)
    email_notifier = NotificationService(EmailSender())
    email_notifier.notify(user.contact, "Your order shipped!")
    
    sms_notifier = NotificationService(SMSSender())
    sms_notifier.notify("+1234567890", "Your code is 1234")
```

## 🟨 JavaScript Example

```javascript
// ISP & DIP: Small, focused interface
class PaymentProcessor {
  process(amount) {
    throw new Error("Must implement process()");
  }
}

// OCP: Add new payment methods without modifying existing code
class StripeProcessor extends PaymentProcessor {
  process(amount) {
    console.log(`Processing $${amount} via Stripe`);
    return { success: true, transactionId: "stripe_123" };
  }
}

class PayPalProcessor extends PaymentProcessor {
  process(amount) {
    console.log(`Processing $${amount} via PayPal`);
    return { success: true, transactionId: "paypal_456" };
  }
}

// SRP: OrderService handles order logic, not payment implementation
class OrderService {
  constructor(paymentProcessor) {
    this.paymentProcessor = paymentProcessor; // DIP: Inject dependency
  }
  
  checkout(order) {
    // Single responsibility: coordinate checkout flow
    const total = this.calculateTotal(order.items);
    const result = this.paymentProcessor.process(total);
    
    if (result.success) {
      console.log(`Order confirmed: ${result.transactionId}`);
    }
    return result;
  }
  
  calculateTotal(items) {
    return items.reduce((sum, item) => sum + item.price, 0);
  }
}

// Usage shows how SOLID makes code flexible
const order = { items: [{ price: 29.99 }, { price: 49.99 }] };

const stripeService = new OrderService(new StripeProcessor());
stripeService.checkout(order);

const paypalService = new OrderService(new PayPalProcessor());
paypalService.checkout(order);
```

## ⚖️ When To Use / When To Avoid

**When to use:**
- Building systems expected to evolve over time with new features
- Working in teams where multiple developers modify the same codebase
- Creating libraries or frameworks that others will extend
- When you find yourself copy-pasting code or making parallel changes in multiple places

**When to avoid (or apply lightly):**
- Simple scripts or one-off utilities that won't be maintained
- Prototypes where you're exploring problem space, not solution space
- Over-engineering trivial features—don't create interfaces for things that will never have multiple implementations
- When abstractions would obscure rather than clarify (premature abstraction is as bad as premature optimization)

## 📚 Further Reading

- [SOLID Principles in Python by Real Python](https://realpython.com/solid-principles-python/) – Comprehensive guide with practical examples
- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) – The philosophy behind SOLID from Uncle Bob himself
- [JavaScript Design Patterns (Addy Osmani)](https://addyosmani.com/resources/essentialjsdesignpatterns/book/) – How SOLID applies in JavaScript's prototype world
- [SOLID: The First 5 Principles of OOP Design (DigitalOcean)](https://www.digitalocean.com/community/conceptual-articles/s-o-l-i-d-the-first-five-principles-of-object-oriented-design) – Beginner-friendly breakdown with examples
- [Dependency Injection in Python](https://python-dependency-injector.ets-labs.org/) – Deep dive into DIP implementation

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*