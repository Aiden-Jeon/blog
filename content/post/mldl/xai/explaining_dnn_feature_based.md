---
title: Feature-Based Explanations
comment: true
categories: [xai]
tags: ["xai"]
toc: true
date: 2021-07-18
author: Jongseob Jeon
---


# Feature-Based Explanations
- $m$: model
- $x=(x_1, x_2, ..., x_n)$: an instance with variable $n$
    - eg) $x_i$ 는 문장 $x$의 $i$번 째 토큰

**Importance weights explanations**는 각 feature $x_i$별로 중요도를 숫자로 표현  
**Subsets explanations**는 $x$가 예측에 영향을 미친 subset $x$를 제공

## 1. Importance Weights
### 1.1) Feature-Additive Weights

*Feature-additivity*란 모든 feature의 중요도의 합이 모델의 예측에서 모델의 편향을 뺀 값과 동일해야 하는 속성을 말합니다.

분류 문제에서는 모델의 예측은 각 클래스별 확률이고, 보통 이 중 가장 높은 확률값으로 반환합니다. 모델의 편향이란 reference, baseline 입력과 같이 아무런 정보(no information)도 없는 예측입니다. 예를 들어서 이미지 처리에서는 검은색 이미지, 자연어 처리에서는 zero-vector embedding을 baseline으로 사용합니다.

더할 수 있는 성질과 유사하게 예측에 대한 설명에 사용된 모든 피쳐들은 occlusion, omission 방법이 가능합니다. Occlusion이란 특정 feature를 baseline feature로 대체하는 것을, Omission은 특정 feature를 완전히 제거하는 것을 의미합니다. 예를 들어 자연어 처리에서 Omission이란 문장의 길이를 변수로 사용하는 방법 입니다. Feature에 변형을 가하는 occlusion과 omission은 out-of-distribution 입력으로 신뢰할 수 없는 중요치를 제공하게 됩니다.

Feature-additivitiy 제약 조건 아래,  feature ${\\{ x_{i} \\}}_{i}$ 는 $\\{w_{i}(m,x)\\}_i$ 의 가중치를 갖게 됩니다.

$$\sum_{i=1}^{\left| \mathbf{x} \right|}{w_{i}(m,\mathbf{x})=m(\mathbf{x})-m(\mathbf{b})}\tag{1}$$

수식에서 $m$은 모델, $\mathbf{x}$는 instance, $\mathbf{b}$는 베이스라인 input을 의미합니다.
회귀 모델에서 $m(\mathbf{x})$는 모델이 예측한 실제 값이며, 분류 모델에서는 클래로 예측한 확률입니다.

#### Shapley values
[Lundberg and Lee (2017)](https://arxiv.org/abs/1705.07874) 연구는 feature-additive 설명 방법을 하나로 통합하기 위해 진행된 연구입니다. 연구에서는 게임이론의 Shapley values만이 연구에서 정의한 세 가지의 특성을 만족하는 유일한 방법임을 증명했습니다.
1. Local Accuracy (completeness)  
   이 특성을 만족하기 위해서는 위에서 제시한 (1)식을 만족해야 합니다. 모든 feature-additive 방법들이 제공하는 중요도가 이 식을 만족하지는 않습니다. 예를 들으서 LIME은 $\mathbf{x}$의 주변에서 선형 회귀 모델을 학습합기 때문에 같은 값을 재현할 수 가 없습니다.
2. Missingness  
   이 특성을 만족하기 위해서는 $\mathbf{x}$에 존재하지 않는 feature는 중요도가 0이어야 합니다. 예를 들어서 모델 $m$과 문장 $\mathbf{x}$이 있을 때, $\mathbf{x}$에 존재하는 토큰들만 0 또는 다른 값의 중요도를 가져야합니다. 문장 $\mathbf{x}$에 존재하지 않는 다른 단어들은 0의 중요도여야 합니다.
3. Consistency  
   이 특성을 만족하기 위해서는 두 개의 모델 $m$, $m'$ 있을 때 어떤 feature $x_i$가 모델 $m'$ 보다 $m$ 에서 더 큰 기여(marginal contribution)를 했다면 모델 $m$에서의 feature $x_i$의 기여도가 $m'$의 $x_i$의 기여도 보다 더 크게 나와야 합니다.

### 1.2) Non-Feature-Additive Weights
대부분의 importance weights explainer들은 feature-additivity 특성을 기본으로 합니다. 다만 일부 연구에서는 이러한 특성을 신경쓰지 않습니다. 예를 들어 LS-Tree는 구분 분석 트리(parse tree)를 언어 데이터에 사용해 문장 내 토큰 간의 상호작용을 감지하고 이를 정량화하는데 가중치를 사용 할 수 있도록 합니다.


## 2. Minimal Sufficient Subsets
다른 유명한 예측을 설명하는 방법으로는 minimal sufficient subset(mss) 가 있습니다. Minimal sufficient subset란 입력으로 받은 feature중 일부분만 사용했을 때 전체 feature를 사용한 값과 같은 예측을 할 수 있는 feature subset을 말합니다.

Minimal sufficient subset를 구하기 위해서는 우선 feature의 부분 집합 $\mathbf{x}_\mathbf{s}$ 를 모델 $m$으로 예측해야 합니다.

이 때 위에서 설명한 omission과 deletion을 이용해 $\mathbf{x}$를 $\mathbf{x}_\mathbf{s}$를 만들 수 있습니다.

그런데 만약 모델이 feature의 부분 집합으로는 예측할 수 없고 feature 전체를 필요로 하는 경우에는 이 방법을 적용할 수 없습니다. 전체 feature를 필요하는 경우 이 방법으로 이용한 설명에는 아무런 정보가 없습니다(uninformative). 
그럼에도 일부 feature만 사용하는 모델에 대해서는 좋은 설명 방법이 됩니다. 예를 들어서 computer vision에서는 모든 픽셀이 문제를 푸는데 필요하지 않습니다. 비슷하게 감정분석에서는 일부 구문만 있어도 충분히 감정을 분류할 수 있습니다.

또한 여러 개의 minimal sufficient subset이 존재할 수 있는데, 이 때 각 minimal sufficient subset들은 인스턴스의 예측을 설명하는 하나의 독립적인 잠재적 설명(potential explanation)이 됩니다. 이런 경우에는 모델의 전체적인 면(complete view) 를 보여주기 위해서는 가능한 모든 minimal sufficient subset를 보여주어야 합니다.

Minimal Sufficient Subsets를 이용하는 설명 방법으로는 L2X, SIS, Anchors, INVASE 등이 있습니다.
L2X는 전체 feature를 이용한 예측과 subset feature를 이용한 예측의 상호 정보(mutual information)을 극대화 시키는 subset를 학습합니다.
다만, L2X를 계산하기 위해서는 최소의 원소 개수(cardinality of a minimal sufficient subset)를 모수로서 정해야 합니다. 
이러한 요소는 L2X를 실제 문제에 적용을 어렵게 합니다.
이를 극복한 알고리즘이 INVASE인데, 이 알고리즘은 각 인스턴스 별로 최소의 원소 개수를 다르게 지정할 수 있습니다.
L2X와 INVASE 알고리즘은 한 개의 subset만을 제공합니다. 반대로 SIS는 서로 겹치지(overlap) 않는 minimal sufficient subset를 제공합니다.
