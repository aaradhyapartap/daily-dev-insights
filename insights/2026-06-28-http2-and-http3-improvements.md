# 📌 HTTP/2 and HTTP/3 improvements
*June 28, 2026 · Daily Dev Insight*

## 🧠 Overview

HTTP/2 and HTTP/3 represent the most significant evolutions of the web's communication protocol in decades, yet many developers still deploy applications optimized for HTTP/1.1 patterns. HTTP/2, now mature and widely supported, introduced multiplexing, header compression, and server push to eliminate the head-of-line blocking that plagued HTTP/1.1. HTTP/3 takes this further by replacing TCP with QUIC (built on UDP), solving TCP-level head-of-line blocking and dramatically improving performance on lossy networks—think mobile connections and Wi-Fi handoffs.

The practical implications are enormous. That elaborate domain sharding you implemented? Counterproductive in HTTP/2. CSS sprites to reduce requests? Actually slower. Connection pooling limits? Often harmful. HTTP/3 goes even further with 0-RTT connection resumption and better congestion control. The catch? Your application code rarely needs changes, but your deployment architecture, CDN configuration, and performance assumptions absolutely do.

Understanding these protocols isn't just about enabling a flag in your web server—it's about rethinking how you structure assets, handle API calls, and reason about network performance. The shift from "minimize requests" to "optimize payload and prioritization" is fundamental.

## 💡 Key Concepts

- **Multiplexing eliminates request queuing**: HTTP/2 and HTTP/3 send multiple requests/responses simultaneously over a single connection, making domain sharding and connection limits obsolete. Focus on connection reuse instead.

- **Header compression (HPACK/QPACK) saves bandwidth**: Repetitive headers in API calls get compressed intelligently. This makes frequent small requests much cheaper, changing the calculus for REST API design and GraphQL fragment strategies.

- **QUIC's UDP foundation prevents TCP head-of-line blocking**: In HTTP/2, a single lost TCP packet stalls all streams. HTTP/3's QUIC allows independent stream recovery, crucial for real-time applications and mobile users.

- **Server Push is deprecated (use 103 Early Hints instead)**: HTTP/2's server push sounded great but proved complex to implement correctly. HTTP/3 prefers 103 Early Hints to inform clients what to preload.

- **0-RTT connection resumption**: HTTP/3 can resume connections with zero round-trip time, making it transformative for microservices and API-heavy SPAs where connection overhead dominated latency.

## 🐍 Python Example

```python
import asyncio
import httpx
from httpx import AsyncClient, Limits

async def fetch_with_http2():
    """
    Demonstrate HTTP/2 multiplexing benefits by making concurrent requests
    over a single connection. Notice we're NOT creating connection pools
    per domain anymore.
    """
    # Configure connection limits for HTTP/2 efficiency
    # Higher max_connections than HTTP/1.1 would use
    limits = Limits(
        max_keepalive_connections=20,
        max_connections=100,
        keepalive_expiry=30.0
    )
    
    async with AsyncClient(
        http2=True,  # Enable HTTP/2
        limits=limits,
        timeout=30.0
    ) as client:
        # These requests multiplex over the same connection
        urls = [
            "https://api.github.com/users/octocat",
            "https://api.github.com/users/octocat/repos",
            "https://api.github.com/users/octocat/followers",
            "https://api.github.com/users/octocat/following",
        ]
        
        # Fire all requests concurrently - HTTP/2 handles the multiplexing
        tasks = [client.get(url) for url in urls]
        responses = await asyncio.gather(*tasks)
        
        for i, response in enumerate(responses):
            # Check which protocol was actually used
            protocol = response.http_version
            print(f"Request {i+1}: HTTP/{protocol}, "
                  f"Status: {response.status_code}, "
                  f"Size: {len(response.content)} bytes")
        
        return responses

# Run the async function
if __name__ == "__main__":
    asyncio.run(fetch_with_http2())
```

## 🟨 JavaScript Example

```javascript
// Node.js HTTP/2 client example with connection reuse
import http2 from 'http2';
import { promisify } from 'util';

class HTTP2Client {
  constructor() {
    this.sessions = new Map();
  }

  /**
   * Get or create an HTTP/2 session for a given origin.
   * Reusing sessions is critical for HTTP/2 performance.
   */
  getSession(origin) {
    if (!this.sessions.has(origin)) {
      const session = http2.connect(origin);
      
      session.on('error', (err) => {
        console.error(`Session error for ${origin}:`, err);
        this.sessions.delete(origin);
      });
      
      this.sessions.set(origin, session);
    }
    
    return this.sessions.get(origin);
  }

  /**
   * Make concurrent requests using HTTP/2 multiplexing
   */
  async fetchMultiple(urls) {
    const requests = urls.map(url => {
      const urlObj = new URL(url);
      const origin = `${urlObj.protocol}//${urlObj.host}`;
      const session = this.getSession(origin);
      
      return new Promise((resolve, reject) => {
        const req = session.request({
          ':path': urlObj.pathname + urlObj.search,
          ':method': 'GET',
        });
        
        let data = '';
        req.setEncoding('utf8');
        req.on('data', chunk => data += chunk);
        req.on('end', () => resolve({ url, data, protocol: 'HTTP/2' }));
        req.on('error', reject);
        req.end();
      });
    });
    
    return Promise.all(requests);
  }

  close() {
    this.sessions.forEach(session => session.close());
    this.sessions.clear();
  }
}

// Usage example
const client = new HTTP2Client();
const urls = [
  'https://httpbin.org/delay/1',
  'https://httpbin.org/delay/1',
  'https://httpbin.org/delay/1',
];

client.fetchMultiple(urls)
  .then(results => {
    results.forEach(({ url, protocol }) => {
      console.log(`Fetched ${url} via ${protocol}`);
    });
    client.close();
  });
```

## ⚖️ When To Use / When To Avoid

**When HTTP/2/3 Shines:**
- API-heavy SPAs with many concurrent requests to the same origin
- Mobile applications where connection setup overhead is significant
- Microservices mesh communication (especially HTTP/3 for internal networking)
- Streaming applications requiring multiple simultaneous data flows

**When to Stick with HTTP/1.1 (or be cautious):**
- Legacy corporate networks with deep packet inspection that blocks HTTP/2
- Extremely resource-constrained embedded devices (HPACK/QPACK overhead)
- Simple static sites with minimal requests where complexity isn't justified
- Debugging scenarios where HTTP/1.1's plain-text nature simplifies troubleshooting

## 📚 Further Reading

- [HTTP/2 Specification (RFC 7540)](https://datatracker.ietf.org/doc/html/rfc7540) - The official HTTP/2 standard with deep technical details
- [HTTP/3 Explained by Daniel Stenberg](https://http3-explained.haxx.se/) - Comprehensive guide by curl's creator on HTTP/3 and QUIC
- [MDN Web Docs: HTTP/2](https://developer.mozilla.org/en-US/docs/Glossary/HTTP_2) - Practical overview with browser compatibility details
- [Cloudflare: HTTP/3 Deep Dive](https://blog.cloudflare.com/http3-the-past-present-and-future/) - Real-world performance data and deployment insights
- [