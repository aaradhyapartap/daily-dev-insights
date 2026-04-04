# 📌 System design: rate limiter
*April 04, 2026 · Daily Dev Insight*

## 🧠 Overview

Rate limiting is one of those system design patterns that looks deceptively simple on the surface but reveals surprising complexity when you start building it at scale. At its core, it's about controlling the flow of requests to protect your system from abuse, overload, and cascading failures. Think of it as a bouncer for your API—it decides who gets in, how often, and when to say "slow down there, friend."

The interesting challenge isn't just counting requests; it's doing so efficiently across distributed systems while maintaining fairness and avoiding the thundering herd problem. Modern rate limiters need to handle millions of requests per second, gracefully degrade under pressure, and provide meaningful feedback to clients. The difference between a naive implementation and a production-ready one often lies in the subtle details of timing windows, memory efficiency, and how you handle edge cases like clock skew in distributed environments.

What makes rate limiting particularly fascinating from a systems perspective is that it sits at the intersection of several computer science domains: algorithms (for efficient counting), distributed systems (for coordination), and performance engineering (for minimal latency overhead). Get it right, and your system stays healthy under load. Get it wrong, and you might accidentally DOS yourself.

## 💡 Key Concepts

• **Token Bucket vs Sliding Window**: Token bucket allows bursts up to bucket capacity, while sliding window provides more consistent rate limiting. Choose token bucket for user-facing APIs where bursty behavior is natural, sliding window for backend service protection.

• **Distributed Coordination**: In multi-instance deployments, rate limits need shared state. Redis is the go-to solution, but consider the latency cost—sometimes approximate limits with local counters are better than perfect limits with high overhead.

• **Granularity Matters**: Rate limit by user ID, IP address, API key, or combinations thereof. Too granular and you'll overwhelm your storage; too coarse and legitimate users suffer from bad actors in the same bucket.

• **Graceful Degradation**: When your rate limiter itself becomes a bottleneck, have a fallback strategy. Better to allow some excess traffic than to fail closed and block all requests.

• **Client-Friendly Headers**: Always return rate limit status in HTTP headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`) so clients can implement intelligent backoff strategies.

## 🐍 Python Example

```python
import time
import threading
from collections import defaultdict
from dataclasses import dataclass
from typing import Dict, Optional

@dataclass
class TokenBucket:
    capacity: int
    tokens: float
    refill_rate: float  # tokens per second
    last_refill: float

class RateLimiter:
    def __init__(self, default_capacity: int = 100, default_refill_rate: float = 10):
        self.default_capacity = default_capacity
        self.default_refill_rate = default_refill_rate
        self.buckets: Dict[str, TokenBucket] = {}
        self.lock = threading.RLock()
    
    def _get_bucket(self, key: str) -> TokenBucket:
        """Get or create a token bucket for the given key."""
        now = time.time()
        if key not in self.buckets:
            self.buckets[key] = TokenBucket(
                capacity=self.default_capacity,
                tokens=self.default_capacity,
                refill_rate=self.default_refill_rate,
                last_refill=now
            )
        return self.buckets[key]
    
    def _refill_bucket(self, bucket: TokenBucket) -> None:
        """Refill tokens based on elapsed time."""
        now = time.time()
        elapsed = now - bucket.last_refill
        new_tokens = elapsed * bucket.refill_rate
        bucket.tokens = min(bucket.capacity, bucket.tokens + new_tokens)
        bucket.last_refill = now
    
    def is_allowed(self, key: str, tokens_requested: int = 1) -> tuple[bool, dict]:
        """Check if request is allowed and return status info."""
        with self.lock:
            bucket = self._get_bucket(key)
            self._refill_bucket(bucket)
            
            allowed = bucket.tokens >= tokens_requested
            if allowed:
                bucket.tokens -= tokens_requested
            
            return allowed, {
                'limit': bucket.capacity,
                'remaining': int(bucket.tokens),
                'retry_after': max(0, (tokens_requested - bucket.tokens) / bucket.refill_rate)
            }

# Usage example
limiter = RateLimiter(capacity=5, refill_rate=1)  # 5 requests, refill 1/sec

for i in range(7):
    allowed, info = limiter.is_allowed("user_123")
    print(f"Request {i+1}: {'✓' if allowed else '✗'} (remaining: {info['remaining']})")
    time.sleep(0.5)
```

## 🟨 JavaScript Example

```javascript
class SlidingWindowRateLimiter {
    constructor(windowSize = 60000, maxRequests = 100) {
        this.windowSize = windowSize; // milliseconds
        this.maxRequests = maxRequests;
        this.windows = new Map(); // key -> array of timestamps
    }

    /**
     * Clean up old timestamps outside the current window
     */
    _cleanupWindow(timestamps, now) {
        const cutoff = now - this.windowSize;
        return timestamps.filter(timestamp => timestamp > cutoff);
    }

    /**
     * Check if request is allowed under sliding window algorithm
     */
    isAllowed(key, requestCount = 1) {
        const now = Date.now();
        
        // Get or initialize timestamp array for this key
        if (!this.windows.has(key)) {
            this.windows.set(key, []);
        }
        
        let timestamps = this.windows.get(key);
        
        // Remove expired timestamps
        timestamps = this._cleanupWindow(timestamps, now);
        this.windows.set(key, timestamps);
        
        // Check if adding this request would exceed limit
        const currentCount = timestamps.length;
        if (currentCount + requestCount > this.maxRequests) {
            const oldestTimestamp = timestamps[0] || now;
            const retryAfter = Math.ceil((oldestTimestamp + this.windowSize - now) / 1000);
            
            return {
                allowed: false,
                limit: this.maxRequests,
                remaining: Math.max(0, this.maxRequests - currentCount),
                retryAfter: retryAfter,
                resetTime: new Date(now + retryAfter * 1000)
            };
        }
        
        // Allow request and record timestamps
        for (let i = 0; i < requestCount; i++) {
            timestamps.push(now);
        }
        
        return {
            allowed: true,
            limit: this.maxRequests,
            remaining: this.maxRequests - timestamps.length,
            retryAfter: 0,
            resetTime: new Date(now + this.windowSize)
        };
    }

    /**
     * Clean up unused keys to prevent memory leaks
     */
    cleanup() {
        const now = Date.now();
        for (const [key, timestamps] of this.windows) {
            const cleaned = this._cleanupWindow(timestamps, now);
            if (cleaned.length === 0) {
                this.windows.delete(key);
            } else {
                this.windows.set(key, cleaned);
            }
        }
    }
}

// Express.js middleware example
function createRateLimitMiddleware(limiter) {
    return (req, res, next) => {
        const key = req.ip || req.connection.remoteAddress;
        const result = limiter.isAllowed(key);
        
        // Set rate limit headers
        res.set({
            'X-RateLimit-Limit': result.limit,
            'X-RateLimit-Remaining': result.remaining,
            'X-RateLimit-Reset':