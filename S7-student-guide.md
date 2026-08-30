# S7 — 학생 사용 가이드 (Student Usage Guide) 복현 프롬프트

> 대상 모듈: 학생 로그인 시 `/guide` 로 진입했을 때 노출되는 **학생 전용 사용 가이드** 전체.
> 관련 파일: `src/pages/Guide.tsx`, `src/pages/StudentGuide.tsx`, `src/components/guide/student/content.ts`, `src/components/guide/student/tourSteps.ts`, `src/components/guide/student/demo/DemoOverview.tsx`, `src/components/guide/student/demo/data.ts`, 공용 `src/components/guide/ModuleCard.tsx`, `src/components/guide/ModuleCarousel.tsx`, `src/components/guide/useGuideTour.ts`, `src/components/guide/tourSteps.ts` (병합 export).
> 본 문서는 T9(교사 사용 가이드)와 **동일 셸·다른 데이터** 구조를 따른다. 공용 컴포넌트는 T9 를 참조하고, 여기서는 학생용 차이점만 완전히 명세한다.

---

## 1. Identity

역할: 학생 사용자에게 플랫폼 사용법을 **자동 둘러보기(전체 스토리보드)** 와 **설명서(모듈 카드)** 두 방식으로 안내하는 페이지를 재현한다. 교사 가이드와 완전히 분리되며, 학생은 자신에게 노출되는 UI 만 학습할 수 있어야 한다.

## 2. Instructions

### 2.1 라우팅 · 진입

- 라우트 `/guide` 는 `src/pages/Guide.tsx` 가 담당한다.
- `useAuth()` 로 `isTeacher / isAdmin / loading` 을 판별.
  - `loading` 중 : `<div class="min-h-screen flex items-center justify-center text-sm text-muted-foreground">로딩 중...</div>` 표시.
  - `isAdmin && ?as=student` 쿼리 : 강제로 `<StudentGuide />` 렌더 (미리보기).
  - `isAdmin && ?as=teacher` 쿼리 : 강제로 `<TeacherGuide />`.
  - 그 외 : `isTeacher ? <TeacherGuide /> : <StudentGuide />`.
- 학생 사이드바(`AppSidebar`)에서 「사용 가이드」 링크가 있으면 `/guide` 로 이동한다.

### 2.2 StudentGuide 셸 (`src/pages/StudentGuide.tsx`)

레이아웃 (Hero + 2-Tab):

1. **Hero Card**
   - `<Card class="relative overflow-hidden border-border/60 bg-gradient-to-br from-primary/10 via-background to-accent/10">`.
   - 우상단 radial gradient 오버레이 `opacity-30`.
   - 좌측 12x12 (sm:16x16) `rounded-2xl bg-primary/15` 아이콘 박스 — `BookOpenCheck` (lucide) `text-primary`.
   - 제목 : `학생 사용 가이드`.
   - 서브카피 : `학생이 사용하는 모든 기능을 한 곳에. 자동 둘러보기 · 설명서 두 가지 방식으로 안내합니다.` (`두 가지 방식` 은 `text-primary font-medium`).
   - 하단 : `Sparkles` 아이콘 + `총 {STUDENT_GUIDE_MODULES.length}개 모듈 · {totalFeatures}개 세부 기능`.
   - `totalFeatures = STUDENT_GUIDE_MODULES.reduce((s, m) => s + m.features.length, 0)`.

2. **Tabs (`value` 상태 = `params.get('tab') ?? "overview"`)**
   - `TabsList` : `grid grid-cols-2 w-full h-auto`.
   - Trigger 1 (`value="overview"`) : `PlayCircle` + `🎬 전체 둘러보기`.
   - Trigger 2 (`value="docs"`) : `BookOpen` + `📖 설명서`.
   - **Overview 탭 컨텐츠** : `<StudentDemoOverview active={tab === "overview"} />`.
   - **Docs 탭 컨텐츠** :
     - 상단 라벨 `모듈 둘러보기` (`text-sm font-semibold uppercase tracking-wide text-muted-foreground`).
     - `<ModuleCarousel modules={STUDENT_GUIDE_MODULES} onSelect={setActiveId} activeId={activeId} />`.
     - 이후 `STUDENT_GUIDE_MODULES.map` → 각 모듈을 `id="mod-{m.id}" scroll-mt-4` 래퍼 + 공용 `<ModuleCard module={m} />`.

3. **푸터** : `<div class="text-center text-xs text-muted-foreground py-6">더 궁금한 점이 있으면 설정 → 문의로 알려주세요.</div>`.

### 2.3 학생 가이드 모듈 데이터 (`STUDENT_GUIDE_MODULES`, 7 개)

각 모듈은 `GuideModule` 타입(교사와 공유): `{ id, route?, icon, accent, ko:{title,subtitle}, features:[{ id, ko:{name,usage,output}, targetSelector? }] }`.

| # | id | icon (lucide) | route | accent (Tailwind gradient) | ko.title / subtitle |
|---|---|---|---|---|---|
| 1 | `student-dash` | `Sparkles` | `/student-home` | `from-blue-500/20 to-indigo-500/20` | 내 공간 / 위젯 편집 가능한 학습 데스크 |
| 2 | `student-songs` | `Music` | `/songs` | `from-pink-500/20 to-rose-500/20` | 노래 아카이브 / 탐색·즐겨찾기·학습 분석 |
| 3 | `course-join` | `UserPlus` | `/student-home` | `from-emerald-500/20 to-teal-500/20` | 반 입장하기 / 초대 링크 → 자동 채움 → 한 번에 입장 |
| 4 | `course-room` | `BookOpen` | `/student-home#my-courses` | `from-cyan-500/20 to-sky-500/20` | 우리 반 / 5개 탭 · 학생 시점 |
| 5 | `student-materials` | `FolderOpen` | `/student-home#my-courses` | `from-amber-500/20 to-orange-500/20` | 자료 다운로드·미리보기 / PDF 인라인·미디어 재생·외부 링크 |
| 6 | `student-settings` | `Settings` | `/settings` | `from-purple-500/20 to-fuchsia-500/20` | 설정 / 프로필·아바타·비밀번호 |
| 7 | `student-trash` | `Trash2` | `/trash` | `from-slate-500/20 to-zinc-500/20` | 휴지통 / 노래·과정 7일간 복원 가능 |

세부 features (아코디언 항목 — 각 항목은 `usage`, `output` 두 컬럼):

1. **student-dash** (5 features)
   - `sd-edit` — 위젯 편집 모드 (`targetSelector: [data-tour='dash-edit']`) — usage: 우상단「위젯 편집」클릭 → 드래그·리사이즈·삭제 가능, 「완료」로 저장 / output: 맞춤형 학습 공간.
   - `sd-favorites` — 즐겨찾기 노래 위젯 — usage: ♥ 한 곡들이 YouTube 썸네일 카드로 표시; 클릭 시 분석 다이얼로그(읽기 전용) 열림 / output: 좋아하는 곡 빠르게 복습.
   - `sd-notice` — 공지 확인 위젯 — usage: 가입한 반 공지 모임, 펼쳐서 전문·답글 / output: 알림 누락 방지.
   - `sd-courses` — 내 수업 / 공개 수업 — usage: 가입한 반 vs 공개 설정된 반 탐색 / output: 반에 빠르게 입장.
   - `sd-deco` — 장식·재미 위젯 — usage: 파티클·오로라·음표·오늘의 운세·동기부여 자유 배치 / output: 감성적인 공간.

2. **student-songs** (5 features)
   - `ss-browse` (`[data-tour='songs-filter']`) — 곡 탐색·검색 / usage: 언어·레벨·교학포인트·테마 필터 + 검색.
   - `ss-fav` (`[data-tour='song-card']`) — ♥ 즐겨찾기 / usage: 카드의 ♥ 클릭만으로 저장; 「내 공간」의 즐겨찾기 위젯에서 재발견.
   - `ss-analysis` — 분석 다이얼로그 (6 탭, 읽기 전용) / usage: 카드의 👁 → 영상·가사·단어·문법·읽기·탐구.
   - `ss-explore` — 🔎 탐구 (심층 학습) / usage: 곡 정보·가사 심화·연습·수업 도구 16편.
   - `ss-tts` — 🔊 읽기·받아쓰기 / usage: 가사 TTS 따라 읽기, 탐구의 「받아쓰기」 3단계 난이도.

3. **course-join** (4) : `cj-link` (초대 링크) · `cj-prefill` (프로필 자동 채움) · `cj-avatar` (12 종 아바타 or 업로드) · `cj-confirm` (확인 후 입장).

4. **course-room** (5) : `cr-home` (홈) · `cr-calendar` (캘린더) · `cr-materials` (강의 자료 3 서브탭) · `cr-notice` (알림 양방향) · `cr-members` (우리 반 그리드, 「Me」 표시).

5. **student-materials** (4) : `sm-pdf` (PDF 인라인) · `sm-media` (오디오·비디오 재생) · `sm-link` (외부 링크·메모) · `sm-download` (다운로드).

6. **student-settings** (3) — 알림 항목 **없음**:
   - `stset-profile` (`[data-tour='settings-tab-profile']`).
   - `stset-avatar`.
   - `stset-security` (`[data-tour='settings-tab-security']`).

7. **student-trash** (3) :
   - `sttrash-tabs` — 2 개 탭 구성 (노래·과정만).
   - `sttrash-restore` — 복원 / 영구 삭제.
   - `sttrash-retention` — 7 일 자동 정리 (남은 일수 배지).

정확한 한국어 카피(`usage`, `output`)는 `src/components/guide/student/content.ts` 를 그대로 사용한다.

### 2.4 공용 셸 컴포넌트 재사용

- **`ModuleCarousel`** (T9 §3 참조): 상단 스크롤 스냅형 캐러셀. `modules`, `activeId`, `onSelect` 3 개 prop. 클릭 시 `document.getElementById('mod-{id}')?.scrollIntoView({behavior:'smooth', block:'start'})` 로 스크롤한다.
- **`ModuleCard`** (`src/components/guide/ModuleCard.tsx`): 헤더 그라디언트 배너 + `현장 체험` 버튼(옵션) + `Accordion type="multiple"` 로 features 나열.
  - `LABELS = { usage: "어떻게 사용", output: "결과", tour: "🚀 현장 체험" }`.
  - `현장 체험` 클릭 시 : `route` 를 `path?tour={module.id}#hash` 로 변환해 `navigate()`.
  - features 아코디언 내부는 `grid sm:grid-cols-2 gap-3` 로 usage/output 두 열.

### 2.5 driver.js 투어 (`STUDENT_TOUR_STEPS`, `src/components/guide/student/tourSteps.ts`)

- 헬퍼 3 종 : `modal(title, description)` · `at(selector, title, description)` · `final(title, description, selector?)` → `FinalStep = DriveStep & { __final?: boolean }`.
- 최종 export 는 `src/components/guide/tourSteps.ts` 에서 `LEGACY_STEPS`, `TEACHER_TOUR_STEPS`, `STUDENT_TOUR_STEPS` 를 spread 병합한 `TOUR_STEPS`.
- 학생 투어 키(6 종) 및 단계 개요:

| 키 | 단계 수 | 주요 selector / 내용 |
|---|---|---|
| `student-dash` | 6 | `[data-tour='dash-edit']` → 위젯 추가 modal → 즐겨찾기 노래 → 공지 확인 → 내 수업·공개 수업 → final(같은 selector 재사용). |
| `student-songs` | 6 | `[data-tour='songs-filter']` → `[data-tour='song-card']` → 분석 6 탭 modal → 탐구 16 편 modal → 읽기·받아쓰기 modal → final. |
| `course-join` | 5 | 초대 링크 · 프로필 자동 채움 · 아바타 · 확인 후 입장 · final. |
| `course-room` | 7 | 5 탭 구조 modal + 각 탭(홈/캘린더/강의 자료 3 서브탭/알림 양방향/우리 반) modal + final. |
| `student-materials` | 5 | 강의 자료 탭 안내 · PDF 인라인 · 미디어 · 외부 링크·메모 · final. |
| `student-settings` | 3 | `[data-tour='settings-tab-profile']` · `[data-tour='settings-tab-security']` · final. **알림 단계 제거**. |
| `student-trash` | 3 | 2 탭 구성 · 복원/영구 삭제 · final(7 일 정리). |

각 스텝의 정확한 한국어 title/description 은 `student/tourSteps.ts` 원문을 그대로 사용한다.

### 2.6 `useGuideTour` 훅 (공용, 상세는 T9 §5)

- `?tour=<key>` 쿼리 감지 → 병합된 `TOUR_STEPS[key]` 로 driver.js 실행.
- anchor 폴링 : 최대 25 × 200ms = 5 초. anchor 없으면 즉시 시작.
- 최종 스텝(`__final`) 은 popover 버튼 오버라이드:
  - `nextBtnText: "🚀 지금 시작"` → `stayOnPage=true` + destroy.
  - `prevBtnText: "← 사용 가이드로"` → destroy 후 `/guide?tab=docs` 이동.
- `onDestroyed` 기본 동작: `stayOnPage` 아니면 `/guide?tab=docs` 로 복귀.
- 라우팅 훅 안에서 학생 페이지(`/student-home`, `/songs`, `/settings`, `/trash`, `/shared/course/*`)의 anchor 를 감지해야 하므로 반드시 `AppLayout` 하위 페이지에 이 훅이 마운트되어야 한다.

### 2.7 Overview 스토리보드 (`StudentDemoOverview`, `student/demo/DemoOverview.tsx`)

- 라이브러리 : `embla-carousel-react` (`{ loop:false, align:"start", duration:25 }`).
- 상태 : `current`, `playing`(기본 true), `timerRef`. `active` prop 이 false 이거나 `playing=false` 면 auto-advance 정지.
- 슬라이드 리스트 : `STUDENT_DEMO_SLIDES` (14 개).
- 슬라이드 종류 : `intro` · `transition` × 6 · `student-dash` · `student-songs` · `course-join` · `course-room` · `student-materials` · `student-settings` · `outro`.
- 각 콘텐츠 슬라이드는 `<SlideMeta>` 래퍼로 감싸며, 좌측 300px 메타 패널(제목/설명/아이콘/accent) + 우측 스냅샷. 모바일에서는 세로 stacked, md 이상에서 2 컬럼 grid `[300px_1fr]`.
- 하단 컨트롤 카드(`Card`):
  - Pause/Play, Restart(`RotateCcw`), Prev(`ChevronLeft`), Next(`ChevronRight`) icon 버튼.
  - `Slider min=0 max={total-1} step=1` 로 진행률 표시 & 이동.
  - 우측 `{current+1} / {total}` 뱃지 (`font-mono tabular-nums`).
- `STUDENT_SLIDE_META` 로 인덱스별 `{ ko:{title,body}, zh:{title,body}, ms }` 지정 :
  - 콘텐츠 슬라이드 지속 시간 `7000ms`.
  - transition 슬라이드 `3500ms`.
  - intro `6000ms`, outro `6000ms`.
- 스냅샷 소규모 컴포넌트 사양:

| 슬라이드 | 함수 | 렌더 |
|---|---|---|
| intro | `IntroSnapshot` | 2×2 그리드, 4 카테고리(내 공간/노래 아카이브/우리 반/설정) 이모지 + 라벨. |
| student-dash | `DashSnapshot` | 4 위젯 카드(즐겨찾기 노래 / 공지 확인 / 내 수업 / 공개 수업). |
| student-songs | `SongsSnapshot` | 곡 카드(썸네일+제목「愛しているから _ 사랑하니까」+아티스트「아이유 · K-pop」+♥) + 3 열 6 탭 pill. |
| course-join | `JoinSnapshot` | 4 단계 리스트(초대 링크→자동 채움→아바타→입장). |
| course-room | `RoomSnapshot` | 5 열 그리드, 이모지+라벨(홈/캘린더/강의 자료/알림/우리 반). |
| student-materials | `MaterialsSnapshot` | 2×2 카드(PDF/MP3/MP4·YouTube/외부 링크). |
| student-settings | `SettingsSnapshot` | 4 줄 리스트: 🪪 프로필 / 🎭 아바타(12종 또는 직접 업로드) / 🔒 비밀번호 변경 / 🗑 휴지통 — 노래·과정 7 일 복원. **알림 항목 없음.** |
| outro | `OutroSnapshot` | 3 단계 리스트 + `Button size=lg` → `navigate('/student-home')`, 라벨 `내 공간으로`. |
| transition | 공용 `DemoTransitionSlide` | `titleKo/titleZh/subKo/subZh` 4 prop. |

- transition 시퀀스 라벨(순서대로) :
  1. `1. 내 공간에서 시작` / `위젯을 자유롭게 편집`
  2. `2. 곡 찾고 학습` / `♥ 즐겨찾기 + 6 탭 분석`
  3. `3. 반에 들어가기` / `초대 링크 한 번이면 끝`
  4. `4. 우리 반 둘러보기` / `5 개 탭의 학생 시점`
  5. `5. 자료 보기` / `PDF·음원·영상 그대로 재생`
  6. `6. 내 설정 정리` / `프로필·아바타·비밀번호`
- outro 스냅샷 3 단계 리스트 : `1. 「내 공간」에서 위젯 추가` / `2. 노래 아카이브에서 ♥` / `3. 교사 초대 링크로 반 입장`.

### 2.8 UI 라벨 사전 (한국어 하드코딩)

```
labels = {
  pause: "일시정지", play: "재생", restart: "처음부터",
  prev: "이전", next: "다음", cta: "내 공간으로"
}
```

## 3. Examples

### 3.1 「현장 체험」 진입 흐름

1. 사용자가 Docs 탭에서 `내 공간` 카드의 「🚀 현장 체험」 클릭.
2. `ModuleCard` 내부 : `navigate('/student-home?tour=student-dash')`.
3. `/student-home` 마운트 → `AppLayout` 하위에서 `useGuideTour` 훅 실행 → `TOUR_STEPS['student-dash']` 로 driver.js 시작.
4. anchor `[data-tour='dash-edit']` 폴링 성공 → 6 스텝 팝오버 순차 노출.
5. 마지막 스텝에서 「🚀 지금 시작」 → 팝오버 destroy, 페이지 유지.
6. 「← 사용 가이드로」 → destroy 후 `/guide?tab=docs` 복귀.

### 3.2 Admin 미리보기

- 관리자 계정으로 `/guide?as=student` 접속 → `TeacherGuide` 대신 `StudentGuide` 강제 렌더 (미리보기용, 다른 사용자 동작에는 영향 없음).

### 3.3 Overview 자동 재생 정지 조건

- `<Tabs>` 가 `docs` 로 이동 → `active=false` 프로퍼티가 전달 → `useEffect` 가 타이머 취소.
- 사용자가 Pause 클릭 → `playing=false` → 타이머 취소, 재개 시 현재 슬라이드의 `ms` 로 다시 스케줄.

## 4. Context

### 4.1 데이터 계약

- 순수 프론트엔드 데이터. Supabase 호출 없음.
- `useAuth()` 만 서버 상태 의존(`isTeacher/isAdmin/loading`).
- `driver.js` (`driver.js/dist/driver.css`) 는 `useGuideTour` 훅에서 로드.

### 4.2 라우팅 매트릭스 (학생 시점)

| 링크 | 유래 | 결과 |
|---|---|---|
| `/guide` | AppSidebar | `<StudentGuide />` 렌더 |
| `/guide?tab=docs` | 투어 종료 시 자동 복귀 | Docs 탭 활성 |
| `/guide?tab=overview` (기본) | 최초 진입 | Overview 탭 활성 |
| `/student-home?tour=student-dash` | 「현장 체험」 클릭 | 대시보드 진입 + 투어 시작 |
| `/songs?tour=student-songs` | 「현장 체험」 | 노래 아카이브 투어 |
| `/student-home?tour=course-join` | 「현장 체험」 | 대시보드에서 반 입장 안내 |
| `/student-home?tour=course-room#my-courses` | 「현장 체험」 | 대시보드로 이동 후 `#my-courses` 앵커 스크롤 |
| `/student-home?tour=student-materials#my-courses` | 「현장 체험」 | 자료 안내 |
| `/settings?tour=student-settings` | 「현장 체험」 | 설정 3 탭 안내 |
| `/trash?tour=student-trash` | 「현장 체험」 | 학생 휴지통 3 스텝 |

### 4.3 접근성

- Tabs, Accordion, Slider, Button 모두 shadcn/ui 기반이라 키보드/스크린리더 지원.
- Overview 슬라이더는 aria-label 없음(장식적) — 필요 시 `Slider` 의 `aria-label="슬라이드 위치"` 추가.
- driver.js 팝오버는 라이브러리 기본 접근성 사용, 스텝 텍스트는 완전한 한국어 문장.

### 4.4 Non-Goals

- **알림 설정 안내 없음** — 알림 탭은 학생/교사 설정에서 이미 제거됨.
- **교사 전용 모듈 노출 금지** — 강의안·과정 편집·학생 관리·AI 강의안 생성 등은 학생 가이드에서 언급하지 않는다.
- 다국어(zh) 문자열은 슬라이드 데이터에는 존재하지만 렌더는 ko 만 사용한다.

### 4.5 관련 문서

- **T9** — 교사 가이드 재현. 공용 `ModuleCard`, `ModuleCarousel`, `useGuideTour`, `DemoTransitionSlide` 사양 참조.
- **S1** — 내 공간(내 데스크) 원본 화면. `[data-tour='dash-edit']` anchor 는 S1 에서 정의.
- **P3a~P3q** — 노래 아카이브 · 분석 다이얼로그 · 탐구. `[data-tour='songs-filter']`, `[data-tour='song-card']` anchor.
- **S2** — 반 입장(초대 링크·프로필 온보딩·아바타 12 종).
- **S4** — 우리 반 5 탭 학생 시점.
- **S6** — 학생 설정·휴지통. `[data-tour='settings-tab-profile']`, `[data-tour='settings-tab-security']` 및 학생 휴지통 2 탭 구조가 이 문서의 6·7 번 모듈 근거.

## 5. Acceptance

- [ ] 학생 계정으로 `/guide` 진입 시 `StudentGuide` 만 렌더되고 교사 가이드는 노출되지 않는다.
- [ ] Hero 카드 하단 카운트가 `총 7개 모듈 · 27개 세부 기능` 을 정확히 표시한다 (5+5+4+5+4+3+3 = 29 가 아니라 실제 features 합계를 자동 계산).
- [ ] Overview 탭 진입 시 자동 재생이 시작되고, Docs 탭으로 이동하면 타이머가 즉시 정지한다.
- [ ] 14 개 슬라이드가 순서대로 재생되며, transition 3.5 초 / 콘텐츠 7 초 / intro·outro 6 초 지속.
- [ ] 각 `ModuleCard` 의 「🚀 현장 체험」 버튼이 정확한 `?tour=<id>` 쿼리와 `#hash` 를 붙여 이동한다.
- [ ] driver.js 투어가 anchor 부재 시 5 초 폴링 후 조용히 스킵되며 마지막 단계에서만 「지금 시작 / 사용 가이드로」 두 버튼이 렌더된다.
- [ ] 학생 설정 투어에 「알림」 스텝이 존재하지 않는다.
- [ ] 학생 휴지통 투어(`student-trash`) 가 정상 실행되며 3 스텝(2 탭 안내 → 복원/영구 삭제 → 7 일 자동 정리)으로 종료된다.
- [ ] Admin 계정으로 `/guide?as=student` 접속 시에도 동일한 학생 가이드가 노출된다.
- [ ] Docs 탭에서 캐러셀의 모듈을 클릭하면 해당 `#mod-{id}` 앵커로 부드럽게 스크롤된다.
- [ ] 화면 폭 < 640px 에서 Hero·Tabs·SlideMeta 가 세로 스택으로 정상 표시된다.
