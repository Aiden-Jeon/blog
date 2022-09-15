---
title: Properties of Explanations
categories: [xai]
tags: ["xai"]
toc: true
date: 2021-07-19
author: Jongseob Jeon
comment: 
  enable: true
---


# Properties of Explanations
딥러닝 모델을 해석하려는 목적은 모델의 의사 결정 과정을 사람이 이해할 수 있게 하는데 있습니다. 
이 목적이 달성되었다고 보려면 다음 두 가지 특성이 만족해야 됩니다.
1. explanations should be easy for humans to interpret
2. explanations should be faithfully describing the decision-making process of the target model

## Easiness to interpret
모델 예측의 설명이 어렵다면 사람들이 사용하기 어렵습니다. 예를 들어서 뉴럴넷 모델의 해석이라고 모델의 모든 함수들의 chain 값을 준다면 이는 모델을 이해하는데 아무런 도움이 되지 않습니다. 

각 모델 설명 방법 별로 해석 방법은 다양합니다. 예를 들어서 minimal suffictient subsets는 일부의 feature만 있어도 모델 예측을 할 수 있는 모든 정보는 들어 있다고 해석할 수 있습니다. 그런데 Shapley value를 이용한 설명 방법은 사람이 이해하기 어렵습니다. Shapley value는 가능한 모든 경우의 수에서 각 marginal contribution을 계산한 값입니다. 이렇게 정량화된 값은 사람마다 다를 수 있지만 직관적이진 않습니다. 주어진 Shapley value의 값과 방향만 보고 feature별 중요도를 비교하는 등 잘못된 해석을 할 수 있습니다.

## Faithfulness
모델 설명의 충실함은 모델이 예측하는 의사 결정 과정을 얼마나 잘 맞추었는지로 평가할 수 있습니다. 
여기서 주의해야 할 것은 이 개념이 모델 설명이 얼마나 정확했는지와는 다른 개념입니다. 

모델의 설명은 사용자들에게 모델에 대한 인식과 신뢰에 영향을 줍니다.
그렇기 때문에 충실하지 못한 모델의 설명은 잘못된 모델을 믿게 만들 수 있습니다.
