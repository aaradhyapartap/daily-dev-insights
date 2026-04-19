# 📌 Microservices communication patterns
*April 19, 2026 · Daily Dev Insight*

## 🧠 Overview

When you're building distributed systems, the way your services talk to each other can make or break your architecture. Microservices communication patterns aren't just about moving data around—they're about defining the reliability, scalability, and maintainability of your entire system. The choice between synchronous REST calls, asynchronous messaging, or event streaming fundamentally shapes how your services handle failures, scale under load, and evolve over time.

The real challenge isn't picking a single pattern, but knowing when to use each one. A user authentication flow might need the immediate consistency of synchronous communication, while order processing could benefit from the resilience of asynchronous messaging. Smart engineers design for the specific needs of each interaction, not just the patterns they're comfortable with.

Most production systems end up as hybrids—using REST for simple queries, message queues for reliable processing, and event streams for real-time updates. The key is establishing clear contracts and monitoring strategies that work across all your communication patterns.

## 💡 Key Concepts

• **Synchronous vs Asynchronous**: Sync communication (REST, gRPC) provides immediate feedback but creates tight coupling. Async patterns (message queues, events) improve resilience but add complexity in handling eventual consistency.

• **Circuit Breaker Pattern**: Essential for sync communication—automatically fail fast when downstream services are unhealthy, preventing cascade failures that can take down your entire system.

• **Event-Driven Architecture**: Services publish events about state changes rather than directly calling other services. This creates loose coupling but requires careful event schema design and versioning strategies.

• **Idempotency**: Critical for both sync retries and async message processing. Design operations so they can be safely repeated without side effects—your future self will thank you during outage recovery.

• **Service Mesh**: Infrastructure layer that handles cross-cutting concerns like service discovery, load balancing, and observability. Consider it when you have more than a handful of services communicating.

## 🐍 Python Example

```python
import asyncio
import aiohttp
import json
from typing import Dict, Any
from dataclasses import dataclass
from enum import Enum

class MessageType(Enum):
    ORDER_CREATED = "order.created"
    PAYMENT_PROCESSED = "payment.processed"

@dataclass
class Message:
    type: MessageType
    payload: Dict[str, Any]
    correlation_id: str

class OrderService:
    def __init__(self, payment_service_url: str, message_broker):
        self.payment_service_url = payment_service_url
        self.message_broker = message_broker
        self.circuit_breaker_failures = 0
        self.circuit_breaker_threshold = 3

    async def create_order(self, order_data: Dict[str, Any]) -> Dict[str, Any]:
        # Synchronous call with circuit breaker
        if self.circuit_breaker_failures >= self.circuit_breaker_threshold:
            raise Exception("Payment service circuit breaker open")
        
        try:
            async with aiohttp.ClientSession() as session:
                async with session.post(
                    f"{self.payment_service_url}/validate",
                    json={"amount": order_data["total"]},
                    timeout=aiohttp.ClientTimeout(total=5)
                ) as response:
                    if response.status != 200:
                        raise Exception("Payment validation failed")
                    
                    self.circuit_breaker_failures = 0  # Reset on success
                    
        except Exception as e:
            self.circuit_breaker_failures += 1
            raise e

        # Asynchronous event publishing
        order_event = Message(
            type=MessageType.ORDER_CREATED,
            payload={"order_id": order_data["id"], "customer_id": order_data["customer_id"]},
            correlation_id=order_data["correlation_id"]
        )
        
        await self.message_broker.publish(order_event)
        return {"status": "created", "order_id": order_data["id"]}

    async def handle_payment_processed(self, message: Message):
        """Idempotent event handler"""
        order_id = message.payload.get("order_id")
        # Check if already processed to ensure idempotency
        if not await self._is_already_processed(message.correlation_id):
            await self._fulfill_order(order_id)
            await self._mark_processed(message.correlation_id)
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const axios = require('axios');
const { EventEmitter } = require('events');

class ServiceCommunicator extends EventEmitter {
    constructor() {
        super();
        this.circuitBreakers = new Map();
        this.retryConfig = { maxRetries: 3, baseDelay: 1000 };
    }

    // Synchronous communication with retry and exponential backoff
    async callService(serviceName, endpoint, data, options = {}) {
        const breaker = this.getCircuitBreaker(serviceName);
        
        if (breaker.isOpen()) {
            throw new Error(`Circuit breaker open for ${serviceName}`);
        }

        for (let attempt = 1; attempt <= this.retryConfig.maxRetries; attempt++) {
            try {
                const response = await axios.post(endpoint, data, {
                    timeout: options.timeout || 5000,
                    headers: {
                        'X-Correlation-ID': options.correlationId,
                        'X-Retry-Attempt': attempt.toString()
                    }
                });
                
                breaker.recordSuccess();
                return response.data;
                
            } catch (error) {
                breaker.recordFailure();
                
                if (attempt === this.retryConfig.maxRetries) {
                    throw error;
                }
                
                // Exponential backoff with jitter
                const delay = this.retryConfig.baseDelay * Math.pow(2, attempt - 1);
                const jitter = Math.random() * 0.1 * delay;
                await this.sleep(delay + jitter);
            }
        }
    }

    // Asynchronous event publishing
    publishEvent(eventType, payload, correlationId) {
        const event = {
            type: eventType,
            payload,
            correlationId,
            timestamp: new Date().toISOString(),
            version: '1.0'
        };
        
        this.emit('event', event);
        
        // In production, this would publish to a message broker
        console.log(`Published event: ${eventType}`, event);
    }

    getCircuitBreaker(serviceName) {
        if (!this.circuitBreakers.has(serviceName)) {
            this.circuitBreakers.set(serviceName, new CircuitBreaker());
        }
        return this.circuitBreakers.get(serviceName);
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

class CircuitBreaker {
    constructor(threshold = 5, timeout = 60000) {
        this.failureThreshold = threshold;
        this.timeout = timeout;
        this.failureCount = 0;
        this.lastFailureTime = null;
        this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    }

    isOpen() {
        if (this.state === 'OPEN') {
            if (Date.now() - this.lastFailureTime > this.timeout) {
                this.state = 'HALF_OPEN';
                return false;
            }
            return true;
        }
        return false;
    }

    recordSuccess() {
        this.failureCount = 0;
        this.state = 'CLOSED';
    }

    recordFailure() {
        this.failureCount++;
        this.lastFailureTime = Date.now();
        
        if (this.failureCount >= this.failureThreshold) {
            this.state = 'OPEN';
        }
    }
}
```

## 