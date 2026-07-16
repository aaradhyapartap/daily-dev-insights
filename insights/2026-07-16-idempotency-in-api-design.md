# 📌 Idempotency in API design
*July 16, 2026 · Daily Dev Insight*

## 🧠 Overview

Idempotency is one of those concepts that sounds academic until you're debugging why a customer got charged three times for the same order. At its core, an idempotent operation can be performed multiple times without changing the result beyond the initial application. Think of it like a light switch at the "on" position—flipping it to "on" repeatedly doesn't make the room "more on."

In API design, idempotency is your safety net against network failures, impatient users clicking "Submit" repeatedly, and retry logic gone wild. When you design an endpoint to be idempotent, you're making a promise: "Call me once or call me a hundred times with the same parameters, and you'll get the same outcome." This isn't just about returning the same response—it's about ensuring the underlying state changes only happen once, even if the request is received multiple times.

The difference between a well-designed API and a production incident at 3 AM often comes down to how you handle these duplicate requests. HTTP GET, PUT, and DELETE are naturally idempotent by design, but POST? That's where things get interesting, and where most developers need to add explicit idempotency handling.

## 💡 Key Concepts

- **Idempotency Keys**: Client-generated unique identifiers (usually UUIDs) sent with requests to identify duplicate operations. The server stores these keys temporarily to detect and handle retries.

- **Safe vs Idempotent**: All safe operations (those that don't modify state) are idempotent, but not all idempotent operations are safe. PUT is idempotent but not safe; GET is both.

- **State Fingerprinting**: For operations where the same action with the same data should only execute once, store a hash or fingerprint of the request along with the result to serve cached responses on retries.

- **Time Windows**: Idempotency keys shouldn't live forever. Implement expiration (typically 24 hours) to prevent infinite storage growth while still catching legitimate retries.

- **Status Codes Matter**: Return `409 Conflict` for duplicate requests still processing, `200 OK` with the original result for completed duplicates, and proper error codes if the idempotency key was used with different parameters.

## 🐍 Python Example

```python
from flask import Flask, request, jsonify
from datetime import datetime, timedelta
import hashlib
import json

app = Flask(__name__)

# In production, use Redis or a database
idempotency_store = {}

def generate_fingerprint(data):
    """Create a deterministic hash of the request data"""
    serialized = json.dumps(data, sort_keys=True)
    return hashlib.sha256(serialized.encode()).hexdigest()

@app.route('/api/payments', methods=['POST'])
def create_payment():
    # Extract idempotency key from headers
    idempotency_key = request.headers.get('Idempotency-Key')
    
    if not idempotency_key:
        return jsonify({'error': 'Idempotency-Key header required'}), 400
    
    request_data = request.get_json()
    fingerprint = generate_fingerprint(request_data)
    
    # Check if we've seen this request before
    if idempotency_key in idempotency_store:
        stored = idempotency_store[idempotency_key]
        
        # Verify the request data matches
        if stored['fingerprint'] != fingerprint:
            return jsonify({
                'error': 'Idempotency key reused with different parameters'
            }), 422
        
        # Return the cached response
        return jsonify(stored['response']), stored['status_code']
    
    # Process the payment (simulate)
    try:
        payment_result = {
            'id': f'pay_{idempotency_key[:8]}',
            'amount': request_data['amount'],
            'status': 'completed',
            'timestamp': datetime.now().isoformat()
        }
        
        # Store the result with expiration metadata
        idempotency_store[idempotency_key] = {
            'fingerprint': fingerprint,
            'response': payment_result,
            'status_code': 201,
            'expires_at': datetime.now() + timedelta(hours=24)
        }
        
        return jsonify(payment_result), 201
        
    except Exception as e:
        return jsonify({'error': str(e)}), 500
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const crypto = require('crypto');

const app = express();
app.use(express.json());

// In production, use Redis with TTL
const idempotencyCache = new Map();

function generateFingerprint(data) {
  const serialized = JSON.stringify(data, Object.keys(data).sort());
  return crypto.createHash('sha256').update(serialized).digest('hex');
}

function cleanupExpiredKeys() {
  const now = Date.now();
  for (const [key, value] of idempotencyCache.entries()) {
    if (value.expiresAt < now) {
      idempotencyCache.delete(key);
    }
  }
}

// Cleanup every hour
setInterval(cleanupExpiredKeys, 3600000);

app.post('/api/orders', async (req, res) => {
  const idempotencyKey = req.headers['idempotency-key'];
  
  if (!idempotencyKey) {
    return res.status(400).json({ 
      error: 'Idempotency-Key header is required' 
    });
  }
  
  const fingerprint = generateFingerprint(req.body);
  const cached = idempotencyCache.get(idempotencyKey);
  
  if (cached) {
    // Check if request data matches
    if (cached.fingerprint !== fingerprint) {
      return res.status(422).json({
        error: 'Idempotency key used with different request data'
      });
    }
    
    // Return cached response
    return res.status(cached.statusCode).json(cached.response);
  }
  
  try {
    // Simulate order processing
    const order = {
      id: `ord_${idempotencyKey.substring(0, 8)}`,
      items: req.body.items,
      total: req.body.items.reduce((sum, item) => sum + item.price, 0),
      status: 'confirmed',
      createdAt: new Date().toISOString()
    };
    
    // Cache the successful response for 24 hours
    idempotencyCache.set(idempotencyKey, {
      fingerprint,
      response: order,
      statusCode: 201,
      expiresAt: Date.now() + (24 * 60 * 60 * 1000)
    });
    
    res.status(201).json(order);
    
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.listen(3000);
```

## ⚖️ When To Use / When To Avoid

**✅ When To Use:**
- Payment processing and any financial transactions
- Creating resources where duplicates would cause problems (user registrations, order placements)
- Any POST operation that could reasonably be retried due to network issues
- Webhook endpoints that might receive duplicate deliveries
- Operations with external side effects (sending emails, SMS, push notifications)

**❌ When To Avoid:**
- Pure read operations (GET requests handle this naturally)
- Operations where duplicates are intentionally meaningful (adding items to a log, recording events)
- High-frequency operations where key storage overhead outweighs retry risk
- Internal APIs where retry logic is fully controlled and guaranteed to be correct

## 📚 Further Reading

- [Stripe API Idempotency Documentation](https://stripe.com/docs/api/idempotent_requests) - Industry gold standard for payment API idempotency
- [RFC 9110 - HTTP Semantics (Section 9.2