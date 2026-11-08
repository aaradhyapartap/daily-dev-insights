# 📌 Caching strategies (Redis, CDN, in-memory)
*November 08, 2026 · Daily Dev Insight*

## 🧠 Overview

Caching is the art of remembering expensive answers so you don't have to recompute them. It's the difference between a snappy user experience and watching your database melt under load at 3 AM. Yet despite being fundamental, caching is often implemented as an afterthought, bolted on when performance problems become unbearable rather than designed into the system architecture from the start.

The three main caching strategies—in-memory, Redis, and CDN—exist at different layers of your application stack, each solving distinct problems. In-memory caching lives in your application process, offering nanosecond lookups at the cost of being local to a single instance. Redis provides shared state across your service fleet with millisecond latency, perfect for session data and computed results. CDNs cache at the network edge, bringing static assets and API responses physically closer to users worldwide.

The real skill isn't choosing one strategy—it's orchestrating all three into a coherent caching hierarchy. Your hottest data deserves in-memory speed. Shared state needs Redis. Public assets belong on a CDN. Understanding which layer handles what separates applications that scale from those that crumble.

## 💡 Key Concepts

- **Cache invalidation is the hard part**: Phil Karlton was right—naming things and cache invalidation are genuinely difficult. Use TTLs aggressively, implement versioning in cache keys, and accept that stale data for 30 seconds is often fine.

- **Multi-layer caching compounds performance**: Check in-memory first, fall back to Redis, then hit the database. Each layer reduces load on slower backends exponentially.

- **Memory is finite, eviction policies matter**: LRU (Least Recently Used) works for most cases, but LFU (Least Frequently Used) can be better for power-law distributions. Know what your cache will evict under pressure.

- **Cache stampedes will wreck your day**: When a popular cache key expires, hundreds of requests simultaneously hit your database. Use techniques like probabilistic early expiration or lock-based cache warming.

- **CDNs aren't just for static files**: Modern CDNs can cache API responses with proper `Cache-Control` headers. This moves compute to the edge and can reduce origin server load by 80%+.

## 🐍 Python Example

```python
import redis
from functools import wraps
from datetime import timedelta
import hashlib
import json

# Multi-layer cache: in-memory + Redis
class MultiLayerCache:
    def __init__(self, redis_url='redis://localhost:6379'):
        self.redis_client = redis.from_url(redis_url)
        self.local_cache = {}  # In-memory cache
        
    def cached(self, ttl_seconds=300):
        """Decorator for multi-layer caching"""
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                # Create cache key from function name and arguments
                cache_key = f"{func.__name__}:{hashlib.md5(
                    json.dumps([args, kwargs], sort_keys=True).encode()
                ).hexdigest()}"
                
                # Layer 1: Check in-memory cache (nanoseconds)
                if cache_key in self.local_cache:
                    print(f"🎯 In-memory hit: {cache_key}")
                    return self.local_cache[cache_key]
                
                # Layer 2: Check Redis (milliseconds)
                redis_value = self.redis_client.get(cache_key)
                if redis_value:
                    print(f"⚡ Redis hit: {cache_key}")
                    result = json.loads(redis_value)
                    self.local_cache[cache_key] = result  # Populate L1
                    return result
                
                # Layer 3: Compute and cache (seconds/minutes)
                print(f"💾 Cache miss, computing: {cache_key}")
                result = func(*args, **kwargs)
                
                # Store in both layers
                self.redis_client.setex(
                    cache_key, 
                    ttl_seconds, 
                    json.dumps(result)
                )
                self.local_cache[cache_key] = result
                return result
            return wrapper
        return decorator

# Usage example
cache = MultiLayerCache()

@cache.cached(ttl_seconds=600)
def expensive_computation(user_id, category):
    # Simulate expensive database query or API call
    import time
    time.sleep(2)
    return {"user_id": user_id, "recommendations": [1, 2, 3]}

# First call: cache miss
result = expensive_computation(123, "books")
# Second call: in-memory hit (instant)
result = expensive_computation(123, "books")
```

## 🟨 JavaScript Example

```javascript
const Redis = require('ioredis');
const NodeCache = require('node-cache');

class CacheManager {
  constructor() {
    this.redis = new Redis({
      host: 'localhost',
      port: 6379,
      retryStrategy: (times) => Math.min(times * 50, 2000)
    });
    
    // In-memory cache with 5 minute default TTL
    this.memCache = new NodeCache({ stdTTL: 300, checkperiod: 60 });
  }

  async get(key, fallbackFn, ttl = 600) {
    // L1: In-memory check
    const memValue = this.memCache.get(key);
    if (memValue !== undefined) {
      console.log(`🎯 In-memory hit: ${key}`);
      return memValue;
    }

    // L2: Redis check with cache stampede protection
    const lockKey = `lock:${key}`;
    try {
      const redisValue = await this.redis.get(key);
      if (redisValue) {
        console.log(`⚡ Redis hit: ${key}`);
        const parsed = JSON.parse(redisValue);
        this.memCache.set(key, parsed);
        return parsed;
      }

      // Acquire lock to prevent stampede (SET NX with expiry)
      const lockAcquired = await this.redis.set(
        lockKey, '1', 'EX', 10, 'NX'
      );
      
      if (!lockAcquired) {
        // Another process is computing, wait briefly and retry
        await new Promise(resolve => setTimeout(resolve, 100));
        return this.get(key, fallbackFn, ttl);
      }

      // L3: Compute and populate caches
      console.log(`💾 Cache miss, computing: ${key}`);
      const result = await fallbackFn();
      
      await this.redis.setex(key, ttl, JSON.stringify(result));
      this.memCache.set(key, result);
      await this.redis.del(lockKey);
      
      return result;
    } catch (error) {
      await this.redis.del(lockKey);
      throw error;
    }
  }
}

// Usage
const cache = new CacheManager();

async function getProductRecommendations(userId) {
  return cache.get(
    `recommendations:${userId}`,
    async () => {
      // Expensive operation
      return { products: [101, 102, 103] };
    },
    3600
  );
}
```

## ⚖️ When To Use / When To Avoid

**✅ When To Use:**
- **In-memory**: Hot path data accessed thousands of times per second (session lookups, feature flags, config)
- **Redis**: Shared state across multiple servers (user sessions, rate limiting, leaderboards)
- **CDN**: Public or semi-public assets and responses (images, videos, API responses with `Vary` headers)
- You have measurably slow operations (>100ms) that are frequently repeated
- Your data can tolerate some staleness (most data can!)

**❌ When To Avoid:**
- You have strong consistency requirements (financial transactions, inventory