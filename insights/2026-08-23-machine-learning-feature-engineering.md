# 📌 Machine learning feature engineering
*August 23, 2026 · Daily Dev Insight*

## 🧠 Overview

Feature engineering is the art and science of transforming raw data into meaningful inputs that machine learning models can actually learn from. Think of it as translating human intuition about a problem into a language your model understands. While deep learning has automated some of this process, feature engineering remains critical for tabular data, resource-constrained environments, and whenever you need interpretable models.

The brutal truth? Your model is only as good as the features you feed it. I've seen simple logistic regression models with well-crafted features outperform complex neural networks operating on raw data. Feature engineering requires domain knowledge, creativity, and iterative experimentation. It's where data science meets software engineering—you're not just analyzing data, you're architecting the foundation of your ML system.

The challenge is balancing automation with intention. Over-engineering features leads to overfitting and maintenance nightmares. Under-engineering means leaving performance on the table. Modern practitioners use techniques like automated feature selection, polynomial features, and embedding layers to find that sweet spot between manual crafting and scalable automation.

## 💡 Key Concepts

- **Feature scaling/normalization**: Standardize numerical features to prevent large-magnitude features from dominating model training. Critical for gradient-based algorithms and distance-based methods like KNN.

- **Encoding categorical variables**: Transform non-numeric data using techniques like one-hot encoding, target encoding, or embeddings. The choice dramatically impacts model performance and training time.

- **Feature extraction from datetime**: Extract cyclical patterns (day of week, hour), trends, and time-based aggregations. Temporal features often contain hidden predictive power that's easy to unlock.

- **Polynomial and interaction features**: Capture non-linear relationships and feature combinations. Use sparingly—quadratic expansion of 100 features creates 5,000 new ones.

- **Domain-specific transformations**: Leverage your understanding of the problem space. For fraud detection, velocity metrics matter. For NLP, TF-IDF captures term importance. Generic approaches miss these insights.

## 🐍 Python Example

```python
import pandas as pd
import numpy as np
from sklearn.preprocessing import StandardScaler, PolynomialFeatures
from sklearn.model_selection import train_test_split

# Sample e-commerce transaction data
data = pd.DataFrame({
    'timestamp': pd.date_range('2026-01-01', periods=1000, freq='h'),
    'amount': np.random.exponential(50, 1000),
    'user_age': np.random.randint(18, 70, 1000),
    'category': np.random.choice(['electronics', 'clothing', 'food'], 1000),
    'is_fraud': np.random.choice([0, 1], 1000, p=[0.95, 0.05])
})

def engineer_features(df):
    """Transform raw data into ML-ready features"""
    df = df.copy()
    
    # Temporal features - capture time patterns
    df['hour'] = df['timestamp'].dt.hour
    df['day_of_week'] = df['timestamp'].dt.dayofweek
    df['is_weekend'] = (df['day_of_week'] >= 5).astype(int)
    
    # Cyclical encoding for hour (handles 23->0 wraparound)
    df['hour_sin'] = np.sin(2 * np.pi * df['hour'] / 24)
    df['hour_cos'] = np.cos(2 * np.pi * df['hour'] / 24)
    
    # Amount transformations - reduce skew
    df['amount_log'] = np.log1p(df['amount'])
    df['amount_squared'] = df['amount'] ** 2
    
    # One-hot encode category
    category_dummies = pd.get_dummies(df['category'], prefix='cat')
    df = pd.concat([df, category_dummies], axis=1)
    
    # Interaction feature - age-amount relationship
    df['age_amount_ratio'] = df['user_age'] / (df['amount'] + 1)
    
    # Select final feature columns
    feature_cols = ['hour_sin', 'hour_cos', 'is_weekend', 'amount_log', 
                    'amount_squared', 'user_age', 'age_amount_ratio'] + \
                   list(category_dummies.columns)
    
    return df[feature_cols], df['is_fraud']

X, y = engineer_features(data)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)

# Scale features for model training
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)

print(f"Engineered {X.shape[1]} features from {len(data.columns)} raw columns")
```

## 🟨 JavaScript Example

```javascript
// Feature engineering for Node.js ML pipelines
class FeatureEngineering {
  constructor(data) {
    this.data = data;
  }

  // Create rolling window statistics
  createRollingFeatures(values, windowSize = 5) {
    const rolling = [];
    for (let i = 0; i < values.length; i++) {
      const window = values.slice(
        Math.max(0, i - windowSize + 1), 
        i + 1
      );
      rolling.push({
        rolling_mean: window.reduce((a, b) => a + b) / window.length,
        rolling_max: Math.max(...window),
        rolling_min: Math.min(...window),
        rolling_std: this.calculateStd(window)
      });
    }
    return rolling;
  }

  calculateStd(arr) {
    const mean = arr.reduce((a, b) => a + b) / arr.length;
    const variance = arr.reduce((a, b) => a + Math.pow(b - mean, 2), 0) / arr.length;
    return Math.sqrt(variance);
  }

  // Normalize numeric features to [0, 1]
  minMaxScale(column) {
    const values = this.data.map(row => row[column]);
    const min = Math.min(...values);
    const max = Math.max(...values);
    return this.data.map(row => ({
      ...row,
      [`${column}_scaled`]: (row[column] - min) / (max - min)
    }));
  }

  // Target encoding for high-cardinality categoricals
  targetEncode(categoricalCol, targetCol) {
    const groupStats = {};
    
    // Calculate mean target per category
    this.data.forEach(row => {
      if (!groupStats[row[categoricalCol]]) {
        groupStats[row[categoricalCol]] = { sum: 0, count: 0 };
      }
      groupStats[row[categoricalCol]].sum += row[targetCol];
      groupStats[row[categoricalCol]].count += 1;
    });

    // Apply encoding
    return this.data.map(row => ({
      ...row,
      [`${categoricalCol}_encoded`]: 
        groupStats[row[categoricalCol]].sum / 
        groupStats[row[categoricalCol]].count
    }));
  }
}

// Example usage
const transactions = [
  { date: new Date('2026-08-01'), amount: 100, merchant: 'Amazon', fraud: 0 },
  { date: new Date('2026-08-02'), amount: 250, merchant: 'Amazon', fraud: 0 },
  { date: new Date('2026-08-03'), amount: 5000, merchant: 'Unknown', fraud: 1 }
];

const fe = new FeatureEngineering(transactions);
const normalized = fe.minMaxScale('amount');
console.log('Engineered features:', normalized);
```

## ⚖️ When To Use / When To Avoid

**Use Feature Engineering When:**
- Working with tabular/structured data (customer records,