---
title: "Playwright Flaky 테스트 줄이기 — auto-waiting·web-first assertion 원리와 네트워크 모킹(route) 실전 패턴"
excerpt: "같은 코드가 10번 중 3번 실패하는 이유는 대개 '기다림을 시간으로 설계했기' 때문입니다. Playwright가 클릭 전에 확인하는 6가지 조건과 web-first assertion의 재시도 루프를 실측 로그로 확인하고, page.route·HAR로 비결정적 네트워크를 고정하는 패턴과 Trace Viewer 진단 절차를 정리합니다."
categories: [QA]
tags: [Playwright, FlakyTest, E2E테스트, auto-waiting, page.route, HAR, TraceViewer, pytest]
permalink: /qa/playwright-flaky-tests-auto-waiting-and-route/
toc: true
toc_sticky: true
date: 2026-11-28 08:30:00 +0900
---

## 들어가며: 같은 코드가 왜 10번 중 3번 실패하는가

E2E 테스트를 붙이고 CI에 올리면 반드시 이 단계를 지난다. 로컬에서는 초록불이던 테스트가 CI에서만 빨간불이 되고, 다시 돌리면 통과한다. 팀은 `sleep(1)`을 `sleep(3)`으로 늘리고, 그다음에는 타임아웃을 30초로 올린다. 잠시 조용해졌다가 몇 주 뒤 같은 테스트가 다시 깨진다.

이 글의 결론을 먼저 적으면 이렇다.

> Flaky의 원인은 "느린 환경"이 아니라 **"시간을 기다리고 상태를 기다리지 않는 코드"** 다.
> 그래서 처방도 타임아웃 늘리기가 아니라 **기다릴 조건을 명시하는 것**이다.

이 글은 그 처방을 네 층으로 나눠 다룬다.

```text
[1] 왜 Flaky가 생기는가        DOM 파싱 · 하이드레이션 · 애니메이션 · 네트워크 지터
        ↓
[2] Playwright는 어떻게 기다리는가   Actionability 6조건 + web-first assertion 폴링
        ↓
[3] 비결정적 네트워크를 어떻게 고정하는가  page.route · route.fetch · HAR · expect_response
        ↓
[4] 실패했을 때 무엇을 보는가   Trace Viewer (액션 타임라인 + 네트워크 + 스냅샷)
```

원리를 확인하려고 로컬에서 지연 응답 서버를 띄우고 Playwright를 실제로 돌려 측정했다. 본문의 **실측** 표시가 붙은 수치는 모두 이 환경에서 나온 값이다.

| 항목 | 값 |
|------|-----|
| Python | 3.12.16 |
| playwright (Python) | 1.65.2 |
| 브라우저 | Chromium 159.0.8154.6 (headless shell) |
| 대상 | `127.0.0.1` 로컬 HTTP 서버 (응답 지연을 0~1000ms로 통제) |
| 측정 스크립트 | 본문 부록의 재현 코드(로컬 하네스) |

---

## 1. Flaky는 어디서 오는가 — 세 개의 시간축

### 1.1 하이드레이션: 보이지만 눌리지 않는 요소

React·Vue·Svelte로 만든 페이지는 대개 서버에서 HTML을 먼저 보낸다(SSR). 브라우저는 그 HTML을 즉시 그리지만, **이벤트 핸들러는 JS 번들이 실행된 뒤에 붙는다**. 이 구간에서 `click()`을 보내면 이렇게 된다.

```text
t=0    HTML 도착, 버튼이 화면에 보인다        ← locator는 이미 찾을 수 있다
t=80   JS 번들 다운로드 시작
t=350  hydrate 완료, onclick 부착            ← 이 순간부터 클릭이 "먹는다"
```

`locator.click()`은 DOM에 요소가 있으면 그 사이에도 성공한다. 클릭 이벤트는 발생했지만 리스너가 없으니 아무 일도 일어나지 않고, 다음 줄의 `expect(...)`가 엉뚱한 화면을 보게 된다. **테스트는 "클릭했다"고 믿지만 애플리케이션은 아무것도 하지 않은 상태** — 가장 찾기 어려운 Flaky가 여기서 나온다.

### 1.2 애니메이션과 레이아웃 이동

드롭다운이 열리면서 300ms 동안 슬라이드하고, 그 사이에 다음 버튼이 아래로 밀린다. 고정 좌표로 클릭을 보내면 애니메이션 중간 좌표를 때리거나, 다른 요소가 그 지점을 덮고 있을 수 있다. `wait_for_timeout(300)`으로 "충분히 기다렸다"고 믿는 코드가 기기 성능에 따라 300ms에 끝나기도 하고 700ms에 끝나기도 하는 이유다.

### 1.3 네트워크 지터

같은 API라도 응답 시간은 매번 다르다. 이 글의 실측에서 사용한 30회 실행의 응답 지연 분포는 **최소 26ms, 최대 889ms, 평균 446ms**였다. 서버는 항상 같은 JSON을 주지만, 화면이 완성되는 시점은 매번 다르다.

```text
실행 #1  ├──26ms──┤완료
실행 #2  ├──────────────560ms──────────────┤완료
실행 #3  ├─────────────────────790ms─────────────────────┤완료
              ↑
        sleep(300) 후 검증  → #1 통과, #2 실패, #3 실패
```

### 1.4 그래서 시간이 아니라 상태를 기다려야 한다

`sleep(300)`은 "300ms가 지났다"는 조건이고, 테스트가 알아야 하는 것은 "**목록이 3개 그려졌다**"는 상태다. 이 둘은 같지 않다. 고정 대기와 조건 대기의 차이를 실측으로 비교해 보자.

```python
# 안티패턴: 고정 대기 + 즉시 판정 (is_visible/count 는 재시도하지 않는다)
def naive(page):
    page.goto(URL)
    page.wait_for_timeout(300)
    return page.locator("#items").is_visible() and page.locator("#items li").count() == 3

# web-first: 조건이 만족될 때까지 자동 재시도
def webfirst(page):
    page.goto(URL)
    expect(page.locator("#items")).to_be_visible()
    expect(page.locator("#items li")).to_have_count(3)
    return True
```

같은 30회, 같은 지연 시퀀스로 돌린 결과는 다음과 같다.

| 방식 | 실패 | 평균 소요 | 최소 | 최대 |
|------|------|-----------|------|------|
| `wait_for_timeout(300)` + `is_visible()`/`count()` | **19 / 30 (63%)** | 341ms | 329ms | 413ms |
| `expect(locator).to_be_visible()` / `to_have_count()` | **0 / 30** | 645ms | 47ms | 1,322ms |

읽을 점이 세 가지다.

1. 고정 300ms는 "지연이 300ms 이하일 때만" 통과한다. 지연이 균등분포 0~900ms라면 이론적으로 2/3가 실패하고, 실측도 19/30으로 거의 일치한다.
2. web-first는 **빠른 실행에서는 고정 대기보다 빠르다**(47ms). 지연이 없으면 기다리지 않기 때문이다. "고정 대기가 안전하다"는 통념과 반대다.
3. web-first의 최대 1,322ms는 실패가 아니라 "그만큼 느린 응답을 정확히 기다린 결과"다. **느린 것은 비용이고, 틀린 것은 결함이다.**

---

## 2. Playwright는 행동 전에 무엇을 확인하는가 (Actionability)

### 2.1 조건 목록

공식 문서의 Auto-waiting 페이지는 액션마다 적용되는 **actionability check** 목록을 정의한다. `locator.click()`을 예로 들면 문서는 다음 여섯 가지를 확인한다고 설명한다.

| # | 조건 | 문서의 정의 | 실무에서 걸리는 전형 |
|---|------|-------------|----------------------|
| 1 | **Attached / resolves to one** | locator가 정확히 하나의 요소로 해석됨 | 목록이 아직 비어 있거나, 같은 셀렉터가 2개 매칭 |
| 2 | **Visible** | 비어 있지 않은 bounding box가 있고 `visibility:hidden`이 아님 | `display:none`, 크기 0. **`opacity:0`은 visible로 본다** |
| 3 | **Stable** | 같은 bounding box를 **연속 2 프레임** 유지 | 슬라이드·페이드 애니메이션, 레이아웃 시프트 |
| 4 | **Receives Events** | 액션 지점의 hit target이 그 요소 | 로딩 오버레이, 토스트, sticky 헤더가 덮음 |
| 5 | **Enabled** | disabled가 아님 (`[disabled]`, `[aria-disabled=true]` 하위 포함) | 서버 검증 후 활성화되는 제출 버튼 |
| 6 | **Not Inert** | `inert` 서브트리에 속하지 않음 | 모달 열린 상태의 배경 콘텐츠 |

`fill()`·`select_option()`처럼 값을 넣는 액션은 여기에 **Editable**(enabled이고 readonly 아님)이 추가되고, 반대로 `hover()`·`drag_to()`는 Enabled/Not Inert를 확인하지 않는다. 문서의 액션별 표를 요약하면 다음과 같다.

| 액션 | Visible | Stable | Receives Events | Enabled | Editable | Not Inert |
|------|:---:|:---:|:---:|:---:|:---:|:---:|
| `click()` / `check()` / `tap()` / `dblclick()` | ✓ | ✓ | ✓ | ✓ | – | ✓ |
| `hover()` / `drag_to()` | ✓ | ✓ | ✓ | – | – | – |
| `fill()` / `clear()` | ✓ | – | – | ✓ | ✓ | ✓ |
| `select_option()` | ✓ | – | – | ✓ | – | ✓ |
| `press()` / `set_input_files()` / `focus()` | – | – | – | – | – | – |
| `screenshot()` | ✓ | ✓ | – | – | – | – |

조건이 timeout 안에 만족되지 않으면 액션은 `TimeoutError`로 실패한다. **핵심은 실패했을 때 어느 조건에서 막혔는지 로그가 말해 준다**는 점이다.

### 2.2 실측: disabled 버튼을 누르면 무슨 일이 일어나는가

영구히 `disabled`인 버튼에 `click(timeout=1500)`을 보내고 로그를 받았다.

```text
Locator.click: Timeout 1500ms exceeded.
Call log:
  - waiting for locator("#b")
    - locator resolved to <button id="b" disabled>Pay</button>
  - attempting click action
    2 × waiting for element to be visible, enabled and stable
      - element is not enabled
    - retrying click action
    - waiting 20ms
    2 × waiting for element to be visible, enabled and stable
      - element is not enabled
    - retrying click action
      - waiting 100ms
    3 × waiting for element to be visible, enabled and stable
      - element is not enabled
    - retrying click action
      - waiting 500ms
```

두 가지가 읽힌다.

- **재시도 간격이 20ms → 100ms → 500ms로 늘어난다.** 초반에는 촘촘하게 보다가, 상태가 잘 바뀌지 않는 상황에서는 간격을 벌려 CPU를 낭비하지 않는다. 그래서 "1초짜리 애니메이션이 끝나기를 기다리는" 비용은 20ms 단위가 아니라 최대 500ms 단위로 발생한다.
- 실패 로그가 **어느 조건에서 막혔는지**를 그대로 알려 준다(`element is not enabled`). 타임아웃을 늘리기 전에 이 줄을 읽어야 한다. `element is not enabled`는 앱 버그이거나 테스트 순서 문제이지, 기다림이 부족한 문제가 아니다.

### 2.3 실측: 즉시 판정 API는 기다리지 않는다

`is_visible()`, `is_enabled()`, `count()`는 **질문에 즉시 답한다**. 기다리지 않는다. 같은 순간에 네 API를 호출한 결과다(응답 지연 600ms 설정, `goto` 직후).

| 호출 | 결과 | 소요 |
|------|------|------|
| `locator("#items").is_visible()` | `False` | 37.9ms (4개 호출 합계) |
| `locator("#save").is_enabled()` | `False` | 〃 |
| `locator("#items li").count()` | `0` | 〃 |
| `locator("#save").click()` (같은 상태에서) | 성공 | **1,354ms** |

즉 "즉시 판정 + assert"는 상태 변화를 기다리지 않으므로 Flaky를 만든다. 반대로 `click()`은 자체적으로 조건이 만족될 때까지 기다렸다가(여기서는 disabled 해제 + 오버레이 제거) 눌렀다.

### 2.4 `force=True`가 끄는 것은 정확히 무엇인가

문서는 `force`가 "non-essential actionability check"를 끈다고 설명한다. 실측으로 그 위험을 확인했다. 페이지에는 렌더 직후 **로딩 오버레이가 300ms 동안 화면 전체를 덮고**, 그 위에 `#save` 버튼이 있다.

| 방식 | 결과 |
|------|------|
| `click()` (기본) | 오버레이가 사라질 때까지 기다린 뒤 클릭 → 카운터 **1** |
| `click(force=True)` | 오버레이를 무시하고 좌표를 때림 → 클릭 이벤트가 오버레이로 → 카운터 **0** |

`force=True`는 "Receives Events를 확인하지 않는다"는 뜻이고, 그 대가로 **테스트는 성공했는데 사용자는 아무것도 누르지 못한 상태**를 초록불로 통과시킨다. Flaky를 줄이려고 `force`를 넣는 것은 원인을 없애는 대신 관측을 끄는 일이다.

---

## 3. Web-first assertion: 재시도하는 검증

### 3.1 locator를 넘겨야 재시도한다

`expect()`는 Python에서 **Locator / Page / APIResponse**를 받는다. 값을 먼저 꺼내서 넘기면 재시도가 없다.

```python
expect(page.locator("#items li")).to_have_count(3)   # 재시도 O
expect(page.locator("#items li").count()).to_be(3)   # 재시도 X — Python에서는 ValueError
assert page.locator("#items li").count() == 3        # 재시도 X — 일반 assert
```

실측 결과(모두 지연 600ms, `goto` 직후 호출):

| 코드 | 결과 |
|------|------|
| `expect(...count()).to_be(3)` | `ValueError: Unsupported type: <class 'int'>` |
| `assert ...count() == 3` | **실패** (그 순간 0개) |
| `expect(locator).to_have_count(3)` | **통과** (재시도 후 3개 확인) |

이 차이가 Flaky의 절반을 설명한다. "Playwright를 쓰는데도 Flaky하다"는 팀의 코드에는 대개 `count()`, `text_content()`, `is_visible()`로 값을 꺼내서 `assert`하는 줄이 섞여 있다. **꺼내는 순간 재시도가 사라진다.**

### 3.2 재시도는 실제로 몇 번 일어나는가

문서는 "조건이 만족될 때까지 요소를 다시 가져와 반복 검사한다"고만 설명하고, 간격은 공개하지 않는다. 그래서 직접 세어 봤다. `#status`가 계속 `"Ready: 3 items"`인데 `"Done"`을 기대한 테스트(timeout 3000ms)의 실패 로그다.

```text
Locator expected to have text 'Done'
Actual value: Ready: 3 items
Call log:
  - Expect "to_have_text" locator("#status") with timeout 3000ms
  - waiting for locator("#status")
    2 × locator resolved to <div id="status">Loading…</div>
      - unexpected value "Loading…"
    8 × locator resolved to <div id="status">Ready: 3 items</div>
      - unexpected value "Ready: 3 items"
```

3초 동안 **10번 재해석**했다. 첫 2번은 아직 `Loading…`이던 시점, 이후 8번은 값이 바뀌지 않은 상태에서의 재확인이다.

### 3.3 감지 지연: 타임아웃은 "언제까지"이지 "즉시"가 아니다

요소가 나타나는 시점을 50ms 단위로 바꿔 가며, `expect(...).to_be_visible()`이 **나타난 뒤 얼마 만에 알아채는지**를 측정했다(감지 지연 = 측정 소요 − 실제 등장 시점 − 기준 오버헤드).

| 요소 등장 시점 | expect 소요 | 감지 지연 |
|------|------|------|
| 0ms | 18ms | 0ms |
| 100ms | 194ms | 76ms |
| 200ms | 298ms | 80ms |
| 300ms | 301ms | −17ms |
| 500ms | 804ms | 286ms |
| 750ms | 804ms | 36ms |
| 850ms | 1,306ms | 438ms |
| 1,000ms | 1,304ms | 286ms |

값이 계단처럼 뭉치는 것이 보인다(약 300ms, 800ms, 1300ms 지점). 재확인 간격이 일정하지 않고 벌어지기 때문이며, 이 실측에서 **감지 지연은 최대 약 450ms**였다. 실무적 의미는 두 가지다.

- 5초 타임아웃은 "5초 동안 즉시 반응"이 아니라 "5초 안에 최대 0.5초 지연으로 알아챈다"는 뜻이다. 1초 이내 응답이 기대되는 검증에는 충분하고, 밀리초 단위 동기화가 필요한 검증에는 부적합하다.
- 그래서 **타임아웃을 늘리는 것과 감지가 빨라지는 것은 무관**하다. 느린 응답을 기다리는 용도로만 늘려야 한다.

### 3.4 개수 변화·애니메이션도 조건으로 표현된다

| 상황 | 검증 코드 | 실측 |
|------|-----------|------|
| 150ms 간격으로 5개 행이 추가됨 | `expect(locator).to_have_count(5)` | 799ms에 통과(마지막 행은 750ms에 등장) |
| 1초 슬라이드 애니메이션 중인 요소 | `locator.click()` | 1,451ms에 클릭 완료, 핸들러 1회 실행 |

Stable 체크가 "연속 2 프레임 같은 bounding box"를 요구하기 때문에, **애니메이션이 끝날 때까지 클릭이 스스로 기다린다**. 슬라이드 중간을 때리는 문제는 `sleep`이 아니라 이 조건이 막아 준다.

### 3.5 안티패턴 → 대체 표

| 안티패턴 | 왜 문제인가 | 대체 |
|----------|-------------|------|
| `page.wait_for_timeout(1000)` | 환경에 따라 부족하거나 과함. 실패 시 원인 정보 없음 | `expect(locator).to_be_visible()` 등 조건 대기 |
| `assert locator.is_visible()` | 재시도 없음. 순간 상태만 봄 | `expect(locator).to_be_visible()` |
| `assert locator.count() == n` | 재시도 없음 | `expect(locator).to_have_count(n)` |
| `locator.text_content()` 비교 | 재시도 없음 | `expect(locator).to_have_text()` / `to_contain_text()` |
| `time.sleep()` + `requests`로 API 확인 | UI와 API의 시점이 분리됨 | `page.expect_response()`로 응답 자체를 검증 |
| `click(force=True)` | Receives Events를 끄고 오클릭을 통과시킴 | 원인(오버레이)을 제거하거나 오버레이 소멸을 조건으로 대기 |
| 타임아웃 30초 상향 | 느린 실패만 남고 Flaky는 그대로 | 실패 로그에서 막힌 조건 확인 후 원인 수정 |

### 3.6 실무 옵션 세 가지

```python
from playwright.sync_api import expect

# 1) 검증별 타임아웃 (기본 5초)
expect(page.get_by_test_id("status")).to_have_text("Submitted", timeout=10_000)

# 2) 실패 메시지에 맥락 붙이기
expect(page.get_by_text("Name"), "로그인 후 사용자명이 보여야 함").to_be_visible()

# 3) soft assertion: 실패해도 다음 검증을 계속 (pytest-playwright 0.8.0+)
expect.soft(page.get_by_test_id("status")).to_have_text("Success")
expect.soft(page.get_by_test_id("eta")).to_have_text("1 day")
page.get_by_role("link", name="next page").click()
```

전역 기본값은 `conftest.py`에서 `expect.set_options(timeout=10_000)`으로 바꾼다. soft assertion은 "한 화면에서 여러 항목을 한 번에 검증"할 때 유용하다. 하나가 깨졌다고 나머지 검증을 건너뛰면, 다음 실행에서 또 다른 실패를 처음부터 다시 만나게 되기 때문이다.

---

## 4. 비결정적 네트워크를 고정한다: `page.route`

여기까지가 "기다림"의 문제였다면, 남은 절반은 **입력의 비결정성**이다. 같은 테스트가 어떤 날은 재고 3개, 어떤 날은 0개를 보는 이유는 대개 외부 API의 상태나 응답 시간이다.

### 4.1 route는 어디에 끼어드는가

공식 문서는 라우트가 **페이지와 브라우저 네트워크 스택 사이**에 앉는다고 설명한다. 핸들러는 네트워크 스택이 요청을 처리하기 전에 실행되고, 핸들러가 고르는 결말은 세 가지다.

```text
  page  ──fetch──▶ ┌──────────────┐ ──continue_()──▶ 실제 네트워크 ──▶ 서버
                   │ route handler│                    ▲
                   │ (내 파이썬   │ ──fulfill()──▶ 응답을 여기서 완성 (서버 미도달)
                   │  함수)       │
                   └──────────────┘ ──abort()────▶ 네트워크 오류로 실패
```

| 메서드 | 동작 | 쓰는 상황 |
|--------|------|-----------|
| `route.continue_()` | 요청을 그대로 네트워크로 보냄 (헤더/메서드/본문 수정 가능) | 인증 헤더 주입, 요청 가로채 관측 |
| `route.fulfill()` | 응답을 직접 완성. **서버에 도달하지 않는다** | 정상·경계·오류 응답 고정 |
| `route.fetch()` + `fulfill(response=..., json=...)` | 실제 응답을 받아 일부만 바꿔서 반환 | 실제 데이터 기반 + 한 필드만 조작 |
| `route.abort(error_code)` | 네트워크 오류로 실패시킴 | 타임아웃·연결 끊김 재현 |
| `route.fallback()` | 다음 핸들러로 넘김 | 핸들러를 역할별로 쪼갤 때 |

등록 순서도 기억해야 한다. **여러 라우트가 매칭되면 나중에 등록한 것이 먼저 실행**되고, `fallback()`으로 이전 핸들러에 넘긴다. 공통 핸들러(예: 이미지 차단)를 먼저, 구체 핸들러를 나중에 등록하는 이유다.

### 4.2 무엇을 고정할지 고르는 결정 트리

```text
이 테스트가 검증하려는 것이 API 그 자체인가?
├── 예  → 실제 API를 부르고 응답을 검증한다 (expect_response)
└── 아니오 → 화면 동작이 목적이다
        ├── 응답 내용이 매번 같아도 되는가?
        │     ├── 예  → route.fulfill(json=...) 로 완전 고정
        │     └── 아니오 → 실제 응답을 받아 필요한 필드만 바꾼다 (route.fetch)
        └── 실패 경로를 봐야 하는가?
              ├── 서버 오류   → fulfill(status=500)
              ├── 네트워크 끊김 → abort("connectionreset")
              └── 느린 응답   → 핸들러에서 sleep 후 continue_
```

### 4.3 실전 코드

```python
import json
import time

from playwright.sync_api import expect

# (1) 응답 완전 고정 — 서버는 호출되지 않는다
page.route("**/api/items", lambda route: route.fulfill(json=[{"name": "mock-a"}, {"name": "mock-b"}]))
page.goto("https://shop.example.com/list")
expect(page.locator("#items li")).to_have_count(2)

# (2) 오류 응답 재현
page.route("**/api/items", lambda route: route.fulfill(
    status=500, content_type="application/json", body=json.dumps({"error": "boom"})))
expect(page.get_by_role("alert")).to_contain_text("다시 시도")

# (3) 네트워크 자체를 끊는다
page.route("**/api/items", lambda route: route.abort("connectionreset"))
expect(page.get_by_text("네트워크 오류")).to_be_visible()

# (4) 실제 응답을 받아 한 필드만 바꾼다 — 실데이터 + 경계값
def patch(route):
    resp = route.fetch()
    data = resp.json()
    data.append({"name": "patched", "price": 0})       # 무료 상품 경계값 추가
    route.fulfill(response=resp, json=data)

page.route("**/api/items", patch)

# (5) 느린 응답을 인위적으로 만든다 — 로딩 UI 검증
def slow(route):
    time.sleep(0.8)
    route.continue_()

page.route("**/api/items", slow)
```

`page.route`는 그 페이지가 여는 팝업에도 적용되고, `browser_context.route()`로 등록하면 컨텍스트 전체(모든 페이지)에 적용된다. 반면 `Service Worker`가 네트워크를 가로채는 환경(MSW 같은 도구)에서는 라우트가 요청을 보지 못할 수 있다. 이때는 `browser.new_context(service_workers="block")`으로 Service Worker를 막고 라우트를 쓴다.

### 4.4 실측: 서버에 몇 번 도달했는가

각 패턴이 "실제 서버 호출"을 실제로 막는지 세어 봤다(서버 측 카운터).

| 패턴 | 서버 도달 | 화면 결과 | 소요 |
|------|-----------|-----------|------|
| `fulfill(json=[2건])` | **0회** | `Ready: 2 items`, 2건 | 86ms |
| `abort("connectionreset")` | 0회 | `Network error` 표시 | – |
| `route.fetch()` + `json` 패치 | **1회** | 3건 + `patched` = 4건 | – |
| `fulfill(body=str(list))` (모킹 실수) | 0회 | **`Parse error`** | – |

마지막 줄이 실무에서 자주 나오는 함정이다. `body`에 Python 객체를 그대로 `str()`해서 넣으면 `[{'name': 'broken'}]` 같은 **작은따옴표 JSON**(=JSON 아님)이 전송되고, 앱의 `response.json()`이 파싱에 실패한다. 테스트는 "API가 깨졌다"고 보고하지만 실제로 깨진 것은 모킹 코드다.

```python
route.fulfill(body=str(payload))                  # ✗ 파이썬 repr가 그대로 나간다
route.fulfill(json=payload)                       # ✓ 자동으로 JSON 직렬화 + content-type
route.fulfill(body=json.dumps(payload), content_type="application/json")  # ✓
```

### 4.5 요청 자체를 검증한다: `expect_response`

화면 확인과 별개로 "무엇을 보냈는가"를 검증해야 할 때가 있다. 이때 `expect_response`로 **응답을 기다리며 캡처**한다. 클릭이 응답을 촉발하므로 반드시 클릭 **전에** 등록한다.

```python
with page.expect_response("**/api/items") as info:
    page.get_by_role("button", name="불러오기").click()

resp = info.value
assert resp.status == 200
assert len(resp.json()) == 3
```

실측(지연 400ms): 응답 상태 200, 3건, 캡처까지 422ms. 패턴 매칭은 glob(`*`, `**`, `{png,jpg}`)과 정규식, predicate 함수를 모두 지원한다.

> Python API에는 `page.wait_for_response`가 없다(실측: `AttributeError`). 같은 역할이 필요하면 `page.expect_response()`를 쓰고, 이미 지나간 이벤트를 잡아야 하는 특수한 경우에만 `page.wait_for_event("response", predicate=...)`를 쓴다. 후자는 등록 시점 이후의 이벤트만 잡으므로 순서에 민감하다(실측: `goto` 직후 등록해 258ms에 캡처).

### 4.6 HAR로 "실제 트래픽"을 통째로 재생한다

응답이 여러 개 얽힌 화면(대시보드, 결제 플로우)은 하나씩 `fulfill`로 쓰는 것이 비효율적이다. 이때 HAR 파일을 기록해 재생한다.

```python
# 1) 기록: update=True면 실제 요청을 보내고 파일을 만든다 (컨텍스트를 닫을 때 기록됨)
ctx.route_from_har("hars/list.har", url="**/api/items", update=True)

# 2) 재생: update를 빼면 파일에서 응답을 서빙한다. 매칭되는 기록이 없으면 abort.
ctx.route_from_har("hars/list.har", url="**/api/items")
```

실측 결과: 기록 시 서버 도달 1회, 1.7KB HAR 생성. 재생 시 **서버 도달 0회**로 같은 화면(`Ready: 3 items`)이 그려졌다. HAR 재생은 URL과 메서드를 엄격히 매칭하고, POST는 본문까지 매칭한다는 점을 문서가 명시한다. 그래서 "무엇이든 응답"하는 `fulfill`보다 엄격하고, 대신 기록 시점의 데이터가 파일에 남는다(테스트 픽스처로 커밋하고 리뷰해야 하는 이유).

### 4.7 Python과 TypeScript API 대응

| 개념 | Python | TypeScript |
|------|--------|------------|
| 라우트 등록 | `page.route(pattern, handler)` | `page.route(pattern, handler)` |
| 응답 완성 | `route.fulfill(json=...)` | `route.fulfill({ json })` |
| 요청 통과 | `route.continue_(headers=...)` | `route.continue({ headers })` |
| 실패 | `route.abort("connectionreset")` | `route.abort("connectionreset")` |
| 검증 | `expect(locator).to_be_visible()` | `expect(locator).toBeVisible()` |
| 응답 대기 | `page.expect_response(...)` | `page.waitForResponse(...)` / `expect_response` |
| HAR | `page.route_from_har(...)` | `page.routeFromHAR(...)` |

이름만 다를 뿐 의미는 같다. 그래서 이 글의 처방은 언어와 무관하게 적용된다.

---

## 5. 실패했을 때 무엇을 보는가: Trace Viewer

### 5.1 켜는 법

```python
context.tracing.start(screenshots=True, snapshots=True, sources=True)
# ... 테스트 본문 ...
context.tracing.stop(path="trace.zip")
```

pytest에서는 플래그로 켠다.

```bash
pytest --tracing retain-on-failure --output=test-results
```

`retain-on-failure`는 실패한 테스트만 남기므로 CI 아티팩트 크기 부담이 적다. 확인은 `playwright show-trace trace.zip`(GUI) 또는 `playwright trace open trace.zip` 후 CLI로 조회한다.

### 5.2 실측: trace에는 무엇이 들어 있는가

간단한 시나리오(이동 → 개수 검증 → 클릭)의 trace는 **15KB, 10개 항목**이었다.

```text
trace.trace
trace.network
screencast/page@...-1795822895495.jpeg   ← 프레임별 화면 (4장)
screencast/page@...-1795822895528.jpeg
screencast/page@...-1795822895696.jpeg
screencast/page@...-1795822895995.jpeg
resources/...                            ← DOM 스냅샷 리소스 (2개)
```

CLI로 바로 요약을 볼 수 있다.

```text
$ playwright trace actions
     # Time       Action                                   Duration
    1. 0:00.010  Create page                              36ms
    2. 0:00.047  Navigate                                  34ms
    3. 0:00.099  Expect "to_have_count"  locator('#items li')  219ms
    4. 0:00.320  Click                   locator('#save')     338ms

$ playwright trace requests
     # Start     Method   Status   Name        Duration   Size
    1. 0:00.053  GET      200      127.0.0.1/      5ms    1.8K
    2. 0:00.079  GET      200      items         212ms     201
```

### 5.3 읽는 순서

1. **실패한 액션의 Duration**: 338ms 걸린 클릭은 "오버레이가 사라지길 기다린" 흔적이다. Duration이 타임아웃에 가깝다면 어떤 조건이 막혔는지 로그에서 확인한다.
2. **직전 네트워크의 Duration과 Status**: 212ms짜리 요청 뒤에 검증이 실패했다면, 검증 로직보다 데이터가 늦게 온 문제일 수 있다. `fulfill`로 그 요청을 고정하면 재현이 사라지는지 본다.
3. **스냅샷과 콘솔**: 실패 시점의 DOM이 남아 있으므로, "요소가 없었다"가 "요소가 다른 상태였다"(예: `aria-disabled=true`)인지 구분된다.

Trace가 없으면 Flaky는 "다시 돌려 보니 통과"로 끝난다. Trace가 있으면 **몇 번 실행 중 어떤 상태에서 왜 기다렸는지**가 남는다.

---

## 6. CI 체크리스트

### 6.1 Flaky를 만났을 때의 절차

1. **재현 빈도를 측정한다**: 같은 테스트를 20~30회 반복(pytest-repeat의 `--count` 또는 셸 루프)해서 실패율을 기록한다. "가끔"은 지표가 아니다.
2. **원인을 분류한다**: (a) 상태 미대기 (b) 셀렉터 불안정 (c) 데이터 의존 (d) 외부 API 지터 (e) 실제 제품 버그.
3. **`retries`는 진단 도구로만 쓴다**: 재시도로 초록불이 된 테스트는 목록에 남기고, 원인 분류 후 고친다. 재시도를 켜 두고 잊으면 제품 버그가 재시도 뒤에 숨는다.
4. **고칠 때는 조건을 명시한다**: `sleep` 삭제 → `expect(locator)`로 교체 → 외부 호출은 `route`/HAR로 고정.
5. **고친 뒤 다시 20~30회** 돌려 실패율이 0인지 확인한다. 1회 통과는 증거가 아니다.

### 6.2 릴리스 전 체크리스트

- [ ] 테스트 코드에 `wait_for_timeout`, `time.sleep`이 남아 있지 않은가
- [ ] `is_visible()`, `count()`, `text_content()`로 값을 꺼내 `assert`하는 줄이 없는가
- [ ] 외부 API·CDN 의존 테스트가 `route`/HAR로 고정되어 있는가
- [ ] 오류·빈 목록·느린 응답 경로를 `fulfill`/`abort`로 각각 검증하는가
- [ ] 모킹 응답이 `json=` 또는 `json.dumps`로 만들어지는가 (repr 문자열이 아닌가)
- [ ] Service Worker를 쓰는 앱에서 네트워크 이벤트가 보이는가(`service_workers="block"` 필요 여부)
- [ ] CI에서 `--tracing retain-on-failure`로 아티팩트를 남기는가
- [ ] Flaky 목록과 실패율이 이슈로 관리되고 있는가(재시도로만 덮여 있지 않은가)

---

## 7. 정리

- Flaky는 세 시간축(하이드레이션·애니메이션·네트워크 지터)에서 생긴다. 공통 원인은 **시간을 기다리는 코드**다.
- Playwright는 액션 전에 **Attached·Visible·Stable·Receives Events·Enabled·Not Inert**를 확인하고, 실패하면 어느 조건에서 막혔는지 로그로 알려 준다. 실측에서 재시도 간격은 20ms → 100ms → 500ms로 늘어났다.
- `is_visible()`·`count()`는 기다리지 않는다(실측 38ms에 `False`/`0`). `expect(locator)`에 **locator를 넘겨야** 재시도가 살아난다(실측 3초에 10회 재확인).
- web-first assertion의 감지 지연은 최대 약 450ms였다. 타임아웃은 "느린 응답을 얼마나 허용할지"이지 감지를 빠르게 하는 값이 아니다.
- 고정 300ms 대기는 실측에서 **63% 실패**, `expect(locator)`는 **0% 실패**였고 빠른 실행에서는 더 빨랐다(47ms).
- `page.route`의 `fulfill`은 서버 도달 0회, `fetch+패치`는 1회, HAR 재생은 0회로 같은 화면을 재현했다. `body=str(...)` 같은 모킹 실수는 앱 오류로 위장된다(`Parse error`).
- 실패를 만났을 때 볼 것은 타임아웃 숫자가 아니라 **Trace의 액션 Duration과 직전 네트워크**다.

Flaky 테스트는 "불안정한 테스트"가 아니라 **불안정한 가정을 드러낸 테스트**다. 무엇을 기다리는지, 무엇을 고정해야 하는지 코드에 적어 두면, 같은 테스트가 100번 돌아도 같은 답을 준다.

---

## 부록: 재현 코드

본문의 실측에 사용한 최소 하네스다. 응답 지연을 통제하는 서버와 두 가지 검증 방식을 같은 시퀀스로 돌린다.

```python
# lab_server.py — /api/items가 DELAY_MS 후 JSON을 돌려주는 서버 (요청 수 카운터 포함)
import json, threading, time
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

PAGE = """<!doctype html><meta charset="utf-8">
<div id="status">Loading…</div><div id="error" hidden>API error</div>
<ul id="items"></ul><button id="save" disabled>Save</button><span id="clicks">0</span>
<script>
  document.getElementById('save').onclick = () => {
    const el = document.getElementById('clicks');
    el.textContent = String(Number(el.textContent) + 1);
  };
  (async () => {
    let res;
    try { res = await fetch('/api/items'); }
    catch (e) { document.getElementById('status').textContent = 'Network error';
                document.getElementById('error').hidden = false; return; }
    if (!res.ok) { document.getElementById('status').textContent = 'Error';
                   document.getElementById('error').hidden = false; return; }
    let items;
    try { items = await res.json(); }
    catch (e) { document.getElementById('status').textContent = 'Parse error';
                document.getElementById('error').hidden = false; return; }
    const ul = document.getElementById('items');
    ul.innerHTML = items.map(i => '<li>' + i.name + '</li>').join('');
    document.getElementById('status').textContent = 'Ready: ' + items.length + ' items';
    document.getElementById('save').disabled = false;
  })();
</script>"""

class State:
    delay_ms = 100
    hits = 0

def serve(port=0):
    class H(BaseHTTPRequestHandler):
        def log_message(self, *a): pass
        def do_GET(self):
            if self.path.startswith('/api/items'):
                State.hits += 1
                time.sleep(State.delay_ms / 1000)
                body = json.dumps([{"name": "alpha"}, {"name": "beta"},
                                   {"name": "gamma"}]).encode()
                self.send_response(200)
                self.send_header('Content-Type', 'application/json')
                self.send_header('Content-Length', str(len(body)))
                self.end_headers()
                self.wfile.write(body)
                return
            body = PAGE.encode()
            self.send_response(200)
            self.send_header('Content-Type', 'text/html; charset=utf-8')
            self.send_header('Content-Length', str(len(body)))
            self.end_headers()
            self.wfile.write(body)

    srv = ThreadingHTTPServer(('127.0.0.1', port), H)
    threading.Thread(target=srv.serve_forever, daemon=True).start()
    return srv
```

```python
# compare.py — 고정 대기 vs web-first, 같은 지연 시퀀스로 30회
import random, time
from playwright.sync_api import sync_playwright, expect
from lab_server import State, serve

srv = serve()
URL = f"http://127.0.0.1:{srv.server_address[1]}/"
rnd = random.Random(20261128)
delays = [int(rnd.uniform(0, 900)) for _ in range(30)]   # 두 방식에 같은 순서로 적용

def naive(page):
    page.goto(URL); page.wait_for_timeout(300)
    return page.locator("#items").is_visible() and page.locator("#items li").count() == 3

def webfirst(page):
    page.goto(URL)
    expect(page.locator("#items")).to_be_visible()
    expect(page.locator("#items li")).to_have_count(3)
    return True

with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page()
    for name, fn in (("naive", naive), ("webfirst", webfirst)):
        fails, times = 0, []
        for d in delays:
            State.delay_ms = d
            t0 = time.perf_counter()
            try: ok = fn(page)
            except Exception: ok = False
            times.append((time.perf_counter() - t0) * 1000)
            fails += 0 if ok else 1
        print(name, "fails", fails, "mean", round(sum(times) / len(times)))
    browser.close()
```

`random` 시드를 고정하지 않으면 두 방식이 서로 다른 지연 시퀀스를 만나 비교가 흔들린다. 위 코드처럼 **한 번 만든 지연 리스트를 두 방식에 같은 순서로 적용**하는 것이 공정한 비교의 조건이다. 이 하네스를 그대로 돌리면 `naive fails 19 / webfirst fails 0`이 나온다(본문 1.4절과 같은 값).

---

## 참고 자료

- Playwright 공식 문서, *Auto-waiting* — 액션별 actionability check 표, Visible·Stable·Receives Events·Enabled·Editable·Not Inert 정의: <https://playwright.dev/python/docs/actionability>
- Playwright 공식 문서, *Assertions* — auto-retrying assertion 목록, 기본 5초 타임아웃, soft assertion, 커스텀 메시지: <https://playwright.dev/python/docs/test-assertions>
- Playwright 공식 문서, *Network* — 라우트가 네트워크 스택에 끼어드는 지점, 금지 헤더, 리다이렉트 처리, glob 패턴, Service Worker 주의점: <https://playwright.dev/python/docs/network>
- Playwright 공식 문서, *Route* API — `continue_`·`fallback`·`fetch`·`fulfill` 옵션과 등록 순서: <https://playwright.dev/python/docs/api/class-route>
- Playwright 공식 문서, *Mock APIs* — `fulfill`/`fetch` 패턴과 HAR 기록·재생(`route_from_har`, `update`): <https://playwright.dev/python/docs/mock>
- MDN, *Forbidden request header* — `Cookie`·`Host`·`Content-Length` 등 라우트에서 덮어쓸 수 없는 헤더: <https://developer.mozilla.org/en-US/docs/Glossary/Forbidden_request_header>
- MDN, *inert* 전역 속성 — Not Inert 조건의 근거: <https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Global_attributes/inert>
- pytest-playwright — `--tracing`, `--output`, `--screenshot`, `--video` 플래그: <https://pypi.org/project/pytest-playwright/>
