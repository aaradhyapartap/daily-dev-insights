# 📌 Data pipelines with Python
*March 27, 2026 · Daily Dev Insight*

## 🧠 Overview

Data pipelines are the backbone of modern data-driven applications, transforming raw information into actionable insights through a series of automated processing steps. Think of them as assembly lines for data—extracting from sources, transforming formats and structures, and loading into destinations where analytics teams can work their magic. The beauty lies not just in automation, but in creating reliable, observable systems that can handle failures gracefully and scale with your data volume.

Python has emerged as the undisputed champion for data pipeline development, thanks to its rich ecosystem of libraries like Pandas, Apache Airflow, and Prefect. Unlike heavyweight enterprise solutions, Python pipelines offer the perfect balance of simplicity and power, letting you iterate quickly while maintaining production-grade reliability. The key is designing for modularity and testability from day one—your future self will thank you when debugging a pipeline at 2 AM.

## 💡 Key Concepts

• **ETL vs ELT Philosophy**: Traditional ETL transforms data before loading, while modern ELT loads raw data first then transforms in the destination. Choose ELT for cloud data warehouses, ETL for limited storage scenarios.

• **Idempotency**: Your pipeline should produce the same result when run multiple times with the same input. This makes retries safe and debugging predictable.

• **Backpressure Handling**: Design for scenarios where downstream systems can't keep up with data flow. Implement queuing, batching, or circuit breakers to prevent cascade failures.

• **Schema Evolution**: Data schemas change over time. Build pipelines that can gracefully handle new fields, missing columns, and type changes without breaking.

• **Observability First**: Instrument your pipeline with logging, metrics, and alerts from the start. You need visibility into data quality, processing times, and failure modes.

## 🐍 Python Example

```python
import pandas as pd
import logging
from datetime import datetime
from pathlib import Path
from typing import Dict, Any

class DataPipeline:
    def __init__(self, config: Dict[str, Any]):
        self.config = config
        self.logger = logging.getLogger(__name__)
        
    def extract(self) -> pd.DataFrame:
        """Extract data from multiple sources with error handling"""
        try:
            # Simulate reading from API and database
            api_data = pd.read_json(self.config['api_endpoint'])
            db_data = pd.read_sql(self.config['sql_query'], self.config['db_conn'])
            
            # Combine sources with common key
            merged_data = pd.merge(api_data, db_data, on='user_id', how='inner')
            self.logger.info(f"Extracted {len(merged_data)} records")
            return merged_data
            
        except Exception as e:
            self.logger.error(f"Extraction failed: {str(e)}")
            raise
    
    def transform(self, data: pd.DataFrame) -> pd.DataFrame:
        """Apply business logic transformations"""
        # Data quality checks
        initial_count = len(data)
        data = data.dropna(subset=['email', 'signup_date'])
        
        # Feature engineering
        data['days_since_signup'] = (
            datetime.now() - pd.to_datetime(data['signup_date'])
        ).dt.days
        
        # Standardize formats
        data['email'] = data['email'].str.lower().str.strip()
        data['revenue'] = pd.to_numeric(data['revenue'], errors='coerce').fillna(0)
        
        # Flag anomalies
        revenue_threshold = data['revenue'].quantile(0.95)
        data['high_value_customer'] = data['revenue'] > revenue_threshold
        
        self.logger.info(f"Transformed data: {initial_count} -> {len(data)} records")
        return data
    
    def load(self, data: pd.DataFrame) -> bool:
        """Load data to destination with transaction safety"""
        output_path = Path(self.config['output_path'])
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        
        try:
            # Atomic write using temporary file
            temp_path = output_path.with_suffix(f'.tmp_{timestamp}')
            data.to_parquet(temp_path, compression='snappy')
            temp_path.rename(output_path)
            
            self.logger.info(f"Successfully loaded {len(data)} records to {output_path}")
            return True
            
        except Exception as e:
            self.logger.error(f"Load failed: {str(e)}")
            return False
    
    def run(self) -> bool:
        """Execute complete pipeline with error handling"""
        try:
            data = self.extract()
            transformed_data = self.transform(data)
            return self.load(transformed_data)
        except Exception as e:
            self.logger.critical(f"Pipeline failed: {str(e)}")
            return False

# Usage
config = {
    'api_endpoint': 'https://api.example.com/users',
    'sql_query': 'SELECT user_id, revenue FROM sales',
    'db_conn': 'postgresql://user:pass@localhost:5432/db',
    'output_path': 'processed_data.parquet'
}

pipeline = DataPipeline(config)
success = pipeline.run()
```

## 🟨 JavaScript Example

```javascript
const fs = require('fs').promises;
const csv = require('csv-parser');
const createCsvWriter = require('csv-writer').createObjectCsvWriter;
const { pipeline, Transform } = require('stream');
const { promisify } = require('util');

class StreamingDataPipeline {
    constructor(config) {
        this.config = config;
        this.stats = { processed: 0, errors: 0, startTime: Date.now() };
    }

    // Transform stream for data processing
    createTransformStream() {
        return new Transform({
            objectMode: true,
            transform(chunk, encoding, callback) {
                try {
                    // Data validation
                    if (!chunk.email || !chunk.amount) {
                        this.stats.errors++;
                        return callback(); // Skip invalid records
                    }

                    // Data transformation
                    const transformed = {
                        email: chunk.email.toLowerCase().trim(),
                        amount: parseFloat(chunk.amount) || 0,
                        category: this.categorizeAmount(parseFloat(chunk.amount)),
                        processed_date: new Date().toISOString(),
                        risk_score: this.calculateRiskScore(chunk)
                    };

                    this.stats.processed++;
                    callback(null, transformed);

                } catch (error) {
                    console.error(`Transform error: ${error.message}`);
                    this.stats.errors++;
                    callback(); // Continue processing
                }
            },

            categorizeAmount(amount) {
                if (amount < 100) return 'low';
                if (amount < 1000) return 'medium';
                return 'high';
            },

            calculateRiskScore(record) {
                // Simple risk scoring based on amount and email domain
                let score = 0;
                const amount = parseFloat(record.amount) || 0;
                
                if (amount > 5000) score += 3;
                else if (amount > 1000) score += 1;
                
                const emailDomain = record.email?.split('@')[1] || '';
                const suspiciousDomains = ['tempmail.com', '10minutemail.com'];
                if (suspiciousDomains.includes(emailDomain)) score += 2;
                
                return Math.min(score, 5); // Cap at 5
            }
        });
    }

    // Main pipeline execution with streaming
    async run() {
        const pipelineAsync = promisify(pipeline);
        
        try {
            console.log('Starting streaming data pipeline...');
            
            // Setup CSV writer
            const csvWriter = createCsvWriter({
                path: this.config.outputPath,
                header: [
                    {