# 순수 HTML/CSS/JS 웹 슬라이드 구현 리서치 (통합본)

> 2026-10-12 · 서브에이전트 4개 병렬 리서치 결과를 통합·조정한 문서
> 참고: Reveal.js, Shower, Slidev 소스/문서 (최종 구현은 라이브러리 없이 바닐라)

---

## 0. 최종 권장 아키텍처 한눈에 보기

| 영역 | 결정 | 근거 |
|---|---|---|
| 캔버스 | **1920×1080 고정 논리 캔버스** + JS `transform: scale()` | 모든 화면·썸네일·발표자 미리보기에서 렌더링 동일 (Reveal.js/Shower 방식) |
| 슬라이드 배치 | 모든 슬라이드를 **한 자리에 겹쳐 쌓기** (Grid `grid-area:1/1` 또는 `absolute; inset:0`) | fade/zoom 등 두 장 겹치는 전환에 필수 |
| 비활성 슬라이드 | `visibility:hidden` + `inert` (`display:none` 금지) | 전환 가능 + 포커스/스크린리더 차단 |
| 상태 모델 | **`{cur, step}` 두 숫자만** | 네비게이션·렌더·해시·창 동기화를 모두 이 값으로 처리 |
| 상태 변경 | **`goTo()` 한 곳에서만** — 모든 입력(키·스와이프·클릭·휠·해시·원격창)이 `next/prev/goTo`로 수렴 | 루프·불일치 방지, 확장은 `onChange` 훅 |
| 슬라이드 전환 | **Web Animations API** (`el.animate` + `finished` Promise) | 취소/즉시완료 가능, `transitionend` 함정 없음 |
| 빌드 스텝 효과 | **CSS 클래스 토글** (`.is-visible`, `.is-current-step`) + CSS transition | 단순, 잠금 불필요 |
| 입력 잠금 | `isAnimating` + **1칸 버퍼**(마지막 입력만 보관) | 빠른 연타에도 반응성 유지 |
| URL | **1-based `#3.2`** (슬라이드 3, 스텝 2), 기본 `replaceState` | 새로고침 유지, 뒤로가기 기록 오염 없음 |
| 발표자 보기 | 같은 HTML을 `?presenter`로 열고 **BroadcastChannel** 양방향 동기화 | 코드 단순, same-origin |
| 애니메이션 속성 | `transform`, `opacity`만 | compositor 전용 → GPU 가속 |

### 에이전트 간 충돌 조정 내역

| 충돌 | 결정 |
|---|---|
| 캔버스 1920×1080(Agent 1) vs 1280×720(Agent 4) | **1920×1080** |
| 슬라이드 전환: CSS transition(Agent 1·4 예제) vs WAAPI(Agent 3) | 슬라이드 전환은 **WAAPI**, 슬라이드 *내부* 스텝은 CSS transition |
| `n`/`p` = 다음/이전(Agent 2) vs `N` = 노트, `P` = 발표자창(Agent 4) | `n/p` 네비게이션 제거. **`N` 노트 토글, `S` 발표자 보기**(Reveal.js 관례), `B`/`.` 블랙아웃 |
| 클릭 존 왼쪽 50%(Agent 3) vs 30%(Agent 2) | **왼쪽 30% = 이전**, 나머지 = 다음 |
| 스텝 마크업 `[data-step]`(Agent 2) vs `.step` + 선택적 `data-step="n"`(Agent 4) | **`.step` 클래스 + 선택적 `data-step` 그룹 번호** |

---

## 1. 레이아웃 & DOM 구조

### 1.1 배치 방식 비교

| 방식 | 겹치기 전환 | 비고 |
|---|---|---|
| Absolute 스택 `position:absolute; inset:0` | ✅ | 가장 확실. Reveal.js 방식 |
| Grid 단일 셀 `grid-area:1/1` | ✅ | normal flow 유지, `place-items`로 정렬 쉬움 |
| Flex 가로 스트립 + `translateX` | ❌ | slide 전환만 가능, fade/zoom 불가 |
| `display:none` 토글 | ❌ | 애니메이션 불가 |

### 1.2 HTML 구조 (W3C APG Carousel 패턴)

```html
<div class="stage">                         <!-- 100dvh, 레터박스 배경 -->
  <main class="deck" aria-roledescription="carousel" aria-label="발표 제목">
    <section class="slide" role="group" aria-roledescription="slide"
             aria-label="1 / 12" data-transition="fade">
      <h1>제목</h1>
      <aside class="notes">발표자 노트</aside>
    </section>
    ...
  </main>
  <div class="progress" aria-hidden="true"></div>
  <div class="counter" aria-live="polite" aria-atomic="true"></div>
</div>
```

- `<section>`에 이름이 있으면 암묵적 `region` 랜드마크가 되므로 `role="group"`을 명시해 랜드마크 남발 방지
- 비활성 슬라이드: `inert` 필수. `aria-hidden`만 쓰면 내부 링크에 Tab 이동 가능 → 접근성 위반
- 설정은 `data-*` (`data-transition`, `data-layout`, `data-step` …)

### 1.3 반응형 & 16:9 고정

**권장 (JS 스케일)**

```css
.stage { position: fixed; inset: 0; height: 100vh; height: 100dvh; overflow: hidden; background: #000; }
.deck  { position: absolute; left: 50%; top: 50%; width: 1920px; height: 1080px;
         transform: translate(-50%, -50%) scale(var(--scale, 1));
         display: grid; overflow: hidden; }
.slide { grid-area: 1 / 1; visibility: hidden; contain: layout paint style; }
.slide.is-active { visibility: visible; }
```

```js
new ResizeObserver(([e]) => {
  const { width, height } = e.contentRect;
  deck.style.setProperty('--scale', Math.min(width / 1920, height / 1080));
}).observe(stage);
```

- `dvh`: 모바일 주소창 변화에 맞춰 실제 보이는 높이. `100vh` 선언 후 `100dvh`로 덮어써 fallback
- 주의: transform은 레이아웃 크기를 바꾸지 않음 → 마우스 좌표 계산 시 scale로 나눠야 함

**대안 (JS 없는 순수 CSS)** — 모든 치수를 `cqi` 단위로 써야 하는 부담이 있음

```css
.stage { height: 100dvh; display: grid; place-items: center; container-type: size; }
.deck  { width: min(100cqw, 100cqh * 16 / 9); aspect-ratio: 16 / 9; container-type: inline-size; }
.slide { --u: calc(100cqi / 1920); font-size: calc(40 * var(--u)); }
```

### 1.4 브라우저 호환성 (caniuse 기준)

| 기능 | Chrome/Edge | Firefox | Safari | 전역 |
|---|---|---|---|---|
| `aspect-ratio` | 88 | 89 | 15 | ~97% |
| `dvh/svh/lvh` | 108 | 101 | 15.4 | ~96% |
| Container query 단위 | 105 | 110 | 16.0 | ~96% |
| `inert` | 102 | 112 | 15.5 | ~96.5% |
| `ResizeObserver` | 64 | 69 | 13.1 | ~97% |
| View Transitions (same-doc) | 111 | 144 | 18 | ~93% |
| BroadcastChannel | Baseline 2022.03 | | | |

---

## 2. 키보드 · 마우스 · 터치 네비게이션

### 2.1 키 매핑 (`KeyboardEvent.key` 사용, `keyCode`는 deprecated)

| 동작 | 키 |
|---|---|
| 다음 | `ArrowRight` `ArrowDown` `PageDown` `Space` `Enter` |
| 이전 | `ArrowLeft` `ArrowUp` `PageUp` `Shift+Space` |
| 처음 / 끝 | `Home` / `End` (끝은 모든 스텝 표시 상태) |
| 노트 토글 / 발표자 보기 | `N` / `S` |
| 블랙아웃 | `B` / `.` (클리커 "화면 끄기" 버튼) |

처리 규칙:
- 무시: `input/textarea/select/[contenteditable]` 포커스, `e.defaultPrevented`, Ctrl/Alt/Meta 조합(브라우저 단축키 보호)
- Shift는 Space에만 허용
- `e.repeat`는 기본 무시 (길게 눌러 10장 넘어가는 사고 방지)
- **실제로 처리한 키에만** `preventDefault()` (Space/PageDown 스크롤 방지)
- 클리커(Logitech R400 등)는 보통 `PageUp/PageDown` 전송 → 필수 지원 *(미검증: 기존 지식)*

### 2.2 URL 해시 동기화

| 쓰기 방법 | history 항목 | `hashchange` 발생 |
|---|---|---|
| `location.hash = …` | 생김 | **발생** → 루프 가드 필요 |
| `history.pushState` | 생김 | 없음 |
| `history.replaceState` (**기본 권장**) | 없음 | 없음 |

- 형식: `#3` / `#3.2` (1-based 슬라이드, 0..N 스텝). 내부 index는 0-based
- 읽기: 초기 로드 시 파싱 + `hashchange`(주소창 입력/링크) + `popstate`(pushState 옵션 시)
- 루프 방지: 같은 상태면 no-op, 같은 hash면 쓰기 생략, 해시에서 온 이동은 `{from:'hash'}`로 재기록 안 함
- Reveal.js는 replaceState를 1000ms 간격으로 debounce

### 2.3 터치 · 클릭 · 휠

- **Pointer Events**(`pointerdown/up/cancel`) 하나로 처리. 스와이프는 `pointerType !== 'mouse'`만
- CSS `touch-action: pan-y pinch-zoom` → `touchmove` preventDefault 불필요, 리스너 passive 가능
- 스와이프 판정: 거리 ≥ 45px **또는** (≥ 20px & 속도 ≥ 0.3px/ms), 800ms 초과 무시, `isPrimary`만, `|dx|>|dy|`면 가로
- 클릭 존: `clientX < innerWidth*0.3` → 이전, 나머지 → 다음. `a, button, input, video, [data-no-nav]` 내부·텍스트 선택 중·스와이프 직후는 무시
- 휠: 기본 **off**. 켜면 debounce 대신 **800ms cooldown lock**, `|deltaY|<10`과 `ctrlKey`(핀치줌) 무시, `{passive:false}`

### 2.4 진행 바 & 카운터

```css
.progress { position: absolute; left: 0; bottom: 0; height: 4px; width: 100%;
            transform-origin: left; transform: scaleX(var(--progress)); transition: transform .3s; }
```
- `width` 대신 `scaleX` (layout/paint 없음)
- 진행 바는 `aria-hidden`, 정보는 카운터 `aria-live="polite"`로 ("3 / 12"). `assertive`는 과함

---

## 3. 슬라이드 전환 애니메이션

### 3.1 방식 비교

| | CSS transition | CSS @keyframes | **WAAPI** | View Transitions |
|---|---|---|---|---|
| 완료 감지 | `transitionend` (속성마다 발생, 값 불변·0ms면 미발생) | `animationend` | **`finished` Promise** | `finished` Promise |
| 취소/스킵 | 클래스 되돌리기 | 클래스 제거 | `cancel/finish/reverse` | `skipTransition()` |
| 방향 인식 | past/future 위치로 자동 | 방향별 키프레임 | 부호만 런타임 주입 | `types` + `:active-view-transition-type()` |
| 단점 | 이벤트 함정, timeout fallback 필수 | 동적 파라미터 어려움 | 효과 정의가 JS로 | 스냅샷 기반(전환 중 상호작용 불가), 세밀 제어 어려움 |

→ **WAAPI 채택**. View Transitions는 progressive enhancement 용도로만.

### 3.2 효과 정의 (dir: +1 앞으로, −1 뒤로)

```js
const EFFECTS = {
  fade:  ()    => [[{opacity:1},{opacity:0}], [{opacity:0},{opacity:1}]],
  slide: (dir) => [[{transform:'translateX(0)'},{transform:`translateX(${-100*dir}%)`}],
                   [{transform:`translateX(${100*dir}%)`},{transform:'translateX(0)'}]],
  zoom:  (dir) => [[{transform:'scale(1)',opacity:1},{transform:`scale(${dir>0?4:0.2})`,opacity:0}],
                   [{transform:`scale(${dir>0?0.2:4})`,opacity:0},{transform:'scale(1)',opacity:1}]],
};
// 효과 선택: 들어오는 슬라이드의 data-transition → deck 기본값 → 'fade'
// 타이밍: 600ms, cubic-bezier(0.26, 0.86, 0.44, 0.985) (Reveal.js 기본 이징)
```

### 3.3 두 장 동시 표시

- 전환 중: 나가는 슬라이드 `.is-leaving`(z 1), 들어오는 슬라이드 `.is-entering`(z 2), 둘 다 visible
- 끝나면 나머지는 `visibility:hidden` + `inert` + `aria-hidden`
- **`fill:'forwards'` → 상태 클래스 확정 → `anim.cancel()`** 순서로 정리 (마지막 프레임 깜빡임 방지, fill 누적 없음)

### 3.4 GPU 가속

- 애니메이션 속성은 `transform`, `opacity`만 (`left/top/width`는 매 프레임 layout, `filter`는 paint 비용 큼)
- `will-change: transform, opacity`는 **전환 중인 두 장에만** 잠깐 → 상시 적용 시 레이어 메모리 폭증
- 각 슬라이드에 `contain: layout paint style`
- 측정(read) 먼저, 쓰기(write) 나중 — layout thrash 방지

### 3.5 입력 잠금 & 접근성

- `isAnimating` + `await Promise.all(anims.map(a => a.finished))`, `cancel()` 시 reject되므로 catch 필수
- 잠금 중 입력: **마지막 1개만 버퍼** → 끝나면 실행. (대안: 진행 중 전환을 `finish()`로 즉시 완료)
- `next/prev`는 버퍼된 목표가 있으면 그 기준으로 계산 (연타 시 누락 방지)
- `prefers-reduced-motion: reduce` → 모든 효과를 150ms fade로 대체, `matchMedia(...).addEventListener('change')`로 런타임 반영

---

## 4. 슬라이드 내 빌드 효과 · 코드 하이라이트 · 화자 노트

### 4.1 스텝 순서 규칙

- `.step`에 `data-step`이 없으면 **자동 번호**(DOM 순서), `data-step="n"`이면 그 그룹에 합류(동시 등장)
- 그룹 번호를 정렬해 `steps[slide] = [[el, el], [el], …]` 2차원 배열 생성
- `next()`: 남은 스텝 있으면 `step++`, 없으면 다음 슬라이드 `step=0`
- `prev()`: `step>0`이면 `step--`, 아니면 이전 슬라이드 **`step=N`(모두 표시된 상태)**로 진입
- 한 슬라이드 안에서는 자동/명시 번호 중 한 방식만 쓰는 것을 권장 (섞으면 순서 예측 어려움)

### 4.2 효과 변형 (CSS 클래스 토글)

```css
.step { opacity: 0; visibility: hidden;
        transition: opacity var(--dur), transform var(--dur), visibility 0s var(--dur); }
.step.is-visible { opacity: 1; visibility: visible; transition-delay: 0s; }
.step.fade-up { transform: translateY(24px); }
.step.fade-up.is-visible { transform: none; }
.step.fade-out-after.is-visible:not(.is-current-step) { opacity: .3; }
/* 제자리 효과: 처음부터 보이고, 스텝 도달 시 장식만 추가 */
.step.highlight, .step.strike { opacity: 1; visibility: visible; }
.step.highlight { background: linear-gradient(#fde68a,#fde68a) no-repeat 0 90% / 0% 40%;
                  transition: background-size var(--dur); }
.step.highlight.is-visible { background-size: 100% 40%; }
.step.strike.is-visible { text-decoration: line-through 3px #e11d48; color: #888; }
@media (prefers-reduced-motion: reduce) { :root { --dur: 0s; } }
```

- `display:none` 대신 `visibility:hidden` → 공간 유지(레이아웃 안 튐) + 접근성 트리에서도 제외(화면과 일치)
- `visibility 0s var(--dur)` 지연: 숨길 땐 fade 끝난 뒤 hidden, 보일 땐 즉시 visible

### 4.3 코드 라인 하이라이트 (라이브러리 없이)

마크업: `<pre class="code" data-lines="|1-3|5|7-9" data-ln-start="10"><code>…</code></pre>`

1. 로드 시 `textContent`를 줄 단위로 나눠 `<span class="line">`으로 감쌈 (escape 필수, 작성자는 `<`를 `&lt;`로)
2. `data-lines`를 `|`로 분리 → 각 단계를 `Set`으로 파싱 (`"1-3,5"` → {1,2,3,5}). 첫 단계 빈 문자열 = 하이라이트 없음
3. 2단계 이후 각각을 **일반 스텝 그룹에 등록** → 네비게이션이 자연스럽게 소비
4. 현재 도달 단계의 줄에 `.is-hl`, 블록에 `.has-highlights`

```css
pre.code code { display: block; counter-reset: line; }
.line { display: block; counter-increment: line; white-space: pre; transition: opacity var(--dur); }
.line::before { content: counter(line); display: inline-block; width: 3em; padding-right: 1em;
                text-align: right; color: #64748b; user-select: none; }
pre.has-highlights .line:not(.is-hl) { opacity: .35; }
```
- 시작 번호: `code.style.counterReset = 'line ' + (start - 1)`
- Reveal.js는 단계마다 `<code>`를 복제하지만, 블록 하나에서 클래스만 토글하는 편이 단순
- 구문 강조가 필요하면 **줄 분할 먼저, 줄마다 토큰화** (여러 줄 주석 span이 줄 경계를 넘지 않게)

### 4.4 화자 노트 & 발표자 보기

- 마크업: 슬라이드 안 `<aside class="notes">`, 기본 `display:none`
- `N`: `body.show-notes` 토글 → 현재 슬라이드 하단 오버레이
- `S`: `window.open(location.pathname + '?presenter' + location.hash, 'deck-presenter', 'popup,width=1200,height=760')`
- 동기화: 두 창 모두 `new BroadcastChannel('deck')`
  - 어느 창에서 이동하든 `{type:'go', cur, step}` 브로드캐스트 (양방향 조작)
  - 수신 측은 `{from:'remote'}`로 적용 → 재전송 안 함 (무한 루프 방지)
  - 발표자 창은 열리자마자 `{type:'hello'}` → 메인이 현재 상태로 응답
- 발표자 화면: 현재/다음 슬라이드 미리보기(`cloneNode(true)` + `scale`, `inert`), 노트, 경과 타이머(클릭 시 리셋), 현재 시각, `3 / 12 · step 1/3`
  - clone은 id 중복·리스너 미복제 → 정적 표시용. 복잡한 슬라이드는 `<iframe src="?preview#3.1">` 대안
- ⚠ BroadcastChannel은 same-origin 전용. `file://`은 브라우저마다 동작이 다름 → 로컬 서버(`npx serve`, `python -m http.server`) 권장. 대안 `window.opener.postMessage`

### 4.5 인쇄 / PDF

```css
@media print {
  @page { size: 1920px 1080px; margin: 0; }
  * { print-color-adjust: exact; -webkit-print-color-adjust: exact; }
  .stage, .deck { position: static; transform: none; height: auto; display: block; }
  .slide { position: relative; width: 1920px; height: 1080px; visibility: visible !important;
           opacity: 1 !important; break-after: page; }
  .step, .line { opacity: 1 !important; visibility: visible !important; transform: none !important; }
  aside.notes, .progress, .counter { display: none !important; }
}
```

---

## 5. 통합 설계: 모듈 간 연결

```
 입력 (keyboard / pointer / click / wheel / hashchange / BroadcastChannel)
        │  next() / prev() / goTo(c, s, {from})
        ▼
 ┌───────────────── goTo ─────────────────┐
 │ clamp → 같으면 no-op → 잠금 중이면 버퍼   │
 │ 1. renderSteps(c, s)   (대상 슬라이드 스텝 먼저 확정) │
 │ 2. 슬라이드가 바뀌면 await transition()  │
 │ 3. setSlideStates(c)   (inert/aria/class) │
 │ 4. progress · counter · aria-live        │
 │ 5. from≠'hash'   → writeHash()            │
 │ 6. from≠'remote' → channel.postMessage()  │
 │ 7. onChange 리스너 호출                    │
 │ 8. 버퍼된 입력 있으면 실행                  │
 └─────────────────────────────────────────┘
```

통합 `goTo` 골격 (각 섹션 코드를 이 흐름에 꽂으면 됨):

```js
const state = { cur: 0, step: 0 };
let animating = false, pending = null;
const target = () => pending ?? state;            // 연타 시 버퍼 기준으로 계산

async function goTo(c, s = 0, { from = 'local' } = {}) {
  c = clamp(c, 0, slides.length - 1);
  s = clamp(s, 0, steps[c].length);
  if (c === state.cur && s === state.step) return;
  if (animating) { pending = { cur: c, step: s, from }; return; }

  const prevCur = state.cur;
  Object.assign(state, { cur: c, step: s });
  renderSteps(c, s);                               // 4.1~4.3
  if (c !== prevCur) {
    animating = true;
    try { await transition(slides[prevCur], slides[c], c > prevCur ? 1 : -1); }  // 3.x
    finally { animating = false; }
  }
  setSlideStates(c);                               // 1.2 (inert, aria-hidden, .is-active)
  updateProgress(c);                               // 2.4
  if (from !== 'hash') writeHash();                // 2.2
  if (from !== 'remote') channel?.postMessage({ type: 'go', ...state });  // 4.4
  listeners.forEach(fn => fn({ ...state, prevCur }));

  if (pending) { const p = pending; pending = null; goTo(p.cur, p.step, { from: p.from }); }
}

function next() {
  const t = target();
  if (t.step < steps[t.cur].length) goTo(t.cur, t.step + 1);
  else if (t.cur < slides.length - 1) goTo(t.cur + 1, 0);
}
function prev() {
  const t = target();
  if (t.step > 0) goTo(t.cur, t.step - 1);
  else if (t.cur > 0) goTo(t.cur - 1, steps[t.cur - 1].length);
}
```

권장 파일 구성: `index.html` (슬라이드 마크업) + `deck.css` + `deck.js` (위 모듈들을 IIFE 또는 ES module 하나로)

---

## 6. 검증 상태 (중요)

- **모든 예제 코드는 브라우저에서 실행 테스트되지 않았음.** 통합 골격(5장)도 미검증 설계안
- 직접 열어 확인한 출처: Reveal.js 소스(keyboard/touch/location 컨트롤러, reveal.scss, highlight·notes 플러그인), Shower keys 모듈, MDN(KeyboardEvent.repeat, transitionend, View Transition API, will-change, BroadcastChannel), caniuse, Reveal.js/Slidev 공식 문서
- 일반 지식 기반(미확인): 클리커 키 매핑, Safari replaceState 빈도 제한 사유, `Element.animate`/`finished`/`skipTransition` 세부 동작, MDN의 pointer/touch-action/passive/inert/contain/counters/@page 문서 내용
- Agent 1(레이아웃)은 출처별 직접 확인 여부를 따로 밝히지 않음

## 7. 출처

- Reveal.js: [keyboard.js](https://github.com/hakimel/reveal.js/blob/master/js/controllers/keyboard.js) · [touch.js](https://github.com/hakimel/reveal.js/blob/master/js/controllers/touch.js) · [location.js](https://github.com/hakimel/reveal.js/blob/master/js/controllers/location.js) · [reveal.js layout()](https://github.com/hakimel/reveal.js/blob/master/js/reveal.js) · [reveal.scss](https://github.com/hakimel/reveal.js/blob/master/css/reveal.scss) · [highlight plugin](https://github.com/hakimel/reveal.js/blob/master/plugin/highlight/plugin.js) · [notes plugin](https://github.com/hakimel/reveal.js/blob/master/plugin/notes/plugin.js)
- Reveal.js 문서: [presentation-size](https://revealjs.com/presentation-size/) · [fragments](https://revealjs.com/fragments/) · [code](https://revealjs.com/code/) · [speaker-view](https://revealjs.com/speaker-view/)
- Shower: [core modules](https://github.com/shower/core/tree/main/lib/modules) · Slidev: [animations](https://sli.dev/guide/animations)
- W3C APG: [Carousel pattern](https://www.w3.org/WAI/ARIA/apg/patterns/carousel/)
- MDN: [KeyboardEvent.key](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/key) · [repeat](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/repeat) · [pushState](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState) · [Pointer events](https://developer.mozilla.org/en-US/docs/Web/API/Pointer_events) · [touch-action](https://developer.mozilla.org/en-US/docs/Web/CSS/touch-action) · [transitionend](https://developer.mozilla.org/en-US/docs/Web/API/Element/transitionend_event) · [Element.animate](https://developer.mozilla.org/en-US/docs/Web/API/Element/animate) · [View Transition API](https://developer.mozilla.org/en-US/docs/Web/API/View_Transition_API) · [will-change](https://developer.mozilla.org/en-US/docs/Web/CSS/will-change) · [contain](https://developer.mozilla.org/en-US/docs/Web/CSS/contain) · [inert](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/inert) · [aria-roledescription](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-roledescription) · [Container query units](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_containment/Container_queries#container_query_length_units) · [BroadcastChannel](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel)
- caniuse: [aspect-ratio](https://caniuse.com/mdn-css_properties_aspect-ratio) · [viewport units](https://caniuse.com/viewport-unit-variants) · [container query units](https://caniuse.com/css-container-query-units) · [inert](https://caniuse.com/mdn-html_global_attributes_inert) · [view-transitions](https://caniuse.com/view-transitions)
- web.dev: [animations guide](https://web.dev/articles/animations-guide) · Chrome: [same-document view transitions](https://developer.chrome.com/docs/web-platform/view-transitions/same-document)
