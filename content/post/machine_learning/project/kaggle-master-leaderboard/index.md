---
title: "Tabular Foundation Model 은 Kaggle 에서 GBDT 를 이길까 — kaggle-master 리더보드"
slug: kaggle-master-leaderboard
categories: [machine-learning]
tags: ["tabular", "tabpfn", "kaggle", "gbdt"]
toc: false
date: 2026-10-27 13:00:00+09:00
author: Jongseob Jeon
---

TabPFN, TabICL, Kumo 같은 Tabular Foundation Model(TFM) 이 정말 LightGBM·XGBoost·CatBoost 보다 나은지 궁금해서
Kaggle 대회 13개에 같은 조건으로 돌려 보고 있습니다.
대회마다 fold 와 피처를 고정해 두고 모든 모델을 튜닝 없이 기본 설정으로 돌린 뒤, 저장한 테스트 예측을 실제로 제출해 리더보드 순위까지 모았습니다.
코드와 원본 결과는 [Aiden-Jeon/kaggle-master](https://github.com/Aiden-Jeon/kaggle-master) 에 있습니다.

지금까지 본 것은 대략 이렇습니다.

- 수천 행 이하의 작은 대회에서는 TFM 이 GBDT 와 비슷하거나 앞섭니다. 수십만 행으로 가면 컨텍스트를 1만 행으로 잘라 쓴 TFM 은 GBDT 에 밀립니다.
- 그 상한을 풀거나 20만 행으로 올리면 kumo 가 큰 대회에서도 GBDT 를 이깁니다. 다른 TFM 은 아직 상한을 건 결과라 같은 조건의 비교는 아닙니다.
- 범주 ID 만 있는 Amazon 에서는 catboost 가 크게 앞섭니다.
- 상위 모델끼리의 차이는 대부분 fold 표준편차 안이라 작은 대회일수록 CV 순위가 LB 에서 자주 뒤집힙니다.

아래 리더보드는 저장소의 `km board` 명령이 결과 CSV 에서 만든 페이지라 실험이 늘면 같이 갱신됩니다.
열 제목으로 정렬하고, 모델 이름을 누르면 그 모델만 강조되고, 히트맵 칸을 누르면 그 대회 상세로 이동합니다.
지표 정의와 해석할 때 주의할 점은 맨 위 "표 읽는 법" 에 적어 두었습니다.

{{< embed src="https://aiden-jeon.github.io/kaggle-master/leaderboard.html" title="kaggle-master 리더보드" height="2400" >}}
