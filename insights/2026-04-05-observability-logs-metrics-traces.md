# 📌 Observability: logs, metrics, traces
*April 05, 2026 · Daily Dev Insight*

## 🧠 Overview

Observability isn't just monitoring with extra steps—it's the difference between frantically guessing what broke at 3 AM and having a clear forensic trail to the root cause. The three pillars of observability (logs, metrics, and traces) work together like a crime scene investigation team: metrics tell you *something happened*, logs tell you *what happened*, and traces tell you *how it happened* across your entire system.

The magic happens when these three data types correlate. A spike in error rates (metric) points you to specific error logs, which contain trace IDs that let you follow a request's journey through dozens of microservices. Without this correlation, you're debugging with a blindfold on. The best observability strategies treat these pillars as interconnected data streams, not separate monitoring silos.

## 💡 Key Concepts

• **Structured logging beats printf debugging**: JSON logs with consistent fields enable powerful queries and automatic parsing by observability tools
• **Metrics drive alerts, traces drive debugging**: Use metrics for real-time alerting on SLIs, but lean on distributed traces to understand complex failure modes
• **Cardinality is your budget**: High-cardinality metrics (like user IDs) can explode costs—use tags wisely and prefer sampling for traces
• **Context propagation is everything**: Trace context must flow through your entire request path, including async operations and message queues
• **Sampling strategies prevent data drowning**: Collect 100% of errors but sample successful traces—you need enough signal without overwhelming noise

## 🐍 Python Example

```python
import logging
import time
import random
from opentelemetry import trace, metrics
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from prometheus_client import Counter, Histogram, start_http_server

# Configure structured logging
logging.basicConfig(
    level=logging.INFO,
    format='{"timestamp":"%(asctime)s","level":"%(levelname)s","message":"%(message)s","trace_id":"%(trace_id)s"}',
    handlers=[logging.StreamHandler()]
)

# Initialize observability stack
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Metrics
REQUEST_COUNT = Counter('http_requests_total', 'Total HTTP requests', ['method', 'status'])
REQUEST_DURATION = Histogram('http_request_duration_seconds', 'HTTP request duration')

class ObservableService:
    def __init__(self):
        self.logger = logging.getLogger(__name__)
    
    def process_order(self, order_id: str, user_id: str):
        # Start distributed trace
        with tracer.start_as_current_span("process_order") as span:
            span.set_attribute("order.id", order_id)
            span.set_attribute("user.id", user_id)
            
            # Add trace context to logs
            trace_id = format(span.get_span_context().trace_id, '032x')
            
            with REQUEST_DURATION.time():
                try:
                    self.logger.info(f"Processing order {order_id} for user {user_id}", 
                                   extra={"trace_id": trace_id, "order_id": order_id})
                    
                    # Simulate processing with nested spans
                    self._validate_order(order_id)
                    self._charge_payment(order_id)
                    
                    REQUEST_COUNT.labels(method='POST', status='200').inc()
                    span.set_attribute("order.status", "completed")
                    
                except Exception as e:
                    REQUEST_COUNT.labels(method='POST', status='500').inc()
                    span.record_exception(e)
                    span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
                    self.logger.error(f"Order processing failed: {str(e)}", 
                                    extra={"trace_id": trace_id, "error": str(e)})
                    raise
    
    def _validate_order(self, order_id: str):
        with tracer.start_as_current_span("validate_order"):
            time.sleep(random.uniform(0.1, 0.3))  # Simulate work
            if random.random() < 0.1:  # 10% failure rate
                raise ValueError("Invalid order data")
    
    def _charge_payment(self, order_id: str):
        with tracer.start_as_current_span("charge_payment"):
            time.sleep(random.uniform(0.2, 0.5))  # Simulate payment processing

# Start metrics server and run service
if __name__ == "__main__":
    start_http_server(8000)  # Prometheus metrics endpoint
    service = ObservableService()
    service.process_order("order-123", "user-456")
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const winston = require('winston');
const { trace, metrics } = require('@opentelemetry/api');
const { NodeTracerProvider } = require('@opentelemetry/sdk-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const promClient = require('prom-client');

// Configure structured logging with trace correlation
const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json(),
    winston.format.printf(({ timestamp, level, message, ...meta }) => {
      const activeSpan = trace.getActiveSpan();
      const traceId = activeSpan?.spanContext().traceId || 'no-trace';
      return JSON.stringify({ timestamp, level, message, traceId, ...meta });
    })
  ),
  transports: [new winston.transports.Console()]
});

// Initialize metrics
const httpRequestsTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status']
});

const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route']
});

// Tracer setup
const tracer = trace.getTracer('user-service');

const app = express();
app.use(express.json());

// Observability middleware
app.use((req, res, next) => {
  const span = tracer.startSpan(`${req.method} ${req.path}`);
  const endTimer = httpRequestDuration.startTimer({ 
    method: req.method, 
    route: req.path 
  });
  
  // Inject trace context into request
  req.span = span;
  req.traceId = span.spanContext().traceId;
  
  res.on('finish', () => {
    httpRequestsTotal.inc({ 
      method: req.method, 
      route: req.path, 
      status: res.statusCode 
    });
    endTimer();
    span.setStatus({ code: res.statusCode >= 400 ? 2 : 1 }); // Error or OK
    span.end();
  });
  
  next();
});

// Business logic with observability
app.post('/users/:id/orders', async (req, res) => {
  const { id: userId } = req.params;
  const { items, total } = req.body;
  
  logger.info('Creating order', { 
    userId, 
    itemCount: items?.length, 
    total,
    traceId: req.traceId 
  });
  
  try {
    // Create child span for order validation
    const validationSpan = tracer.startSpan('validate_order', { parent: req.span });
    await validateOrder(