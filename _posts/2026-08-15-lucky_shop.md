---
layout: post
title: "운빨 상점 코드 분석 (HTML/CSS/JS)"
excerpt: "상자를 사서 랜덤 아이템을 얻는 미니 웹게임 '운빨 상점'의 코드를 HTML, CSS, JavaScript로 나눠서 뜯어봤습니다."
comments: true
---

[🎮 운빨 상점 플레이하기](/assets/운빨상점_플레이용.html)

# 운빨 상점 코드 분석

코인으로 **상자를 사서 랜덤 아이템**을 얻고, 인벤토리에서 **팔고**, **도감**을 채워가는 미니 웹게임입니다. HTML 한 파일 안에 구조(HTML), 스타일(CSS), 동작(JavaScript)이 모두 들어 있습니다.

## 핵심 포인트

* **등급 확률은 가중치 방식**으로 뽑는다. (일반 60.9 / 고급 25 / 희귀 10 / 영웅 4 / 전설 0.1, 합계 100)
* 상자는 **개봉 횟수로 해금**된다. (희귀 상자 10개, 전설 상자 30개)
* 게임 상태는 **`state` 객체 하나**로 관리하고, **`localStorage`에 저장**해서 새로고침해도 유지된다.
* 상태가 바뀌면 **`saveState()` → `renderAll()`** 순서로 저장 후 화면 전체를 다시 그린다.
* HTML에는 **빈 컨테이너만** 두고, 카드와 목록은 **JS가 만들어 채운다.**

## 게임 규칙

| 항목 | 내용 |
|---|---|
| 시작 코인 | 1,000 |
| 상자 | 일반 100 / 희귀 300 / 전설 700 코인 |
| 해금 | 희귀 상자 10개 개봉, 전설 상자 30개 개봉 |
| 구제 | 코인이 100 미만일 때만 5코인 받기 |
| 화면 | 상점 / 인벤토리 / 도감 (탭 3개) |

## 1. HTML — 구조

```html
<nav class="tab-bar">
  <button class="tab-btn active" data-tab="shop">상점</button>
  <button class="tab-btn" data-tab="inventory">인벤토리</button>
  <button class="tab-btn" data-tab="dex">도감</button>
</nav>

<main>
  <section id="tab-shop" class="tab-panel active">
    <div id="shop-boxes" class="box-grid"></div>
  </section>
  <section id="tab-inventory" class="tab-panel"> ... </section>
  <section id="tab-dex" class="tab-panel"> ... </section>
</main>
```

* 탭 버튼의 **`data-tab`** 이 어떤 패널을 열지 알려줍니다.
* **`shop-boxes`, `inventory-list`, `dex-grid`** 는 비워 두고 JS가 채웁니다.
* 상자를 열면 **복권 티켓 모양의 결과 모달**(`#reveal-modal`)이 뜹니다.

## 2. CSS — 스타일

* **`:root`의 CSS 변수**(`--ground`, `--panel`, `--accent`, 등급별 색 `--r-common` 등)로 색을 한곳에서 관리합니다.
* 탭 패널은 **`.active` 클래스가 붙은 것만** 보이고, 모달은 **`.hidden`** 으로 숨깁니다.
* 결과 카드는 **`.pop` 클래스를 붙여** 등장 애니메이션을 줍니다.

## 3. JavaScript — 동작

### 데이터 정의

```javascript
const RARITIES = [
  { key: 'common',    label: '일반', weight: 60.9 },
  { key: 'uncommon',  label: '고급', weight: 25 },
  { key: 'rare',      label: '희귀', weight: 10 },
  { key: 'epic',      label: '영웅', weight: 4 },
  { key: 'legendary', label: '전설', weight: 0.1 },
];

const BOXES = [
  { key: 'normal',    label: '일반 상자', cost: 100, unlock: { type: 'always' } },
  { key: 'rare',      label: '희귀 상자', cost: 300, unlock: { type: 'boxesOpened', value: 10 } },
  { key: 'legendary', label: '전설 상자', cost: 700, unlock: { type: 'boxesOpened', value: 30 } },
];
```

* 확률, 가격, 해금 조건이 **데이터로 분리**되어 있어서 숫자만 바꾸면 밸런스를 조절할 수 있습니다.
* 아이템은 `ITEM_POOL[상자][등급]`에 `[이름, 판매가]`로 들어 있고, 반복문으로 `ALL_ITEMS`와 `ITEM_BY_ID`(id로 빠르게 찾는 표)를 만듭니다.

### 상태 저장

```javascript
function defaultState() {
  return { coins: START_COINS, boxesOpened: 0, inventory: {}, dex: {} };
}

function loadState() {
  try {
    const raw = localStorage.getItem(SAVE_KEY);
    if (!raw) return defaultState();
    return { ...defaultState(), ...JSON.parse(raw) };
  } catch (e) {
    return defaultState();
  }
}
```

* 저장 데이터가 **없거나 깨져 있으면 기본값**으로 시작합니다.
* `{ ...defaultState(), ...parsed }`로 **빠진 항목은 기본값으로 채웁니다.**

### 가중치 추첨 (가장 중요한 부분)

```javascript
function pickRarity() {
  const total = RARITIES.reduce((s, r) => s + r.weight, 0);
  let roll = Math.random() * total;
  for (const r of RARITIES) {
    if (roll < r.weight) return r;
    roll -= r.weight;
  }
  return RARITIES[0];
}
```

전체 가중치만큼의 범위에서 난수를 뽑고, **등급 가중치를 차례로 빼면서** 어느 구간에 떨어지는지 찾습니다. 가중치가 클수록 구간이 넓어서 더 자주 뽑힙니다.

### 상자 구매

```javascript
function buyBox(boxKey) {
  const box = BOXES.find((b) => b.key === boxKey);
  if (!box || !isUnlocked(box) || state.coins < box.cost) return;

  state.coins -= box.cost;
  const rarity = pickRarity();
  const item = pickItem(box.key, rarity.key);

  const isNew = !state.dex[item.id];
  state.dex[item.id] = true;
  state.inventory[item.id] = (state.inventory[item.id] || 0) + 1;
  state.boxesOpened += 1;

  const justUnlocked = BOXES.filter(
    (b) => b.unlock.type === 'boxesOpened' && b.unlock.value === state.boxesOpened
  );

  saveState();
  renderAll();
  showReveal(box, rarity, item, isNew, justUnlocked);
}
```

게임의 핵심 흐름입니다.

1. **해금 여부와 코인**을 확인한다.
2. 코인을 차감하고 **등급 → 아이템** 순서로 뽑는다.
3. **인벤토리와 도감**에 반영한다. (처음 얻은 아이템이면 `NEW`)
4. 이번 개봉으로 **새로 해금된 상자**가 있는지 확인한다.
5. 저장하고, 화면을 다시 그리고, 결과 모달을 띄운다.

### 판매

```javascript
function sellItem(itemId, qty) {
  const owned = state.inventory[itemId] || 0;
  const sellQty = Math.min(qty, owned);      // 보유량 초과 방지
  if (sellQty <= 0) return;
  state.coins += ITEM_BY_ID[itemId].price * sellQty;
  state.inventory[itemId] -= sellQty;
  if (state.inventory[itemId] <= 0) delete state.inventory[itemId];
  saveState();
  renderAll();
}
```

* **`Math.min`** 으로 가진 수량보다 많이 팔 수 없게 합니다.
* 전체 판매(`sellAllInventory`)와 초기화(`resetGame`)는 **`confirm()` 확인창**을 거칩니다.

### 화면 그리기

```javascript
function renderAll() {
  renderHeader();     // 코인, 개봉 수, 도감 진행도
  renderShop();       // 상자 카드
  renderInventory();  // 보유 아이템 (비싼 순)
  renderDex();        // 도감 (미발견은 ???)
}
```

* 상태가 바뀔 때마다 **전체를 다시 그리는 단순한 구조**입니다.
* 잠긴 상자는 **해금 조건을 안내**하고, 코인이 부족하면 **구매 버튼을 비활성화**합니다.
* 도감은 발견하지 못한 아이템을 **`???`로 가려서** 수집 욕구를 만듭니다.

### 시작

```javascript
function init() {
  initTabs();        // 탭 전환
  el('#reveal-close').addEventListener('click', hideReveal);
  el('#reset-btn').addEventListener('click', resetGame);
  el('#coin-aid-btn').addEventListener('click', claimCoinAid);
  el('#sell-all-inv-btn').addEventListener('click', sellAllInventory);
  renderAll();       // 처음 한 번 그리기
}
document.addEventListener('DOMContentLoaded', init);
```

페이지가 로드되면 **버튼 이벤트를 연결하고 화면을 처음 한 번 그립니다.**

## 아쉬운 점

* 매번 **화면 전체를 다시 그려서** 아이템이 많아지면 비효율적일 수 있습니다.
* 저장이 **브라우저 `localStorage`에만** 되어 있어서 다른 기기에서는 이어서 할 수 없습니다.
