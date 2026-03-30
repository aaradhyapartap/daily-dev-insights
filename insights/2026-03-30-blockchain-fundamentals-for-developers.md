# 📌 Blockchain fundamentals for developers
*March 30, 2026 · Daily Dev Insight*

## 🧠 Overview

Blockchain is essentially a distributed database with a twist: instead of trusting a central authority, the network maintains consensus through cryptographic proofs and economic incentives. Think of it as a ledger that's replicated across thousands of computers, where each new entry (block) is cryptographically linked to the previous one, making historical tampering computationally infeasible.

As developers, we need to shift our mental model from traditional CRUD operations to thinking about immutable state transitions and eventual consistency. Unlike your typical REST API where you can update records directly, blockchain development involves broadcasting transactions that compete for inclusion in the next block. This fundamental difference impacts everything from data modeling to user experience design.

The real power isn't just in the immutability—it's in the programmability. Smart contracts let us encode business logic directly into the blockchain, creating self-executing agreements that run exactly as programmed without downtime, censorship, or third-party interference. However, this comes with the trade-off of higher costs and slower throughput compared to traditional databases.

## 💡 Key Concepts

• **Hash Functions & Merkle Trees**: Each block contains a cryptographic hash of the previous block plus a Merkle root of all transactions, creating an tamper-evident chain where changing any historical data would require recalculating all subsequent blocks

• **Consensus Mechanisms**: Networks use algorithms like Proof of Work (Bitcoin) or Proof of Stake (Ethereum 2.0) to agree on the canonical state without a central coordinator—understanding these is crucial for predicting network behavior and costs

• **Gas & Transaction Fees**: Every operation consumes computational resources measured in "gas"—optimizing your smart contract code isn't just about performance, it directly impacts user costs

• **Public/Private Key Cryptography**: Users control assets through cryptographic key pairs; there's no password recovery or customer support—if you lose the private key, the assets are gone forever

• **State Machines**: Blockchains are deterministic state machines where each transaction represents a state transition that must be reproducible across all nodes in the network

## 🐍 Python Example

```python
import hashlib
import json
from datetime import datetime
from typing import List, Dict, Any

class Block:
    def __init__(self, transactions: List[Dict], previous_hash: str = ""):
        self.timestamp = datetime.now().isoformat()
        self.transactions = transactions
        self.previous_hash = previous_hash
        self.nonce = 0
        self.hash = self.calculate_hash()
    
    def calculate_hash(self) -> str:
        """Calculate SHA-256 hash of block contents"""
        block_data = {
            'timestamp': self.timestamp,
            'transactions': self.transactions,
            'previous_hash': self.previous_hash,
            'nonce': self.nonce
        }
        block_string = json.dumps(block_data, sort_keys=True)
        return hashlib.sha256(block_string.encode()).hexdigest()
    
    def mine_block(self, difficulty: int):
        """Proof of Work mining - find hash with required leading zeros"""
        target = "0" * difficulty
        print(f"Mining block with difficulty {difficulty}...")
        
        while self.hash[:difficulty] != target:
            self.nonce += 1
            self.hash = self.calculate_hash()
        
        print(f"Block mined: {self.hash}")

class SimpleBlockchain:
    def __init__(self):
        self.chain = [self.create_genesis_block()]
        self.difficulty = 2  # Number of leading zeros required
        self.pending_transactions = []
    
    def create_genesis_block(self) -> Block:
        """Create the first block in the chain"""
        return Block([{"from": "genesis", "to": "genesis", "amount": 0}])
    
    def add_transaction(self, transaction: Dict[str, Any]):
        """Add transaction to pending pool"""
        self.pending_transactions.append(transaction)
    
    def mine_pending_transactions(self):
        """Mine a new block with pending transactions"""
        block = Block(self.pending_transactions.copy(), 
                     self.get_latest_block().hash)
        block.mine_block(self.difficulty)
        
        self.chain.append(block)
        self.pending_transactions = []  # Clear pending transactions
    
    def get_latest_block(self) -> Block:
        return self.chain[-1]
    
    def is_chain_valid(self) -> bool:
        """Validate the entire blockchain"""
        for i in range(1, len(self.chain)):
            current_block = self.chain[i]
            previous_block = self.chain[i-1]
            
            # Check if current block's hash is valid
            if current_block.hash != current_block.calculate_hash():
                return False
            
            # Check if current block points to previous block
            if current_block.previous_hash != previous_block.hash:
                return False
        
        return True

# Usage example
blockchain = SimpleBlockchain()
blockchain.add_transaction({"from": "Alice", "to": "Bob", "amount": 50})
blockchain.add_transaction({"from": "Bob", "to": "Charlie", "amount": 25})
blockchain.mine_pending_transactions()

print(f"Blockchain valid: {blockchain.is_chain_valid()}")
```

## 🟨 JavaScript Example

```javascript
const crypto = require('crypto');

class Transaction {
    constructor(fromAddress, toAddress, amount) {
        this.fromAddress = fromAddress;
        this.toAddress = toAddress;
        this.amount = amount;
        this.timestamp = Date.now();
    }
    
    calculateHash() {
        return crypto.createHash('sha256')
            .update(this.fromAddress + this.toAddress + this.amount + this.timestamp)
            .digest('hex');
    }
    
    signTransaction(signingKey) {
        // In real implementation, would use elliptic curve cryptography
        const hashTx = this.calculateHash();
        const sig = crypto.createSign('SHA256');
        sig.update(hashTx).end();
        this.signature = sig.sign(signingKey, 'hex');
    }
    
    isValid() {
        // Genesis transaction
        if (this.fromAddress === null) return true;
        
        if (!this.signature || this.signature.length === 0) {
            throw new Error('No signature in this transaction');
        }
        
        // Verify signature matches the hash
        const verify = crypto.createVerify('SHA256');
        verify.update(this.calculateHash()).end();
        return verify.verify(this.fromAddress, this.signature, 'hex');
    }
}

class Wallet {
    constructor() {
        const keyPair = crypto.generateKeyPairSync('rsa', {
            modulusLength: 2048,
            publicKeyEncoding: { type: 'spki', format: 'pem' },
            privateKeyEncoding: { type: 'pkcs8', format: 'pem' }
        });
        
        this.privateKey = keyPair.privateKey;
        this.publicKey = keyPair.publicKey;
    }
    
    getBalance(blockchain) {
        let balance = 0;
        
        // Calculate balance by iterating through all transactions
        for (const block of blockchain.chain) {
            for (const trans of block.transactions) {
                if (trans.fromAddress === this.publicKey) {
                    balance -= trans.amount;
                }
                if (trans.toAddress === this.publicKey) {
                    balance += trans.amount;
                }
            }
        }
        
        return balance;
    }
    
    sendMoney(amount, toAddress, blockchain) {
        const transaction = new Transaction(this.publicKey, toAddress, amount);
        transaction.signTransaction(this.privateKey);
        blockchain.addTransaction(transaction);
        
        return transaction;
    }
}

// Usage example
const { SimpleBlockchain } = require('./blockchain'); // Assuming previous code
const blockchain = new SimpleBlockchain();

const wallet1 