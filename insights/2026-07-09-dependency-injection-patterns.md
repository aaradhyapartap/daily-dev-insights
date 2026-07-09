# 📌 Dependency injection patterns
*July 09, 2026 · Daily Dev Insight*

## 🧠 Overview

Dependency injection (DI) is one of those patterns that sounds more intimidating than it actually is. At its core, it's about giving your code what it needs from the outside rather than having it create dependencies internally. Think of it like ordering takeout instead of cooking in someone else's kitchen — you bring the ingredients (dependencies) with you rather than rummaging through their cabinets.

The real power of DI isn't just about loose coupling or testability (though those are huge wins). It's about making your code's requirements *explicit* and *visible*. When you look at a class constructor or function signature that takes dependencies as parameters, you immediately know what that code needs to work. No hidden global state, no surprise file system calls buried three layers deep, no mysterious database connections that appear out of nowhere.

The pattern has evolved significantly across languages and frameworks. While enterprise Java made it famous (some might say infamous) with heavyweight containers, modern implementations are refreshingly lightweight. You don't need a framework to practice DI — just thoughtful parameter passing and a willingness to invert control flow.

## 💡 Key Concepts

- **Inversion of Control**: Instead of your code creating its dependencies (with `new` or `import`), dependencies are passed in from the outside, letting callers control what implementations get used.

- **Constructor vs. Method Injection**: Constructor injection makes dependencies required and immutable, while method injection allows flexibility for optional or varying dependencies per operation.

- **Interface Segregation**: DI works best when you depend on abstractions (interfaces, protocols, abstract base classes) rather than concrete implementations, enabling easy swapping for tests or different environments.

- **Composition Over Configuration**: Modern DI favors explicit wiring in code rather than XML config files or magic annotations — if a human can't trace the dependency graph, your debugger will struggle too.

- **Dependency Lifetime Management**: Understanding scopes (singleton, transient, request-scoped) prevents subtle bugs like sharing state across requests or creating unnecessary instances.

## 🐍 Python Example

```python
from abc import ABC, abstractmethod
from datetime import datetime

# Define abstractions (interfaces)
class Logger(ABC):
    @abstractmethod
    def log(self, message: str) -> None:
        pass

class EmailSender(ABC):
    @abstractmethod
    def send(self, to: str, subject: str, body: str) -> bool:
        pass

# Concrete implementations
class ConsoleLogger(Logger):
    def log(self, message: str) -> None:
        print(f"[{datetime.now().isoformat()}] {message}")

class SMTPEmailSender(EmailSender):
    def __init__(self, smtp_host: str):
        self.smtp_host = smtp_host
    
    def send(self, to: str, subject: str, body: str) -> bool:
        # Actual SMTP logic would go here
        print(f"Sending email via {self.smtp_host} to {to}")
        return True

# Business logic with injected dependencies
class UserRegistrationService:
    def __init__(self, logger: Logger, email_sender: EmailSender):
        # Dependencies injected via constructor
        self.logger = logger
        self.email_sender = email_sender
    
    def register_user(self, email: str, username: str) -> bool:
        self.logger.log(f"Registering user: {username}")
        
        # Business logic here
        success = self.email_sender.send(
            to=email,
            subject="Welcome!",
            body=f"Hi {username}, thanks for registering!"
        )
        
        if success:
            self.logger.log(f"User {username} registered successfully")
        return success

# Manual dependency wiring (could use a DI container for larger apps)
logger = ConsoleLogger()
email_sender = SMTPEmailSender(smtp_host="smtp.example.com")
registration_service = UserRegistrationService(logger, email_sender)

# Easy to test with mock dependencies
registration_service.register_user("user@example.com", "john_doe")
```

## 🟨 JavaScript Example

```javascript
// Interface-like abstractions using JSDoc for type hints
class CacheService {
  async get(key) { throw new Error('Not implemented'); }
  async set(key, value, ttl) { throw new Error('Not implemented'); }
}

class MetricsCollector {
  recordEvent(eventName, metadata) { throw new Error('Not implemented'); }
}

// Concrete implementations
class RedisCache extends CacheService {
  constructor(redisClient) {
    super();
    this.client = redisClient;
  }
  
  async get(key) {
    console.log(`Redis GET: ${key}`);
    return await this.client.get(key);
  }
  
  async set(key, value, ttl = 3600) {
    console.log(`Redis SET: ${key} (TTL: ${ttl}s)`);
    return await this.client.set(key, value, 'EX', ttl);
  }
}

class DatadogMetrics extends MetricsCollector {
  recordEvent(eventName, metadata = {}) {
    console.log(`[Datadog] Event: ${eventName}`, metadata);
    // Actual Datadog API call would go here
  }
}

// Business service with dependency injection
class ProductService {
  constructor({ cache, metrics }) {
    // Object destructuring makes optional dependencies clear
    this.cache = cache;
    this.metrics = metrics;
  }
  
  async getProduct(productId) {
    const cacheKey = `product:${productId}`;
    
    // Check cache first
    const cached = await this.cache.get(cacheKey);
    if (cached) {
      this.metrics.recordEvent('cache_hit', { productId });
      return JSON.parse(cached);
    }
    
    // Simulate database fetch
    this.metrics.recordEvent('cache_miss', { productId });
    const product = { id: productId, name: 'Widget', price: 29.99 };
    
    await this.cache.set(cacheKey, JSON.stringify(product), 1800);
    return product;
  }
}

// Wire up dependencies
const mockRedisClient = {
  get: async (key) => null,
  set: async (key, val, ...args) => 'OK'
};

const productService = new ProductService({
  cache: new RedisCache(mockRedisClient),
  metrics: new DatadogMetrics()
});

// Usage
productService.getProduct(12345);
```

## ⚖️ When To Use / When To Avoid

**Use Dependency Injection when:**
- You need to write unit tests and want to mock external dependencies
- Multiple implementations of the same interface exist (dev/prod configs)
- Your code has complex object graphs that benefit from centralized wiring
- You're building libraries or frameworks meant to be extended by others

**Avoid or simplify when:**
- You're writing scripts or one-off utilities where testing isn't critical
- Dependencies are genuinely stable and unlikely to change (Python's `datetime`, for example)
- Your team finds heavyweight DI containers add more confusion than value
- Premature abstraction is creating interfaces with only one implementation

## 📚 Further Reading

- [Dependency Injection Principles, Practices, and Patterns](https://www.manning.com/books/dependency-injection-principles-practices-patterns) — Comprehensive book covering DI across languages and paradigms
- [Python dependency-injector Documentation](https://python-dependency-injector.ets-labs.org/) — Modern DI container for Python with excellent examples
- [Martin Fowler: Inversion of Control Containers](https://martinfowler.com/articles/injection.html) — Classic article explaining the pattern's fundamentals
- [TypeScript Dependency Injection with tsyringe](https://github.com/microsoft/tsyringe) — Lightweight DI container for TypeScript/JavaScript projects
- [Testing Without Mocks: A Pattern Language](https://www.jamesshore.com/v2/blog/2018/testing-without-mocks) — Alternative perspective on when DI and mocking might be overkill

---
*