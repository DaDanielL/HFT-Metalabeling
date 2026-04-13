# **Learning When to Trade**

### *Meta-Labeling for Signal Reliability in High-Frequency Markets*

---

## Overview

This project builds a **production-style trading signal pipeline** using high-frequency futures data.

Instead of only predicting direction, we answer a more important question:

> ❝ *When should we trust a trading signal?* ❞

The system combines:

* a **base signal model** (direction prediction)
* a **meta-labeling model** (trade filtering)

---

## End-to-End Pipeline

```
Raw Tick Data (L3)
        ↓
Data Cleaning & Ordering
        ↓
Event-Based Sampling (Volume Bars)
        ↓
Feature Engineering
        ↓
Base Signal Model (Direction)
        ↓
Walk-Forward Validation → OOS Predictions
        ↓
Meta-Labeling (Trade Quality)
        ↓
Final Strategy Evaluation
```

---

# 1. Spark Setup & Data Infrastructure

To handle large-scale nanosecond-level market data (~50GB), the pipeline is built on:

* **Apache Spark (PySpark)**
* Distributed processing (EMR / cluster-ready)
* Columnar storage (Parquet)

This enables scalable:

* data ingestion
* transformation
* feature computation
  
---

# 2. Data Loading

We ingest **market-by-order (L3) futures data**, including:

* timestamps (`ts_event`)
* price
* size (volume)
* order events (add, cancel, modify, fill)

Focus is placed on a **single instrument (e.g., CL futures)** for consistency.

---

# 3. Data Cleaning

Cleaning Steps:

* removal of null values
* deduplication using sequence/order identifiers
* strict chronological ordering
* validation of event consistency

---

# 4. Event-Based Sampling (Volume Bars)

**Volume bars**:

```
Aggregate trades until volume threshold → create one bar
```
* reduces noise from irregular tick activity
* normalizes information flow
* produces more stable statistical properties

Each bar contains:

* OHLC prices
* total volume
* trade counts

---

# 5. Feature Engineering

Features are engineered at the **bar level** to capture market behavior.

### 🔹 Core categories

* **Price features** → returns, momentum
* **Volatility features** → rolling std, realized volatility
* **Microstructure features** → order imbalance, trade flow
* **Statistical features** → rolling means, z-scores
* **Interaction features** → nonlinear relationships

---

# 6. Base Signal Model (Direction)

The base model predicts:

```
side ∈ { -1 (down), +1 (up) }
```

Using:

* Logistic Regression (baseline)

Output:

* predicted direction
* prediction probability

---

# 7. Walk-Forward Validation (Critical Step)

To avoid **look-ahead bias**, we use time-aware validation:

```
Train on past → Predict next chunk → Repeat
```

Example:

```
Train: 1–40 → Predict: 41–50  
Train: 1–50 → Predict: 51–60  
...
```
* ensures predictions are **out-of-sample (OOS)**
* simulates real trading conditions
* prevents leakage from future data

---

# 8. Base Model Evaluation

All walk-forward predictions are combined into a single dataset:

* true direction
* predicted direction
* probabilities

We evaluate:

* accuracy, F1, AUC
* confusion matrix
* probability calibration

---

# 9. Meta-Labeling (Trade Filtering)

Instead of taking every signal, we learn **when signals are reliable**

---

## Meta Dataset Construction

The meta dataset is built from **out-of-sample base predictions**:

```
(features + base_pred + base_prob) → meta_label
```

Where:

* `meta_label = 1` → profitable trade
* `meta_label = 0` → unprofitable trade

---

## 🤖 Meta Model

A second model (e.g., Random Forest) learns:

> “Given this signal and market condition, should we take the trade?”

---

# 10. Strategy Simulation

We compare:

| Strategy | Description                |
| -------- | -------------------------- |
| Base     | Take all signals           |
| Meta     | Only take filtered signals |

Expected result:

> Fewer trades, but higher-quality signals

---

# 11. Final Evaluation

A final test set is kept **completely untouched**:

1. Train base model on full training data
2. Generate predictions on test
3. Apply meta-labeler
4. Evaluate performance
