# 📌 Dependency injection patterns
*October 17, 2026 · Daily Dev Insight*

## 🧠 Overview

Dependency injection (DI) is one of those patterns that sounds academic but solves a brutally practical problem: how do you write testable, flexible code without turning your codebase into a tangled mess of imports and globals? At its core, DI means passing dependencies into a component rather than having that component create or import them directly. This inversion of control is what separates junior code from production-grade architecture.

The beauty of DI isn't just about testing—though being able to swap a real database for a mock in tests is incredibly valuable. It's about making your code honest about what it needs. When a class declares its dependencies upfront (usually via constructor parameters), you can look at the signature and immediately understand its requirements. No hidden global state, no surprise side effects, no hunting through nested imports to understand what's actually being used.

Here's the controversial take: DI frameworks are often overkill. In Python and JavaScript, simple constructor injection gets you 90% of the benefits with 10% of the complexity. Save the heavyweight frameworks for when you're actually building enterprise applications with hundreds of services and complex lifecycle management needs.

## 💡 Key Concepts

- **Constructor Injection**: Pass dependencies as parameters when creating an object. This is the most explicit and testable approach, making dependencies impossible to overlook.

- **Interface Segregation**: Depend on abstractions (protocols, interfaces) rather than concrete implementations. This lets you swap implementations without changing dependent code.

- **Composition Over Inheritance**: DI encourages building functionality by composing small, focused objects rather than creating deep inheritance hierarchies that are hard to test and modify.

- **Dependency Inversion Principle**: High-level modules shouldn't depend on low-level modules. Both should depend on abstractions. This is the "D" in SOLID and the philosophical foundation of DI.

- **Service Locator Anti-pattern**: Avoid having classes reach into a global registry to fetch dependencies. It hides dependencies and makes testing painful. Be explicit instead.

## 🐍 Python Example

```python
from typing import Protocol
from dataclasses import dataclass

# Define abstract interfaces using Protocol
class MessageSender(Protocol):
    def send(self, recipient: str, message: str) -> bool:
        ...

class UserRepository(Protocol):
    def get_email(self, user_id: str) -> str | None:
        ...

# Concrete implementations
class EmailSender:
    def __init__(self, smtp_host: str):
        self.smtp_host = smtp_host
    
    def send(self, recipient: str, message: str) -> bool:
        print(f"Sending email via {self.smtp_host} to {recipient}")
        return True

class DatabaseUserRepo:
    def get_email(self, user_id: str) -> str | None:
        # In reality, this would query a database
        return f"user{user_id}@example.com"

# Business logic class that depends on abstractions
class NotificationService:
    def __init__(self, sender: MessageSender, user_repo: UserRepository):
        # Dependencies injected via constructor
        self.sender = sender
        self.user_repo = user_repo
    
    def notify_user(self, user_id: str, message: str) -> bool:
        email = self.user_repo.get_email(user_id)
        if not email:
            return False
        return self.sender.send(email, message)

# Assembly/composition happens at the application boundary
def create_production_service() -> NotificationService:
    sender = EmailSender(smtp_host="smtp.gmail.com")
    repo = DatabaseUserRepo()
    return NotificationService(sender, repo)

# Easy to test with mocks
def create_test_service(mock_sender, mock_repo) -> NotificationService:
    return NotificationService(mock_sender, mock_repo)

# Usage
service = create_production_service()
service.notify_user("123", "Your order has shipped!")
```

## 🟨 JavaScript Example

```javascript
// Define interfaces through documentation and runtime checks
class NotificationService {
  /**
   * @param {Object} emailClient - Must implement send(to, subject, body)
   * @param {Object} userRepo - Must implement findById(id)
   * @param {Object} logger - Must implement info(msg) and error(msg)
   */
  constructor(emailClient, userRepo, logger) {
    this.emailClient = emailClient;
    this.userRepo = userRepo;
    this.logger = logger;
  }

  async notifyUser(userId, subject, message) {
    try {
      const user = await this.userRepo.findById(userId);
      
      if (!user || !user.email) {
        this.logger.error(`User ${userId} not found or has no email`);
        return false;
      }

      await this.emailClient.send(user.email, subject, message);
      this.logger.info(`Notification sent to user ${userId}`);
      return true;
    } catch (error) {
      this.logger.error(`Failed to notify user: ${error.message}`);
      return false;
    }
  }
}

// Concrete implementations
class SMTPEmailClient {
  constructor(config) {
    this.config = config;
  }
  
  async send(to, subject, body) {
    console.log(`Sending via SMTP: ${to}`);
    // Actual SMTP logic here
    return true;
  }
}

class PostgresUserRepository {
  constructor(dbConnection) {
    this.db = dbConnection;
  }
  
  async findById(id) {
    // Simulate DB query
    return { id, email: `user${id}@test.com`, name: "Test User" };
  }
}

// Factory pattern for dependency assembly
function createNotificationService(env = 'production') {
  const logger = console; // Or winston, pino, etc.
  
  if (env === 'test') {
    // Inject test doubles
    return new NotificationService(
      { send: async () => true },
      { findById: async (id) => ({ id, email: 'test@example.com' }) },
      { info: () => {}, error: () => {} }
    );
  }
  
  // Production dependencies
  const emailClient = new SMTPEmailClient({ host: 'smtp.example.com' });
  const userRepo = new PostgresUserRepository(/* db connection */);
  
  return new NotificationService(emailClient, userRepo, logger);
}

// Usage
const service = createNotificationService();
service.notifyUser('42', 'Welcome!', 'Thanks for signing up');
```

## ⚖️ When To Use / When To Avoid

**Use Dependency Injection When:**
- You need to write unit tests without hitting real databases/APIs/filesystems
- Multiple implementations of a dependency exist (e.g., different storage backends)
- Your application has clear architectural boundaries (services, repositories, controllers)
- You're building libraries or frameworks that need to be configuration-agnostic

**Avoid Dependency Injection When:**
- You're writing simple scripts or one-off utilities where testing isn't critical
- Dependencies are truly stable and have zero chance of needing alternatives
- The overhead of abstraction exceeds the benefit (not everything needs to be injectable)
- You're prematurely optimizing for flexibility you don't actually need yet

## 📚 Further Reading

- [Dependency Inversion Principle - Python Type Hints with Protocol](https://docs.python.org/3/library/typing.html#typing.Protocol) - Official Python documentation on structural subtyping for DI
- [Inversion of Control Containers and the Dependency Injection pattern](https://martinfowler.com/articles/injection.html) - Martin Fowler's definitive article on DI patterns
- [SOLID Principles: Dependency Inversion](https://en.wikipedia.org/wiki/Dependency_inversion_principle) - Wikipedia's comprehensive explanation of the principle underlying DI
- [Testing with Dependency Injection in Node.js](https://developer.mozilla.org/en-US/docs/Learn/Server-side/Express_Nodejs/Testing) - MDN guide on testing practices with DI in Express applications

---