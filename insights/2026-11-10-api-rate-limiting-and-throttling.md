# 📌 API rate limiting and throttling
*November 10, 2026 · Daily Dev Insight*

## 🧠 Overview

Rate limiting and throttling are your API's immune system—they protect your backend from being overwhelmed while ensuring fair resource distribution among clients. Think of them as bouncers at a club: rate limiting sets a hard cap on how many requests a client can make in a time window (e.g., 100 requests per minute), while throttling gradually slows down clients as they approach their limits, like easing the flow of traffic instead of slamming on the brakes.

The distinction matters in practice. Rate limiting typically results in hard rejections (HTTP 429), which is clean but potentially jarring for users. Throttling introduces deliberate delays, spreading the load more smoothly but potentially masking problems in client code. Most production systems use a hybrid approach: throttle near the limit, reject at the ceiling.

Why should you care? Beyond preventing DDoS attacks and resource exhaustion, proper rate limiting is essential for cost control (especially with pay-per-request cloud services), maintaining SLA commitments to paying customers, and preventing noisy neighbors in multi-tenant architectures. It's not just defensive—it's good product design that forces you to think about usage tiers and pricing from day one.

## 💡 Key Concepts

- **Token Bucket Algorithm**: The gold standard for rate limiting. Tokens regenerate at a fixed rate, and each request consumes a token. When the bucket is empty, requests are denied or delayed. Allows for brief bursts while maintaining average rate limits.

- **Sliding Window vs Fixed Window**: Fixed windows (e.g., 100 requests per minute starting at :00) can allow 200 requests in quick succession at window boundaries. Sliding windows track requests in a rolling timeframe, providing smoother enforcement but requiring more memory.

- **Distributed Rate Limiting**: In multi-server environments, rate limits must be coordinated across instances using shared state (Redis is the typical choice). Without this, a 100 req/min limit becomes 100 req/min per server—not what you want.

- **Backoff Strategies**: Clients should implement exponential backoff when hitting rate limits. Your API should communicate limits via headers (`X-RateLimit-Remaining`, `Retry-After`) so smart clients can self-regulate.

- **Cost-Based Rate Limiting**: Not all endpoints are equal. Consider weighting expensive operations (database writes, AI model calls) more heavily than cheap reads. A single cost budget is more flexible than per-endpoint limits.

## 🐍 Python Example

```python
from datetime import datetime, timedelta
from collections import defaultdict
import time

class TokenBucketRateLimiter:
    """
    Thread-safe token bucket rate limiter implementation.
    Ideal for protecting API endpoints in Flask/FastAPI apps.
    """
    def __init__(self, rate: int, per_seconds: int, burst: int = None):
        self.rate = rate  # tokens per time period
        self.per_seconds = per_seconds
        self.burst = burst or rate  # max bucket size
        self.buckets = defaultdict(lambda: {
            'tokens': self.burst,
            'last_update': datetime.now()
        })
    
    def allow_request(self, client_id: str, cost: int = 1) -> bool:
        """
        Check if request is allowed and consume tokens if so.
        Returns True if allowed, False if rate limited.
        """
        bucket = self.buckets[client_id]
        now = datetime.now()
        
        # Refill tokens based on time elapsed
        elapsed = (now - bucket['last_update']).total_seconds()
        refill = elapsed * (self.rate / self.per_seconds)
        bucket['tokens'] = min(self.burst, bucket['tokens'] + refill)
        bucket['last_update'] = now
        
        # Check if enough tokens available
        if bucket['tokens'] >= cost:
            bucket['tokens'] -= cost
            return True
        return False
    
    def get_retry_after(self, client_id: str, cost: int = 1) -> float:
        """Calculate seconds until enough tokens available."""
        bucket = self.buckets[client_id]
        needed = cost - bucket['tokens']
        return max(0, needed / (self.rate / self.per_seconds))

# Usage example
limiter = TokenBucketRateLimiter(rate=10, per_seconds=60)  # 10 req/min

for i in range(15):
    allowed = limiter.allow_request("user_123")
    print(f"Request {i+1}: {'✓ Allowed' if allowed else '✗ Rate limited'}")
    time.sleep(0.5)
```

## 🟨 JavaScript Example

```javascript
/**
 * Redis-backed distributed rate limiter using sliding window
 * Perfect for Node.js microservices with multiple instances
 */
const redis = require('redis');

class DistributedRateLimiter {
  constructor(redisClient, options = {}) {
    this.redis = redisClient;
    this.windowMs = options.windowMs || 60000; // 1 minute default
    this.maxRequests = options.maxRequests || 100;
  }

  async checkLimit(clientId) {
    const now = Date.now();
    const windowStart = now - this.windowMs;
    const key = `ratelimit:${clientId}`;

    // Use Redis sorted set with scores as timestamps
    const multi = this.redis.multi();
    
    // Remove old entries outside the sliding window
    multi.zRemRangeByScore(key, 0, windowStart);
    
    // Count requests in current window
    multi.zCard(key);
    
    // Add current request
    multi.zAdd(key, now, `${now}-${Math.random()}`);
    
    // Set expiration to cleanup
    multi.expire(key, Math.ceil(this.windowMs / 1000));
    
    const results = await multi.exec();
    const requestCount = results[1][1]; // zCard result

    const allowed = requestCount < this.maxRequests;
    const remaining = Math.max(0, this.maxRequests - requestCount - 1);
    
    return {
      allowed,
      remaining,
      resetAt: now + this.windowMs,
      retryAfter: allowed ? null : Math.ceil(this.windowMs / 1000)
    };
  }

  // Express middleware factory
  middleware() {
    return async (req, res, next) => {
      const clientId = req.ip || req.headers['x-forwarded-for'];
      const result = await this.checkLimit(clientId);

      // Set standard rate limit headers
      res.set({
        'X-RateLimit-Limit': this.maxRequests,
        'X-RateLimit-Remaining': result.remaining,
        'X-RateLimit-Reset': new Date(result.resetAt).toISOString()
      });

      if (!result.allowed) {
        res.set('Retry-After', result.retryAfter);
        return res.status(429).json({
          error: 'Too many requests',
          retryAfter: result.retryAfter
        });
      }

      next();
    };
  }
}

module.exports = DistributedRateLimiter;
```

## ⚖️ When To Use / When To Avoid

**When to use:**
- ✅ Public APIs or any endpoint exposed to untrusted clients
- ✅ Protecting expensive operations (database writes, external API calls, ML inference)
- ✅ Multi-tenant systems where you need fair resource allocation
- ✅ Implementing usage-based pricing tiers
- ✅ Preventing abuse and scraping

**When to avoid or reconsider:**
- ❌ Internal microservice-to-microservice communication (use circuit breakers instead)
- ❌ When your bottleneck is elsewhere (database, not API gateway)
- ❌ Real-time systems where predictable latency matters more than throughput
- ❌ Over-engineering for APIs with naturally low traffic

## 📚 Further Reading

- [Token Bucket Algorithm Explained](https://en.wikipedia