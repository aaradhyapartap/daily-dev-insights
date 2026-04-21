# 📌 SOLID principles with examples
*April 21, 2026 · Daily Dev Insight*

## 🧠 Overview

SOLID principles aren't just academic theory—they're battle-tested guidelines that separate maintainable code from technical debt nightmares. These five principles help you write code that's easier to understand, extend, and refactor without breaking everything downstream. Think of them as guardrails that keep your architecture from becoming a tangled mess of dependencies.

The beauty of SOLID lies in its focus on *separation of concerns* and *dependency management*. When you follow these principles, you're essentially future-proofing your code against the inevitable changes that come with evolving requirements. Your classes become more focused, your interfaces become more stable, and your tests become easier to write and maintain.

## 💡 Key Concepts

• **Single Responsibility Principle (SRP)**: Each class should have only one reason to change—one job, one responsibility
• **Open/Closed Principle (OCP)**: Classes should be open for extension but closed for modification—add new features without changing existing code
• **Liskov Substitution Principle (LSP)**: Subtypes must be substitutable for their base types without breaking functionality
• **Interface Segregation Principle (ISP)**: Clients shouldn't depend on interfaces they don't use—keep interfaces focused and minimal
• **Dependency Inversion Principle (DIP)**: Depend on abstractions, not concretions—high-level modules shouldn't depend on low-level implementation details

## 🐍 Python Example

```python
from abc import ABC, abstractmethod
from typing import List, Protocol

# DIP: Abstract interface for notifications
class NotificationSender(Protocol):
    def send(self, message: str, recipient: str) -> bool:
        ...

# SRP: Each sender has one responsibility
class EmailSender:
    def send(self, message: str, recipient: str) -> bool:
        print(f"Email sent to {recipient}: {message}")
        return True

class SMSSender:
    def send(self, message: str, recipient: str) -> bool:
        print(f"SMS sent to {recipient}: {message}")
        return True

# ISP: Focused interfaces for different user types
class BasicUser(ABC):
    @abstractmethod
    def get_contact(self) -> str: ...

class PremiumUser(BasicUser):
    @abstractmethod
    def get_preferences(self) -> dict: ...

# LSP: PremiumCustomer can substitute Customer
class Customer(BasicUser):
    def __init__(self, email: str):
        self.email = email
    
    def get_contact(self) -> str:
        return self.email

class PremiumCustomer(PremiumUser):
    def __init__(self, email: str, phone: str):
        self.email = email
        self.phone = phone
    
    def get_contact(self) -> str:
        return self.email
    
    def get_preferences(self) -> dict:
        return {"sms_enabled": True}

# OCP: Extensible without modifying existing code
class NotificationService:
    def __init__(self, sender: NotificationSender):
        self._sender = sender  # DIP: Depend on abstraction
    
    def notify_user(self, user: BasicUser, message: str):
        return self._sender.send(message, user.get_contact())

# Usage demonstrates all principles working together
email_service = NotificationService(EmailSender())
sms_service = NotificationService(SMSSender())

users = [Customer("user@example.com"), PremiumCustomer("vip@example.com", "555-0123")]
for user in users:
    email_service.notify_user(user, "Welcome!")
```

## 🟨 JavaScript Example

```javascript
// DIP: Abstract payment processor interface
class PaymentProcessor {
  process(amount, cardDetails) {
    throw new Error("Must implement process method");
  }
}

// SRP: Each processor handles one payment type
class StripeProcessor extends PaymentProcessor {
  process(amount, cardDetails) {
    console log(`Processing $${amount} via Stripe`);
    return { success: true, transactionId: `stripe_${Date.now()}` };
  }
}

class PayPalProcessor extends PaymentProcessor {
  process(amount, cardDetails) {
    console.log(`Processing $${amount} via PayPal`);
    return { success: true, transactionId: `paypal_${Date.now()}` };
  }
}

// ISP: Separate interfaces for different capabilities
class Discountable {
  applyDiscount(percentage) {
    throw new Error("Must implement applyDiscount");
  }
}

// LSP: RegularOrder and PremiumOrder are substitutable
class Order {
  constructor(amount, items) {
    this.amount = amount;
    this.items = items;
  }
  
  getTotal() { return this.amount; }
}

class PremiumOrder extends Order {
  constructor(amount, items, discountRate = 0.1) {
    super(amount, items);
    this.discountRate = discountRate;
  }
  
  // LSP: Maintains expected behavior contract
  getTotal() {
    return this.amount * (1 - this.discountRate);
  }
  
  applyDiscount(percentage) {
    this.discountRate = Math.min(percentage / 100, 0.5);
  }
}

// OCP: Can extend with new order types without modification
class OrderProcessor {
  constructor(paymentProcessor) {
    this.paymentProcessor = paymentProcessor; // DIP
  }
  
  processOrder(order) {
    const total = order.getTotal();
    return this.paymentProcessor.process(total, order.cardDetails);
  }
}

// Usage showcasing extensibility
const stripeProcessor = new OrderProcessor(new StripeProcessor());
const orders = [
  new Order(100, ['laptop']),
  new PremiumOrder(200, ['phone', 'case'])
];

orders.forEach(order => stripeProcessor.processOrder(order));
```

## ⚖️ When To Use / When To Avoid

**When To Use:**
- Building systems expected to evolve and scale over time
- Working in teams where code maintainability is crucial  
- Creating libraries or frameworks that others will extend
- When requirements change frequently and you need flexibility

**When To Avoid:**
- Simple scripts or one-off utilities with clear, unchanging requirements
- Performance-critical code where abstraction layers add unacceptable overhead
- Very small codebases where the overhead of interfaces outweighs benefits
- Prototypes or proof-of-concepts where speed of development trumps architecture

## 📚 Further Reading

• [Clean Architecture by Robert C. Martin](https://docs.python.org/3/library/abc.html) - The definitive guide to SOLID and clean code principles
• [MDN: Classes and Inheritance in JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes) - Modern JS patterns for SOLID implementation
• [Python Abstract Base Classes Documentation](https://docs.python.org/3/library/abc.html) - Official guide to interfaces and abstractions in Python
• [TypeScript Handbook: Interfaces](https://www.typescriptlang.org/docs/handbook/interfaces.html) - Type-safe approaches to interface segregation
• [Refactoring Guru: SOLID Principles](https://refactoring.guru/design-patterns) - Interactive examples and anti-patterns

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*