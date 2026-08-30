# T1b · 대시보드 위젯 — 장식(Deco) 7종 + 재미(Fun) 6종 복현 프롬프트

> 본 문서는 「00-template.md」의 5-Section 골격(Identity · Instructions · Examples · Context · Acceptance & Output)을 그대로 따른다. 이론적 근거(OpenAI 개발자 메시지 4-요소, Lovable 실천 원칙, IEEE 830 §4.3.6, Cohn 2004)는 템플릿 문서를 참조한다.
>
> **합본 근거**: 본 절의 13개 위젯은 모두 **순수 프론트엔드**(외부 데이터·백엔드·RLS 없음. 예외: WeatherModule은 인증 없는 공용 Open-Meteo REST 1건)로만 구성되며, 하나의 `ModuleCtxProps` 인터페이스를 공유한다. 원자 규칙·컴포넌트 트리·수용 기준이 문서 전체에서 반복 인용되므로, 「장식」과 「재미」를 하나의 파일로 합쳐도 각 위젯의 복현 프롬프트는 독립된 소절(§A1–§A7, §B1–§B6)로 완결된다. 즉 합본은 재현 수준을 저하시키지 않는다. T1c는 발행하지 않는다.
>
> **역할 정위**: 본 절의 위젯은 `catalogCategory ∈ {deco, fun}`이며, `teacherOnly`/`studentOnly` 플래그가 없다 → 교사·학생 대시보드 어디에나 배치 가능하다. 시스템 내 위치는 T1(교사 대시보드 골격) §5 "위젯 카탈로그"에 등록된다.

---

## ① Identity (신원)

너는 React 18 + Vite 5 + Tailwind CSS v3 + shadcn/ui 기반 「멜로디 클래스」의 시니어 프론트엔드 엔지니어 겸 한중 이중언어 교육 UX 라이터다. 한국 대학의 K-Chinese / K-Korean 교사·학습자를 위한 **대시보드 장식·재미 위젯 13종**을 순수 프론트엔드로 구현한다. 백엔드·RLS·Supabase 쿼리는 본 절에서 발생하지 않는다(예외: §B5 WeatherModule의 Open-Meteo REST 호출 1건).

---

## ② Instructions (지시)

### 2.1 산출물 (Prompt by Component, Not Page)

`src/components/dashboard/modules/` 아래에 각 위젯 1파일씩, 총 **13개 컴포넌트 파일**을 생성한다. 각 컴포넌트는 다음 시그니처를 만족한다.

```ts
// src/components/dashboard/ModuleRenderer.tsx 에서 정의
export interface ModuleCtxProps {
  instance: ModuleInstance;                                  // {id, type, size, config?}
  onConfigChange: (id: string, config: Record<string, any>) => void;
  editing: boolean;
}
```

컴포넌트 목록·기본 크기·카탈로그 카테고리는 `src/components/dashboard/dashboardModuleCatalog.tsx`의 등록 순서와 완전히 일치해야 한다.

| # | ModuleType (`type`) | 파일명 | 카탈로그 label | defaultSize | category |
|---|---------------------|--------|----------------|-------------|----------|
| A1 | `aurora_deco` | `AuroraDecoModule.tsx` | 오로라 그라데이션 | S | deco |
| A2 | `floating_notes` | `FloatingNotesModule.tsx` | 음표 떠오르기 | S | deco |
| A3 | `space_deco` | `SpaceDecoModule.tsx` | 별자리/우주 | S | deco |
| A4 | `ripple_interactive` | `RippleInteractiveModule.tsx` | 파문 인터랙티브 | S | deco |
| A5 | `music_visualizer` | `MusicVisualizerModule.tsx` | 음악 비주얼라이저 | S | deco |
| A6 | `art_text` | `ArtTextModule.tsx` | 아트 텍스트 | S | deco |
| A7 | `wave_deco` | `WaveDecoModule.tsx` | 음파 장식 | S | deco (legacy) |
| B1 | `fortune` | `FortuneModule.tsx` | 오늘의 운세 | S | fun |
| B2 | `tarot` | `TarotModule.tsx` | 타로 카드 | M | fun |
| B3 | `fortune_cookie` | `FortuneCookieModule.tsx` | 포춘쿠키 | S | fun |
| B4 | `gacha` | `GachaModule.tsx` | 랜덤 뽑기 | S | fun |
| B5 | `weather` | `WeatherModule.tsx` | 날씨 위젯 | S | fun |
| B6 | `motivation` | `MotivationModule.tsx` | 동기부여 카드 | S | fun |

각 파일 등록 후 `ModuleRenderer.tsx`의 `switch (instance.type)`에 케이스를 추가하고, `dashboardModuleCatalog.tsx`의 `DASHBOARD_CATALOG` 배열에 정확한 순서(deco 6항목 + fun 6항목 + legacy `wave_deco`)로 등록한다.

### 2.2 원자적 UI 규칙 (Speak Atomic)

모든 위젯 공통:
- 루트 컨테이너는 `relative h-full min-h-[140px] rounded-[14px] overflow-hidden`(§B1·B2·B5·B6은 예외로 `min-h-[160px]|[200px]`).
- shadcn/ui `ModuleShell`(`./_ModuleShell`) 을 쓰는 위젯: §B2, §B3, §B4. 나머지는 자체 카드로 그린다.
- 애니메이션 keyframe은 위젯 내부 `<style>{`@keyframes dash-<name>{...}`}</style>` 로만 선언한다. 전역 CSS 오염 금지.
- 하드코드 색상(`bg-[#…]`, `text-white`, `bg-black`)은 **그라데이션 배경/캔버스 픽셀 색상**에 한해 허용된다(장식·재미 위젯은 시각 자체가 상품이므로 semantic token 예외). 텍스트 색상은 `text-white/70` 등 Tailwind opacity 유틸리티를 우선 사용한다.
- Canvas 사용 위젯(§A1, §A3, §A5)은 `ResizeObserver` + `devicePixelRatio` 로 고DPI 대응하고, 언마운트 시 `cancelAnimationFrame` + `ro.disconnect()` 로 누수 없이 정리한다.
- `onConfigChange`를 호출하는 위젯(§A1 variant, §A5 color, §A6 text/style)은 debounce **400 ms** 를 지킨다(§A6만 debounce, 나머지는 select 변경 즉시 저장).

### 2.3 강제 제약 (Design with Real Content)

- 모든 문구는 순수 한국어. lorem ipsum·placeholder 영어 금지.
- `<h1>` 사용 금지(대시보드 페이지에는 이미 h1이 존재).
- 외부 fetch는 §B5 WeatherModule 1건뿐. 반드시 `AbortController` **없이** — 이 위젯은 마운트 시 1회만 호출하고 결과를 상태에 캐시한다(대시보드 재렌더 부담 최소화). 실패 시 `err=true` 상태로 「날씨 정보를 불러올 수 없습니다.」를 렌더한다.
- 랜덤성이 있는 위젯(§B1 오늘의 운세)은 **날짜 시드**를 사용해 하루 동안 결과가 고정되도록 한다. `다른 운세 보기` 버튼은 `offset` 을 +1하여 시드를 흔든다.

---

## ③ Examples (예시)

### 3.1 확정 카피 표 (Design with Real Content)

| 위치 | 문자열 |
|------|--------|
| §A1 중앙 오버레이 | `오늘도 화이팅 🎵` |
| §A1 variant select | `메쉬` / `파도` |
| §A3 하단 라벨 | `🌌 우주 산책` |
| §A4 중앙 안내 | `클릭해서 파문을 만들어보세요` |
| §A5 color select | `인디고` / `그린` / `앰버` / `핑크` |
| §A6 style select | `네온` / `레인보우` / `글리치` / `타이핑` |
| §A6 기본 텍스트 | `MELODY CLASS` |
| §B1 상단 캡션 | `오늘의 운세` |
| §B1 배지 3종 | `금전 N★` / `애정 N★` / `학업 N★` |
| §B1 버튼 | `다른 운세 보기` |
| §B1 라인 풀 | `오늘은 새로운 영감이 떠오르는 날.` / `작은 친절이 큰 행운으로 돌아옵니다.` / `용기 있는 선택이 미래를 바꿉니다.` / `음악과 함께라면 모든 게 잘 풀립니다.` / `예상치 못한 만남이 기다리고 있어요.` / `오늘의 한 걸음이 내일의 도약입니다.` |
| §B2 타이틀 | `타로 카드` |
| §B2 우상단 | `다시 섞기` (Shuffle 아이콘) |
| §B2 하단 안내 | `카드를 클릭해 뒤집어 보세요` |
| §B2 카드 풀(12장) | `태양·달·별·검·탑·연인·광대·황제·여사제·정의·운명의 수레바퀴·장미` |
| §B3 타이틀 | `포춘쿠키` |
| §B3 초기 메시지 | `쿠키를 클릭해 보세요` |
| §B3 메시지 풀 | `오늘 부른 노래가 누군가의 하루를 바꿉니다.` / `가르치는 즐거움이 곧 배움의 시작.` / `한 곡의 감동이 한 권의 책보다 깊을 수 있습니다.` / `작은 연습이 큰 무대를 만듭니다.` / `오늘의 학생은 내일의 스승.` / `음악은 마음의 언어, 멈추지 마세요.` |
| §B4 타이틀 | `랜덤 뽑기` |
| §B4 안내 | `버튼을 눌러 뽑아보세요` |
| §B4 아이템 8종 | `행운 선물(SSR) · 특별한 멜로디(SR) · 지식의 책(R) · 별빛 조각(SR) · 영감의 붓(R) · 보석(SSR) · 네잎클로버(SR) · 마법의 지팡이(SSR)` |
| §B5 도시 라벨 | `서울` |
| §B5 실패 | `날씨 정보를 불러올 수 없습니다.` |
| §B5 로딩 | `불러오는 중…` |
| §B6 문구 풀 | `오늘의 한 곡이 누군가의 인생곡이 됩니다.` / `포기하지 마세요. 당신의 가르침이 누군가의 길을 밝힙니다.` / `어려운 날도, 음악이 있다면 견딜 수 있어요.` / `작은 진보도 진보입니다. 한 걸음씩.` / `오늘도 한 명의 학생에게 영감을 주세요.` |
| §B6 버튼 | `다른 메시지` (RefreshCw 아이콘) |

### 3.2 파일 트리 (Use Prompt Patterns for Layouts)

```text
src/components/dashboard/
├── ModuleRenderer.tsx                # switch 케이스 13건 추가
├── dashboardModuleCatalog.tsx        # DASHBOARD_CATALOG 배열에 13항목 등록
└── modules/
    ├── _ModuleShell.tsx              # (기존) title/icon/right 슬롯 카드
    ├── AuroraDecoModule.tsx          # §A1  canvas + variant select
    ├── FloatingNotesModule.tsx       # §A2  ♩♪♫♬🎵🎶 부유 애니메이션
    ├── SpaceDecoModule.tsx           # §A3  canvas + 별 90개 + 유성 랜덤
    ├── RippleInteractiveModule.tsx   # §A4  onClick 파문 (0.85s)
    ├── MusicVisualizerModule.tsx     # §A5  40개 막대 + color select
    ├── ArtTextModule.tsx             # §A6  4스타일 텍스트 + 입력·debounce 400ms
    ├── WaveDecoModule.tsx            # §A7  SVG 5막대(legacy)
    ├── FortuneModule.tsx             # §B1  별자리+운세 (일-시드)
    ├── TarotModule.tsx               # §B2  3장 flip
    ├── FortuneCookieModule.tsx       # §B3  🥠 균열 애니메이션
    ├── GachaModule.tsx               # §B4  8아이템 + SSR/SR/R
    ├── WeatherModule.tsx             # §B5  Open-Meteo 5일 예보
    └── MotivationModule.tsx          # §B6  5문구 순환
```

### 3.3 위젯별 핵심 스펙 요약

**§A1 AuroraDecoModule** — `<canvas>` 위에 3개 radial gradient(색 3종)를 `globalCompositeOperation="screen"`으로 겹쳐 오로라를 만든다. `time = t*0.0006` 기준으로 각 중심을 `cos/sin`으로 흔든다. variant `mesh`(핑크/보라/파랑) / `wave`(하늘/인디고/보라). 우상단 select로 전환 즉시 `onConfigChange` 호출. 중앙에 `오늘도 화이팅 🎵` 오버레이(font-extrabold, drop-shadow-lg, pointer-events-none).

**§A2 FloatingNotesModule** — `♩♪♫♬🎵🎶` 6종 × 14개를 하단에서 위로 부유(`@keyframes dash-floatup`, 4–8s, -160px 이동 + 20deg 회전, opacity 0→1→0.8→0). 배경 `linear-gradient(135deg,#1a1340,#2d1b69)`. 초기 랜덤값은 `useMemo`로 고정(재마운트 전까지 동일).

**§A3 SpaceDecoModule** — 별 90개(반짝임: `sin(p)`), 우상단에 소행성 1개(radial gradient), 유성은 `Math.random()<0.01`로 소환·`vx=6*dpr, vy=2*dpr`·수명 100프레임. 하단 라벨 `🌌 우주 산책`.

**§A4 RippleInteractiveModule** — `onClick`시 클릭 좌표에 `border-2 border-white/60` 원을 생성, `@keyframes dash-rippleout` 0.85s 후 자동 제거. `useState` 리스트 + `idRef` 카운터. 배경 `linear-gradient(135deg,#0ea5e9,#0284c7)`, 커서 `cursor-crosshair`.

**§A5 MusicVisualizerModule** — 4px 폭 막대 40개, 각 막대 높이 `10 + (sin(t*0.005 + i*0.4)*0.5+0.5)*90` %. 팔레트 4종(인디고/그린/앰버/핑크) `linear-gradient(to top, ...)`. 배경 `linear-gradient(to bottom,#1a1340,#0d0920)`.

**§A6 ArtTextModule** — 4스타일:
- `neon`: `#a5b4fc`, `@keyframes dash-neon` 2s alternate(text-shadow 강도 변화).
- `rainbow`: 6색 그라데이션 텍스트 + `background-size:200%` + `@keyframes dash-rainbow` 2s linear(background-position 이동).
- `glitch`: white text + `@keyframes dash-glitch` 3s(translate ±2px + hue-rotate).
- `typing`: 문자 하나씩 200ms 간격 추가, `|` 커서 `animate-pulse`.
- 입력 `<input>` + `<select>` 하단 배치. debounce 400ms → `onConfigChange({text, style})`.

**§A7 WaveDecoModule (legacy)** — 200×60 SVG, 5개 `<rect>` height 6→30→6 애니메이션(각각 dur 0.8+i*0.15s), 배경 `bg-gradient-to-br from-indigo-50 to-violet-50`. 카탈로그 배열 **맨 끝**에 `Legacy keep` 주석과 함께 배치.

**§B1 FortuneModule** — 별자리 12종 상수 배열. 시드: `seedFromDate(new Date(), offset)` = `ISO(YYYY-MM-DD)`를 31진 해시 + `offset*7919`. `LCG(1664525, 1013904223)`로 별자리·별점(1–5)·금전·애정·학업·라인을 순서대로 뽑음. 배지 3종 pill 스타일(금전=앰버·애정=핑크·학업=인디고). 배경 별 20개 `((i*53)%100, (i*37)%100)` 결정론적 배치.

**§B2 TarotModule** — 12장 풀에서 3장 뽑음(중복 없이 `arr.splice`). 카드 컨테이너 `perspective:800`, 각 카드 `72×120px`, `transformStyle:preserve-3d`, `transition:transform .6s`, 클릭시 `rotateY(180deg)`. 뒷면 보라 그라데이션 + `✦`, 앞면 앰버 그라데이션 + 이모지·한글명. `다시 섞기` 클릭시 `round++` & flipped 전부 false.

**§B3 FortuneCookieModule** — 🥠 이모지 버튼(48px). 클릭 시 `crack=true` → 250ms 후 문구 교체 & `crack=false`. `transform: scale(1.15) rotate(8deg)` 만 사용.

**§B4 GachaModule** — 아이템 8종, 등급 SSR(앰버 #f59e0b) / SR(바이올렛 #8b5cf6) / R(에메랄드 #10b981). 클릭 → `spin=true` → `@keyframes dash-spin` 0.5s(rotate 0→360deg + scale 1→1.1→1) → 700ms 후 결과 표시. 결과 카드에 등급 pill.

**§B5 WeatherModule** — `useEffect(() => fetch(...), [])`로 Open-Meteo GET `https://api.open-meteo.com/v1/forecast?latitude=37.5665&longitude=126.9780&current=temperature_2m,weather_code&daily=temperature_2m_max,temperature_2m_min,weather_code&timezone=Asia/Seoul&forecast_days=5`. `WCODE` 상수로 코드→(이모지·한글) 매핑. 상단 도시명 `서울`, 현재 온도(text-[44px] font-black) + 상태. 하단 5일 카드(요일 `["일","월",…]`, 이모지, 최고/최저).

**§B6 MotivationModule** — 문구 5종(icon·text·tags[]). 마운트 시 `Math.floor(Math.random()*5)` 랜덤 시작, `다른 메시지` 버튼으로 `(i+1)%5` 순환. 아이콘은 `@keyframes dash-float`(3s, translateY 0↔-6px). 배경 `linear-gradient(135deg,#fef3c7,#fde68a)`, 텍스트 `#92400e`.

---

## ④ Context (배경)

### 4.1 프로젝트 맥락

「멜로디 클래스」의 대시보드(`/dashboard`)는 4가지 카테고리(**공용 · 장식 · 재미 · 역할별**)로 구성된 최대 44종 위젯 카탈로그를 가진다. 본 문서의 **장식 7종**과 **재미 6종**은 카테고리 필드 `deco` / `fun`이며 `teacherOnly`/`studentOnly` 플래그가 없어 교사·학생 대시보드 어디에나 배치 가능하다. 이들은 데이터 종속이 없어 온보딩 직후에도 시각적 "차 있음"을 즉시 제공하는 역할을 한다.

### 4.2 Lovable Cloud 후경 (Build with Lovable Cloud in Mind)

- 인증 무관 · RLS 무관 · Supabase 쿼리 무관.
- 예외: §B5 WeatherModule 은 인증 없는 공용 REST(Open-Meteo)를 1회 호출한다. 4-상태 렌더링을 지킨다: 로딩(`불러오는 중…`) · 실패(`날씨 정보를 불러올 수 없습니다.`) · 빈(해당 없음, 성공 시 최소 1일치 보장) · 성공(`current` + 5일).
- 사용자 설정(`instance.config`)은 상위 대시보드 훅(`useDashboardLayout`)이 Supabase 프로필 컬럼에 직렬화하여 저장하므로 본 절에서는 `onConfigChange` 호출만 담당한다.

### 4.3 데이터 계약

본 절에서는 신규 테이블·정책을 만들지 않는다. `instance.config` 스키마만 정의한다.

```ts
// AuroraDeco
config?: { variant: "mesh" | "wave" }         // default "mesh"
// MusicVisualizer
config?: { color: "indigo" | "green" | "amber" | "pink" }  // default "indigo"
// ArtText
config?: { text: string; style: "neon" | "rainbow" | "glitch" | "typing" }
//         default { text: "MELODY CLASS", style: "neon" }
// 그 외 11종: config 없음
```

---

## ⑤ Acceptance & Output (검증·산출)

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6, 정량 임계값만)

- 파일 수: `src/components/dashboard/modules/`에 신규 파일 정확히 **13개** 생성.
- `ModuleRenderer.tsx` `switch`문 케이스 수 = 등록된 `ModuleType` 수와 일치(`grep -c "case \"" src/components/dashboard/ModuleRenderer.tsx` ≥ 44).
- `dashboardModuleCatalog.tsx`의 `DASHBOARD_CATALOG` 배열 길이 ≥ 44, `category==="deco"` 항목 = **7건**, `category==="fun"` 항목 = **6건**.
- 정적 검사:
  - `grep -R "text-white\|bg-black\|bg-\[#" src/components/dashboard/modules/{AuroraDeco,FloatingNotes,SpaceDeco,RippleInteractive,MusicVisualizer,ArtText,WaveDeco,Fortune,Tarot,FortuneCookie,Gacha,Weather,Motivation}Module.tsx` — 하드코드 허용 위치(그라데이션·캔버스 fillStyle·drop-shadow overlay) 외 0건.
  - `grep -R "lorem\|Lorem\|ipsum" src/components/dashboard/modules/` = 0건.
  - `grep -R "<h1" src/components/dashboard/modules/` = 0건.
- 성능(Chrome DevTools Performance, 위젯 1개 마운트 기준):
  - Canvas 3종(§A1·A3·A5) FPS 평균 ≥ 55(60Hz 디스플레이).
  - §A4 파문 클릭 시 새 DOM 노드는 850 ms 이내 자동 제거(리크 0).
  - §B5 fetch 1회 응답 시간 실패 시 UI 반영 ≤ 300 ms(에러 상태 즉시 표시).
- 접근성: 모든 인터랙션 요소(`<button>`, `<select>`)는 키보드로 focus 가능, tab-index 무결.
- 정적 시드 재현성(§B1): 동일 날짜·동일 offset에서 별자리·별점·라인이 100% 동일해야 한다.

### 5.2 Output Format

LLM은 다음 순서로 파일만 반환한다. 설명·사과·주석·마크다운 헤더·빈 줄 이외의 안내문 금지.

1. `src/components/dashboard/modules/AuroraDecoModule.tsx`
2. `src/components/dashboard/modules/FloatingNotesModule.tsx`
3. `src/components/dashboard/modules/SpaceDecoModule.tsx`
4. `src/components/dashboard/modules/RippleInteractiveModule.tsx`
5. `src/components/dashboard/modules/MusicVisualizerModule.tsx`
6. `src/components/dashboard/modules/ArtTextModule.tsx`
7. `src/components/dashboard/modules/WaveDecoModule.tsx`
8. `src/components/dashboard/modules/FortuneModule.tsx`
9. `src/components/dashboard/modules/TarotModule.tsx`
10. `src/components/dashboard/modules/FortuneCookieModule.tsx`
11. `src/components/dashboard/modules/GachaModule.tsx`
12. `src/components/dashboard/modules/WeatherModule.tsx`
13. `src/components/dashboard/modules/MotivationModule.tsx`
14. `src/components/dashboard/ModuleRenderer.tsx` (13개 case 병합본)
15. `src/components/dashboard/dashboardModuleCatalog.tsx` (deco 7 + fun 6 항목 병합본)
