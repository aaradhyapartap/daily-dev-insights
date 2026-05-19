# 📌 Blockchain fundamentals for developers
*May 19, 2026 · Daily Dev Insight*

## 🧠 Overview

Blockchain isn't just cryptocurrency hype—it's a distributed ledger technology that solves real problems around trust, transparency, and decentralization. At its core, a blockchain is an immutable chain of cryptographically linked blocks, each containing transaction data. Think of it as a distributed database where every participant has the complete history, making tampering nearly impossible without consensus from the majority.

The real magic happens in the consensus mechanisms (Proof of Work, Proof of Stake) that allow a network of untrusting parties to agree on a single version of truth without a central authority. For developers, understanding blockchain fundamentals opens doors to building decentralized applications (dApps), smart contracts, and systems that operate without traditional intermediaries. However, blockchain isn't a silver bullet—it comes with tradeoffs in performance, scalability, and complexity that you need to carefully consider.

## 💡 Key Concepts

• **Cryptographic Hashing**: Each block contains a hash of the previous block, creating an immutable chain. SHA-256 is commonly used, where changing even one bit in input data completely changes the output hash.

• **Merkle Trees**: Efficiently summarize all transactions in a block using a binary tree of hashes. This allows quick verification of any transaction without downloading the entire block.

• **Consensus Mechanisms**: Algorithms that ensure all nodes agree on the blockchain state. Proof of Work (mining) vs Proof of Stake (validators) have different energy and security tradeoffs.

• **Smart Contracts**: Self-executing code that runs on the blockchain, automatically enforcing agreements without intermediaries. Think automated escrow services or decentralized exchanges.

• **Gas Fees**: Transaction costs that prevent spam and compensate network validators. Higher fees typically mean faster processing, creating an economic incentive structure.

## 🐍 Python Example

```python
import hashlib
import json
import time
from typing import List, Dict, Any

class Block:
    def __init__(self, transactions: List[Dict], previous_hash: str):
        self.timestamp = time.time()
        self.transactions = transactions
        self.previous_hash = previous_hash
        self.nonce = 0  # Used for proof of work
        self.hash = self.calculate_hash()
    
    def calculate_hash(self) -> str:
        """Generate SHA-256 hash of block contents"""
        block_data = {
            'timestamp': self.timestamp,
            'transactions': self.transactions,
            'previous_hash': self.previous_hash,
            'nonce': self.nonce
        }
        block_string = json.dumps(block_data, sort_keys=True)
        return hashlib.sha256(block_string.encode()).hexdigest()
    
    def mine_block(self, difficulty: int):
        """Simple proof of work - find hash starting with zeros"""
        target = "0" * difficulty
        while self.hash[:difficulty] != target:
            self.nonce += 1
            self.hash = self.calculate_hash()
        print(f"Block mined: {self.hash}")

class Blockchain:
    def __init__(self):
        self.chain = [self.create_genesis_block()]
        self.difficulty = 2  # Mining difficulty
        self.pending_transactions = []
    
    def create_genesis_block(self) -> Block:
        """First block in the chain"""
        return Block([], "0")
    
    def add_block(self, transactions: List[Dict]):
        """Add new block to chain after mining"""
        previous_block = self.chain[-1]
        new_block = Block(transactions, previous_block.hash)
        new_block.mine_block(self.difficulty)
        self.chain.append(new_block)
    
    def is_chain_valid(self) -> bool:
        """Verify blockchain integrity"""
        for i in range(1, len(self.chain)):
            current = self.chain[i]
            previous = self.chain[i-1]
            
            if current.hash != current.calculate_hash():
                return False
            if current.previous_hash != previous.hash:
                return False
        return True

# Example usage
blockchain = Blockchain()
blockchain.add_block([
    {'from': 'Alice', 'to': 'Bob', 'amount': 50},
    {'from': 'Bob', 'to': 'Charlie', 'amount': 25}
])
print(f"Blockchain valid: {blockchain.is_chain_valid()}")
```

## 🟨 JavaScript Example

```javascript
const crypto = require('crypto');

class Block {
    constructor(transactions, previousHash) {
        this.timestamp = Date.now();
        this.transactions = transactions;
        this.previousHash = previousHash;
        this.nonce = 0;
        this.hash = this.calculateHash();
    }
    
    calculateHash() {
        // Create deterministic hash from block contents
        const blockData = JSON.stringify({
            timestamp: this.timestamp,
            transactions: this.transactions,
            previousHash: this.previousHash,
            nonce: this.nonce
        });
        return crypto.createHash('sha256').update(blockData).digest('hex');
    }
    
    mineBlock(difficulty) {
        // Proof of work: find hash with leading zeros
        const target = Array(difficulty + 1).join("0");
        
        while (this.hash.substring(0, difficulty) !== target) {
            this.nonce++;
            this.hash = this.calculateHash();
        }
        
        console.log(`Block mined: ${this.hash}`);
    }
}

class Blockchain {
    constructor() {
        this.chain = [this.createGenesisBlock()];
        this.difficulty = 2;
        this.pendingTransactions = [];
        this.miningReward = 10;
    }
    
    createGenesisBlock() {
        return new Block([], "0");
    }
    
    getLatestBlock() {
        return this.chain[this.chain.length - 1];
    }
    
    minePendingTransactions(miningRewardAddress) {
        // Add mining reward transaction
        this.pendingTransactions.push({
            from: null,
            to: miningRewardAddress,
            amount: this.miningReward
        });
        
        // Create and mine new block
        const block = new Block(
            this.pendingTransactions, 
            this.getLatestBlock().hash
        );
        block.mineBlock(this.difficulty);
        
        this.chain.push(block);
        this.pendingTransactions = [];
    }
    
    createTransaction(transaction) {
        this.pendingTransactions.push(transaction);
    }
    
    getBalance(address) {
        let balance = 0;
        
        // Scan all transactions in all blocks
        for (const block of this.chain) {
            for (const trans of block.transactions) {
                if (trans.from === address) {
                    balance -= trans.amount;
                }
                if (trans.to === address) {
                    balance += trans.amount;
                }
            }
        }
        return balance;
    }
    
    isChainValid() {
        for (let i = 1; i < this.chain.length; i++) {
            const currentBlock = this.chain[i];
            const previousBlock = this.chain[i - 1];
            
            if (currentBlock.hash !== currentBlock.calculateHash()) {
                return false;
            }
            if (currentBlock.previousHash !== previousBlock.hash) {
                return false;
            }
        }
        return true;
    }
}

// Example usage
const myCoin = new Blockchain();
myCoin.createTransaction({from: 'Alice', to: 'Bob', amount: 100});
myCoin.createTransaction({from: 'Bob', to: 'Alice', amount: 50});

console.log('Starting mining...');
myCoin.minePendingTransactions('miner-address');

console.log(`Miner balance: ${myCoin.getBalance