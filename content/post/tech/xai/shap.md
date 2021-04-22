---
title: SHAP
comment: true
categories: [XAI]
toc: true
date: 2021-03-31
author: Jongseob Jeon
draft: true
---


# A Unified Approach to Interpreting Model Predictions

SHAP음 XAI에서 가장 주목 받고 있는 모델 설명 방법입니다. SHAP은 SHapley Additive exPlanations의 약자로 Shapley Value를 이용합니다. SHAP의 가장 큰 특징은 두 가지입니다.

1. 앞서 나온 다른 기법들을 모두 아우를 수 있음.
2. 게임 이론에 근거한 Additive Feature Attribution Methods 중 유일한 solution.

이제부터 각 특징들을 설명하겠습니다.

## Additive Feature Attribution Methods

'Additive'는 사전으로 '덧셈의' 입니다. 그리고 좀 더 풀면 더할 수 있는 성질입니다. 왜 SHAP에서는 Additive라는 단어를 사용했을까요? 예를 들어, Random Forest에서 Feature Importance를 계산하면 다음과 같습니다.

| Feature   | Feature Importance |
| --------- | ------------------ |
| Feature A | 0.31               |
| Feature B | 0.17               |
| Feature C | 0.15               |

이 때 Feature B와 Featuer C를 더하면 0.32이므로, Feature A보다 중요하다고 말할 수 있을까요? Random Forest에서는 Gini 계수를 사용하는데 이 값은 더할 수 있는 값이 아닙니다. 따라서 위와 같은 해석을 할 수 없습니다. 이번에는 Linear Regression을 생각해 봅시다.

$$y=0.3+1.2x_1-1.3x_2+0.2x_3$$

이 때 우리는 x2는 결과 값을 음수로 보내고, x1이 x3보다 더 큰 영향을 미친다고 해석할 수 있습니다. 이처럼 '더할수 있는' 성질은 사람들이 더 직관적으로 이해할 수 있습니다. 그래서 SHAP은 다음과 같이 정의됩니다. 

$$g(z')=\phi_{0}+\sum_{i=1}^{M}\phi_{i}z'_{i} \\ \text{where }z'\in\{0,1\}^{M},M\text{ is the number of simplified input features, and }\phi_{i}\in\R$$

이러한 정의를 갖는 XAI 기법들은 다음과 같습니다.

1. LIME
2. DeepLIFT
3. Layer-Wise Relevance Propagation
4. Classic Shapley Value Estimation

## Simple Properties Uniquely Determine Additive Feature Attributions

Additive feature attribution methods에는 게임 이론에 근거한 unique solution이 존재합니다. 그런데 왜 게임 이론을 근거로 할까요? 게임 이론에서는 "공평함(Fairness)"를 중요하게 여깁니다. 그리고 이런 공평함은 다른 사람을 설득할 때 큰 힘이 됩니다. 그래서 SHAP에서는 게임 이론을 모델의 설명과 연관시켰습니다. 게임 이론에서는 공평함을 정의를 하였습니다. 

1. People are willing to sacrifice their own material well-being to help those who are being kind
2. People are willing to sacrifice their own material well-being to punish those who are being unkind
3. Both of these motivations have a greater effect on behavior as the material cost of sacrificing becomes smaller

이를 머신러닝에 맞추어서 해석하면 아래와 같습니다.

1. Local accuracy
2. Missingness
3. Consistency

그런데 앞서 나왔던 XAI 방법들은 이 3가지 성질 중 한 가지씩 만족시키지 못하기 때문에 unique solution이 될 수 없습니다.

## SHAP (SHapley Additive exPlanation) Values

### Kernel SHAP (Linear LIME + Shapley values)

### Linear SHAP

### Low-Order SHAP

### Max SHAP

### Deep SHAP (DeepLIFT + Shapley values)