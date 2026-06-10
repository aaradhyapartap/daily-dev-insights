# 📌 SOLID principles with examples
*June 10, 2026 · Daily Dev Insight*

## 🧠 Overview

SOLID principles aren't just academic computer science theory—they're battle-tested guidelines that separate maintainable codebases from the ones that make developers cry at 2 AM. These five principles (Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion) form the backbone of object-oriented design, but their wisdom extends far beyond OOP.

The real power of SOLID isn't in following them religiously, but in understanding the trade-offs they represent. They help you write code that's easier to test, debug, and extend without breaking existing functionality. However, like any architectural pattern, they can be over-applied, leading to unnecessary abstraction and complexity. The key is knowing when the benefits outweigh the cognitive overhead.

## 💡 Key Concepts

• **Single Responsibility Principle (SRP)**: Each class should have one reason to change. If you're struggling to name a class without using "and" or "or", you're probably violating SRP
• **Open/Closed Principle (OCP)**: Open for extension, closed for modification. Use composition and interfaces to add new behavior without changing existing code
• **Liskov Substitution Principle (LSP)**: Subclasses should be substitutable for their base classes without breaking functionality. If your override changes expected behavior, you're doing it wrong
• **Interface Segregation Principle (ISP)**: Many specific interfaces are better than one general-purpose interface. Don't force classes to depend on methods they don't use
• **Dependency Inversion Principle (DIP)**: Depend on abstractions, not concretions. High-level modules shouldn't depend on low-level implementation details

## 🐍 Python Example

```python
from abc import ABC, abstractmethod
from typing import List

# DIP & ISP: Define focused abstractions
class PaymentProcessor(ABC):
    @abstractmethod
    def process_payment(self, amount: float) -> bool:
        pass

class NotificationService(ABC):
    @abstractmethod
    def send_notification(self, message: str, recipient: str) -> None:
        pass

# SRP: Each class has a single responsibility
class StripePaymentProcessor(PaymentProcessor):
    def process_payment(self, amount: float) -> bool:
        # Stripe-specific payment logic
        print(f"Processing ${amount} via Stripe")
        return amount > 0

class EmailNotificationService(NotificationService):
    def send_notification(self, message: str, recipient: str) -> None:
        print(f"Email to {recipient}: {message}")

class SMSNotificationService(NotificationService):
    def send_notification(self, message: str, recipient: str) -> None:
        print(f"SMS to {recipient}: {message}")

# OCP: Easy to extend with new payment types without modifying existing code
class PayPalPaymentProcessor(PaymentProcessor):
    def process_payment(self, amount: float) -> bool:
        print(f"Processing ${amount} via PayPal")
        return amount > 0

# SRP: Order service only handles order logic
class OrderService:
    def __init__(self, payment_processor: PaymentProcessor, 
                 notification_service: NotificationService):
        self.payment_processor = payment_processor
        self.notification_service = notification_service
    
    def process_order(self, amount: float, customer_email: str) -> bool:
        if self.payment_processor.process_payment(amount):
            self.notification_service.send_notification(
                f"Order confirmed for ${amount}", customer_email
            )
            return True
        return False

# Usage demonstrates DIP: high-level OrderService depends on abstractions
order_service = OrderService(
    StripePaymentProcessor(),
    EmailNotificationService()
)
order_service.process_order(99.99, "customer@example.com")
```

## 🟨 JavaScript Example

```javascript
// ISP & DIP: Focused interfaces using dependency injection
class PaymentGateway {
  processPayment(amount) {
    throw new Error('Method must be implemented');
  }
}

class Logger {
  log(message) {
    throw new Error('Method must be implemented');
  }
}

// SRP: Single responsibility implementations
class StripeGateway extends PaymentGateway {
  processPayment(amount) {
    console.log(`Stripe processing $${amount}`);
    return Promise.resolve({ success: amount > 0, transactionId: 'stripe_123' });
  }
}

class ConsoleLogger extends Logger {
  log(message) {
    console.log(`[${new Date().toISOString()}] ${message}`);
  }
}

// LSP: Subclasses maintain expected behavior contract
class PayPalGateway extends PaymentGateway {
  processPayment(amount) {
    console.log(`PayPal processing $${amount}`);
    return Promise.resolve({ success: amount > 0, transactionId: 'paypal_456' });
  }
}

// OCP: Easily extensible without modifying existing payment logic
class SubscriptionService {
  constructor(paymentGateway, logger) {
    this.paymentGateway = paymentGateway;
    this.logger = logger;
  }

  async createSubscription(userId, planAmount) {
    try {
      this.logger.log(`Creating subscription for user ${userId}`);
      
      const result = await this.paymentGateway.processPayment(planAmount);
      
      if (result.success) {
        this.logger.log(`Subscription created: ${result.transactionId}`);
        return { success: true, subscriptionId: `sub_${Date.now()}` };
      }
      
      this.logger.log('Payment failed');
      return { success: false, error: 'Payment processing failed' };
    } catch (error) {
      this.logger.log(`Error: ${error.message}`);
      return { success: false, error: error.message };
    }
  }
}

// Usage: Easy to swap implementations without changing business logic
const subscriptionService = new SubscriptionService(
  new StripeGateway(),
  new ConsoleLogger()
);

subscriptionService.createSubscription('user_123', 29.99);
```

## ⚖️ When To Use / When To Avoid

**✅ Use SOLID When:**
- Building systems with multiple developers or long-term maintenance requirements
- You anticipate frequent changes to business logic or external integrations  
- Writing library code that others will extend or modify
- Test coverage is a priority (SOLID makes mocking and unit testing much easier)

**❌ Avoid SOLID When:**
- Building quick prototypes or proof-of-concepts where speed matters more than structure
- Working with simple, stable requirements that are unlikely to change
- The team is small and communication overhead is low
- Over-engineering would add significant complexity for minimal benefit

## 📚 Further Reading

• [Clean Code by Robert Martin](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882) - The definitive guide to SOLID principles from their creator
• [Python Type Hints Documentation](https://docs.python.org/3/library/typing.html) - Essential for implementing ISP and DIP effectively in Python
• [MDN JavaScript Classes Guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes) - Modern JavaScript patterns for applying SOLID principles
• [Refactoring Guru Design Patterns](https://refactoring.guru/design-patterns) - Visual examples of how SOLID principles enable common design patterns
• [Martin Fowler's Bliki on SOLID](https://martinfowler.com/bliki/SOLID.html) - Practical insights and critiques from a respected software architect

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*