# 📌 Async/Await patterns and event loops
*July 19, 2026 · Daily Dev Insight*

## 🧠 Overview

Async/await isn't just syntactic sugar—it's a fundamental shift in how we reason about concurrent operations. At its core, async/await provides a synchronous-looking interface to asynchronous code, letting you write non-blocking I/O operations without drowning in callback hell or promise chains. But here's what many developers miss: understanding the event loop beneath async/await is crucial for debugging performance issues and avoiding subtle race conditions.

The event loop is the beating heart of asynchronous programming. It's a single-threaded mechanism that continuously checks for and executes tasks from various queues. When you `await` something, you're not blocking the thread—you're yielding control back to the event loop, which can then process other tasks. This cooperative multitasking model is why Node.js can handle thousands of concurrent connections on a single thread, and why Python's asyncio can orchestrate complex I/O-bound workflows efficiently.

The common pitfall? Developers often treat async/await as magical parallelism. It's not. It's concurrency, not parallelism. Your async functions still run on a single thread (unless you explicitly use worker threads or multiprocessing). The power comes from not wasting time waiting—while one operation waits for I/O, others can progress.

## 💡 Key Concepts

- **Await points are yield points**: Every `await` keyword is where your function yields control back to the event loop. This is where context switching happens, not at arbitrary points in your code.

- **Event loop phases matter**: The event loop has distinct phases (timers, I/O callbacks, idle, poll, check, close callbacks in Node.js). Understanding these helps debug timing issues and explain "why did this run before that?"

- **Async functions always return promises**: Even if you `return 42`, an async function wraps it in a Promise. This means you can't escape the async context—it's infectious by design.

- **Blocking the event loop is the cardinal sin**: Any CPU-intensive synchronous operation (tight loops, heavy JSON parsing, crypto) will freeze all async operations until it completes. Use worker threads or external processes for heavy lifting.

- **Error handling cascades differently**: Unhandled promise rejections won't crash your program by default (though Node 15+ changed this), but they will silently fail if not properly caught with try/catch or `.catch()`.

## 🟨 JavaScript Example

```javascript
// Simulating a database query service with proper async patterns
class DatabaseService {
  constructor() {
    this.connectionPool = [];
    this.maxConnections = 5;
  }

  // Simulate getting a connection from pool
  async acquireConnection() {
    if (this.connectionPool.length < this.maxConnections) {
      // Simulate connection delay
      await new Promise(resolve => setTimeout(resolve, 100));
      return { id: Date.now() };
    }
    throw new Error('Connection pool exhausted');
  }

  // Query with timeout and retry logic
  async queryWithRetry(sql, maxRetries = 3) {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        const connection = await this.acquireConnection();
        
        // Race between query and timeout
        const result = await Promise.race([
          this.executeQuery(connection, sql),
          this.timeout(5000, `Query timeout on attempt ${attempt}`)
        ]);
        
        return result;
      } catch (error) {
        console.error(`Attempt ${attempt} failed:`, error.message);
        if (attempt === maxRetries) throw error;
        // Exponential backoff
        await new Promise(r => setTimeout(r, Math.pow(2, attempt) * 100));
      }
    }
  }

  async executeQuery(connection, sql) {
    // Simulate query execution
    await new Promise(resolve => setTimeout(resolve, 200));
    return { rows: [{ id: 1, data: 'result' }], connectionId: connection.id };
  }

  timeout(ms, message) {
    return new Promise((_, reject) => 
      setTimeout(() => reject(new Error(message)), ms)
    );
  }
}

// Usage example
(async () => {
  const db = new DatabaseService();
  try {
    const result = await db.queryWithRetry('SELECT * FROM users');
    console.log('Query succeeded:', result);
  } catch (error) {
    console.error('All retries exhausted:', error.message);
  }
})();
```

## 🐍 Python Example

```python
import asyncio
import aiohttp
from typing import List, Dict
import time

class APIAggregator:
    """Fetch data from multiple APIs concurrently with rate limiting"""
    
    def __init__(self, rate_limit: int = 5):
        self.semaphore = asyncio.Semaphore(rate_limit)
        self.results = []
    
    async def fetch_with_limit(self, session: aiohttp.ClientSession, 
                                url: str, timeout: int = 10) -> Dict:
        """Fetch URL with semaphore-based rate limiting"""
        async with self.semaphore:  # Limit concurrent requests
            try:
                async with session.get(url, timeout=timeout) as response:
                    # This await yields to event loop while waiting for I/O
                    data = await response.json()
                    return {'url': url, 'status': response.status, 'data': data}
            except asyncio.TimeoutError:
                return {'url': url, 'status': 'timeout', 'data': None}
            except Exception as e:
                return {'url': url, 'status': 'error', 'error': str(e)}
    
    async def aggregate(self, urls: List[str]) -> List[Dict]:
        """Fetch multiple URLs concurrently and aggregate results"""
        async with aiohttp.ClientSession() as session:
            # Create tasks for all URLs - they start immediately
            tasks = [self.fetch_with_limit(session, url) for url in urls]
            
            # gather() waits for ALL tasks, unlike as_completed()
            results = await asyncio.gather(*tasks, return_exceptions=True)
            
            return [r for r in results if not isinstance(r, Exception)]

# Example usage
async def main():
    urls = [
        'https://api.github.com/repos/python/cpython',
        'https://api.github.com/repos/nodejs/node',
        'https://api.github.com/repos/rust-lang/rust',
    ]
    
    aggregator = APIAggregator(rate_limit=2)  # Max 2 concurrent requests
    start = time.time()
    
    results = await aggregator.aggregate(urls)
    
    print(f"Fetched {len(results)} URLs in {time.time() - start:.2f}s")
    for result in results:
        print(f"{result['url']}: {result['status']}")

# Run the event loop
if __name__ == "__main__":
    asyncio.run(main())
```

## ⚖️ When To Use / When To Avoid

**✅ Use Async/Await When:**
- Performing I/O-bound operations (network requests, file I/O, database queries)
- Coordinating multiple independent operations that can run concurrently
- Building web servers or API clients that handle many simultaneous connections
- You need responsive UIs that can't block on long-running operations

**❌ Avoid Async/Await When:**
- Doing CPU-intensive work (use worker threads/processes instead)
- Your codebase is entirely synchronous and simple—premature async adds complexity
- You're working with libraries that don't support async (mixing sync/async is painful)
- The overhead of event loop management exceeds the I/O wait time (tiny, fast operations)

## 📚 Further Reading

- [MDN Web Docs: async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) - Comprehensive guide to JavaScript async/await syntax and behavior
- [Python asyncio Documentation](https://