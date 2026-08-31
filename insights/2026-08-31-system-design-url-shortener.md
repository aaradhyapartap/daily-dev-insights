# 📌 System design: URL shortener
*August 31, 2026 · Daily Dev Insight*

## 🧠 Overview

URL shorteners seem deceptively simple on the surface—take a long URL, give back a short one. But peek under the hood and you'll find a fascinating intersection of distributed systems, database design, and API architecture. The classic interview question has become a classic for good reason: it scales from a 30-minute coding exercise to a multi-region, billion-request-per-day system depending on how deep you go.

At its core, a URL shortener is a key-value mapping service with some interesting constraints. You need to generate unique, short identifiers (typically 6-8 characters), store the mapping efficiently, and redirect users with sub-100ms latency. The real complexity emerges when you consider collision handling, analytics tracking, custom aliases, expiration policies, and rate limiting. A production system must also handle the read-heavy nature of the workload—shortened URLs get clicked far more often than they're created—which makes caching strategy critical.

The beauty of this system design is that it forces you to think about trade-offs at every layer: SQL vs NoSQL for storage, base62 vs hash-based encoding for ID generation, atomic counters vs distributed ID generators for uniqueness, and CDN strategies for global distribution. These decisions ripple through your entire architecture.

## 💡 Key Concepts

- **ID Generation Strategy**: Use base62 encoding (a-z, A-Z, 0-9) to convert numeric IDs into short strings, or hash the URL with MD5/SHA and take the first N characters (with collision handling). Auto-incrementing counters work for single-server setups, but distributed systems need Snowflake-like ID generation or coordination services like Zookeeper.

- **Database Choice Matters**: NoSQL databases like Redis or DynamoDB excel at simple key-value lookups with microsecond latency. SQL works fine for smaller scales and when you need complex analytics queries. The read:write ratio (typically 100:1 or higher) heavily favors cache-first architectures.

- **Two-Way Indexing**: You need fast lookups in both directions—short code to long URL (for redirects) and long URL to short code (to prevent duplicate shortened URLs for the same destination). This typically means two indexes or a bidirectional hash table.

- **Scalability & Caching**: CDNs and in-memory caches (Redis, Memcached) should handle 90%+ of redirect requests. Geographic distribution becomes critical at scale—users in Tokyo shouldn't hit your US-East datacenter for every click.

- **Analytics & Metadata**: Production systems track clicks, referrers, geographic data, and user agents. This telemetry often lives in a separate analytics pipeline (Kafka + clickstream processing) to avoid slowing down the hot path.

## 🐍 Python Example

```python
import hashlib
import string
from typing import Optional

class URLShortener:
    """Simple in-memory URL shortener with base62 encoding"""
    
    BASE62 = string.ascii_letters + string.digits
    
    def __init__(self):
        self.url_to_code = {}  # Long URL -> short code
        self.code_to_url = {}  # Short code -> long URL
        self.counter = 100000  # Start at 100k for 5+ char codes
    
    def encode_base62(self, num: int) -> str:
        """Convert integer to base62 string"""
        if num == 0:
            return self.BASE62[0]
        
        result = []
        while num > 0:
            result.append(self.BASE62[num % 62])
            num //= 62
        return ''.join(reversed(result))
    
    def shorten(self, long_url: str, custom_alias: Optional[str] = None) -> str:
        """Create a shortened URL, returning existing code if URL already shortened"""
        # Check if we've already shortened this URL
        if long_url in self.url_to_code:
            return self.url_to_code[long_url]
        
        # Use custom alias if provided, otherwise generate from counter
        if custom_alias:
            if custom_alias in self.code_to_url:
                raise ValueError(f"Alias '{custom_alias}' already taken")
            short_code = custom_alias
        else:
            short_code = self.encode_base62(self.counter)
            self.counter += 1
        
        # Store bidirectional mapping
        self.url_to_code[long_url] = short_code
        self.code_to_url[short_code] = long_url
        
        return short_code
    
    def expand(self, short_code: str) -> Optional[str]:
        """Retrieve original URL from short code"""
        return self.code_to_url.get(short_code)

# Usage example
shortener = URLShortener()
code = shortener.shorten("https://github.com/torvalds/linux")
print(f"Shortened: {code}")  # Output: "q0U" or similar
print(f"Expanded: {shortener.expand(code)}")

# Custom alias
custom = shortener.shorten("https://python.org", custom_alias="pyorg")
print(f"Custom: {custom}")  # Output: "pyorg"
```

## 🟨 JavaScript Example

```javascript
class URLShortener {
  constructor() {
    this.urlToCode = new Map();
    this.codeToUrl = new Map();
    this.counter = 100000;
    this.BASE62 = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
  }

  /**
   * Convert integer to base62 string
   */
  encodeBase62(num) {
    if (num === 0) return this.BASE62[0];
    
    let result = '';
    while (num > 0) {
      result = this.BASE62[num % 62] + result;
      num = Math.floor(num / 62);
    }
    return result;
  }

  /**
   * Shorten a URL with optional custom alias
   */
  shorten(longUrl, customAlias = null) {
    // Return existing code if already shortened
    if (this.urlToCode.has(longUrl)) {
      return this.urlToCode.get(longUrl);
    }

    let shortCode;
    if (customAlias) {
      if (this.codeToUrl.has(customAlias)) {
        throw new Error(`Alias '${customAlias}' already taken`);
      }
      shortCode = customAlias;
    } else {
      shortCode = this.encodeBase62(this.counter++);
    }

    // Bidirectional mapping
    this.urlToCode.set(longUrl, shortCode);
    this.codeToUrl.set(shortCode, longUrl);

    return shortCode;
  }

  /**
   * Expand short code to original URL
   */
  expand(shortCode) {
    return this.codeToUrl.get(shortCode) || null;
  }

  /**
   * Get analytics: total URLs and mappings
   */
  getStats() {
    return {
      totalUrls: this.codeToUrl.size,
      nextId: this.counter
    };
  }
}

// Usage example
const shortener = new URLShortener();
const code = shortener.shorten('https://nodejs.org/api/crypto.html');
console.log(`Shortened: ${code}`);
console.log(`Expanded: ${shortener.expand(code)}`);
console.log(`Stats:`, shortener.getStats());
```

## ⚖️ When To Use / When To Avoid

**✅ When To Use:**
- Marketing campaigns needing trackable, branded links
- Character-limited platforms (SMS, Twitter-style social media)
- Masking complex query parameters or internal routing logic
- A/B testing different landing pages with identical-looking links
- Providing user-friendly aliases for API endpoints or documentation

**❌ When To Avoid:**
- Security-