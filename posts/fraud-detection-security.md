# Fraud Detection & Data Security: Real-Time Threat Detection

*Published: January 2025*

## Overview

Fraud detection and data security are critical components of modern data platforms. This article explores production implementations of real-time fraud detection systems, security architectures, and anomaly detection using machine learning and rule-based approaches.

## The Fraud Detection Challenge

Organizations must detect:
- **Transaction fraud** in real-time
- **Account takeover** attempts
- **Identity theft** patterns
- **Payment fraud** across channels
- **Anomalous behavior** patterns

## Fraud Detection Architecture

A comprehensive fraud detection system includes:

```
Transactions → Feature Engineering → ML Models → Rule Engine → Decision Engine → Actions
      ↓                ↓                  ↓             ↓              ↓            ↓
  Real-Time      Feature Store      Scoring      Business Rules   Risk Score   Block/Allow
  Streaming      Historical Data   Models       Validation       Threshold    Alert
```

## Use Case 1: Real-Time Fraud Detection with Airbyte

### Business Requirements
An e-commerce platform needed a fraud detection system to:
- Process 100K+ transactions per minute
- Detect fraud in < 100ms
- Integrate data from multiple sources
- Support both rule-based and ML-based detection
- Maintain low false positive rate

### Implementation Architecture

**Data Integration with Airbyte:**
```
Data Sources → Airbyte Connectors → Data Pipeline → Feature Store → Fraud Detection
      ↓               ↓                  ↓               ↓                ↓
  Transactions    PostgreSQL        ETL Pipeline    ML Features     ML Models
  User Activity    MongoDB          Data Quality    Historical      Rules Engine
  Payment Data     APIs              Enrichment      Aggregations    Alerting
```

**Key Components:**

#### 1. Feature Engineering Pipeline
```python
class FraudFeatureEngineering:
    def __init__(self):
        self.feature_store = FeatureStore()
        self.historical_aggregator = HistoricalAggregator()
    
    def extract_features(self, transaction):
        """Extract features for fraud detection"""
        features = {}
        
        # Transaction features
        features['amount'] = transaction.amount
        features['amount_log'] = np.log(transaction.amount + 1)
        features['time_of_day'] = transaction.timestamp.hour
        features['day_of_week'] = transaction.timestamp.weekday()
        features['is_weekend'] = 1 if transaction.timestamp.weekday() >= 5 else 0
        features['is_night'] = 1 if transaction.timestamp.hour < 6 else 0
        
        # User historical features
        user_features = self.get_user_historical_features(transaction.user_id)
        features.update(user_features)
        
        # Device features
        device_features = self.get_device_features(transaction.device_id)
        features.update(device_features)
        
        # Location features
        location_features = self.get_location_features(transaction)
        features.update(location_features)
        
        # Behavioral features
        behavioral_features = self.get_behavioral_features(transaction)
        features.update(behavioral_features)
        
        return features
    
    def get_user_historical_features(self, user_id):
        """Get historical features for user"""
        # Last 24 hours
        last_24h = self.feature_store.get_features(
            entity_id=user_id,
            feature_names=[
                'transaction_count_24h',
                'total_amount_24h',
                'avg_transaction_amount_24h',
                'unique_merchants_24h'
            ],
            lookback_hours=24
        )
        
        # Last 7 days
        last_7d = self.feature_store.get_features(
            entity_id=user_id,
            feature_names=[
                'transaction_count_7d',
                'total_amount_7d',
                'chargeback_count_7d',
                'refund_rate_7d'
            ],
            lookback_hours=168
        )
        
        # Last 30 days
        last_30d = self.feature_store.get_features(
            entity_id=user_id,
            feature_names=[
                'transaction_count_30d',
                'total_amount_30d',
                'unique_locations_30d',
                'velocity_score_30d'
            ],
            lookback_hours=720
        )
        
        return {**last_24h, **last_7d, **last_30d}
    
    def get_behavioral_features(self, transaction):
        """Get behavioral features"""
        # Velocity features
        velocity_features = self.calculate_velocity(transaction)
        
        # Pattern features
        pattern_features = self.detect_patterns(transaction)
        
        # Anomaly features
        anomaly_features = self.detect_anomalies(transaction)
        
        return {
            **velocity_features,
            **pattern_features,
            **anomaly_features
        }
```

#### 2. ML Model Scoring
```python
class FraudMLModel:
    def __init__(self):
        self.models = {
            'xgboost': self.load_model('fraud_xgboost'),
            'random_forest': self.load_model('fraud_random_forest'),
            'neural_network': self.load_model('fraud_nn')
        }
        self.ensemble = EnsembleModel(self.models)
    
    def score_transaction(self, features):
        """Score transaction for fraud probability"""
        # Individual model predictions
        predictions = {}
        for model_name, model in self.models.items():
            predictions[model_name] = model.predict_proba(features)[0][1]
        
        # Ensemble prediction
        ensemble_score = self.ensemble.predict(features)
        
        # Feature importance
        feature_importance = self.get_feature_importance(features)
        
        return {
            'fraud_probability': ensemble_score,
            'model_predictions': predictions,
            'feature_importance': feature_importance,
            'top_risk_factors': self.get_top_risk_factors(feature_importance)
        }
```

#### 3. Rule-Based Detection
```python
class FraudRuleEngine:
    def __init__(self):
        self.rules = self.load_rules()
        self.rule_engine = RulesEngine()
    
    def evaluate_rules(self, transaction):
        """Evaluate fraud detection rules"""
        rule_results = []
        
        # Velocity rules
        if transaction.amount > self.get_user_avg(transaction.user_id) * 5:
            rule_results.append({
                'rule': 'high_amount_deviation',
                'severity': 'high',
                'triggered': True
            })
        
        # Location rules
        if self.is_unusual_location(transaction):
            rule_results.append({
                'rule': 'unusual_location',
                'severity': 'medium',
                'triggered': True
            })
        
        # Time-based rules
        if self.is_unusual_time(transaction):
            rule_results.append({
                'rule': 'unusual_time',
                'severity': 'low',
                'triggered': True
            })
        
        # Device rules
        if self.is_new_device(transaction.user_id, transaction.device_id):
            rule_results.append({
                'rule': 'new_device',
                'severity': 'medium',
                'triggered': True
            })
        
        return rule_results
    
    def is_unusual_location(self, transaction):
        """Check if transaction is from unusual location"""
        user_locations = self.get_user_locations(transaction.user_id, days=30)
        current_location = (transaction.latitude, transaction.longitude)
        
        # Check if location is far from user's usual locations
        min_distance = min([
            self.calculate_distance(current_location, loc)
            for loc in user_locations
        ])
        
        return min_distance > 1000  # More than 1000km
```

#### 4. Decision Engine
```python
class FraudDecisionEngine:
    def __init__(self):
        self.ml_model = FraudMLModel()
        self.rule_engine = FraudRuleEngine()
        self.alert_system = AlertSystem()
    
    def make_decision(self, transaction):
        """Make fraud decision for transaction"""
        # Extract features
        features = self.extract_features(transaction)
        
        # ML model scoring
        ml_score = self.ml_model.score_transaction(features)
        
        # Rule-based evaluation
        rule_results = self.rule_engine.evaluate_rules(transaction)
        
        # Combine scores
        fraud_score = self.combine_scores(ml_score, rule_results)
        
        # Make decision
        if fraud_score > 0.9:
            decision = 'BLOCK'
            action = self.block_transaction(transaction)
        elif fraud_score > 0.7:
            decision = 'REVIEW'
            action = self.flag_for_review(transaction)
        elif fraud_score > 0.5:
            decision = 'MONITOR'
            action = self.monitor_transaction(transaction)
        else:
            decision = 'ALLOW'
            action = self.allow_transaction(transaction)
        
        # Log decision
        self.log_decision(transaction, fraud_score, decision, action)
        
        # Alert if high risk
        if fraud_score > 0.8:
            self.alert_system.send_alert({
                'transaction_id': transaction.id,
                'fraud_score': fraud_score,
                'decision': decision,
                'risk_factors': self.get_risk_factors(ml_score, rule_results)
            })
        
        return {
            'decision': decision,
            'fraud_score': fraud_score,
            'action': action,
            'ml_score': ml_score['fraud_probability'],
            'rule_triggers': len([r for r in rule_results if r['triggered']])
        }
    
    def combine_scores(self, ml_score, rule_results):
        """Combine ML and rule-based scores"""
        # ML score weight
        ml_weight = 0.7
        
        # Rule-based score
        rule_score = 0.0
        if rule_results:
            high_severity = len([r for r in rule_results if r['severity'] == 'high'])
            medium_severity = len([r for r in rule_results if r['severity'] == 'medium'])
            rule_score = min(1.0, (high_severity * 0.3 + medium_severity * 0.1))
        
        # Combined score
        combined_score = ml_score['fraud_probability'] * ml_weight + rule_score * (1 - ml_weight)
        
        return combined_score
```

**Results:**
- **100K+ transactions/minute** processed
- **< 100ms** detection latency
- **95% fraud detection** accuracy
- **< 2% false positive** rate
- **Real-time** decision making

## Use Case 2: E-commerce Fraud Detection Platform

### Business Requirements
An e-commerce platform needed comprehensive fraud detection for:
- Payment fraud detection
- Account takeover prevention
- Card testing detection
- Refund fraud prevention
- Chargeback reduction

### Implementation

**Multi-Layer Fraud Detection:**
```python
class EcommerceFraudDetection:
    def __init__(self):
        self.payment_fraud_detector = PaymentFraudDetector()
        self.account_takeover_detector = AccountTakeoverDetector()
        self.card_testing_detector = CardTestingDetector()
        self.refund_fraud_detector = RefundFraudDetector()
    
    def detect_fraud(self, transaction):
        """Comprehensive fraud detection"""
        fraud_indicators = []
        
        # Payment fraud
        payment_fraud = self.payment_fraud_detector.detect(transaction)
        if payment_fraud.detected:
            fraud_indicators.append({
                'type': 'payment_fraud',
                'score': payment_fraud.score,
                'reasons': payment_fraud.reasons
            })
        
        # Account takeover
        account_takeover = self.account_takeover_detector.detect(transaction)
        if account_takeover.detected:
            fraud_indicators.append({
                'type': 'account_takeover',
                'score': account_takeover.score,
                'reasons': account_takeover.reasons
            })
        
        # Card testing
        card_testing = self.card_testing_detector.detect(transaction)
        if card_testing.detected:
            fraud_indicators.append({
                'type': 'card_testing',
                'score': card_testing.score,
                'reasons': card_testing.reasons
            })
        
        # Refund fraud
        if transaction.type == 'refund':
            refund_fraud = self.refund_fraud_detector.detect(transaction)
            if refund_fraud.detected:
                fraud_indicators.append({
                    'type': 'refund_fraud',
                    'score': refund_fraud.score,
                    'reasons': refund_fraud.reasons
                })
        
        # Calculate overall risk
        overall_risk = self.calculate_overall_risk(fraud_indicators)
        
        return {
            'fraud_detected': len(fraud_indicators) > 0,
            'overall_risk_score': overall_risk,
            'fraud_indicators': fraud_indicators
        }
```

**Results:**
- **60% reduction** in chargebacks
- **80% detection** rate for account takeover
- **90% detection** rate for card testing
- **Real-time** fraud prevention

## Use Case 3: Data Security & Access Control

### Business Requirements
An organization needed comprehensive data security for:
- Data encryption at rest and in transit
- Role-based access control
- Audit logging
- Data masking
- Compliance with regulations

### Implementation

**Security Architecture:**
```python
class DataSecurityFramework:
    def __init__(self):
        self.encryption_service = EncryptionService()
        self.access_controller = AccessController()
        self.audit_logger = AuditLogger()
        self.data_masking = DataMaskingService()
    
    def secure_data(self, data, classification):
        """Apply security measures based on data classification"""
        secured_data = data.copy()
        
        # Encrypt sensitive data
        if classification['sensitivity'] in ['high', 'critical']:
            secured_data = self.encryption_service.encrypt(
                secured_data,
                fields=classification['sensitive_fields'],
                algorithm='AES-256'
            )
        
        # Apply access control
        access_policies = self.access_controller.get_policies(classification)
        secured_data['access_policies'] = access_policies
        
        return secured_data
    
    def check_access(self, user, resource, action):
        """Check if user has access to resource"""
        # Get user roles
        user_roles = self.get_user_roles(user)
        
        # Check policies
        for role in user_roles:
            policy = self.access_controller.get_policy(role, resource)
            if policy.allows(action):
                # Log access
                self.audit_logger.log_access(
                    user=user,
                    resource=resource,
                    action=action,
                    granted=True
                )
                return True
        
        # Log denied access
        self.audit_logger.log_access(
            user=user,
            resource=resource,
            action=action,
            granted=False
        )
        return False
    
    def mask_data(self, data, user_role):
        """Apply data masking based on user role"""
        if user_role == 'analyst':
            # Mask PII
            masked_data = self.data_masking.mask_pii(data)
        elif user_role == 'compliance':
            # Full access
            masked_data = data
        else:
            # Aggregate only
            masked_data = self.data_masking.aggregate(data)
        
        return masked_data
```

## Technical Best Practices

### 1. Real-Time Feature Engineering

```python
class RealTimeFeatureEngineering:
    def calculate_velocity_features(self, transaction):
        """Calculate velocity features in real-time"""
        # Transaction velocity (count per time period)
        velocity_1h = self.get_transaction_count(
            user_id=transaction.user_id,
            time_window=timedelta(hours=1)
        )
        
        velocity_24h = self.get_transaction_count(
            user_id=transaction.user_id,
            time_window=timedelta(hours=24)
        )
        
        # Amount velocity
        amount_velocity_24h = self.get_total_amount(
            user_id=transaction.user_id,
            time_window=timedelta(hours=24)
        )
        
        return {
            'txn_velocity_1h': velocity_1h,
            'txn_velocity_24h': velocity_24h,
            'amount_velocity_24h': amount_velocity_24h
        }
```

### 2. Model Retraining Pipeline

```python
class ModelRetrainingPipeline:
    def retrain_fraud_model(self):
        """Retrain fraud detection model"""
        # Get labeled data
        training_data = self.get_labeled_transactions(period_days=90)
        
        # Feature engineering
        features = self.extract_features(training_data)
        
        # Train model
        model = self.train_model(features, training_data.labels)
        
        # Evaluate model
        evaluation = self.evaluate_model(model, test_data)
        
        # A/B test
        if evaluation.accuracy > self.current_model.accuracy:
            self.deploy_model(model)
        
        return model
```

### 3. Adaptive Thresholds

```python
class AdaptiveThresholds:
    def adjust_thresholds(self, performance_metrics):
        """Adjust fraud detection thresholds based on performance"""
        if performance_metrics.false_positive_rate > 0.05:
            # Too many false positives, increase threshold
            new_threshold = self.current_threshold * 1.1
        elif performance_metrics.detection_rate < 0.90:
            # Missing fraud, decrease threshold
            new_threshold = self.current_threshold * 0.9
        else:
            new_threshold = self.current_threshold
        
        return new_threshold
```

## Related Projects

- [Fraud Detection with Airbyte](../projects/fraud_detection_airbyte/)
- [E-commerce Fraud Detection](../projects/ecommerce-fraud-detection/)

## Conclusion

Fraud detection and data security require a multi-layered approach combining ML models, rule-based systems, real-time processing, and comprehensive security measures. Key success factors include real-time feature engineering, ensemble models, adaptive thresholds, and continuous monitoring.

**Key Takeaways:**
1. Real-time feature engineering enables fast fraud detection
2. Combining ML and rule-based approaches improves accuracy
3. Ensemble models reduce false positives
4. Adaptive thresholds optimize performance over time
5. Comprehensive security frameworks protect sensitive data

---

**Next Steps:**
- [Financial Services Data Engineering](./financial-services-data-engineering.md)
- [Data Governance & Quality](./data-governance-quality.md)

