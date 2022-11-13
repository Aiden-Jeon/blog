---
title: Chapter 9. Continual Learning and Test in Production
categories: [mlops]
tags: ["ml-system"]
toc: true
date: 2022-11-13
author: Jongseob Jeon
---

Summary of [Designing Machine Learning Systems](https://learning.oreilly.com/library/view/designing-machine-learning/9781098107956/) written by Chip Huyen.

---
# Chapter 9. Continual Learning and Test in Production
- continual learning
    - to adapt our models to data distribution shifts
    - infrastructural problem
- test in production
    - model is retrained to adapt to the changing environment
        - evaluating it on a stationary set
        - also test in production
    - monitoring & test in production
        - monitoring: passively keeping track of the outputs
        - test in production: proactively choosing which model to produce outputs
        - Goal: to understand a model’s performance and figure out when to update it
- Goal of continual learning
    - to safely and efficiently automate the update

## Continual Learning

- Misunderstand in term of “Continual learning”
    - A models updates itself with every incoming sample in production
        - problems
            1. catastrophic forgetting
                - tendency of a neural network to completely and abruptly forget previous learned information upon learning new information
            2. make training more expensive
    - Update their model in micro-batch (512, 1024)
- updated model shouldn’t be deployed until it’s been evaluated
    - existing model → champion model
    - replica model → challenger model
- Don’t need to update model frequently
    1. don’t have enough traffic
    2. model don’t decay that fast

### 1. Stateless Retraining Versus Stateful Training

- continual learning isn’t about retraining frequency → manner in which model is retrained

#### 1. Stateless retraining

- the model is trained from scratch
- for example
    
    ```
    <--> (model v1)
      <--> (model v2)
        <--> (model v3)
    ```
    
- require a lot more data

#### 2. Stateful training

- the model continues training on new data
- fine-tuning or incremental learning
- for example
    
    ```
    <--> (model v1)
        <-> (model v1.1)
           <-> (model v1.2)
    ```
    
- allows to update model with less data

#### 3. Types of model updates

1. Model iteration
    - A new feature is added to an existing model architecture
    - model architecture is changed
    - to be stateful training
        - knowledge transfer
        - model surgery
2. Data iteration
    - model architecture and features remain the same
    - but refresh this model with new data
    - means stateful training

### 2. Why Continual Learning

- Continual Learning
    - setting up infrastructure so that you can update your model and deploy these changes as fast as you want
- Use case
    1. to combat data distribution shifts, especially when the shifts happens suddenly
    2. to adapt to rare events
- Continuous cold start problem
    - arises when your model has to make predictions for a new user without any historical data

### 3. Continual Learning Challenges

#### 1. Fresh data access challenge

#### 2. Evaluation challenged

- The biggest challenge of continual learning
    - making sure that this update is good enough to be deployed
- The risk of catastrophic failures amplify with continual learning
    1. The more frequently you update your models → the more opportunities there are for updates to fail
    2. make your models more susceptible to coordinated manipulation and adversarial attack
- evaluation pipeline
    - evaluation takes time → can be another bottleneck for model update frequency

#### 3. Algorithm challenged

### 4. Four Stages of continual learning

1. Stage 1: Manual, stateless retraining
2. Stage 2: Automated retraining
3. Stage 3: Automated, stateful retraining
4. Stage 4: Continual learning

### 5. How Often to Update Your Model

#### 1. Value of data freshness

- Q) How often to update a model? → How much the model performance will improve with updating?
- To figure out the gain
    - training your model on the data from different time windows in the past and see how the performance changes
    - for example
        
        ```
        <--> model A
          <--> model B
            <--> model C
                 Test data
        ```
        

#### 2. Model iteration versus data iteration

- model iteration
    - data iterating doesn’t give you much performance gain
    - → spend resources on finding a better model
- data iteration
    - finding a better model architecture requires 100X compute for training and gives 1% performance gain
    - whereas data iteration requires 1X compute and also gives 1% performance gain

## Test in Production

- To sufficiently evaluate models
    - use mixture of offline evaluation and online evaluation
- Offline evaluation
    - Good old test split to evaluate models
    - → not sufficient to evaluate new model
    - →backtest
- backtest
    - method of testing a predictive model on data from a specific period of time in the past
        - not quite sufficient

### 1. Shadow Deployment

- the safest way to deploy model
- steps
    1. Deploy the candidate model in parallel with the existing model
    2. For each incoming request, route it both models to make predictions, but only serve the existing model’s prediction to user
    3. Log the predictions from the new model for analysis purpose
- Replace model when new model’s predictions are satisfactory
- But expensive: doubling cost

### 2. A/B Testing

- a way to compare two variants of an object
- testing responses to these two variants → determining which of two variants is more effective
- steps
    1. Deploy the candidate model alongside the existing model
    2. A percentage of traffic is routed to the new model, the rest is routed to the existing model
    3. Monitor and analyze the predictions and user feedback, if any, from both models to determine whether the difference in the two models’ performance is statistically significant
- A/B testing requires
    1. A/B testing consists of a randomized experiment
    2. A/B test should be run on a sufficient number of samples to gain enough confidence about the outcome

### 3. Canary Release

- technique to reduce the risk of introducing a new software version in production
    - by slow rolling out the change to a small subset of users
    - before rolling it out to everybody
- steps
    1. Deploy the candidate model alongside the existing model.
        - candidate model is called canary
    2. A portion of the traffic is routed to the candidate model
    3. If its performance is satisfactory increase the traffic to the candidate model. If not, abort the canary and route all traffic to the existing model
    4. Stop when either the canary serves all the traffic or the canary is aborted

### 4. Interleaving Experiments

- Reliably identifies the best algorithms with considerably smaller sample size compare to traditional A/B testomg
- A/B testing**********************************:********************************** core metrics are compared
- Interleaving: compared by measuring user preferences

### 5. Bandits

- A/B testing
    - randomly route traffic
    - stateless
- Bandits
    - allow to determine how to route traffic
    - stateful
- a lot more data-efficient that A/B testing
    - require less data
    - reduce opportunity cost as they route traffic to the better model more quickly
    - A/B testing 630,000 to get a 95% confidence interval
    - 12,000 samples to determine
- a lot more difficult to implement
    - bandits requires computing and keeping track of models’ payoffs
