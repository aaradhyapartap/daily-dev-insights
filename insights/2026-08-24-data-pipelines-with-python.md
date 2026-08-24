# 📌 Data pipelines with Python
*August 24, 2026 · Daily Dev Insight*

## 🧠 Overview

Data pipelines are the backbone of modern data-driven applications, orchestrating the flow of data from source to destination through a series of transformations. Think of them as assembly lines for your data—each stage performs a specific operation, whether it's extraction from APIs, cleaning messy records, enriching with external data, or loading into a warehouse. Python has become the lingua franca for building these pipelines thanks to its rich ecosystem of libraries like Pandas, Apache Airflow, and prefect, combined with its readability and flexibility.

The beauty of well-designed data pipelines lies in their composability and fault tolerance. Rather than writing monolithic scripts that fail catastrophically, modern pipelines break work into discrete, testable stages with clear inputs and outputs. This approach enables partial reruns when things go wrong (and they will), makes debugging significantly easier, and allows teams to reason about data lineage—tracking exactly how each piece of data was transformed from raw input to final output.

What separates amateur pipelines from production-grade systems is how they handle the inevitable: schema changes, network failures, duplicate data, and partial outages. Professional pipelines embrace idempotency (running the same operation multiple times produces the same result), implement proper logging and monitoring, and use backpressure mechanisms to prevent overwhelming downstream systems. Whether you're building ETL for a data warehouse or processing real-time event streams, these principles remain constant.

## 💡 Key Concepts

- **ETL vs ELT**: Extract-Transform-Load (traditional) processes data before storage, while Extract-Load-Transform (modern) leverages warehouse computing power. Choose ETL for complex transformations and ELT when your warehouse can handle the heavy lifting.

- **Idempotency and Checkpointing**: Design pipeline stages so they can be safely rerun without duplicating data or causing inconsistencies. Use checkpointing to save progress and resume from failure points rather than starting over.

- **Backpressure Management**: When consumers can't keep up with producers, you need strategies to slow down or buffer data flow. Unbounded queues will eventually exhaust memory—implement proper rate limiting and circuit breakers.

- **Data Quality Gates**: Validate data at pipeline boundaries with schema checks, null constraints, and business rule validation. Fail fast when data quality issues are detected rather than propagating bad data downstream.

- **Incremental Processing**: Process only new or changed data rather than full refreshes. Use watermarks, timestamps, or change data capture (CDC) to identify what needs processing, dramatically improving efficiency.

## 🐍 Python Example

```python
from dataclasses import dataclass
from typing import Iterator, List
import json
import logging
from datetime import datetime

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class Event:
    user_id: int
    action: str
    timestamp: str
    metadata: dict

class DataPipeline:
    """A composable data pipeline with stage isolation and error handling."""
    
    def __init__(self, checkpoint_file: str = "checkpoint.json"):
        self.checkpoint_file = checkpoint_file
        self.processed_count = 0
    
    def extract(self, source: List[dict]) -> Iterator[Event]:
        """Extract and parse raw events from source."""
        for raw_event in source:
            try:
                yield Event(**raw_event)
            except TypeError as e:
                logger.warning(f"Skipping malformed event: {e}")
                continue
    
    def transform(self, events: Iterator[Event]) -> Iterator[Event]:
        """Filter and enrich events."""
        for event in events:
            # Skip test users
            if event.user_id < 1000:
                continue
            
            # Enrich with derived fields
            event.metadata['processed_at'] = datetime.utcnow().isoformat()
            event.metadata['action_category'] = event.action.split('_')[0]
            
            yield event
    
    def load(self, events: Iterator[Event], batch_size: int = 100):
        """Batch load events with checkpoint recovery."""
        batch = []
        
        for event in events:
            batch.append(event)
            
            if len(batch) >= batch_size:
                self._write_batch(batch)
                batch = []
        
        # Write remaining events
        if batch:
            self._write_batch(batch)
        
        logger.info(f"Pipeline complete. Processed {self.processed_count} events")
    
    def _write_batch(self, batch: List[Event]):
        """Simulate writing to database/warehouse."""
        # In production: use database transactions, handle retries
        logger.info(f"Writing batch of {len(batch)} events")
        self.processed_count += len(batch)
        # Simulate: db.insert_many([vars(e) for e in batch])

# Usage
raw_data = [
    {"user_id": 1001, "action": "purchase_completed", "timestamp": "2026-08-24T10:00:00Z", "metadata": {}},
    {"user_id": 500, "action": "page_view", "timestamp": "2026-08-24T10:01:00Z", "metadata": {}},
    {"user_id": 1002, "action": "purchase_abandoned", "timestamp": "2026-08-24T10:02:00Z", "metadata": {}},
]

pipeline = DataPipeline()
events = pipeline.extract(raw_data)
transformed = pipeline.transform(events)
pipeline.load(transformed, batch_size=2)
```

## 🟨 JavaScript Example

```javascript
const { Transform, pipeline } = require('stream');
const { promisify } = require('util');

const pipelineAsync = promisify(pipeline);

// Extract stage: read from data source
class ExtractStream extends Transform {
  constructor() {
    super({ objectMode: true });
  }
  
  _transform(chunk, encoding, callback) {
    try {
      // Simulate parsing JSON lines
      const event = JSON.parse(chunk.toString());
      this.push(event);
      callback();
    } catch (error) {
      console.warn(`Parse error: ${error.message}`);
      callback(); // Skip bad records, don't fail pipeline
    }
  }
}

// Transform stage: clean and enrich data
class TransformStream extends Transform {
  constructor() {
    super({ objectMode: true });
    this.processedCount = 0;
  }
  
  _transform(event, encoding, callback) {
    // Data quality check
    if (!event.user_id || !event.action) {
      callback();
      return;
    }
    
    // Enrich event
    const enriched = {
      ...event,
      processed_at: new Date().toISOString(),
      action_category: event.action.split('_')[0],
      is_high_value: event.metadata?.amount > 100
    };
    
    this.processedCount++;
    this.push(enriched);
    callback();
  }
}

// Load stage: batch writes with backpressure handling
class LoadStream extends Transform {
  constructor(batchSize = 50) {
    super({ objectMode: true });
    this.batch = [];
    this.batchSize = batchSize;
  }
  
  _transform(event, encoding, callback) {
    this.batch.push(event);
    
    if (this.batch.length >= this.batchSize) {
      this._writeBatch()
        .then(() => callback())
        .catch(callback);
    } else {
      callback();
    }
  }
  
  async _flush(callback) {
    if (this.batch.length > 0) {
      await this._writeBatch();
    }
    callback();
  }
  
  async _writeBatch() {
    console.log(`Writing batch of ${this.batch.length} events`);
    // Simulate async database write
    await new Promise(resolve => setTimeout(resolve, 100));
    this.batch = [];
  }
}

// Run the pipeline
async function runPipeline() {
  const extract = new ExtractStream();
  const transform = new TransformStream