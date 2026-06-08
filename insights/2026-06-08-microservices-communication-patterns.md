# 📌 Microservices communication patterns
*June 08, 2026 · Daily Dev Insight*

## 🧠 Overview

Microservices communication patterns are the architectural blueprints that determine how distributed services talk to each other. Unlike monolithic applications where components communicate through direct method calls, microservices must navigate network boundaries, handle failures gracefully, and maintain data consistency across service boundaries. The pattern you choose fundamentally shapes your system's reliability, performance, and complexity.

The two primary paradigms—synchronous and asynchronous communication—each solve different problems. Synchronous patterns (HTTP REST, gRPC) offer simplicity and immediate consistency but create tight coupling and potential cascade failures. Asynchronous patterns (message queues, event streams) provide better decoupling and resilience but introduce eventual consistency challenges. Modern systems often blend both approaches strategically.

Understanding these patterns isn't just about picking the right tool—it's about designing for failure, planning for scale, and maintaining system observability as complexity grows. The wrong communication pattern can turn a well-intentioned microservices architecture into a distributed monolith nightmare.

## 💡 Key Concepts

• **Synchronous vs Asynchronous**: Sync communication (REST, gRPC) blocks until response arrives; async communication (events, messaging) allows fire-and-forget or eventual consistency patterns
• **Circuit Breaker Pattern**: Prevents cascade failures by automatically failing fast when downstream services are unhealthy, with configurable recovery mechanisms
• **Event-Driven Architecture**: Services publish domain events when state changes occur, allowing other services to react without direct coupling
• **Saga Pattern**: Manages distributed transactions across microservices through choreographed or orchestrated sequences of compensating actions
• **Service Mesh**: Infrastructure layer that handles service-to-service communication concerns like retry logic, load balancing, and encryption transparently

## 🐍 Python Example

```python
import asyncio
import aiohttp
from dataclasses import dataclass
from typing import Optional
import json

@dataclass
class OrderEvent:
    order_id: str
    customer_id: str
    amount: float
    event_type: str

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    async def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if asyncio.get_event_loop().time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = await func(*args, **kwargs)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
                self.last_failure_time = asyncio.get_event_loop().time()
            raise e

class EventBus:
    def __init__(self):
        self.subscribers = {}
    
    def subscribe(self, event_type: str, handler):
        if event_type not in self.subscribers:
            self.subscribers[event_type] = []
        self.subscribers[event_type].append(handler)
    
    async def publish(self, event: OrderEvent):
        handlers = self.subscribers.get(event.event_type, [])
        await asyncio.gather(*[handler(event) for handler in handlers])

# Example microservice using both patterns
class PaymentService:
    def __init__(self, event_bus: EventBus):
        self.circuit_breaker = CircuitBreaker()
        self.event_bus = event_bus
    
    async def process_payment(self, order_event: OrderEvent):
        # Synchronous call to external payment gateway
        try:
            result = await self.circuit_breaker.call(self._charge_payment, order_event.amount)
            # Publish async event for other services
            payment_event = OrderEvent(
                order_event.order_id, order_event.customer_id, 
                order_event.amount, "payment_processed"
            )
            await self.event_bus.publish(payment_event)
            return result
        except Exception:
            # Publish failure event
            failure_event = OrderEvent(
                order_event.order_id, order_event.customer_id,
                order_event.amount, "payment_failed"
            )
            await self.event_bus.publish(failure_event)
            raise
    
    async def _charge_payment(self, amount: float):
        # Simulate external API call
        async with aiohttp.ClientSession() as session:
            async with session.post(
                "https://payment-gateway.api/charge",
                json={"amount": amount}
            ) as response:
                return await response.json()
```

## 🟨 JavaScript Example

```javascript
const EventEmitter = require('events');
const axios = require('axios');

class RetryPolicy {
  constructor(maxRetries = 3, baseDelay = 1000) {
    this.maxRetries = maxRetries;
    this.baseDelay = baseDelay;
  }

  async execute(fn) {
    let lastError;
    
    for (let attempt = 0; attempt <= this.maxRetries; attempt++) {
      try {
        return await fn();
      } catch (error) {
        lastError = error;
        if (attempt < this.maxRetries) {
          const delay = this.baseDelay * Math.pow(2, attempt); // Exponential backoff
          await new Promise(resolve => setTimeout(resolve, delay));
        }
      }
    }
    throw lastError;
  }
}

class MessageBroker extends EventEmitter {
  constructor() {
    super();
    this.topics = new Map();
  }

  async publish(topic, message) {
    console.log(`Publishing to ${topic}:`, message);
    this.emit(topic, message);
    
    // Simulate message persistence/delivery
    return new Promise((resolve) => {
      setTimeout(() => resolve({ messageId: Date.now() }), 10);
    });
  }

  subscribe(topic, handler) {
    this.on(topic, handler);
  }
}

class InventoryService {
  constructor(messageBroker) {
    this.messageBroker = messageBroker;
    this.retryPolicy = new RetryPolicy();
    this.inventory = new Map([['item1', 100], ['item2', 50]]);
    
    // Subscribe to order events
    this.messageBroker.subscribe('order.created', this.handleOrderCreated.bind(this));
  }

  async handleOrderCreated(orderData) {
    try {
      console.log('Processing order:', orderData.orderId);
      
      // Check inventory synchronously
      const available = this.inventory.get(orderData.itemId) || 0;
      
      if (available >= orderData.quantity) {
        // Update inventory
        this.inventory.set(orderData.itemId, available - orderData.quantity);
        
        // Notify other services asynchronously
        await this.messageBroker.publish('inventory.reserved', {
          orderId: orderData.orderId,
          itemId: orderData.itemId,
          quantity: orderData.quantity
        });
        
        // Sync call to warehouse with retry logic
        await this.retryPolicy.execute(async () => {
          const response = await axios.post('http://warehouse-service/reserve', {
            itemId: orderData.itemId,
            quantity: orderData.quantity
          });
          return response.data;
        });
        
      } else {
        // Publish failure event
        await this.messageBroker.publish('inventory.insufficient', {
          orderId: orderData.orderId,
          item