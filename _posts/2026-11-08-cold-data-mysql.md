---
layout: post
title: "감기 진료 데이터: CSV → 딕셔너리 → MySQL JOIN"
excerpt: "공공 감기 진료 데이터 CSV 3개를 파이썬 딕셔너리로 먼저 조회해 보고, 같은 데이터를 MySQL에 넣어 JOIN으로 다시 조회해 봤습니다."
comments: true
---

# 감기 진료 데이터 조회

지역과 날짜를 입력하면 그 지역의 **시군구별 감기 진료 건수와 합계**를 보여주는 프로그램입니다. 같은 문제를 두 가지 방법으로 풀었습니다.

1. CSV 파일을 읽어서 **파이썬 딕셔너리**로 찾기
2. 같은 데이터를 **MySQL 테이블**에 넣고 **JOIN**으로 찾기

## 사용한 데이터

| 파일 | 내용 | 예시 |
|---|---|---|
| 시도 지역코드.csv | 시도 코드, 시도 이름 | 11, 서울 |
| 시군구 지역코드.csv | 시도 코드, 시군구 코드, 시군구 이름 | 11, 11110, 종로구 |
| 실제진료정보_감기_시군구.csv | 날짜, 시군구 코드, 진료 건수 | 20140101, 11110, 52 |

## 핵심 포인트

* 엑셀에서 만든 CSV라서 **`encoding="cp949"`**로 열고, 첫 줄(제목)은 **`next(f)`**로 건너뛴다.
* 용도에 맞게 딕셔너리를 따로 만든다. (이름 → 코드, 코드 → 이름, 시도 → 시군구 목록)
* **`(날짜, 지역코드)` 튜플을 key**로 써서 원하는 값을 정확하게 찾는다.
* DB에는 진료 데이터처럼 양이 많은 건 **리스트에 모았다가 `executemany`로 한 번에** 넣는다.
* 조회는 sido · sigungu · medical_data **세 테이블을 JOIN**한다.

## 1. 파이썬 딕셔너리로 풀기

```python
d = {}            # 시도 이름 → 시도 코드
with open("시도 지역코드.csv", "r", encoding="cp949") as f:
    next(f)
    for line in f:
        c, n = line.strip().split(",")
        d[n] = c

b = {}            # 시군구 코드 → 시군구 이름
region_map = {}   # 시도 코드 → 시군구 코드 리스트
with open("시군구 지역코드.csv", "r", encoding="cp949") as s:
    next(s)
    for line in s:
        a, c, n = line.strip().split(",")
        b[c] = n
        if a not in region_map:
            region_map[a] = []
        region_map[a].append(c)

data = {}         # (날짜, 시군구 코드) → 진료 건수
with open("실제진료정보_감기_시군구.csv", "r", encoding="cp949") as w:
    next(w)
    for line in w:
        y, c, i = line.strip().split(",")
        data[(y, c)] = int(i)

ct = input("지역을 입력하세요: ")
gm = input("날짜를 입력하세요: ")

total = 0
for c in region_map[d[ct]]:
    if (gm, c) in data:            # 없는 데이터는 건너뛰기 (KeyError 방지)
        print(b[c], data[(gm, c)])
        total += data[(gm, c)]
print("합계", total)
```

* "서울"을 넣으면 `d["서울"]` → `"11"` → `region_map["11"]` → 서울의 시군구 코드 목록이 나옵니다.
* 그 코드를 하나씩 돌면서 `(날짜, 코드)`로 진료 건수를 찾아 더합니다.
* 날짜 데이터가 없는 시군구도 있어서 **`in`으로 먼저 확인**하지 않으면 KeyError가 납니다.

## 2. MySQL에 넣기

```python
import pymysql

conn = pymysql.connect(host='****', port=3306, user='root', password='****',
                       database='****', charset='utf8')
cursor = conn.cursor()

# 시도: 이미 있으면 넘어가기
with open("시도 지역코드.csv", "r", encoding="cp949") as f:
    next(f)
    for line in f:
        code, name = line.strip().split(",")
        cursor.execute("INSERT IGNORE INTO sido (sido_code, sido_name) VALUES (%s, %s)",
                       (int(code), name))
conn.commit()

# 진료 데이터: 양이 많아서 리스트에 모았다가 한 번에
data_list = []
with open("실제진료정보_감기_시군구.csv", "r", encoding="cp949") as ga:
    next(ga)
    for line in ga:
        y, c, i = line.strip().split(",")
        data_list.append((y, int(c), int(i)))

cursor.executemany(
    "INSERT INTO medical_data (dt, sigungu_code, patient_count) VALUES (%s, %s, %s)",
    data_list
)
conn.commit()
```

* `INSERT IGNORE`는 같은 키가 이미 있으면 에러 대신 그냥 넘어갑니다.
* 한 줄씩 `execute`하면 느려서, 진료 데이터는 **`executemany`로 한 번에** 넣었습니다.
* `commit()`을 해야 실제로 저장됩니다.

## 3. JOIN으로 조회

```python
ct = input("지역을 입력하세요 (예: 서울): ")
dt = input("날짜를 입력하세요 (예: 20140101): ")

sql = """
SELECT g.sigungu_name, m.patient_count
FROM medical_data m
JOIN sigungu g ON m.sigungu_code = g.sigungu_code
JOIN sido s ON g.sido_code = s.sido_code
WHERE s.sido_name = %s AND m.dt = %s
"""
cursor.execute(sql, (ct, dt))

total = 0
for name, count in cursor.fetchall():
    print(name, count)
    total += int(count)
print("합계:", total)
```

* 딕셔너리 3개로 하던 일을 **JOIN 두 번**으로 합니다. 진료 데이터 → 시군구 이름 → 시도 이름 순서로 연결됩니다.
* 처음에는 시도로만 조회하고 파이썬에서 날짜를 걸렀는데, **WHERE에 날짜까지 넣으면 DB가 바로 걸러 줘서** 더 빠릅니다.

## 두 방법 비교

| | 딕셔너리 | MySQL |
|---|---|---|
| 준비 | 실행할 때마다 CSV를 다시 읽음 | 한 번 넣어 두면 계속 씀 |
| 찾기 | 딕셔너리 3개를 차례로 따라감 | JOIN 한 문장 |
| 데이터가 많아지면 | 메모리를 많이 씀 | DB가 알아서 관리 |

## 아쉬운 점

* 없는 지역 이름을 넣으면 `d[ct]`에서 KeyError가 납니다. `if ct not in d:`로 먼저 확인하면 좋겠습니다.
* 진료 데이터를 넣을 때 외래키 검사를 끄고(`SET FOREIGN_KEY_CHECKS = 0`) 넣었는데, 시군구 코드가 맞지 않는 데이터가 섞여도 모르고 지나갈 수 있습니다.
