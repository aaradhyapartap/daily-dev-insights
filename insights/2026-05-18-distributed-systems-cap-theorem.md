# 📌 Distributed systems: CAP theorem
*May 18, 2026 · Daily Dev Insight*

## 🧠 Overview

The CAP theorem isn't just academic theory—it's the harsh reality check every distributed systems engineer faces. Formalized by Eric Brewer in 2000, it states that any distributed system can only guarantee two out of three properties: **Consistency** (all nodes see the same data simultaneously), **Availability** (system remains operational), and **Partition tolerance** (system continues despite network failures). The cruel twist? In real-world networks, partitions *will* happen, forcing you to choose between consistency and availability.

Understanding CAP isn't about memorizing definitions—it's about making informed architectural decisions. When your microservices can't talk to each other due to a network split, do you serve potentially stale data (choosing availability) or reject requests until consistency is restored? This trade-off shapes everything from your database choice to your API design. Modern systems often embrace "eventual consistency," accepting temporary inconsistencies for the sake of availability and user experience.

## 💡 Key Concepts

• **Consistency vs Availability is the real choice** - Since network partitions are inevitable in distributed systems, you're essentially choosing between CP (consistent but potentially unavailable) or AP (available but potentially inconsistent)

• **Different consistency models exist** - From strong consistency (immediate) to eventual consistency (delayed), understanding the spectrum helps you make nuanced decisions rather than binary ones

• **CAP applies at the operation level** - You don't have to make the same trade-off for your entire system; different operations can prioritize different guarantees based on business requirements

• **Modern systems use hybrid approaches** - Technologies like CRDTs (Conflict-free Replicated Data Types) and consensus algorithms help minimize the practical impact of CAP constraints

• **Observability is crucial** - When you can't guarantee perfect consistency, monitoring and alerting become essential for understanding your system's actual state

## 🐍 Python Example

```python
import asyncio
import random
from dataclasses import dataclass
from typing import Dict, Optional
from enum import Enum

class ConsistencyMode(Enum):
    STRONG = "strong"      # CP - Wait for all nodes
    EVENTUAL = "eventual"  # AP - Accept inconsistency

@dataclass
class Node:
    id: str
    data: Dict[str, str]
    is_available: bool = True
    
class DistributedStore:
    def __init__(self, nodes: list[Node], consistency: ConsistencyMode):
        self.nodes = nodes
        self.consistency = consistency
    
    async def write(self, key: str, value: str) -> bool:
        """Write data according to CAP theorem constraints"""
        available_nodes = [n for n in self.nodes if n.is_available]
        
        if self.consistency == ConsistencyMode.STRONG:
            # CP: Require all nodes to be available for consistency
            if len(available_nodes) < len(self.nodes):
                print(f"❌ Write failed: Not all nodes available for strong consistency")
                return False
            
            # Write to all nodes synchronously
            for node in available_nodes:
                node.data[key] = value
            print(f"✅ Strong consistency write: {key}={value}")
            return True
            
        else:  # EVENTUAL consistency
            # AP: Write to available nodes, accept inconsistency
            if not available_nodes:
                print(f"❌ Write failed: No nodes available")
                return False
                
            for node in available_nodes:
                node.data[key] = value
            print(f"✅ Eventual consistency write: {key}={value} to {len(available_nodes)}/{len(self.nodes)} nodes")
            return True
    
    async def read(self, key: str) -> Optional[str]:
        """Read data with different consistency guarantees"""
        available_nodes = [n for n in self.nodes if n.is_available]
        
        if not available_nodes:
            return None
            
        # Return from first available node (may be inconsistent)
        return available_nodes[0].data.get(key)

# Demo the CAP theorem in action
async def demo_cap_theorem():
    nodes = [Node(f"node-{i}", {}) for i in range(3)]
    
    # Test with strong consistency (CP)
    cp_store = DistributedStore(nodes, ConsistencyMode.STRONG)
    await cp_store.write("user:123", "John")
    
    # Simulate network partition
    nodes[2].is_available = False
    print(f"\n🔥 Network partition! Node-2 unavailable")
    
    # This will fail due to partition
    await cp_store.write("user:456", "Jane")
    
    # Test with eventual consistency (AP)
    print(f"\n🔄 Switching to eventual consistency...")
    ap_store = DistributedStore(nodes, ConsistencyMode.EVENTUAL)
    await ap_store.write("user:456", "Jane")  # This succeeds

if __name__ == "__main__":
    asyncio.run(demo_cap_theorem())
```

## 🟨 JavaScript Example

```javascript
const EventEmitter = require('events');

class DistributedCache extends EventEmitter {
    constructor(nodeIds, consistencyLevel = 'eventual') {
        super();
        this.nodes = new Map();
        this.consistencyLevel = consistencyLevel;
        
        // Initialize nodes
        nodeIds.forEach(id => {
            this.nodes.set(id, {
                id,
                data: new Map(),
                isHealthy: true,
                lastHeartbeat: Date.now()
            });
        });
        
        this.startHealthCheck();
    }
    
    async set(key, value) {
        const healthyNodes = this.getHealthyNodes();
        
        if (this.consistencyLevel === 'strong') {
            // CP: All nodes must be healthy for strong consistency
            if (healthyNodes.length !== this.nodes.size) {
                throw new Error('CAP: Cannot guarantee consistency - some nodes unhealthy');
            }
            
            // Simulate synchronous write to all nodes
            const promises = healthyNodes.map(async node => {
                await this.simulateNetworkDelay();
                node.data.set(key, { value, timestamp: Date.now() });
            });
            
            await Promise.all(promises);
            console.log(`✅ Strong consistency SET: ${key}=${value}`);
            
        } else {
            // AP: Write to available nodes, accept inconsistency
            if (healthyNodes.length === 0) {
                throw new Error('No healthy nodes available');
            }
            
            // Write to available nodes without waiting for all
            healthyNodes.forEach(node => {
                node.data.set(key, { value, timestamp: Date.now() });
            });
            
            console.log(`✅ Eventual consistency SET: ${key}=${value} (${healthyNodes.length}/${this.nodes.size} nodes)`);
        }
        
        this.emit('write', { key, value, nodesWritten: healthyNodes.length });
    }
    
    async get(key) {
        const healthyNodes = this.getHealthyNodes();
        
        if (healthyNodes.length === 0) {
            return null; // System unavailable
        }
        
        // For simplicity, read from first available node
        // In practice, you might implement read repair or quorum reads
        const node = healthyNodes[0];
        const result = node.data.get(key);
        
        return result ? result.value : null;
    }
    
    getHealthyNodes() {
        return Array.from(this.nodes.values()).filter(node => node.isHealthy);
    }
    
    // Simulate network partitions
    simulatePartition(nodeId, duration = 5000) {
        const node = this.nodes.get(nodeId);
        if (node) {
            node.isHealthy = false;
            console.log(`🔥 Network partition: ${nodeId} isolated`);
            
            setTimeout(() => {
                node.is