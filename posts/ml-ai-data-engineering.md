# Production ML Pipelines: From Data to Deployed Models

*Published: January 2025*

## Overview

Machine learning and AI have become integral to modern data engineering. This article explores production ML pipelines covering the entire ML lifecycle: data preparation, feature engineering, model training, deployment, and monitoring.

## The ML Data Engineering Challenge

Building production ML systems requires:
- **Data pipelines** for training and inference
- **Feature engineering** at scale
- **Model versioning** and management
- **A/B testing** capabilities
- **Monitoring** for model drift and performance

## ML Pipeline Architecture

A complete ML pipeline follows the MLOps lifecycle:

```
Data → Feature Engineering → Model Training → Validation → Deployment → Monitoring
  ↓              ↓                  ↓             ↓            ↓            ↓
Ingestion    Feature Store    Experimentation  Testing    Serving    Observability
```

## Use Case 1: ML ETL Pipeline Platform

### Business Requirements
A data science organization needed an end-to-end ML platform to:
- Process training data from multiple sources
- Automate feature engineering
- Enable model experimentation
- Deploy models to production
- Monitor model performance

### Implementation

**Pipeline Architecture:**
```
Data Sources → Data Pipeline → Feature Store → Model Training → Model Registry → Serving
      ↓             ↓              ↓               ↓                ↓             ↓
  Databases      ETL Jobs      Feature API    MLflow/AML      Versioning    REST API
  APIs          Validation     Caching        Experiments     Metadata      Batch/Real-time
```

**Key Components:**

#### 1. Feature Engineering Pipeline
```python
class FeatureEngineering:
    def create_features(self, raw_data):
        features = {}
        
        # Temporal features
        features['day_of_week'] = raw_data['date'].dt.dayofweek
        features['month'] = raw_data['date'].dt.month
        features['is_weekend'] = (features['day_of_week'] >= 5).astype(int)
        
        # Aggregation features
        features['rolling_avg_7d'] = raw_data.groupby('customer_id')[
            'amount'].rolling(window=7).mean()
        
        # Interaction features
        features['price_per_unit'] = raw_data['total'] / raw_data['quantity']
        
        return features
```

#### 2. Feature Store
```python
class FeatureStore:
    def __init__(self):
        self.store = {}  # Redis or database
    
    def save_features(self, entity_id, features, timestamp):
        key = f"{entity_id}:{timestamp}"
        self.store[key] = features
    
    def get_features(self, entity_id, timestamp=None):
        if timestamp:
            key = f"{entity_id}:{timestamp}"
            return self.store.get(key)
        # Return latest
        return self.store.get_latest(entity_id)
```

#### 3. Model Training Pipeline
```python
import mlflow

def train_model(X_train, y_train, experiment_name):
    with mlflow.start_run(experiment_id=experiment_name):
        # Model training
        model = XGBRegressor(
            n_estimators=100,
            max_depth=6,
            learning_rate=0.1
        )
        model.fit(X_train, y_train)
        
        # Evaluate
        predictions = model.predict(X_train)
        rmse = mean_squared_error(y_train, predictions, squared=False)
        
        # Log metrics
        mlflow.log_metric("rmse", rmse)
        mlflow.log_param("n_estimators", 100)
        
        # Log model
        mlflow.xgboost.log_model(model, "model")
        
        return model
```

#### 4. Model Serving
```python
from flask import Flask, request, jsonify
import mlflow.pyfunc

app = Flask(__name__)

# Load model
model_path = "models://production/1"
model = mlflow.pyfunc.load_model(model_path)

@app.route('/predict', methods=['POST'])
def predict():
    data = request.json
    features = prepare_features(data)
    prediction = model.predict(features)
    return jsonify({'prediction': prediction.tolist()})
```

**Results:**
- **70% reduction** in model deployment time
- **Automated** feature engineering
- **Model versioning** and tracking
- **A/B testing** capabilities

## Use Case 2: Snowflake ML AI Platform

### Business Requirements
An organization wanted to leverage Snowflake's built-in ML capabilities for:
- In-database ML training
- Real-time inference
- Feature engineering at scale
- Model sharing across teams

### Implementation

**Snowflake ML Pipeline:**
```
Data → Snowflake → Feature Engineering (SQL) → Model Training (Snowpark) → Model Serving
  ↓         ↓              ↓                          ↓                         ↓
Sources   Storage    SQL UDFs/Procedures        Python Functions          REST Endpoints
```

**Key Features:**

#### 1. In-Database Feature Engineering
```sql
-- Feature engineering with SQL
CREATE OR REPLACE VIEW customer_features AS
SELECT 
    customer_id,
    COUNT(DISTINCT order_id) as total_orders,
    SUM(amount) as total_spent,
    AVG(amount) as avg_order_value,
    MAX(order_date) as last_order_date,
    DATEDIFF('day', MAX(order_date), CURRENT_DATE()) as days_since_last_order,
    -- Lag features
    LAG(amount, 1) OVER (PARTITION BY customer_id ORDER BY order_date) as prev_order_amount
FROM orders
GROUP BY customer_id;
```

#### 2. Model Training with Snowpark
```python
from snowflake.snowpark import Session
from snowflake.snowpark.functions import col
import xgboost as xgb

session = Session.builder.configs(connection_parameters).create()

# Load training data
train_df = session.table("customer_features")

# Convert to pandas for model training
train_pd = train_df.to_pandas()

# Train model
X_train = train_pd.drop('target', axis=1)
y_train = train_pd['target']

model = xgb.XGBClassifier()
model.fit(X_train, y_train)

# Save model
model.save_model('@models/churn_model.pkl')
```

#### 3. Model Inference
```sql
-- In-database inference
CREATE OR REPLACE FUNCTION predict_churn(customer_id INT)
RETURNS FLOAT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.8'
HANDLER = 'predict'
PACKAGES = ('xgboost', 'pandas')
AS $$
import xgboost as xgb
import pandas as pd

model = xgb.XGBClassifier()
model.load_model('/models/churn_model.pkl')

def predict(session, customer_id):
    # Get features for customer
    features = session.table("customer_features") \
        .filter(col("customer_id") == customer_id) \
        .to_pandas()
    
    # Predict
    prediction = model.predict_proba(features)[0][1]
    return prediction
$$;
```

**Results:**
- **No data movement** - ML in Snowflake
- **Fast inference** with in-database processing
- **Scalable** to millions of predictions
- **Integrated** with existing data warehouse

## Use Case 3: AI-Driven Retail Prediction Platform

### Business Requirements
A retail analytics company needed ML models for:
- Demand forecasting for 10,000+ SKUs
- Inventory optimization
- Price optimization
- Promotion effectiveness analysis

### Implementation

**ML Models:**

#### 1. Prophet Time Series Forecasting
```python
from prophet import Prophet

def forecast_demand(product_data):
    # Prepare data for Prophet
    df = product_data[['date', 'sales']].rename(
        columns={'date': 'ds', 'sales': 'y'}
    )
    
    # Initialize and fit model
    model = Prophet(
        yearly_seasonality=True,
        weekly_seasonality=True,
        daily_seasonality=False,
        holidays=holiday_df
    )
    model.fit(df)
    
    # Generate forecast
    future = model.make_future_dataframe(periods=30)
    forecast = model.predict(future)
    
    return forecast[['ds', 'yhat', 'yhat_lower', 'yhat_upper']]
```

#### 2. XGBoost Demand Prediction
```python
import xgboost as xgb

def train_demand_model(features, target):
    # Feature engineering
    features_engineered = create_features(features)
    
    # Train model
    model = xgb.XGBRegressor(
        n_estimators=200,
        max_depth=8,
        learning_rate=0.05,
        subsample=0.8,
        colsample_bytree=0.8
    )
    
    model.fit(features_engineered, target)
    
    # Feature importance
    importance = model.feature_importances_
    
    return model, importance
```

#### 3. Inventory Optimization Model
```python
def optimize_inventory(demand_forecast, current_stock, lead_time):
    # Calculate optimal stock levels
    safety_stock = calculate_safety_stock(
        demand_std=demand_forecast.std(),
        lead_time=lead_time,
        service_level=0.95
    )
    
    reorder_point = demand_forecast.mean() * lead_time + safety_stock
    order_quantity = calculate_eoq(
        demand=demand_forecast.mean(),
        ordering_cost=100,
        holding_cost=0.2
    )
    
    return {
        'reorder_point': reorder_point,
        'order_quantity': order_quantity,
        'safety_stock': safety_stock
    }
```

**Results:**
- **35% improvement** in forecast accuracy
- **20% reduction** in inventory costs
- **15% increase** in sales
- **Automated** inventory recommendations

## Technical Deep Dive

### Feature Engineering Best Practices

#### 1. Temporal Features
```python
def create_temporal_features(df, date_col):
    df['year'] = df[date_col].dt.year
    df['month'] = df[date_col].dt.month
    df['day'] = df[date_col].dt.day
    df['day_of_week'] = df[date_col].dt.dayofweek
    df['is_weekend'] = (df['day_of_week'] >= 5).astype(int)
    df['is_month_end'] = df[date_col].dt.is_month_end.astype(int)
    return df
```

#### 2. Aggregation Features
```python
def create_aggregation_features(df, group_col, value_col):
    agg_features = df.groupby(group_col)[value_col].agg([
        'mean', 'std', 'min', 'max', 'sum', 'count'
    ]).add_suffix(f'_{value_col}')
    return agg_features
```

#### 3. Lag Features
```python
def create_lag_features(df, value_col, lags=[1, 7, 30]):
    for lag in lags:
        df[f'{value_col}_lag_{lag}'] = df.groupby('entity_id')[
            value_col].shift(lag)
    return df
```

### Model Monitoring

#### 1. Data Drift Detection
```python
from evidently import DataDriftProfile

def detect_data_drift(reference_data, current_data):
    drift_profile = DataDriftProfile()
    drift_profile.calculate(reference_data, current_data)
    
    if drift_profile.get_metrics()['dataset_drift']:
        alert('Data drift detected!')
        return True
    return False
```

#### 2. Model Performance Monitoring
```python
def monitor_model_performance(predictions, actuals):
    metrics = {
        'rmse': mean_squared_error(actuals, predictions, squared=False),
        'mae': mean_absolute_error(actuals, predictions),
        'r2': r2_score(actuals, predictions)
    }
    
    # Alert if performance degrades
    if metrics['rmse'] > threshold:
        alert('Model performance degraded!')
    
    return metrics
```

#### 3. Prediction Distribution Monitoring
```python
def monitor_prediction_distribution(current_predictions, reference_predictions):
    ks_statistic = ks_2samp(
        reference_predictions, 
        current_predictions
    )
    
    if ks_statistic.pvalue < 0.05:
        alert('Prediction distribution changed significantly!')
```

## Best Practices

### 1. Feature Store
- Centralize feature definitions
- Enable feature reuse
- Track feature lineage
- Monitor feature quality

### 2. Experiment Tracking
- Use MLflow or similar tools
- Track all hyperparameters
- Log all experiments
- Compare model versions

### 3. Model Versioning
- Version control for models
- Track model metadata
- Enable rollback capabilities
- A/B testing framework

### 4. Monitoring
- Track data drift
- Monitor model performance
- Alert on anomalies
- Regular model retraining

## Related Projects

- [ML ETL Pipeline](../projects/ml_etl_pipeline/)
- [Snowflake ML AI Platform](../projects/snowflake_ml_ai_platform/)
- [AI-Driven Retail Prediction](../projects/ai-driven-retail-prediction/)
- [Real Estate AI & NLP](../projects/real-estate-ai-nlp/)

## Conclusion

Production ML pipelines require careful engineering to handle the entire ML lifecycle. Key success factors include robust feature engineering, model versioning, automated deployment, and comprehensive monitoring.

**Key Takeaways:**
1. Feature stores enable feature reuse and consistency
2. Experiment tracking is essential for model development
3. Model monitoring detects drift and performance issues
4. Automated pipelines reduce deployment time
5. ML in data warehouses enables faster insights

---

**Next Steps:**
- [Data Modeling & Architecture](./data-modeling-architecture.md)
- [Business Intelligence & Reporting](./bi-reporting.md)

