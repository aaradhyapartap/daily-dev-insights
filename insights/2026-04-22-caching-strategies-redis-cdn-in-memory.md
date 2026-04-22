# 📌 Caching strategies (Redis, CDN, in-memory)
*April 22, 2026 · Daily Dev Insight*

## 🧠 Overview

Caching is the art of storing frequently accessed data in a faster, more accessible location—and it's arguably the most bang-for-your-buck performance optimization you can make. Think of it as your application's short-term memory: instead of repeatedly asking your database "What's the weather in Tokyo?" you remember the answer for a reasonable amount of time.

The tricky part isn't implementing caching (that's usually straightforward), but choosing the right strategy and invalidation approach. Redis excels for shared, persistent cache across services; CDNs dominate for static assets and geographic distribution; in-memory caching wins for single-process performance. The real engineering challenge lies in cache consistency, thundering herd problems, and knowing when your cache has become more complex than the problem it's solving.

Most performance issues I've debugged in production boil down to either missing caching opportunities or poorly implemented cache invalidation. Master these three strategies, and you'll solve 90% of your performance bottlenecks before they become firefighting sessions.

## 💡 Key Concepts

• **Cache hierarchy matters**: Start with in-memory for hot data, Redis for shared state, CDN for static assets. Layer them strategically rather than picking just one.

• **TTL vs invalidation trade-offs**: Time-based expiration is simple but potentially stale; event-based invalidation is accurate but complex. Choose based on your consistency requirements.

• **Cache warming prevents cold start disasters**: Pre-populate critical cache entries during deployment rather than letting the first users suffer slow responses.

• **Monitor cache hit rates religiously**: A 90% hit rate sounds good until you realize the 10% of misses are your most expensive queries hammering your database.

• **Thundering herd protection**: When cache expires, ensure only one process rebuilds it while others wait, preventing your database from getting stampeded.

## 🐍 Python Example

```python
import asyncio
import aioredis
from datetime import datetime, timedelta
from typing import Optional, Any
import json

class MultiLayerCache:
    def __init__(self, redis_url: str):
        self.memory_cache = {}  # In-memory L1 cache
        self.redis = None  # Redis L2 cache
        self.redis_url = redis_url
        
    async def connect(self):
        self.redis = await aioredis.from_url(self.redis_url)
    
    async def get(self, key: str, ttl_seconds: int = 300) -> Optional[Any]:
        # L1: Check in-memory cache first
        if key in self.memory_cache:
            data, expires_at = self.memory_cache[key]
            if datetime.now() < expires_at:
                print(f"Cache HIT (memory): {key}")
                return data
            else:
                del self.memory_cache[key]
        
        # L2: Check Redis cache
        redis_value = await self.redis.get(key)
        if redis_value:
            data = json.loads(redis_value)
            # Populate L1 cache with shorter TTL
            self.memory_cache[key] = (
                data, 
                datetime.now() + timedelta(seconds=min(ttl_seconds, 60))
            )
            print(f"Cache HIT (redis): {key}")
            return data
        
        print(f"Cache MISS: {key}")
        return None
    
    async def set(self, key: str, value: Any, ttl_seconds: int = 300):
        # Set in both layers
        expires_at = datetime.now() + timedelta(seconds=min(ttl_seconds, 60))
        self.memory_cache[key] = (value, expires_at)
        
        await self.redis.setex(
            key, 
            ttl_seconds, 
            json.dumps(value, default=str)
        )
    
    async def invalidate(self, pattern: str):
        """Invalidate cache entries matching pattern"""
        # Clear memory cache
        keys_to_delete = [k for k in self.memory_cache.keys() if pattern in k]
        for k in keys_to_delete:
            del self.memory_cache[k]
        
        # Clear Redis cache
        redis_keys = await self.redis.keys(f"*{pattern}*")
        if redis_keys:
            await self.redis.delete(*redis_keys)

# Usage example with database fallback
async def get_user_profile(user_id: int, cache: MultiLayerCache):
    cache_key = f"user_profile:{user_id}"
    
    # Try cache first
    profile = await cache.get(cache_key, ttl_seconds=600)
    if profile:
        return profile
    
    # Cache miss - fetch from database
    profile = await fetch_user_from_database(user_id)  # Your DB call here
    await cache.set(cache_key, profile, ttl_seconds=600)
    
    return profile
```

## 🟨 JavaScript Example

```javascript
const redis = require('redis');
const NodeCache = require('node-cache');

class SmartCache {
  constructor(redisConfig = {}) {
    // L1: In-memory cache with 5-minute default TTL
    this.memoryCache = new NodeCache({ stdTTL: 300, checkperiod: 60 });
    
    // L2: Redis for shared cache across instances
    this.redisClient = redis.createClient(redisConfig);
    this.redisClient.on('error', err => console.error('Redis error:', err));
    
    // Track cache performance
    this.stats = { hits: 0, misses: 0, sets: 0 };
  }

  async get(key, options = {}) {
    const { ttl = 300, skipMemory = false } = options;
    
    // L1: Check memory cache first
    if (!skipMemory) {
      const memoryValue = this.memoryCache.get(key);
      if (memoryValue !== undefined) {
        this.stats.hits++;
        console.log(`💚 Memory cache HIT: ${key}`);
        return memoryValue;
      }
    }
    
    try {
      // L2: Check Redis
      const redisValue = await this.redisClient.get(key);
      if (redisValue) {
        const parsed = JSON.parse(redisValue);
        
        // Populate memory cache with shorter TTL
        this.memoryCache.set(key, parsed, Math.min(ttl, 60));
        
        this.stats.hits++;
        console.log(`🔶 Redis cache HIT: ${key}`);
        return parsed;
      }
    } catch (error) {
      console.warn(`Redis get failed for ${key}:`, error.message);
    }
    
    this.stats.misses++;
    console.log(`❌ Cache MISS: ${key}`);
    return null;
  }

  async set(key, value, ttl = 300) {
    this.stats.sets++;
    
    // Set in memory with shorter TTL to prevent memory bloat
    this.memoryCache.set(key, value, Math.min(ttl, 60));
    
    try {
      // Set in Redis with full TTL
      await this.redisClient.setEx(key, ttl, JSON.stringify(value));
    } catch (error) {
      console.warn(`Redis set failed for ${key}:`, error.message);
    }
  }

  async getOrCompute(key, computeFn, options = {}) {
    const { ttl = 300, computeTimeout = 5000 } = options;
    
    // Try cache first
    const cached = await this.get(key, { ttl });
    if (cached !== null) return cached;
    
    // Cache miss - compute value with timeout protection
    try {
      const value = await Promise.race([
        computeFn(),
        new Promise((_, reject) => 
          setTimeout(() => reject(new Error('Compute timeout')), compute