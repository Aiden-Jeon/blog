---
title: Pandas를 이용한 tf-idf 구하기
comment:   
    enable: true
categories: [nlp]
tags: ["nlp"]
toc: true
date: 2021-05-05
author: Jongseob Jeon
---

이번 포스트는 Pandas를 이용해 tf-idf를 계산하는 방법에 대해서 설명합니다.

---
### TF-IDF
- **TF(Term Frequency)**: 한 문장에서 나타난 각 단어들의 빈도
- **IDF(Inverse Document Frequency)**: 문서들에 나타난 단어들의 빈도의 역수
- **TF-IDF**: TF*IDF


### Data
데이터는 이미 빈도수가 계산되어 있다고 가정합니다.
```python
data = [
    [0, 0, 1, 0, 1, 0, 0, 0, 0, 0],
    [0, 0, 0, 0, 2, 1, 0, 0, 0, 0],
    [0, 0, 0, 0, 0, 0, 0, 0, 0, 2],
    [0, 0, 0, 0, 0, 0, 0, 3, 0, 0],
    [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
]
columns = [
    f"feature_{0}" for i in range(10)
]
sample_data = pd.DataFrame(data, columns=columns)
```

데이터는 다음과 같습니다.

|    |   feature_0 |   feature_1 |   feature_2 |   feature_3 |   feature_4 |   feature_5 |   feature_6 |   feature_7 |   feature_8 |   feature_9 |
|---:|------------:|------------:|------------:|------------:|------------:|------------:|------------:|------------:|------------:|------------:|
|  0 |           0 |           0 |           1 |           0 |           1 |           0 |           0 |           0 |           0 |           0 |
|  1 |           0 |           0 |           0 |           0 |           2 |           1 |           0 |           0 |           0 |           0 |
|  2 |           0 |           0 |           0 |           0 |           0 |           0 |           0 |           0 |           0 |           2 |
|  3 |           0 |           0 |           0 |           0 |           0 |           0 |           0 |           3 |           0 |           0 |
|  4 |           0 |           0 |           0 |           0 |           0 |           0 |           0 |           0 |           0 |           0 |

### TF(Term Frequency)
각 문장은 하나의 row입니다.
우선, 각 문장에서 나타난 전체 단어의 개수를 구합니다.
```python
sample_df.sum(axis=1)
```
계산하면 다음과 같습니다.

|    |   0 |
|---:|----:|
|  0 |   2 |
|  1 |   3 |
|  2 |   2 |
|  3 |   3 |
|  4 |   0 |

이 값을 각 row에 나눠줍니다.
```python
tf = sample_df.div(sample_df.sum(axis=1), axis=0)
```

계산하면 다음과 같습니다.

|    |   feature_0 |   feature_1 |   feature_2 |   feature_3 |   feature_4 |   feature_5 |   feature_6 |   feature_7 |   feature_8 |   feature_9 |
|---:|------------:|------------:|------------:|------------:|------------:|------------:|------------:|------------:|------------:|------------:|
|  0 |           0 |           0 |         0.5 |           0 |    0.5      |    0        |           0 |           0 |           0 |           0 |
|  1 |           0 |           0 |         0   |           0 |    0.666667 |    0.333333 |           0 |           0 |           0 |           0 |
|  2 |           0 |           0 |         0   |           0 |    0        |    0        |           0 |           0 |           0 |           1 |
|  3 |           0 |           0 |         0   |           0 |    0        |    0        |           0 |           1 |           0 |           0 |
|  4 |         nan |         nan |       nan   |         nan |  nan        |  nan        |         nan |         nan |         nan |         nan |

5번째 문장의 경우 단어의 합이 0 이기 때문에 nan값이 생겼습니다.
이를 처리하기 위해 na 값에 0을 채워줍니다.

```python
tf = tf.fillna(0)
```

전체 단어의 빈도수로 나누었기 때문에 tf의 각 row의 합은 1과 같아야 합니다.
```python
tf.sum(axis=1)
```

|    |   0 |
|---:|----:|
|  0 |   1 |
|  1 |   1 |
|  2 |   1 |
|  3 |   1 |
|  4 |   0 |

### IDF(Inverse Document Frequency)
각 단어는 하나의 column입니다.
다음으로 각 단어들이 문장들에서 나타난 개수를 구합니다.
```python
(sample_df != 0).sum(axis=0)
```

|           |   0 |
|:----------|----:|
| feature_0 |   0 |
| feature_1 |   0 |
| feature_2 |   1 |
| feature_3 |   0 |
| feature_4 |   2 |
| feature_5 |   1 |
| feature_6 |   0 |
| feature_7 |   1 |
| feature_8 |   0 |
| feature_9 |   1 |

이제 이 값을 각 column에 나눠준 후 log를 취합니다.
```python
idf = np.log(len(sample_df) / (sample_df != 0).sum(axis=0))
```

idf값은 다음과 같습니다.
|           |          0 |
|:----------|-----------:|
| feature_0 | inf        |
| feature_1 | inf        |
| feature_2 |   1.60944  |
| feature_3 | inf        |
| feature_4 |   0.916291 |
| feature_5 |   1.60944  |
| feature_6 | inf        |
| feature_7 |   1.60944  |
| feature_8 | inf        |
| feature_9 |   1.60944  |

0으로 나눠서 inf값이 생긴 곳에 0을 채워줍니다.
```python
idf = idf.replace(np.inf, 0)
```

|           |        0 |
|:----------|---------:|
| feature_0 | 0        |
| feature_1 | 0        |
| feature_2 | 1.60944  |
| feature_3 | 0        |
| feature_4 | 0.916291 |
| feature_5 | 1.60944  |
| feature_6 | 0        |
| feature_7 | 1.60944  |
| feature_8 | 0        |
| feature_9 | 1.60944  |

### TF-IDF
이제 위에서 계산한 TF와 IDF를 곱해줍니다.
```python
tf_idf = tf * idf
```

다음과 같이 tf-idf를 구할 수 있습니다.
|    |   feature_0 |   feature_1 |   feature_2 |   feature_3 |   feature_4 |   feature_5 |   feature_6 |   feature_7 |   feature_8 |   feature_9 |
|---:|------------:|------------:|------------:|------------:|------------:|------------:|------------:|------------:|------------:|------------:|
|  0 |           0 |           0 |    0.804719 |           0 |    0.458145 |    0        |           0 |     0       |           0 |     0       |
|  1 |           0 |           0 |    0        |           0 |    0.61086  |    0.536479 |           0 |     0       |           0 |     0       |
|  2 |           0 |           0 |    0        |           0 |    0        |    0        |           0 |     0       |           0 |     1.60944 |
|  3 |           0 |           0 |    0        |           0 |    0        |    0        |           0 |     1.60944 |           0 |     0       |
|  4 |           0 |           0 |    0        |           0 |    0        |    0        |           0 |     0       |           0 |     0       |



### Reference
- https://ko.wikipedia.org/wiki/Tf-idf