---
title: setup.py 와 Makefile 설정하기
comment: true
categories: [setup-module]
toc: true
weight: 1
date: 2021-03-14
author: Jongseob Jeon
---

**Module setup Contents 순서**
1. [setup.py 와 Makefile 설정하기](https://aiden-jeon.github.io/setup-module/makefile)
2. [프로젝트별 pylint 설정하기](https://aiden-jeon.github.io/setup-module/pylintrc)

---

이번 포스트에서는 setup.py 와 Makefile을 설정하는 법에 대해 소개하겠습니다. 포스트에서 사용된 코드는 [github](https://github.com/aiden-jeon/github-cicd) 에서 확인할 수 있습니다.

## 1. `setup.py`
`setup.py` 에서 사용되는 `setuptools.setup`은 개발하고 있는 소스 코드를 package 형식으로 배포 할 때 이용합니다.
```python
# setup.py
from setuptools import find_packages, setup


setup(
    name="github_cicd",
    version="0.0.1",
    packages=find_packages("src"),
    python_requires=">=3.*",
    author="Aiden-jeon",
    author_email="ells2124@gmail.com",
    description="Example of github-cicd",
    keywords="ML pipeline kubeflow automation",
    url="https://github.com/aiden-jeon/github-cicd",
    project_urls={
        "Documentation": "https://github.com/aiden-jeon/github-cicd",
        "Source Code": "https://github.com/aiden-jeon/github-cicd",
    },
)

```
- `name`: 패키지가 어떤 이름으로 설치 될지를 나타냅니다.
- `version`: 배포된 소스코드의 버전입니다.
- `packages`: 배포에 포함 될 package들을 의미한다. 보통 `setuptools.find_packages`를 이용해 자동으로 찾습니다.
	- 예를 들어서 src 폴더 밑에 `module_1`, `module_2` 가 있다면 두 모듈을 모두 찾아서 설치를 해줍니다. 이렇게 설치된 경우 아래와 같이 사용할 수 있습니다.
		```python
		import module_1
		import module_2
		```
	- 만약, src 폴더 없이 `module_1` 폴더를 설치하고 싶을 경우에는 아래와 같이 설정을 해야 합니다.
		```python
		packages=["module_1"]
		```

- `python_requires`: 실행 가능한 파이썬 버전을 의미합니다.

아래 코드를 이용해 작성이 완료된 `setup.py`를 설치할 수 있습니다.
```python
pip install - e .
```

사용한 python 버전의 site-packages에 가면 아래와 같이 패키지가 설치되어 있는 것을 확인 할 수 있습니다.
```bash
> ls|grep github

github-cicd.egg-link
```

## 2. Makefile
이번에는 Makefile을 작성해보겠습니다.

## 2.1 Makefile을 사용하는 이유
> Compiling the source code files can be tiring, especially when you have to include several source files and type the compiling command every time you need to compile. Makefiles are the solution to simplify this task.
> 
> Makefiles are special format files that help build and manage the projects automatically.  
> 출처: [link](https://www.tutorialspoint.com/makefile/why_makefile.htm#:~:text=Compiling%20the%20source%20code%20files,and%20manage%20the%20projects%20automatically)

요약하면 compile을 반복적으로 해야할 때 이러한 작업을 간단히 하기 위해 사용합니다.

## 2.2 코드 작성
Python에서는 협업을 위한 여러가지 tool이 존재합니다. 
이러한 tool을 이용해 코드의 일관성, 타입체크, 코드 테스트를 수행할 수 있습니다.
그런데 이 과정이 반복적으로 이루어지기 때문에 이를 코드화하려고 합니다.

- init
Makefile을 수행하기 위해서는 제일 처음 필요한 패키지와 소스코드를 설치해야 합니다.
```bash
init:
	pip install -U pip
	pip install -r requirements.txt
	pip install -e .
```

- format
코드 format을 맞추기 위해 사용되는 툴은 주로 black 과 isort 가 있습니다. 
이 때 isort 가 black과 충돌을 일으키지 않게 `--profile black`을 추가합니다.
```bash
format:
	black .
	isort . --skip-gitignore --profile black
```

- lint
linting을 검사하기 위해 pytest 를 이용하고 pylint, flake8, mypy 를 사용하겠습니다.
```bash
lint:
	pytest src/mrxflow/ --pylint --flake8 --mypy
```

- test
test를 수행하기 위해 pytest를 이용하겠습니다.
```bash
test:
	pytest tests -s --verbose --cov=src/ --cov-report=html --cov-report=term-missing
```

전체 코드는 다음과 같습니다.
```bash
SHELL := /bin/bash

all:
	echo 'Makefile for github-cicd'

init:
	pip install -U pip
	pip install -r requirements.txt
	pip install -e .

format:
	black .
	isort . --skip-gitignore --profile black

lint:
	pytest src/mrxflow/ --pylint --flake8 --mypy

test:
	pytest tests -s --verbose --cov=src/ --cov-report=html --cov-report=term-missing
```

## 3. 사용하기
`setup.py`와 Makefile의 작성을 다 했다면 다음과 같이 사용할 수 있습니다.

1. init
Makefile에서 수행할 때 필요한 패키지를 설치합니다.
```bash
make init
```

2. 다른 명령어 사용해보기

- format
```bash
make format
```

- lint
```bash
make lint
```

- test
```bash
make test
```
