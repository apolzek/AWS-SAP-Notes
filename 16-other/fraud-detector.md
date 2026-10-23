# Amazon Fraud Detector

> **Status (2026):** Amazon Fraud Detector is **no longer open to new customers** (since Nov 7, 2025, maintenance mode). Existing customers can continue to use it. AWS alternatives: **AWS WAF Fraud Control** (ATP - account takeover prevention, ACFP - account creation fraud prevention) for login/sign-up fraud, and **Amazon SageMaker** / AutoGluon for custom fraud models.

- Is a fully managed fraud detection service
- Allows us to look at various historical trends and other related data and identify any potential fraud as it related to certain only activities such as new account creation, payments or guest checkouts
- We upload some historical data and we have to chose a model type:
    - Online Fraud (Online Fraud Insights): needs little historical data
    - Transaction Fraud: ideal when we have a transactional history for a customer, identifies suspect payments
    - Account Takeover (Account Takeover Insights): used to identify phishing or another social based attack (compromised credentials on login events)
- Events are scored, based on which we can create rules/decisions to react to events according to our business activity
- Concepts: event types, entities, variables, labels (fraud/legit), models, rules (written in a simple rule language using model scores), outcomes (e.g. `approve`, `review`, `block`), detectors (versioned)
- Real-time prediction with `GetEventPrediction` API; batch predictions from S3
- Can include external SageMaker models in a detector
- Exam (current): "block fake account sign-ups / credential stuffing on login page" -> WAF Fraud Control ACFP/ATP; "custom fraud scoring model on transactions" -> SageMaker
