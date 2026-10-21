# 📌 System design: rate limiter
*October 21, 2026 · Daily Dev Insight*

## 🧠 Overview

Rate limiters are the unsung heroes of API stability and resource protection. At their core, they control how frequently a user, service, or IP address can access a resource within a time window. Think of them as bouncers at an exclusive club—they don't prevent everyone from entering, they just ensure the venue doesn't get dangerously overcrowded.

In production systems, rate limiters serve multiple critical functions: they protect against DDoS attacks, prevent resource exhaustion, enforce API tier limits, and ensure fair usage across clients. A well-designed rate limiter can mean the difference between a service that gracefully handles traffic spikes and one that collapses under load. The tricky part? Implementing one that's both accurate and performant at scale.

The most common algorithms include Token Bucket (refills tokens at a constant rate), Leaky Bucket (processes requests at a fixed rate), Fixed Window (simple but prone to burst issues), Sliding Window Log (accurate but memory-intensive), and Sliding Window Counter (a pragmatic middle ground). Choosing the right one depends on your tolerance for burstiness and your infrastructure constraints.

## 💡 Key Concepts

- **Distributed vs. Local**: Local rate limiters work great for single-instance services, but distributed systems need shared state (Redis, Memcached) to track limits across multiple servers—accepting some eventual consistency in exchange for availability.

- **Token Bucket Algorithm**: The most popular approach that allows for controlled bursts while maintaining an average rate. Tokens refill at a constant rate, and each request consumes a token—simple, effective, and flexible.

- **Response Headers**: Always return rate limit information in headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`) so clients can implement backoff strategies intelligently.

- **Multiple Dimensions**: Production rate limiters often track limits across multiple keys—per user, per IP, per API key, per endpoint—with different thresholds for each dimension.

- **Fail-Open vs. Fail-Closed**: When your rate limiter's backing store is unavailable, decide whether to allow all requests (fail-open, preserving availability) or deny them (fail-closed, preserving protection). There's no universal right answer.

## 🐍 Python Example

```python
import time
from threading import Lock
from collections import defaultdict

class TokenBucketRateLimiter:
    """Thread-safe token bucket rate limiter for API endpoints."""
    
    def __init__(self, rate: int, capacity: int):
        """
        Args:
            rate: Tokens refilled per second
            capacity: Maximum tokens in bucket
        """
        self.rate = rate
        self.capacity = capacity
        self.buckets = defaultdict(lambda: {
            'tokens': capacity,
            'last_update': time.time()
        })
        self.lock = Lock()
    
    def _refill(self, bucket: dict) -> None:
        """Refill tokens based on elapsed time."""
        now = time.time()
        elapsed = now - bucket['last_update']
        # Add tokens based on time passed
        bucket['tokens'] = min(
            self.capacity,
            bucket['tokens'] + elapsed * self.rate
        )
        bucket['last_update'] = now
    
    def allow_request(self, client_id: str) -> tuple[bool, dict]:
        """
        Check if request should be allowed.
        Returns: (is_allowed, metadata)
        """
        with self.lock:
            bucket = self.buckets[client_id]
            self._refill(bucket)
            
            if bucket['tokens'] >= 1:
                bucket['tokens'] -= 1
                return True, {
                    'remaining': int(bucket['tokens']),
                    'limit': self.capacity
                }
            
            return False, {
                'remaining': 0,
                'limit': self.capacity,
                'retry_after': int((1 - bucket['tokens']) / self.rate)
            }

# Usage
limiter = TokenBucketRateLimiter(rate=10, capacity=20)  # 10 req/sec, burst of 20

allowed, metadata = limiter.allow_request('user_123')
print(f"Request allowed: {allowed}, Remaining: {metadata['remaining']}")
```

## 🟨 JavaScript Example

```javascript
class SlidingWindowRateLimiter {
  /**
   * Sliding window counter rate limiter using Redis-like logic
   * @param {number} windowMs - Time window in milliseconds
   * @param {number} maxRequests - Maximum requests per window
   */
  constructor(windowMs, maxRequests) {
    this.windowMs = windowMs;
    this.maxRequests = maxRequests;
    // In production, replace with Redis
    this.requests = new Map();
  }

  /**
   * Check if request is allowed and update counters
   * @param {string} clientId - Unique client identifier
   * @returns {Object} Rate limit decision and metadata
   */
  async allowRequest(clientId) {
    const now = Date.now();
    const windowStart = now - this.windowMs;

    // Get or initialize request log for this client
    if (!this.requests.has(clientId)) {
      this.requests.set(clientId, []);
    }

    const requestLog = this.requests.get(clientId);
    
    // Remove requests outside the current window
    const validRequests = requestLog.filter(
      timestamp => timestamp > windowStart
    );
    
    this.requests.set(clientId, validRequests);

    // Check if under limit
    if (validRequests.length < this.maxRequests) {
      validRequests.push(now);
      return {
        allowed: true,
        remaining: this.maxRequests - validRequests.length,
        resetAt: new Date(validRequests[0] + this.windowMs)
      };
    }

    // Rate limited - calculate retry time
    const oldestRequest = validRequests[0];
    const retryAfter = Math.ceil((oldestRequest + this.windowMs - now) / 1000);

    return {
      allowed: false,
      remaining: 0,
      retryAfter,
      resetAt: new Date(oldestRequest + this.windowMs)
    };
  }
}

// Express.js middleware example
const limiter = new SlidingWindowRateLimiter(60000, 100); // 100 req/min

const rateLimitMiddleware = async (req, res, next) => {
  const clientId = req.ip || req.headers['x-forwarded-for'];
  const result = await limiter.allowRequest(clientId);
  
  res.set('X-RateLimit-Limit', limiter.maxRequests);
  res.set('X-RateLimit-Remaining', result.remaining);
  
  if (!result.allowed) {
    res.set('Retry-After', result.retryAfter);
    return res.status(429).json({ error: 'Too many requests' });
  }
  
  next();
};
```

## ⚖️ When To Use / When To Avoid

**Use Rate Limiters When:**
- Protecting public APIs from abuse or DDoS attacks
- Implementing tiered pricing models (free tier: 100 req/hour, premium: 10K req/hour)
- Preventing resource exhaustion in backend services
- Ensuring fair usage among multiple tenants in a multi-tenant system

**Avoid or Reconsider When:**
- Internal service-to-service communication where trust exists (use circuit breakers instead)
- Latency is absolutely critical and you can't afford the Redis roundtrip
- Your traffic is so low that the complexity isn't justified
- You need perfect accuracy at massive scale (consider admission control or queue-based approaches)

## 📚 Further Reading

- [Redis Rate Limiting Patterns](https://redis.io/docs/reference/patterns/rate-limiting/) - Official Redis documentation on implementing distributed rate limiters
- [NGINX Rate Limiting](https://www.nginx.com/blog/rate-limiting-nginx/) - Production-grade rate limiting at the reverse proxy level