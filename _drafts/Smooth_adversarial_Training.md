---
layout: posts
title: Smooth Adversarial Training
categories: 
  - paper
tags:
  - adversarial
toc: true
toc_sticky: true
comment: true
---

## Abstract

 Adversarial training 에서는 robustness 와 accuracy는 trade-off 관계에 있다고 널리 알려져 있다. 쉽게 생각하면 robustness를 높이면 accuracy가 감소하게 된다. 선행된 여러 연구들로 인해 이러한 trade-off 관계는 정설로 믿어지는데, 저자들은 이 현상이 ReLU 활성화 함수 때문이라고 주장한다. ReLU 함수의 non-smooth 때문에 gradient 의 품질인 안 좋아지게 되고 이는 adversarial attack에 취약해지게 된다. 만약 smooth 한 활성화 함수를 사용할 경우 robustness를 "free" 하게 높일 수 있고 말한다.

## 2. Related Works

### Adversarial Training

CNN 계열은 adversarial attack에 취약하다고 알려져 있다. 그렇다면 adversarial attack 이란 무엇일까 ?

![img](/assets/imgs/sat/adversarial_img_1.png)

 흔히 알려진 예시는 위 사진처럼 판다의 사진에 약간의 노이즈를 더하면, 사람의 눈에는 여전히 판다이지만 네트워크는 더 이상 이사진을 판다라고 판단하지 않게 된다. 이렇게 네트워크의 분류 결과를 방해하는 작업들을 adversarial attack이라고 한다. 

### Gradient Masking

Attacker들이 입력 이미지들에 노이즈를 입히는 방법은 여러가지가 있다. 그중 가장 대표적인 방법이 gradient를 이용하는 방법이다. 어떤 모델이 (99.9% airplane, 0.1% cat) 의 결과를 주었다. 그렇다면 이 결과를 이용해 gradient를 역 추적해 노이즈로 주게 되면 (cat) 으로 분류하도록 바꿀 수 있다.

![img](/assets/imgs/sat/adversarial_img_2.png)
Gradient Masking이란 이를 방어하기 위해서 모델이 확률이 아닌 하나의 결과만 출력하게 수정하였다. 이제 모델은 (airplane) 이라는 결과만 출력하게 되고 attacker는 더 이상 gradient를 얻을 수 없기 때문에 gradient masking이라고 말한다. 하지만 이 또한 완벽한 방법은 아닌데, Teacher-Student 학습 방법처럼 입력 이미지를 모델에 넣어서 나온 결과를 이용해 다른 smooth model에 학습을 시킨 후 smooth model에서 나온 gradient 를 이용한다면 원래 모델의 결과를 오분류시킬 수 있다.

## 3. ReLU Weakens Adversarial Training

### 3.1 Adversarial Training

- Objective Function

$$arg\min_{\theta}\mathbb{E}_{(x,y)\sim\mathbb{D}}[\max_{\epsilon\in\mathbb{S}}L(\theta,x+\epsilon,y)]$$

- Experiment Setting
  - ResNet-50
  - $single-step$ PGD attacker

### 3.2 How Gradient Quality Affects Adversarial Training?

figure 1

~~

table 1

## 4. Smooth Adversarial Training

### 4.1 Adversarial Training with Smooth Activation Functions

figure2

### 4.2 Ruling Out the Effect From $x < 0$

Figure 3

### 4.3 Case Study: Stabilizing Adversarial Training with ELU using CELU

table2

## 5. Exploring the Limits of Smooth Adversarial Training

확장, efficient net



## 6. Conclusion




## Refrence

- https://arxiv.org/pdf/2006.14536.pdf
- https://openai.com/blog/adversarial-example-research/