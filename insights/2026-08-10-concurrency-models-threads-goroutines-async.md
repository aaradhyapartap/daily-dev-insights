# 📌 Concurrency models: threads, goroutines, async
*August 10, 2026 · Daily Dev Insight*

## 🧠 Overview

Concurrency is one of those topics that separates junior developers from senior ones—not because it's impossibly complex, but because choosing the *right* concurrency model requires understanding trade-offs that only come with battle scars. The three dominant models today—OS threads, goroutines (green threads), and async/await—solve the same problem (doing multiple things at once) in fundamentally different ways.

Here's the mental model that matters: **threads are about parallelism through preemption, async is about concurrency through cooperation, and goroutines are the pragmatic middle ground**. Threads let the OS interrupt your code whenever it wants, giving you true parallelism but expensive context switching. Async/await requires your code to explicitly yield control, making it lightweight but forcing you to "color" your functions (async spreads virally through your codebase). Goroutines give you thread-like simplicity with async-like performance by using a smart runtime scheduler.

The mistake I see most often? Developers picking threads because they're "traditional," then wondering why their app falls over at 10,000 concurrent operations. Or choosing async, then spending weeks refactoring perfectly good synchronous code. The model you choose dictates your application's scalability ceiling and your team's cognitive load for the next several years.

## 💡 Key Concepts

- **OS Threads are heavy but simple**: Each thread gets its own stack (1-8MB), managed by the kernel. Great for CPU-bound work, terrible when you need 50,000 concurrent connections. Context switching is expensive (~1-10μs).

- **Async/await is lightweight but viral**: Event loop + cooperative multitasking = millions of concurrent operations on one thread. The catch? Every function in your call stack must be async-aware, and one blocking call murders your entire event loop.

- **Goroutines split the difference**: User-space threads (2KB initial stack) multiplexed onto OS threads by the Go runtime. You write synchronous-looking code, the runtime handles scheduling. Sweet spot for I/O-heavy services.

- **CPU-bound vs I/O-bound determines your model**: Threads shine for CPU parallelism (video encoding, data processing). Async/goroutines excel at I/O concurrency (web servers, database queries, API calls).

- **Structured concurrency is the future**: Raw threads and callbacks are legacy patterns. Modern approaches (async/await, goroutines with contexts, Python's TaskGroups) give you cancellation, error propagation, and sane lifecycle management.

## 🐍 Python Example

```python
import asyncio
import aiohttp
import time

async def fetch_user_data(session, user_id):
    """Simulates fetching user data from an API"""
    url = f"https://jsonplaceholder.typicode.com/users/{user_id}"
    async with session.get(url) as response:
        data = await response.json()
        # Simulate some processing
        await asyncio.sleep(0.1)
        return {"id": user_id, "name": data.get("name")}

async def fetch_all_users(user_ids):
    """Concurrently fetch multiple users using async/await"""
    async with aiohttp.ClientSession() as session:
        # Create tasks for all users
        tasks = [fetch_user_data(session, uid) for uid in user_ids]
        
        # Wait for all tasks to complete
        # gather() runs them concurrently on the event loop
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Filter out any errors
        return [r for r in results if not isinstance(r, Exception)]

async def main():
    start = time.time()
    user_ids = range(1, 11)  # Fetch 10 users
    
    users = await fetch_all_users(user_ids)
    
    elapsed = time.time() - start
    print(f"Fetched {len(users)} users in {elapsed:.2f}s")
    print(f"First user: {users[0]}")
    
    # With async: ~0.2s total (concurrent)
    # With sequential requests: ~1.0s+ (10 × 0.1s each)

# Python 3.7+ style
if __name__ == "__main__":
    asyncio.run(main())
```

## 🟨 JavaScript Example

```javascript
const https = require('https');

// Callback-style request wrapper
function fetchUser(userId) {
  return new Promise((resolve, reject) => {
    const url = `https://jsonplaceholder.typicode.com/users/${userId}`;
    
    https.get(url, (res) => {
      let data = '';
      
      res.on('data', (chunk) => data += chunk);
      res.on('end', () => {
        try {
          const user = JSON.parse(data);
          // Simulate processing delay
          setTimeout(() => resolve({
            id: userId,
            name: user.name
          }), 100);
        } catch (e) {
          reject(e);
        }
      });
    }).on('error', reject);
  });
}

async function fetchAllUsersConcurrently(userIds) {
  const startTime = Date.now();
  
  // Promise.all runs all promises concurrently
  // This is the key to async concurrency in JS
  const users = await Promise.all(
    userIds.map(id => fetchUser(id))
  );
  
  const elapsed = (Date.now() - startTime) / 1000;
  console.log(`Fetched ${users.length} users in ${elapsed.toFixed(2)}s`);
  console.log(`First user:`, users[0]);
  
  return users;
}

// Execute
const userIds = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
fetchAllUsersConcurrently(userIds)
  .catch(err => console.error('Error:', err));
```

## ⚖️ When To Use / When To Avoid

**Use OS Threads when:**
- ✅ You have CPU-intensive work that benefits from multiple cores
- ✅ You're working with blocking libraries that can't be made async
- ✅ Concurrency count is low (<1000 operations)

**Use Async/Await when:**
- ✅ You're I/O bound (network requests, file operations, database queries)
- ✅ You need to handle 10,000+ concurrent operations
- ✅ Your language/ecosystem has mature async support

**Use Goroutines when:**
- ✅ You're writing in Go (obviously)
- ✅ You want simple code that scales without async "coloring"

**Avoid these models when:**
- ❌ Threads: You need massive concurrency or have limited memory
- ❌ Async: Your dependencies are blocking or you can't refactor to async
- ❌ Goroutines: You're not using Go, or you need fine-grained scheduling control

## 📚 Further Reading

- [Python asyncio Documentation](https://docs.python.org/3/library/asyncio.html) - Official guide to async/await in Python with patterns and best practices
- [JavaScript Event Loop Explained (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Event_loop) - Deep dive into how JS handles concurrency under the hood
- [Go Concurrency Patterns](https://go.dev/blog/pipelines) - Google's official blog on goroutines and channels
- [The C10k Problem](http://www.kegel.com/c10k.html) - Classic article explaining why traditional threading doesn't scale
- [Structured Concurrency](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/) - Nathaniel Smith's influential piece on modern concurrency patterns

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*