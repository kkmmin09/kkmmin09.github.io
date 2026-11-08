---
layout: post
title: "Java 모바일 기프티콘 지갑: 불변 객체로 버그 잡기"
excerpt: "민수가 기프티콘을 쓰면 지은이 잔액도 같이 줄어드는 버그를 재현하고, 불변 객체로 4단계에 걸쳐 고쳐 봤습니다."
comments: true
---

# 모바일 기프티콘 지갑

민수, 지은, 서연이가 메가커피 기프티콘을 같이 쓰기로 했는데, **민수가 쓰면 지은이 잔액도 같이 줄어드는** 이상한 버그가 생겼습니다. 왜 이런 일이 생기는지 확인하고, **불변 객체**로 고쳐 가는 실습입니다.

## 핵심 포인트

* 참조형 변수에 대입하면 객체가 복사되는 게 아니라 **주소(참조값)가 복사**돼서, 여러 변수가 **같은 객체**를 가리킨다.
* 그 상태에서 한쪽이 값을 바꾸면 다른 쪽도 바뀐다. 이게 **사이드 이펙트**다.
* 공유를 막을 수는 없으니, **객체를 아예 못 바꾸게** 만든다. (필드 `private final`, setter 없음)
* 값을 바꿔야 하면 **새 객체를 만들어 돌려주고**, 그 반환값을 꼭 다시 받는다.

## 단계별 흐름

| 단계 | 한 일 |
|---|---|
| Step 1 | 가변 객체(MutableGifticon)로 버그 재현 |
| 1단계 | 값이 안 바뀌는 ImmutableGifticon 만들기 |
| 2단계 | `use()` — 잔액을 뺀 **새 기프티콘** 돌려주기 |
| 3단계 | `withOwner()` — 주인만 바꾼 **새 기프티콘** 돌려주기 (선물하기) |
| 4단계 | `GifticonWallet` — List로 기프티콘을 모아 총 잔액 계산 |

## Step 1. 버그 재현

```java
public class MutableGifticon {
    private String brand;
    private int balance;

    public MutableGifticon(String brand, int balance) {
        this.brand = brand;
        this.balance = balance;
    }

    public void use(int amount) {
        this.balance = this.balance - amount;   // 자기 값을 직접 바꿈
    }
}
```

```java
MutableGifticon megacoffee = new MutableGifticon("메가커피", 30000);
MutableGifticon minsuGifticon = megacoffee;   // 주소 복사
MutableGifticon jieunGifticon = megacoffee;   // 같은 주소

minsuGifticon.use(15000);
// 민수 것 balance=15000, 지은 것도 balance=15000 (!!)
```

```
minsuGifticon ─┐
               ├──▶ [ 메가커피 | 30000 → 15000 ]
jieunGifticon ─┘
```

* 변수는 둘이지만 **객체는 하나**라서, 민수가 쓰면 지은이 것도 같이 줄어듭니다.
* 처음에는 `toString()`이 없어서 출력하면 `MutableGifticon@1b6d3586` 같은 값만 나와 **버그가 눈에 안 보였습니다.** `toString()`을 재정의하고 나서야 잔액이 같이 줄어드는 걸 확인할 수 있었습니다.

## 1~3단계. 불변 객체로 바꾸기

```java
public class ImmutableGifticon {
    private final String brand;      // final: 한 번 정하면 못 바꿈
    private final int balance;
    private final String owner;

    public ImmutableGifticon(String brand, int balance, String owner) {
        this.brand = brand;
        this.balance = balance;
        this.owner = owner;
    }

    // 2단계: 잔액을 뺀 새 기프티콘 (기존 객체는 그대로)
    public ImmutableGifticon use(int amount) {
        return new ImmutableGifticon(brand, balance - amount, owner);
    }

    // 3단계: 주인만 바뀐 새 기프티콘 (선물하기)
    public ImmutableGifticon withOwner(String newOwner) {
        return new ImmutableGifticon(brand, balance, newOwner);
    }

    public int getBalance() { return balance; }

    @Override
    public String toString() {
        return "ImmutableGifticon{brand='" + brand + "', balance=" + balance + ", owner='" + owner + "'}";
    }
}
```

* 필드를 모두 `private final`로 만들고 **setter를 없앴습니다.** 만든 뒤에는 절대 안 바뀝니다.
* `use()`는 자기 잔액을 바꾸지 않고, **잔액을 뺀 새 객체**를 만들어 돌려줍니다.
* **with로 시작하는 이름**은 "원래 것은 그대로 두고, 이 값만 바꾼 복사본을 만든다"는 뜻의 관례입니다. 자바의 `LocalDate.withYear()`도 같은 방식입니다. `set`으로 시작하면 원래 객체를 바꾸는 것처럼 보여서 불변 객체에서는 `with`를 씁니다.

```java
ImmutableGifticon minsuGifticon = megacoffee;
ImmutableGifticon jieunGifticon = megacoffee;

minsuGifticon = minsuGifticon.use(15000);   // 반환값을 꼭 다시 받기!

// 민수 것 balance=15000
// 지은 것 balance=30000  ← 이제 안 줄어듦

ImmutableGifticon seoyeonGifticon = minsuGifticon.withOwner("서연");   // 선물
```

## 4단계. 지갑으로 모으기

```java
import java.util.ArrayList;
import java.util.List;

public class GifticonWallet {
    private final List<ImmutableGifticon> gifticons = new ArrayList<>();

    public void add(ImmutableGifticon gifticon) {
        gifticons.add(gifticon);
    }

    public int totalBalance() {
        int total = 0;
        for (ImmutableGifticon g : gifticons) {
            total += g.getBalance();
        }
        return total;
    }

    public void printAll() {
        for (ImmutableGifticon g : gifticons) {
            System.out.println(g);
        }
    }
}
```

## 실행 결과

```txt
[민수]가 15000원 사용 후
민수 것 = ImmutableGifticon{brand='메가커피', balance=15000, owner='민수'}
지은 것 = ImmutableGifticon{brand='메가커피', balance=30000, owner='민수'}
[민수]가 서연에게 선물 후
서연 것 = ImmutableGifticon{brand='메가커피', balance=15000, owner='서연'}
[지갑 전체]
ImmutableGifticon{brand='메가커피', balance=30000, owner='민수'}
ImmutableGifticon{brand='메가커피', balance=15000, owner='서연'}
ImmutableGifticon{brand='스타벅스', balance=10000, owner='서연'}
총 잔액 = 55000원
```

## 정리

* 자바의 `String`, `Integer`도 불변 객체입니다. 그래서 `str.concat("java")`도 결과를 **다시 받아야** 바뀐 값을 쓸 수 있습니다.
* 불변 객체에서 가장 많이 하는 실수는 `minsuGifticon.use(15000);`처럼 **반환값을 안 받는 것**입니다. 그러면 사용한 금액이 그냥 사라집니다.

## 아쉬운 점

* 지금 `use()`는 잔액보다 큰 금액을 넣으면 잔액이 **음수**가 됩니다. `if (amount > balance)`로 막는 코드를 넣으면 좋겠습니다.
