# 📌 Database indexing strategies
*November 01, 2026 · Daily Dev Insight*

## 🧠 Overview

Database indexes are like the index in the back of a textbook—they help you find information fast without scanning every page. But here's the thing most developers learn the hard way: indexes aren't free. Every index you add speeds up reads but slows down writes, consumes disk space, and adds complexity to your query planner's decision-making process.

The art of indexing isn't just about slapping `CREATE INDEX` on every column that appears in a WHERE clause. It's about understanding your query patterns, knowing the difference between B-tree, hash, and specialized indexes, and making informed tradeoffs. A well-indexed database can handle 10x the load; a poorly indexed one will bring your application to its knees while consuming twice the storage.

The biggest mistake I see? Over-indexing on low-cardinality columns (like boolean flags) or creating redundant indexes that your query planner will never use. The second biggest? Not indexing foreign keys in high-traffic JOIN operations. Let's dive into strategies that actually matter in production systems.

## 💡 Key Concepts

- **Composite indexes matter more than single-column ones**: If you query on `(user_id, created_at)` together, create a composite index in that order. The leftmost prefix rule means you get `user_id` lookups for free, but not `created_at` alone.

- **Cardinality is king**: Indexes work best on high-cardinality columns (many unique values). Indexing `gender` with 3 possible values? Probably wasteful. Indexing `email`? Absolutely essential.

- **Covering indexes eliminate table lookups**: If your index contains all columns in your SELECT, the database never touches the main table. This is a massive performance win for read-heavy workloads.

- **Index maintenance is real cost**: Every INSERT, UPDATE, and DELETE must update all relevant indexes. In write-heavy systems, each additional index can reduce throughput by 5-15%.

- **Partial and filtered indexes are underutilized**: Why index deleted records if you never query them? Partial indexes (with WHERE clauses) give you the performance benefits while reducing storage and maintenance costs.

## 🐍 Python Example

```python
import psycopg2
from datetime import datetime, timedelta

# Example: Optimizing a slow query with strategic indexing
def setup_optimized_indexes(conn):
    cursor = conn.cursor()
    
    # BAD: Over-indexing on low-cardinality column
    # cursor.execute("CREATE INDEX idx_users_active ON users(is_active)")
    
    # GOOD: Composite index matching query pattern (leftmost prefix rule)
    cursor.execute("""
        CREATE INDEX IF NOT EXISTS idx_orders_user_created 
        ON orders(user_id, created_at DESC)
    """)
    
    # EXCELLENT: Covering index (includes all SELECT columns)
    cursor.execute("""
        CREATE INDEX IF NOT EXISTS idx_orders_covering 
        ON orders(user_id, status, created_at) 
        INCLUDE (total_amount, order_number)
    """)
    
    # SMART: Partial index for common query pattern
    cursor.execute("""
        CREATE INDEX IF NOT EXISTS idx_orders_pending 
        ON orders(created_at DESC) 
        WHERE status = 'pending'
    """)
    
    conn.commit()

def get_user_recent_orders(conn, user_id, days=30):
    cursor = conn.cursor()
    cutoff_date = datetime.now() - timedelta(days=days)
    
    # This query benefits from idx_orders_covering - no table access needed!
    cursor.execute("""
        SELECT order_number, total_amount, created_at, status
        FROM orders
        WHERE user_id = %s 
          AND created_at > %s
        ORDER BY created_at DESC
        LIMIT 100
    """, (user_id, cutoff_date))
    
    return cursor.fetchall()

# Check if indexes are actually being used
def analyze_query_plan(conn, user_id):
    cursor = conn.cursor()
    cursor.execute("""
        EXPLAIN ANALYZE
        SELECT * FROM orders 
        WHERE user_id = %s 
        ORDER BY created_at DESC
    """, (user_id,))
    
    print("Query plan:")
    for row in cursor.fetchall():
        print(row[0])
```

## 🟨 JavaScript Example

```javascript
// Using MongoDB with Mongoose - indexing strategies
const mongoose = require('mongoose');

const orderSchema = new mongoose.Schema({
  userId: { 
    type: mongoose.Schema.Types.ObjectId, 
    required: true,
    index: true  // Simple index for foreign key
  },
  orderNumber: { 
    type: String, 
    unique: true,  // Unique index prevents duplicates
    sparse: true   // Only index non-null values
  },
  status: { 
    type: String, 
    enum: ['pending', 'processing', 'completed', 'cancelled']
  },
  totalAmount: Number,
  createdAt: { type: Date, default: Date.now }
});

// Compound index for common query pattern
orderSchema.index({ userId: 1, createdAt: -1 });

// Partial index - only index active orders
orderSchema.index(
  { status: 1, createdAt: -1 },
  { 
    partialFilterExpression: { 
      status: { $in: ['pending', 'processing'] } 
    }
  }
);

// Text index for search functionality
orderSchema.index({ 
  orderNumber: 'text', 
  'items.productName': 'text' 
});

const Order = mongoose.model('Order', orderSchema);

// Analyze index usage in production
async function analyzeIndexUsage() {
  const stats = await Order.collection.stats();
  console.log('Index sizes:', stats.indexSizes);
  
  // Find queries not using indexes
  const slowQueries = await mongoose.connection.db
    .admin()
    .command({ profile: 1, slowms: 100 });
    
  console.log('Slow query log enabled');
}

// Query that leverages compound index effectively
async function getUserRecentOrders(userId, limit = 50) {
  return await Order
    .find({ userId })
    .sort({ createdAt: -1 })
    .limit(limit)
    .hint({ userId: 1, createdAt: -1 })  // Force index usage
    .lean()  // Return plain objects, faster
    .exec();
}

module.exports = { Order, analyzeIndexUsage, getUserRecentOrders };
```

## ⚖️ When To Use / When To Avoid

**✅ Use indexes when:**
- Columns frequently appear in WHERE, JOIN, or ORDER BY clauses
- Query patterns are predictable and well-defined
- Read-to-write ratio is high (10:1 or greater)
- Cardinality is high (many unique values)
- You need to enforce uniqueness constraints

**❌ Avoid or reconsider when:**
- Table has fewer than 10,000 rows (full scans are fast enough)
- Write-heavy workloads where every index adds latency
- Low-cardinality columns (booleans, small enums)
- Columns that change frequently in UPDATE operations
- You already have 5+ indexes on a table (diminishing returns)

## 📚 Further Reading

- [PostgreSQL Documentation: Indexes](https://www.postgresql.org/docs/current/indexes.html) - The definitive guide to index types and strategies
- [Use The Index, Luke!](https://use-the-index-luke.com/) - Excellent deep-dive into how indexes actually work
- [MongoDB Index Strategies](https://www.mongodb.com/docs/manual/indexes/) - NoSQL indexing patterns and best practices
- [MySQL: How to Avoid Full Table Scans](https://dev.mysql.com/doc/refman/8.0/en/table-scan-avoidance.html) - Practical optimization techniques
- [Database Indexing Explained (Video)](https://www.youtube.com/