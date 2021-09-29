---
title: 프로젝트별 pylint 설정하기
categories: [setup-module]
toc: true
date: 2021-03-14
author: Jongseob Jeon
---

**Module setup Contents 순서**
1. [setup.py 와 Makefile 설정하기](https://aiden-jeon.github.io/setup-module/makefile)
2. [프로젝트별 pylint 설정하기](https://aiden-jeon.github.io/setup-module/pylintrc)

---

이번 포스트에서는 `.pylintrc`를 이용해 프로젝트별 pylint를 설정하는 법에 대해 설명하겠습니다.
포스트에서 사용된 코드는 [github](https://github.com/aiden-jeon/github-cicd) 에서 확인할 수 있습니다.

## 1. generate rcfile
우선 `.pylintrc` 를 생성하겠습니다.
```bash
pylint --generate-rcfile > .pylintrc
```

`.pylintrc`를 확인해보면 여러가지 설정들이 있습니다.


## 2. pylint 사용
다음으로 지난번 포스트에서 만든 Makefile을 이용해 소스코드를 검사해보겠습니다.
```bash
> make lint
pytest src/ --pylint --flake8 --mypy
================= test session starts ==================
platform darwin -- Python 3.8.6, pytest-6.2.1, py-1.10.0, pluggy-0.13.1
rootdir: /Users/jongseob/workspace/github-cicd
plugins: flake8-1.0.7, mypy-0.8.0, cov-2.11.1, pylint-0.18.0
collected 4 items
--------------------------------------------------------
Linting files
.
--------------------------------------------------------

src/calculator.py F..s                             [100%]

======================= FAILURES =======================
______________ [pylint] src/calculator.py ______________
C:  1, 0: Missing module docstring (missing-module-docstring)
C:  4, 4: Argument name "a" doesn't conform to snake_case naming style (invalid-name)
C:  4, 4: Argument name "b" doesn't conform to snake_case naming style (invalid-name)
R:  4, 4: Method could be a function (no-self-use)
C: 26, 4: Argument name "a" doesn't conform to snake_case naming style (invalid-name)
C: 26, 4: Argument name "b" doesn't conform to snake_case naming style (invalid-name)
R: 26, 4: Method could be a function (no-self-use)
========================= mypy =========================
Success: no issues found in 1 source file
=============== short test summary info ================
FAILED src/calculator.py::PYLINT
======== 1 failed, 2 passed, 1 skipped in 0.19s ========
make: *** [lint] Error 1
```

3가지 error가 있는 것을 볼 수 있습니다.
- missing-module-docstring : class docstring이 없어서 나온 에러입니다.
- invalid-name : 변수 이름을 snake_case로 짓지 않아서 나온 에러입니다.
- no-self-use : self를 사용하지 않아서 나온 에러입니다.

에러를 발생하지 않기 위해서는 각 에러에 맞게 코드를 수정해야 합니다.
예를 들어서 invalid-name은 변수명 a와 b 대신 num_1, num_2 로 수정하면 해결 됩니다.

그런데 가끔은 이러한 변수명을 사용해야 할 경우가 있습니다.
이런 경우 해결 할 수 있는 방법은 두 가지가 있습니다.
1. 해당 라인을 검사하지 않게 하기
간단한 해법으로는 pylint가 해당 라인을 검사하지 않게 하는 것입니다.
원래의 코드가 아래와 같습니다.
```python
    def add(self, a: float, b: float) -> float:
```
여기에 주석을 추가해주면 됩니다.
```python
    def add(self, a: float, b: float) -> float:  # pylint: disable=invalid-name
```
이렇게 코드를 수정하면 pylint가 더 이상 해당 line에서 동일한 에러를 발생시키지 않습니다.


2. 에러 자체를 검사하지 않기
그런데 코드 한줄이 아니라 코드 전체에서 해당 lint를 검사하지 않게 하고 싶을 수도 있습니다. 이 때는 `.pylintrc`에 직접 해당 lint를 검사하지 않게 추가해주면 됩니다.
```
disable=invalid-name,
        ...
```
disable에 위와 같이 작성을 하면 더 이상 해당 lint를 검사하지 않습니다.

이번 포스트에서는 위에서 발생한 모든 에러를 `.pylintrc` 에서 검사하지 않도록 했습니다. 이제 다시 lint를 확인해보겠습니다.
```bash
> make lint
pytest src/ --pylint --flake8 --mypy
================= test session starts ==================
platform darwin -- Python 3.8.6, pytest-6.2.1, py-1.10.0, pluggy-0.13.1
rootdir: /Users/jongseob/workspace/github-cicd
plugins: flake8-1.0.7, mypy-0.8.0, cov-2.11.1, pylint-0.18.0
collected 4 items                                           
--------------------------------------------------------
Linting files
.
--------------------------------------------------------

src/calculator.py ....                             [100%]
========================= mypy =========================

Success: no issues found in 1 source file
================== 4 passed in 0.41s ===================
```
이제 더 이상 에러가 안나오는 것을 확인 할 수 있습니다.
