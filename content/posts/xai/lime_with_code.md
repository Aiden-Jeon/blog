---
title: LIME with code
comment: true
categories: [XAI]
toc: true
date: 2021-03-31
author: Jongseob Jeon
---


## **(LIME) Local Interpretable Model-agnostic Explanations**​

### **Objective of LIME**

$$\xi(x)=\underset{g\in G}{\operatorname{argmin}} L(f,g,\pi_{x})+\Omega(g)$$

​

### **Train $f$**

---

이번 포스트에서는 예제 데이터로 fetch_20newsgroups를 사용했습니다.
fetch_20newsgroups 데이터에는 총 20개의 Class가 존재합니다. 
하지만 문제를 단순하게 하기 위해서 이번 포스트에서는 20개 Class 모두를 사용하기 보다는 "atheism", "christian" 2개의 카테고리만 이용하겠습니다.
이렇게 되면 이제 두 클래스를 분류하는 Binary Text Classification 문제가 되고 모델로는 Random Forest를 사용해 보겠습니다.

그리고 학습된 Random forest를 $f$ 라고 정의하겠습니다.

```python
import sklearn
import sklearn.ensemble
from sklearn.pipeline import make_pipeline
from sklearn.datasets import fetch_20newsgroups

categories = ["alt.atheism", "soc.religion.christian"]
newsgroups_train = fetch_20newsgroups(subset="train", categories=categories)
newsgroups_test = fetch_20newsgroups(subset="test", categories=categories)
class_names = ["atheism", "christian"]

vectorizer = sklearn.feature_extraction.text.TfidfVectorizer(lowercase=False)
train_vectors = vectorizer.fit_transform(newsgroups_train.data)
test_vectors = vectorizer.transform(newsgroups_test.data)
rf = sklearn.ensemble.RandomForestClassifier(n_estimators=500)
rf.fit(train_vectors, newsgroups_train.target)

c = make_pipeline(vectorizer, rf)
```

다음으로 test data에서 $x$를 선택합니다.
선택된 $x$에 대해서 Random Forest 모델 $f$를 이용해 예측된 $f(x)$를 해석하고자 합니다.
하지만 Random Forest $f$ 는 블랙박스 모델이기 때문에 이를 직접적으로 해석할 수는 없습니다.
그래서 $f(x)$를 바로 해석하는 대신 해석할 수 있는 모델 $g$를 이용해 $f(x)$의 결과를 해석해 보겠습니다.

newsgroups_test에서 첫 번째 데이터를 이용해서 LIME의 동작 방법에 대해서 알아보겠습니다.

```python
text_instance, instance_label = newsgroups_test.data[0], newsgroups_test.target[0]
```
데이터를 한번 확인 해보겠습니다.

```python
> text_instance
'From: crackle!dabbott@munnari.oz.au (NAME)\nSubject: "Why I am not Bertrand Russell" (2nd request)\nReply-To: dabbott@augean.eleceng.adelaide.edu.au (Derek Abbott)\nOrganization: Electrical & Electronic Eng., University of Adelaide\nLines: 4\n\nCould the guy who wrote the article "Why I am not Bertrand Russell"\nresend me a copy?\n\nSorry, I accidently deleted my copy and forgot your name.\n'
```

이 데이터의 레이블은 1, "christian" 입니다.

```python
> instance_label
1
```

위에서 우리가 학습한 모델 $f$을 이용해 예측해 보면 모델이 정답인 1로 예측합니다.

```python
> c.predict([text_instance])
array([1])
```

위에서 학습한 모델이 잘 예측을 하는 것을 확인했습니다. 이제 이 모델은 어떻게 1이라고 예측을 하게 됐을까요? 이제부터 그 과정에 대해서 알아보려고 합니다.

### **Interpretable Data Representation**

---

우선 데이터 $x$ 를 interpretable representation이 가능한 $x'$ 으로 변환합니다. Text의 경우 interpretable represenstion은 단어가 존재한다/존재하지 않는다 입니다. 

아래 코드는 단어를 숫자로 mapping 시켜주는 역할을 합니다.
```python
from lime.lime_text import TextDomainMapper, IndexedString

indexed_string = IndexedString(text_instance, bow=True, split_expression=r"\W+", mask_string=None)
domain_mapper = TextDomainMapper(indexed_string)
```

​
$x'$ 는 모든 단어가 존재하기 때문에 아래와 같이 모든 값이 존재한다를 뜻하는 1로 채워져 있습니다.
```python
[1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1.,
 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1.,
 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1.,
 1., 1., 1.]
```



### **Sampling for Local Exploration**

---

이제 위에서 변환된 $x'$ 주변에서 $z'$을 샘플링 합니다. 샘플링 방법은 random 하게 값을 고른 후 1을 0으로 바꿔주면 됩니다.

```python
import numpy as np
from sklearn.utils import check_random_state

random_state = check_random_state(2020)
num_samples = 10

doc_size = indexed_string.num_words()
sample = random_state.randint(1, doc_size + 1, num_samples - 1)

data = np.ones((num_samples, doc_size))
data[0] = np.ones(doc_size)
```

샘플링된 $z'$ 를 확인하면 다음과 같습니다. data의 첫번째는 $x'$ 이며, 나머지는 $z'$ 입니다.

```python
> data
array([[1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1.,
        1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1.,
        1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1.,
        1., 1., 1.],
       [0., 1., 0., 1., 0., 1., 0., 0., 1., 1., 1., 1., 0., 0., 0., 1.,
        0., 0., 0., 0., 1., 0., 0., 0., 1., 0., 0., 0., 1., 0., 0., 0.,
        1., 0., 0., 0., 0., 1., 1., 0., 0., 1., 0., 1., 1., 0., 0., 0.,
        1., 0., 0.],
       [1., 1., 0., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1.,
        1., 1., 1., 1., 1., 0., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1.,
        0., 1., 1., 1., 0., 0., 1., 0., 1., 1., 1., 1., 0., 0., 1., 1.,
        1., 1., 0.],
       [1., 1., 1., 0., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1.,
        1., 1., 1., 1., 1., 1., 1., 1., 0., 1., 1., 1., 1., 1., 1., 1.,
        1., 0., 1., 1., 1., 1., 1., 1., 0., 1., 1., 1., 1., 1., 1., 1.,
        1., 1., 1.],
       [1., 1., 1., 1., 1., 1., 1., 1., 1., 0., 1., 1., 1., 1., 1., 1.,
        0., 0., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 0., 1.,
        1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1.,
        1., 1., 1.],
       [1., 1., 1., 0., 0., 0., 1., 0., 0., 1., 0., 1., 0., 1., 0., 0.,
        0., 1., 1., 0., 0., 0., 1., 1., 0., 0., 0., 1., 1., 0., 1., 0.,
        0., 0., 0., 0., 1., 0., 1., 1., 0., 1., 1., 0., 1., 0., 1., 1.,
        0., 0., 1.],
       [1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1.,
        1., 1., 0., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1.,
        1., 1., 1., 1., 1., 1., 0., 0., 1., 1., 1., 1., 1., 1., 1., 1.,
        1., 0., 1.],
       [1., 1., 1., 1., 1., 1., 0., 0., 1., 1., 0., 1., 1., 1., 1., 1.,
        1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 0.,
        1., 0., 0., 0., 0., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1., 1.,
        1., 1., 1.],
       [1., 1., 0., 0., 0., 1., 0., 0., 1., 1., 0., 1., 1., 1., 1., 1.,
        0., 0., 1., 0., 0., 0., 0., 0., 1., 1., 1., 0., 1., 0., 1., 1.,
        0., 0., 0., 0., 1., 0., 1., 0., 0., 0., 0., 0., 0., 1., 0., 0.,
        0., 1., 0.],
       [0., 0., 0., 0., 0., 0., 0., 0., 0., 0., 0., 0., 0., 0., 0., 0.,
        0., 0., 0., 1., 0., 0., 0., 0., 0., 0., 0., 0., 0., 0., 0., 0.,
        0., 0., 0., 0., 0., 0., 0., 0., 1., 0., 0., 0., 0., 0., 0., 0.,
        0., 0., 0.]])
```

$z'$ 를 원래의 표현(텍스트) 로 복원시키면 이 값이 $z$ 가 됩니다.

```python
features_range = range(doc_size)
inverse_data = [indexed_string.raw_string()]
for i, size in enumerate(sample, start=1):
    inactive = random_state.choice(features_range, size, replace=False)
    data[i, inactive] = 0
    inverse_data.append(indexed_string.inverse_removing(inactive))
```

inverse_data를 확인하면 다음과 같습니다.
```python
> inverse_data
['From: crackle!dabbott@munnari.oz.au (NAME)\nSubject: "Why I am not Bertrand Russell" (2nd request)\nReply-To: dabbott@augean.eleceng.adelaide.edu.au (Derek Abbott)\nOrganization: Electrical & Electronic Eng., University of Adelaide\nLines: 4\n\nCould the guy who wrote the article "Why I am not Bertrand Russell"\nresend me a copy?\n\nSorry, I accidently deleted my copy and forgot your name.\n',
 ': crackle!@munnari..au ()\n: "Why I am not  " ( request)\n-: @..adelaide..au ( )\nOrganization:  &  ., University  \n: 4\n\n    wrote  article "Why I am not  "\n  a ?\n\nSorry, I accidently     forgot  .\n',
 'From: crackle!@munnari.oz.au (NAME)\nSubject: "Why I am not Bertrand Russell" (2nd request)\nReply-To: @augean.eleceng.adelaide..au (Derek Abbott)\nOrganization: Electrical & Electronic Eng., University of Adelaide\nLines: \n\nCould the guy   the article "Why I am not Bertrand Russell"\n me a copy?\n\nSorry, I   my copy and forgot your .\n',
 'From: crackle!dabbott@.oz.au (NAME)\nSubject: "Why I am not Bertrand Russell" (2nd request)\nReply-To: dabbott@augean.eleceng.adelaide.edu.au (Derek Abbott)\n: Electrical & Electronic Eng., University of Adelaide\nLines: 4\n\n the guy who wrote the article "Why I am not Bertrand Russell"\nresend  a copy?\n\nSorry, I accidently deleted my copy and forgot your name.\n',
 'From: crackle!dabbott@munnari.oz.au (NAME)\nSubject: "Why  am not Bertrand Russell" (2nd request)\n-: dabbott@augean.eleceng.adelaide.edu.au (Derek Abbott)\nOrganization: Electrical & Electronic Eng., University of \nLines: 4\n\nCould the guy who wrote the article "Why  am not Bertrand Russell"\nresend me a copy?\n\nSorry,  accidently deleted my copy and forgot your name.\n',
 'From: crackle!dabbott@.. (NAME)\n: " I  not  Russell" ( )\n-To: dabbott@augean.... (Derek Abbott)\n:  &  Eng., University  Adelaide\n: \n\n   who   article " I  not  Russell"\nresend  a copy?\n\n, I accidently  my copy and   name.\n',
 'From: crackle!dabbott@munnari.oz.au (NAME)\nSubject: "Why I am not Bertrand Russell" (2nd request)\nReply-To: dabbott@.eleceng.adelaide.edu.au (Derek Abbott)\nOrganization: Electrical & Electronic Eng., University of Adelaide\nLines: 4\n\nCould the guy who wrote the  "Why I am not Bertrand Russell"\n me a copy?\n\nSorry, I accidently deleted my copy and forgot  name.\n',
 'From: crackle!dabbott@munnari.oz.au ()\n: "Why I  not Bertrand Russell" (2nd request)\nReply-To: dabbott@augean.eleceng.adelaide.edu.au (Derek Abbott)\nOrganization: Electrical & Electronic Eng., University of Adelaide\n: 4\n\n    wrote  article "Why I  not Bertrand Russell"\nresend me a copy?\n\nSorry, I accidently deleted my copy and forgot your name.\n',
 'From: crackle!@..au ()\n: "Why I  not Bertrand Russell" (2nd request)\n-: @augean....au ( )\nOrganization: Electrical & Electronic ., University  Adelaide\nLines: \n\n   who   article "Why I  not Bertrand Russell"\n   ?\n\n, I  deleted     your .\n',
 ': !@.. ()\n: "     " ( )\n-: @.eleceng... ( )\n:  &  .,   \n: \n\n       "     "\n me  ?\n\n,         .\n']
```

​

샘플링된 $z$ 값을 해석하려는 모델 $f$ 에 넣어서 $g$를 학습할 때 사용할 $label$ 을 만듭니다.

$$label = f(z)$$

```python
labels = c.predict_proba(inverse_data)
```

예측된 label 값은 다음과 같습니다.
```python
> labels
array([[0.284, 0.716],
       [0.276, 0.724],
       [0.262, 0.738],
       [0.292, 0.708],
       [0.282, 0.718],
       [0.174, 0.826],
       [0.232, 0.768],
       [0.298, 0.702],
       [0.246, 0.754],
       [0.098, 0.902]])
```
​

### **Sparse Linear Explanations**

---

다음으로는 $g$를 학습시킬 때 필요한 $weight$를 계산해야 합니다.

위에서 random 하게 뽑힌 $z'$들이 원본 데이터와 거리가 얼마나 먼 곳에서 있는지에 따라서 학습할 때 가중치로 사용합니다.

$$\pi_{x}=exp(-D(x,z)^{2}/\sigma^{2}): \text{sample weight}$$

$D$는 거리를 계산하는 함수이며 각 데이터 특성별로 사용하는 Distance function은 다음과 같습니다.
- text: cosine distance
- image: $L2$ distance

​

#### **Distance**

우선 $x$ 와 $z$ 사이의 거리를 계산합니다. 이 예시는 text 이기 때문에 cosine distance 를 구했습니다.
```python
**import scipy as sp

distance_metric = "cosine"

def distance_fn(x):
    return sklearn.metrics.pairwise.pairwise_distances(x, x[0], metric=distance_metric).ravel() * 100

distances = distance_fn(sp.sparse.csr_matrix(data))
```

계산된 거리는 다음과 같습니다.
```python
> distances
array([ 0.        , 40.59114742,  9.2514787 ,  4.001634  ,  4.001634  ,
       32.84492632,  4.001634  ,  8.17749432, 35.83110521, 80.19704914])
```
​

#### **Kerenl function**

$pi_{x}$는 exponential kernel 입니다. 앞서 계산한 distance를 kernel에 넣어서 값을 변환시켜줍니다. 이 때 식의 sigma는 kernel width 로 해석됩니다.

```python
from functools import partial

def kernel(d, kernel_width):
    return np.sqrt(np.exp(-(d ** 2) / kernel_width ** 2))

kernel_width = 25
kernel_fn = partial(kernel, kernel_width=kernel_width)

weights = kernel_fn(distances)
```

계산된 weight는 다음과 같습니다.
```python
> weights
array([1.        , 0.26763986, 0.93381971, 0.98727124, 0.98727124,
       0.42188127, 0.98727124, 0.94790866, 0.35804576, 0.005827  ])
```
​

#### **Feature selection**

다음으로 계산해야 할 것은 $\Omega(g)$입니다. 논문에서는 이 부분을 K-LASSO로 대신했지만 실제 코드에서는 Ridge 또는 이용자가 준 값 k를 사용합니다. 설명할 변수들을 Ridge모델로 학습 후 선택합니다.

```python
from sklearn.linear_model import Ridge

labels_column = labels[:,1]
num_features = 10

clf = Ridge(alpha=0.01, fit_intercept=True, random_state=random_state)
clf.fit(data, labels_column, sample_weight=weights)

coef = clf.coef_

weighted_data = coef * data[0]
feature_weights = sorted(zip(range(data.shape[1]), weighted_data), reverse=True)

used_features = np.array([x[0] for x in feature_weights[:num_features]])
```

선택된 feature은 다음과 같습니다.
```python
> used_features
array([50, 49, 48, 47, 46, 45, 44, 43, 42, 41])
```
​

### **Train $g$**
---

선택된 변수들을 이용하여서 $g$를 학습합니다.

```python
easy_model = Ridge(alpha=1, fit_intercept=True, random_state=random_state)
easy_model.fit(data[:, used_features], labels_column, sample_weight=weights)
prediction_score = easy_model.score(data[:, used_features], labels_column, sample_weight=weights)
```

예측된 점수값은 다음과 같습니다.
```python
> prediction_score
0.7873828293020748
```

학습된 모델을 이용해 설명하려는 instance의 예측합니다.

```python
local_pred = easy_model.predict(data[0, used_features].reshape(1, -1))
```

예측된 값은 다음과 같습니다.
```python
> local_pred
array([0.71951834])
```

설명 변수들을 coefficient 크기를 기준으로 정렬합니다.

```python
local_exp = sorted(zip(used_features, easy_model.coef_), key=lambda x: np.abs(x[1]), reverse=True)
```

정렬된 변수들의 순서는 다음과 같습니다.
```python
> local_exp
[(49, -0.028394961697102435),
 (43, -0.016671889703629914),
 (48, -0.01667188970362991),
 (45, -0.011273894261784378),
 (44, -0.005261889646530821),
 (46, 0.003666088585078779),
 (47, 0.0036660885850787785),
 (42, 0.0036660885850787785),
 (41, -0.0025519331421568372),
 (50, 0.0009561320807047926)]
 ```

feature를 원래 값(text)으로 복원합니다.

```python
> domain_mapper.map_exp_ids(local_exp)
[('your', -0.028394961697102435),
 ('Sorry', -0.016671889703629914),
 ('forgot', -0.01667188970362991),
 ('deleted', -0.011273894261784378),
 ('accidently', -0.005261889646530821),
 ('my', 0.003666088585078779),
 ('and', 0.0036660885850787785),
 ('copy', 0.0036660885850787785),
 ('a', -0.0025519331421568372),
 ('name', 0.0009561320807047926)]
```

​
LIME의 해석에 관한 부분은 공식 github repo를 참조해주세요.

*https://github.com/marcotcr/lime*
