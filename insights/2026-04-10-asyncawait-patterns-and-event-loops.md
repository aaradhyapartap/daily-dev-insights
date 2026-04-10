# 📌 Async/Await patterns and event loops
*April 10, 2026 · Daily Dev Insight*

## 🧠 Overview

Async/await isn't just syntactic sugar—it's a fundamental shift in how we think about concurrent programming. While traditional threading models force us to juggle locks and shared state, async/await leverages event loops to create the illusion of parallelism within a single thread. The magic happens when your program yields control during I/O operations, allowing thousands of concurrent operations without the overhead of context switching between threads.

The event loop acts as the conductor of this orchestra, managing a queue of tasks and efficiently switching between them when they're waiting for network calls, file reads, or database queries. This cooperative multitasking model shines in I/O-bound applications but requires a different mental model than traditional synchronous code. Understanding when and how the event loop yields control is crucial for writing performant async code that doesn't accidentally block the entire application.

## 💡 Key Concepts

• **Non-blocking I/O**: Async operations yield control back to the event loop during waiting periods, allowing other tasks to execute
• **Cooperative multitasking**: Unlike preemptive threading, async functions must explicitly yield control using `await`
• **Event loop scheduling**: The runtime manages a queue of ready tasks and efficiently switches between them
• **Backpressure handling**: Proper async design includes mechanisms to prevent overwhelming slower downstream services
• **Error propagation**: Exceptions in async code can behave differently, requiring careful handling to prevent silent failures

## �🐍 Python Example

```python
import asyncio
import aiohttp
import time
from typing import List, Dict

async def fetch_user_data(session: aiohttp.ClientSession, user_id: int) -> Dict:
    """Fetch user data from API with timeout and retry logic"""
    url = f"https://jsonplaceholder.typicode.com/users/{user_id}"
    
    for attempt in range(3):  # Retry up to 3 times
        try:
            async with session.get(url, timeout=aiohttp.ClientTimeout(total=5)) as response:
                if response.status == 200:
                    user_data = await response.json()
                    # Simulate additional async processing
                    await asyncio.sleep(0.1)
                    return {"id": user_id, "name": user_data.get("name"), "email": user_data.get("email")}
                else:
                    print(f"HTTP {response.status} for user {user_id}")
        except asyncio.TimeoutError:
            print(f"Timeout for user {user_id}, attempt {attempt + 1}")
            if attempt < 2:  # Don't sleep on final attempt
                await asyncio.sleep(1 * (attempt + 1))  # Exponential backoff
    
    return {"id": user_id, "error": "Failed to fetch after 3 attempts"}

async def process_users_concurrently(user_ids: List[int]) -> List[Dict]:
    """Process multiple users concurrently with connection pooling"""
    # Use connection pooling for efficiency
    connector = aiohttp.TCPConnector(limit=10, limit_per_host=5)
    
    async with aiohttp.ClientSession(connector=connector) as session:
        # Create semaphore to limit concurrent requests
        semaphore = asyncio.Semaphore(20)
        
        async def bounded_fetch(user_id: int):
            async with semaphore:  # Limit concurrent operations
                return await fetch_user_data(session, user_id)
        
        # Execute all requests concurrently
        tasks = [bounded_fetch(user_id) for user_id in user_ids]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Handle any exceptions that occurred
        processed_results = []
        for i, result in enumerate(results):
            if isinstance(result, Exception):
                processed_results.append({"id": user_ids[i], "error": str(result)})
            else:
                processed_results.append(result)
        
        return processed_results

# Usage example
async def main():
    start_time = time.time()
    user_ids = list(range(1, 21))  # Process 20 users
    
    results = await process_users_concurrently(user_ids)
    
    successful = len([r for r in results if "error" not in r])
    elapsed = time.time() - start_time
    
    print(f"Processed {successful}/{len(user_ids)} users in {elapsed:.2f}s")

# Run the async program
if __name__ == "__main__":
    asyncio.run(main())
```

## 🟨 JavaScript Example

```javascript
const https = require('https');
const { promisify } = require('util');

class AsyncUserProcessor {
    constructor(maxConcurrency = 15) {
        this.maxConcurrency = maxConcurrency;
        this.activeRequests = 0;
        this.requestQueue = [];
    }

    // Convert callback-based HTTPS to Promise
    httpRequest(url) {
        return new Promise((resolve, reject) => {
            const request = https.get(url, {
                timeout: 5000,
                headers: { 'User-Agent': 'AsyncExample/1.0' }
            }, (response) => {
                let data = '';
                
                response.on('data', chunk => data += chunk);
                response.on('end', () => {
                    try {
                        resolve(JSON.parse(data));
                    } catch (error) {
                        reject(new Error(`Invalid JSON: ${error.message}`));
                    }
                });
            });

            request.on('error', reject);
            request.on('timeout', () => {
                request.destroy();
                reject(new Error('Request timeout'));
            });
        });
    }

    // Implement backpressure control
    async executeWithBackpressure(asyncFn) {
        return new Promise((resolve, reject) => {
            const execute = async () => {
                if (this.activeRequests >= this.maxConcurrency) {
                    // Queue the request if we're at capacity
                    this.requestQueue.push(execute);
                    return;
                }

                this.activeRequests++;
                try {
                    const result = await asyncFn();
                    resolve(result);
                } catch (error) {
                    reject(error);
                } finally {
                    this.activeRequests--;
                    // Process next queued request
                    if (this.requestQueue.length > 0) {
                        const nextRequest = this.requestQueue.shift();
                        // Use setImmediate to yield to event loop
                        setImmediate(nextRequest);
                    }
                }
            };

            execute();
        });
    }

    async fetchUserWithRetry(userId, maxRetries = 3) {
        const url = `https://jsonplaceholder.typicode.com/users/${userId}`;
        
        for (let attempt = 0; attempt < maxRetries; attempt++) {
            try {
                const userData = await this.executeWithBackpressure(() => 
                    this.httpRequest(url)
                );

                // Simulate async processing with actual async work
                await new Promise(resolve => setTimeout(resolve, 50));
                
                return {
                    id: userId,
                    name: userData.name,
                    email: userData.email,
                    attempt: attempt + 1
                };
            } catch (error) {
                console.log(`Attempt ${attempt + 1} failed for user ${userId}: ${error.message}`);
                
                if (attempt < maxRetries - 1) {
                    // Exponential backoff with jitter
                    const delay = Math.min(1000 * Math.pow(2, attempt) + Math.random() * 1000, 5000);
                    await new Promise(resolve => setTimeout(resolve, delay));
                }
            }
        }
        
        return { id: userId, error: 'Failed after all retry attempts' };
    }

    async processUsers