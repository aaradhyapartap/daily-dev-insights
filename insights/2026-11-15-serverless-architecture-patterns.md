# 📌 Serverless architecture patterns
*November 15, 2026 · Daily Dev Insight*

## 🧠 Overview

Serverless architecture isn't about eliminating servers—it's about eliminating server *management*. The paradigm shift here is moving from "always-on" infrastructure to ephemeral, event-driven execution units that scale automatically. You write functions, wire them to triggers, and your cloud provider handles provisioning, scaling, and maintenance. It's infrastructure as a non-problem.

The real power emerges when you combine these functions into architectural patterns: API gateways fronting Lambda functions, event-driven workflows, async processing queues, and scheduled tasks. Each pattern solves specific problems elegantly, but they also introduce distributed system complexity. Your monolith's stack trace becomes a cascade of CloudWatch logs across dozen of functions. The trick is knowing which patterns fit your use case and accepting the tradeoffs.

Modern serverless has matured beyond simple REST endpoints. We're seeing sophisticated patterns like saga orchestration for distributed transactions, CQRS with event sourcing on managed streams, and edge computing with globally distributed functions. The ecosystem has evolved—cold starts are measured in milliseconds, state management solutions are robust, and observability tools actually work. If you dismissed serverless in 2020, it's time for a second look.

## 💡 Key Concepts

- **Event-driven architecture**: Functions respond to events (HTTP requests, database changes, file uploads, scheduled triggers) rather than running continuously. This decouples components and enables elastic scaling.

- **Backend for Frontend (BFF)**: Each client type (web, mobile, IoT) gets its own lightweight serverless API layer that aggregates and shapes backend data specifically for that interface, avoiding over-fetching and under-fetching.

- **Fan-out/Fan-in pattern**: A single event triggers multiple parallel Lambda executions (fan-out) that process data independently, then aggregate results (fan-in). Perfect for parallel processing of large datasets or batch operations.

- **Event sourcing with streams**: Store state changes as immutable events in managed streams (Kinesis, EventBridge) and use serverless functions to build read-optimized views. This gives you audit logs, time travel, and replay capabilities for free.

- **Cold start mitigation**: Use provisioned concurrency for latency-sensitive endpoints, implement lazy loading, minimize dependencies, and consider keeping functions "warm" with scheduled pings only where user experience demands it.

## 🐍 Python Example

```python
import json
import boto3
from datetime import datetime
from decimal import Decimal

# Fan-out/Fan-in pattern for parallel order processing
dynamodb = boto3.resource('dynamodb')
sns = boto3.client('sns')
orders_table = dynamodb.Table('orders')

def lambda_handler(event, context):
    """
    Processes individual order items in parallel.
    Triggered by SQS messages from order submission.
    """
    
    # Parse the order item from event
    for record in event['Records']:
        order_data = json.loads(record['body'])
        order_id = order_data['orderId']
        item = order_data['item']
        
        try:
            # Validate inventory (simulated external call)
            inventory_available = check_inventory(item['sku'], item['quantity'])
            
            if not inventory_available:
                # Publish failure event for saga compensation
                sns.publish(
                    TopicArn='arn:aws:sns:us-east-1:123456789:order-failures',
                    Message=json.dumps({
                        'orderId': order_id,
                        'reason': 'INSUFFICIENT_INVENTORY',
                        'item': item
                    })
                )
                continue
            
            # Update order status atomically
            orders_table.update_item(
                Key={'orderId': order_id},
                UpdateExpression='SET itemStatus.#sku = :status, updatedAt = :now',
                ExpressionAttributeNames={'#sku': item['sku']},
                ExpressionAttributeValues={
                    ':status': 'PROCESSING',
                    ':now': datetime.utcnow().isoformat()
                }
            )
            
            # Trigger downstream payment processing
            sns.publish(
                TopicArn='arn:aws:sns:us-east-1:123456789:payment-queue',
                Message=json.dumps(order_data)
            )
            
        except Exception as e:
            print(f"Error processing order {order_id}: {str(e)}")
            # Dead letter queue handles retry logic
            raise
    
    return {'statusCode': 200, 'body': 'Processed successfully'}

def check_inventory(sku, quantity):
    """Simulates inventory service call"""
    # In production: HTTP call to inventory microservice
    return True
```

## 🟨 JavaScript Example

```javascript
// Backend for Frontend (BFF) pattern with data aggregation
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, QueryCommand, GetCommand } = require('@aws-sdk/lib-dynamodb');

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

exports.handler = async (event) => {
    /**
     * Mobile BFF endpoint - aggregates user profile, recent orders,
     * and recommendations in a single optimized response
     */
    
    const userId = event.pathParameters?.userId;
    
    if (!userId) {
        return {
            statusCode: 400,
            body: JSON.stringify({ error: 'userId required' })
        };
    }
    
    try {
        // Parallel data fetching - all execute simultaneously
        const [profile, orders, recommendations] = await Promise.all([
            getUserProfile(userId),
            getRecentOrders(userId),
            getRecommendations(userId)
        ]);
        
        // Shape response specifically for mobile constraints
        const mobileOptimizedResponse = {
            user: {
                name: profile.name,
                avatar: profile.avatarUrl,
                tier: profile.membershipTier
            },
            recentOrders: orders.slice(0, 3).map(order => ({
                id: order.orderId,
                date: order.createdAt,
                total: order.total,
                status: order.status,
                itemCount: order.items.length
            })),
            recommendations: recommendations.slice(0, 5).map(rec => ({
                id: rec.productId,
                title: rec.name,
                price: rec.price,
                image: rec.thumbnailUrl // Low-res for mobile
            }))
        };
        
        return {
            statusCode: 200,
            headers: {
                'Content-Type': 'application/json',
                'Cache-Control': 'max-age=60' // Client-side cache
            },
            body: JSON.stringify(mobileOptimizedResponse)
        };
        
    } catch (error) {
        console.error('BFF aggregation error:', error);
        return {
            statusCode: 500,
            body: JSON.stringify({ error: 'Failed to load dashboard' })
        };
    }
};

async function getUserProfile(userId) {
    const result = await docClient.send(new GetCommand({
        TableName: 'users',
        Key: { userId }
    }));
    return result.Item;
}

async function getRecentOrders(userId) {
    const result = await docClient.send(new QueryCommand({
        TableName: 'orders',
        IndexName: 'userId-createdAt-index',
        KeyConditionExpression: 'userId = :uid',
        ExpressionAttributeValues: { ':uid': userId },
        ScanIndexForward: false,
        Limit: 10
    }));
    return result.Items;
}

async function getRecommendations(userId) {
    // Simulated ML service call
    return [];
}
```

## ⚖️ When To Use / When To Avoid

**✅ Use serverless when:**
- You have **unpredictable or spiky traffic** (0 to 10k requests/min)
- Building **event-driven workflows** (file processing, webh