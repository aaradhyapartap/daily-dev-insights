# 📌 Database indexing strategies
*July 24, 2026 · Daily Dev Insight*

## 🧠 Overview

Database indexes are like the index at the back of a textbook—they help you find what you need without reading every single page. But here's the catch: badly designed indexes can actually slow down your database more than having no indexes at all. Every write operation (INSERT, UPDATE, DELETE) needs to update your indexes, and poorly chosen indexes consume memory while providing zero query benefit.

The art of indexing isn't about slapping an index on every column users might search. It's about understanding your query patterns, recognizing the trade-offs between read and write performance, and strategically placing indexes where they'll have maximum impact. A compound index on (user_id, created_at) is fundamentally different from two separate indexes, and knowing which to choose separates okay engineers from great ones.

Most production databases succeed or fail based on their indexing strategy. I've seen applications crawl to a halt because someone indexed every column "just in case," and I've seen 10-second queries drop to 50ms with a single well-placed index. Understanding B-trees, covering indexes, partial indexes, and index selectivity isn't academic—it's the difference between a database that scales and one that crashes at 3am.

## 💡 Key Concepts

- **Selectivity matters more than you think**: An index on a boolean column (true/false) is almost always useless. Good indexes have high cardinality—the `user_id` column is great, the `is_active` column is not.

- **Column order in compound indexes is critical**: An index on (A, B, C) can optimize queries filtering on A, or A+B, or A+B+C, but NOT queries filtering only on B or C. Leftmost prefix rule is non-negotiable.

- **Covering indexes eliminate table lookups**: If your index contains all columns in your SELECT clause, the database never needs to touch the actual table. This is a massive performance win.

- **Partial/filtered indexes save space and boost performance**: Why index deleted rows if you never query them? Partial indexes let you index only the subset of data you actually search.

- **Monitor your indexes**: Unused indexes are worse than no indexes. They slow down writes and consume memory for zero benefit. Regularly audit with database-specific tools.

## 🐍 Python Example

```python
import psycopg2
from datetime import datetime, timedelta

# Example: Optimizing a user analytics query with strategic indexing

def setup_indexes(conn):
    """
    Demonstrates different indexing strategies for a user events table
    """
    with conn.cursor() as cur:
        # Create the events table
        cur.execute("""
            CREATE TABLE IF NOT EXISTS user_events (
                id SERIAL PRIMARY KEY,
                user_id INTEGER NOT NULL,
                event_type VARCHAR(50) NOT NULL,
                created_at TIMESTAMP NOT NULL,
                metadata JSONB
            )
        """)
        
        # Strategy 1: Compound index for common query pattern
        # This index supports: WHERE user_id = X ORDER BY created_at
        cur.execute("""
            CREATE INDEX IF NOT EXISTS idx_user_events_user_time 
            ON user_events(user_id, created_at DESC)
        """)
        
        # Strategy 2: Partial index for active/recent events only
        # Saves space by only indexing last 30 days
        cur.execute("""
            CREATE INDEX IF NOT EXISTS idx_recent_events 
            ON user_events(event_type, created_at)
            WHERE created_at > NOW() - INTERVAL '30 days'
        """)
        
        # Strategy 3: Covering index to avoid table lookups
        # Includes all columns needed for dashboard query
        cur.execute("""
            CREATE INDEX IF NOT EXISTS idx_event_summary 
            ON user_events(user_id, event_type, created_at)
            INCLUDE (id)
        """)
        
        conn.commit()

def get_user_activity(conn, user_id, days=7):
    """
    Query optimized by idx_user_events_user_time
    """
    with conn.cursor() as cur:
        # EXPLAIN ANALYZE to verify index usage
        cur.execute("""
            SELECT event_type, COUNT(*) as count
            FROM user_events
            WHERE user_id = %s 
              AND created_at > NOW() - INTERVAL '%s days'
            GROUP BY event_type
        """, (user_id, days))
        return cur.fetchall()

# Usage
conn = psycopg2.connect("dbname=myapp user=postgres")
setup_indexes(conn)
results = get_user_activity(conn, user_id=12345)
print(f"User activity: {results}")
```

## 🟨 JavaScript Example

```javascript
// Using MongoDB with strategic indexing patterns
const { MongoClient } = require('mongodb');

async function setupProductIndexes(db) {
    const products = db.collection('products');
    
    // Strategy 1: Compound index for filtered searches
    // Supports queries filtering by category AND sorting by price
    await products.createIndex(
        { category: 1, price: 1 },
        { name: 'idx_category_price' }
    );
    
    // Strategy 2: Text index for search functionality
    // Enables full-text search across name and description
    await products.createIndex(
        { name: 'text', description: 'text' },
        { 
            name: 'idx_product_search',
            weights: { name: 10, description: 5 } // Boost name matches
        }
    );
    
    // Strategy 3: Partial index for in-stock items only
    // 80% of queries ignore out-of-stock items, save space
    await products.createIndex(
        { category: 1, createdAt: -1 },
        { 
            partialFilterExpression: { stock: { $gt: 0 } },
            name: 'idx_available_products'
        }
    );
    
    // Strategy 4: TTL index for automatic cleanup
    // Auto-delete expired promotional items
    await products.createIndex(
        { expiresAt: 1 },
        { expireAfterSeconds: 0, name: 'idx_ttl_cleanup' }
    );
}

async function findProducts(db, category, minPrice, maxPrice) {
    const products = db.collection('products');
    
    // This query uses idx_category_price efficiently
    return await products.find({
        category: category,
        price: { $gte: minPrice, $lte: maxPrice },
        stock: { $gt: 0 }
    })
    .sort({ price: 1 })
    .limit(20)
    .explain('executionStats'); // Always verify index usage!
}

// Usage example
(async () => {
    const client = await MongoClient.connect('mongodb://localhost:27017');
    const db = client.db('ecommerce');
    
    await setupProductIndexes(db);
    const results = await findProducts(db, 'electronics', 100, 500);
    
    console.log('Index used:', results.executionStats.executionStages);
    await client.close();
})();
```

## ⚖️ When To Use / When To Avoid

**✅ When to invest in indexes:**
- Columns frequently used in WHERE, JOIN, or ORDER BY clauses
- High-selectivity columns (many unique values)
- Read-heavy workloads where query speed matters
- Covering indexes when returning the same columns repeatedly
- Partial indexes when queries filter on consistent subsets

**❌ When to avoid or remove indexes:**
- Write-heavy tables with frequent INSERTs/UPDATEs
- Low-selectivity columns (gender, boolean flags)
- Small tables that fit in memory (< 1000 rows)
- Columns that are never used in query predicates
- Duplicate/redundant indexes that overlap

## 📚 Further Reading

- [PostgreSQL Index Types Documentation](https://www.postgresql.org/docs/current/indexes-types.html) - Comprehensive guide to B-tree, Hash, GiST, and GIN indexes
- [MongoDB Indexing Strategies Guide