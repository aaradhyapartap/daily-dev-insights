# 📌 Caching strategies (Redis, CDN, in-memory)
*July 31, 2026 · Daily Dev Insight*

## 🧠 Overview

Caching is the art of remembering expensive computations so you don't have to repeat them. But here's the thing: not all caching is created equal. Choosing between Redis, CDN, and in-memory caching isn't just a technical decision—it's a strategic one that affects your application's performance, cost, and architectural complexity.

In-memory caching is your scratchpad—blazingly fast but volatile and isolated to a single process. Redis is your shared notebook—persistent, distributed, and perfect for multi-server deployments. CDNs are your global publishing network—geographically distributed edge servers that cache static assets close to users. The real skill isn't knowing what each does; it's knowing which layer to use for which data, and how to orchestrate them together.

Most production systems use all three in a layered approach: in-memory for hot paths within a service, Redis for shared state across services, and CDN for static assets and API responses at the edge. The key is understanding the tradeoffs in latency, consistency, and operational overhead.

## 💡 Key Concepts

- **Cache invalidation is the hard part** — Phil Karlton's famous quote remains true: "There are only two hard things in Computer Science: cache invalidation and naming things." Use TTLs liberally and plan your invalidation strategy upfront.

- **Latency hierarchy matters** — In-memory: ~1μs, Redis: ~1ms, CDN: ~10-100ms. Choose based on how often data is accessed and how stale it can be.

- **Write-through vs write-behind** — Write-through updates cache and database synchronously (slower writes, guaranteed consistency). Write-behind queues updates asynchronously (faster writes, eventual consistency).

- **Cache stampede protection** — When a popular cache key expires, don't let 1000 concurrent requests all recompute the same value. Use locking mechanisms or probabilistic early expiration.

- **Size your cache strategically** — The Pareto principle applies: 20% of your data likely handles 80% of requests. Focus on caching high-value, frequently-accessed data, not everything.

## 🐍 Python Example

```python
import redis
import time
from functools import wraps
from threading import Lock

# Multi-layer caching decorator with Redis + in-memory
class CacheLayer:
    def __init__(self, redis_url='redis://localhost:6379'):
        self.redis_client = redis.Redis.from_url(redis_url, decode_responses=True)
        self.memory_cache = {}
        self.locks = {}
        self.lock_manager = Lock()
    
    def cached(self, ttl=300, memory_ttl=60):
        """Two-tier caching: check memory first, then Redis"""
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                cache_key = f"{func.__name__}:{str(args)}:{str(kwargs)}"
                
                # Layer 1: In-memory cache (fastest)
                if cache_key in self.memory_cache:
                    value, expires = self.memory_cache[cache_key]
                    if time.time() < expires:
                        return value
                
                # Layer 2: Redis cache (shared across instances)
                redis_value = self.redis_client.get(cache_key)
                if redis_value:
                    # Populate memory cache for next time
                    self.memory_cache[cache_key] = (redis_value, time.time() + memory_ttl)
                    return redis_value
                
                # Cache miss - prevent stampede with locking
                with self.lock_manager:
                    if cache_key not in self.locks:
                        self.locks[cache_key] = Lock()
                
                with self.locks[cache_key]:
                    # Double-check after acquiring lock
                    redis_value = self.redis_client.get(cache_key)
                    if redis_value:
                        return redis_value
                    
                    # Actually compute the value
                    result = func(*args, **kwargs)
                    
                    # Store in both layers
                    self.redis_client.setex(cache_key, ttl, result)
                    self.memory_cache[cache_key] = (result, time.time() + memory_ttl)
                    
                    return result
            return wrapper
        return decorator

# Usage
cache = CacheLayer()

@cache.cached(ttl=600, memory_ttl=120)
def expensive_api_call(user_id):
    time.sleep(2)  # Simulate expensive operation
    return f"Data for user {user_id}"

print(expensive_api_call(42))  # Takes 2 seconds
print(expensive_api_call(42))  # Instant from memory
```

## 🟨 JavaScript Example

```javascript
const Redis = require('ioredis');
const NodeCache = require('node-cache');

class MultiTierCache {
  constructor() {
    this.redis = new Redis({ host: 'localhost', port: 6379 });
    this.memoryCache = new NodeCache({ stdTTL: 60, checkperiod: 120 });
    this.pendingRequests = new Map();
  }

  /**
   * Cached wrapper with stampede protection
   * @param {Function} fn - Async function to cache
   * @param {string} key - Cache key
   * @param {number} ttl - Redis TTL in seconds
   */
  async withCache(fn, key, ttl = 300) {
    // Layer 1: Check in-memory cache
    const memValue = this.memoryCache.get(key);
    if (memValue !== undefined) {
      return memValue;
    }

    // Layer 2: Check Redis
    const redisValue = await this.redis.get(key);
    if (redisValue !== null) {
      const parsed = JSON.parse(redisValue);
      this.memoryCache.set(key, parsed, 60); // 1 min in memory
      return parsed;
    }

    // Stampede protection: coalesce concurrent requests
    if (this.pendingRequests.has(key)) {
      return await this.pendingRequests.get(key);
    }

    // Create the promise and store it
    const promise = (async () => {
      try {
        const result = await fn();
        
        // Store in both layers
        await this.redis.setex(key, ttl, JSON.stringify(result));
        this.memoryCache.set(key, result, 60);
        
        return result;
      } finally {
        this.pendingRequests.delete(key);
      }
    })();

    this.pendingRequests.set(key, promise);
    return await promise;
  }

  // Invalidate across all layers
  async invalidate(key) {
    this.memoryCache.del(key);
    await this.redis.del(key);
  }
}

// Usage example
const cache = new MultiTierCache();

async function getUserProfile(userId) {
  return await cache.withCache(
    async () => {
      // Simulate expensive DB query
      await new Promise(resolve => setTimeout(resolve, 1000));
      return { id: userId, name: 'John Doe', email: 'john@example.com' };
    },
    `user:${userId}`,
    600 // 10 min TTL
  );
}

// First call: slow
getUserProfile(123).then(console.log);
// Subsequent calls: instant
```

## ⚖️ When To Use / When To Avoid

**✅ Use In-Memory Caching When:**
- Data is accessed very frequently (hot path)
- You can tolerate cache inconsistency across instances
- Working with small datasets (< 1GB typically)
- Single-server deployments or request-scoped caching

**✅ Use Redis When:**
- Need shared cache across multiple application instances
- Require persistence and atomic operations
- Implementing distributed locks or pub/sub patterns
- Cache size exceeds available application