---
layout: post
title: "PyQt5 + MySQL 뉴스 조회 프로그램 코드 분석"
excerpt: "감성 분석 기능이 포함된 뉴스 조회 GUI 프로그램 코드를 한 줄씩 뜯어봤습니다."
comments: true
---

# PyQt5 + MySQL 뉴스 조회 프로그램

**날짜와 카테고리**를 고르면 MySQL에서 기사를 조회하고, **감성 사전**으로 기사마다 감성 점수를 계산해 표에 보여주는 프로그램입니다. **제목을 더블클릭**하면 원본 기사가 브라우저로 열립니다.

## 핵심 포인트

* 화면(UI)은 **QtDesigner의 `.ui` 파일**을 `uic.loadUi`로 불러와 만든다.
* 카테고리 이름은 `category_map`으로 **DB 코드 번호**로 바꿔서 조회한다.
* SQL은 **`%s` 파라미터**로 값을 넘긴다. (SQL 인젝션 방지)
* 감성 점수는 **사전 단어가 본문에 있으면 극성(polarity)을 더하는 방식**이다.
* 기사 URL은 제목 칸의 **`Qt.UserRole`에 숨겨 저장**해서 더블클릭 때 꺼내 쓴다.
* 종료할 때 **`finally`로 DB 연결을 닫는다.**

## 동작 흐름

| 단계 | 내용 |
|---|---|
| 1 | 검색 버튼 클릭 → 날짜, 카테고리 읽기 |
| 2 | DB 조회 (`search_news`) |
| 3 | 기사별 감성 점수 계산 (`analyze_sentiment`) |
| 4 | 표에 제목 / 카테고리 / 작성일 / 감성 점수 출력 |
| 5 | 제목 더블클릭 → 숨겨둔 URL을 브라우저로 열기 |

## 1. 준비: 감성 사전과 DB 연결

```python
UI_FILE = r"C:\Users\user\Desktop\ㄱㄱㄱㅁui.ui"
SENTIMENT_FILE = r"C:\Users\user\Desktop\sentiment.json"

with open(SENTIMENT_FILE, "r", encoding="utf-8") as f:
    sentiment_dict = json.load(f)

conn = pymysql.connect(host="xxx.xxx.xxx.xxx", user="xxxx", password="xxxxxx",
                       db="xxxxxxxxxxxx", port=xxxx, charset="xxxx")
cursor = conn.cursor()
```

* 감성 사전(JSON)을 **한 번만 읽어** 메모리에 둡니다.
* DB 접속 정보는 공개 글이므로 **가려서 표시**했습니다.

## 2. 뉴스 조회

```python
category_map = {"정치": 100, "경제": 101, "사회": 102, "생활/문화": 103,
                "세계": 104, "IT/과학": 105, "연예": 106, "속보": 107, "오피니언": 110}

def search_news(date, category_name):
    if category_name not in category_map:
        return []
    category_code = category_map[category_name]

    sql = """
    SELECT title, content, url, page, created_at
    FROM articles
    WHERE DATE(created_at) = %s
      AND category = %s
    ORDER BY page ASC, id DESC
    """
    cursor.execute(sql, (date, category_code))
    return cursor.fetchall()
```

* 지정한 **날짜 + 카테고리**의 기사만 가져오고, **`page` 오름차순 → `id` 내림차순**으로 정렬합니다.
* 값은 문자열로 이어 붙이지 않고 **`%s`로 전달**합니다.

## 3. 감성 점수 계산

```python
def analyze_sentiment(content):
    if not content:
        return 0
    score = 0
    for item in sentiment_dict:
        word_root = item["word_root"]
        polarity = int(item["polarity"])
        if word_root in content:
            score += polarity
    return score
```

* 사전의 단어(`word_root`)가 **본문에 포함되면 `polarity`를 더합니다.**
* 점수가 **높을수록 긍정, 낮을수록 부정**입니다.
* 같은 단어가 여러 번 나와도 **한 번만** 더해집니다.

## 4. 검색 결과를 표에 채우기

```python
title_item = QTableWidgetItem(str(title))
title_item.setData(Qt.UserRole, url)          # 화면엔 안 보이게 URL 저장

self.tableWidget.setItem(row, 0, title_item)                           # 제목
self.tableWidget.setItem(row, 1, QTableWidgetItem(str(category)))      # 카테고리
self.tableWidget.setItem(row, 2, QTableWidgetItem(str(created_at)))    # 작성일
self.tableWidget.setItem(row, 3, QTableWidgetItem(str(sentiment_score)))  # 감성 점수
```

* 결과가 없으면 **안내 메시지**를 띄우고, 오류가 나면 **오류 창**을 띄웁니다.
* 다 채운 뒤 `resizeColumnsToContents()`로 열 너비를 맞춥니다.

## 5. 제목 더블클릭 → 기사 열기

```python
def title_double_clicked(self, row, column):
    if column != 0:
        return
    item = self.tableWidget.item(row, column)
    if item is None:
        return

    url = item.data(Qt.UserRole)
    if url:
        webbrowser.open(str(url))
    else:
        QMessageBox.warning(self, "URL 오류", "해당 기사에 저장된 URL이 없습니다.")
```

**제목 열(0번)** 만 반응하고, 숨겨둔 URL이 있으면 브라우저로 엽니다.

## 6. 실행과 종료

```python
if __name__ == "__main__":
    try:
        app = QApplication(sys.argv)
        window = MyWindow()
        window.show()
        sys.exit(app.exec_())
    finally:
        cursor.close()
        conn.close()
```

정상 종료든 오류든 **DB 커서와 연결을 닫습니다.**

## 아쉬운 점

* 감성 점수가 **단어 포함 여부만** 보기 때문에 문맥이나 부정 표현("좋지 않다")은 반영하지 못합니다.
* 기사마다 사전 전체를 도는 구조라 **기사나 사전이 많아지면 느려질 수 있습니다.**
