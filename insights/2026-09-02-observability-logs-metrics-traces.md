# 📌 Observability: logs, metrics, traces
*September 02, 2026 · Daily Dev Insight*

## 🧠 Overview

Observability isn't just monitoring with a fancy name—it's a fundamental shift in how we understand system behavior. While monitoring tells you *what* is broken, observability helps you understand *why* it's broken, even for failure modes you've never seen before. The three pillars—logs, metrics, and traces—each answer different questions: logs tell the story of discrete events, metrics provide aggregated numerical data over time, and traces follow the journey of a request through your distributed system.

The real power emerges when these three pillars work together. A metric spike alerts you to increased latency, traces show you which service is slow, and logs from that service reveal the root cause. Without this trinity, you're debugging blindfolded in production. Modern systems are too complex, too distributed, and too dynamic for traditional monitoring alone.

Think of it this way: metrics are your dashboard gauges, logs are your detailed maintenance records, and traces are your GPS tracking for individual requests. You wouldn't drive a car with only one of these—why would you run a production system that way?

## 💡 Key Concepts

- **Structured Logging**: Always log in structured formats (JSON) with consistent field names. This makes logs searchable, aggregatable, and actually useful during incidents instead of just noise.

- **Golden Signals**: Focus your metrics on the four golden signals—latency, traffic, errors, and saturation. Everything else is supplementary. These tell you 80% of what you need to know about system health.

- **Distributed Context Propagation**: Traces are only valuable if you can follow a request across service boundaries. Use correlation IDs and OpenTelemetry standards to maintain context through your entire stack.

- **Cardinality Matters**: High-cardinality data (user IDs, request IDs) belongs in logs and traces, not metrics. Metrics should be aggregatable; use labels wisely or face exploding storage costs.

- **Sampling Strategies**: You can't trace every request in production without destroying performance. Implement intelligent sampling—always trace errors, sample successful requests based on load.

## 🐍 Python Example

```python
import logging
import time
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.metrics import MeterProvider
import json

# Configure structured logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Initialize OpenTelemetry
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)
meter = metrics.get_meter(__name__)

# Create metrics
request_counter = meter.create_counter(
    "http_requests_total",
    description="Total HTTP requests"
)
request_duration = meter.create_histogram(
    "http_request_duration_seconds",
    description="HTTP request duration"
)

def process_order(order_id: str, user_id: str):
    # Start a trace span for this operation
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        span.set_attribute("user.id", user_id)
        
        start_time = time.time()
        
        try:
            # Structured log with context
            logger.info(json.dumps({
                "event": "order_processing_started",
                "order_id": order_id,
                "user_id": user_id,
                "trace_id": format(span.get_span_context().trace_id, '032x')
            }))
            
            # Simulate processing
            time.sleep(0.1)
            
            # Record successful metrics
            request_counter.add(1, {"status": "success", "operation": "process_order"})
            
        except Exception as e:
            # Log error with full context
            logger.error(json.dumps({
                "event": "order_processing_failed",
                "order_id": order_id,
                "error": str(e),
                "trace_id": format(span.get_span_context().trace_id, '032x')
            }))
            request_counter.add(1, {"status": "error", "operation": "process_order"})
            span.record_exception(e)
            raise
        finally:
            duration = time.time() - start_time
            request_duration.record(duration, {"operation": "process_order"})

# Usage
process_order("ord_12345", "usr_67890")
```

## 🟨 JavaScript Example

```javascript
const { trace, metrics } = require('@opentelemetry/api');
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { MeterProvider } = require('@opentelemetry/sdk-metrics');
const winston = require('winston');

// Configure structured logging with Winston
const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [new winston.transports.Console()]
});

// Initialize OpenTelemetry
const tracerProvider = new NodeTracerProvider();
trace.setGlobalTracerProvider(tracerProvider);
const tracer = trace.getTracer('payment-service');

const meterProvider = new MeterProvider();
metrics.setGlobalMeterProvider(meterProvider);
const meter = metrics.getMeter('payment-service');

// Create metrics
const paymentCounter = meter.createCounter('payments_total', {
  description: 'Total payment attempts'
});
const paymentDuration = meter.createHistogram('payment_duration_ms', {
  description: 'Payment processing duration'
});

async function processPayment(paymentId, amount, currency) {
  const span = tracer.startSpan('process_payment');
  span.setAttributes({
    'payment.id': paymentId,
    'payment.amount': amount,
    'payment.currency': currency
  });
  
  const startTime = Date.now();
  
  try {
    logger.info({
      event: 'payment_started',
      payment_id: paymentId,
      amount: amount,
      currency: currency,
      trace_id: span.spanContext().traceId
    });
    
    // Simulate payment gateway call
    await new Promise(resolve => setTimeout(resolve, 150));
    
    paymentCounter.add(1, { status: 'success', currency });
    
    logger.info({
      event: 'payment_completed',
      payment_id: paymentId,
      trace_id: span.spanContext().traceId
    });
    
  } catch (error) {
    logger.error({
      event: 'payment_failed',
      payment_id: paymentId,
      error: error.message,
      trace_id: span.spanContext().traceId
    });
    
    paymentCounter.add(1, { status: 'error', currency });
    span.recordException(error);
    throw error;
  } finally {
    const duration = Date.now() - startTime;
    paymentDuration.record(duration, { currency });
    span.end();
  }
}

// Usage
processPayment('pay_xyz789', 99.99, 'USD');
```

## ⚖️ When To Use / When To Avoid

**Use comprehensive observability when:**
- Running distributed microservices architectures
- System complexity makes traditional debugging impossible
- You need to diagnose issues you've never encountered before
- Compliance or SLAs require detailed audit trails
- Your team is on-call and needs to resolve incidents quickly

**Avoid or simplify when:**
- Building a simple monolith with predictable failure modes
- Operating resource-constrained embedded systems
- Storage and bandwidth costs outweigh diagnostic benefits
- Your system is so small you can mentally model all interactions
- Privacy regulations prohibit detailed request logging

## 📚 Further Reading

- [OpenTelemetry Official Documentation](https://opentelemetry.io/docs/) - The standard for observability instrumentation across languages and platforms
- [Distributed Tracing: A Complete Guide](https://www.datadoghq.com/knowledge-center/distributed