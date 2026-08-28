# 📌 Dependency injection patterns
*August 28, 2026 · Daily Dev Insight*

## 🧠 Overview

Dependency Injection (DI) is one of those patterns that sounds more intimidating than it actually is. At its core, it's simply about giving an object its dependencies from the outside rather than having it create them internally. Think of it like this: instead of a chef going out to buy ingredients (tight coupling), someone delivers the ingredients to the chef's kitchen (loose coupling). This seemingly simple shift has profound implications for testability, modularity, and maintainability.

The beauty of DI isn't in following a rigid framework—it's in the flexibility it provides. When your database layer is injected rather than hardcoded, you can swap in a mock for testing without changing a single line of production code. When your logging service is injected, you can switch from console output to cloud logging by changing one configuration point. The pattern scales from simple constructor injection to sophisticated IoC containers, but the principle remains the same: depend on abstractions, inject concrete implementations.

What separates good DI from cargo-culting is understanding *why* you're doing it. Over-injecting everything creates unnecessary complexity. The goal isn't to inject for injection's sake—it's to make your code more flexible where flexibility actually matters: at integration boundaries, for external services, and anywhere you need different behavior in different contexts.

## 💡 Key Concepts

- **Constructor Injection**: Pass dependencies through the constructor—the most common and straightforward approach that makes dependencies explicit and required
- **Interface Segregation**: Depend on abstractions (interfaces/protocols) rather than concrete implementations, allowing any compatible implementation to be swapped in
- **Inversion of Control (IoC)**: The calling code controls what implementations are used, not the code being called—reversing the traditional dependency flow
- **Lifetime Management**: Understanding when dependencies should be singletons, transient (new each time), or scoped (per request/session) prevents subtle bugs
- **Poor Man's DI**: You don't always need a framework—simple factory functions or manual wiring in your main function is often sufficient and more transparent

## 🐍 Python Example

```python
from abc import ABC, abstractmethod
from typing import Protocol

# Define the abstraction (interface)
class EmailService(Protocol):
    def send(self, to: str, subject: str, body: str) -> bool:
        ...

# Concrete implementations
class SendGridEmailService:
    def __init__(self, api_key: str):
        self.api_key = api_key
    
    def send(self, to: str, subject: str, body: str) -> bool:
        print(f"[SendGrid] Sending to {to}: {subject}")
        # Actual SendGrid logic here
        return True

class MockEmailService:
    def __init__(self):
        self.sent_emails = []
    
    def send(self, to: str, subject: str, body: str) -> bool:
        self.sent_emails.append((to, subject, body))
        print(f"[Mock] Captured email to {to}")
        return True

# Service that depends on email functionality
class UserRegistrationService:
    def __init__(self, email_service: EmailService):
        # Dependency injected via constructor
        self.email_service = email_service
    
    def register_user(self, email: str, username: str) -> bool:
        # Business logic here...
        print(f"Registering user: {username}")
        
        # Use the injected dependency
        success = self.email_service.send(
            to=email,
            subject="Welcome!",
            body=f"Hello {username}, welcome to our platform!"
        )
        return success

# Usage in production
def main():
    email_service = SendGridEmailService(api_key="sk_live_xxx")
    user_service = UserRegistrationService(email_service)
    user_service.register_user("user@example.com", "johndoe")

# Usage in tests
def test_registration():
    mock_email = MockEmailService()
    user_service = UserRegistrationService(mock_email)
    user_service.register_user("test@example.com", "testuser")
    assert len(mock_email.sent_emails) == 1
    assert mock_email.sent_emails[0][0] == "test@example.com"

if __name__ == "__main__":
    main()
```

## 🟨 JavaScript Example

```javascript
// Define the interface through duck typing
class PaymentProcessor {
  processPayment(amount, currency) {
    throw new Error('Must implement processPayment');
  }
}

// Concrete implementations
class StripePaymentProcessor extends PaymentProcessor {
  constructor(secretKey) {
    super();
    this.secretKey = secretKey;
  }
  
  processPayment(amount, currency) {
    console.log(`[Stripe] Processing ${currency} ${amount}`);
    // Actual Stripe API call here
    return { success: true, transactionId: 'txn_123' };
  }
}

class MockPaymentProcessor extends PaymentProcessor {
  constructor() {
    super();
    this.transactions = [];
  }
  
  processPayment(amount, currency) {
    console.log(`[Mock] Captured payment: ${currency} ${amount}`);
    const transaction = { amount, currency, timestamp: Date.now() };
    this.transactions.push(transaction);
    return { success: true, transactionId: 'mock_123' };
  }
}

// Service with injected dependencies
class OrderService {
  constructor(paymentProcessor, logger = console) {
    // Multiple dependencies injected
    this.paymentProcessor = paymentProcessor;
    this.logger = logger;
  }
  
  async createOrder(userId, items, total) {
    this.logger.log(`Creating order for user ${userId}`);
    
    // Use injected payment processor
    const result = this.paymentProcessor.processPayment(total, 'USD');
    
    if (result.success) {
      this.logger.log(`Order completed: ${result.transactionId}`);
      return { orderId: 'ord_456', ...result };
    }
    
    throw new Error('Payment failed');
  }
}

// Production setup
const stripeProcessor = new StripePaymentProcessor('sk_live_xxx');
const orderService = new OrderService(stripeProcessor);
orderService.createOrder('user_789', ['item1'], 99.99);

// Test setup
const mockProcessor = new MockPaymentProcessor();
const testOrderService = new OrderService(mockProcessor);
testOrderService.createOrder('test_user', ['item1'], 10.00);
console.assert(mockProcessor.transactions.length === 1);
```

## ⚖️ When To Use / When To Avoid

**✅ Use Dependency Injection When:**
- You need to swap implementations (production vs. test, different providers)
- Working with external services (databases, APIs, file systems)
- Building libraries or frameworks that others will extend
- You want to unit test without hitting real infrastructure
- Multiple implementations of the same behavior exist

**❌ Avoid Dependency Injection When:**
- Dependencies are simple, stable utilities (like Math functions)
- You're injecting everything "just in case"—YAGNI applies
- The abstraction is more complex than the implementations
- You're in a small script or prototype where flexibility isn't needed
- Adding layers makes the code harder to understand than it helps

## 📚 Further Reading

- [Python Type Hints and Protocols - Official Documentation](https://docs.python.org/3/library/typing.html#typing.Protocol) - Understanding structural subtyping for DI
- [Dependency Injection Principles - Martin Fowler](https://martinfowler.com/articles/injection.html) - The definitive guide to DI patterns and trade-offs
- [InversifyJS Documentation](https://inversify.io/) - A powerful IoC container for TypeScript/JavaScript applications
- [Python Dependency Injector Library](https://python-dependency-injector.ets-labs.org/) - When you need more than constructor injection
- [SOLID Principles Explained](https://stackoverflow.blog/2021/11/01/why-solid-principles-are-still-the-foundation-for-modern-software