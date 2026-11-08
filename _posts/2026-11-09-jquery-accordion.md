---
layout: post
title: "jQuery 아코디언 메뉴 만들기"
excerpt: "메뉴 제목을 누르면 서브 메뉴가 펼쳐지고, 다른 메뉴는 자동으로 닫히는 아코디언 메뉴를 jQuery 두 줄로 만들어 봤습니다."
comments: true
---

[🪗 아코디언 메뉴 직접 눌러 보기](/assets/accordion.html)

# jQuery 아코디언 메뉴

메뉴 제목을 누르면 아래 서브 메뉴가 **펼쳐지고**, 다른 메뉴를 누르면 열려 있던 건 **자동으로 닫혀서** 항상 하나만 열려 있는 메뉴입니다. 수업에서 HTML과 CSS로 모양을 만들고, jQuery 코드는 지금까지 배운 걸로 채워서 완성했습니다.

## 핵심 포인트

* 서브 메뉴는 CSS에서 **`display: none`**으로 처음엔 숨겨 둔다.
* **`$(this).next('.sub')`** — 누른 제목 바로 다음에 있는 서브 메뉴만 고른다.
* **`$(this).parent().siblings()`** — 내 메뉴를 뺀 나머지 메뉴들을 고른다.
* **`slideToggle()`**은 열려 있으면 닫고, 닫혀 있으면 연다.
* 빠르게 여러 번 눌러도 출렁이지 않게 **`stop()`**을 먼저 쓴다.

## 1. HTML 구조

```html
<ul id="navi">
    <li>
        <div class="title">MainMenu1</div>
        <ul class="sub">
            <li><a href="#">sub1</a></li>
            <li><a href="#">sub2</a></li>
            <li><a href="#">sub3</a></li>
        </ul>
    </li>
    <li>
        <div class="title">MainMenu2</div>
        <ul class="sub"> ... </ul>
    </li>
    <!-- MainMenu3, MainMenu4도 같은 구조 -->
</ul>
```

```
ul#navi
 ├─ li ── div.title (MainMenu1)
 │     └─ ul.sub (sub1~3)
 ├─ li ── div.title (MainMenu2)
 │     └─ ul.sub
 ...
```

* 메뉴 하나가 `li` 하나이고, 그 안에 **제목(div.title)과 서브 메뉴(ul.sub)가 형제**로 들어 있습니다. 이 구조 덕분에 `next()`와 `siblings()`로 쉽게 찾을 수 있습니다.

## 2. CSS

```css
#navi { width: 200px; margin: 20px auto 0; }
#navi .title {
    height: 36px;
    line-height: 36px;          /* 높이와 같게 → 글자 세로 가운데 */
    background-color: #390;
    color: #fff;
    cursor: pointer;            /* 누를 수 있다는 표시 */
}
#navi .sub { display: none; }   /* 처음엔 서브 메뉴 닫기 */
#navi .sub li { height: 36px; line-height: 36px; background-color: #cf9; }
#navi .sub li a { display: block; width: 100%; }   /* 줄 전체를 눌러도 되게 */
#navi .sub li:hover a { background-color: #cf0; }
```

* `a`는 인라인이라 글자 부분만 눌리는데, `display: block`으로 바꿔서 **줄 전체**가 눌리게 했습니다.

## 3. jQuery

```javascript
$(function(){
    $('#navi .title').click(function(){
        // 누른 제목 바로 다음의 서브 메뉴 열고 닫기
        $(this).next('.sub').stop().slideToggle(300);

        // 다른 메뉴의 서브 메뉴는 닫기
        $(this).parent().siblings().find('.sub').stop().slideUp(300);
    });
});
```

| 코드 | 고르는 것 |
|---|---|
| `$(this)` | 지금 누른 제목 (div.title) |
| `.next('.sub')` | 그 제목 바로 다음 형제인 서브 메뉴 |
| `.parent()` | 제목을 감싼 `li` |
| `.siblings()` | 그 `li`를 뺀 나머지 `li`들 |
| `.find('.sub')` | 그 안에 있는 서브 메뉴들 |

* 첫 줄은 **누른 메뉴**를 열거나 닫고, 둘째 줄은 **나머지 메뉴**를 전부 닫습니다.
* `slideToggle`이라서 같은 제목을 한 번 더 누르면 다시 닫힙니다.
* `stop()`이 없으면 빠르게 연타했을 때 쌓인 애니메이션이 다 끝날 때까지 메뉴가 계속 출렁입니다.

## 정리

* 수업에서 만든 세로 메뉴(4_vmenu1)는 **마우스를 올리면** 열렸고, 아코디언은 **클릭으로** 연다는 게 다릅니다.
* jQuery 코드는 반드시 jQuery를 불러오는 `<script src>` **아래에** 써야 합니다. 순서가 바뀌면 `$ is not defined` 오류가 납니다.
* 같은 기능을 순수 JavaScript로 만들면 `nextElementSibling`, `classList.toggle()`과 CSS `transition`으로 바꿀 수 있습니다.
