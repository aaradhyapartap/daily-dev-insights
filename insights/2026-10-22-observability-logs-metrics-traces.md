# 📌 Observability: logs, metrics, traces
*October 22, 2026 · Daily Dev Insight*

## 🧠 Overview

Observability isn't just monitoring with extra steps—it's the ability to ask arbitrary questions about your system without having to predict every failure mode in advance. While monitoring tells you *what* is broken (via predefined dashboards and alerts), observability lets you investigate *why* it's broken by exploring your system's internal state through external outputs. The three pillars—logs, metrics, and traces—each serve distinct purposes and shine in different scenarios.

Think of it this way: metrics tell you your API latency spiked at 3 AM, logs reveal a database connection timeout error, and traces show you the exact chain of microservices calls that led to the failure. Used together, they transform debugging from hopeful log-grepping into systematic investigation. The real power emerges when you correlate these pillars: attaching trace IDs to logs, emitting metrics from trace spans, and using logs to add context to metric anomalies.

The modern observability stack has matured significantly. OpenTelemetry has become the de facto standard for instrumentation, and vendors have converged around similar data models. But the tooling is only as good as your instrumentation strategy—over-instrumenting creates noise and cost, under-instrumenting leaves blind spots. The key is intentional observability: instrument what matters, with enough context to debug effectively.

## 💡 Key Concepts

- **Logs are discrete events** — Best for debugging specific requests and capturing rich contextual information. Unstructured logs are searchable but expensive; structured logs (JSON) enable aggregation and filtering at scale.

- **Metrics are aggregated measurements** — Perfect for dashboards, alerting, and trend analysis. They're cheap to store and query (cardinality matters!) but lose individual request details. Counter, gauge, and histogram are your fundamental types.

- **Traces show request flows** — Essential for distributed systems. They reveal latency bottlenecks across service boundaries and help you understand system dependencies. Each trace contains spans representing work units.

- **Correlation is the multiplier** — The real value comes from connecting the three: trace IDs in logs, exemplars linking metrics to traces, and span events bridging traces and logs. Without correlation, you're debugging with one eye closed.

- **Cardinality is your enemy** — High-cardinality labels (user IDs, request IDs) will explode your metrics storage costs. Use them in traces and logs, but keep metric labels low-cardinality (service name, endpoint, status code).

## 🐍 Python Example

```python
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.metrics import MeterProvider
import logging
import time

# Initialize OpenTelemetry
trace.set_tracer_provider(TracerProvider())
metrics.set_meter_provider(MeterProvider())

tracer = trace.get_tracer(__name__)
meter = metrics.get_meter(__name__)

# Define metrics
request_counter = meter.create_counter(
    "api_requests_total",
    description="Total API requests"
)
request_duration = meter.create_histogram(
    "api_request_duration_seconds",
    description="API request duration"
)

# Structured logging with trace context
logging.basicConfig(level=logging.INFO, format='%(message)s')
logger = logging.getLogger(__name__)

def process_order(order_id: str, user_id: str):
    """Process an order with full observability."""
    # Start a trace span
    with tracer.start_as_current_span("process_order") as span:
        # Add trace context
        span.set_attribute("order.id", order_id)
        span.set_attribute("user.id", user_id)
        
        # Get trace ID for log correlation
        ctx = span.get_span_context()
        trace_id = format(ctx.trace_id, '032x')
        
        start_time = time.time()
        
        try:
            # Structured log with trace correlation
            logger.info(f'{{"event": "order_processing_started", '
                       f'"order_id": "{order_id}", '
                       f'"trace_id": "{trace_id}"}}')
            
            # Simulate work with nested span
            with tracer.start_as_current_span("validate_inventory"):
                time.sleep(0.1)  # Simulated work
                span.add_event("inventory_validated")
            
            # Record metrics
            request_counter.add(1, {"endpoint": "/orders", "status": "success"})
            
        except Exception as e:
            span.set_status(trace.Status(trace.StatusCode.ERROR))
            span.record_exception(e)
            logger.error(f'{{"event": "order_failed", "error": "{e}", '
                        f'"trace_id": "{trace_id}"}}')
            request_counter.add(1, {"endpoint": "/orders", "status": "error"})
            raise
        finally:
            duration = time.time() - start_time
            request_duration.record(duration, {"endpoint": "/orders"})

# Usage
process_order("ord_12345", "usr_789")
```

## 🟨 JavaScript Example

```javascript
const { trace, metrics, context } = require('@opentelemetry/api');
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { MeterProvider } = require('@opentelemetry/sdk-metrics');

// Initialize providers
const tracerProvider = new NodeTracerProvider();
const meterProvider = new MeterProvider();
trace.setGlobalTracerProvider(tracerProvider);
metrics.setGlobalMeterProvider(meterProvider);

const tracer = trace.getTracer('order-service');
const meter = metrics.getMeter('order-service');

// Create metrics
const requestCounter = meter.createCounter('api_requests_total', {
  description: 'Total API requests'
});

const requestDuration = meter.createHistogram('api_request_duration_ms', {
  description: 'API request duration in milliseconds'
});

async function processPayment(orderId, amount) {
  // Create span with attributes
  return tracer.startActiveSpan('process_payment', async (span) => {
    span.setAttribute('order.id', orderId);
    span.setAttribute('payment.amount', amount);
    
    // Extract trace ID for logging
    const spanContext = span.spanContext();
    const traceId = spanContext.traceId;
    
    const startTime = Date.now();
    
    try {
      // Structured log with trace correlation
      console.log(JSON.stringify({
        event: 'payment_processing_started',
        order_id: orderId,
        amount: amount,
        trace_id: traceId,
        timestamp: new Date().toISOString()
      }));
      
      // Simulate async work with nested span
      await tracer.startActiveSpan('validate_card', async (childSpan) => {
        await new Promise(resolve => setTimeout(resolve, 50));
        childSpan.addEvent('card_validated');
        childSpan.end();
      });
      
      // Record success metrics
      requestCounter.add(1, { 
        endpoint: '/payment', 
        status: 'success' 
      });
      
      span.setStatus({ code: trace.SpanStatusCode.OK });
      
    } catch (error) {
      // Record failure
      span.recordException(error);
      span.setStatus({ 
        code: trace.SpanStatusCode.ERROR,
        message: error.message 
      });
      
      console.error(JSON.stringify({
        event: 'payment_failed',
        error: error.message,
        trace_id: traceId
      }));
      
      requestCounter.add(1, { 
        endpoint: '/payment', 
        status: 'error' 
      });
      
      throw error;
    } finally {
      const duration = Date.now() - startTime;
      requestDuration.record(duration, { endpoint: '/payment' });
      span.end();
    }
  });
}

// Usage