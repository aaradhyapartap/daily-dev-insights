# 📌 System design: URL shortener
*April 03, 2026 · Daily Dev Insight*

## 🧠 Overview

URL shorteners like bit.ly and tinyurl might seem trivial on the surface, but they're actually fascinating distributed systems problems in disguise. At their core, they're bijective functions that map long URLs to short, unique identifiers. The real challenge isn't the mapping itself—it's handling massive scale, ensuring global availability, and maintaining sub-millisecond response times while serving billions of redirects daily.

The beauty of URL shorteners lies in their deceptive simplicity. What appears to be a basic CRUD application quickly reveals complexities around collision handling, cache invalidation, analytics collection, and abuse prevention. They're excellent system design interview questions because they scale from a simple hash table to a multi-region distributed system with sophisticated caching layers, load balancing, and real-time analytics pipelines.

## 💡 Key Concepts

• **Base62 Encoding**: Convert database IDs to short alphanumeric strings (0-9, a-z, A-Z) to maximize URL density while keeping them readable and URL-safe
• **Cache-Heavy Architecture**: Implement multi-layer caching (CDN, Redis, application-level) since read-to-write ratios are typically 100:1 or higher
• **Collision Resistance**: Use auto-incrementing IDs with base conversion rather than pure hashing to guarantee uniqueness without expensive collision detection
• **Geographic Distribution**: Deploy redirect servers globally since latency directly impacts user experience—every millisecond counts for mobile users
• **Analytics Pipeline**: Design for high-throughput click tracking without impacting redirect performance, often using async event streaming

## 🐍 Python Example

```python
import string
from datetime import datetime, timedelta
from typing import Optional
import redis
import hashlib

class URLShortener:
    def __init__(self):
        self.redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
        self.base62_chars = string.digits + string.ascii_letters
        self.counter_key = "url_counter"
        
    def _base62_encode(self, num: int) -> str:
        """Convert integer to base62 string for compact URLs"""
        if num == 0:
            return self.base62_chars[0]
        
        result = []
        while num > 0:
            result.append(self.base62_chars[num % 62])
            num //= 62
        return ''.join(reversed(result))
    
    def _base62_decode(self, encoded: str) -> int:
        """Convert base62 string back to integer"""
        result = 0
        for char in encoded:
            result = result * 62 + self.base62_chars.index(char)
        return result
    
    def shorten_url(self, long_url: str, custom_alias: Optional[str] = None, ttl: int = 86400) -> str:
        """Create short URL with optional custom alias and TTL"""
        # Validate URL format
        if not long_url.startswith(('http://', 'https://')):
            raise ValueError("URL must start with http:// or https://")
        
        if custom_alias:
            # Check if custom alias is available
            if self.redis_client.exists(f"short:{custom_alias}"):
                raise ValueError(f"Custom alias '{custom_alias}' already taken")
            short_code = custom_alias
        else:
            # Generate unique ID and encode
            unique_id = self.redis_client.incr(self.counter_key)
            short_code = self._base62_encode(unique_id)
        
        # Store mapping with TTL and metadata
        mapping_data = {
            'url': long_url,
            'created_at': datetime.utcnow().isoformat(),
            'click_count': 0
        }
        
        pipe = self.redis_client.pipeline()
        pipe.hset(f"short:{short_code}", mapping=mapping_data)
        pipe.expire(f"short:{short_code}", ttl)
        pipe.execute()
        
        return f"https://short.ly/{short_code}"
    
    def expand_url(self, short_code: str) -> Optional[str]:
        """Expand short URL and track analytics"""
        mapping = self.redis_client.hgetall(f"short:{short_code}")
        
        if not mapping:
            return None
        
        # Increment click counter asynchronously
        self.redis_client.hincrby(f"short:{short_code}", 'click_count', 1)
        
        return mapping.get('url')
```

## 🟨 JavaScript Example

```javascript
const Redis = require('ioredis');
const crypto = require('crypto');

class URLShortener {
    constructor() {
        this.redis = new Redis({
            host: 'localhost',
            port: 6379,
            retryDelayOnFailover: 100,
            maxRetriesPerRequest: 3
        });
        this.base62 = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    }

    /**
     * Encode number to base62 for compact representation
     */
    base62Encode(num) {
        if (num === 0) return this.base62[0];
        
        let result = '';
        while (num > 0) {
            result = this.base62[num % 62] + result;
            num = Math.floor(num / 62);
        }
        return result;
    }

    /**
     * Generate cryptographically secure hash for collision resistance
     */
    generateHash(url, timestamp) {
        const hash = crypto.createHash('sha256');
        hash.update(url + timestamp.toString());
        return hash.digest('hex').substring(0, 8);
    }

    /**
     * Create shortened URL with built-in analytics and caching
     */
    async shortenURL(longURL, options = {}) {
        const { customAlias, ttl = 86400, enableAnalytics = true } = options;
        
        // URL validation
        const urlPattern = /^https?:\/\/.+/;
        if (!urlPattern.test(longURL)) {
            throw new Error('Invalid URL format - must start with http:// or https://');
        }

        let shortCode;
        
        if (customAlias) {
            // Check availability of custom alias
            const exists = await this.redis.exists(`url:${customAlias}`);
            if (exists) {
                throw new Error(`Custom alias '${customAlias}' is already taken`);
            }
            shortCode = customAlias;
        } else {
            // Generate unique short code using counter + hash hybrid
            const counter = await this.redis.incr('global:counter');
            const timestamp = Date.now();
            const hash = this.generateHash(longURL, timestamp);
            shortCode = this.base62Encode(counter) + hash.substring(0, 2);
        }

        // Store URL mapping with metadata
        const urlData = {
            originalURL: longURL,
            shortCode,
            createdAt: new Date().toISOString(),
            clickCount: 0,
            lastAccessed: null,
            analyticsEnabled: enableAnalytics
        };

        const pipeline = this.redis.pipeline();
        pipeline.hmset(`url:${shortCode}`, urlData);
        pipeline.expire(`url:${shortCode}`, ttl);
        
        // Add to sorted set for analytics and cleanup
        if (enableAnalytics) {
            pipeline.zadd('url:created', Date.now(), shortCode);
        }
        
        await pipeline.exec();
        
        return {
            shortURL: `https://sho.rt/${shortCode}`,
            shortCode,
            originalURL: longURL,
            expiresAt: new Date(Date.now() + ttl * 1000).toISOString()
        };
    }

    /**
     * Expand short URL