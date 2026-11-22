# ADR-007: ML Strategy and MLOps Architecture

- **Status:** Accepted
- **Date:** 2026-11-19

## Context

NetGuard must detect abnormal network behavior from incoming telemetry while supporting model training, evaluation, versioning, deployment and monitoring.

The ML architecture must support:
- Initial anomaly detection without depending on a large labeled dataset.
- Reproducible experiments and model comparisons.
- Independent model deployment without requiring changes to the platform API.
- Controlled model promotion and rollback.
- Low infrastructure costs, particularly during local development and initial Azure deployment.

We must also distinguish between two workflows:

- **Training and evaluation:** offline operations using historical and synthetic datasets.
- **Inference:** continuous predictions on incoming processed telemetry.

## Decision

### 1. Initial ML strategy

NetGuard will initially use a hybrid anomaly detection approach:

- **Rule-based detection:** identify known critical conditions using configurable thresholds.
- **Unsupervised ML:** use Isolation Forest to identify unusual patterns in telemetry, including combinations of metrics that may not be covered by predefined rules.

Supervised ML algorithms may be introduced later when sufficiently representative and reliable labeled data become available.

Synthetic data will help generate additional scenarios and test detection behavior, but performance on synthetic data alone will not be considered proof of real-world generalization.

### 2. Training and evaluation

Model training will be performed separately from production inference.

The initial workflow will:
1. Prepare historical and synthetic telemetry datasets.
2. Separate training and evaluation observations, respecting temporal ordering where appropriate.
3. Train candidate models.
4. Evaluate models on observations not used for training.
5. Compare model quality and operational suitability.
6. Register successful candidates in MLflow.

Evaluation will consider relevant metrics, including false positive rate, false negative rate, precision, recall and generalization to unseen observations.

### 3. Experiment tracking and model management

NetGuard will use **MLflow** for experiment tracking and model management.

MLflow will provide:
- Experiment tracking, including parameters, metrics and results.
- Model artifact management.
- Model versioning and registry.
- Traceability between experiments and registered models.

Initial storage configuration:

- **Backend store:** PostgreSQL, using a dedicated database on the existing PostgreSQL instance.
- **Artifact store:** local filesystem, backed by persistent storage in Kubernetes.

Azure object storage may be considered later if deployment requirements justify it.

### 4. Model inference architecture

A dedicated ML inference worker will consume processed telemetry from Kafka.

Its responsibilities will include:
1. Loading the currently approved production model from MLflow.
2. Consuming events from the `telemetry.processed` topic.
3. Extracting the required features.
4. Performing predictions.
5. Publishing detected anomalies to the `anomalies.detected` topic.

The worker will not train models during normal inference operations.

It will normally use one approved production model at a time, rather than evaluating every registered model for every observation.

### 5. Model promotion and deployment

New models will not automatically become production models when registered.

The promotion workflow will be:

1. Train and register a candidate model.
2. Evaluate it against defined acceptance criteria.
3. Explicitly approve and promote the candidate.
4. Allow the inference worker to detect the new approved version.
5. Load and validate the new model before switching.

The worker will use **dynamic model loading**, avoiding a full worker restart when switching model versions.

The transition must preserve the currently active model until the new model has loaded successfully. If loading fails, the worker continues using the previous version.

Previous production versions will remain available for rollback.

### 6. Model and worker monitoring

Model quality evaluation and production monitoring will be treated as separate concerns.

During model development and evaluation, we will measure:
- False positive and false negative rates.
- Precision and recall.
- Generalization to unseen observations.

During production inference, we will monitor:
- Inference latency.
- Prediction throughput.
- Prediction errors.
- Prediction distribution.
- Input data drift.

Production quality metrics such as false positive and false negative rates can also be evaluated when reliable labels become available for incoming observations.

Prometheus and Grafana will be used for operational monitoring. MLflow will manage experiment and model information.

## Alternatives Considered

### A. Rule-based detection only

**Advantages:**
- Simple to implement.
- Explainable decisions.
- No model training or lifecycle management.

**Disadvantages:**
- Fixed thresholds cannot capture every unusual combination of network metrics.
- Limited ability to adapt to different network baselines.

### B. Supervised ML as the initial strategy

**Advantages:**
- Can learn from labeled examples.
- Supports quantitative evaluation when reliable labels are available.

**Disadvantages:**
- Requires representative labeled data.
- May fail to identify anomaly types absent from training data.
- Synthetic labels may not fully represent real network behavior.

### C. Unsupervised ML only

**Advantages:**
- Does not require labeled training observations.
- Can identify unusual patterns not explicitly defined by rules.

**Disadvantages:**
- Unusual observations are not necessarily real failures.
- Can generate false positives.
- Requires careful evaluation and interpretation.

### D. Hybrid rules and unsupervised ML — Selected

Combines explicit detection of known critical conditions with the ability to identify less obvious unusual patterns.

This provides a practical initial strategy while allowing supervised ML to be introduced later.

## Consequences

### Positive consequences

- Detection does not depend entirely on labeled datasets.
- Known critical conditions can be detected explicitly.
- ML experiments and model versions are traceable.
- Model inference is independently scalable.
- Model updates do not require restarting the inference worker.
- Previous model versions support rollback.
- Initial ML infrastructure avoids requiring expensive managed cloud services.

### Negative consequences

- Maintaining both rules and ML models increases implementation complexity.
- Unsupervised detection requires careful interpretation and evaluation.
- MLflow introduces an additional service to operate.
- Dynamic model loading requires careful handling of model transitions and failures.
- Local artifact storage requires persistent-volume management and appropriate backups.

## Future Considerations

- Introduce supervised algorithms when reliable labeled data becomes available.
- Evaluate advanced time-series anomaly detection and forecasting.
- Consider Azure Blob Storage for model artifacts.
- Introduce shadow inference or canary model deployment if model updates require safer production validation.
- Establish automated model drift detection and retraining workflows when sufficient data and evaluation processes are available.