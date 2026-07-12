# 📌 System design: URL shortener
*July 12, 2026 · Daily Dev Insight*

## 🧠 Overview

URL shorteners seem deceptively simple on the surface—take a long URL, generate a short code, redirect users—but they're actually a masterclass in distributed systems design. This is why companies like Bitly and TinyURL continue to thrive, and why "design a URL shortener" remains one of the most popular system design interview questions. The challenge isn't just about storing mappings; it's about doing so at massive scale while maintaining single-digit millisecond latency.

What makes this system design fascinating is the interplay between write and read patterns. URL shorteners are heavily read-skewed (some links get clicked millions of times while being created just once), which demands aggressive caching strategies. You'll also need to think about unique ID generation in a distributed environment, collision handling, and analytics tracking—all while keeping your database from becoming a bottleneck.

The architectural decisions you make here—choosing between hash-based vs. counter-based ID generation, SQL vs. NoSQL, synchronous vs. asynchronous analytics—have direct implications on scalability, data consistency, and operational costs. This is systems thinking at its finest.

## 💡 Key Concepts

- **ID Generation Strategy**: Choose between base62 encoding of auto-incrementing IDs (predictable, requires coordination) or hash-based approaches (distributed-friendly but needs collision handling). Base62 gives you ~3.5 trillion combinations with just 7 characters.

- **Database Selection**: NoSQL (like Redis/DynamoDB) excels at key-value lookups with horizontal scaling, while SQL provides ACID guarantees and easier analytics. Most production systems use a hybrid approach.

- **Caching Layer**: Since 80% of clicks typically target 20% of URLs, an LRU cache (Redis/Memcached) in front of your database can reduce read latency from ~50ms to <5ms and dramatically cut database load.

- **Rate Limiting**: Prevent abuse through API rate limiting per user/IP. Without this, malicious actors can exhaust your ID space or overwhelm your system with spam URLs.

- **Analytics Pipeline**: Use asynchronous processing (message queues like Kafka) to track clicks without impacting redirect latency. Store click events in a time-series database for efficient aggregation.

## 🐍 Python Example

```python
import hashlib
import string
from datetime import datetime, timedelta

class URLShortener:
    def __init__(self):
        # In production, use Redis or DynamoDB
        self.url_map = {}
        self.analytics = {}
        self.base62_chars = string.ascii_letters + string.digits
        self.counter = 100000  # Start at 6-digit base62
    
    def _encode_base62(self, num):
        """Convert number to base62 string"""
        if num == 0:
            return self.base62_chars[0]
        
        result = []
        while num:
            result.append(self.base62_chars[num % 62])
            num //= 62
        return ''.join(reversed(result))
    
    def shorten(self, long_url, custom_alias=None):
        """Create shortened URL with optional custom alias"""
        if custom_alias:
            if custom_alias in self.url_map:
                raise ValueError("Alias already exists")
            short_code = custom_alias
        else:
            # Use counter-based approach for guaranteed uniqueness
            short_code = self._encode_base62(self.counter)
            self.counter += 1
        
        self.url_map[short_code] = {
            'url': long_url,
            'created': datetime.now(),
            'clicks': 0
        }
        return f"short.ly/{short_code}"
    
    def redirect(self, short_code):
        """Retrieve original URL and track analytics"""
        if short_code not in self.url_map:
            return None
        
        # Increment click counter (in prod, do this async)
        self.url_map[short_code]['clicks'] += 1
        return self.url_map[short_code]['url']

# Usage
shortener = URLShortener()
short_url = shortener.shorten("https://github.com/user/very-long-repository-name")
print(f"Shortened: {short_url}")  # short.ly/q0K

original = shortener.redirect("q0K")
print(f"Redirects to: {original}")
```

## 🟨 JavaScript Example

```javascript
const crypto = require('crypto');

class URLShortener {
  constructor() {
    this.urlMap = new Map();  // In production: Redis/DynamoDB
    this.cache = new Map();   // LRU cache (use node-cache in prod)
    this.cacheMaxSize = 1000;
    this.counter = 100000;
  }

  // Base62 encoding for compact URLs
  encodeBase62(num) {
    const chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
    let result = '';
    
    while (num > 0) {
      result = chars[num % 62] + result;
      num = Math.floor(num / 62);
    }
    return result || chars[0];
  }

  // Hash-based approach (alternative to counter)
  generateHash(url) {
    const hash = crypto.createHash('md5').update(url).digest('hex');
    return hash.substring(0, 7);  // Take first 7 chars
  }

  async shorten(longUrl, userId = null) {
    // Rate limiting check (simplified)
    if (userId && this.checkRateLimit(userId)) {
      throw new Error('Rate limit exceeded');
    }

    const shortCode = this.encodeBase62(this.counter++);
    
    this.urlMap.set(shortCode, {
      url: longUrl,
      created: Date.now(),
      clicks: 0,
      userId: userId
    });

    return `https://short.ly/${shortCode}`;
  }

  async redirect(shortCode) {
    // Check cache first
    if (this.cache.has(shortCode)) {
      return this.cache.get(shortCode);
    }

    const entry = this.urlMap.get(shortCode);
    if (!entry) return null;

    // Update cache
    this.cache.set(shortCode, entry.url);
    if (this.cache.size > this.cacheMaxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);  // Simple LRU
    }

    // Track analytics asynchronously (non-blocking)
    setImmediate(() => {
      entry.clicks++;
    });

    return entry.url;
  }

  checkRateLimit(userId) {
    // Implement token bucket or sliding window
    return false;  // Simplified
  }
}

module.exports = URLShortener;
```

## ⚖️ When To Use / When To Avoid

**When To Use:**
- Building marketing campaign trackers with click analytics
- Creating branded short links for social media sharing
- Implementing QR code generators (shorter URLs = simpler QR codes)
- Learning distributed systems concepts (ID generation, caching, sharding)

**When To Avoid:**
- Internal microservice communication (use service discovery instead)
- Situations requiring guaranteed link longevity (URL shorteners can shut down)
- When URL transparency is critical for security/trust
- High-security applications (shortened URLs obscure destination)

## 📚 Further Reading

- [Designing Data-Intensive Applications](https://dataintensive.net/) - Chapter on distributed ID generation and caching strategies
- [System Design Primer: URL Shortener](https://github.com/donnemartin/system-design-primer#design-a-url-shortening-service-like-bitly) - Comprehensive breakdown with scalability considerations
- [Base62 Encoding Explained](https://en.wikipedia.org/wiki/Base62) - Mathematics behind compact URL generation
-