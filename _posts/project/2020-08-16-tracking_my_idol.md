---
title: 동영상에서 원하는 인물 따라가기
comment: true
categories: [project]
toc: true
toc_sticky: true
---

## 0. 참고 자료 검색
-  [빵형의 개발도상국](https://www.youtube.com/watch?v=cx7VONjFEE0&t=83s)

## 1. mp4 동영상 다운로드 

- youtube 영상 다운로드
  - ssyoutube로 바꾸면 다운로드 할 수 있는 사이트로 연결됨.
  - https://www.youtube.com/watch?v=naRRqAGIAqQ
  - https://www.ssyoutube.com/watch?v=naRRqAGIAqQ

## 2. 프로젝트를 위한 가상환경 만들기

```bash
pyenv virtualenv 3.7.8 tracking_idol
pyenv activate tracking_idol
pip install opencv-python==4.1.2.30 opencv-contrib-python==4.1.2.30 numpy
```
- opencv 최신 버전은 mac catalina 버전에서 실행이 안된다.
  - https://solarianprogrammer.com/2019/10/21/install-opencv-python-macos/
3. opencv 를 이용해서 동영상 실행시켜보기

```python
import cv2

video_path = "tracking_myidol/dumbi_dumbdi.mp4"
cap = cv2.VideoCapture(video_path)

ret, img = cap.read()
cv2.imshow("img", img)
while cap.isOpened():
    ret, img = cap.read()

    if not ret:
        exit()

    cv2.imshow("img", img)
    if cv2.waitKey(1) == ord("q"):
        break

```

## 3. ROI Setting 

- ROI (Region of Interest)
  - 관심 영역, 영상처리할 때 쓰는 보편적인 용어

```python
ret, img = cap.read()
cv2.namedWindow("Select Window")
cv2.imshow("Select Window", img)
rect = cv2.selectROI("Select Window", img, fromCenter=False, showCrosshair=True)
cv2.destroyWindow("Select Window")
```

동영상 제일 처음 frame을 불러와서 cv2.selectROI 를 이용해 처음에 ROI 를 세팅할 수 있는 창을 만든다.

## 4. tracker

user가 선택한 ROI를 rect 라는 변수에 저장을 하는데, 이를 계속해서 추적하는 obejct tracker가 필요하다.

```python
# init tracker
tracker = cv2.TrackerCSRT_create()
tracker.init(img, rect)

while cap.isOpened():
    ret, img = cap.read()

    if not ret:
        exit()

    success, box = tracker.update(img)
    (left, top, w, h) = list(map(int, box))
    cv2.rectangle(
        img,
        pt1=(left, top),
        pt2=(left + w, top + h),
        color=(255, 255, 255),
        thickness=3,
    )
    cv2.imshow("img", img)
    if cv2.waitKey(1) == ord("q"):
        break

```

이렇게 하면 Select Window 창에서 선택된 ROI를 계속해서 tracking 하는 것을 볼 수 있다.



## 5. 예상되는 문제점 

- 첫 frame에 원하는 인물이 나오지 않을 경우 다음 frame으로 넘길 수 있는 기능이 필요 (2020.08.17 해결)
  - while loop으로 q 외에 다른 키를 누를 경우 frame 넘길 수 있게 처리함.
  - 그리고 rect가 없을 경우 (0,0,0,0) 의 값을 갖는데 bounding box가 (0,0,0,0) 일 경우는 없다고 판단해서 합이 0이 아니면 break로 while loop 을 break 시킴.

```python
cv2.namedWindow("Select Window")
while cap.isOpened():
    i += 1
    print(i)
    ret, img = cap.read()
    cv2.imshow("Select Window", img)
    key = cv2.waitKey(10)
    # Quit when 'q' is pressed
    if key == ord('q'):
        break
    rect = cv2.selectROI("Select Window", img, fromCenter=False, showCrosshair=True)
    print(rect)
    if sum(rect) != 0:
        cv2.destroyWindow("Select Window")
        break
```

- 처음 ROI 설정 할 경우 앉은 상태에서 시작할 때, 일어나는 사람의 전체 size를 tracking 하지 못함
- 인물이 다른 인물에 가려져서 안보일 경우 어떻게 처리해야 할지.


## 6. 전체코드

```python
import cv2

video_path = "tracking_myidol/dumbi_dumbdi.mp4"
cap = cv2.VideoCapture(video_path)
cap.set(cv2.CAP_PROP_POS_FRAMES, 80)


## ROI Setting
if not cap.isOpened:
    exit()

cv2.namedWindow("Select Window")
while cap.isOpened():
    ret, img = cap.read()
    cv2.imshow("Select Window", img)
    key = cv2.waitKey(10)
    # Quit when 'q' is pressed
    if key == ord('q'):
        break
    rect = cv2.selectROI("Select Window", img, fromCenter=False, showCrosshair=True)
    print(rect)
    if sum(rect) != 0:
        cv2.destroyWindow("Select Window")
        break

# init tracker
tracker = cv2.TrackerCSRT_create()
tracker.init(img, rect)

while cap.isOpened():
    ret, img = cap.read()

    if not ret:
        exit()

    success, box = tracker.update(img)
    (left, top, w, h) = list(map(int, box))
    cv2.rectangle(
        img,
        pt1=(left, top),
        pt2=(left + w, top + h),
        color=(255, 255, 255),
        thickness=3,
    )
    cv2.imshow("img", img)
    if cv2.waitKey(1) == ord("q"):
        break

```
