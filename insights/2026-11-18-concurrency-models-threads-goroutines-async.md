# 📌 Concurrency models: threads, goroutines, async
*November 18, 2026 · Daily Dev Insight*

## 🧠 Overview

Concurrency isn't about doing things simultaneously—it's about managing multiple tasks in overlapping time periods. The model you choose fundamentally shapes how your application handles I/O, scales under load, and how complex your debugging sessions become at 2 AM. Threads give you OS-level parallelism with shared memory (and all the mutex headaches). Async/await provides cooperative multitasking on a single thread, perfect for I/O-bound work. Goroutines sit in a sweet spot: lightweight coroutines multiplexed over OS threads by the Go runtime.

Here's the thing most tutorials won't tell you: **the "best" model depends less on performance benchmarks and more on your team's ability to reason about concurrency bugs**. Threads can achieve true parallelism but require defensive programming around every shared state. Async is brilliantly simple for I/O but can't leverage multiple cores without additional processes. Goroutines offer an elegant middle ground but lock you into Go's ecosystem. Your choice should match your problem domain, team expertise, and operational constraints.

## 💡 Key Concepts

- **Parallelism vs. Concurrency**: Parallelism is doing multiple things at once (multi-core). Concurrency is structuring your program to *deal* with multiple things at once. Async is concurrent but not parallel. Threads can be both.

- **Blocking vs. Non-blocking**: Traditional threads block on I/O, consuming resources while idle. Async/await and goroutines use non-blocking I/O, yielding control back to the scheduler when waiting, dramatically improving resource utilization.

- **Shared Memory vs. Message Passing**: Threads typically share memory (requires locks). Goroutines encourage "share memory by communicating" via channels. Async sidesteps this by running on a single thread—no sharing needed.

- **Cognitive Load**: Async code with proper async/await is often easiest to read (looks synchronous). Goroutines are nearly as clean. Raw threads with mutexes quickly become spaghetti code that only the original author understands.

- **Scalability Ceiling**: Node.js with async can handle 10K concurrent connections on modest hardware. Go with goroutines can spawn millions of goroutines. OS threads? You'll struggle past a few thousand before context switching kills you.

## 🐍 Python Example

```python
import asyncio
import aiohttp
import time

# Async/await model: single-threaded concurrency for I/O-bound work
# Perfect for making many HTTP requests, database queries, etc.

async def fetch_user_data(session, user_id):
    """Simulate fetching user data from an API"""
    url = f"https://jsonplaceholder.typicode.com/users/{user_id}"
    async with session.get(url) as response:
        data = await response.json()
        # Simulate some processing
        await asyncio.sleep(0.1)
        return {"id": user_id, "name": data.get("name", "Unknown")}

async def fetch_all_users(user_ids):
    """Fetch multiple users concurrently using async/await"""
    async with aiohttp.ClientSession() as session:
        # Create tasks for all fetches - they run concurrently
        tasks = [fetch_user_data(session, uid) for uid in user_ids]
        # Gather waits for all tasks to complete
        results = await asyncio.gather(*tasks)
        return results

async def main():
    start = time.time()
    user_ids = range(1, 11)  # Fetch 10 users
    
    users = await fetch_all_users(user_ids)
    
    elapsed = time.time() - start
    print(f"Fetched {len(users)} users in {elapsed:.2f}s")
    print(f"First user: {users[0]}")
    # With concurrency: ~1 second total
    # Sequential would take: ~10 seconds

if __name__ == "__main__":
    asyncio.run(main())
```

## 🟨 JavaScript Example

```javascript
// Native async/await in Node.js - the dominant concurrency model
// for JavaScript on both frontend and backend

const fetch = require('node-fetch'); // or use native fetch in Node 18+

// Simulate a database query with artificial delay
async function queryDatabase(userId) {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve({ userId, balance: Math.floor(Math.random() * 10000) });
    }, 100);
  });
}

// Fetch data from external API
async function fetchUserProfile(userId) {
  const response = await fetch(
    `https://jsonplaceholder.typicode.com/users/${userId}`
  );
  return response.json();
}

// Combine multiple async operations efficiently
async function getUserDashboard(userId) {
  try {
    // Promise.all runs these concurrently - doesn't wait for one to finish
    // before starting the next. Total time = slowest operation, not sum.
    const [profile, dbData] = await Promise.all([
      fetchUserProfile(userId),
      queryDatabase(userId)
    ]);
    
    return {
      name: profile.name,
      email: profile.email,
      balance: dbData.balance,
      timestamp: new Date().toISOString()
    };
  } catch (error) {
    console.error(`Failed to load dashboard for user ${userId}:`, error);
    throw error;
  }
}

// Process multiple users with controlled concurrency
async function main() {
  const userIds = [1, 2, 3, 4, 5];
  
  console.time('Processing');
  const dashboards = await Promise.all(
    userIds.map(id => getUserDashboard(id))
  );
  console.timeEnd('Processing');
  
  console.log(`Loaded ${dashboards.length} dashboards concurrently`);
}

main().catch(console.error);
```

## ⚖️ When To Use / When To Avoid

**Use Async/Await when:**
- Building I/O-heavy applications (web servers, API clients, scrapers)
- You want simple, readable code that "looks synchronous"
- Single-core performance is sufficient for your workload
- Working in JavaScript/Python ecosystems where it's idiomatic

**Use Threads when:**
- You need true CPU parallelism (video encoding, data processing)
- Interfacing with thread-based libraries or legacy code
- Fine-grained control over scheduling is required
- You have experienced engineers who understand concurrent data structures

**Use Goroutines when:**
- Building highly concurrent systems (microservices, load balancers)
- You want simplicity of async with power of threads
- Willing to adopt Go as your language
- Need to spawn thousands/millions of concurrent tasks

**Avoid these models when:**
- Your problem is fundamentally sequential (single-file processing)
- Team lacks concurrency experience (start simple, add complexity later)
- Debugging complexity outweighs performance gains

## 📚 Further Reading

- [Python asyncio documentation - Official Guide](https://docs.python.org/3/library/asyncio.html)
- [JavaScript Promises and async/await - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)
- [Concurrency is not Parallelism - Rob Pike's Classic Talk](https://go.dev/blog/waza-talk)
- [The Go Memory Model - Understanding Goroutines](https://go.dev/ref/mem)
- [Structured Concurrency - A Modern Pattern](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/)

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*