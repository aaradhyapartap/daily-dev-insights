# 📌 Distributed systems: CAP theorem
*October 15, 2026 · Daily Dev Insight*

## 🧠 Overview

The CAP theorem isn't just academic theory—it's the reality check every distributed systems engineer faces when their architecture hits production. Formulated by Eric Brewer in 2000, CAP states that any distributed data store can only guarantee two out of three properties simultaneously: **Consistency** (all nodes see the same data), **Availability** (every request gets a response), and **Partition tolerance** (the system works despite network failures).

Here's the kicker: since network partitions are inevitable in distributed systems (networks *will* fail), you're really choosing between consistency and availability. This isn't a one-time architectural decision—it's a trade-off you'll make repeatedly across different parts of your system. Your user-facing API might prioritize availability while your financial transaction system demands consistency.

Understanding CAP helps you stop chasing the impossible and start making informed trade-offs. MongoDB chose CP. Cassandra chose AP. PostgreSQL clusters? That depends on your configuration. The theorem doesn't tell you what to choose—it tells you what you're sacrificing when you make that choice.

## 💡 Key Concepts

- **Network partitions are unavoidable**: In real-world distributed systems, nodes will lose communication. CAP forces you to decide what happens when that occurs—do you sacrifice consistency or availability?

- **Consistency models exist on a spectrum**: "Consistency" in CAP means linearizability (strong consistency), but real systems implement various weaker models like eventual consistency, causal consistency, or session consistency based on their needs.

- **The choice isn't binary**: Modern systems like Cosmos DB and Cassandra let you tune consistency levels per-operation, trading latency and availability dynamically based on business requirements.

- **Most systems are actually CP or AP, not CA**: A truly "CA" system can't tolerate partitions, making it not truly distributed. If you need CA guarantees, you probably want a single-node database with replication for failover, not a distributed system.

- **CAP informs, not dictates**: Use CAP as a mental model for trade-offs, but also consider PACELC theorem (if Partition, choose A or C, Else choose Latency or Consistency) for a more complete picture.

## 🐍 Python Example

```python
import time
from dataclasses import dataclass
from typing import Dict, Optional
from enum import Enum

class ConsistencyMode(Enum):
    STRONG = "strong"  # CP: Wait for all replicas
    EVENTUAL = "eventual"  # AP: Accept stale reads

@dataclass
class DataNode:
    """Simulates a distributed data store node"""
    node_id: int
    data: Dict[str, str]
    is_available: bool = True
    last_sync: float = time.time()

class DistributedStore:
    """Demonstrates CAP trade-offs in a key-value store"""
    
    def __init__(self, num_nodes: int = 3):
        self.nodes = [DataNode(i, {}) for i in range(num_nodes)]
        self.replication_lag = 0.5  # Simulated network delay
    
    def write(self, key: str, value: str, mode: ConsistencyMode) -> bool:
        """Write with different consistency guarantees"""
        if mode == ConsistencyMode.STRONG:
            # CP: Write to all available nodes or fail
            available_nodes = [n for n in self.nodes if n.is_available]
            if len(available_nodes) < len(self.nodes):
                return False  # Sacrifice availability for consistency
            
            for node in self.nodes:
                node.data[key] = value
                node.last_sync = time.time()
            return True
        
        else:  # EVENTUAL
            # AP: Write to any available node
            for node in self.nodes:
                if node.is_available:
                    node.data[key] = value
                    return True  # Sacrifice consistency for availability
            return False
    
    def read(self, key: str, mode: ConsistencyMode) -> Optional[str]:
        """Read with different consistency guarantees"""
        if mode == ConsistencyMode.STRONG:
            # CP: Read from majority or fail
            available_nodes = [n for n in self.nodes if n.is_available]
            if len(available_nodes) < (len(self.nodes) // 2 + 1):
                return None  # Can't guarantee consistency
            
            # Return most recent value from available nodes
            return available_nodes[0].data.get(key)
        
        else:  # EVENTUAL
            # AP: Read from any available node (might be stale)
            for node in self.nodes:
                if node.is_available and key in node.data:
                    return node.data[key]
            return None

# Usage example
store = DistributedStore(num_nodes=3)

# Simulate network partition
store.nodes[2].is_available = False

# Strong consistency write will fail during partition
success = store.write("user:123", "Alice", ConsistencyMode.STRONG)
print(f"CP write during partition: {success}")  # False

# Eventual consistency write succeeds
success = store.write("user:123", "Alice", ConsistencyMode.EVENTUAL)
print(f"AP write during partition: {success}")  # True
```

## 🟨 JavaScript Example

```javascript
class CAPDistributedCache {
  constructor(nodes = 3) {
    this.nodes = Array.from({ length: nodes }, (_, i) => ({
      id: i,
      data: new Map(),
      isAvailable: true,
      version: 0
    }));
    this.writeQuorum = Math.floor(nodes / 2) + 1;
  }

  /**
   * CP mode: Write to majority (quorum) or fail
   */
  async writeCP(key, value) {
    const availableNodes = this.nodes.filter(n => n.isAvailable);
    
    if (availableNodes.length < this.writeQuorum) {
      throw new Error(
        'CP mode: Insufficient nodes for quorum. Sacrificing availability.'
      );
    }

    // Write to quorum nodes with version vector
    const version = Date.now();
    const writes = availableNodes
      .slice(0, this.writeQuorum)
      .map(node => {
        node.data.set(key, { value, version });
        return node.id;
      });

    return { 
      success: true, 
      nodesWritten: writes,
      consistency: 'strong' 
    };
  }

  /**
   * AP mode: Write to any available node immediately
   */
  async writeAP(key, value) {
    const availableNode = this.nodes.find(n => n.isAvailable);
    
    if (!availableNode) {
      throw new Error('AP mode: No nodes available.');
    }

    // Write immediately, replicate asynchronously
    const version = Date.now();
    availableNode.data.set(key, { value, version });

    // Simulate async replication (eventual consistency)
    setTimeout(() => this.replicateToOtherNodes(key, value, version), 100);

    return { 
      success: true, 
      nodesWritten: [availableNode.id],
      consistency: 'eventual' 
    };
  }

  replicateToOtherNodes(key, value, version) {
    this.nodes.forEach(node => {
      if (node.isAvailable) {
        const existing = node.data.get(key);
        if (!existing || existing.version < version) {
          node.data.set(key, { value, version });
        }
      }
    });
  }

  simulatePartition(nodeId) {
    this.nodes[nodeId].isAvailable = false;
    console.log(`⚠️  Network partition: Node ${nodeId} unreachable`);
  }
}

// Demonstration
const cache = new CAPDistributedCache(3);

(async () => {
  try {
    // Normal operation: CP write succeeds