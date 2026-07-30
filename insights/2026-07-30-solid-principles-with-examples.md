# 📌 SOLID principles with examples
*July 30, 2026 · Daily Dev Insight*

## 🧠 Overview

SOLID principles aren't just academic theory—they're battle-tested guidelines that separate maintainable codebases from nightmare spaghetti. Coined by Robert C. Martin (Uncle Bob), these five principles form the foundation of object-oriented design that actually scales. The acronym stands for Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion. Each principle addresses a specific code smell that plagues growing projects.

Here's the reality: you don't need to apply all five principles to every class you write. That's cargo-cult programming. But understanding them gives you a mental toolkit for recognizing when your code is fighting against you. When a class becomes hard to test, when small changes ripple across your codebase, or when adding features feels like defusing a bomb—that's when SOLID principles offer concrete solutions.

The magic happens when these principles work together. A class with a single responsibility is easier to extend without modification. Proper abstractions enable dependency inversion, which makes your code testable. It's a cascading effect that transforms brittle code into flexible systems.

## 💡 Key Concepts

- **Single Responsibility Principle (SRP)**: A class should have one, and only one, reason to change. If you're struggling to name a class without using "and" or "Manager," you're likely violating SRP.

- **Open/Closed Principle (OCP)**: Software entities should be open for extension but closed for modification. Use abstractions and polymorphism to add features without editing existing, working code.

- **Liskov Substitution Principle (LSP)**: Subtypes must be substitutable for their base types without breaking the program. If your override throws "NotImplementedError," you're doing inheritance wrong.

- **Interface Segregation Principle (ISP)**: Clients shouldn't be forced to depend on interfaces they don't use. Many small, focused interfaces beat one bloated "god interface."

- **Dependency Inversion Principle (DIP)**: Depend on abstractions, not concretions. High-level modules shouldn't import low-level modules directly—both should depend on interfaces.

## 🐍 Python Example

```python
from abc import ABC, abstractmethod
from typing import List

# DIP: Define abstraction for notifications
class NotificationSender(ABC):
    @abstractmethod
    def send(self, message: str, recipient: str) -> None:
        pass

# LSP: Concrete implementations are substitutable
class EmailSender(NotificationSender):
    def send(self, message: str, recipient: str) -> None:
        print(f"Sending email to {recipient}: {message}")

class SMSSender(NotificationSender):
    def send(self, message: str, recipient: str) -> None:
        print(f"Sending SMS to {recipient}: {message}")

# SRP: This class only handles user registration logic
class UserRegistration:
    def __init__(self, notifier: NotificationSender):
        # DIP: Depend on abstraction, not concrete class
        self.notifier = notifier
    
    def register(self, email: str, phone: str) -> None:
        # Registration logic here
        print(f"Registering user: {email}")
        self.notifier.send("Welcome!", email)

# OCP: Add new notification types without modifying existing code
class PushNotificationSender(NotificationSender):
    def send(self, message: str, recipient: str) -> None:
        print(f"Sending push notification to {recipient}: {message}")

# ISP: Small, focused interface (NotificationSender has one method)
# Usage demonstrates flexibility
if __name__ == "__main__":
    # Can swap implementations without changing UserRegistration
    email_registration = UserRegistration(EmailSender())
    email_registration.register("user@example.com", "555-0100")
    
    sms_registration = UserRegistration(SMSSender())
    sms_registration.register("user@example.com", "555-0100")
```

## 🟨 JavaScript Example

```javascript
// DIP & ISP: Simple, focused interface for payment processing
class PaymentProcessor {
  processPayment(amount, details) {
    throw new Error("Must implement processPayment");
  }
}

// LSP: Substitutable implementations
class StripeProcessor extends PaymentProcessor {
  processPayment(amount, details) {
    console.log(`Processing $${amount} via Stripe`);
    // Stripe API call here
    return { success: true, transactionId: "stripe_123" };
  }
}

class PayPalProcessor extends PaymentProcessor {
  processPayment(amount, details) {
    console.log(`Processing $${amount} via PayPal`);
    // PayPal API call here
    return { success: true, transactionId: "paypal_456" };
  }
}

// SRP: OrderService only manages order workflow
class OrderService {
  constructor(paymentProcessor) {
    // DIP: Inject dependency, don't create it
    this.paymentProcessor = paymentProcessor;
  }
  
  placeOrder(items, paymentDetails) {
    const total = items.reduce((sum, item) => sum + item.price, 0);
    console.log(`Order total: $${total}`);
    
    // Delegate payment to injected processor
    const result = this.paymentProcessor.processPayment(total, paymentDetails);
    
    if (result.success) {
      console.log(`Order confirmed! Transaction: ${result.transactionId}`);
    }
  }
}

// OCP: Add new payment methods without modifying OrderService
class CryptoProcessor extends PaymentProcessor {
  processPayment(amount, details) {
    console.log(`Processing $${amount} via Cryptocurrency`);
    return { success: true, transactionId: "crypto_789" };
  }
}

// Usage shows easy swapping of implementations
const items = [{ name: "Widget", price: 29.99 }, { name: "Gadget", price: 49.99 }];

const stripeOrder = new OrderService(new StripeProcessor());
stripeOrder.placeOrder(items, { card: "4242..." });

const cryptoOrder = new OrderService(new CryptoProcessor());
cryptoOrder.placeOrder(items, { wallet: "0x123..." });
```

## ⚖️ When To Use / When To Avoid

**Use SOLID principles when:**
- Building systems that will grow and change over time
- Working on team projects where multiple developers touch the same code
- You need high testability (dependency injection enables mocking)
- Requirements are evolving and you need flexibility
- You're refactoring legacy code into maintainable modules

**Avoid over-applying when:**
- Building simple scripts or one-off utilities (don't over-engineer)
- Prototyping or MVP development where requirements are unclear
- Performance is critical and abstraction layers add measurable overhead
- The domain is extremely stable with no expected changes
- Your team lacks OOP experience (introduce gradually, not all at once)

## 📚 Further Reading

- [SOLID Principles in Python - Real Python](https://realpython.com/solid-principles-python/) — Comprehensive guide with practical Python examples
- [The Principles of OOD by Uncle Bob Martin](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod) — Original writings from the creator of SOLID
- [JavaScript Design Patterns - Addy Osmani](https://addyosmani.com/resources/essentialjsdesignpatterns/) — How SOLID applies to modern JavaScript
- [Refactoring Guru: SOLID Principles](https://refactoring.guru/design-patterns/solid-principles) — Visual explanations with multi-language examples
- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) — How SOLID principles scale to system architecture

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*