# 📌 Data pipelines with Python
*May 16, 2026 · Daily Dev Insight*

## 🧠 Overview

Data pipelines are the backbone of modern data-driven applications, orchestrating the flow of information from raw sources to actionable insights. Think of them as assembly lines for data – they extract, transform, and load information through a series of connected processing stages. Python has emerged as the lingua franca for pipeline development because of its rich ecosystem of data libraries, readable syntax, and excellent integration capabilities.

The real magic happens when you design pipelines that are both resilient and maintainable. Too many engineers jump straight into complex frameworks when a well-structured Python script would suffice. The key is understanding your data volume, processing requirements, and failure scenarios before choosing your tools. A good pipeline handles errors gracefully, provides visibility into processing status, and can recover from failures without data loss.

## 💡 Key Concepts

• **Idempotency**: Your pipeline should produce the same results when run multiple times on the same input data, making it safe to retry failed operations
• **Incremental Processing**: Process only new or changed data rather than reprocessing everything, using techniques like watermarking or change data capture
• **Error Handling & Dead Letter Queues**: Failed records should be isolated and logged without breaking the entire pipeline flow
• **Monitoring & Observability**: Instrument your pipelines with metrics, logging, and health checks to catch issues before they cascade
• **Backpressure Management**: Handle situations where downstream systems can't keep up with the data flow rate

## 🐍 Python Example

```python
import pandas as pd
import logging
from datetime import datetime
from pathlib import Path
from typing import Dict, List
import json

class DataPipeline:
    def __init__(self, config_path: str):
        self.config = self._load_config(config_path)
        self.setup_logging()
        self.failed_records = []
    
    def setup_logging(self):
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(levelname)s - %(message)s'
        )
        self.logger = logging.getLogger(__name__)
    
    def _load_config(self, path: str) -> Dict:
        with open(path, 'r') as f:
            return json.load(f)
    
    def extract_data(self, source_path: str) -> pd.DataFrame:
        """Extract data with error handling"""
        try:
            df = pd.read_csv(source_path)
            self.logger.info(f"Extracted {len(df)} records from {source_path}")
            return df
        except Exception as e:
            self.logger.error(f"Failed to extract data: {e}")
            raise
    
    def transform_data(self, df: pd.DataFrame) -> pd.DataFrame:
        """Apply transformations with record-level error handling"""
        processed_records = []
        
        for idx, row in df.iterrows():
            try:
                # Clean and validate data
                cleaned_row = {
                    'id': int(row['id']),
                    'email': str(row['email']).lower().strip(),
                    'amount': float(row['amount']),
                    'processed_at': datetime.now().isoformat()
                }
                
                # Business logic validation
                if cleaned_row['amount'] > 0 and '@' in cleaned_row['email']:
                    processed_records.append(cleaned_row)
                else:
                    raise ValueError("Invalid amount or email format")
                    
            except Exception as e:
                self.failed_records.append({'row': idx, 'error': str(e), 'data': row.to_dict()})
                self.logger.warning(f"Failed to process row {idx}: {e}")
        
        return pd.DataFrame(processed_records)
    
    def load_data(self, df: pd.DataFrame, output_path: str):
        """Load data with atomic writes"""
        temp_path = f"{output_path}.tmp"
        try:
            df.to_csv(temp_path, index=False)
            Path(temp_path).rename(output_path)  # Atomic rename
            self.logger.info(f"Successfully loaded {len(df)} records")
        except Exception as e:
            Path(temp_path).unlink(missing_ok=True)  # Cleanup temp file
            raise
    
    def run(self):
        """Execute the complete pipeline"""
        try:
            raw_data = self.extract_data(self.config['source_path'])
            transformed_data = self.transform_data(raw_data)
            self.load_data(transformed_data, self.config['output_path'])
            
            # Log pipeline metrics
            self.logger.info(f"Pipeline completed. Success: {len(transformed_data)}, "
                           f"Failed: {len(self.failed_records)}")
            
            if self.failed_records:
                self._save_failed_records()
                
        except Exception as e:
            self.logger.error(f"Pipeline failed: {e}")
            raise
    
    def _save_failed_records(self):
        """Save failed records to dead letter queue"""
        with open('failed_records.json', 'w') as f:
            json.dump(self.failed_records, f, indent=2)

# Usage
if __name__ == "__main__":
    pipeline = DataPipeline('config.json')
    pipeline.run()
```

## 🟨 JavaScript Example

```javascript
const fs = require('fs').promises;
const csv = require('csv-parser');
const createCsvWriter = require('csv-writer').createObjectCsvWriter;
const winston = require('winston');

class DataPipeline {
    constructor(configPath) {
        this.failedRecords = [];
        this.setupLogging();
        this.loadConfig(configPath);
    }

    setupLogging() {
        this.logger = winston.createLogger({
            level: 'info',
            format: winston.format.combine(
                winston.format.timestamp(),
                winston.format.json()
            ),
            transports: [
                new winston.transports.Console(),
                new winston.transports.File({ filename: 'pipeline.log' })
            ]
        });
    }

    async loadConfig(path) {
        try {
            const configData = await fs.readFile(path, 'utf8');
            this.config = JSON.parse(configData);
        } catch (error) {
            this.logger.error('Failed to load config:', error);
            throw error;
        }
    }

    async extractData(sourcePath) {
        return new Promise((resolve, reject) => {
            const results = [];
            const stream = require('fs').createReadStream(sourcePath);
            
            stream
                .pipe(csv())
                .on('data', (data) => results.push(data))
                .on('end', () => {
                    this.logger.info(`Extracted ${results.length} records`);
                    resolve(results);
                })
                .on('error', reject);
        });
    }

    async transformData(rawData) {
        const processedRecords = [];
        
        for (const [index, record] of rawData.entries()) {
            try {
                // Apply transformations with validation
                const cleanedRecord = {
                    id: parseInt(record.id),
                    email: record.email.toLowerCase().trim(),
                    amount: parseFloat(record.amount),
                    processedAt: new Date().toISOString()
                };

                // Business rules validation
                if (cleanedRecord.amount > 0 && cleanedRecord.email.includes('@')) {
                    processedRecords.push(cleanedRecord);
                } else {
                    throw new Error('Invalid amount or email format');
                }

            } catch (error) {
                this.failedRecords.push({
                    row: index,
                    error: error.message,
                    data: record
                });
                this.logger.warn(`Failed to process row ${index}: ${error.message}`);
            }
        }

        return processedRecords;
    