---
layout: post
title: "파이썬 스레드: 30초 걸리던 작업을 3초로"
excerpt: "3초짜리 작업 10개를 순서대로 돌리면 30초, 스레드로 동시에 돌리면 3초. threading 모듈의 함수형과 클래스형 스레드를 비교해 봤습니다."
comments: true
---

# 파이썬 스레드 (threading)

**스레드**는 프로그램 안에서 여러 작업을 **동시에** 실행하는 방법입니다. 3초 걸리는 작업 10개를 하나씩 하면 30초가 걸리지만, 스레드 10개로 동시에 시작하면 약 3초 만에 끝납니다.

## 핵심 포인트

* 함수형: **`threading.Thread(target=함수, name=..., args=(...))`**를 만들고 **`start()`**
* 클래스형: **`threading.Thread`를 상속**받고 **`run()`**을 만들면, `start()` 할 때 `run()`이 실행된다.
* 동시에 실행되기 때문에 **출력 순서가 매번 섞일 수 있다.**
* 그냥 함수를 부르면(`worker()`) 스레드가 아니라 **순서대로** 실행된다.

## 1. 일하는 함수

```python
import threading
import time

def worker(msg, slow):
    print("%s 실행 중 : %s" % (threading.current_thread().name, msg))
    time.sleep(slow)      # slow초 동안 기다리기 (오래 걸리는 작업 흉내)
    print("%s 종료" % threading.current_thread().name)
```

* 실행 중 출력 → `slow`초 대기 → 종료 출력 순서로 동작합니다.
* `threading.current_thread().name`으로 지금 이 코드를 돌리는 스레드 이름을 볼 수 있습니다.

## 2. 그냥 부르면: 30초

```python
def main():
    for i in range(10):
        msg = "작업 %d" % i
        worker(msg, 3)      # 하나 끝나야 다음 것 시작

if __name__ == "__main__":
    main()
```

* `worker`를 그냥 부르면 하나가 끝나야 다음 것이 시작돼서 **3초 × 10개 = 약 30초**가 걸립니다.
* 이름은 전부 메인 스레드(`MainThread`)로 나옵니다.

## 3. 함수형 스레드: 3초

```python
def main():
    for i in range(10):
        msg = "함수형 스레드 %d" % i
        th = threading.Thread(target=worker, name="[스레드 이름 %d]" % i, args=(msg, 3))
        th.start()          # 시작만 시키고 바로 다음으로

if __name__ == "__main__":
    main()
```

| 매개변수 | 뜻 |
|---|---|
| `target` | 스레드에서 실행할 함수 이름 (괄호 X) |
| `name` | 스레드 이름 |
| `args` | 함수에 넘길 값 (튜플이라서 하나만 넘길 때도 `(값,)`) |

* `start()`는 스레드를 시작시키고 **기다리지 않고 바로** 다음 줄로 갑니다. 그래서 10개가 거의 동시에 시작돼서 **약 3초**면 끝납니다.

## 4. 클래스형 스레드

```python
class Worker(threading.Thread):
    def __init__(self, args, name=""):
        super().__init__()          # 부모(Thread) 초기화 먼저
        self.name = name
        self.args = args

    def run(self):                  # start() 하면 이게 실행됨
        print("{} 실행 중 : {}".format(threading.current_thread().name, self.args))
        time.sleep(3)
        print("{} 종료".format(threading.current_thread().name))

def main():
    for i in range(10):
        th = Worker(name="[스레드 이름 {}]".format(i), args="클래스형 스레드 {}".format(i))
        th.start()

if __name__ == "__main__":
    main()
```

* `Thread`를 상속받아서 **`run()`에 할 일**을 적습니다.
* `start()`를 부르면 파이썬이 새 스레드를 만들고 그 안에서 `run()`을 실행합니다. `run()`을 직접 부르면 스레드가 아니라 그냥 함수 호출이 됩니다.
* 생성자에서 **`super().__init__()`**을 빼먹으면 에러가 납니다.

## 실행 결과 예시 (스레드)

```txt
[스레드 이름 0] 실행 중 : 함수형 스레드 0
[스레드 이름 1] 실행 중 : 함수형 스레드 1
[스레드 이름 2] 실행 중 : 함수형 스레드 2
...
[스레드 이름 1] 종료
[스레드 이름 0] 종료
[스레드 이름 3] 종료
...
```

* "실행 중"이 먼저 10줄 쭉 나오고, 3초 뒤에 "종료"가 몰려서 나옵니다.
* 종료 순서는 실행할 때마다 달라집니다.

## 함수형 vs 클래스형

| | 함수형 | 클래스형 |
|---|---|---|
| 만드는 법 | `Thread(target=함수)` | `Thread` 상속 + `run()` |
| 어울리는 경우 | 간단한 작업 하나 | 스레드마다 데이터나 상태를 들고 있어야 할 때 |

## 정리

* [OpenCV 웹캠 프로그램](/OpenCV/)에서 PyQt의 `QThread`를 쓴 것도 같은 이유입니다. 오래 걸리는 작업을 따로 돌려야 화면이 멈추지 않습니다.
* `threading.currentThread().getName()`은 예전 방식이고, 요즘은 `threading.current_thread().name`을 씁니다.
* 모든 스레드가 끝날 때까지 기다리려면 스레드마다 **`join()`**을 부르면 됩니다.
