# 📌 Database indexing strategies
*June 04, 2026 · Daily Dev Insight*

## 🧠 Overview

Database indexing is like creating a smart filing system for your data warehouse. While many developers understand that indexes speed up queries, the real art lies in choosing the right indexing strategy for your specific access patterns. A poorly designed index strategy can actually hurt performance more than having no indexes at all, consuming precious memory and slowing down writes while providing minimal query benefits.

The key insight most engineers miss is that indexing is fundamentally about trading space and write performance for read speed. Modern applications often have complex query patterns that require sophisticated indexing strategies beyond simple single-column indexes. Understanding when to use composite indexes, partial indexes, and covering indexes can transform a sluggish application into a lightning-fast user experience.

## 💡 Key Concepts

• **Composite indexes** order matters — `(user_id, created_at)` is different from `(created_at, user_id)` and serves different query patterns
• **Selectivity is king** — indexes on high-cardinality columns (like email) are more effective than low-cardinality ones (like status)
• **Covering indexes** include all columns needed for a query, eliminating the need to access the actual table rows
• **Partial indexes** with WHERE clauses can dramatically reduce index size while serving specific query patterns
• **Write amplification** increases with every index — each INSERT/UPDATE must maintain all relevant indexes

## 🐍 Python Example

```python
import sqlite3
from datetime import datetime, timedelta
import time

# Example: User activity tracking with strategic indexing
def setup_database_with_indexes():
    conn = sqlite3.connect(':memory:')
    cursor = conn.cursor()
    
    # Create table with typical user activity data
    cursor.execute("""
        CREATE TABLE user_activities (
            id INTEGER PRIMARY KEY,
            user_id INTEGER NOT NULL,
            activity_type VARCHAR(50) NOT NULL,
            created_at TIMESTAMP NOT NULL,
            is_premium BOOLEAN NOT NULL DEFAULT 0,
            metadata TEXT
        )
    """)
    
    # Strategic indexing based on common query patterns
    
    # 1. Composite index for user timeline queries (most selective first)
    cursor.execute("""
        CREATE INDEX idx_user_timeline 
        ON user_activities(user_id, created_at DESC)
    """)
    
    # 2. Partial index for premium user analytics (reduces index size)
    cursor.execute("""
        CREATE INDEX idx_premium_activities 
        ON user_activities(activity_type, created_at) 
        WHERE is_premium = 1
    """)
    
    # 3. Covering index for dashboard queries (includes all needed columns)
    cursor.execute("""
        CREATE INDEX idx_recent_summary 
        ON user_activities(created_at DESC, user_id, activity_type) 
        WHERE created_at > datetime('now', '-7 days')
    """)
    
    return conn

def benchmark_queries(conn):
    cursor = conn.cursor()
    
    # Insert sample data
    for i in range(10000):
        cursor.execute("""
            INSERT INTO user_activities (user_id, activity_type, created_at, is_premium)
            VALUES (?, ?, ?, ?)
        """, (i % 100, f"action_{i % 5}", datetime.now() - timedelta(days=i % 30), i % 10 == 0))
    
    # Query that benefits from composite index
    start_time = time.time()
    cursor.execute("""
        SELECT activity_type, created_at 
        FROM user_activities 
        WHERE user_id = 42 
        ORDER BY created_at DESC 
        LIMIT 10
    """)
    results = cursor.fetchall()
    print(f"User timeline query: {time.time() - start_time:.4f}s")
    
    conn.commit()
    return conn

# Usage
db = setup_database_with_indexes()
benchmark_queries(db)
```

## 🟨 JavaScript Example

```javascript
// MongoDB indexing strategies with Mongoose
const mongoose = require('mongoose');

// Schema with strategic indexing for e-commerce orders
const orderSchema = new mongoose.Schema({
  userId: { type: mongoose.Schema.Types.ObjectId, required: true },
  status: { type: String, enum: ['pending', 'processing', 'shipped', 'delivered'], required: true },
  total: { type: Number, required: true },
  items: [{
    productId: mongoose.Schema.Types.ObjectId,
    quantity: Number,
    price: Number
  }],
  createdAt: { type: Date, default: Date.now },
  shippedAt: { type: Date }
});

// Compound index for user order history (most selective field first)
orderSchema.index({ userId: 1, createdAt: -1 });

// Partial index for active orders only (reduces index size by ~80%)
orderSchema.index(
  { status: 1, createdAt: -1 }, 
  { partialFilterExpression: { status: { $in: ['pending', 'processing'] } } }
);

// Sparse index for shipped orders (only indexes documents with shippedAt)
orderSchema.index({ shippedAt: -1 }, { sparse: true });

// Text index for order search functionality
orderSchema.index({ 'items.productId': 1, status: 1 });

const Order = mongoose.model('Order', orderSchema);

class OrderService {
  // Optimized query using compound index
  async getUserOrders(userId, limit = 20) {
    return await Order.find({ userId })
      .sort({ createdAt: -1 })  // Uses userId + createdAt index
      .limit(limit)
      .lean();  // Faster queries when you don't need Mongoose docs
  }
  
  // Query optimized by partial index
  async getActiveOrders() {
    return await Order.find({ 
      status: { $in: ['pending', 'processing'] } 
    })
    .sort({ createdAt: -1 })
    .hint({ status: 1, createdAt: -1 });  // Force index usage
  }
  
  // Aggregation pipeline that benefits from multiple indexes
  async getOrderStats(userId) {
    return await Order.aggregate([
      { $match: { userId: new mongoose.Types.ObjectId(userId) } },  // Uses userId index
      {
        $group: {
          _id: '$status',
          count: { $sum: 1 },
          totalValue: { $sum: '$total' }
        }
      }
    ]);
  }
}

module.exports = { Order, OrderService };
```

## ⚖️ When To Use / When To Avoid

**Use strategic indexing when:**
• You have predictable, frequent query patterns (user dashboards, search features)
• Read-heavy workloads where query speed matters more than write speed
• Large datasets where full table scans become prohibitively expensive
• You can measure and monitor index effectiveness with query analysis tools

**Avoid over-indexing when:**
• Write-heavy workloads (logging, real-time analytics) where insert speed is critical
• Tables with frequent schema changes that would require constant index maintenance
• Very small tables (< 1000 rows) where indexes add overhead without benefit
• Uncertain query patterns — better to profile first, then add targeted indexes

## 📚 Further Reading

• [PostgreSQL Index Types and Performance](https://www.postgresql.org/docs/current/indexes-types.html) — Comprehensive guide to B-tree, Hash, GiST, and other index types
• [MongoDB Indexing Strategies](https://docs.mongodb.com/manual/indexes/) — Detailed documentation on compound, partial, and text indexes
• [MySQL Query Optimization Guide](https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html) — Best practices for index design and query performance
• [SQLite Query Planning](https://sqlite.org/queryplanner.html) — Understanding how SQLite's query planner uses indexes
• [Use the Index, Luke!](https://use-the-index-luke.com/) — Practical guide to database performance for developers

---