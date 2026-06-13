# 📌 API rate limiting and throttling
*June 13, 2026 · Daily Dev Insight*

## 🧠 Overview

API rate limiting isn't just about preventing abuse—it's about building resilient systems that can gracefully handle demand while maintaining quality of service. Think of it as the bouncer at your favorite club: they don't exist to ruin your night, but to ensure everyone inside has a good experience. Without proper rate limiting, a single misbehaving client or unexpected traffic spike can bring down your entire service.

The subtle art lies in choosing the right strategy. Token bucket algorithms provide burst capacity for legitimate use cases, while sliding window approaches offer more predictable resource consumption. The key is understanding that rate limiting is a product decision as much as a technical one—too restrictive and you frustrate users, too lenient and you risk system instability. Modern APIs often implement multiple layers: per-user limits, global throttling, and even intelligent rate limiting that adapts based on user behavior patterns.

## 💡 Key Concepts

• **Token Bucket Algorithm**: Allows burst traffic up to a limit, then enforces steady-state rates—perfect for APIs that need to handle legitimate spikes
• **Sliding Window**: More memory-intensive but provides smoother rate enforcement by tracking requests over a rolling time period
• **Distributed Rate Limiting**: Using Redis or similar stores to coordinate limits across multiple server instances
• **Graceful Degradation**: Instead of hard rejections, consider queuing, reducing response detail, or offering cached results
• **Rate Limit Headers**: Always communicate remaining quotas and reset times to clients via standardized headers

## 🐍 Python Example

```python
import time
import redis
from typing import Optional
from datetime import datetime, timedelta

class DistributedRateLimiter:
    def __init__(self, redis_client: redis.Redis, default_limit: int = 100, window_seconds: int = 3600):
        self.redis = redis_client
        self.default_limit = default_limit
        self.window_seconds = window_seconds
    
    def is_allowed(self, key: str, limit: Optional[int] = None) -> dict:
        """
        Sliding window rate limiter using Redis sorted sets
        Returns dict with allowed status and metadata
        """
        limit = limit or self.default_limit
        now = time.time()
        window_start = now - self.window_seconds
        
        pipe = self.redis.pipeline()
        
        # Remove expired entries
        pipe.zremrangebyscore(key, 0, window_start)
        
        # Count current requests in window
        pipe.zcard(key)
        
        # Add current request timestamp
        pipe.zadd(key, {str(now): now})
        
        # Set expiration on the key
        pipe.expire(key, self.window_seconds + 1)
        
        results = pipe.execute()
        current_count = results[1] + 1  # +1 for the request we just added
        
        allowed = current_count <= limit
        remaining = max(0, limit - current_count)
        
        # If over limit, remove the request we just added
        if not allowed:
            self.redis.zrem(key, str(now))
        
        return {
            'allowed': allowed,
            'limit': limit,
            'remaining': remaining,
            'reset_time': int(now + self.window_seconds),
            'current_count': current_count - (0 if allowed else 1)
        }

# Usage example
redis_client = redis.Redis(host='localhost', port=6379, db=0)
limiter = DistributedRateLimiter(redis_client)

def api_endpoint_decorator(limit: int = 100):
    def decorator(func):
        def wrapper(user_id: str, *args, **kwargs):
            result = limiter.is_allowed(f"rate_limit:{user_id}", limit)
            
            if not result['allowed']:
                return {
                    'error': 'Rate limit exceeded',
                    'retry_after': result['reset_time'] - int(time.time())
                }, 429
            
            # Add rate limit headers to response
            response = func(*args, **kwargs)
            response['rate_limit_remaining'] = result['remaining']
            return response
        return wrapper
    return decorator
```

## 🟨 JavaScript Example

```javascript
const Redis = require('ioredis');

class TokenBucketLimiter {
    constructor(redisClient, capacity = 100, refillRate = 10) {
        this.redis = redisClient;
        this.capacity = capacity;
        this.refillRate = refillRate; // tokens per second
    }

    async checkAndConsume(key, tokens = 1) {
        const now = Date.now() / 1000;
        const bucketKey = `bucket:${key}`;
        
        // Lua script for atomic token bucket operations
        const luaScript = `
            local key = KEYS[1]
            local capacity = tonumber(ARGV[1])
            local refill_rate = tonumber(ARGV[2])
            local requested_tokens = tonumber(ARGV[3])
            local now = tonumber(ARGV[4])
            
            local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
            local tokens = tonumber(bucket[1]) or capacity
            local last_refill = tonumber(bucket[2]) or now
            
            -- Calculate tokens to add based on time elapsed
            local time_elapsed = now - last_refill
            local tokens_to_add = math.floor(time_elapsed * refill_rate)
            tokens = math.min(capacity, tokens + tokens_to_add)
            
            local allowed = tokens >= requested_tokens
            
            if allowed then
                tokens = tokens - requested_tokens
            end
            
            -- Update bucket state
            redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
            redis.call('EXPIRE', key, 3600)
            
            return {
                allowed and 1 or 0,
                tokens,
                capacity,
                math.ceil(math.max(0, requested_tokens - tokens) / refill_rate)
            }
        `;

        const result = await this.redis.eval(
            luaScript,
            1,
            bucketKey,
            this.capacity,
            this.refillRate,
            tokens,
            now
        );

        return {
            allowed: result[0] === 1,
            remaining: result[1],
            capacity: result[2],
            retryAfter: result[3] // seconds until enough tokens available
        };
    }
}

// Express.js middleware example
const redis = new Redis(process.env.REDIS_URL);
const limiter = new TokenBucketLimiter(redis, 1000, 10);

const rateLimitMiddleware = (tokensRequired = 1) => {
    return async (req, res, next) => {
        const userId = req.user?.id || req.ip;
        const result = await limiter.checkAndConsume(userId, tokensRequired);
        
        // Set rate limit headers
        res.set({
            'X-RateLimit-Limit': result.capacity,
            'X-RateLimit-Remaining': result.remaining,
            'X-RateLimit-Reset': new Date(Date.now() + 3600000).toISOString()
        });
        
        if (!result.allowed) {
            res.set('Retry-After', result.retryAfter);
            return res.status(429).json({
                error: 'Rate limit exceeded',
                retryAfter: result.retryAfter
            });
        }
        
        next();
    };
};

// Usage: app.get('/api/expensive-operation', rateLimitMiddleware(5), handler);
```

## ⚖️ When To Use / When To Avoid

**✅ When To Use:**
- Public APIs that could face abuse or unexpected traffic spikes
- Resource-intensive operations (database queries, external API calls, file processing)
- Multi-tenant systems where you need to ensure fair resource allocation
-