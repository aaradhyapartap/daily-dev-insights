# 📌 Concurrency models: threads, goroutines, async
*September 29, 2026 · Daily Dev Insight*

## 🧠 Overview

Concurrency isn't about doing multiple things at once—it's about *managing* multiple things at once. The distinction matters because the three dominant concurrency models (OS threads, goroutines, and async/await) represent fundamentally different philosophies about how to structure programs that need to juggle many tasks. Threads give you true parallelism but with heavyweight context switching. Goroutines provide cheap concurrency through cooperative scheduling. Async/await offers explicit, single-threaded concurrency with minimal overhead.

Here's the kicker: most developers reach for threads because they're familiar, but threads are often overkill. A Python thread carries ~8MB of stack overhead, while a goroutine starts at 2KB and async tasks are even lighter. The real question isn't "which is fastest?" but "which model matches your problem's structure?" I/O-bound services thrive with async. CPU-bound workloads need true parallelism. Mixed workloads? That's where it gets interesting.

The landscape has shifted dramatically. Ten years ago, callbacks were hell and threads were king. Today, async/await has made cooperative concurrency ergonomic, and languages like Go have proven that you don't need OS threads for massive concurrency. Understanding these models isn't academic—it's the difference between a server handling 10K connections and one handling 10M.

## 💡 Key Concepts

- **Preemptive vs Cooperative Scheduling**: Threads are preemptively scheduled by the OS (can be interrupted anywhere), while async and goroutines use cooperative scheduling (yield control explicitly or at defined points). This makes async/goroutines lighter but requires careful handling of CPU-intensive work.

- **The M:N Threading Model**: Goroutines implement M:N scheduling—many goroutines multiplexed onto fewer OS threads. This gives you the lightweight nature of green threads with the parallelism of OS threads. Python's asyncio is 1:N (many coroutines, one thread by default).

- **Blocking vs Non-Blocking Operations**: Traditional threads can block freely—the OS just schedules another thread. Async requires all I/O to be non-blocking or you'll stall the entire event loop. This is why you can't just drop `time.sleep()` into an async function.

- **Memory and Scaling**: The concurrency model you choose determines your scaling ceiling. Thread-per-connection models hit limits around 10K connections due to memory overhead. Async models regularly handle 100K+ connections on modest hardware.

- **Composability and Cancellation**: Async/await makes cancellation explicit and composable (through promises/futures). Thread cancellation is notoriously difficult. Goroutines use context for elegant cancellation propagation across concurrent operations.

## 🐍 Python Example

```python
import asyncio
import aiohttp
import time
from concurrent.futures import ThreadPoolExecutor

# Async model: perfect for I/O-bound concurrent operations
async def fetch_async(session, url):
    """Non-blocking HTTP fetch using async/await"""
    async with session.get(url) as response:
        return await response.text()

async def async_approach(urls):
    """Fetches multiple URLs concurrently with asyncio"""
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_async(session, url) for url in urls]
        # All requests happen concurrently on a single thread
        results = await asyncio.gather(*tasks)
        return results

# Thread model: useful when you must use blocking libraries
def fetch_threaded(url):
    """Blocking fetch - must use threads for concurrency"""
    import requests
    return requests.get(url).text

def threaded_approach(urls):
    """Fetches multiple URLs using thread pool"""
    with ThreadPoolExecutor(max_workers=10) as executor:
        # Each request runs on separate OS thread
        results = list(executor.map(fetch_threaded, urls))
        return results

# Comparison benchmark
async def main():
    urls = ['https://api.github.com/users/github'] * 20
    
    # Async approach
    start = time.time()
    async_results = await async_approach(urls)
    print(f"Async: {time.time() - start:.2f}s, {len(async_results)} results")
    
    # Thread approach (for comparison)
    start = time.time()
    thread_results = threaded_approach(urls)
    print(f"Threads: {time.time() - start:.2f}s, {len(thread_results)} results")

# asyncio.run(main())  # Uncomment to run
```

## 🟨 JavaScript Example

```javascript
// JavaScript's async model with comparison to Promise patterns
const fetch = require('node-fetch');

// Modern async/await: clean and intuitive
async function fetchUserData(userId) {
    try {
        const response = await fetch(`https://api.github.com/users/${userId}`);
        const data = await response.json();
        return data.login;
    } catch (error) {
        console.error(`Failed to fetch user ${userId}:`, error.message);
        return null;
    }
}

// Concurrent pattern: Promise.all for parallel execution
async function fetchMultipleUsers(userIds) {
    // All fetches start immediately and run concurrently
    const promises = userIds.map(id => fetchUserData(id));
    
    // Wait for all to complete (fails fast on any rejection)
    const results = await Promise.all(promises);
    return results.filter(r => r !== null);
}

// Controlled concurrency: limit parallel requests
async function fetchWithConcurrencyLimit(userIds, limit = 5) {
    const results = [];
    
    // Process in batches to avoid overwhelming the server
    for (let i = 0; i < userIds.length; i += limit) {
        const batch = userIds.slice(i, i + limit);
        const batchResults = await Promise.all(
            batch.map(id => fetchUserData(id))
        );
        results.push(...batchResults);
        console.log(`Completed batch ${Math.floor(i/limit) + 1}`);
    }
    
    return results;
}

// Race pattern: useful for timeouts and redundancy
async function fetchWithTimeout(userId, timeoutMs = 5000) {
    const timeoutPromise = new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Timeout')), timeoutMs)
    );
    
    return Promise.race([
        fetchUserData(userId),
        timeoutPromise
    ]);
}

// Example usage
(async () => {
    const users = ['torvalds', 'gvanrossum', 'tj', 'sindresorhus'];
    const results = await fetchWithConcurrencyLimit(users, 2);
    console.log('Fetched users:', results);
})();
```

## ⚖️ When To Use / When To Avoid

**Use Async/Await when:**
- Building I/O-bound services (web servers, API clients, database apps)
- You need high concurrency (10K+ simultaneous operations)
- Your language has good async ecosystem support (JS, Python, Rust, C#)

**Avoid Async when:**
- Your dependencies are blocking/synchronous (you'll block the event loop)
- The problem is naturally sequential with little concurrency
- You're doing CPU-intensive work without worker threads

**Use OS Threads when:**
- You need true parallelism for CPU-bound work
- Working with blocking libraries that can't be made async
- The number of concurrent operations is relatively small (<1000)

**Avoid Threads when:**
- You need massive concurrency (memory overhead becomes prohibitive)
- Shared state complexity outweighs benefits
- Your platform has poor threading performance (Python GIL, browser JS)

**Use Goroutines when:**
- You're in Go (obviously), or evaluating languages for concurrent services
- You want threads-like simplicity with async-like performance
- You need both parallelism and massive concurrency

## 📚 Further Reading

- [Python asyncio Documentation](https://docs.python.org/3/library/asyncio.html) — Official guide to Python