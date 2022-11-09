---
title: Chapter 7. Model Deployment and Prediction Service
categories: [mlops]
tags: ["ml-system"]
toc: true
date: 2022-11-09
author: Jongseob Jeon
---

Summary of [Designing Machine Learning Systems](https://learning.oreilly.com/library/view/designing-machine-learning/9781098107956/) written by Chip Huyen.

---
# Chapter 7. Model Deployment and Prediction Service

- ML App Logic
    - data engineering → feature engineering → model → metrics
- Deploy & Inference
    - Deploy: a loose term that generally means making your model running and accessible
    - Inference: the process of generating predictions
- To be deployed:
    - model will have to leave the development environment
    - model can be deployed to
        - a staging environment for testing
        - a production environment to be used by end users

## 1. Machine Learning Deployment Myths

### 1. Myth1: You Only Deploy One or Two ML Models at a Time

### 2. Myth2: If we Don’t Do Anything, Model Performance Remains the same.

- “software rot” or “bit rot”
    - software program degrades over time even if nothing has been changed.
- ML Systems suffer from data distribution shifts

### 3. Myth3: You Won’t Need to Update Your Model as Much

- “How often *SHOULD* I update my models?” → “How often *CAN* I update my models?”
- Model’s performance decays over time → want to update model as fast as possible

### 4. Myth4: Most ML Engineers Don’t Need to Worry About Scale

- scale
    - eg) a system that serves hundreds of queries per second of millions of users a month

## 2. Batch Prediction versus Online Prediction

- types of predictions
    1. Batch prediction, which use only batch features
    2. Online prediction that uses only batch features (eg. precomputed embeddings)
    3. Online prediction(Streaming prediction) that use both batch features and streaming features

### 1. Online Prediction

- when predictions are generated and returned as soon as requests for these predictions arrive
- on-demand prediction, synchronous prediction

### 2. Batch Prediction

- when predictions are generated periodically or whenever triggered.
- predictions are store somewhere like in-memory or SQL Tables → retrieved as needed
- asynchronous prediction

## 3. From Batch Prediction to Online Prediction

### 1. Online Prediction

- easy to start
- problem with online prediction:
    - *model might take too long to generate predictions*
    - to solve…
        - compute predictions in advance → store in database → fetch then when request arrive
        - → called batch prediction

### 2. Batch Prediction

- predictions are precomputed → trick to reduce the inference latency
- good to generate  a lot of predictions and don’t need the results immediately
- problem of batch prediction:
    1. Less responsive to users’ change preferences
    2. Need to know what requests to generate predictions in advance

### 3. Online prediction becomes default

- As hardware becomes more powerful → Online prediction becomes default
- To overcome the latency challenge of online prediction:
    1. A (near) real-time pipeline that can work with incoming data:
        - extract streaming features → input them into a model → return prediction in a near real time
    2. A model that can generate predictions at a speed acceptable to its end users

## 4. Unifying Batch Pipeline and Streaming Pipeline

- using sliding features
    - In training this feature is computed in batch
    - Whereas during inference this feature is computed in a streaming pipeline
        - Apache Flink

## 5. Model Compression

- Deployed model takes too long to generate predictions:
    1. make it do inference faster 
        - → inference optimization
    2. make the model smaller
        - → model compression
            - originally, to make model fit on edge device
    3. make the hardware it’s deployed on run faster
- model compression
    1. low-rank optimization
    2. knowledge distillation
    3. pruning
    4. quantization

### 1. Low-Rank Factorization

- key-idea
    - replace high-dimensional tensors with low-dimensional tensors
- compact convolutional filters
    - replace over-parameterized (having too many parameters) convolutional filters to compact convolutional filters
    - compact blocks to both reduce the number of parameters and increase speed
        - eg) 3x3 conv → 1x1 conv

### 2. Knowledge Distillation

- smaller model (student) is train to mimic a larger model or ensemble model (teacher)
- can work regardless of the architectural differences between teacher and student
- disadvantages
    - highly dependent on the availability of a teacher network

### 3. Pruning

- in neural network, it means
    1. remove entire nodes of a neural network
        - changing its architecture and reducing its number of parameters
    2. find parameters least useful to predictions and set them to zero(0).
        - do not change architecture, only the number of nonzero parameters
        - sparse architecture
            - make a neural network more sparse
            - require less storage than dense structure

### 4. Quantization

- most general and commonly used model compression method
- reduce model size by using fewer bits to represent its parameters
- advantage
    - reduce memory size
    - improves the computational speed
        1. allows to increase batch size
        2. less precision speeds up computation
- disadvantage
    - rounding numbers → rounding errors
    - small rounding errors → large performance change
- lower-precision training increasingly popular
- Fixed-point inference for edge device

## ML on the Cloud and on the Edge

- where your model’s computation will happen?
- ⇒ due to cost of cloud, trend are moving to edge