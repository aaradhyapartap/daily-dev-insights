# 📌 System design: URL shortener
*October 20, 2026 · Daily Dev Insight*

## 🧠 Overview

URL shorteners seem deceptively simple on the surface—take a long URL, generate a short code, store the mapping, and redirect users. But scratch beneath that surface and you'll find a fascinating system design challenge that touches on distributed systems, database scaling, caching strategies, and API rate limiting. Companies like Bitly and TinyURL handle billions of redirects monthly, making this a legitimate large-scale distributed systems problem.

The core challenge isn't just generating short codes—it's doing so at scale while ensuring uniqueness across distributed servers, handling high read-to-write ratios (most URLs are clicked far more than created), and maintaining low-latency redirects globally. You'll need to think about collision handling, base62 encoding for compact URLs, database sharding strategies, and CDN integration. The beauty of this problem is that it forces you to make explicit tradeoffs between consistency, availability, and partition tolerance.

What makes URL shorteners particularly interesting is the write-once, read-many pattern. Once created, a short URL rarely changes, making it perfect for aggressive caching. However, you'll need analytics tracking, spam prevention, expiration handling, and possibly custom alias support—all while keeping redirect latency under 100ms globally.

## 💡 Key Concepts

- **Base62 encoding**: Convert numeric IDs to alphanumeric strings (a-z, A-Z, 0-9) to create compact, URL-safe short codes. A 7-character base62 string gives you 62^7 = 3.5 trillion possible URLs.

- **ID generation strategies**: Choose between auto-incrementing IDs (centralized), distributed ID generators (Snowflake, Twitter's approach), or hash-based methods. Each has tradeoffs for collision risk, predictability, and scalability.

- **Read-heavy architecture**: With read-to-write ratios often exceeding 100:1, design around multi-layer caching (Redis, CDN edge caching) and read replicas rather than write optimization.

- **Database partitioning**: Shard by hash of short code rather than ranges to ensure even distribution. Consider separate hot and cold storage for frequently vs. rarely accessed URLs.

- **Analytics without blocking**: Track clicks asynchronously via message queues to avoid slowing down redirects. Users care about speed, not whether their click was logged instantly.

## 🐍 Python Example

```python
import hashlib
import redis
from datetime import datetime, timedelta

class URLShortener:
    def __init__(self):
        self.redis_client = redis.Redis(host='localhost', decode_responses=True)
        self.base62_chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
        self.counter_key = "url:counter"
    
    def encode_base62(self, num):
        """Convert integer to base62 string"""
        if num == 0:
            return self.base62_chars[0]
        
        result = []
        while num > 0:
            result.append(self.base62_chars[num % 62])
            num //= 62
        return ''.join(reversed(result))
    
    def shorten_url(self, long_url, custom_alias=None, ttl_days=365):
        """Create shortened URL with optional custom alias"""
        if custom_alias:
            # Check if custom alias already exists
            if self.redis_client.exists(f"short:{custom_alias}"):
                raise ValueError("Custom alias already taken")
            short_code = custom_alias
        else:
            # Generate unique ID using Redis counter (distributed-safe)
            url_id = self.redis_client.incr(self.counter_key)
            short_code = self.encode_base62(url_id)
        
        # Store mapping with expiration
        key = f"short:{short_code}"
        self.redis_client.setex(
            key,
            timedelta(days=ttl_days),
            long_url
        )
        
        # Store metadata for analytics
        meta_key = f"meta:{short_code}"
        self.redis_client.hset(meta_key, mapping={
            "created": datetime.utcnow().isoformat(),
            "clicks": 0
        })
        
        return short_code
    
    def resolve_url(self, short_code):
        """Resolve short code to original URL and track click"""
        key = f"short:{short_code}"
        long_url = self.redis_client.get(key)
        
        if long_url:
            # Async increment click counter (fire-and-forget)
            self.redis_client.hincrby(f"meta:{short_code}", "clicks", 1)
            return long_url
        
        return None

# Usage
shortener = URLShortener()
short = shortener.shorten_url("https://github.com/very/long/repository/url")
print(f"Shortened: {short}")  # Output: "b5k" (depends on counter)
```

## 🟨 JavaScript Example

```javascript
const crypto = require('crypto');

class URLShortener {
  constructor() {
    this.storage = new Map(); // Use Redis/DynamoDB in production
    this.base62 = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
  }

  // MD5 hash-based approach (alternative to counter)
  generateShortCode(longUrl, length = 7) {
    const hash = crypto.createHash('md5').update(longUrl).digest('hex');
    let code = '';
    
    // Convert hash segments to base62
    for (let i = 0; i < length; i++) {
      const chunk = parseInt(hash.substr(i * 4, 4), 16);
      code += this.base62[chunk % 62];
    }
    
    // Handle collision (rare with MD5)
    if (this.storage.has(code)) {
      // Append timestamp entropy and retry
      return this.generateShortCode(longUrl + Date.now(), length);
    }
    
    return code;
  }

  async shortenUrl(longUrl, options = {}) {
    const { customAlias, expiresInDays = 365 } = options;
    
    // Validate URL format
    try {
      new URL(longUrl);
    } catch {
      throw new Error('Invalid URL format');
    }

    const shortCode = customAlias || this.generateShortCode(longUrl);
    const expiresAt = Date.now() + (expiresInDays * 86400000);

    this.storage.set(shortCode, {
      longUrl,
      createdAt: Date.now(),
      expiresAt,
      clicks: 0
    });

    return {
      shortCode,
      shortUrl: `https://short.link/${shortCode}`,
      expiresAt: new Date(expiresAt).toISOString()
    };
  }

  async resolveUrl(shortCode) {
    const entry = this.storage.get(shortCode);
    
    if (!entry) return null;
    
    // Check expiration
    if (Date.now() > entry.expiresAt) {
      this.storage.delete(shortCode);
      return null;
    }

    // Increment click count asynchronously
    setImmediate(() => entry.clicks++);
    
    return entry.longUrl;
  }
}

// Usage
const shortener = new URLShortener();
const result = await shortener.shortenUrl('https://github.com/trending');
console.log(result); // { shortCode: 'aB3xK9p', shortUrl: '...', expiresAt: '...' }
```

## ⚖️ When To Use / When To Avoid

**Use URL shorteners when:**
- Character limits matter (SMS, Twitter, printed materials)
- You need click tracking and analytics on shared