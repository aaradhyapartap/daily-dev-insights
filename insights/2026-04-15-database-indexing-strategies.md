# 📌 Database indexing strategies
*April 15, 2026 · Daily Dev Insight*

## 🧠 Overview

Database indexes are like the table of contents in a book—they help you find what you're looking for without scanning every page. But unlike that simple analogy, database indexing is a nuanced art that can make or break your application's performance. The difference between a well-indexed query returning in 5ms versus a full table scan taking 30 seconds isn't just about user experience; it's about the fundamental scalability of your system.

Most developers treat indexing as an afterthought, slapping indexes on slow queries when performance issues arise. This reactive approach leads to over-indexing (slowing down writes) or under-indexing (crushing read performance). The sweet spot lies in understanding your data access patterns, query frequency, and the trade-offs between read and write performance. Modern applications need a strategic approach that considers composite indexes, partial indexes, and even specialized index types for different use cases.

## 💡 Key Concepts

• **B-tree vs Hash indexes**: B-trees excel at range queries and sorting, while hash indexes are lightning-fast for exact matches but useless for ranges or ordering
• **Composite index column order**: The leftmost columns in a composite index matter most—`(user_id, created_at)` can optimize queries filtering on `user_id` alone, but not `created_at` alone  
• **Index selectivity**: High-cardinality columns (like UUIDs) make better index candidates than low-cardinality ones (like boolean flags or status enums)
• **Write penalty**: Every index adds overhead to INSERT, UPDATE, and DELETE operations—more indexes mean slower writes and larger storage footprint
• **Covering indexes**: Including frequently-selected columns in your index can eliminate table lookups entirely, turning random I/O into sequential reads

## 🐍 Python Example

```python
import sqlite3
import time
import random
from contextlib import contextmanager

class IndexingDemo:
    def __init__(self, db_path=":memory:"):
        self.db_path = db_path
        self.setup_database()
    
    def setup_database(self):
        """Create tables and populate with test data"""
        with self.get_connection() as conn:
            # Create users table without indexes initially
            conn.execute("""
                CREATE TABLE users (
                    id INTEGER PRIMARY KEY,
                    email TEXT NOT NULL,
                    department TEXT,
                    created_at TIMESTAMP,
                    salary INTEGER
                )
            """)
            
            # Insert 100k test records
            test_data = [
                (f"user{i}@company.com", 
                 random.choice(['Engineering', 'Sales', 'Marketing', 'HR']),
                 f"2024-{random.randint(1,12):02d}-{random.randint(1,28):02d}",
                 random.randint(50000, 200000))
                for i in range(100000)
            ]
            
            conn.executemany(
                "INSERT INTO users (email, department, created_at, salary) VALUES (?, ?, ?, ?)",
                test_data
            )
    
    @contextmanager
    def get_connection(self):
        conn = sqlite3.connect(self.db_path)
        try:
            yield conn
            conn.commit()
        finally:
            conn.close()
    
    def benchmark_query(self, query, params=None):
        """Benchmark a query execution time"""
        with self.get_connection() as conn:
            start = time.time()
            cursor = conn.execute(query, params or [])
            results = cursor.fetchall()
            end = time.time()
            return len(results), (end - start) * 1000  # Return count and ms
    
    def demonstrate_indexing_impact(self):
        """Show before/after performance with strategic indexing"""
        # Query without index - slow!
        count, time_ms = self.benchmark_query(
            "SELECT * FROM users WHERE department = 'Engineering' AND salary > 120000"
        )
        print(f"Without index: {count} rows in {time_ms:.2f}ms")
        
        # Add composite index - order matters!
        with self.get_connection() as conn:
            conn.execute("CREATE INDEX idx_dept_salary ON users(department, salary)")
        
        count, time_ms = self.benchmark_query(
            "SELECT * FROM users WHERE department = 'Engineering' AND salary > 120000"
        )
        print(f"With composite index: {count} rows in {time_ms:.2f}ms")
        
        # Covering index - include commonly selected columns
        with self.get_connection() as conn:
            conn.execute("DROP INDEX idx_dept_salary")
            conn.execute("""
                CREATE INDEX idx_covering ON users(department, salary) 
                INCLUDE (email, created_at)
            """.replace("INCLUDE", ""))  # SQLite doesn't support INCLUDE, but concept applies
        
        count, time_ms = self.benchmark_query(
            "SELECT email, created_at FROM users WHERE department = 'Engineering' AND salary > 120000"
        )
        print(f"With covering strategy: {count} rows in {time_ms:.2f}ms")

# Usage
demo = IndexingDemo()
demo.demonstrate_indexing_impact()
```

## 🟨 JavaScript Example

```javascript
const { Pool } = require('pg');

class PostgreSQLIndexingStrategy {
    constructor() {
        this.pool = new Pool({
            user: 'postgres',
            host: 'localhost',
            database: 'performance_test',
            password: 'password',
            port: 5432,
        });
    }

    async setupTestEnvironment() {
        const client = await this.pool.connect();
        try {
            // Create orders table for e-commerce scenario
            await client.query(`
                CREATE TABLE IF NOT EXISTS orders (
                    id SERIAL PRIMARY KEY,
                    user_id INTEGER NOT NULL,
                    product_id INTEGER NOT NULL,
                    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    status VARCHAR(20) DEFAULT 'pending',
                    amount DECIMAL(10,2),
                    shipping_country VARCHAR(3)
                )
            `);

            // Generate realistic test data
            const batchSize = 10000;
            for (let batch = 0; batch < 5; batch++) {
                const values = [];
                for (let i = 0; i < batchSize; i++) {
                    const userId = Math.floor(Math.random() * 10000) + 1;
                    const productId = Math.floor(Math.random() * 1000) + 1;
                    const amount = (Math.random() * 500 + 10).toFixed(2);
                    const countries = ['USA', 'CAN', 'GBR', 'DEU', 'FRA'];
                    const country = countries[Math.floor(Math.random() * countries.length)];
                    const statuses = ['pending', 'shipped', 'delivered', 'cancelled'];
                    const status = statuses[Math.floor(Math.random() * statuses.length)];
                    
                    values.push(`(${userId}, ${productId}, CURRENT_TIMESTAMP - INTERVAL '${Math.floor(Math.random() * 365)} days', '${status}', ${amount}, '${country}')`);
                }
                
                await client.query(`
                    INSERT INTO orders (user_id, product_id, order_date, status, amount, shipping_country) 
                    VALUES ${values.join(', ')}
                `);
            }
            console.log('Test data created successfully');
        } finally {
            client.release();
        }
    }

    async benchmarkQuery(query, params = []) {
        const client = await this.pool.connect();
        try {
            const start = Date.now();
            const result = await client.query(query, params);
            const duration = Date.now() - start;
            return { count: result.rows.length, duration };
        } finally {
            client.release();
        }
    }