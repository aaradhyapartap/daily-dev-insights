# 📌 Concurrency models: threads, goroutines, async
*May 02, 2026 · Daily Dev Insight*

## 🧠 Overview

Concurrency is one of those topics that separates junior developers from senior ones—not because it's inherently complex, but because choosing the wrong model can tank your application's performance or, worse, create debugging nightmares that haunt you at 3 AM. The fundamental challenge isn't just making things run simultaneously; it's managing shared state, avoiding race conditions, and scaling gracefully under load.

The three dominant models—OS threads, goroutines, and async/await—each solve concurrency differently. Threads give you true parallelism but come with heavy memory overhead and complex synchronization. Goroutines offer lightweight concurrency with excellent composability but require buying into Go's ecosystem. Async/await provides cooperative concurrency that's perfect for I/O-heavy workloads but can struggle with CPU-intensive tasks. Understanding when each model shines (and when it fails spectacularly) is crucial for building robust systems.

Your choice often comes down to your problem domain: are you building a high-throughput web service, processing massive datasets, or orchestrating complex distributed workflows? Each scenario has different concurrency sweet spots, and picking the wrong one early can force painful rewrites later.

## 💡 Key Concepts

• **Memory footprint matters**: OS threads consume 2-8MB each, goroutines use ~2KB, and async tasks share heap space—this fundamentally limits your concurrency ceiling
• **Preemptive vs cooperative scheduling**: Threads can be interrupted anywhere (potential race conditions), while async requires explicit yield points (potential blocking)
• **Error propagation patterns**: Each model handles failures differently—threads often crash siblings, goroutines panic upward, async allows granular error handling
• **Backpressure and flow control**: Understanding how each model handles overwhelming workloads prevents cascading failures in production
• **Debugging complexity**: Stack traces, race detectors, and profiling tools vary dramatically between models—factor this into your decision

## 🐍 Python Example

```python
import asyncio
import aiohttp
import time
from concurrent.futures import ThreadPoolExecutor
from typing import List

class ConcurrencyComparison:
    def __init__(self):
        self.urls = [
            "https://httpbin.org/delay/1",
            "https://httpbin.org/delay/2", 
            "https://httpbin.org/delay/1",
            "https://httpbin.org/delay/3"
        ]
    
    # Thread-based approach - true parallelism, higher memory cost
    def fetch_with_threads(self) -> List[int]:
        import requests
        
        def fetch_url(url: str) -> int:
            response = requests.get(url)
            return response.status_code
        
        with ThreadPoolExecutor(max_workers=4) as executor:
            # Threads can run truly parallel on multiple CPU cores
            results = list(executor.map(fetch_url, self.urls))
        return results
    
    # Async approach - cooperative concurrency, single-threaded
    async def fetch_with_async(self) -> List[int]:
        async def fetch_url(session: aiohttp.ClientSession, url: str) -> int:
            # Yields control during I/O operations
            async with session.get(url) as response:
                return response.status
        
        # Single event loop manages all concurrent operations
        async with aiohttp.ClientSession() as session:
            tasks = [fetch_url(session, url) for url in self.urls]
            results = await asyncio.gather(tasks)
        return results

# Benchmark both approaches
async def main():
    comparator = ConcurrencyComparison()
    
    # Time async version
    start = time.time()
    async_results = await comparator.fetch_with_async()
    async_time = time.time() - start
    
    # Time threaded version
    start = time.time()
    thread_results = comparator.fetch_with_threads()
    thread_time = time.time() - start
    
    print(f"Async: {async_time:.2f}s, Threads: {thread_time:.2f}s")
    # Async typically wins for I/O-bound tasks due to lower overhead

if __name__ == "__main__":
    asyncio.run(main())
```

## 🟨 JavaScript Example

```javascript
const https = require('https');
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

class NodeConcurrencyPatterns {
    constructor() {
        this.urls = [
            'https://httpbin.org/delay/1',
            'https://httpbin.org/delay/2', 
            'https://httpbin.org/delay/1',
            'https://httpbin.org/delay/3'
        ];
    }

    // Promise-based async - Node's bread and butter
    async fetchWithPromises() {
        const fetchUrl = (url) => {
            return new Promise((resolve, reject) => {
                const request = https.get(url, (response) => {
                    // Non-blocking I/O handled by event loop
                    resolve(response.statusCode);
                });
                request.on('error', reject);
                request.setTimeout(5000, () => reject(new Error('Timeout')));
            });
        };

        // All requests run concurrently on single thread
        const startTime = Date.now();
        try {
            const results = await Promise.all(this.urls.map(fetchUrl));
            console.log(`Promises: ${Date.now() - startTime}ms`);
            return results;
        } catch (error) {
            console.error('Promise failed:', error.message);
            throw error;
        }
    }

    // Worker threads - for CPU-intensive tasks
    async fetchWithWorkers() {
        if (!isMainThread) {
            // Worker thread code
            const { url } = workerData;
            const https = require('https');
            
            https.get(url, (response) => {
                parentPort.postMessage(response.statusCode);
            }).on('error', (err) => {
                parentPort.postMessage({ error: err.message });
            });
            return;
        }

        // Main thread orchestrates workers
        const startTime = Date.now();
        const workers = this.urls.map(url => {
            return new Promise((resolve, reject) => {
                const worker = new Worker(__filename, { workerData: { url } });
                worker.on('message', (statusCode) => {
                    worker.terminate();
                    resolve(statusCode);
                });
                worker.on('error', reject);
            });
        });

        try {
            const results = await Promise.all(workers);
            console.log(`Workers: ${Date.now() - startTime}ms`);
            return results;
        } catch (error) {
            console.error('Worker failed:', error.message);
            throw error;
        }
    }
}

// Run comparison if this is the main thread
if (isMainThread) {
    const patterns = new NodeConcurrencyPatterns();
    
    (async () => {
        await patterns.fetchWithPromises();
        await patterns.fetchWithWorkers();
    })();
}
```

## ⚖️ When To Use / When To Avoid

**Use Threads When:**
- CPU-intensive work that benefits from multiple cores
- Existing libraries aren't async-compatible
- Need true parallelism for computational tasks

**Use Async/Await When:**
- I/O-heavy applications (web servers, APIs, file processing)
- Need to handle thousands of concurrent connections
- Working with modern frameworks that embrace async patterns

**Use Goroutines When:**
- Building in Go and need lightweight concurrency
- Complex concurrent workflows with channels for communication
- High-throughput systems with predictable scaling needs

**Avoid Threads When:**
- Memory usage is constrained (mobile, embedded)
- Dealing with thousands of concurrent I/O operations
- Team lacks experience with synchronization primitives

## 📚 Further Reading

• [Python asyncio documentation](https://docs.python.org/3/library/asyncio.html) - Comprehensive guide to Python