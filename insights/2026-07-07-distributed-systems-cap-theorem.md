# 📌 Distributed systems: CAP theorem
*July 07, 2026 · Daily Dev Insight*

## 🧠 Overview

The CAP theorem, formulated by Eric Brewer in 2000, is one of those fundamental trade-offs in distributed systems that every engineer eventually bumps into—usually at 3 AM during an outage. It states that any distributed data store can only guarantee two out of three properties: **Consistency** (all nodes see the same data at the same time), **Availability** (every request receives a response), and **Partition tolerance** (the system continues operating despite network failures).

Here's the kicker: partition tolerance isn't really optional in real-world distributed systems. Networks *will* fail, packets *will* drop, and that rogue fiber cut *will* happen on Black Friday. So the practical choice becomes: when a network partition occurs, do you sacrifice consistency (AP systems like Cassandra, DynamoDB) or availability (CP systems like MongoDB with strong consistency, etcd)? There's no universally correct answer—it depends entirely on your use case.

The CAP theorem isn't a license to ignore consistency or availability; it's a framework for making intentional trade-offs. Modern systems often aim for "tuneable consistency," allowing you to dial up or down these guarantees per operation. Understanding CAP means understanding that perfect is the enemy of good in distributed systems, and that your architecture should reflect your actual business requirements, not theoretical ideals.

## 💡 Key Concepts

- **Consistency means linearizability**: After a write completes, all subsequent reads should return that value or a newer one. It's not eventual consistency—that's a different beast entirely.

- **Availability means every non-failing node returns a response**: Not just "the system is up," but rather that every operational node responds to requests in a reasonable time, regardless of what's happening elsewhere in the cluster.

- **Network partitions are inevitable**: The real-world internet is messy. CAP forces you to decide: would you rather serve stale data (AP) or return errors until consistency is restored (CP)?

- **Pick two, but P is mandatory**: In practice, you're choosing between CP (sacrifice availability during partitions) and AP (sacrifice consistency during partitions). Pure CA systems only work on a single node.

- **Most modern systems are "tuneable"**: Databases like Cassandra allow per-query consistency levels. You can choose AP for user profiles and CP for financial transactions in the same system.

## 🐍 Python Example

```python
import time
import random
from dataclasses import dataclass
from typing import Optional

@dataclass
class Node:
    """Simulates a distributed database node"""
    name: str
    data: dict
    is_partitioned: bool = False
    
    def write(self, key: str, value: any, consistency: str = "eventual") -> bool:
        """Write to node with consistency level"""
        if self.is_partitioned and consistency == "strong":
            # CP choice: reject writes during partition
            raise Exception(f"{self.name}: Cannot guarantee consistency during partition")
        
        # AP choice: accept writes even during partition
        self.data[key] = value
        print(f"{self.name}: Wrote {key}={value}")
        return True
    
    def read(self, key: str, consistency: str = "eventual") -> Optional[any]:
        """Read from node with consistency level"""
        if self.is_partitioned and consistency == "strong":
            # CP choice: reject reads during partition
            raise Exception(f"{self.name}: Cannot guarantee consistency during partition")
        
        # AP choice: return potentially stale data
        return self.data.get(key)

# Simulate a 3-node cluster
nodes = [Node(f"Node-{i}", {}) for i in range(3)]

# Normal operation - all nodes accessible
nodes[0].write("user:123", {"name": "Alice"}, consistency="eventual")

# Simulate network partition
nodes[1].is_partitioned = True
print("\n⚠️  Network partition detected!")

try:
    # AP approach: accepts potentially inconsistent data
    result = nodes[1].read("user:123", consistency="eventual")
    print(f"AP read result: {result}")  # May be None (stale)
    
    # CP approach: fails fast to maintain consistency
    nodes[1].read("user:123", consistency="strong")
except Exception as e:
    print(f"CP read failed: {e}")
```

## 🟨 JavaScript Example

```javascript
class DistributedCache {
  constructor(nodeId) {
    this.nodeId = nodeId;
    this.storage = new Map();
    this.isHealthy = true;
    this.peers = [];
  }

  // AP approach: Always accept writes, sync later
  async writeAP(key, value) {
    this.storage.set(key, { value, timestamp: Date.now() });
    console.log(`[${this.nodeId}] AP Write: ${key}=${value}`);
    
    // Async replication - fire and forget
    this.replicateAsync(key, value).catch(err => 
      console.log(`[${this.nodeId}] Replication failed (will retry)`)
    );
    
    return true; // Always succeeds
  }

  // CP approach: Only accept writes if quorum is reachable
  async writeCP(key, value) {
    const healthyPeers = this.peers.filter(p => p.isHealthy);
    const quorum = Math.floor((this.peers.length + 1) / 2) + 1;
    
    if (healthyPeers.length + 1 < quorum) {
      throw new Error(
        `Cannot achieve quorum (${healthyPeers.length + 1}/${quorum})`
      );
    }
    
    // Write to self
    this.storage.set(key, { value, timestamp: Date.now() });
    
    // Synchronously replicate to quorum
    await Promise.all(
      healthyPeers.slice(0, quorum - 1).map(peer => 
        peer.writeAP(key, value)
      )
    );
    
    console.log(`[${this.nodeId}] CP Write: ${key}=${value} (quorum achieved)`);
    return true;
  }

  async replicateAsync(key, value) {
    // Simulate network delay and potential failure
    await new Promise(resolve => setTimeout(resolve, 100));
    if (Math.random() > 0.8) throw new Error("Network timeout");
  }
}

// Demo: Simulate a partition scenario
const node1 = new DistributedCache("node1");
const node2 = new DistributedCache("node2");
const node3 = new DistributedCache("node3");

node1.peers = [node2, node3];

// Simulate partition
node2.isHealthy = false;

node1.writeAP("session:abc", "active"); // Succeeds (AP)

node1.writeCP("payment:123", "$100")
  .catch(err => console.log(`❌ CP write failed: ${err.message}`));
```

## ⚖️ When To Use / When To Avoid

**Choose AP (Availability + Partition Tolerance) when:**
- Stale data is acceptable (social media feeds, recommendations)
- User experience must never show errors (shopping carts, session data)
- Writes can be reconciled later (CRDTs, last-write-wins)

**Choose CP (Consistency + Partition Tolerance) when:**
- Data correctness is non-negotiable (financial transactions, inventory)
- You can tolerate temporary unavailability (configuration systems, leader election)
- Conflicting writes would cause serious problems (seat reservations, username registration)

**Avoid distributed systems entirely when:**
- A single beefy Postgres instance would suffice (most startups)
- You don't have the operational maturity to handle complexity
- Your data naturally fits on one machine

## 📚 Further Reading

- [Brewer's CAP Theorem - Original paper](https://www.cs.berkeley.edu/~brewer/cs262b-2004/PODC-keynote.pdf) - The foundational 2000 paper that started it all
- [CAP