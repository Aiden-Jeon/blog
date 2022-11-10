---
title: Chapter 8. Data Distribution Shifts and Monitoring
categories: [mlops]
tags: ["ml-system"]
toc: true
date: 2022-11-10
author: Jongseob Jeon
---

Summary of [Designing Machine Learning Systems](https://learning.oreilly.com/library/view/designing-machine-learning/9781098107956/) written by Chip Huyen.

---
# Chapter 8. Data Distribution Shifts and Monitoring

- Deploying a model isn’t the end of process
    - Model’s performance degrades over time in production
    - Once models has deployed, still have to continually monitor its performance to detect issue
        - → deploy updates to  fix issues

## 1. Causes of ML System Failures

- Failure
    - one or more expectations of the system is violated
- Traditional software
    - system’s operational expectations
        - system executes its logic with in expected operational metrics
        - latency, throughput
- ML System
    - operational expectations and ML performance metrics
        - operational expectation violates → easier to detect
        - ML performance metric violates → harder to detect

### 1. Software System Failures

1. Dependency failure
2. Deployment failure
3. Hardware failure
4. Downtime or Crashing

### 2. ML-Specific Failures

#### 1. Production data differing from training data

- model generalizes to unseen data
    - generate accurate predictions for unseen data
- Assumption: unseen data comes from a stationary distribution that is the same as the training data distribution
- → incorrect in most case
    1. underlying distribution of the real-world data is unlikely to be the same as the training data distribution
    2. the real-word isn’t stationary

#### 2. Edge cases

- edge cases: the data samples so extreme that cause the model to make catastrophic mistakes
- outlier vs edge case
    - outlier
        - refers to data
        - an example that differs significantly differs from other examples
    - edge case
        - refers to performance
        - an example where a model performs significantly worse than other examples

### 3. Degenerate Feedback Loops

- feedback loop
    - the time it takes from when a prediction is show until the time feedback on the prediction is provided.
- degenerate feedback loop
    - predictions themselves influence feedback → influences the next iteration of the model
    - created when systems’s outputs are used → to generate the system’s future inputs
        - ⇒ influence the system’s output
- eg) recommendation system

### 4. Detecting Degenerate Feedback Loop

### 5. Correcting Degenerate Feedback Loop

## 2. Data Distribution Shifts

- Data distribution shifts
    - phenomenon in supervised learning when the data a model works with changes over time
        - → causes this model’s prediction to become less accurate as time passes
- Source distribution
- Target distribution

### 1. Types of Data Distribution Shifts

1. Covariate Shift: when $P(X)$ changes but $P(Y|X)$ remains the same
2. Label Shift: when $P(Y)$ changes but $P(X|Y)$ remains the same
3. Concept Drift: when $P(Y|X)$ changes but $P(X)$ remains the same

#### 1. Covariate shift

- one of most widely studied forms
- model development
    - during data selection process
        1. difficult to collect data
        2. training data is artificially altered (under-sampling, over-sampling)
- model’s learning process
    - active learning
- In production
    - major change in
        - the environment
        - the way application is used

#### 2. Label shift

- a.k.a prior shift, target shift
- closely related to covariate shift, methods for detecting and adapting models are similar

#### 3. Concept Drift

- a.k.a posterior shift
- same input, different output
- usually cyclic or seasonal

### 2. General Data Distribution Shifts

- feature change
    - new features are added
    - old features are removed
    - set of all possible values of a feature changed
- Label schema change
    - set of possible value for Y change

### 3. Detecting Data Distribution Shifts

- monitoring model’s accuracy-related metrics
- Input
- Output
- Joint dist

#### 1. Statistical method

1. compare statistics
2. two-sample hypothesis test (two-sample test)
    - Kolmogrov-Smirnov test (KS test)
        - non-parametric test
        - can used for one-dimensional data
3. Least-Square Density Difference
    - Maximum Mean Discrepancy (MMD)
    - Learned Kernel MMD

#### 2. Time scale windows for detecting shifts

- shifts across two dimensions:
    - spatial: happens across points
    - temporal: happens across time
        - → to detect: treat input data as time-series data

### 4. Addressing Data Distribution Shifts

- Assume data shifts are inevitable → periodically retrain their model
- To make a model work with a new distribution in production:
    1. Train models using massive datasets
    2. Adopt a trained model to a target distribution without new labels
        - Domain Adoption under Target and Conditional Shift
        - On Learning Invariant Representations for Domain Adoption
    3. Retrain model using the labeled data from the target distribution
        1. whether to
            1. train model from scratch (stateless training)
            2. continuing training the existing model (stateful training)
        2. what data to use

## 3. Monitoring and Observability

- monitoring
    - refers to act of tracking, measuring, and logging different metrics that can help us determine when something goes wrong
    - operational metrics: health of systems
        1. network
        2. machine
        3. applications
- observability
    - setting up our system (instrumentation) in a way that give us visibility into our system to help us investigate what meant wrong
    - part of monitoring

### 1. ML-Specific Metrics

- Types
    1. model accuracy-related metrics
    2. predictions
    3. features
    4. raw inputs
- from 1 to 4
    - easier to monitor ←→ harder to monitor
    - closer to business metrics ←→ less likely to be caused by human errors

#### 1. Monitoring accuracy-related metrics

- direct metrics to help decide whether a model’s performance has degraded

#### 2. Monitoring predictions

- most common artifact to monitor
- easy to visualize
- monitor predictions for distribution shifts

#### 3. Monitoring features

- feature validation
    - ensuring that features follow an expected schema

#### 4. Monitoring raw inputs

### 2. Monitoring Toolbox

1. logs
2. dashboards
3. alerts

### 3. Observability

- better visibility into understanding the complex behavior of software using [outputs] collected from the system at run time
- telemetry
    - system’s outputs collected at runtime
    - remote measures
        - logs and metrics collected from remote component such as
            - cloud services
            - applications on customer device
