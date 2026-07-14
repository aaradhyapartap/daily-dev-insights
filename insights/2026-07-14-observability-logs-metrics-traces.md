# 📌 Observability: logs, metrics, traces
*July 14, 2026 · Daily Dev Insight*

## 🧠 Overview

Observability isn't just monitoring with a fancy name—it's the difference between knowing *that* your system is broken and understanding *why* it's broken. While traditional monitoring tells you when predefined thresholds are crossed, observability lets you ask arbitrary questions about your system's behavior after the fact. The three pillars—logs, metrics, and traces—work together like a detective's toolkit: logs are your witness statements, metrics are your statistical evidence, and traces are your crime scene reconstruction.

The real power emerges when you correlate these three data types. A spike in error rate (metric) can lead you to specific failed requests (traces), which point to the exact exception messages (logs). Modern distributed systems are too complex to debug with logs alone or dashboard-watch with metrics. You need all three, properly instrumented and queryable, to achieve true observability.

Think of it this way: logs tell you *what happened*, metrics tell you *how much* and *how often*, and traces tell you *where* the problem occurred in your call chain. Master all three, and you'll spend less time in war rooms and more time shipping features.

## 💡 Key Concepts

- **Logs are unstructured events**: Best for debugging specific issues and understanding context. Use structured logging (JSON) to make them queryable. Don't over-log; high cardinality data kills performance and costs money.

- **Metrics are aggregated measurements**: Perfect for dashboards, alerts, and understanding trends over time. Use counters for things that increment (requests, errors), gauges for point-in-time values (CPU, memory), and histograms for distributions (latency percentiles).

- **Traces follow request flows**: Critical for distributed systems. They show you the path a request takes through microservices, with timing for each hop. Sampling is essential—you can't trace everything in production without massive overhead.

- **Correlation is the superpower**: Use correlation IDs to connect logs, metrics, and traces for the same request. This turns three separate data streams into one coherent story about what went wrong.

- **Cardinality matters**: High-cardinality data (like user IDs in metric labels) will explode your costs and query times. Keep metrics low-cardinality; save high-cardinality data for logs and trace attributes.

## 🐍 Python Example

```python
import logging
import time
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.metrics import MeterProvider
from prometheus_client import Counter, Histogram

# Configure structured logging
logging.basicConfig(level=logging.INFO, format='%(message)s')
logger = logging.getLogger(__name__)

# Initialize OpenTelemetry
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Define metrics
request_counter = Counter('api_requests_total', 'Total API requests', ['endpoint', 'status'])
request_duration = Histogram('api_request_duration_seconds', 'Request duration', ['endpoint'])

def process_order(order_id: str, user_id: str):
    """Example function demonstrating all three observability pillars."""
    # Start a trace span
    with tracer.start_as_current_span("process_order") as span:
        # Add trace attributes
        span.set_attribute("order.id", order_id)
        span.set_attribute("user.id", user_id)
        
        start_time = time.time()
        
        try:
            # Structured log with correlation info
            logger.info({
                "event": "order_processing_started",
                "order_id": order_id,
                "trace_id": format(span.get_span_context().trace_id, '032x')
            })
            
            # Simulate work
            time.sleep(0.1)
            
            # Record success metrics
            request_counter.labels(endpoint='process_order', status='success').inc()
            
        except Exception as e:
            # Log the error with context
            logger.error({
                "event": "order_processing_failed",
                "order_id": order_id,
                "error": str(e),
                "trace_id": format(span.get_span_context().trace_id, '032x')
            })
            span.record_exception(e)
            request_counter.labels(endpoint='process_order', status='error').inc()
            raise
        finally:
            # Record duration metric
            duration = time.time() - start_time
            request_duration.labels(endpoint='process_order').observe(duration)
```

## 🟨 JavaScript Example

```javascript
const winston = require('winston');
const { trace, metrics } = require('@opentelemetry/api');
const { PrometheusExporter } = require('@opentelemetry/exporter-prometheus');

// Configure structured logging
const logger = winston.createLogger({
  format: winston.format.json(),
  transports: [new winston.transports.Console()]
});

// Initialize tracer
const tracer = trace.getTracer('example-service');

// Define metrics
const meter = metrics.getMeter('example-service');
const requestCounter = meter.createCounter('api_requests_total', {
  description: 'Total API requests'
});
const requestHistogram = meter.createHistogram('api_request_duration', {
  description: 'Request duration in milliseconds'
});

async function processPayment(paymentId, amount, userId) {
  // Start distributed trace
  return tracer.startActiveSpan('process_payment', async (span) => {
    const startTime = Date.now();
    
    // Add trace attributes for debugging
    span.setAttribute('payment.id', paymentId);
    span.setAttribute('payment.amount', amount);
    span.setAttribute('user.id', userId);
    
    try {
      // Structured log with trace correlation
      logger.info({
        event: 'payment_processing_started',
        payment_id: paymentId,
        amount: amount,
        trace_id: span.spanContext().traceId
      });
      
      // Simulate async payment processing
      await new Promise(resolve => setTimeout(resolve, 150));
      
      // Record success metric
      requestCounter.add(1, { endpoint: 'process_payment', status: 'success' });
      
      return { success: true, paymentId };
      
    } catch (error) {
      // Log error with full context
      logger.error({
        event: 'payment_processing_failed',
        payment_id: paymentId,
        error: error.message,
        trace_id: span.spanContext().traceId
      });
      
      span.recordException(error);
      requestCounter.add(1, { endpoint: 'process_payment', status: 'error' });
      throw error;
      
    } finally {
      const duration = Date.now() - startTime;
      requestHistogram.record(duration, { endpoint: 'process_payment' });
      span.end();
    }
  });
}
```

## ⚖️ When To Use / When To Avoid

**Use observability when:**
- ✅ Running distributed systems or microservices
- ✅ You need to debug production issues quickly
- ✅ Understanding user experience and system behavior matters
- ✅ You have unpredictable failure modes
- ✅ Compliance requires audit trails

**Avoid or minimize when:**
- ❌ Building a simple monolith with one database (logs alone might suffice)
- ❌ Cost constraints are severe (observability platforms are expensive)
- ❌ Your traffic is low enough to reproduce issues locally
- ❌ You're over-instrumenting without clear questions to answer

## 📚 Further Reading

- [OpenTelemetry Official Documentation](https://opentelemetry.io/docs/) - The standard for unified observability instrumentation
- [Prometheus Best Practices](https://prometheus.io/docs/practices/naming/) - How to design effective metrics and avoid cardinality explosions
- [Google's SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems