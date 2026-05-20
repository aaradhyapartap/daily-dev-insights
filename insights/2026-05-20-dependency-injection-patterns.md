# 📌 Dependency injection patterns
*May 20, 2026 · Daily Dev Insight*

## 🧠 Overview

Dependency injection (DI) is one of those patterns that separates junior developers from seasoned engineers. At its core, it's about inverting control — instead of your classes creating their own dependencies, you inject them from the outside. This seemingly simple shift transforms rigid, tightly-coupled code into flexible, testable architecture.

The real power of DI isn't just in the pattern itself, but in how it forces you to think about interfaces and abstractions. When you design with DI in mind, you naturally create more modular systems where components can be swapped, mocked, or configured without touching the core business logic. This becomes invaluable when you're dealing with databases, external APIs, or any component that needs different behavior in testing vs production environments.

Modern frameworks have made DI so seamless that many developers use it without fully understanding the underlying principles. But mastering these patterns manually will make you a better architect, even when working with frameworks that handle the heavy lifting.

## 💡 Key Concepts

• **Inversion of Control**: Dependencies flow from high-level modules to low-level ones, not the reverse
• **Constructor vs Setter vs Interface Injection**: Three main approaches to getting dependencies into your objects
• **Dependency Container**: A registry that manages object creation and lifetime, resolving dependency graphs automatically
• **Service Locator vs Pure DI**: Trade-offs between convenience and explicitness in dependency resolution
• **Composition Root**: The single place in your application where all dependencies are wired together

## 🐍 Python Example

```python
from abc import ABC, abstractmethod
from typing import Dict, Type, Any

# Abstract interfaces define contracts
class EmailService(ABC):
    @abstractmethod
    def send_email(self, to: str, subject: str, body: str) -> bool:
        pass

class UserRepository(ABC):
    @abstractmethod
    def get_user(self, user_id: str) -> Dict[str, Any]:
        pass

# Concrete implementations
class SMTPEmailService(EmailService):
    def __init__(self, smtp_host: str, port: int):
        self.smtp_host = smtp_host
        self.port = port
    
    def send_email(self, to: str, subject: str, body: str) -> bool:
        print(f"Sending email via SMTP {self.smtp_host}:{self.port}")
        return True

class PostgreSQLUserRepository(UserRepository):
    def __init__(self, connection_string: str):
        self.connection_string = connection_string
    
    def get_user(self, user_id: str) -> Dict[str, Any]:
        print(f"Fetching user {user_id} from PostgreSQL")
        return {"id": user_id, "name": "John Doe"}

# Business logic with injected dependencies
class UserNotificationService:
    def __init__(self, email_service: EmailService, user_repo: UserRepository):
        self.email_service = email_service
        self.user_repo = user_repo
    
    def notify_user(self, user_id: str, message: str) -> bool:
        user = self.user_repo.get_user(user_id)
        return self.email_service.send_email(
            user["email"] if "email" in user else "user@example.com",
            "Notification",
            message
        )

# Simple DI Container
class DIContainer:
    def __init__(self):
        self._services: Dict[Type, Any] = {}
    
    def register(self, interface: Type, implementation: Any):
        self._services[interface] = implementation
    
    def resolve(self, interface: Type) -> Any:
        return self._services.get(interface)

# Composition root - wire everything together
def create_app() -> UserNotificationService:
    container = DIContainer()
    
    # Register implementations
    email_service = SMTPEmailService("smtp.gmail.com", 587)
    user_repo = PostgreSQLUserRepository("postgresql://localhost/mydb")
    
    container.register(EmailService, email_service)
    container.register(UserRepository, user_repo)
    
    # Create main service with injected dependencies
    return UserNotificationService(
        container.resolve(EmailService),
        container.resolve(UserRepository)
    )
```

## 🟨 JavaScript Example

```javascript
// Interface-like contracts using classes
class PaymentProcessor {
  async processPayment(amount, cardToken) {
    throw new Error("Must implement processPayment");
  }
}

class NotificationService {
  async notify(userId, message) {
    throw new Error("Must implement notify");
  }
}

// Concrete implementations
class StripePaymentProcessor extends PaymentProcessor {
  constructor(apiKey) {
    super();
    this.apiKey = apiKey;
  }
  
  async processPayment(amount, cardToken) {
    console.log(`Processing $${amount} via Stripe`);
    // Simulate API call
    return { success: true, transactionId: 'stripe_' + Date.now() };
  }
}

class SlackNotificationService extends NotificationService {
  constructor(webhookUrl) {
    super();
    this.webhookUrl = webhookUrl;
  }
  
  async notify(userId, message) {
    console.log(`Sending Slack notification to user ${userId}: ${message}`);
    return true;
  }
}

// Business service with dependencies
class OrderService {
  constructor(paymentProcessor, notificationService) {
    this.paymentProcessor = paymentProcessor;
    this.notificationService = notificationService;
  }
  
  async processOrder(userId, amount, cardToken) {
    try {
      const payment = await this.paymentProcessor.processPayment(amount, cardToken);
      
      if (payment.success) {
        await this.notificationService.notify(
          userId, 
          `Payment of $${amount} processed successfully!`
        );
        return { orderId: Date.now(), status: 'completed' };
      }
    } catch (error) {
      await this.notificationService.notify(userId, `Payment failed: ${error.message}`);
      throw error;
    }
  }
}

// Modern DI Container with auto-wiring
class Container {
  constructor() {
    this.dependencies = new Map();
    this.singletons = new Map();
  }
  
  register(name, factory, singleton = false) {
    this.dependencies.set(name, { factory, singleton });
    return this;
  }
  
  resolve(name) {
    const dependency = this.dependencies.get(name);
    if (!dependency) throw new Error(`Dependency ${name} not found`);
    
    if (dependency.singleton && this.singletons.has(name)) {
      return this.singletons.get(name);
    }
    
    const instance = dependency.factory(this);
    if (dependency.singleton) {
      this.singletons.set(name, instance);
    }
    return instance;
  }
}

// Application setup with dependency wiring
function bootstrapApp() {
  const container = new Container();
  
  container
    .register('paymentProcessor', () => 
      new StripePaymentProcessor(process.env.STRIPE_API_KEY), true)
    .register('notificationService', () => 
      new SlackNotificationService(process.env.SLACK_WEBHOOK), true)
    .register('orderService', (container) => 
      new OrderService(
        container.resolve('paymentProcessor'),
        container.resolve('notificationService')
      ));
  
  return container.resolve('orderService');
}
```

## ⚖️ When To Use / When To Avoid

**✅ Use dependency injection when:**
• You need extensive unit testing with mocked dependencies
• Working with external services (databases, APIs, payment processors)
• Building applications with multiple environment configurations
• Creating libraries or frameworks that others will extend
• Team size is large and code needs clear separation of concerns

**❌ Avoid dependency injection when:**
• Building simple scripts or one-off utilities
• Performance is critical