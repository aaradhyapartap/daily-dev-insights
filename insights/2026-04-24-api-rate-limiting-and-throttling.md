# 📌 API rate limiting and throttling
*April 24, 2026 · Daily Dev Insight*

## 🧠 Overview

API rate limiting isn't just about preventing abuse—it's about designing sustainable systems that gracefully degrade under pressure. Think of it as the traffic signals of the internet: without proper flow control, even the most robust APIs can crumble under unexpected load spikes, legitimate or otherwise.

The fundamental challenge lies in balancing accessibility with stability. Too restrictive, and you frustrate legitimate users; too lenient, and a single misbehaving client can bring down your entire service. Modern rate limiting strategies go beyond simple request counting, incorporating adaptive algorithms that consider user behavior patterns, resource consumption, and system health metrics.

Smart rate limiting also serves as a forcing function for better API design. When clients know they have limited requests, they naturally gravitate toward more efficient usage patterns—batching operations, implementing proper caching, and avoiding redundant calls. This creates a virtuous cycle where both API providers and consumers benefit from more thoughtful integration patterns.

## 💡 Key Concepts

• **Token Bucket Algorithm**: Most flexible approach where tokens replenish at a steady rate, allowing burst traffic while maintaining long-term limits
• **Sliding Window**: Tracks requests over a moving time period, providing smoother rate limiting than fixed windows but requiring more memory
• **Rate Limit Headers**: Essential for client cooperation—always include `X-RateLimit-Remaining`, `X-RateLimit-Reset`, and `Retry-After` headers
• **Hierarchical Limiting**: Implement multiple tiers (per-user, per-endpoint, per-IP) to prevent various attack vectors while maintaining usability
• **Graceful Degradation**: Design rate limits that preserve core functionality while throttling less critical operations during high load

## 🐍 Python Example

```python
import time
import asyncio
from collections import defaultdict, deque
from dataclasses import dataclass
from typing import Dict, Optional

@dataclass
class RateLimitConfig:
    requests_per_window: int
    window_seconds: int
    burst_allowance: int = 0

class SlidingWindowRateLimiter:
    def __init__(self):
        # Store request timestamps for each client
        self.client_requests: Dict[str, deque] = defaultdict(lambda: deque())
        self.configs: Dict[str, RateLimitConfig] = {}
    
    def configure_endpoint(self, endpoint: str, config: RateLimitConfig):
        """Configure rate limiting for a specific endpoint"""
        self.configs[endpoint] = config
    
    def is_allowed(self, client_id: str, endpoint: str) -> tuple[bool, dict]:
        """Check if request is allowed and return rate limit info"""
        if endpoint not in self.configs:
            return True, {}
        
        config = self.configs[endpoint]
        now = time.time()
        client_key = f"{client_id}:{endpoint}"
        requests = self.client_requests[client_key]
        
        # Clean old requests outside the window
        cutoff = now - config.window_seconds
        while requests and requests[0] < cutoff:
            requests.popleft()
        
        # Check if we're within limits
        current_count = len(requests)
        limit = config.requests_per_window + config.burst_allowance
        
        if current_count >= limit:
            # Calculate when the oldest request will expire
            retry_after = int(requests[0] - cutoff) if requests else 1
            return False, {
                'X-RateLimit-Limit': str(limit),
                'X-RateLimit-Remaining': '0',
                'X-RateLimit-Reset': str(int(now + retry_after)),
                'Retry-After': str(retry_after)
            }
        
        # Allow the request and record it
        requests.append(now)
        remaining = limit - current_count - 1
        
        return True, {
            'X-RateLimit-Limit': str(limit),
            'X-RateLimit-Remaining': str(remaining),
            'X-RateLimit-Reset': str(int(now + config.window_seconds))
        }

# Usage example
limiter = SlidingWindowRateLimiter()
limiter.configure_endpoint('/api/search', RateLimitConfig(100, 3600, 20))  # 100/hour + 20 burst

allowed, headers = limiter.is_allowed('user123', '/api/search')
print(f"Request allowed: {allowed}")
print(f"Headers: {headers}")
```

## 🟨 JavaScript Example

```javascript
class TokenBucketRateLimiter {
    constructor() {
        this.buckets = new Map();
        this.configs = new Map();
        
        // Clean up old buckets every 5 minutes
        setInterval(() => this.cleanup(), 5 * 60 * 1000);
    }
    
    configure(endpoint, { capacity, refillRate, refillPeriod = 1000 }) {
        this.configs.set(endpoint, {
            capacity,
            refillRate,
            refillPeriod,
            lastRefill: Date.now()
        });
    }
    
    async checkLimit(clientId, endpoint) {
        const config = this.configs.get(endpoint);
        if (!config) return { allowed: true, headers: {} };
        
        const bucketKey = `${clientId}:${endpoint}`;
        const bucket = this.getBucket(bucketKey, config);
        
        // Refill tokens based on elapsed time
        this.refillBucket(bucket, config);
        
        if (bucket.tokens >= 1) {
            bucket.tokens -= 1;
            return {
                allowed: true,
                headers: {
                    'X-RateLimit-Limit': config.capacity.toString(),
                    'X-RateLimit-Remaining': Math.floor(bucket.tokens).toString(),
                    'X-RateLimit-Reset': (Date.now() + config.refillPeriod).toString()
                }
            };
        }
        
        // Calculate retry delay based on refill rate
        const tokensNeeded = 1 - bucket.tokens;
        const retryAfter = Math.ceil((tokensNeeded / config.refillRate) * config.refillPeriod / 1000);
        
        return {
            allowed: false,
            headers: {
                'X-RateLimit-Limit': config.capacity.toString(),
                'X-RateLimit-Remaining': '0',
                'X-RateLimit-Reset': (Date.now() + retryAfter * 1000).toString(),
                'Retry-After': retryAfter.toString()
            }
        };
    }
    
    getBucket(bucketKey, config) {
        if (!this.buckets.has(bucketKey)) {
            this.buckets.set(bucketKey, {
                tokens: config.capacity,
                lastRefill: Date.now()
            });
        }
        return this.buckets.get(bucketKey);
    }
    
    refillBucket(bucket, config) {
        const now = Date.now();
        const elapsed = now - bucket.lastRefill;
        const tokensToAdd = (elapsed / config.refillPeriod) * config.refillRate;
        
        bucket.tokens = Math.min(config.capacity, bucket.tokens + tokensToAdd);
        bucket.lastRefill = now;
    }
    
    cleanup() {
        const oneHourAgo = Date.now() - 60 * 60 * 1000;
        for (const [key, bucket] of this.buckets) {
            if (bucket.lastRefill < oneHourAgo) {
                this.buckets.delete(key);
            }
        }
    }
}

// Express.js middleware example
const limiter = new TokenBucketRateLimiter();
limiter.configure('/api/upload', { capacity: 10, refillRate: 1 }); // 1 token per second, 10 max

const rateLimitMiddleware = (endpoint) => async (req