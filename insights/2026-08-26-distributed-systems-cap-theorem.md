# 📌 Distributed systems: CAP theorem
*August 26, 2026 · Daily Dev Insight*

## 🧠 Overview

The CAP theorem isn't just academic theory—it's the fundamental trade-off that keeps you up at night when designing distributed systems. Formulated by Eric Brewer in 2000, it states that any distributed data store can only guarantee two of three properties: **Consistency** (all nodes see the same data), **Availability** (every request gets a response), and **Partition tolerance** (the system works despite network failures). The kicker? In real-world networks, partitions *will* happen, so you're really choosing between consistency and availability.

Here's what many engineers get wrong: CAP isn't a binary choice you make once at architecture time. It's a spectrum of trade-offs you navigate continuously. When a network partition occurs, your system must decide: do I return stale data (choosing availability) or refuse to respond (choosing consistency)? Modern systems often employ tunable consistency, letting you dial these knobs per-operation. Understanding CAP means knowing *when* to be consistent and *when* to be available.

The theorem's practical impact shows up everywhere: Why does DynamoDB offer eventual consistency by default? Why does your banking app sometimes show outdated balances? Why do distributed databases have dozens of configuration options? They're all wrestling with CAP trade-offs, and so will you.

## 💡 Key Concepts

- **Network partitions are inevitable**: In any distributed system spanning multiple nodes or datacenters, network failures will occur. CAP assumes partition tolerance is non-negotiable, making it a CP vs AP decision.

- **Consistency means linearizability**: When the theorem says "consistency," it means every read sees the most recent write. This is stronger than eventual consistency and often requires coordination that impacts latency.

- **Availability means every node responds**: True availability requires that every non-failing node returns a response in reasonable time, even if it can't contact other nodes to verify data freshness.

- **You can tune consistency per-operation**: Modern databases (Cassandra, DynamoDB) let you specify consistency requirements per-query, choosing different points on the CAP spectrum based on your needs.

- **CAP is about guarantees, not averages**: Your system might be 99.9% consistent and available, but the theorem is about what you can *guarantee* during the worst-case partition scenario.

## 🐍 Python Example

```python
import time
import threading
from dataclasses import dataclass
from typing import Optional

@dataclass
class DataNode:
    """Simulates a distributed data node with CAP trade-offs"""
    node_id: str
    data: dict
    is_partitioned: bool = False
    
    def read(self, key: str, consistency_level: str = "eventual") -> Optional[str]:
        """
        Read with configurable consistency
        - 'strong': Refuse reads during partition (CP choice)
        - 'eventual': Return local data even if partitioned (AP choice)
        """
        if consistency_level == "strong" and self.is_partitioned:
            # CP system: Refuse to serve stale data
            raise Exception(f"Node {self.node_id}: Cannot guarantee consistency during partition")
        
        # AP system: Return available data (might be stale)
        return self.data.get(key)
    
    def write(self, key: str, value: str, require_quorum: bool = True):
        """Write with optional quorum requirement"""
        if require_quorum and self.is_partitioned:
            raise Exception(f"Node {self.node_id}: Cannot reach quorum during partition")
        
        self.data[key] = value
        print(f"[{self.node_id}] Wrote {key}={value}")

# Simulate a distributed system
node_a = DataNode("NodeA", {"user:1": "Alice"})
node_b = DataNode("NodeB", {"user:1": "Alice"})

# Normal operation: both nodes accessible
print("=== Normal Operation ===")
print(f"NodeA read: {node_a.read('user:1', 'strong')}")

# Simulate network partition
print("\n=== Network Partition Occurs ===")
node_b.is_partitioned = True

# AP choice: eventual consistency, returns potentially stale data
print(f"NodeB (AP/eventual): {node_b.read('user:1', 'eventual')}")

# CP choice: strong consistency, refuses to serve
try:
    print(f"NodeB (CP/strong): {node_b.read('user:1', 'strong')}")
except Exception as e:
    print(f"Error: {e}")
```

## 🟨 JavaScript Example

```javascript
class DistributedCache {
  constructor(nodeId) {
    this.nodeId = nodeId;
    this.cache = new Map();
    this.isPartitioned = false;
    this.peers = [];
  }

  /**
   * Write with configurable replication strategy
   * @param {string} key 
   * @param {any} value 
   * @param {string} strategy - 'sync' (CP) or 'async' (AP)
   */
  async write(key, value, strategy = 'async') {
    this.cache.set(key, { value, timestamp: Date.now() });
    console.log(`[${this.nodeId}] Local write: ${key}=${value}`);

    if (strategy === 'sync') {
      // CP: Wait for synchronous replication (fails during partition)
      if (this.isPartitioned) {
        throw new Error(`Cannot sync replicate during partition`);
      }
      await this.replicateSync(key, value);
      console.log(`[${this.nodeId}] Sync replication complete`);
    } else {
      // AP: Fire-and-forget async replication
      this.replicateAsync(key, value);
    }
  }

  /**
   * Read with quorum option
   */
  async read(key, useQuorum = false) {
    if (useQuorum && this.isPartitioned) {
      throw new Error(`Cannot achieve read quorum during partition`);
    }

    const entry = this.cache.get(key);
    return entry ? entry.value : null;
  }

  async replicateSync(key, value) {
    // Simulate synchronous replication delay
    await new Promise(resolve => setTimeout(resolve, 50));
  }

  replicateAsync(key, value) {
    // Fire-and-forget: doesn't block on partition
    setTimeout(() => {
      if (!this.isPartitioned) {
        console.log(`[${this.nodeId}] Async replication completed`);
      }
    }, 100);
  }
}

// Demo usage
(async () => {
  const node = new DistributedCache('Node1');

  console.log('=== AP Strategy (Available during partition) ===');
  node.isPartitioned = true;
  await node.write('session:123', 'user_data', 'async'); // Succeeds
  
  console.log('\n=== CP Strategy (Consistent, unavailable during partition) ===');
  try {
    await node.write('critical:balance', '1000', 'sync'); // Fails
  } catch (err) {
    console.log(`Error: ${err.message}`);
  }
})();
```

## ⚖️ When To Use / When To Avoid

**Choose CP (Consistency + Partition Tolerance):**
- ✅ Financial transactions, inventory management
- ✅ Systems where stale data causes critical errors
- ✅ Coordination services (ZooKeeper, etcd)
- ❌ Social media feeds, recommendation systems
- ❌ Systems requiring sub-10ms latency globally

**Choose AP (Availability + Partition Tolerance):**
- ✅ Shopping carts, session storage, caching
- ✅ Analytics and monitoring data
- ✅ Content delivery and feeds
- ❌ Bank account balances, booking systems
- ❌ Situations where conflicts are expensive to resolve

**Tune per-operation:**
- ✅ Most modern applications (use strong consistency for critical writes, eventual for reads)

## 📚