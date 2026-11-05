# 📌 Microservices communication patterns
*November 05, 2026 · Daily Dev Insight*

## 🧠 Overview

When you split a monolith into microservices, you're essentially trading local function calls for network calls. The way these services talk to each other becomes the backbone of your entire system's reliability, performance, and maintainability. Get it wrong, and you'll spend your days debugging cascading failures and timeouts. Get it right, and you'll have a resilient, scalable architecture that your team can actually reason about.

There are two fundamental approaches: synchronous (request-response) and asynchronous (event-driven). Synchronous communication, typically via REST or gRPC, is straightforward and familiar—Service A directly asks Service B for something and waits for an answer. Asynchronous communication uses message queues or event streams, where services publish events without waiting for responses. Most mature microservice architectures use both patterns strategically.

The real art isn't picking one pattern and sticking to it religiously—it's knowing when each pattern serves your business needs. User authentication? Synchronous makes sense. Processing a million order confirmations? You'd better be asynchronous. The communication pattern you choose directly impacts your system's coupling, fault tolerance, and ability to scale independently.

## 💡 Key Concepts

- **Synchronous patterns** (REST, gRPC) create tight coupling and require both services to be available simultaneously, but offer simplicity and immediate consistency
- **Asynchronous patterns** (message queues, pub/sub) enable loose coupling and better fault tolerance, but introduce eventual consistency and complexity in tracking request flows
- **Service mesh** tools like Istio or Linkerd handle cross-cutting concerns (retries, circuit breaking, observability) at the infrastructure level rather than in application code
- **Circuit breakers** prevent cascading failures by failing fast when a downstream service is unhealthy, giving it time to recover
- **Event sourcing and CQRS** patterns work naturally with async communication, storing state changes as events rather than current state snapshots

## 🐍 Python Example

```python
import asyncio
import aiohttp
from typing import Optional
import time

class CircuitBreaker:
    """Prevents cascading failures in sync communication"""
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    def can_execute(self) -> bool:
        if self.state == "CLOSED":
            return True
        if self.state == "OPEN":
            if time.time() - self.last_failure_time >= self.timeout:
                self.state = "HALF_OPEN"
                return True
            return False
        return True  # HALF_OPEN
    
    def record_success(self):
        self.failure_count = 0
        self.state = "CLOSED"
    
    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"

async def call_service_with_circuit_breaker(
    url: str, 
    circuit_breaker: CircuitBreaker
) -> Optional[dict]:
    """Make HTTP call with circuit breaker pattern"""
    if not circuit_breaker.can_execute():
        print(f"Circuit breaker OPEN for {url}")
        return None
    
    try:
        async with aiohttp.ClientSession() as session:
            async with session.get(url, timeout=2) as response:
                data = await response.json()
                circuit_breaker.record_success()
                return data
    except Exception as e:
        print(f"Request failed: {e}")
        circuit_breaker.record_failure()
        return None

# Usage
async def main():
    breaker = CircuitBreaker(failure_threshold=3, timeout=30)
    result = await call_service_with_circuit_breaker(
        "http://payment-service/api/process",
        breaker
    )
    print(f"Result: {result}")

asyncio.run(main())
```

## 🟨 JavaScript Example

```javascript
const amqp = require('amqplib');

class AsyncMessagePublisher {
  constructor(rabbitmqUrl = 'amqp://localhost') {
    this.url = rabbitmqUrl;
    this.connection = null;
    this.channel = null;
  }

  async connect() {
    this.connection = await amqp.connect(this.url);
    this.channel = await this.connection.createChannel();
    
    // Declare exchange for pub/sub pattern
    await this.channel.assertExchange('orders', 'topic', {
      durable: true
    });
    
    console.log('Connected to RabbitMQ');
  }

  async publishEvent(routingKey, eventData) {
    const message = {
      eventId: crypto.randomUUID(),
      timestamp: new Date().toISOString(),
      data: eventData
    };

    this.channel.publish(
      'orders',
      routingKey,
      Buffer.from(JSON.stringify(message)),
      { persistent: true }
    );

    console.log(`Published event: ${routingKey}`, message.eventId);
  }

  async close() {
    await this.channel?.close();
    await this.connection?.close();
  }
}

// Consumer service example
async function startOrderProcessor() {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();
  
  await channel.assertExchange('orders', 'topic', { durable: true });
  const q = await channel.assertQueue('', { exclusive: true });
  
  // Subscribe to specific order events
  await channel.bindQueue(q.queue, 'orders', 'order.created');
  await channel.bindQueue(q.queue, 'orders', 'order.updated');
  
  channel.consume(q.queue, (msg) => {
    const event = JSON.parse(msg.content.toString());
    console.log('Processing order event:', event);
    
    // Acknowledge message after processing
    channel.ack(msg);
  });
}

// Usage
const publisher = new AsyncMessagePublisher();
await publisher.connect();
await publisher.publishEvent('order.created', { orderId: 12345, total: 99.99 });
```

## ⚖️ When To Use / When To Avoid

**Use Synchronous (REST/gRPC) when:**
- You need immediate response or strong consistency
- The operation is genuinely request-response by nature (user login, fetch profile)
- Services are tightly related in the business domain
- You're building simple, read-heavy operations

**Use Asynchronous (Events/Messages) when:**
- Operations can be processed later without blocking the user
- Multiple services need to react to the same event
- You need to handle traffic spikes with queuing
- Building audit logs, notifications, or analytics pipelines

**Avoid microservices communication altogether when:**
- Your team is small (<10 engineers) and a monolith would suffice
- You're experiencing chatty communication (services calling each other repeatedly)
- Network latency becomes a bottleneck—consider merging services

## 📚 Further Reading

- [Microsoft Azure: Inter-service communication patterns](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/interservice-communication) - Comprehensive guide on sync vs async patterns
- [Martin Fowler: Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html) - Deep dive into event-driven microservices
- [Python aiohttp documentation](https://docs.aiohttp.org/en/stable/) - Async HTTP client/server framework
- [RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html) - Official guides for message queue patterns
- [gRPC documentation](https://grpc.io/docs/) - High-performance R