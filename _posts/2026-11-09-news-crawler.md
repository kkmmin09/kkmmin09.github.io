---
layout: post
title: "네이버 뉴스 크롤러: 수집 → 감성 분석 → DB 저장"
excerpt: "requests와 BeautifulSoup으로 네이버 뉴스를 날짜·카테고리별로 모으고, 감성 사전으로 점수를 매겨 MySQL에 저장하는 크롤러 코드를 뜯어봤습니다."
comments: true
---

# 네이버 뉴스 크롤러

[뉴스 조회 프로그램](/News-Collection/)이 DB에 있는 기사를 **화면에 보여주는** 쪽이라면, 이 글은 그 기사를 **모아서 DB에 넣는** 쪽입니다. 날짜와 카테고리를 입력하면 네이버 뉴스 목록 1~5페이지를 돌면서 기사를 가져오고, 감성 점수를 매겨 MySQL에 저장한 뒤 다시 조회해서 보여줍니다.

## 핵심 포인트

* `requests.get(url).text`로 HTML을 가져오고 **BeautifulSoup**으로 원하는 태그를 꺼낸다.
* 기사 URL을 **`sha256`으로 해시**해서 저장하고, **`INSERT IGNORE`**로 같은 기사가 두 번 들어가지 않게 한다.
* 감성 점수는 **감성 사전(json)의 단어가 본문에 있으면 점수(polarity)를 더하는 방식**이다.
* URL 수집 / 기사 가져오기 / 감성 분석 / 저장 / 조회를 **함수 하나씩**으로 나눴다.
* 기사마다 **`time.sleep(0.3)`**으로 쉬어서 서버에 부담을 덜 준다.

## 동작 흐름

| 단계 | 함수 | 하는 일 |
|---|---|---|
| 1 | `get_urls` | 뉴스 목록 1~5페이지에서 기사 주소 모으기 |
| 2 | `get_article` | 기사 페이지에서 제목, 본문 꺼내기 |
| 3 | `analyze_sentiment` | 본문으로 감성 점수 계산 |
| 4 | `save_article` | URL 해시와 함께 DB에 저장 |
| 5 | `search_news` | 날짜·카테고리로 다시 조회해서 출력 |

## 0. 먼저 크롤링 기초: 영화 순위

본격적인 크롤러 전에 영화 순위 사이트로 기본기를 연습했습니다.

```python
import requests
from bs4 import BeautifulSoup

html = requests.get('https://m.moviechart.co.kr/rank/realtime/index/image').text
soup = BeautifulSoup(html, 'html.parser')

for b in soup.select("ul[class=movieBox-list] li[class=movieBox-item]"):
    rank = b.select('a p')[0].text
    title = b.select('div[class=movie-title] h3')[0].text
    link = b.select('a')[0].get('href')
    print("순위: " + rank + ", 제목: " + title + ", 주소: " + link)
```

* `select("CSS 선택자")`는 **리스트**를 돌려줘서 `[0]`으로 꺼냅니다.
* 글자는 `.text`, 속성 값은 `.get('href')`로 꺼냅니다.

## 1. 카테고리와 URL 해시

```python
category_map = {
    "정치": 100, "경제": 101, "사회": 102, "생활/문화": 103,
    "세계": 104, "IT/과학": 105, "연예": 106, "속보": 107, "오피니언": 110
}

def make_hash(url):
    return hashlib.sha256(url.encode()).hexdigest()
```

* 사람이 입력한 카테고리 이름을 **네이버 뉴스 코드**(정치 = 100)로 바꿉니다.
* URL은 길어서 그대로 UNIQUE 키로 쓰기 불편하니, **항상 같은 길이의 해시값**으로 바꿔서 비교합니다.

## 2. 기사 주소 모으기

```python
def get_urls(category_code, date):
    result = []
    for page in range(1, 6):
        list_url = (f"https://news.naver.com/main/list.naver?mode=LS2D"
                    f"&sid1={category_code}&mid=shm&date={date}&page={page}")
        try:
            soup = BeautifulSoup(requests.get(list_url, timeout=5).text, "lxml")
            for a in soup.select("a[href]"):
                href = a.get("href")
                if href and "/article/" in href:
                    if not href.startswith("http"):
                        href = "https://news.naver.com" + href   # 상대 주소면 앞에 붙이기
                    result.append((href, page))
        except Exception as e:
            print("URL 수집 실패:", e)
    return list(set(result))   # 중복 제거
```

* 링크 중에서 주소에 **`/article/`**이 들어간 것만 기사로 봅니다.
* 같은 기사가 여러 번 잡힐 수 있어서 **`set`으로 중복을 없앱니다.**

## 3. 기사 가져오기와 감성 분석

```python
def get_article(url):
    try:
        soup = BeautifulSoup(requests.get(url, timeout=5).text, "lxml")
        title_tag = soup.select_one("#title_area")
        content_tag = soup.select_one("#dic_area")
        if not title_tag:
            return None, None
        content = content_tag.get_text("\n", strip=True) if content_tag else ""
        return title_tag.get_text(strip=True), content
    except Exception as e:
        print("기사 가져오기 실패:", e)
        return None, None

def analyze_sentiment(content):
    if not content:
        return 0
    score = 0
    for item in sentiment_dict:
        if item["word_root"] in content:
            score += int(item["polarity"])
    return score
```

* 실패하면 프로그램을 멈추지 않고 **`(None, None)`**을 돌려줘서 그 기사만 건너뜁니다.
* 감성 사전에 있는 단어가 본문에 나오면 그 단어의 점수(+1, -1 등)를 더합니다.

## 4. DB에 저장

```python
def save_article(title, content, url, category, page):
    sentiment_score = analyze_sentiment(content)
    sql = """
    INSERT IGNORE INTO articles
    (title, content, url, url_hash, category, page, sentiment_score)
    VALUES (%s, %s, %s, %s, %s, %s, %s)
    """
    cursor.execute(sql, (title, content, url, make_hash(url), category, page, sentiment_score))
    conn.commit()
```

* 값은 SQL 문자열에 직접 붙이지 않고 **`%s`로 넘겨서** SQL 인젝션을 막습니다.
* `url_hash`가 UNIQUE라서 이미 있는 기사는 `INSERT IGNORE`가 조용히 넘어갑니다.

## 5. 실행

```python
if __name__ == "__main__":
    try:
        date = input("날짜 입력 (YYYY-MM-DD): ")
        category = input("카테고리 입력: ")
        crawl_news(date, category)          # 수집
        for i, row in enumerate(search_news(date, category), 1):   # 조회
            title, content, url, page, created_at, score = row
            print(f"[{i}] PAGE {page} | {title} | 감성 점수: {score}")
    finally:
        cursor.close()
        conn.close()
```

* 오류가 나도 **`finally`에서 DB 연결을 닫습니다.**

## 아쉬운 점

* `created_at`은 기사 작성일이 아니라 **DB에 저장한 시간**이라서, 지난 날짜를 크롤링하면 오늘 날짜로 조회해야 나옵니다. 기사 작성일 컬럼을 따로 저장하는 게 좋겠습니다.
* 감성 분석이 **단어가 들어 있는지만** 보기 때문에 "좋지 않다"처럼 부정문은 제대로 판단하지 못합니다.
* 네이버 페이지 구조(`#title_area`, `#dic_area`)가 바뀌면 선택자도 같이 고쳐야 합니다.
