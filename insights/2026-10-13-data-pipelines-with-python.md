# 📌 Data pipelines with Python
*October 13, 2026 · Daily Dev Insight*

## 🧠 Overview

Data pipelines are the backbone of modern data-driven applications, orchestrating the flow of information from raw sources through transformation stages to final destinations. Think of them as assembly lines for data: extracting from databases or APIs, transforming through cleaning and enrichment, and loading into warehouses or analytics platforms. Python has become the de facto language for building these pipelines thanks to its rich ecosystem of data libraries and readable syntax that makes complex transformations maintainable.

The real challenge isn't just moving data from point A to point B—it's doing so reliably, with proper error handling, monitoring, and the ability to replay or rollback when things go wrong. Modern data pipelines need to handle late-arriving data, schema evolution, and operate at scale while remaining debuggable. Whether you're building a simple ETL job or a complex streaming architecture, the principles remain the same: make your pipeline idempotent, observable, and testable.

What separates production-grade pipelines from quick scripts is their resilience and clarity. Good pipelines fail gracefully, log extensively, and provide clear visibility into what data moved when. They're designed with the understanding that data quality issues will occur, and they need to surface problems quickly while continuing to process valid records.

## 💡 Key Concepts

- **ETL vs ELT**: Extract-Transform-Load (traditional warehousing) versus Extract-Load-Transform (modern cloud approach where raw data lands first, transformations happen in the warehouse)
- **Idempotency**: Running the same pipeline multiple times with the same input produces the same output—critical for recovery and reprocessing scenarios
- **Backpressure handling**: Managing situations where downstream systems can't keep up with the data flow rate, preventing memory overflow and system crashes
- **Data partitioning**: Organizing data by time windows or keys to enable parallel processing and efficient incremental updates
- **Schema validation**: Explicitly checking data structure and types early in the pipeline to fail fast and prevent corrupted data propagation

## �🐍 Python Example

```python
import pandas as pd
from datetime import datetime
import logging
from typing import List, Dict

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class DataPipeline:
    def __init__(self, batch_size: int = 1000):
        self.batch_size = batch_size
        self.processed_count = 0
        
    def extract(self, source: str) -> pd.DataFrame:
        """Extract data from source with error handling"""
        try:
            logger.info(f"Extracting data from {source}")
            # Simulating API/database read
            df = pd.read_csv(source)
            logger.info(f"Extracted {len(df)} records")
            return df
        except Exception as e:
            logger.error(f"Extraction failed: {e}")
            raise
    
    def transform(self, df: pd.DataFrame) -> pd.DataFrame:
        """Apply business logic transformations"""
        logger.info("Starting transformation")
        
        # Remove duplicates based on ID
        df = df.drop_duplicates(subset=['user_id'], keep='last')
        
        # Clean and standardize
        df['email'] = df['email'].str.lower().str.strip()
        df['created_at'] = pd.to_datetime(df['created_at'])
        
        # Enrich with derived columns
        df['signup_year'] = df['created_at'].dt.year
        df['is_recent'] = df['created_at'] > '2026-01-01'
        
        # Filter invalid records
        df = df[df['email'].str.contains('@')]
        
        logger.info(f"Transformed to {len(df)} valid records")
        return df
    
    def load(self, df: pd.DataFrame, destination: str):
        """Load data in batches with transaction safety"""
        for i in range(0, len(df), self.batch_size):
            batch = df.iloc[i:i + self.batch_size]
            try:
                # Simulating database write
                batch.to_csv(destination, mode='a', header=(i==0), index=False)
                self.processed_count += len(batch)
                logger.info(f"Loaded batch {i//self.batch_size + 1}")
            except Exception as e:
                logger.error(f"Load failed at batch {i}: {e}")
                raise
    
    def run(self, source: str, destination: str):
        """Execute the full pipeline"""
        start_time = datetime.now()
        df = self.extract(source)
        df = self.transform(df)
        self.load(df, destination)
        duration = (datetime.now() - start_time).total_seconds()
        logger.info(f"Pipeline complete: {self.processed_count} records in {duration}s")

# Usage
# pipeline = DataPipeline(batch_size=500)
# pipeline.run('users.csv', 'output.csv')
```

## 🟨 JavaScript Example

```javascript
const fs = require('fs').promises;
const csv = require('csv-parser');
const { createWriteStream } = require('fs');
const { pipeline } = require('stream/promises');

class DataPipeline {
  constructor(batchSize = 1000) {
    this.batchSize = batchSize;
    this.processedCount = 0;
  }

  async extract(source) {
    console.log(`Extracting data from ${source}`);
    const records = [];
    
    return new Promise((resolve, reject) => {
      require('fs').createReadStream(source)
        .pipe(csv())
        .on('data', (row) => records.push(row))
        .on('end', () => {
          console.log(`Extracted ${records.length} records`);
          resolve(records);
        })
        .on('error', reject);
    });
  }

  transform(records) {
    console.log('Starting transformation');
    
    // Remove duplicates by user_id
    const uniqueRecords = Array.from(
      new Map(records.map(r => [r.user_id, r])).values()
    );
    
    // Clean and enrich
    const transformed = uniqueRecords
      .map(record => ({
        ...record,
        email: record.email?.toLowerCase().trim(),
        created_at: new Date(record.created_at),
        signup_year: new Date(record.created_at).getFullYear(),
        is_recent: new Date(record.created_at) > new Date('2026-01-01')
      }))
      .filter(r => r.email && r.email.includes('@'));
    
    console.log(`Transformed to ${transformed.length} valid records`);
    return transformed;
  }

  async load(records, destination) {
    const writeStream = createWriteStream(destination);
    
    // Write header
    writeStream.write(Object.keys(records[0]).join(',') + '\n');
    
    // Write in batches
    for (let i = 0; i < records.length; i += this.batchSize) {
      const batch = records.slice(i, i + this.batchSize);
      batch.forEach(record => {
        writeStream.write(Object.values(record).join(',') + '\n');
      });
      this.processedCount += batch.length;
      console.log(`Loaded batch ${Math.floor(i / this.batchSize) + 1}`);
    }
    
    writeStream.end();
  }

  async run(source, destination) {
    const startTime = Date.now();
    const records = await this.extract(source);
    const transformed = this.transform(records);
    await this.load(transformed, destination);
    const duration = (Date.now() - startTime) / 1000;
    console.log(`Pipeline complete: ${this.processedCount} records in ${duration}s`);
  }
}

// Usage
// const pipeline = new DataPipeline(500);
// pipeline.run('users.csv', 'output.csv');
```

## ⚖️