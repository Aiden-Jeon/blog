---
title: Types of Explanatory Methods
comment: true
categories: [xai]
tags: ["xai"]
toc: true
date: 2021-07-14
author: Jongseob Jeon
---

**들어가기 앞서..**  
본 포스트는 [Explaining Deep Neural Networks
](https://arxiv.org/abs/2010.01496?fbclid=IwAR3Y5gfxtckZR4lHpFpQgo6ba-v_O0Fj-93G0sRMtXKYTBdESeH29uN4mg8) 논문을 읽고 요약한 내용입니다.
더 자세한 내용은 해당 논문을 참고해주세요.

# Types of Explanatory Methods

## 1. Post-Hoc versus Self-Explanatory
**Post-hoc explanatory methods**  
사후 설명 방법은 이미 학습이 끝나서 고정된 모델을 대상으로 합니다.  
이러한 방법의 대표적인 예시로는 LIME이 있습니다. LIME은 설명하고자 하는 모델의 예측을 그 주변의 값들을 이용해 선형 회귀와 같은 해석 가능한 모델을 학습시켜서 설명합니다.  
    - 장점) 고정된 모델을 대상으로 하기 때문에 모델에 영향을 미치지 않음  
    - 단점) 모델 학습 외에 추가적인 학습이 필요하며 이를 위한 설명이 있는 데이터셋이 필요

**Self-explanatory models**  
자기 설명(Self-explanatory)이 가능한 모델들은 아키텍쳐 속에 모델의 예측의 해석을 제공할 수 있는 방법을 내장하고 있습니다. 큰 그림으로 보자면 self-explanatory model들은 두 개의 모듈,  predictor 와 explanation generator를 갖고 있습니다.  
    - 장점) 모델의 학습과 explanation generator를 동시에 함으로 추가적인 데이터가 필요 없음  
    - 단점) predictor와 explanation generator가 묶여서(jointly) 학습되는데 이 때 explanation generator 가 predictor의 학습에 영향을 미치고 이는 모델의 성능의 저하로 이어짐  

## 2. Black-Box versus White-Box
모델을 설명하기 위해서 다른 분류 방법으로는 모델의 구조의 이해가 필요한지에 따라 나눌 수 있습니다. 이 분류 방법은 Post-hoc explanatory methods의 세분화한 것입니다. 

**Black-box/model-agnostic explainers**  
블랙 박스 explainer들은 대상 모델에 input에 대한 결과만을 요청할 수 있습니다. 대표적으로 LIME, KernalSHAP, L2X 그리고 LS-Tree 와 같은 방법들이 있습니다.  
    - 장점) 모델에 대한 이해가 필요 없기 때문에 어떤 형태의 모델에도 빠르게 적용할 수 있음  
    - 단점) 모델에 대한 이해가 없기 때문에 input과 prediction 사이의 correlation에 영향을 받아모델의 설명력이 부정확해질 수 있음  

**White-box/model-dependant explainers**  
화이트 박스 explainer들은 대상 모델의 구조(Architecture)에 접근이 가능해야 합니다. 대표적으로 LRP, DeepLIFT, saliency maps, integrated gradients, Grad-CAM, DeepSHAP 그리고 MaxSHAP와 같은 방법이 있습니다.

## 3. Instance-Wise versus Global

모델을 설명하는 또 다른 분류 방법은 설명의 범위에 따라 나눌 수 있습니다. 이 분류 방법 또한 주로 Post-hoc explanatory methods을 세분화한 것 입니다. Self-explanatory model들은 기본적으로 Instance-wise explainers 입니다.

**Instance-wise explainers**  
모델의 단일 예측(individual instance) 결과에 대해서 설명을 제공합니다. 예를 들어 LIME은 instance 별로 linear regression을 학습해 설명합니다.  
    - 장점) End user가 요구하는 모델의 결정에 대한 설명을 제공할 수 있음

**Global explainers**  
조금 더 high-level에서 모델의 내부 동작 방식에 대해서 설명합니다. 예를 들어서 뉴럴 네트워크를 soft decision tree로 distilling 하여서 전체 설명력을 제공합니다.  
    - 장점) 모델에 발생할 수 있는 biases를 빠르게 진단하거나 지식의 발견에 유용함

Global explainers은 주로 모델을 해석할 수 있는 모델로 distiling 하기 때문에 instance-wise에 대한 설명도 제공 할 수 있습니다. 하지만 Instance-wise explainer로 모델의 전체 동작 방식을 설명하기에는 어려움이 있습니다. 비록 Instance-wise explainer 방법이 모델의 전체 동작 방식을 설명하기에는 어렵지만, 전체 모델의 설명력을 얻기 위한 시작점으로 사용할 수 있습니다.

## 4. Forms of Explanations
모델을 설명하기 위한 방법에 따라서 나눌 수 도 있습니다. 각 방법에 따른 방법론들은 논문을 참고해주세요. 이 챕터에서는 간단하게 개념만 설명하고 넘어갑니다.

### Feature-based explanations
Feature에 기반한 설명은 현재 가장 많이 사용되는 방법입니다. 이 방법은 모델의 개별 예측을 중요도(importance)또는 기여도(contribution)로 평가하는 방식입니다. 텍스트의 토큰과 이미지의 super-pixel 도 feature에 포함합니다.

**importance weights**  
Importance weights explanation은 입력 feature의 모델이 수행한 예측에 대한 기여 정도를 숫자로 나타냅니다. 

**subsets of features**  
Subset explanation은 각 instance 별로 예측에 중요한 feature의 subset을 제공합니다.

예를 들어서문장 감정 분류 모델이 `“The movie was very good."`를, `"4/5점, 긍정"`이라고 분류했습니다. 이 때 subset explanation은 `{“very”, “good”}`을 중요한 feature로 제공합니다. 반면, importance weights explanation는 `{"good": 3, "very":1}`(이 때 두 점수의 합은 분류 모델의 예측인 4점 입니다)을 제공합니다. 

### Natural language explanations
자연어에 대한 설명은 예측에 대한 결과를 사람과 같은 방식으로 전달합니다. 예를 들어서 “A woman is walking her dog in the park.”라는 문장에 “A person is in the park.”를 포함시켜서 설명한다면, “A woman is a person, and walking in the park implies being in the park.”과 같이 설명할 수 있습니다.

### Concept-based explanations
Concept-based explainers는 사용자가 정의한 high-level의 컨셉의 중요도를 정령화 하는 것을 목표로 합니다.

### Example-based explanations
Example-based는 모델이 예측한 결과에 영향을 미친 training set의 instance를 제시합니다.

### Surrogate explanations
Surrogate explainers는 설명이 가능한 대체 모델을 제공하고자 합니다.

### Combinations of forms of explanations
단일 explainer는 여러 개의 설명을 제공할 수 있습니다.
