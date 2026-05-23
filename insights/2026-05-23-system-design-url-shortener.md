# 📌 System design: URL shortener
*May 23, 2026 · Daily Dev Insight*

## 🧠 Overview

URL shorteners like bit.ly, tinyurl.com, and t.co seem deceptively simple on the surface—take a long URL, generate a short code, store the mapping. But beneath this simplicity lies fascinating engineering challenges around distributed systems, caching strategies, and massive scale. When Twitter serves billions of shortened URLs daily, you're not just dealing with a hash map anymore.

The real complexity emerges when you consider the non-functional requirements: sub-100ms response times, 99.9% uptime, analytics tracking, spam prevention, and geographic distribution. These systems are excellent case studies for understanding database sharding, cache invalidation patterns, and the trade-offs between consistency and availability in distributed architectures.

## 💡 Key Concepts

• **Base62 encoding** - Convert numeric IDs to alphanumeric short codes (0-9, a-z, A-Z) for URL-friendly strings
• **Database sharding** - Distribute URL mappings across multiple databases using consistent hashing or range-based partitioning
• **Cache-aside pattern** - Store frequently accessed URLs in Redis/Memcached with TTL-based expiration
• **Rate limiting** - Prevent abuse using token bucket or sliding window algorithms per IP/user
• **Analytics pipeline** - Stream click events to data warehouses for real-time and batch analytics processing

## 🐍 Python Example

```python
import hashlib
import time
from typing import Optional
import redis
import sqlite3

class URLShortener:
    def __init__(self):
        self.redis_client = redis.Redis(host='localhost', port=6379, db=0)
        self.db_conn = sqlite3.connect('urls.db', check_same_thread=False)
        self.base62_chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
        self._init_db()
    
    def _init_db(self):
        """Initialize database schema"""
        cursor = self.db_conn.cursor()
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS urls (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                original_url TEXT NOT NULL,
                short_code TEXT UNIQUE NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                click_count INTEGER DEFAULT 0
            )
        ''')
        self.db_conn.commit()
    
    def _encode_base62(self, num: int) -> str:
        """Convert integer to base62 string"""
        if num == 0:
            return self.base62_chars[0]
        
        result = ""
        while num > 0:
            result = self.base62_chars[num % 62] + result
            num //= 62
        return result
    
    def shorten_url(self, original_url: str) -> str:
        """Generate short URL with collision handling"""
        # Check if URL already exists in cache
        cached_code = self.redis_client.get(f"url:{original_url}")
        if cached_code:
            return cached_code.decode('utf-8')
        
        # Generate short code using timestamp + hash for uniqueness
        timestamp = int(time.time() * 1000)  # milliseconds
        hash_suffix = int(hashlib.md5(original_url.encode()).hexdigest()[:8], 16)
        unique_id = timestamp + hash_suffix
        
        short_code = self._encode_base62(unique_id)[:7]  # Keep it short
        
        try:
            cursor = self.db_conn.cursor()
            cursor.execute(
                "INSERT INTO urls (original_url, short_code) VALUES (?, ?)",
                (original_url, short_code)
            )
            self.db_conn.commit()
            
            # Cache the mapping with 1-hour TTL
            self.redis_client.setex(f"url:{original_url}", 3600, short_code)
            self.redis_client.setex(f"code:{short_code}", 3600, original_url)
            
            return short_code
        except sqlite3.IntegrityError:
            # Handle collision - rare but possible
            return self.shorten_url(original_url + str(time.time()))
```

## 🟨 JavaScript Example

```javascript
const express = require('express');
const redis = require('redis');
const crypto = require('crypto');

class URLShortenerService {
    constructor() {
        this.redisClient = redis.createClient();
        this.base62Chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
        this.rateLimiter = new Map(); // In production, use Redis for distributed rate limiting
    }

    encodeBase62(num) {
        if (num === 0) return this.base62Chars[0];
        
        let result = '';
        while (num > 0) {
            result = this.base62Chars[num % 62] + result;
            num = Math.floor(num / 62);
        }
        return result;
    }

    async checkRateLimit(clientIP) {
        const now = Date.now();
        const windowSize = 60000; // 1 minute
        const maxRequests = 100;
        
        const clientData = this.rateLimiter.get(clientIP) || { count: 0, windowStart: now };
        
        if (now - clientData.windowStart > windowSize) {
            // Reset window
            clientData.count = 1;
            clientData.windowStart = now;
        } else {
            clientData.count++;
        }
        
        this.rateLimiter.set(clientIP, clientData);
        return clientData.count <= maxRequests;
    }

    async expandURL(shortCode) {
        try {
            // Try cache first (cache-aside pattern)
            const cachedURL = await this.redisClient.get(`code:${shortCode}`);
            if (cachedURL) {
                // Increment analytics counter asynchronously
                this.redisClient.incr(`analytics:${shortCode}:${new Date().toISOString().split('T')[0]}`);
                return cachedURL;
            }
            
            // Fallback to database (simulate with Redis hash)
            const originalURL = await this.redisClient.hget('urls', shortCode);
            if (originalURL) {
                // Warm up cache with 2-hour TTL
                await this.redisClient.setex(`code:${shortCode}`, 7200, originalURL);
                this.redisClient.incr(`analytics:${shortCode}:${new Date().toISOString().split('T')[0]}`);
                return originalURL;
            }
            
            return null;
        } catch (error) {
            console.error('Error expanding URL:', error);
            return null;
        }
    }

    generateShortCode(originalURL) {
        // Combine timestamp with hash for uniqueness
        const timestamp = Date.now();
        const hash = crypto.createHash('md5').update(originalURL).digest('hex').substring(0, 8);
        const uniqueId = timestamp + parseInt(hash, 16);
        
        return this.encodeBase62(uniqueId).substring(0, 7);
    }
}
```

## ⚖️ When To Use / When To Avoid

**✅ When To Use:**
• High-volume applications needing URL tracking and analytics
• Social media platforms with character limits
• Marketing campaigns requiring branded short links
• Mobile apps where long URLs break user experience

**❌ When To Avoid:**
• Simple internal tools with < 1000 users
• When SEO and link transparency are critical
• Applications where link permanence isn't guaranteed
• When the overhead of maintaining distributed infrastructure isn't justified

## 📚 Further Reading

• [Designing Data-Intensive Applications](https