# 📌 System design: rate limiter
*May 24, 2026 · Daily Dev Insight*

## 🧠 Overview

Rate limiters are the unsung heroes of distributed systems, quietly preventing your API from becoming the digital equivalent of a Black Friday stampede. At their core, they're traffic cops for your endpoints, ensuring that no single user or service can overwhelm your system with requests. But here's the thing most engineers miss: rate limiting isn't just about preventing abuse—it's about ensuring fair resource allocation and maintaining quality of service for all users.

The elegance of rate limiting lies in its simplicity, yet the devil is in the implementation details. Should you use a token bucket, sliding window, or fixed window approach? Where do you store the state—in memory, Redis, or a database? These decisions directly impact your system's scalability, accuracy, and performance. A well-designed rate limiter can be the difference between a system that gracefully handles traffic spikes and one that crumbles under pressure.

## 💡 Key Concepts

• **Algorithm Choice Matters**: Token bucket allows burst traffic, sliding window provides smooth limiting, and fixed window is simple but can cause thundering herd issues at window boundaries

• **State Storage Strategy**: In-memory is fastest but doesn't scale across instances, Redis provides shared state with good performance, while databases offer persistence at the cost of latency

• **Distributed Challenges**: Race conditions become real problems at scale—you need atomic operations or accept eventual consistency trade-offs

• **Graceful Degradation**: When your rate limiter itself fails, fail open (allow requests) rather than fail closed (block everything) to maintain system availability

• **Multiple Dimensions**: Consider rate limiting by user ID, IP address, API key, or even request complexity—one size doesn't fit all scenarios

## 🐍 Python Example

```python
import time
import threading
from collections import defaultdict, deque
from typing import Dict, Tuple

class SlidingWindowRateLimiter:
    def __init__(self, max_requests: int, window_seconds: int):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        # Store request timestamps for each key
        self.requests: Dict[str, deque] = defaultdict(deque)
        self.lock = threading.RLock()
    
    def is_allowed(self, key: str) -> Tuple[bool, dict]:
        """
        Check if request is allowed and return status with metadata
        """
        with self.lock:
            now = time.time()
            key_requests = self.requests[key]
            
            # Remove expired requests from sliding window
            cutoff_time = now - self.window_seconds
            while key_requests and key_requests[0] <= cutoff_time:
                key_requests.popleft()
            
            # Check if we can allow this request
            if len(key_requests) < self.max_requests:
                key_requests.append(now)
                allowed = True
            else:
                allowed = False
            
            # Calculate when the next request would be allowed
            if key_requests:
                retry_after = key_requests[0] + self.window_seconds - now
            else:
                retry_after = 0
                
            return allowed, {
                'requests_remaining': max(0, self.max_requests - len(key_requests)),
                'retry_after_seconds': max(0, retry_after),
                'reset_time': now + retry_after
            }

# Usage example
limiter = SlidingWindowRateLimiter(max_requests=10, window_seconds=60)

# Simulate API endpoint
def api_call(user_id: str):
    allowed, metadata = limiter.is_allowed(user_id)
    
    if allowed:
        print(f"✅ Request allowed for {user_id}. "
              f"Remaining: {metadata['requests_remaining']}")
        return {"status": "success", "data": "your_api_response"}
    else:
        print(f"❌ Rate limited! Retry after {metadata['retry_after_seconds']:.1f}s")
        return {"status": "rate_limited", "retry_after": metadata['retry_after_seconds']}
```

## 🟨 JavaScript Example

```javascript
class TokenBucketRateLimiter {
    constructor(maxTokens, refillRate, refillInterval = 1000) {
        this.maxTokens = maxTokens;
        this.refillRate = refillRate; // tokens per interval
        this.refillInterval = refillInterval; // milliseconds
        
        // Map of key -> bucket state
        this.buckets = new Map();
        
        // Start the refill process
        this.startRefillTimer();
    }
    
    isAllowed(key, tokensRequested = 1) {
        const now = Date.now();
        
        // Get or create bucket for this key
        if (!this.buckets.has(key)) {
            this.buckets.set(key, {
                tokens: this.maxTokens,
                lastRefill: now
            });
        }
        
        const bucket = this.buckets.get(key);
        
        // Refill tokens based on time passed (catch up mechanism)
        const timePassed = now - bucket.lastRefill;
        const intervalsElapsed = Math.floor(timePassed / this.refillInterval);
        
        if (intervalsElapsed > 0) {
            const tokensToAdd = intervalsElapsed * this.refillRate;
            bucket.tokens = Math.min(this.maxTokens, bucket.tokens + tokensToAdd);
            bucket.lastRefill = now;
        }
        
        // Check if we have enough tokens
        if (bucket.tokens >= tokensRequested) {
            bucket.tokens -= tokensRequested;
            
            return {
                allowed: true,
                tokensRemaining: bucket.tokens,
                retryAfter: 0
            };
        } else {
            // Calculate when enough tokens will be available
            const tokensNeeded = tokensRequested - bucket.tokens;
            const timeToWait = Math.ceil(tokensNeeded / this.refillRate) * this.refillInterval;
            
            return {
                allowed: false,
                tokensRemaining: bucket.tokens,
                retryAfter: timeToWait
            };
        }
    }
    
    startRefillTimer() {
        setInterval(() => {
            const now = Date.now();
            
            // Cleanup old buckets and refill active ones
            for (const [key, bucket] of this.buckets.entries()) {
                // Remove buckets that haven't been used for 10 minutes
                if (now - bucket.lastRefill > 600000) {
                    this.buckets.delete(key);
                }
            }
        }, this.refillInterval);
    }
}

// Express.js middleware example
function createRateLimitMiddleware(limiter) {
    return (req, res, next) => {
        const key = req.ip; // or req.user?.id for authenticated users
        const result = limiter.isAllowed(key);
        
        // Add rate limit headers
        res.set({
            'X-RateLimit-Remaining': result.tokensRemaining,
            'X-RateLimit-Retry-After': Math.ceil(result.retryAfter / 1000)
        });
        
        if (result.allowed) {
            next();
        } else {
            res.status(429).json({
                error: 'Rate limit exceeded',
                retryAfter: result.retryAfter
            });
        }
    };
}

// Usage
const limiter = new TokenBucketRateLimiter(100, 10, 1000); // 100 tokens, refill 10/second
const rateLimitMiddleware = createRateLimitMiddleware(limiter);
```

## ⚖️ When To Use / When To Avoid

**✅ Use Rate Limiting When:**
• You have public APIs that could be abused or overwhelmed
• You need to ensure fair usage across multiple tenants/users
• Your downstream services have capacity constraints
• You want to prevent accidental DDoS