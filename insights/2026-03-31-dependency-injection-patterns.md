# 📌 Dependency injection patterns
*March 31, 2026 · Daily Dev Insight*

## 🧠 Overview

Dependency injection (DI) is one of those patterns that separates junior developers from senior ones—not because it's complex, but because it fundamentally changes how you think about code architecture. At its core, DI is about inverting control: instead of your classes creating their own dependencies, you inject them from the outside. This seemingly simple shift transforms rigid, tightly-coupled code into flexible, testable systems.

The beauty of DI isn't just in testing (though that's huge), it's in how it forces you to think about interfaces and contracts. When you inject dependencies, you're essentially saying "I don't care what you are, I care what you can do." This mindset leads to better abstractions and more maintainable codebases. Modern frameworks have made DI so seamless that many developers use it without fully grasping its power—but understanding the underlying patterns will make you a significantly better architect.

## 💡 Key Concepts

• **Inversion of Control**: Dependencies flow inward from the application's outer layers, reversing the traditional "new" keyword approach
• **Constructor vs Setter vs Interface Injection**: Three main ways to provide dependencies, each with distinct trade-offs for immutability and flexibility  
• **Dependency Container/Registry**: Central location that manages object creation and lifetime, enabling complex dependency graphs
• **Service Locator Anti-pattern**: Avoid having objects ask for dependencies from a global registry—explicit injection is cleaner
• **Composition Root**: Single location near your application's entry point where all dependencies are wired together

## 🐍 Python Example

```python
from abc import ABC, abstractmethod
from typing import Protocol, List
import sqlite3

# Define contracts with protocols (Python's structural typing)
class EmailService(Protocol):
    def send_email(self, to: str, subject: str, body: str) -> bool: ...

class UserRepository(Protocol):
    def get_user_by_email(self, email: str) -> dict | None: ...

# Concrete implementations
class SMTPEmailService:
    def __init__(self, smtp_host: str, port: int):
        self.smtp_host = smtp_host
        self.port = port
    
    def send_email(self, to: str, subject: str, body: str) -> bool:
        print(f"Sending email via {self.smtp_host}:{self.port} to {to}")
        return True  # Simplified for example

class SQLiteUserRepository:
    def __init__(self, db_path: str):
        self.db_path = db_path
    
    def get_user_by_email(self, email: str) -> dict | None:
        # Simplified database query
        return {"email": email, "name": "John Doe"}

# Business logic with injected dependencies
class UserNotificationService:
    def __init__(self, email_service: EmailService, user_repo: UserRepository):
        self._email_service = email_service
        self._user_repo = user_repo
    
    def notify_user(self, email: str, message: str) -> bool:
        user = self._user_repo.get_user_by_email(email)
        if not user:
            return False
        
        return self._email_service.send_email(
            email, 
            "Notification", 
            f"Hello {user['name']}, {message}"
        )

# Composition root - wire everything together
def create_notification_service() -> UserNotificationService:
    email_service = SMTPEmailService("smtp.gmail.com", 587)
    user_repo = SQLiteUserRepository("/path/to/users.db")
    return UserNotificationService(email_service, user_repo)
```

## 🟨 JavaScript Example

```javascript
// Define interfaces using JSDoc for better IDE support
/**
 * @typedef {Object} PaymentProcessor
 * @property {function(number, string): Promise<boolean>} processPayment
 */

/**
 * @typedef {Object} OrderRepository
 * @property {function(Object): Promise<string>} saveOrder
 * @property {function(string): Promise<Object|null>} getOrder
 */

// Concrete implementations
class StripePaymentProcessor {
    constructor(apiKey) {
        this.apiKey = apiKey;
    }
    
    async processPayment(amount, currency) {
        console.log(`Processing $${amount} ${currency} via Stripe`);
        // Simulate API call
        return new Promise(resolve => setTimeout(() => resolve(true), 100));
    }
}

class MongoOrderRepository {
    constructor(connectionString) {
        this.connectionString = connectionString;
    }
    
    async saveOrder(orderData) {
        console.log('Saving order to MongoDB:', orderData.id);
        return orderData.id;
    }
    
    async getOrder(orderId) {
        return { id: orderId, status: 'pending', amount: 99.99 };
    }
}

// Business logic with dependency injection
class OrderService {
    /**
     * @param {PaymentProcessor} paymentProcessor
     * @param {OrderRepository} orderRepository
     */
    constructor(paymentProcessor, orderRepository) {
        this.paymentProcessor = paymentProcessor;
        this.orderRepository = orderRepository;
    }
    
    async processOrder(orderData) {
        try {
            // Save order first
            const orderId = await this.orderRepository.saveOrder(orderData);
            
            // Process payment
            const paymentSuccess = await this.paymentProcessor.processPayment(
                orderData.amount, 
                orderData.currency
            );
            
            if (paymentSuccess) {
                console.log(`Order ${orderId} processed successfully`);
                return { success: true, orderId };
            }
            
            throw new Error('Payment failed');
        } catch (error) {
            console.error('Order processing failed:', error.message);
            return { success: false, error: error.message };
        }
    }
}

// Simple DI container
class DIContainer {
    constructor() {
        this.services = new Map();
        this.singletons = new Map();
    }
    
    register(name, factory, singleton = false) {
        this.services.set(name, { factory, singleton });
        return this;
    }
    
    resolve(name) {
        const service = this.services.get(name);
        if (!service) throw new Error(`Service ${name} not found`);
        
        if (service.singleton) {
            if (!this.singletons.has(name)) {
                this.singletons.set(name, service.factory(this));
            }
            return this.singletons.get(name);
        }
        
        return service.factory(this);
    }
}

// Composition root
const container = new DIContainer()
    .register('paymentProcessor', () => new StripePaymentProcessor('sk_test_123'))
    .register('orderRepository', () => new MongoOrderRepository('mongodb://localhost:27017'))
    .register('orderService', (c) => new OrderService(
        c.resolve('paymentProcessor'),
        c.resolve('orderRepository')
    ), true);

// Usage
const orderService = container.resolve('orderService');
```

## ⚖️ When To Use / When To Avoid

**✅ When To Use:**
- Multi-layered applications with clear separation of concerns
- Code that needs extensive unit testing with mocked dependencies  
- Systems with multiple implementations of the same interface
- Applications where configuration varies between environments

**❌ When To Avoid:**
- Simple scripts or utilities with minimal dependencies
- Performance-critical code where DI containers add measurable overhead
- Teams unfamiliar with the pattern (introduce gradually)
- Over-engineering simple problems that don't warrant the abstraction

## 📚 Further Reading

• [Dependency Inversion Principle - Uncle Bob's Clean Architecture](https://blog.cleancoder.com/uncle-bob/2016/01/04/ALittleArchitecture.html)
• [Python Type Hints and Protocols Documentation](https://docs.python.org/3/library/typing.html#protocols)