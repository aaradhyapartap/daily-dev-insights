# 📌 System design: rate limiter
*September 01, 2026 · Daily Dev Insight*

## 🧠 Overview

Rate limiting is one of those system design components that seems straightforward until you actually need to implement it at scale. At its core, a rate limiter controls how frequently a user, service, or IP address can perform an action within a given timeframe. Think of it as a bouncer for your API—it ensures no single client monopolizes your resources, protects against DDoS attacks, and helps you enforce fair usage policies across your user base.

What makes rate limiting particularly interesting is the trade-off between accuracy and performance. A naive implementation might use a simple counter that resets every minute, but this creates the "double-spending" problem where a user can make 100 requests at 11:59 and another 100 at 12:01, effectively doubling their limit. Production-grade rate limiters use algorithms like token bucket, leaky bucket, or sliding window to provide smoother, more accurate throttling.

The choice of where to implement rate limiting matters enormously. You can do it at the application layer, API gateway, CDN, or even in a dedicated service. Each approach has different latency characteristics, failure modes, and operational complexity. In distributed systems, you'll also need to consider whether you need global rate limiting (harder, requires coordination) or per-node limiting (easier, but less precise).

## 💡 Key Concepts

- **Token Bucket Algorithm**: Tokens are added to a bucket at a fixed rate, and each request consumes a token. When the bucket is empty, requests are rejected. This allows for bursts while maintaining an average rate.

- **Sliding Window vs Fixed Window**: Fixed windows reset at specific intervals (prone to boundary issues), while sliding windows track requests over a rolling time period for more accurate rate limiting.

- **Distributed Rate Limiting**: In multi-server environments, you need shared state (Redis, Memcached) to enforce limits globally, or accept that per-instance limits may allow higher total throughput.

- **Rate Limit Headers**: Always return `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` headers so clients can self-regulate and implement proper backoff strategies.

- **Graceful Degradation**: Consider returning 429 status codes with `Retry-After` headers instead of hard failures, and think about tiered limits for different user classes (free vs paid).

## 🐍 Python Example

```python
import time
from collections import defaultdict
from typing import Dict, Tuple

class SlidingWindowRateLimiter:
    """Token bucket rate limiter with sliding window implementation"""
    
    def __init__(self, max_requests: int, window_seconds: int):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        # Store timestamps of requests per identifier
        self.requests: Dict[str, list] = defaultdict(list)
    
    def is_allowed(self, identifier: str) -> Tuple[bool, dict]:
        """
        Check if request is allowed and return rate limit info
        Returns: (is_allowed, headers_dict)
        """
        current_time = time.time()
        window_start = current_time - self.window_seconds
        
        # Remove timestamps outside the current window
        self.requests[identifier] = [
            ts for ts in self.requests[identifier] 
            if ts > window_start
        ]
        
        request_count = len(self.requests[identifier])
        
        # Prepare rate limit headers
        headers = {
            'X-RateLimit-Limit': self.max_requests,
            'X-RateLimit-Remaining': max(0, self.max_requests - request_count),
            'X-RateLimit-Reset': int(current_time + self.window_seconds)
        }
        
        if request_count < self.max_requests:
            self.requests[identifier].append(current_time)
            return True, headers
        
        # Calculate retry-after from oldest request in window
        retry_after = int(self.requests[identifier][0] - window_start)
        headers['Retry-After'] = retry_after
        return False, headers

# Usage example
limiter = SlidingWindowRateLimiter(max_requests=5, window_seconds=60)

# Simulate API endpoint
def api_call(user_id: str):
    allowed, headers = limiter.is_allowed(user_id)
    if allowed:
        print(f"✓ Request allowed. Remaining: {headers['X-RateLimit-Remaining']}")
    else:
        print(f"✗ Rate limited. Retry after {headers['Retry-After']}s")

# Test it
for i in range(7):
    api_call("user_123")
```

## 🟨 JavaScript Example

```javascript
class TokenBucketRateLimiter {
  /**
   * Token bucket implementation for rate limiting
   * Allows bursts while maintaining average rate
   */
  constructor(maxTokens, refillRate) {
    this.maxTokens = maxTokens;        // Max tokens in bucket
    this.refillRate = refillRate;      // Tokens added per second
    this.buckets = new Map();          // User buckets
  }

  /**
   * Check if request is allowed for given identifier
   * @returns {Object} { allowed: boolean, headers: Object }
   */
  isAllowed(identifier) {
    const now = Date.now() / 1000; // Convert to seconds
    
    // Initialize bucket if it doesn't exist
    if (!this.buckets.has(identifier)) {
      this.buckets.set(identifier, {
        tokens: this.maxTokens,
        lastRefill: now
      });
    }

    const bucket = this.buckets.get(identifier);
    
    // Refill tokens based on time elapsed
    const timePassed = now - bucket.lastRefill;
    const tokensToAdd = timePassed * this.refillRate;
    bucket.tokens = Math.min(
      this.maxTokens, 
      bucket.tokens + tokensToAdd
    );
    bucket.lastRefill = now;

    const headers = {
      'X-RateLimit-Limit': this.maxTokens,
      'X-RateLimit-Remaining': Math.floor(bucket.tokens),
      'X-RateLimit-Reset': Math.ceil(now + (this.maxTokens / this.refillRate))
    };

    // Consume a token if available
    if (bucket.tokens >= 1) {
      bucket.tokens -= 1;
      return { allowed: true, headers };
    }

    // Calculate retry-after
    const tokensNeeded = 1 - bucket.tokens;
    headers['Retry-After'] = Math.ceil(tokensNeeded / this.refillRate);
    return { allowed: false, headers };
  }
}

// Express.js middleware example
const limiter = new TokenBucketRateLimiter(10, 2); // 10 tokens, 2/sec refill

function rateLimitMiddleware(req, res, next) {
  const clientId = req.ip || req.headers['x-forwarded-for'];
  const { allowed, headers } = limiter.isAllowed(clientId);
  
  // Set rate limit headers
  Object.entries(headers).forEach(([key, value]) => {
    res.setHeader(key, value);
  });
  
  if (!allowed) {
    return res.status(429).json({ 
      error: 'Too many requests',
      retryAfter: headers['Retry-After']
    });
  }
  
  next();
}
```

## ⚖️ When To Use / When To Avoid

**Use rate limiting when:**
- ✅ Protecting public APIs from abuse or DDoS attacks
- ✅ Enforcing tiered pricing models (free vs premium users)
- ✅ Preventing resource exhaustion from runaway clients or bugs
- ✅ Ensuring fair usage across multi-tenant systems
- ✅ Complying with downstream API limits (e.g., third-party services)

**Avoid or reconsider when:**
- ❌ You have