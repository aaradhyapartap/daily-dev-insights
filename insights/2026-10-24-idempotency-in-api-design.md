# 📌 Idempotency in API design
*October 24, 2026 · Daily Dev Insight*

## 🧠 Overview

Idempotency is one of those concepts that sounds academic until you're debugging why a user was charged three times for the same purchase. In API design, an idempotent operation is one that produces the same result no matter how many times you execute it with the same parameters. Making a PUT request to update a user's email? Idempotent. Charging a credit card? Definitely not idempotent by default—and that's where things get interesting.

The real-world value of idempotency isn't just theoretical elegance; it's about building resilient systems in an unreliable world. Network requests fail, timeout, and retry. Mobile apps lose connectivity mid-request. Users double-click submit buttons. Without idempotency, these everyday occurrences become data corruption nightmares. The trick is knowing where to enforce it and how to implement it without overcomplicating your API.

Most developers get idempotency wrong by treating it as binary: either an endpoint is idempotent or it isn't. The truth is more nuanced. GET and PUT are naturally idempotent by HTTP standards, but POST and DELETE require intentional design. You need idempotency keys, request deduplication, and careful state management. Done right, it's invisible to your users; done wrong, it's midnight debugging sessions.

## 💡 Key Concepts

- **Idempotency ≠ Safety**: Safe operations (like GET) don't modify state at all. Idempotent operations can modify state, but repeated calls don't create additional changes. DELETE is idempotent but not safe.

- **Idempotency Keys**: For non-idempotent operations like POST, clients send a unique idempotency key (usually a UUID) in headers. The server stores this key with the result, returning the cached response for duplicate requests.

- **HTTP Method Semantics**: GET, PUT, DELETE, and PATCH (when properly implemented) should be idempotent by design. POST is the wild card—payments, order creation, and message sending need explicit idempotency handling.

- **Time Windows Matter**: Most systems don't store idempotency keys forever. A 24-72 hour window is typical. After that, the same key might be treated as a new request.

- **State-Based vs Result-Based**: Deleting a resource that's already gone can return 204 (state-based idempotency) or 404 (strict interpretation). Both are valid; consistency is what matters.

## 🐍 Python Example

```python
from flask import Flask, request, jsonify
from datetime import datetime, timedelta
import uuid

app = Flask(__name__)

# In-memory store (use Redis/database in production)
idempotency_store = {}
payments = {}

@app.route('/api/payments', methods=['POST'])
def create_payment():
    # Extract idempotency key from header
    idempotency_key = request.headers.get('Idempotency-Key')
    
    if not idempotency_key:
        return jsonify({'error': 'Idempotency-Key header required'}), 400
    
    # Check if we've seen this request before
    if idempotency_key in idempotency_store:
        cached_response = idempotency_store[idempotency_key]
        
        # Verify the request hasn't expired (24 hour window)
        if datetime.now() < cached_response['expires_at']:
            return jsonify(cached_response['result']), cached_response['status']
    
    # Process the payment (simulate with UUID)
    payment_data = request.get_json()
    payment_id = str(uuid.uuid4())
    
    result = {
        'payment_id': payment_id,
        'amount': payment_data.get('amount'),
        'status': 'completed',
        'timestamp': datetime.now().isoformat()
    }
    
    payments[payment_id] = result
    
    # Cache the response with expiration
    idempotency_store[idempotency_key] = {
        'result': result,
        'status': 201,
        'expires_at': datetime.now() + timedelta(hours=24)
    }
    
    return jsonify(result), 201

# Idempotent update example
@app.route('/api/users/<user_id>', methods=['PUT'])
def update_user(user_id):
    # PUT is naturally idempotent - same input = same state
    user_data = request.get_json()
    # Update user... (implementation omitted)
    return jsonify({'user_id': user_id, 'updated': True}), 200
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const crypto = require('crypto');

const app = express();
app.use(express.json());

// Idempotency cache (use Redis in production)
const idempotencyCache = new Map();
const EXPIRY_MS = 24 * 60 * 60 * 1000; // 24 hours

// Middleware to handle idempotency for POST requests
const idempotencyMiddleware = (req, res, next) => {
  if (req.method !== 'POST') {
    return next();
  }

  const idempotencyKey = req.headers['idempotency-key'];
  
  if (!idempotencyKey) {
    return res.status(400).json({ 
      error: 'Idempotency-Key header required for POST requests' 
    });
  }

  const cached = idempotencyCache.get(idempotencyKey);
  
  if (cached && Date.now() < cached.expiresAt) {
    // Return cached response
    return res.status(cached.statusCode).json(cached.body);
  }

  // Attach function to cache the response
  res.cacheIdempotentResponse = (statusCode, body) => {
    idempotencyCache.set(idempotencyKey, {
      statusCode,
      body,
      expiresAt: Date.now() + EXPIRY_MS
    });
  };

  next();
};

app.use(idempotencyMiddleware);

app.post('/api/orders', async (req, res) => {
  // Simulate order creation
  const orderId = crypto.randomUUID();
  const { items, total } = req.body;

  const order = {
    orderId,
    items,
    total,
    status: 'confirmed',
    createdAt: new Date().toISOString()
  };

  // Cache this response for future duplicate requests
  res.cacheIdempotentResponse(201, order);
  
  res.status(201).json(order);
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

## ⚖️ When To Use / When To Avoid

**✅ When To Use:**
- Payment processing, order creation, or any financial transaction
- APIs consumed by mobile apps (unreliable networks mean frequent retries)
- Webhook endpoints that might receive duplicate deliveries
- Any POST operation where duplicate execution would cause problems

**❌ When To Avoid:**
- Truly unique operations by design (uploading different files, logging events)
- Read-only operations (already idempotent by nature)
- Internal APIs with guaranteed single execution
- Operations where duplicates are semantically meaningful (chat messages, analytics events)

## 📚 Further Reading

- [Stripe's Idempotent Requests Guide](https://stripe.com/docs/api/idempotent_requests) - Real-world implementation from a payment API
- [RFC 7231: HTTP Semantics](https://tools.ietf.org/html/rfc7231#section-4.2.2) - Official HTTP method idempotency specifications
- [AWS API Gateway Idempotency](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-parameter-mapping.html) - Cloud-native idempotency patterns
- [Django REST Framework: Idempotency](https://www.django-rest-framework.org/