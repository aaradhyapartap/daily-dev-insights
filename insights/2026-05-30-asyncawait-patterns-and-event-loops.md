# 📌 Async/Await patterns and event loops
*May 30, 2026 · Daily Dev Insight*

## 🧠 Overview

Async/await isn't just syntactic sugar—it's a fundamental shift in how we think about concurrency. While threads give us true parallelism at the cost of complexity, async/await leverages cooperative multitasking through event loops to handle I/O-bound operations efficiently. The magic happens when your code yields control during blocking operations, allowing the event loop to juggle thousands of concurrent tasks without the overhead of thread context switching.

The real power emerges when you understand that async/await is about *coordination*, not speed. A single-threaded event loop can often outperform multi-threaded code for I/O-heavy workloads because it eliminates lock contention and reduces memory overhead. However, this comes with trade-offs: CPU-bound tasks can starve the event loop, and one poorly written async function can block everything.

## 💡 Key Concepts

• **Event loops are single-threaded schedulers** that coordinate async tasks, not parallel execution engines
• **Await points are yield points** where your function surrenders control back to the event loop
• **Async functions are generators in disguise** that return promises/futures, not direct values
• **Concurrency vs Parallelism**: async/await provides concurrency (doing multiple things at once) but not necessarily parallelism (doing multiple things simultaneously)
• **Backpressure matters**: uncontrolled async task creation can overwhelm your system just like thread spawning

## 🐍 Python Example

```python
import asyncio
import aiohttp
import time
from typing import List, Dict

class APIAggregator:
    def __init__(self, max_concurrent: int = 10):
        self.semaphore = asyncio.Semaphore(max_concurrent)
        
    async def fetch_user_data(self, session: aiohttp.ClientSession, 
                             user_id: int) -> Dict:
        # Semaphore prevents overwhelming the API
        async with self.semaphore:
            async with session.get(f"https://api.example.com/users/{user_id}") as response:
                if response.status == 200:
                    return await response.json()
                return {"error": f"Status {response.status}"}
    
    async def fetch_user_posts(self, session: aiohttp.ClientSession, 
                              user_id: int) -> List[Dict]:
        async with self.semaphore:
            async with session.get(f"https://api.example.com/users/{user_id}/posts") as response:
                if response.status == 200:
                    return await response.json()
                return []
    
    async def get_user_profile(self, user_id: int) -> Dict:
        """Demonstrates concurrent async operations with proper resource management."""
        async with aiohttp.ClientSession() as session:
            # Launch both requests concurrently
            user_task = asyncio.create_task(self.fetch_user_data(session, user_id))
            posts_task = asyncio.create_task(self.fetch_user_posts(session, user_id))
            
            # Wait for both to complete
            user_data, posts = await asyncio.gather(user_task, posts_task)
            
            return {
                "user": user_data,
                "posts": posts,
                "total_posts": len(posts)
            }

# Usage example
async def main():
    aggregator = APIAggregator(max_concurrent=5)
    
    # Process multiple users concurrently
    user_ids = [1, 2, 3, 4, 5]
    profiles = await asyncio.gather(
        *[aggregator.get_user_profile(uid) for uid in user_ids]
    )
    
    for profile in profiles:
        print(f"User: {profile['user'].get('name', 'Unknown')} "
              f"Posts: {profile['total_posts']}")

# Run the event loop
# asyncio.run(main())
```

## 🟨 JavaScript Example

```javascript
const EventEmitter = require('events');

class TaskQueue extends EventEmitter {
    constructor(concurrency = 3) {
        super();
        this.concurrency = concurrency;
        this.running = 0;
        this.queue = [];
    }
    
    async add(asyncTask) {
        return new Promise((resolve, reject) => {
            this.queue.push({
                task: asyncTask,
                resolve,
                reject
            });
            this.process();
        });
    }
    
    async process() {
        if (this.running >= this.concurrency || this.queue.length === 0) {
            return;
        }
        
        this.running++;
        const { task, resolve, reject } = this.queue.shift();
        
        try {
            // Execute the async task
            const result = await task();
            resolve(result);
        } catch (error) {
            reject(error);
        } finally {
            this.running--;
            this.emit('taskComplete', this.queue.length);
            // Process next task in queue
            setImmediate(() => this.process());
        }
    }
}

// Simulated API calls with different delays
const fetchUserData = async (userId) => {
    const delay = Math.random() * 2000 + 500; // 500-2500ms
    await new Promise(resolve => setTimeout(resolve, delay));
    return { id: userId, name: `User ${userId}`, delay: Math.round(delay) };
};

const batchProcessUsers = async (userIds) => {
    const queue = new TaskQueue(3); // Max 3 concurrent requests
    
    // Monitor queue progress
    queue.on('taskComplete', (remaining) => {
        console.log(`Tasks remaining: ${remaining}`);
    });
    
    console.time('Batch Processing');
    
    // Add all tasks to queue - they'll be processed with controlled concurrency
    const results = await Promise.all(
        userIds.map(id => 
            queue.add(() => fetchUserData(id))
        )
    );
    
    console.timeEnd('Batch Processing');
    return results;
};

// Usage
// batchProcessUsers([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
//     .then(results => console.log('Results:', results))
//     .catch(console.error);
```

## ⚖️ When To Use / When To Avoid

**✅ Use async/await when:**
• Handling multiple I/O operations (API calls, file reads, database queries)
• Building web servers or real-time applications
• You need to coordinate many concurrent tasks efficiently
• Working with streaming data or event-driven architectures

**❌ Avoid async/await when:**
• CPU-intensive computations dominate your workload
• Simple, sequential operations with no I/O waiting
• You're in a performance-critical tight loop
• The complexity overhead outweighs the concurrency benefits

## 📚 Further Reading

• [MDN Async Functions Guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) - Comprehensive JavaScript async/await reference
• [Python asyncio Documentation](https://docs.python.org/3/library/asyncio.html) - Official Python async programming guide
• [Event Loop Visualization](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/) - Node.js event loop deep dive
• [Async Patterns Anti-patterns](https://blog.stephencleary.com/2013/11/there-is-no-thread.html) - Stephen Cleary's classic async misconceptions article
• [High Performance Browser Networking](https://hpbn.co/building-blocks-of-tcp/) - Understanding the I/O foundations that make async valuable

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*