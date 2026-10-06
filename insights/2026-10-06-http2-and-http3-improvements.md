# 📌 HTTP/2 and HTTP/3 improvements
*October 06, 2026 · Daily Dev Insight*

## 🧠 Overview

The evolution from HTTP/1.1 to HTTP/2 and now HTTP/3 represents one of the most significant infrastructure upgrades in web development over the past decade. While HTTP/2 (finalized in 2015) addressed the infamous head-of-line blocking problem at the application layer through multiplexing, it still suffered from TCP-level blocking. HTTP/3 took the bold step of replacing TCP with QUIC (a UDP-based protocol developed by Google), eliminating head-of-line blocking entirely while adding built-in encryption and faster connection establishment.

As senior engineers, we need to understand that these aren't just protocol upgrades—they fundamentally change how we should think about asset optimization. The old HTTP/1.1 tricks like domain sharding, sprite sheets, and aggressive file concatenation are now anti-patterns. HTTP/2's multiplexing means multiple requests over a single connection are cheap, while HTTP/3's improved congestion control and connection migration make it ideal for mobile users switching between networks.

The practical reality in 2026 is that HTTP/3 is now supported by over 85% of browsers and most major CDNs enable it by default. If your infrastructure isn't leveraging these protocols, you're leaving significant performance gains on the table—particularly for users on high-latency or lossy networks where HTTP/3 truly shines.

## 💡 Key Concepts

- **Multiplexing without head-of-line blocking**: HTTP/2 allows multiple request/response pairs simultaneously over one connection, but TCP packet loss can still block all streams. HTTP/3's QUIC solves this by handling streams independently at the transport layer.

- **0-RTT connection resumption**: HTTP/3 can resume previous connections with zero round trips, reducing latency dramatically for repeat visitors—a game-changer for perceived performance on mobile networks.

- **Built-in encryption**: Unlike HTTP/2 which can technically run unencrypted (though browsers won't support it), HTTP/3/QUIC has TLS 1.3 baked into the protocol itself, making security non-negotiable.

- **Connection migration**: HTTP/3 connections survive network changes (WiFi to cellular) because they're identified by connection IDs rather than IP:port tuples, providing seamless user experience during network transitions.

- **Server Push deprecation**: Interestingly, server push—a headline HTTP/2 feature—has been largely deprecated in HTTP/3 due to poor adoption and complexity. Focus on resource hints instead (preload, prefetch).

## 🟨 JavaScript Example

```javascript
// Node.js HTTP/3 server using the 'quic' module with Express-like routing
import { createQuicSocket } from 'node:quic';
import { readFileSync } from 'node:fs';
import { join } from 'node:path';

// Create HTTP/3 (QUIC) server
const socket = createQuicSocket({ 
  endpoint: { port: 8443 },
  // HTTP/3 requires TLS certificates
  server: {
    key: readFileSync('./server-key.pem'),
    cert: readFileSync('./server-cert.pem'),
    alpn: 'h3' // HTTP/3 protocol identifier
  }
});

socket.on('session', (session) => {
  console.log('New HTTP/3 session established');
  
  session.on('stream', (stream) => {
    let requestData = '';
    
    stream.on('data', (chunk) => {
      requestData += chunk.toString();
    });
    
    stream.on('end', () => {
      // Parse simple HTTP/3 request
      const [requestLine] = requestData.split('\r\n');
      const [method, path] = requestLine.split(' ');
      
      console.log(`${method} ${path} via HTTP/3`);
      
      // Leverage HTTP/3's independent stream handling
      // Each resource loads without blocking others
      const response = {
        ':status': '200',
        'content-type': 'application/json',
        'alt-svc': 'h3=":8443"; ma=86400' // Advertise HTTP/3 support
      };
      
      const body = JSON.stringify({
        protocol: 'HTTP/3',
        path: path,
        features: ['0-RTT', 'multiplexing', 'no-HOL-blocking'],
        timestamp: new Date().toISOString()
      });
      
      // Send headers and body
      stream.respond(response);
      stream.end(body);
    });
  });
});

await socket.listen();
console.log('HTTP/3 server listening on port 8443');
```

## 🐍 Python Example

```python
# Python HTTP/3 client using httpx with HTTP/3 support
import httpx
import asyncio
import time

async def benchmark_protocols():
    """Compare HTTP/2 vs HTTP/3 performance for multiple requests"""
    
    # URLs to fetch - simulating a typical web page load
    urls = [
        'https://example.com/api/user',
        'https://example.com/api/posts',
        'https://example.com/assets/main.css',
        'https://example.com/assets/app.js',
        'https://example.com/images/hero.jpg',
    ]
    
    # HTTP/2 client
    async with httpx.AsyncClient(http2=True, http3=False) as client:
        start = time.perf_counter()
        
        # Fire all requests concurrently - HTTP/2 multiplexing
        tasks = [client.get(url) for url in urls]
        responses_h2 = await asyncio.gather(*tasks, return_exceptions=True)
        
        h2_time = time.perf_counter() - start
        print(f"HTTP/2 total time: {h2_time:.3f}s")
    
    # HTTP/3 client (QUIC)
    async with httpx.AsyncClient(http3=True) as client:
        start = time.perf_counter()
        
        # Same requests over HTTP/3 - benefits from QUIC improvements
        tasks = [client.get(url) for url in urls]
        responses_h3 = await asyncio.gather(*tasks, return_exceptions=True)
        
        h3_time = time.perf_counter() - start
        print(f"HTTP/3 total time: {h3_time:.3f}s")
        
        # Check protocol used
        for response in responses_h3:
            if not isinstance(response, Exception):
                print(f"Protocol: {response.http_version}")
                # Look for Alt-Svc header indicating HTTP/3 support
                if 'alt-svc' in response.headers:
                    print(f"Server advertises: {response.headers['alt-svc']}")
    
    improvement = ((h2_time - h3_time) / h2_time) * 100
    print(f"\nHTTP/3 improvement: {improvement:.1f}%")

# Run the benchmark
asyncio.run(benchmark_protocols())
```

## ⚖️ When To Use / When To Avoid

**✅ When to prioritize HTTP/3:**
- Mobile-first applications where network switching is common
- High-latency or lossy networks (international users, cellular)
- Real-time applications requiring fast connection establishment
- APIs with many concurrent small requests
- When using a modern CDN that handles HTTP/3 negotiation

**❌ When HTTP/2 is sufficient:**
- Internal enterprise networks with stable, low-latency connections
- Legacy client environments where HTTP/3 support is uncertain
- Simple request/response patterns without high concurrency
- When UDP is blocked by corporate firewalls (not uncommon)
- Embedded devices with limited QUIC stack support

## 📚 Further Reading

- [HTTP/3 explained by Daniel Stenberg](https://http3-explained.haxx.se/en/) - Comprehensive guide from the curl author
- [MDN HTTP/3 documentation](https://developer.mozilla.org/en-US/docs/Glossary/HTTP_3) - Browser compatibility