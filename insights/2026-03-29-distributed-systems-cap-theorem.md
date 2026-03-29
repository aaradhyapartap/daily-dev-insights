# 📌 Distributed systems: CAP theorem
*March 29, 2026 · Daily Dev Insight*

## 🧠 Overview

The CAP theorem isn't just theoretical computer science—it's a daily reality check for every distributed system you build. When Eric Brewer first proposed it in 2000, he gave us a fundamental truth: in any distributed system, you can only guarantee two out of three properties: Consistency (all nodes see the same data simultaneously), Availability (the system remains operational), and Partition tolerance (the system continues despite network failures).

The real insight isn't that you must choose—it's that network partitions *will* happen in production. Your database will lose connection to some nodes, your microservices will experience timeouts, and your load balancers will route around failed instances. The CAP theorem forces you to decide upfront: when the network fails, do you sacrifice consistency (AP systems like DynamoDB) or availability (CP systems like MongoDB with strong consistency)? This decision shapes your entire architecture, from database choice to error handling strategies.

Understanding CAP isn't about memorizing definitions—it's about making conscious trade-offs. Every time you choose between immediate consistency and high availability, you're applying CAP theorem. Every time you decide how to handle network splits, you're living it.

## 💡 Key Concepts

• **Consistency vs Eventual Consistency**: Strong consistency means all reads receive the most recent write, but eventual consistency allows temporary inconsistencies that resolve over time. Choose based on whether stale data breaks your business logic.

• **Availability isn't Uptime**: A system can be "up" but unavailable if it can't process requests due to consistency requirements. True availability means every request gets a response, even if it's not the latest data.

• **Network Partitions are Inevitable**: Don't plan for "if" partitions happen—plan for "when." Modern cloud infrastructure, Docker networks, and Kubernetes clusters all experience regular network hiccups.

• **PACELC Extension**: In practice, even when there's no partition, you're trading Latency vs Consistency. Low latency often means accepting slightly stale data.

• **Business Requirements Drive CAP Choices**: Banking systems lean CP (consistency over availability), while social media feeds lean AP (availability over consistency). Your domain determines your trade-offs.

## 🐍 Python Example

```python
import asyncio
import aiohttp
import time
from typing import Dict, Optional

class DistributedCache:
    """Demonstrates CAP theorem trade-offs in a distributed cache system"""
    
    def __init__(self, nodes: list, consistency_level: str = "eventual"):
        self.nodes = nodes  # List of cache node URLs
        self.consistency_level = consistency_level  # "strong" or "eventual"
        self.local_cache = {}
        
    async def write(self, key: str, value: str) -> bool:
        """Write operation showing CP vs AP trade-offs"""
        if self.consistency_level == "strong":
            # CP: All nodes must acknowledge write (sacrifices availability)
            return await self._write_all_nodes(key, value)
        else:
            # AP: Write to any available node (sacrifices consistency)
            return await self._write_any_node(key, value)
    
    async def read(self, key: str) -> Optional[str]:
        """Read operation handling partition scenarios"""
        if self.consistency_level == "strong":
            # CP: Read from majority of nodes for consistency
            return await self._read_consistent(key)
        else:
            # AP: Read from any available node, faster but possibly stale
            return await self._read_available(key)
    
    async def _write_all_nodes(self, key: str, value: str) -> bool:
        """Strong consistency: all nodes must confirm write"""
        successful_writes = 0
        async with aiohttp.ClientSession(timeout=aiohttp.ClientTimeout(total=1)) as session:
            tasks = []
            for node in self.nodes:
                tasks.append(self._write_to_node(session, node, key, value))
            
            results = await asyncio.gather(*tasks, return_exceptions=True)
            successful_writes = sum(1 for r in results if r is True)
            
            # Require majority consensus for strong consistency
            return successful_writes > len(self.nodes) // 2
    
    async def _write_any_node(self, key: str, value: str) -> bool:
        """High availability: succeed if any node accepts write"""
        async with aiohttp.ClientSession(timeout=aiohttp.ClientTimeout(total=0.5)) as session:
            for node in self.nodes:
                try:
                    success = await self._write_to_node(session, node, key, value)
                    if success:
                        return True
                except:
                    continue
            return False
    
    async def _write_to_node(self, session: aiohttp.ClientSession, 
                           node: str, key: str, value: str) -> bool:
        """Simulate writing to a distributed node"""
        try:
            async with session.post(f"{node}/cache/{key}", 
                                  json={"value": value}) as response:
                return response.status == 200
        except:
            return False
```

## 🟨 JavaScript Example

```javascript
class DistributedDatabase {
  constructor(replicas, mode = 'AP') {
    this.replicas = replicas; // Array of database replica URLs
    this.mode = mode; // 'CP' for consistency, 'AP' for availability
    this.writeQuorum = Math.ceil(replicas.length / 2);
    this.readQuorum = Math.ceil(replicas.length / 2);
  }

  async write(key, value) {
    const writePromises = this.replicas.map(replica => 
      this.writeToReplica(replica, key, value)
    );

    if (this.mode === 'CP') {
      // CP: Wait for quorum, fail if network partition prevents consensus
      const results = await Promise.allSettled(writePromises);
      const successful = results.filter(r => r.status === 'fulfilled').length;
      
      if (successful < this.writeQuorum) {
        throw new Error('Cannot maintain consistency - insufficient replicas');
      }
      return true;
    } else {
      // AP: Accept write if ANY replica succeeds, handle conflicts later
      try {
        await Promise.any(writePromises);
        return true;
      } catch (error) {
        throw new Error('All replicas unavailable');
      }
    }
  }

  async read(key) {
    const readPromises = this.replicas.map(replica => 
      this.readFromReplica(replica, key)
    );

    if (this.mode === 'CP') {
      // CP: Read from quorum to ensure consistency
      const results = await Promise.allSettled(readPromises);
      const values = results
        .filter(r => r.status === 'fulfilled')
        .map(r => r.value);
      
      if (values.length < this.readQuorum) {
        throw new Error('Cannot guarantee consistency - insufficient replicas');
      }
      
      // Return most common value (simple conflict resolution)
      return this.getMostCommonValue(values);
    } else {
      // AP: Return first successful read, might be stale
      try {
        return await Promise.any(readPromises);
      } catch (error) {
        throw new Error('No replicas available for read');
      }
    }
  }

  async writeToReplica(replica, key, value) {
    // Simulate network call with potential partition
    return new Promise((resolve, reject) => {
      const networkDelay = Math.random() * 100;
      const partitionChance = 0.1; // 10% chance of network partition
      
      setTimeout(() => {
        if (Math.random() < partitionChance) {
          reject(new Error(`Network partition to ${replica}`));
        } else {
          resolve({ replica, key, value, timestamp: Date.now() });
        }
      }, networkDelay);
    });
  }

  async readFromReplica(replica, key) {
    