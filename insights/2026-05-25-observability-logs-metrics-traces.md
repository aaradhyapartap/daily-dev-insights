# 📌 Observability: logs, metrics, traces
*May 25, 2026 · Daily Dev Insight*

## 🧠 Overview

Observability is your system's ability to explain its internal state through external outputs. While monitoring tells you *what* is happening, observability tells you *why*. The three pillars—logs, metrics, and traces—work together like a detective's toolkit: logs provide the narrative, metrics give you the vital signs, and traces show you the complete journey of a request through your system.

The real magic happens when these three pillars converge. A spike in error metrics leads you to specific log entries, which contain trace IDs that reveal the exact path a failing request took. This correlation transforms debugging from guesswork into methodical investigation. Modern applications are too complex for reactive monitoring alone—you need proactive observability to understand emergent behaviors and edge cases you never anticipated.

## 💡 Key Concepts

• **Structured logging**: Use consistent, machine-readable formats (JSON) with correlation IDs to connect related events across services
• **Golden signals**: Focus metrics on latency, traffic, errors, and saturation—the four indicators that matter most for user experience  
• **Distributed tracing**: Track requests across microservices with span hierarchies to identify bottlenecks in complex call chains
• **High cardinality data**: Modern observability platforms handle millions of unique metric combinations, enabling deep slicing and dicing
• **Sampling strategies**: Balance observability depth with cost by intelligently sampling traces while preserving error cases and outliers

## 🐍 Python Example

```python
import logging
import time
import random
from opentelemetry import trace, metrics
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
import structlog

# Configure structured logging
structlog.configure(
    processors=[structlog.dev.ConsoleRenderer()],
    wrapper_class=structlog.stdlib.BoundLogger,
    logger_factory=structlog.stdlib.LoggerFactory(),
)

# Initialize tracing
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Configure Jaeger exporter
jaeger_exporter = JaegerExporter(
    agent_host_name="localhost",
    agent_port=14268,
)
span_processor = BatchSpanProcessor(jaeger_exporter)
trace.get_tracer_provider().add_span_processor(span_processor)

class OrderService:
    def __init__(self):
        self.logger = structlog.get_logger()
        
    def process_order(self, user_id: int, order_amount: float):
        # Start distributed trace
        with tracer.start_as_current_span("process_order") as span:
            span.set_attribute("user_id", user_id)
            span.set_attribute("order_amount", order_amount)
            
            # Structured logging with trace context
            trace_id = trace.get_current_span().get_span_context().trace_id
            self.logger.info(
                "Processing order",
                user_id=user_id,
                order_amount=order_amount,
                trace_id=hex(trace_id)
            )
            
            # Simulate payment processing with child span
            with tracer.start_as_current_span("payment_processing") as payment_span:
                payment_time = random.uniform(0.1, 2.0)
                time.sleep(payment_time)
                
                # Record custom metric
                payment_span.set_attribute("payment_duration_ms", payment_time * 1000)
                
                if random.random() < 0.1:  # 10% failure rate
                    self.logger.error(
                        "Payment failed",
                        user_id=user_id,
                        error_code="PAYMENT_DECLINED",
                        trace_id=hex(trace_id)
                    )
                    span.set_status(trace.Status(trace.StatusCode.ERROR))
                    return False
                    
            self.logger.info(
                "Order completed successfully",
                user_id=user_id,
                processing_time_ms=payment_time * 1000,
                trace_id=hex(trace_id)
            )
            return True

# Usage
service = OrderService()
service.process_order(12345, 99.99)
```

## 🟨 JavaScript Example

```javascript
const winston = require('winston');
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const { trace, metrics } = require('@opentelemetry/api');

// Initialize OpenTelemetry SDK
const sdk = new NodeSDK({
  traceExporter: new JaegerExporter({
    endpoint: 'http://localhost:14268/api/traces',
  }),
});
sdk.start();

// Configure structured logging
const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'app.log' })
  ]
});

class UserService {
  constructor() {
    this.tracer = trace.getTracer('user-service');
    this.meter = metrics.getMeter('user-service');
    
    // Define custom metrics
    this.requestCounter = this.meter.createCounter('user_requests_total');
    this.responseTimeHistogram = this.meter.createHistogram('user_response_time_ms');
  }
  
  async getUserProfile(userId) {
    const startTime = Date.now();
    
    return this.tracer.startActiveSpan('get_user_profile', async (span) => {
      try {
        span.setAttributes({
          'user.id': userId,
          'service.name': 'user-service'
        });
        
        // Get trace context for logging correlation
        const traceId = span.spanContext().traceId;
        
        logger.info('Fetching user profile', {
          userId,
          traceId,
          operation: 'get_user_profile'
        });
        
        // Simulate database call with child span
        const userData = await this.tracer.startActiveSpan('db_query', async (dbSpan) => {
          dbSpan.setAttributes({
            'db.operation': 'SELECT',
            'db.table': 'users'
          });
          
          // Simulate variable response time
          const delay = Math.random() * 500 + 50;
          await new Promise(resolve => setTimeout(resolve, delay));
          
          if (Math.random() < 0.05) { // 5% error rate
            throw new Error('Database connection timeout');
          }
          
          return { id: userId, name: 'John Doe', email: 'john@example.com' };
        });
        
        const duration = Date.now() - startTime;
        
        // Record metrics
        this.requestCounter.add(1, { 
          status: 'success',
          endpoint: 'get_user_profile' 
        });
        this.responseTimeHistogram.record(duration);
        
        logger.info('User profile retrieved successfully', {
          userId,
          traceId,
          duration,
          cacheHit: Math.random() > 0.3
        });
        
        span.setStatus({ code: trace.SpanStatusCode.OK });
        return userData;
        
      } catch (error) {
        const duration = Date.now() - startTime;
        
        // Log error with full context
        logger.error('Failed to fetch user profile', {
          userId,
          traceId: span.spanContext().traceId,
          error: error.message,
          duration
        });
        
        // Record error metrics
        this.requestCounter.add(1, { 
          status: 'error',
          endpoint: 'get_user_profile' 