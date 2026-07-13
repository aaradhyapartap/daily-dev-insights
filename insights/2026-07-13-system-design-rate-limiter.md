# 📌 System design: rate limiter
*July 13, 2026 · Daily Dev Insight*

## 🧠 Overview

Rate limiters are the unsung heroes of API stability and security. At their core, they control the frequency of operations—whether that's API requests, login attempts, or database queries—preventing resource exhaustion and abuse. But here's what makes them fascinating from a system design perspective: they're not just about saying "no" to requests. They're about graceful degradation, fair resource allocation, and maintaining quality of service under pressure.

The challenge isn't just implementing a counter; it's choosing the right algorithm for your use case. A fixed window counter is simple but can allow traffic bursts at window boundaries. A sliding window log gives precision but requires storing timestamps. Token bucket allows bursts while maintaining average rates. Each approach trades off memory, accuracy, and complexity differently. Understanding these tradeoffs is what separates a junior engineer slapping on middleware from a senior engineer architecting for scale.

In production systems, rate limiters often work in layers: per-user limits at the application level, per-IP limits at the edge, and per-service limits for backend protection. They're your first line of defense against DDoS attacks, abusive users, and—perhaps most critically—your own bugs that might cause request loops.

## 💡 Key Concepts

- **Algorithm selection matters**: Token bucket allows controlled bursts and is ideal for APIs. Fixed window is simpler but less fair. Sliding window log is precise but memory-intensive. Leaky bucket smooths traffic perfectly but may frustrate legitimate users during spikes.

- **Distributed rate limiting requires shared state**: In multi-server environments, you need Redis or similar to coordinate limits. Watch out for race conditions—use atomic operations like INCR or Lua scripts for accuracy.

- **Return meaningful feedback**: Always include headers like `X-RateLimit-Remaining`, `X-RateLimit-Reset`, and `Retry-After`. Good rate limiters communicate; great ones teach clients to behave better.

- **Consider hierarchical limits**: Implement both burst limits (e.g., 10 req/sec) and longer-term quotas (e.g., 10,000 req/day). This prevents both spike abuse and sustained overuse.

- **Rate limiting is a product decision**: Your limits directly impact user experience. Too strict and you frustrate power users; too lenient and you risk outages. Monitor actual usage patterns before setting thresholds.

## 🐍 Python Example

```python
import time
from collections import deque
from threading import Lock

class SlidingWindowRateLimiter:
    """Thread-safe sliding window log rate limiter"""
    
    def __init__(self, max_requests: int, window_seconds: int):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.request_log = {}  # user_id -> deque of timestamps
        self.lock = Lock()
    
    def allow_request(self, user_id: str) -> tuple[bool, dict]:
        """
        Check if request should be allowed.
        Returns (allowed, metadata) tuple.
        """
        with self.lock:
            now = time.time()
            
            # Initialize log for new user
            if user_id not in self.request_log:
                self.request_log[user_id] = deque()
            
            log = self.request_log[user_id]
            
            # Remove timestamps outside current window
            cutoff = now - self.window_seconds
            while log and log[0] < cutoff:
                log.popleft()
            
            # Check if under limit
            remaining = self.max_requests - len(log)
            
            if len(log) < self.max_requests:
                log.append(now)
                return True, {
                    'remaining': remaining - 1,
                    'reset': int(now + self.window_seconds)
                }
            
            # Rate limited
            oldest = log[0]
            retry_after = int(oldest + self.window_seconds - now)
            return False, {
                'remaining': 0,
                'reset': int(oldest + self.window_seconds),
                'retry_after': retry_after
            }

# Usage example
limiter = SlidingWindowRateLimiter(max_requests=5, window_seconds=60)

for i in range(7):
    allowed, meta = limiter.allow_request("user_123")
    print(f"Request {i+1}: {'✓ Allowed' if allowed else '✗ Denied'} - {meta}")
    time.sleep(0.1)
```

## 🟨 JavaScript Example

```javascript
class TokenBucketRateLimiter {
  /**
   * Token bucket implementation - allows bursts with average rate control
   */
  constructor(capacity, refillRate) {
    this.capacity = capacity;          // Max tokens (burst size)
    this.tokens = capacity;            // Current tokens
    this.refillRate = refillRate;      // Tokens per second
    this.lastRefill = Date.now();
    this.users = new Map();            // user_id -> bucket state
  }

  /**
   * Refill tokens based on elapsed time
   */
  refillBucket(bucket) {
    const now = Date.now();
    const elapsed = (now - bucket.lastRefill) / 1000;
    const tokensToAdd = elapsed * this.refillRate;
    
    bucket.tokens = Math.min(
      this.capacity,
      bucket.tokens + tokensToAdd
    );
    bucket.lastRefill = now;
  }

  /**
   * Attempt to consume a token for the request
   */
  tryConsume(userId, tokens = 1) {
    // Initialize bucket for new user
    if (!this.users.has(userId)) {
      this.users.set(userId, {
        tokens: this.capacity,
        lastRefill: Date.now()
      });
    }

    const bucket = this.users.get(userId);
    this.refillBucket(bucket);

    // Check if enough tokens available
    if (bucket.tokens >= tokens) {
      bucket.tokens -= tokens;
      return {
        allowed: true,
        remaining: Math.floor(bucket.tokens),
        retryAfter: null
      };
    }

    // Calculate retry time
    const tokensNeeded = tokens - bucket.tokens;
    const retryAfter = Math.ceil(tokensNeeded / this.refillRate);

    return {
      allowed: false,
      remaining: 0,
      retryAfter
    };
  }
}

// Usage example
const limiter = new TokenBucketRateLimiter(10, 2); // 10 burst, 2/sec refill

for (let i = 0; i < 12; i++) {
  const result = limiter.tryConsume('user_456');
  console.log(`Request ${i+1}:`, result.allowed ? '✓' : '✗', result);
}
```

## ⚖️ When To Use / When To Avoid

**Use rate limiting when:**
- ✅ Exposing public APIs that could be abused or overloaded
- ✅ Protecting expensive operations (AI inference, video processing, etc.)
- ✅ Enforcing tiered pricing models or quota systems
- ✅ Preventing credential stuffing and brute force attacks
- ✅ Ensuring fair resource allocation among users

**Avoid or reconsider when:**
- ❌ Your service is entirely internal with trusted clients (adds unnecessary complexity)
- ❌ You're rate limiting instead of fixing actual performance problems
- ❌ Your limits are so high they'd never trigger (just monitoring overhead)
- ❌ You can't provide users a way to request limit increases (creates support burden)

## 📚 Further Reading

- [Stripe's rate limiter design](https://stripe.com/blog/rate-limiters) - Real-world implementation with Redis and Go
- [IETF Draft: RateLimit Header Fields](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/) - Standardized header specifications
- [Redis INCR command documentation](https://redis.io