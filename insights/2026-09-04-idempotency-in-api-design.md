# 📌 Idempotency in API design
*September 04, 2026 · Daily Dev Insight*

## 🧠 Overview

Idempotency is one of those concepts that separates production-ready APIs from fragile ones. At its core, an idempotent operation produces the same result no matter how many times you execute it with the same inputs. Think of it like a light switch set to "on" — flipping it to "on" ten more times doesn't make the room any brighter. In API design, this means a client can safely retry a request without fear of duplicating actions like charging a credit card twice or creating multiple user accounts.

The beauty of idempotency lies in its practical value for distributed systems. Networks are unreliable. Timeouts happen. Clients crash mid-request. Without idempotency, you're forcing clients to guess whether their request succeeded, leading to duplicate orders, inconsistent state, and 3 AM pages about phantom database records. When you design idempotent endpoints, you're giving clients a superpower: the ability to retry freely without consequences.

Here's the nuance most developers miss: not all HTTP methods need the same treatment. GET, PUT, and DELETE are naturally idempotent by design, but POST operations (creating resources, processing payments) require explicit idempotency handling through techniques like idempotency keys. This is where intentional design separates great APIs from mediocre ones.

## 💡 Key Concepts

- **Idempotent operations produce identical results on repeated calls** — calling `DELETE /users/123` ten times has the same effect as calling it once (the user is deleted or returns 404)

- **Use idempotency keys for non-idempotent operations** — clients generate unique request identifiers that the server stores temporarily to detect and handle duplicate requests

- **Safe != Idempotent** — safe operations (like GET) don't modify state, while idempotent operations may modify state but produce the same result when repeated

- **Time-bound your idempotency windows** — store idempotency keys for a reasonable duration (24-72 hours), then expire them to prevent unbounded storage growth

- **Return cached responses for duplicate requests** — when detecting a duplicate via idempotency key, return the original response instead of processing again

## 🐍 Python Example

```python
from flask import Flask, request, jsonify
from datetime import datetime, timedelta
import uuid

app = Flask(__name__)

# In-memory store (use Redis in production)
idempotency_store = {}
payments_db = {}

@app.route('/api/payments', methods=['POST'])
def create_payment():
    # Extract idempotency key from header
    idempotency_key = request.headers.get('Idempotency-Key')
    
    if not idempotency_key:
        return jsonify({'error': 'Idempotency-Key header required'}), 400
    
    # Check if we've seen this request before
    if idempotency_key in idempotency_store:
        cached = idempotency_store[idempotency_key]
        
        # Verify request hasn't expired (24 hour window)
        if datetime.now() < cached['expires_at']:
            return jsonify(cached['response']), cached['status_code']
        else:
            # Key expired, remove it and process as new
            del idempotency_store[idempotency_key]
    
    # Process the payment (simulate)
    data = request.json
    payment_id = str(uuid.uuid4())
    payment_record = {
        'id': payment_id,
        'amount': data['amount'],
        'currency': data['currency'],
        'status': 'completed'
    }
    
    payments_db[payment_id] = payment_record
    
    # Cache the response with expiration
    response_data = {'payment': payment_record}
    idempotency_store[idempotency_key] = {
        'response': response_data,
        'status_code': 201,
        'expires_at': datetime.now() + timedelta(hours=24)
    }
    
    return jsonify(response_data), 201

if __name__ == '__main__':
    app.run(debug=True)
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const { v4: uuidv4 } = require('uuid');

const app = express();
app.use(express.json());

// In-memory stores (use Redis in production)
const idempotencyStore = new Map();
const paymentsDB = new Map();

// Cleanup expired keys every hour
setInterval(() => {
  const now = Date.now();
  for (const [key, value] of idempotencyStore.entries()) {
    if (now > value.expiresAt) {
      idempotencyStore.delete(key);
    }
  }
}, 3600000);

app.post('/api/payments', (req, res) => {
  const idempotencyKey = req.headers['idempotency-key'];
  
  if (!idempotencyKey) {
    return res.status(400).json({ 
      error: 'Idempotency-Key header required' 
    });
  }
  
  // Return cached response for duplicate requests
  if (idempotencyStore.has(idempotencyKey)) {
    const cached = idempotencyStore.get(idempotencyKey);
    
    if (Date.now() < cached.expiresAt) {
      return res.status(cached.statusCode).json(cached.response);
    }
    
    idempotencyStore.delete(idempotencyKey);
  }
  
  // Process new payment
  const paymentId = uuidv4();
  const payment = {
    id: paymentId,
    amount: req.body.amount,
    currency: req.body.currency,
    status: 'completed',
    createdAt: new Date().toISOString()
  };
  
  paymentsDB.set(paymentId, payment);
  
  // Cache response for 24 hours
  const responseData = { payment };
  idempotencyStore.set(idempotencyKey, {
    response: responseData,
    statusCode: 201,
    expiresAt: Date.now() + 24 * 60 * 60 * 1000
  });
  
  res.status(201).json(responseData);
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

## ⚖️ When To Use / When To Avoid

**✅ When To Use:**
- POST endpoints that create resources (orders, payments, user accounts)
- Any operation with side effects that shouldn't be duplicated (sending emails, charging cards)
- APIs consumed by mobile clients with unreliable networks
- Webhook receivers where duplicate delivery is common

**❌ When To Avoid:**
- Pure GET requests (already naturally idempotent)
- Internal APIs between services you fully control with reliable networks
- Operations where duplicate execution is actually desired (like logging events)
- High-throughput endpoints where the storage overhead outweighs the benefits

## 📚 Further Reading

- [Stripe API: Idempotent Requests](https://stripe.com/docs/api/idempotent_requests) — Real-world implementation from a payments API that gets it right
- [RFC 7231: HTTP Semantics (Section 4.2.2)](https://datatracker.ietf.org/doc/html/rfc7231#section-4.2.2) — Official specification on idempotent methods
- [AWS Architecture Blog: Idempotency Patterns](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/) — Distributed systems perspective on safe retries
- [MDN Web Docs: Idempotent HTTP Methods](https://developer.mozilla.org/en-US/docs/Glossary/Idempotent) — Accessible explanation of HTTP idempotency
- [Martin Fowler: Patterns of Enterprise Application Architecture](https://martinfowler.com/eaaCatalog/) — Context on idempot