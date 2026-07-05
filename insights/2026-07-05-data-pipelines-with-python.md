# 📌 Data pipelines with Python
*July 05, 2026 · Daily Dev Insight*

## 🧠 Overview

Data pipelines are the unsung heroes of modern software systems—they're the plumbing that moves, transforms, and validates data from source to destination. Think of them as assembly lines for information: raw data comes in one end, goes through a series of transformations, and emerges clean, enriched, and ready for analysis or storage. While the concept sounds straightforward, building robust pipelines that handle failures gracefully, scale efficiently, and remain maintainable is an art form.

Python has become the de facto language for data pipeline development, and for good reason. Its rich ecosystem of libraries (Pandas, Apache Airflow, Prefect, Dagster), combined with readable syntax, makes it ideal for data engineers and analysts alike. Whether you're building a simple ETL job that runs nightly or orchestrating complex DAGs with dozens of interdependent tasks, Python gives you the tools to start simple and scale as needed.

The beauty of well-designed pipelines is that they're composable and testable. Each stage should do one thing well, making debugging easier and enabling you to swap out components without rewriting everything. The examples below demonstrate pipeline patterns you can adapt to real-world scenarios—from file processing to API-based data extraction.

## 💡 Key Concepts

- **Idempotency**: Pipeline stages should produce the same output when run multiple times with the same input. This makes retries safe and debugging predictable.

- **Error handling & logging**: Pipelines fail. Networks drop, APIs rate-limit, and files get corrupted. Build with failure in mind—use structured logging, implement retry logic with exponential backoff, and always validate data at stage boundaries.

- **Incremental processing**: Don't reprocess everything on every run. Use timestamps, watermarks, or checkpointing to track what's been processed. Your future self (and your cloud bill) will thank you.

- **Separation of concerns**: Keep extraction, transformation, and loading as distinct stages. This makes testing easier and allows you to swap out data sources or destinations without touching business logic.

- **Backpressure & batching**: Don't try to load a million records into memory at once. Use generators, process in batches, and implement backpressure mechanisms when downstream systems can't keep up.

## 🐍 Python Example

```python
import logging
from typing import Iterator, Dict, Any
from datetime import datetime
import json

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class DataPipeline:
    """A simple but practical data pipeline framework"""
    
    def __init__(self, batch_size: int = 100):
        self.batch_size = batch_size
        self.processed_count = 0
    
    def extract(self, source_file: str) -> Iterator[Dict[str, Any]]:
        """Extract records from a JSON Lines file"""
        logger.info(f"Starting extraction from {source_file}")
        try:
            with open(source_file, 'r') as f:
                for line in f:
                    yield json.loads(line.strip())
        except FileNotFoundError:
            logger.error(f"Source file not found: {source_file}")
            raise
    
    def transform(self, record: Dict[str, Any]) -> Dict[str, Any]:
        """Transform and validate a single record"""
        # Add processing timestamp
        record['processed_at'] = datetime.utcnow().isoformat()
        
        # Validate required fields
        required = ['id', 'amount']
        if not all(field in record for field in required):
            raise ValueError(f"Missing required fields in {record}")
        
        # Transform amount to cents (assuming it's in dollars)
        record['amount_cents'] = int(float(record['amount']) * 100)
        
        return record
    
    def load(self, records: list[Dict[str, Any]]) -> None:
        """Load batch of records to destination"""
        logger.info(f"Loading batch of {len(records)} records")
        # In production, this would write to DB, S3, etc.
        for record in records:
            self.processed_count += 1
    
    def run(self, source_file: str) -> None:
        """Execute the complete pipeline"""
        batch = []
        
        for raw_record in self.extract(source_file):
            try:
                transformed = self.transform(raw_record)
                batch.append(transformed)
                
                # Load in batches
                if len(batch) >= self.batch_size:
                    self.load(batch)
                    batch = []
                    
            except Exception as e:
                logger.error(f"Failed to process record: {e}")
                continue
        
        # Load remaining records
        if batch:
            self.load(batch)
        
        logger.info(f"Pipeline complete. Processed {self.processed_count} records")

# Usage
if __name__ == "__main__":
    pipeline = DataPipeline(batch_size=50)
    pipeline.run("transactions.jsonl")
```

## 🟨 JavaScript Example

```javascript
const fs = require('fs');
const readline = require('readline');
const { pipeline } = require('stream/promises');
const { Transform } = require('stream');

class DataPipeline {
  constructor(batchSize = 100) {
    this.batchSize = batchSize;
    this.processedCount = 0;
  }

  // Create a transform stream for data processing
  createTransformStream() {
    let batch = [];
    
    return new Transform({
      objectMode: true,
      
      transform(chunk, encoding, callback) {
        try {
          // Parse and transform the record
          const record = JSON.parse(chunk);
          record.processed_at = new Date().toISOString();
          
          // Validate required fields
          if (!record.id || record.amount === undefined) {
            console.error(`Invalid record: ${chunk}`);
            return callback();
          }
          
          // Transform amount to cents
          record.amount_cents = Math.round(parseFloat(record.amount) * 100);
          
          batch.push(record);
          
          // Emit batch when size is reached
          if (batch.length >= this.batchSize) {
            this.push(JSON.stringify(batch));
            batch = [];
          }
          
          callback();
          
        } catch (error) {
          console.error(`Transform error: ${error.message}`);
          callback();
        }
      },
      
      flush(callback) {
        // Emit remaining records
        if (batch.length > 0) {
          this.push(JSON.stringify(batch));
        }
        callback();
      }
    });
  }

  async run(sourceFile, destinationFile) {
    const fileStream = fs.createReadStream(sourceFile);
    const rl = readline.createInterface({ input: fileStream });
    const transformStream = this.createTransformStream();
    const writeStream = fs.createWriteStream(destinationFile);

    try {
      // Process line by line through transform stream
      for await (const line of rl) {
        transformStream.write(line);
        this.processedCount++;
      }
      
      transformStream.end();
      
      // Write transformed batches to destination
      transformStream.pipe(writeStream);
      
      console.log(`Pipeline complete. Processed ${this.processedCount} records`);
      
    } catch (error) {
      console.error(`Pipeline failed: ${error.message}`);
      throw error;
    }
  }
}

// Usage
const pipeline = new DataPipeline(50);
pipeline.run('transactions.jsonl', 'output.jsonl');
```

## ⚖️ When To Use / When To Avoid

**When To Use:**
- ✅ You need to move data between systems on a schedule (ETL/ELT workflows)
- ✅ You're processing large datasets that don't fit in memory
- ✅ You need reproducible, auditable data transformations
- ✅ Multiple data sources need to be combined and normalized
- ✅ You want to decouple data producers from consumers

**When To