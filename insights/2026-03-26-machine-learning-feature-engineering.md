# 📌 Machine learning feature engineering
*March 26, 2026 · Daily Dev Insight*

## 🧠 Overview

Feature engineering is where the magic happens in machine learning—it's the art and science of transforming raw data into meaningful inputs that your models can actually learn from. While algorithms get all the hype, experienced practitioners know that thoughtful feature engineering often makes the difference between a model that barely works and one that delivers real business value. It's the process of selecting, modifying, and creating variables that best represent the underlying patterns in your data.

The reality is that most real-world data is messy, incomplete, and not in a format that machine learning algorithms can digest effectively. Feature engineering bridges this gap by applying domain knowledge, statistical techniques, and creative thinking to extract signal from noise. Whether you're dealing with categorical variables that need encoding, timestamps that hide seasonal patterns, or text that needs vectorization, feature engineering is your toolkit for making data ML-ready.

## 💡 Key Concepts

• **Feature Selection vs. Feature Creation**: Selection removes irrelevant features to reduce noise and dimensionality, while creation generates new features from existing ones (like ratios, interactions, or polynomial features)

• **Encoding Categorical Data**: Transform non-numeric data using techniques like one-hot encoding, label encoding, or target encoding—each with different implications for model performance

• **Handling Missing Values**: Strategic approaches like imputation, creating indicator variables for missingness, or using algorithms that handle nulls natively

• **Feature Scaling**: Normalize or standardize features so algorithms that rely on distance calculations (like neural networks or SVM) aren't dominated by features with larger scales

• **Temporal and Domain-Specific Features**: Extract meaningful patterns from timestamps (day of week, seasonality) and leverage domain expertise to create business-relevant features

## 🐍 Python Example

```python
import pandas as pd
import numpy as np
from sklearn.preprocessing import StandardScaler, LabelEncoder
from sklearn.feature_selection import SelectKBest, f_regression
from datetime import datetime

# Sample e-commerce dataset
data = pd.DataFrame({
    'purchase_date': ['2026-01-15', '2026-02-20', '2026-03-10', '2026-01-30'],
    'product_category': ['Electronics', 'Clothing', 'Electronics', 'Books'],
    'price': [299.99, 45.50, 189.99, 12.99],
    'user_age': [25, 34, np.nan, 45],
    'previous_purchases': [3, 0, 7, 12],
    'rating': [4.5, 3.2, 5.0, 4.1]
})

def engineer_features(df):
    df = df.copy()
    
    # Handle missing values - fill with median and create indicator
    df['user_age_missing'] = df['user_age'].isna().astype(int)
    df['user_age'].fillna(df['user_age'].median(), inplace=True)
    
    # Create temporal features
    df['purchase_date'] = pd.to_datetime(df['purchase_date'])
    df['purchase_month'] = df['purchase_date'].dt.month
    df['purchase_weekday'] = df['purchase_date'].dt.dayofweek
    
    # Create interaction features
    df['price_per_previous_purchase'] = df['price'] / (df['previous_purchases'] + 1)
    df['is_repeat_customer'] = (df['previous_purchases'] > 0).astype(int)
    
    # One-hot encode categorical variables
    category_encoded = pd.get_dummies(df['product_category'], prefix='category')
    df = pd.concat([df, category_encoded], axis=1)
    
    # Create price buckets
    df['price_tier'] = pd.cut(df['price'], bins=[0, 20, 100, 500], 
                             labels=['Low', 'Medium', 'High'])
    
    # Scale numerical features
    scaler = StandardScaler()
    numerical_cols = ['price', 'user_age', 'previous_purchases']
    df[numerical_cols] = scaler.fit_transform(df[numerical_cols])
    
    return df.drop(['purchase_date', 'product_category'], axis=1)

# Apply feature engineering
engineered_data = engineer_features(data)
print("Engineered features:")
print(engineered_data.columns.tolist())
```

## 🟨 JavaScript Example

```javascript
// Feature engineering utilities for web-based ML
class FeatureEngineer {
  constructor() {
    this.scalers = {};
    this.encoders = {};
  }

  // Handle missing values with median imputation
  handleMissingValues(data, column) {
    const values = data.map(row => row[column]).filter(val => val !== null && val !== undefined);
    const median = this.calculateMedian(values);
    
    return data.map(row => ({
      ...row,
      [column]: row[column] ?? median,
      [`${column}_was_missing`]: row[column] === null || row[column] === undefined ? 1 : 0
    }));
  }

  // Create temporal features from date strings
  createTimeFeatures(data, dateColumn) {
    return data.map(row => {
      const date = new Date(row[dateColumn]);
      return {
        ...row,
        [`${dateColumn}_month`]: date.getMonth() + 1,
        [`${dateColumn}_day_of_week`]: date.getDay(),
        [`${dateColumn}_hour`]: date.getHours(),
        [`${dateColumn}_is_weekend`]: [0, 6].includes(date.getDay()) ? 1 : 0
      };
    });
  }

  // One-hot encode categorical variables
  oneHotEncode(data, column) {
    const uniqueValues = [...new Set(data.map(row => row[column]))];
    
    return data.map(row => {
      const encoded = {};
      uniqueValues.forEach(value => {
        encoded[`${column}_${value}`] = row[column] === value ? 1 : 0;
      });
      return { ...row, ...encoded };
    });
  }

  // Normalize numerical features
  standardScale(data, columns) {
    columns.forEach(col => {
      const values = data.map(row => row[col]);
      const mean = values.reduce((sum, val) => sum + val, 0) / values.length;
      const std = Math.sqrt(values.reduce((sum, val) => sum + Math.pow(val - mean, 2), 0) / values.length);
      
      data.forEach(row => {
        row[col] = (row[col] - mean) / std;
      });
    });
    return data;
  }

  calculateMedian(arr) {
    const sorted = [...arr].sort((a, b) => a - b);
    const mid = Math.floor(sorted.length / 2);
    return sorted.length % 2 ? sorted[mid] : (sorted[mid - 1] + sorted[mid]) / 2;
  }
}

// Usage example
const engineer = new FeatureEngineer();
let userData = [
  { date: '2026-03-26T10:30:00', category: 'premium', spend: 150, age: null },
  { date: '2026-03-25T15:45:00', category: 'basic', spend: 50, age: 28 }
];

userData = engineer.handleMissingValues(userData, 'age');
userData = engineer.createTimeFeatures(userData, 'date');
userData = engineer.oneHotEncode(userData, 'category');
userData = engineer.standardScale(userData, ['spend', 'age']);
```

## ⚖️ When To Use / When To Avoid

**✅ Use Feature Engineering When:**
• Working with structured data where domain knowledge can guide feature creation
• Model performance is plateauing and you need to extract more signal
• Dealing with categorical variables, missing data, or mixed data types
• You have sufficient data to validate that new features generalize well

**❌