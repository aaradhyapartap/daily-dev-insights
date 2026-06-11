# 📌 Caching strategies (Redis, CDN, in-memory)
*June 11, 2026 · Daily Dev Insight*

## 🧠 Overview

Caching is the closest thing we have to magic in software engineering. Done right, it can transform a sluggish application into a lightning-fast user experience. Done wrong, it becomes a source of stale data nightmares and debugging hell. The key is understanding that not all caches are created equal—each type serves different use cases and comes with distinct trade-offs.

In-memory caching gives you microsecond access times but dies with your process. Redis provides persistence and sharing across services but adds network latency. CDNs excel at static content delivery to global users but aren't suitable for dynamic, personalized data. The art lies in layering these strategies effectively, creating a caching hierarchy that maximizes performance while maintaining data consistency.

Modern applications typically employ a multi-tier caching strategy: in-memory for hot data, Redis for session storage and frequently accessed database queries, and CDNs for static assets and API responses that can be cached at the edge. Understanding when and how to invalidate each layer is what separates senior engineers from junior ones.

## 💡 Key Concepts

• **Cache invalidation patterns**: Time-based expiration (TTL), event-driven invalidation, and cache-aside vs write-through strategies each solve different consistency requirements
• **Cache locality principles**: Keep frequently accessed data closest to the computation—CPU cache, application memory, local Redis, then remote CDN
• **Thundering herd protection**: Use techniques like cache warming, distributed locks, or stale-while-revalidate to prevent cache stampedes
• **Cache key design**: Hierarchical, predictable keys enable efficient bulk invalidation and debugging (e.g., `user:123:profile` vs random UUIDs)
• **Monitoring and observability**: Track hit/miss ratios, eviction rates, and cache penetration to identify optimization opportunities

## 🐍 Python Example

```python
import redis
import asyncio
from functools import wraps
from typing import Any, Optional
import json
import time

class MultiTierCache:
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis_client = redis.from_url(redis_url, decode_responses=True)
        self.memory_cache = {}  # Simple in-memory cache
        self.memory_ttl = {}    # TTL tracking for memory cache
    
    def cache_strategy(self, memory_ttl: int = 60, redis_ttl: int = 300):
        """Decorator implementing multi-tier caching strategy"""
        def decorator(func):
            @wraps(func)
            async def wrapper(*args, **kwargs):
                # Generate cache key from function name and arguments
                cache_key = f"{func.__name__}:{hash(str(args) + str(kwargs))}"
                
                # Tier 1: Check in-memory cache
                if self._is_memory_cache_valid(cache_key):
                    print(f"🎯 Memory cache hit: {cache_key}")
                    return self.memory_cache[cache_key]
                
                # Tier 2: Check Redis cache
                redis_value = self.redis_client.get(cache_key)
                if redis_value:
                    print(f"🔄 Redis cache hit: {cache_key}")
                    result = json.loads(redis_value)
                    # Populate memory cache
                    self.memory_cache[cache_key] = result
                    self.memory_ttl[cache_key] = time.time() + memory_ttl
                    return result
                
                # Cache miss: Execute function and cache result
                print(f"❌ Cache miss: {cache_key}")
                result = await func(*args, **kwargs)
                
                # Store in both tiers
                self.memory_cache[cache_key] = result
                self.memory_ttl[cache_key] = time.time() + memory_ttl
                self.redis_client.setex(cache_key, redis_ttl, json.dumps(result))
                
                return result
            return wrapper
        return decorator
    
    def _is_memory_cache_valid(self, key: str) -> bool:
        return (key in self.memory_cache and 
                key in self.memory_ttl and 
                time.time() < self.memory_ttl[key])

# Usage example
cache = MultiTierCache()

@cache.cache_strategy(memory_ttl=30, redis_ttl=600)
async def expensive_user_query(user_id: int):
    """Simulate expensive database operation"""
    await asyncio.sleep(0.5)  # Simulate DB latency
    return {"user_id": user_id, "name": f"User {user_id}", "data": "expensive_computation"}
```

## 🟨 JavaScript Example

```javascript
const redis = require('redis');
const NodeCache = require('node-cache');

class CacheManager {
  constructor() {
    this.redisClient = redis.createClient({ url: 'redis://localhost:6379' });
    this.memoryCache = new NodeCache({ stdTTL: 60, checkperiod: 120 });
    this.redisClient.connect();
  }

  // Cache-aside pattern with fallback strategy
  async get(key, fetchFunction, options = {}) {
    const { memoryTTL = 60, redisTTL = 300, skipMemory = false } = options;
    
    try {
      // Tier 1: Memory cache (fastest)
      if (!skipMemory) {
        const memoryResult = this.memoryCache.get(key);
        if (memoryResult !== undefined) {
          console.log(`🚀 Memory hit: ${key}`);
          return memoryResult;
        }
      }

      // Tier 2: Redis cache (shared across instances)
      const redisResult = await this.redisClient.get(key);
      if (redisResult) {
        console.log(`📡 Redis hit: ${key}`);
        const parsed = JSON.parse(redisResult);
        
        // Populate memory cache for next time
        if (!skipMemory) {
          this.memoryCache.set(key, parsed, memoryTTL);
        }
        return parsed;
      }

      // Cache miss: fetch data and populate all tiers
      console.log(`💾 Cache miss, fetching: ${key}`);
      const freshData = await fetchFunction();
      
      // Store in Redis with error handling
      try {
        await this.redisClient.setEx(key, redisTTL, JSON.stringify(freshData));
      } catch (redisError) {
        console.warn(`Redis write failed for ${key}:`, redisError.message);
      }
      
      // Store in memory
      if (!skipMemory) {
        this.memoryCache.set(key, freshData, memoryTTL);
      }
      
      return freshData;
    } catch (error) {
      console.error(`Cache error for ${key}:`, error);
      // Fallback to direct fetch if caching fails
      return await fetchFunction();
    }
  }

  // Smart invalidation with pattern support
  async invalidate(pattern) {
    if (pattern.includes('*')) {
      // Pattern-based invalidation
      const keys = await this.redisClient.keys(pattern);
      if (keys.length > 0) {
        await this.redisClient.del(keys);
      }
      this.memoryCache.flushAll(); // Simple approach for memory cache
    } else {
      // Single key invalidation
      await this.redisClient.del(pattern);
      this.memoryCache.del(pattern);
    }
    console.log(`🗑️  Invalidated: ${pattern}`);
  }
}

// Usage with API endpoint caching
const cache = new CacheManager();

async function getUserProfile(userId) {
  return cache.get(
    `user:profile:${userId}`,
    async () => {
      // Simulate database call
      await new Promise(resolve => setTimeout(resolve, 200));
      return { id: