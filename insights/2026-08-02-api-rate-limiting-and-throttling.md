# 📌 API rate limiting and throttling
*August 02, 2026 · Daily Dev Insight*

## 🧠 Overview

Rate limiting and throttling are your API's immune system—they protect your infrastructure from being overwhelmed while ensuring fair resource distribution among clients. While often used interchangeably, they serve slightly different purposes: **rate limiting** enforces hard caps on requests over a time window (e.g., 100 requests per minute), while **throttling** can dynamically slow down or queue requests when load spikes occur. Both are essential for production APIs, whether you're protecting a small service or scaling to millions of users.

The challenge isn't just implementing these mechanisms—it's choosing the right strategy. A naive implementation might block legitimate traffic during flash sales or organic growth spikes. Smart rate limiting considers client identity, endpoint sensitivity, and adaptive thresholds. For example, authenticated users might get higher limits than anonymous ones, and read operations could be more permissive than writes.

Understanding rate limiting isn't just about protecting your own APIs; it's crucial when consuming third-party services too. Every major API (GitHub, Stripe, OpenAI) enforces limits, and your code needs to gracefully handle 429 responses with exponential backoff. The difference between a resilient system and a brittle one often comes down to how well you handle rate limits on both sides of the API boundary.

## 💡 Key Concepts

- **Token Bucket Algorithm**: The most flexible approach—tokens regenerate at a fixed rate, and each request consumes tokens. Allows brief bursts while maintaining average rate control.

- **Sliding Window Logs**: Tracks timestamps of recent requests to enforce limits precisely. More accurate than fixed windows but requires more memory to store request history.

- **Distributed Rate Limiting**: In multi-server environments, use Redis or similar to share state across instances. Race conditions and synchronization become your main challenges.

- **Exponential Backoff**: When you hit a rate limit as a client, don't retry immediately. Wait progressively longer between attempts (1s, 2s, 4s, 8s...) to avoid thundering herd problems.

- **Rate Limit Headers**: Always communicate limits to clients via headers like `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `Retry-After` so they can self-regulate.

## 🐍 Python Example

```python
import time
import redis
from functools import wraps
from flask import Flask, request, jsonify

app = Flask(__name__)
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def rate_limit(max_requests=10, window_seconds=60):
    """
    Token bucket rate limiter using Redis for distributed environments.
    Allows burst traffic while maintaining average rate.
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Use client IP as identifier (use auth token in production)
            client_id = request.remote_addr
            key = f"rate_limit:{client_id}:{func.__name__}"
            
            # Redis pipeline for atomic operations
            pipe = redis_client.pipeline()
            now = time.time()
            
            # Remove requests older than the window
            pipe.zremrangebyscore(key, 0, now - window_seconds)
            # Count requests in current window
            pipe.zcard(key)
            # Add current request timestamp
            pipe.zadd(key, {str(now): now})
            # Set expiry to clean up old keys
            pipe.expire(key, window_seconds)
            
            results = pipe.execute()
            request_count = results[1]
            
            # Calculate rate limit headers
            remaining = max(0, max_requests - request_count - 1)
            reset_time = int(now + window_seconds)
            
            if request_count >= max_requests:
                response = jsonify({
                    "error": "Rate limit exceeded",
                    "retry_after": window_seconds
                })
                response.status_code = 429
                response.headers['X-RateLimit-Limit'] = str(max_requests)
                response.headers['X-RateLimit-Remaining'] = '0'
                response.headers['X-RateLimit-Reset'] = str(reset_time)
                response.headers['Retry-After'] = str(window_seconds)
                return response
            
            # Execute the actual endpoint
            result = func(*args, **kwargs)
            
            # Add rate limit headers to successful response
            if hasattr(result, 'headers'):
                result.headers['X-RateLimit-Limit'] = str(max_requests)
                result.headers['X-RateLimit-Remaining'] = str(remaining)
                result.headers['X-RateLimit-Reset'] = str(reset_time)
            
            return result
        return wrapper
    return decorator

@app.route('/api/data')
@rate_limit(max_requests=5, window_seconds=60)
def get_data():
    return jsonify({"message": "Success", "data": [1, 2, 3]})
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const Redis = require('ioredis');

const app = express();
const redis = new Redis();

/**
 * Sliding window rate limiter with exponential backoff client
 * Demonstrates both server and client-side rate limiting patterns
 */
class RateLimiter {
  constructor(maxRequests = 10, windowMs = 60000) {
    this.maxRequests = maxRequests;
    this.windowMs = windowMs;
  }

  async checkLimit(identifier) {
    const key = `ratelimit:${identifier}`;
    const now = Date.now();
    const windowStart = now - this.windowMs;

    // Use Redis transaction for atomic operations
    const multi = redis.multi();
    multi.zremrangebyscore(key, 0, windowStart);
    multi.zadd(key, now, `${now}`);
    multi.zcard(key);
    multi.pexpire(key, this.windowMs);
    
    const results = await multi.exec();
    const requestCount = results[2][1];

    const allowed = requestCount <= this.maxRequests;
    const remaining = Math.max(0, this.maxRequests - requestCount);
    
    return {
      allowed,
      remaining,
      resetTime: new Date(now + this.windowMs).toISOString()
    };
  }
}

// Middleware implementation
const limiter = new RateLimiter(100, 60000);

const rateLimitMiddleware = async (req, res, next) => {
  const identifier = req.ip || req.connection.remoteAddress;
  const limit = await limiter.checkLimit(identifier);

  res.setHeader('X-RateLimit-Limit', limiter.maxRequests);
  res.setHeader('X-RateLimit-Remaining', limit.remaining);
  res.setHeader('X-RateLimit-Reset', limit.resetTime);

  if (!limit.allowed) {
    return res.status(429).json({
      error: 'Too many requests',
      retryAfter: Math.ceil(limiter.windowMs / 1000)
    });
  }

  next();
};

// Client-side exponential backoff helper
async function fetchWithRetry(url, options = {}, maxRetries = 5) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const response = await fetch(url, options);
    
    if (response.status !== 429) {
      return response;
    }

    // Exponential backoff: 1s, 2s, 4s, 8s, 16s
    const backoffMs = Math.pow(2, attempt) * 1000;
    const retryAfter = response.headers.get('Retry-After');
    const waitTime = retryAfter ? parseInt(retryAfter) * 1000 : backoffMs;
    
    console.log(`Rate limited. Retrying in ${waitTime}ms...`);
    await new Promise(resolve => setTimeout(resolve, waitTime));
  }
  
  throw new Error