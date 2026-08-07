# 📌 Serverless architecture patterns
*August 07, 2026 · Daily Dev Insight*

## 🧠 Overview

Serverless architecture isn't about eliminating servers—it's about eliminating the *worry* about servers. At its core, serverless represents a shift from managing infrastructure to managing functions and events. You write code that responds to triggers (HTTP requests, database changes, file uploads, scheduled tasks), and the cloud provider handles scaling, provisioning, and availability. This paradigm has matured significantly, moving from simple webhook handlers to sophisticated distributed systems.

The real power of serverless patterns emerges when you stop thinking in terms of "applications" and start thinking in terms of "workflows." Each function becomes a focused, single-purpose unit that does one thing well. The architecture patterns that have emerged—event-driven processing, API composition, fan-out/fan-in, CQRS, and choreography over orchestration—are all about composing these small units into reliable systems. However, the abstraction comes with tradeoffs: cold starts, vendor lock-in, debugging complexity, and a fundamentally different mental model for application design.

Understanding serverless patterns means understanding how to handle state in a stateless world, how to manage dependencies when each invocation is ephemeral, and how to orchestrate complex workflows without a persistent orchestrator. It's a powerful paradigm, but only when applied thoughtfully.

## 💡 Key Concepts

- **Event-driven by default**: Everything is a reaction to a trigger. Design your system as a graph of events flowing through functions, not as long-running processes. This naturally leads to decoupled, scalable architectures.

- **Cold starts are real**: The first invocation (or after idle periods) incurs initialization latency. Minimize by keeping functions small, using provisioned concurrency for critical paths, and choosing runtimes wisely (compiled languages warm faster but may have larger payloads).

- **Statelessness requires strategy**: Functions don't persist state between invocations. Use external stores (DynamoDB, Redis, S3) strategically, pass state through event payloads when possible, and embrace idempotency for reliability.

- **Pay-per-execution economics**: Costs scale with actual usage, not capacity. This makes serverless incredibly cost-effective for variable workloads but potentially expensive for constant high-throughput scenarios. Always model your cost at expected scale.

- **Observability is non-negotiable**: With distributed execution across ephemeral containers, structured logging, distributed tracing, and correlation IDs aren't optional—they're survival tools. Invest in observability from day one.

## 🐍 Python Example

```python
import json
import boto3
import os
from datetime import datetime
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.utilities.typing import LambdaContext

# Initialize observability tools
logger = Logger()
tracer = Tracer()

# Client initialization outside handler for connection reuse
s3_client = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['PROCESSED_IMAGES_TABLE'])

@logger.inject_lambda_context
@tracer.capture_lambda_handler
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    """
    Event-driven pattern: Triggered when image uploaded to S3.
    Demonstrates: S3 event processing, DynamoDB recording, error handling.
    """
    try:
        # Parse S3 event (typically contains multiple records)
        for record in event['Records']:
            bucket = record['s3']['bucket']['name']
            key = record['s3']['object']['key']
            
            logger.info(f"Processing image", extra={
                "bucket": bucket, "key": key
            })
            
            # Download and process image (simplified)
            response = s3_client.get_object(Bucket=bucket, Key=key)
            image_size = response['ContentLength']
            
            # Record metadata in DynamoDB with idempotency key
            table.put_item(
                Item={
                    'imageKey': key,
                    'processedAt': datetime.utcnow().isoformat(),
                    'sizeBytes': image_size,
                    'status': 'processed'
                },
                ConditionExpression='attribute_not_exists(imageKey)'
            )
            
        return {
            'statusCode': 200,
            'body': json.dumps({'processed': len(event['Records'])})
        }
        
    except Exception as e:
        logger.exception("Processing failed")
        # Let Lambda retry on failure
        raise
```

## 🟨 JavaScript Example

```javascript
// API Gateway + Lambda pattern with middleware composition
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, QueryCommand } = require('@aws-sdk/lib-dynamodb');

// Initialize DynamoDB client outside handler for reuse
const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

// Middleware pattern for cross-cutting concerns
const withErrorHandling = (handler) => async (event, context) => {
  try {
    return await handler(event, context);
  } catch (error) {
    console.error('Handler error:', error);
    return {
      statusCode: 500,
      body: JSON.stringify({ error: 'Internal server error' }),
      headers: { 'Content-Type': 'application/json' }
    };
  }
};

const withCORS = (handler) => async (event, context) => {
  const response = await handler(event, context);
  return {
    ...response,
    headers: {
      ...response.headers,
      'Access-Control-Allow-Origin': '*'
    }
  };
};

// Core business logic
const getUserOrders = async (event, context) => {
  const userId = event.pathParameters?.userId;
  
  if (!userId) {
    return {
      statusCode: 400,
      body: JSON.stringify({ error: 'userId required' })
    };
  }

  // Query with pagination support
  const params = {
    TableName: process.env.ORDERS_TABLE,
    KeyConditionExpression: 'userId = :userId',
    ExpressionAttributeValues: { ':userId': userId },
    Limit: 20
  };

  const result = await docClient.send(new QueryCommand(params));

  return {
    statusCode: 200,
    body: JSON.stringify({
      orders: result.Items,
      lastKey: result.LastEvaluatedKey
    }),
    headers: { 'Content-Type': 'application/json' }
  };
};

// Compose middleware with handler
exports.handler = withCORS(withErrorHandling(getUserOrders));
```

## ⚖️ When To Use / When To Avoid

**Use serverless when:**
- Traffic is unpredictable or spiky (e-commerce sales, event registrations)
- You want to minimize operational overhead and focus on features
- Workloads are naturally event-driven (webhooks, data processing pipelines)
- You're building MVPs or side projects with minimal baseline costs
- Individual operations complete in under 15 minutes

**Avoid serverless when:**
- You need sub-10ms consistent latency (cold starts won't work)
- Processing requires long-running connections (WebSockets can work but are complex)
- Workload is constant and predictable (traditional compute may be cheaper)
- You require very specific runtime environments or system-level access
- Team lacks experience with distributed systems debugging

## 📚 Further Reading

- [AWS Lambda Best Practices - Official AWS Documentation](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html) — Comprehensive guide covering performance, security, and cost optimization

- [Serverless Patterns Collection - serverlessland.com](https://serverlessland.com/patterns) — Real-world architecture patterns with infrastructure-as-code examples

- [The Serverless Framework Documentation](https://www.serverless.com/framework/docs) — Popular IaC tool for deploying serverless applications across providers

- [Azure Functions JavaScript Developer Guide](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-node) — Microsoft's