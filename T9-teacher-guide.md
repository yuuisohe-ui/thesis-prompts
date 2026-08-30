# T9 · 교사 사용 가이드 (Teacher Usage Guide)

> 사이드바 「사용 가이드」 → 교사 계정일 때 렌더링되는 `/guide` 화면.
> `Guide.tsx` 가 역할에 따라 `TeacherGuide` / `StudentGuide` 로 분기하며,
> 본 문서는 그중 **교사용 가이드 전체(페이지 셸 + 8 개 모듈 콘텐츠 + driver.js 투어 + Overview 스토리보드)** 를 한 번에 재현하기 위한 프롬프트다.

---

## 1. Identity

당신은 시니어 React/TypeScript 개발자다. 다음을 만든다:

- **역할 라우팅 페이지** `src/pages/Guide.tsx`
- **교사 가이드 셸** `src/pages/TeacherGuide.tsx` (Hero + 2-Tab)
- **모듈 콘텐츠 데이터** `src/components/guide/teacher/content.ts` (7 개 모듈)
- **driver.js 투어 스텝** `src/components/guide/teacher/tourSteps.ts`
- **전체 둘러보기 스토리보드** `src/components/guide/teacher/demo/DemoOverview.tsx` + `data.ts` + 4 개 preview 컴포넌트
- **공용 카드/캐러셀** `src/components/guide/ModuleCard.tsx`, `ModuleCarousel.tsx`
- **투어 실행 훅** `src/components/guide/useGuideTour.ts` (모든 페이지에서 `?tour=<key>` 감지)

모든 UI 는 한국어. 코드 주석은 한국어/중국어 자유. 스타일은 shadcn/ui + Tailwind, 다크 네이비 톤(#1b2641/#233057/#2e3d6b) 은 사용하지 않고 프로젝트의 `--primary`/`--accent` 세맨틱 토큰을 그대로 쓴다.

---

## 2. Instructions

### 2.1 라우팅

- `src/App.tsx` 에서 `/guide` 를 `RequireAuth` 로 감싼 `Guide` 로 라우팅.
- `Guide.tsx`
  - `useAuth()` 에서 `isTeacher`, `isAdmin`, `loading` 를 읽는다.
  - `loading` 이면 `로딩 중...` 텍스트만.
  - `useSearchParams()` 로 `?as=teacher|student` 를 지원 → **관리자 한정**으로 강제 미리보기.
  - 최종 렌더링: `isTeacher ? <TeacherGuide /> : <StudentGuide />`.

### 2.2 `TeacherGuide.tsx` 셸

```
┌─ 최대 폭 max-w-6xl, px-2 sm:px-6, py-4 sm:py-8, space-y-4 sm:space-y-6 ─┐
│  Hero Card (gradient primary/10 → background → accent/10)                 │
│    · 16x16 rounded-2xl bg-primary/15, <BookOpenCheck>                     │
│    · h1 「교사 사용 가이드」                                              │
│    · p 「교사가 사용하는 모든 기능을 한 곳에. …자동 둘러보기 · 설명서…」  │
│    · 「총 N 개 모듈 · M 개 세부 기능」 (Sparkles 아이콘)                   │
├─ Tabs value={tab} onValueChange={setTab} defaultValue = ?tab or overview ┤
│    ├ 🎬 전체 둘러보기 (PlayCircle)  → <TeacherDemoOverview active={…}/>   │
│    └ 📖 설명서       (BookOpen)    → ModuleCarousel + ModuleCard grid    │
├─ 「더 궁금한 점이 있으면 설정 → 문의로 알려주세요.」 (푸터)               │
└──────────────────────────────────────────────────────────────────────────┘
```

- 초기 `tab` 값은 `useSearchParams().get("tab") ?? "overview"`.
- 「설명서」 탭 안: `<ModuleCarousel modules={TEACHER_GUIDE_MODULES} activeId onSelect>` + 모든 모듈을 `id="mod-<id>"` 앵커로 세로 나열.

### 2.3 모듈 콘텐츠 (`teacher/content.ts`)

`GuideModule` 타입(`src/components/guide/content.ts`) 을 그대로 사용. 각 모듈: `id, icon(lucide 문자열), accent(Tailwind gradient), route(진입 경로 + 선택적 #hash), ko:{title,subtitle}, features:[{id, ko:{name,usage,output}, targetSelector?}]`.

교사 가이드는 정확히 **7 개 모듈**을 export 한다 (Materials / Activity 은 임시 숨김 상태 유지):

| # | id              | route                        | icon           | title           | 특징 개수 |
| - | --------------- | ---------------------------- | -------------- | --------------- | -------- |
| 1 | dashboard       | `/dashboard`                 | LayoutDashboard| 대시보드        | 8        |
| 2 | songs           | `/songs`                     | Music          | 노래 아카이브   | 10       |
| 3 | song-generator  | `/songs`                     | Sparkles       | AI 맞춤 노래 생성 | 4      |
| 4 | lessons         | `/workspace#lessons-section` | CalendarDays   | 강의안 제작     | 6        |
| 5 | courses         | `/workspace#class-section`   | BookOpen       | 과정 (우리 반)  | 8        |
| 6 | students        | `/workspace#student-section` | Users          | 학생 관리       | 4        |
| 7 | settings        | `/settings`                  | Settings       | 설정            | 4        |

각 feature 의 문구는 아래 「4. Examples · 4.1」의 원문을 그대로 사용한다. 문구 임의 요약 금지.

### 2.4 driver.js 투어 (`teacher/tourSteps.ts`)

- `driver.js@1.x` 사용, `driver.js/dist/driver.css` 는 `useGuideTour.ts` 에서 import.
- 헬퍼 3 개 로컬 정의:
  - `modal(title, description)` → 앵커 없는 popover.
  - `at(selector, title, description)` → DOM 앵커 필수.
  - `final(title, description, selector?)` → 마지막 스텝 표시용 `__final: true` 마커 부착.
- `TEACHER_TOUR_STEPS: Record<string, DriveStep[]>` 로 export.
- 총 6 개 key 를 정의: `dashboard / songs / song-generator / lessons / courses / students / settings` (7 개). 각 스텝 문구는 「4. Examples · 4.2」참조.
- 각 시퀀스의 마지막 스텝은 반드시 `final(...)` 로 만들고 실제 페이지의 CTA 버튼을 앵커로 지정한다 (예: `[data-tour='songs-add-url']`).
- `TOUR_STEPS` merge: `src/components/guide/tourSteps.ts` 가 `{ ...LEGACY_STEPS, ...TEACHER_TOUR_STEPS, ...STUDENT_TOUR_STEPS }` 로 병합한다 (교사·학생이 legacy 를 우선 덮어씀).

### 2.5 투어 훅 (`useGuideTour.ts`)

- 페이지 컴포넌트가 `useGuideTour()` 를 호출.
- `useLocation` + `URLSearchParams` 로 `?tour=<key>` 감지.
- `TOUR_STEPS[key]` 에서 스텝을 꺼내되, `element` 셀렉터가 지정된 스텝은 실제 DOM 이 없으면 필터링(자동 스킵).
- **anchor 폴링**: anchor 하나라도 발견될 때까지 200 ms × 최대 25 회, anchor 가 아예 없는 modal-only 투어는 300 ms 후 즉시 시작.
- `driver({ showProgress:true, allowClose:true, nextBtnText:"다음", prevBtnText:"이전", doneBtnText:"완료 · 가이드로 돌아가기" })`.
- 마지막 스텝(`__final`) 은 popover 를 다음처럼 덮어쓴다:
  ```ts
  showButtons: ["next", "previous", "close"],
  nextBtnText: "🚀 지금 시작",   // stayOnPage=true 로 표시 후 destroy
  prevBtnText: "← 사용 가이드로", // destroy 만 (onDestroyed 에서 navigate)
  ```
- `onDestroyed`: `stayOnPage` 가 아니면 `/guide?tab=docs` 로 `replace` 이동.
- pathname 이 바뀌면 정리(cleanup).

### 2.6 카드 컴포넌트

**`ModuleCard.tsx`** — 아코디언식 feature 목록.
- 헤더: `bg-gradient-to-br <accent>` + 아이콘 박스 + `title/subtitle` + 우측에 `route` 가 있으면 **「🚀 현장 체험」 버튼** 노출. 클릭 시 `navigate(\`${path}?tour=${module.id}${hash ? \`#${hash}\` : ""}\`)`.
- 본문: `Accordion type="multiple"` 로 features 를 세로 나열. 각 아이템 열림 상태에서 `sm:grid-cols-2` 로 「어떻게 사용 / 결과」 2 열 표시. 라벨 상수 `LABELS = { usage:"어떻게 사용", output:"결과", tour:"🚀 현장 체험" }`.

**`ModuleCarousel.tsx`** — 상단 스티키 칩 캐러셀. 각 칩 클릭 → `onSelect(id)` + `#mod-<id>` 로 스무스 스크롤.

### 2.7 전체 둘러보기 (`teacher/demo/DemoOverview.tsx`)

- **Embla Carousel** (`embla-carousel-react`) — `loop:false, align:"start", duration:25`.
- Auto-advance: `active && playing` 일 때 `TEACHER_SLIDE_META[current].ms` (기본 7000 ms) 후 자동 다음. 끝이면 처음으로.
- 하단 컨트롤 카드: `Pause/Play, RotateCcw(restart), Prev, Next, Slider(0..total-1), "{current+1}/{total}"`.
- 슬라이드 종류 8 종 (`kind`): `intro / dashboard / songs / song_analysis / lessons / course_home / students / transition / outro`.
- 각 `kind` 별로 좌측 300 px 메타 영역(`SlideMeta`) + 우측 preview 컴포넌트를 렌더:
  - `intro` → `IntroSnapshot` (2×2 카테고리 그리드).
  - `dashboard` → `DemoDashboardPreview`.
  - `songs` → `DemoSongsPreview` (공용).
  - `song_analysis` → `DemoSongAnalysisPreview`.
  - `lessons` → `DemoLessonsPreview` (공용).
  - `course_home` → `DemoCourseHomePreview`.
  - `students` → `DemoStudentsPreview` (공용).
  - `transition` → `DemoTransitionSlide` (제목 카드).
  - `outro` → `OutroSnapshot` (3-step + 「지금 시작하기」 → `/songs`).
- `TEACHER_DEMO_SLIDES` 배열 및 `TEACHER_SLIDE_META` 사전 문구는 「4. Examples · 4.3」에서 그대로 사용한다.

### 2.8 학생 가이드와의 관계

- 본 문서는 **교사 가이드만** 재현한다. `StudentGuide.tsx`, `student/content.ts`, `student/tourSteps.ts`, `student/demo/*` 는 별도 문서에서 다룬다.
- 두 가이드가 공유하는 자산: `ModuleCard`, `ModuleCarousel`, `useGuideTour`, `demo/DemoTransitionSlide`, `demo/DemoSongsPreview`, `demo/DemoLessonsPreview`, `demo/DemoStudentsPreview`, `demo/data.ts`.

### 2.9 이 문서에서 하지 않는 것 (Non-Goals)

- Materials(자료), Activity(최근 활동), 강의안 즐겨찾기 모듈 — **현재 UI 에서 노출되지 않음**. 절대로 추가하지 말 것.
- driver.js 로 실제 클릭/폼 자동 채움 실행. 모든 스텝은 **popover 안내만**.
- 다국어(중문) 렌더링. `ko` 필드만 사용.
- 학생 계정 로직.

---

## 3. Context

- **인접 파일**: `src/hooks/useAuth.tsx` (isTeacher, isAdmin), `src/components/AppLayout.tsx` (SidebarProvider 컨텍스트), `src/components/AppSidebar.tsx` (`사용 가이드` 항목이 `/guide` 로 이동).
- **투어 앵커 규약**: 각 페이지가 `data-tour="…"` 속성을 실제 요소에 붙여둔다. 대표 예:
  - Dashboard: `dash-edit`, `dash-inspiration`.
  - Songs: `songs-add-url`, `songs-search`, `songs-filter`, `song-card`, `songs-ai-promo`, `songs-ai-generate`.
  - Lessons: `lessons-list`, `lessons-create`.
  - Courses: `courses-create`.
  - Students: `students-csv`.
  - Settings: `settings-tab-profile / -class / -notify / -security`.
- **legacy 스텝**: `src/components/guide/tourSteps.ts` 안 `LEGACY_STEPS` 는 최소 셋만 유지(호환용). 교사/학생 키가 있으면 그쪽이 이긴다.
- **의존 패키지**: `driver.js`, `embla-carousel-react`, `lucide-react`, `react-router-dom`, shadcn `Card/Button/Slider/Tabs/Accordion`.
- **경로 alias**: `@/components/...`, `@/hooks/...`, `@/pages/...`.

---

## 4. Examples

### 4.1 `teacher/content.ts` (완전한 데이터)

```ts
import type { GuideModule } from "../content";

export const TEACHER_GUIDE_MODULES: GuideModule[] = [
  // 1. Dashboard
  {
    id: "dashboard", route: "/dashboard", icon: "LayoutDashboard",
    accent: "from-blue-500/20 to-indigo-500/20",
    ko: { title: "대시보드", subtitle: "위젯 편집 가능한 나만의 데스크" },
    features: [
      { id: "dash-edit", targetSelector: "[data-tour='dash-edit']",
        ko: { name: "위젯 편집 모드", usage: "우상단「위젯 편집」클릭 → 드래그/리사이즈/삭제 가능, 「완료」로 저장", output: "맞춤형 대시보드" } },
      { id: "dash-catalog",
        ko: { name: "위젯 카탈로그", usage: "편집 모드 → 「위젯 추가」드로어 → 공통/장식/재미/교사/학생 카테고리에서 선택", output: "S/M/L 사이즈 자유 선택" } },
      { id: "dash-teacher",
        ko: { name: "교사 전용 위젯", usage: "AI 초안 · 통계 · 내 반 · 최근 활동 · 학생 메시지 · 공지 · Hero 인사이트 · 플랫폼 흐름", output: "수업 운영 허브" } },
      { id: "dash-deco",
        ko: { name: "장식 위젯", usage: "파티클·오로라·음표·별자리·파문·뮤직 비주얼·아트 텍스트 등 배경 효과", output: "감성적인 대시보드" } },
      { id: "dash-fun",
        ko: { name: "재미 위젯", usage: "오늘의 운세·타로·포춘쿠키·가챠·날씨·동기부여", output: "쉬는 시간의 작은 즐거움" } },
      { id: "dash-calendar",
        ko: { name: "캘린더 위젯", usage: "날짜 클릭으로 메모 추가·확인", output: "월별 일정 뷰" } },
      { id: "dash-hero",
        ko: { name: "Hero 인사이트", usage: "AI 가 매일 언어·음악 지식 한 컷 + 추천 곡 생성", output: "오늘의 교학 영감" } },
      { id: "dash-save",
        ko: { name: "자동 저장", usage: "변경 즉시 저장; 「기본값」으로 초기화 가능", output: "다음 방문에 그대로 유지" } },
    ],
  },
  // 2. Songs
  {
    id: "songs", route: "/songs", icon: "Music",
    accent: "from-pink-500/20 to-rose-500/20",
    ko: { title: "노래 아카이브", subtitle: "수집·분석·탐구·공유" },
    features: [
      { id: "songs-filter", targetSelector: "[data-tour='songs-filter']",
        ko: { name: "필터·검색", usage: "언어·레벨·교학포인트·테마·즐겨찾기 필터; 검색·정렬", output: "100/페이지 페이징" } },
      { id: "songs-add-url", targetSelector: "[data-tour='songs-add-url']",
        ko: { name: "URL로 추가", usage: "링크 붙여넣기 → 언어 선택 → 자막·AI 분석 자동", output: "SongCard 자동 생성" } },
      { id: "songs-search", targetSelector: "[data-tour='songs-search']",
        ko: { name: "YouTube 검색", usage: "검색 다이얼로그 → 키워드 → 결과 선택 → 분석 확인", output: "URL 복사 불필요" } },
      { id: "songs-card", targetSelector: "[data-tour='song-card']",
        ko: { name: "SongCard 액션", usage: "♥ 즐찾 / ✏ 편집 / ↻ 재분석 / 🗑 삭제 / 🔗 공유 / 👁 분석", output: "곡 라이프사이클" } },
      { id: "songs-analysis-tabs",
        ko: { name: "분석 다이얼로그 (6 탭)", usage: "영상·가사·단어·문법·읽기·탐구 6개 탭", output: "완전한 학습 자료" } },
      { id: "songs-explore",
        ko: { name: "🔎 탐구 — 4 모듈", usage: "곡 정보 · 가사 심화 · 연습 · 수업 도구 (각 4 항목)", output: "16편 심층 콘텐츠" } },
      { id: "songs-status",
        ko: { name: "실시간 분석 상태", usage: "추가 직후 펄스 애니메이션; 완료 시 자동 갱신 (현재 세션 한정)", output: "백그라운드 진행 가시화" } },
      { id: "songs-csv",
        ko: { name: "CSV 일괄 등록", usage: "youtube_url/title/artist CSV 업로드", output: "한꺼번에 등록" } },
      { id: "songs-favorite",
        ko: { name: "즐겨찾기·My Songs", usage: "♥ 표시; 공개 아카이브와 RLS 분리", output: "개인 곡 모음" } },
      { id: "songs-share",
        ko: { name: "공유·임베드", usage: "🔗 → 링크 복사 또는 iframe 임베드 (/shared, /embed)", output: "외부 사이트 공유" } },
    ],
  },
  // 3. Song Generator
  {
    id: "song-generator", route: "/songs", icon: "Sparkles",
    accent: "from-purple-500/20 to-fuchsia-500/20",
    ko: { title: "AI 맞춤 노래 생성", subtitle: "노래 아카이브 내 다이얼로그에서 한 번에" },
    features: [
      { id: "sg-1-entry", targetSelector: "[data-tour='songs-ai-generate']",
        ko: { name: "① 진입 — 노래 아카이브에서", usage: "「노래 아카이브」 상단 「AI 맞춤 노래 생성 →」 버튼 클릭", output: "AI 생성 다이얼로그 오픈" } },
      { id: "sg-2-lyrics",
        ko: { name: "② 참고곡 + 가사 생성", usage: "YouTube 참고 URL · 주제 · 언어 · 난이도 입력 → GPT 가 가사 자동 생성 후 편집", output: "편집 가능한 가사" } },
      { id: "sg-3-compose",
        ko: { name: "③ Suno V4.5 작곡", usage: "비동기 호출 + 5초 폴링, 1~3분 소요, 새로고침에도 복구", output: "실재생 MP3" } },
      { id: "sg-4-video-save",
        ko: { name: "④ 영상 매칭·아카이브 저장", usage: "video-trigger(GitHub Actions) 가 Pixabay 영상 매칭 → 「저장」 → 아카이브 등록 + 분석 자동", output: "신규 SongCard" } },
    ],
  },
  // 4. Lessons
  {
    id: "lessons", route: "/workspace#lessons-section", icon: "CalendarDays",
    accent: "from-emerald-500/20 to-teal-500/20",
    ko: { title: "강의안 제작", subtitle: "AI로 15주 syllabus 자동 생성 (작업실 · 내 강의안)" },
    features: [
      { id: "lp-list", targetSelector: "[data-tour='lessons-list']",
        ko: { name: "강의안 목록", usage: "카드 그리드·드래그 정렬; 🟡 초안 / 🔵 정식 / ⭐ 즐겨찾기", output: "전체 강의안 한눈에" } },
      { id: "lp-create", targetSelector: "[data-tour='lessons-create']",
        ko: { name: "AI 생성", usage: "제목·레벨·주차·시간·주당 곡수·시험 주차 설정", output: "W1 OT / W8 중간 / W15 기말 15주" } },
      { id: "lp-week",
        ko: { name: "주차 상세", usage: "주차 클릭 → AI 상세 계획 자동 생성", output: "실행 가능한 주차 계획" } },
      { id: "lp-modules",
        ko: { name: "주차 내장 모듈", usage: "아티스트·문화·수사·Pixabay·AI 채팅", output: "풍부한 멀티미디어 주차 페이지" } },
      { id: "lp-song-pick",
        ko: { name: "곡은 아카이브에서만", usage: "주차 SongCard는 분석된 곡 중에서 선택", output: "강의안·아카이브 연동" } },
      { id: "lp-bookmark",
        ko: { name: "주차 즐겨찾기", usage: "여러 강의안의 특정 주차를 즐겨찾기로 모음", output: "재사용 주차 컬렉션" } },
    ],
  },
  // 5. Courses
  {
    id: "courses", route: "/workspace#class-section", icon: "BookOpen",
    accent: "from-cyan-500/20 to-sky-500/20",
    ko: { title: "과정 (우리 반)", subtitle: "수업 운영 허브 · 5개 탭 (작업실 · 내 반)" },
    features: [
      { id: "co-create", targetSelector: "[data-tour='courses-create']",
        ko: { name: "새 반 만들기", usage: "이름·레벨·학기·학생수·강의안 연결 → AI 가 소개·목표·커리큘럼 자동", output: "초대 링크 포함" } },
      { id: "co-home",
        ko: { name: "탭 A · 홈", usage: "편집 모드 → 5종 AI 블록 (과정 소개·주차 한눈에·학생 동향·교사 메시지·자료 요약) 자동 생성", output: "맞춤형 과정 홈" } },
      { id: "co-calendar",
        ko: { name: "탭 B · 캘린더", usage: "반 전용 일정", output: "월간 캘린더" } },
      { id: "co-materials",
        ko: { name: "탭 C · 강의 자료 (3 서브탭)", usage: "강의안 (주차 아코디언·잠금/공개) · 노래 (아카이브·YouTube 카드 추가) · 자료 (PDF/Word/Excel/PPT/HWP/이미지/오디오/비디오 ≤20MB · 외부 링크 · 메모 · YouTube 임베드). 학생은 내용이 있는 서브탭만 표시", output: "강의 자료 허브" } },
      { id: "co-notice",
        ko: { name: "탭 D · 알림", usage: "교사 공지 (AI 다듬기) + 학생 Q&A; 학생도 공지 확장·답글; 실시간 동기화", output: "양방향 소통판" } },
      { id: "co-community",
        ko: { name: "탭 E · 우리 반", usage: "초대 링크 복사 → 학생 자동 등록; 160px 카드 그리드", output: "학생 멤버 관리" } },
      { id: "co-onboard",
        ko: { name: "학생 자동 입장", usage: "프로필 정보 자동 채움 → 확인·수정 → 기본 아바타 12종 또는 직접 업로드", output: "재입력 불필요" } },
      { id: "co-comments",
        ko: { name: "AI 다듬기·답글", usage: "공지/Q&A 작성 시 AI 다듬기; 답글 가능", output: "원활한 소통" } },
    ],
  },
  // 6. Students
  {
    id: "students", route: "/workspace#student-section", icon: "Users",
    accent: "from-amber-500/20 to-orange-500/20",
    ko: { title: "학생 관리", subtitle: "프로필·아바타·온보딩 (작업실 · 학생 관리)" },
    features: [
      { id: "stu-view",
        ko: { name: "카드/테이블 전환", usage: "우상단 토글; 이름·학번·학과·과정 검색", output: "유연한 뷰" } },
      { id: "stu-csv", targetSelector: "[data-tour='students-csv']",
        ko: { name: "CSV 일괄 등록", usage: "이름·학번·학과·성별·레벨 CSV 업로드", output: "여러 학생 동시 등록" } },
      { id: "stu-avatar",
        ko: { name: "학생 아바타", usage: "12종 기본 아바타 또는 직접 업로드 (이모지 대체)", output: "실제감 있는 학생 카드" } },
      { id: "stu-onboard",
        ko: { name: "셀프 온보딩", usage: "초대 링크 → 정보 자동 입력 → 확인 → 입장", output: "학생 자가 가입" } },
    ],
  },
  // 7. Settings
  {
    id: "settings", route: "/settings", icon: "Settings",
    accent: "from-slate-500/20 to-zinc-500/20",
    ko: { title: "설정", subtitle: "프로필·수업 환경·알림·보안" },
    features: [
      { id: "set-profile",
        ko: { name: "프로필", usage: "이름·아바타·자기소개·소속을 편집. 학생은 첫 로그인 시 입력한 정보를 그대로 표시·수정", output: "최신 프로필" } },
      { id: "set-class",
        ko: { name: "수업 환경 (교사 전용)", usage: "기본 언어·레벨·테마 등 수업 기본값. 학생 화면에는 노출되지 않음", output: "교사 작업 환경" } },
      { id: "set-notify",
        ko: { name: "알림 (준비 중)", usage: "알림 옵션 UI 노출. 실제 발송 기능은 추후 제공", output: "UI 미리보기" } },
      { id: "set-security",
        ko: { name: "보안", usage: "비밀번호 변경·계정 삭제. 비밀번호 변경은 현재 사용 가능", output: "계정 보호" } },
    ],
  },
];
```

### 4.2 `teacher/tourSteps.ts` (7 개 시퀀스 문구 원문)

각 시퀀스의 스텝 개수와 마지막 `final` 앵커가 아래와 같아야 한다.

**dashboard (5 step)**
1. `at([data-tour='dash-edit'], "🧩 위젯 편집 시작", "이 버튼을 누르면 편집 모드가 켜져요. 카드를 드래그·리사이즈·삭제할 수 있고, 끝나면 같은 버튼이 「완료」로 바뀝니다.")`
2. `modal("➕ 위젯 추가 (40+)", "편집 모드에서 「위젯 추가」 드로어가 열립니다. 공통 / 장식 / 재미 / 교사 / 학생 카테고리에서 S·M·L 사이즈를 골라 배치하세요.")`
3. `modal("👩‍🏫 교사 전용 위젯", "AI 초안 · 통계 · 내 반 · 최근 활동 · 학생 메시지 · 공지 · Hero 인사이트 · 플랫폼 흐름 — 수업 운영 허브를 구성합니다.")`
4. `modal("💾 자동 저장", "레이아웃은 변경 즉시 저장되며, 「기본값」으로 되돌릴 수 있어요.")`
5. `final("✅ 완성!", "이제 본인만의 대시보드를 직접 꾸며볼 차례예요. 「지금 시작」을 누르면 편집 모드로 바로 진입할 수 있어요.", "[data-tour='dash-edit']")`

**songs (7 step)** — anchors: `songs-add-url`, `songs-search`, `song-card`, modal×3, `final → [data-tour='songs-add-url']`. 원문:
```
① URL 로 곡 추가 / 여기에 YouTube 링크를 붙여넣고 언어를 선택하면, 자막 추출과 AI 분석이 자동으로 진행됩니다. 약 30초 ~ 1분 소요.
② YouTube 키워드 검색 / URL 이 없어도 키워드로 곡을 찾을 수 있어요. 결과 카드 클릭 → 언어 선택 → 자동 분석.
③ SongCard 액션 / 분석이 끝나면 이런 카드가 생겨요. ♥ 즐겨찾기 / ✏ 편집 / ↻ 재분석 / 🗑 삭제 / 🔗 공유 / 👁 분석 열기 — 곡 라이프사이클을 모두 처리합니다.
④ 분석 다이얼로그 — 6 탭 / 카드의 👁 을 누르면 영상 · 가사 · 단어 · 문법 · 읽기 · 탐구 6 개 탭이 열려요. 완전한 학습 자료가 한곳에.
⑤ 탐구 — 4 모듈 / 곡 정보 · 가사 심화 · 연습 · 수업 도구 (각 4 항목, 총 16 편 AI 콘텐츠). 클릭할 때 필요한 만큼만 생성됩니다.
⑥ CSV 일괄 등록 · 공유 / youtube_url/title/artist CSV 로 한꺼번에 등록 가능. 🔗 버튼으로 /shared/* · /embed/* 외부 공유 링크도 만들 수 있어요.
final: 결과: 나만의 노래 아카이브 / 수집 → 분석 → 탐구 → 공유 — 곡 한 곡당 16 편 이상의 학습 자료가 자동으로 쌓입니다. 지금 한 곡 추가해 볼까요?
```

**song-generator (7 step)** — anchors: `songs-ai-promo`, `songs-ai-generate`, modal×4, `final → [data-tour='songs-ai-generate']`.
```
진입 — 노래 아카이브에서 / 교학에 딱 맞는 곡이 없을 때, 노래 아카이브 상단의 이 영역에서 AI 맞춤 곡을 바로 만들 수 있어요.
① 「AI 맞춤 노래 생성 →」 버튼 / 이 버튼을 누르면 AI 노래 생성 다이얼로그가 열립니다. 별도 페이지 이동 없이 이 화면 위에서 모든 과정이 진행돼요.
② 참고 YouTube + 가사 생성 (≈ 5 초) / 다이얼로그에서 참고용 YouTube URL · 주제 · 언어 · 난이도를 입력하고 「가사 생성」을 누르면 GPT 가 가사를 만들어 줍니다. textarea 에서 자유롭게 편집 가능, 마음에 들지 않으면 재생성하세요.
③ Suno V4.5 작곡 (1~3 분) / 「작곡」 버튼을 누르면 Suno V4.5 가 비동기로 곡을 만들고 5 초마다 진행률을 폴링합니다. 브라우저를 새로고침해도 진행 상태가 그대로 복구되니, 안심하고 다른 작업을 해도 돼요.
④ Pixabay 영상 자동 매칭 / 작곡이 끝나면 GitHub Actions 기반 video-trigger 가 가사·분위기에 어울리는 Pixabay 배경 영상을 자동으로 찾아 매칭합니다. 결과가 마음에 들 때까지 영상만 다시 고를 수도 있어요.
⑤ 아카이브 저장 + 백그라운드 분석 / 「저장」 한 번이면 노래 아카이브에 SongCard 가 등록되고, 단어·문법·탐구 분석까지 백그라운드에서 자동 진행됩니다.
final: 🎁 최종 결과 / 주제 한 줄 → 가사 + MP3 + 배경 영상 + 6 탭 분석까지 완비된 SongCard 한 장. 「지금 시작」을 누르면 다이얼로그를 바로 열 수 있어요.
```

**lessons (6 step)** — anchors: `lessons-list`, `lessons-create`, modal×3, `final → [data-tour='lessons-create']`.
```
📋 내 강의안 영역 / 작업실의 「내 강의안」 섹션이에요. 왼쪽은 정식 강의안, 오른쪽은 AI 초안. AI 초안은 검토 후 「정식으로 확정하기」를 눌러야 반에 연결할 수 있어요.
✨ AI 강의안 생성 / 이 버튼을 누르면 다이얼로그가 열려요. 제목·레벨·주차·시간·주당 곡 수·시험 주차를 입력하면 15 주 syllabus (W1 OT / W8 중간 / W15 기말) 가 1~2 분 안에 자동 생성됩니다.
🟡 초안 → 🔵 정식 확정 / 생성된 강의안은 오른쪽 「AI 초안」 카드로 들어와요. 내용을 검토하고 「정식으로 확정하기」를 누르면 왼쪽 정식 목록으로 이동하고, 그제서야 반에 연결할 수 있습니다.
📅 주차 클릭 → 상세 계획 / 강의안 카드를 열어 특정 주차를 누르면 목표·활동·과제·평가가 포함된 상세 계획이 AI 로 생성됩니다 (≈ 30 초). 아티스트 소개·문화 배경·수사 분석·Pixabay 이미지·AI 채팅 모듈도 함께 들어가요.
🔗 반에 연결하기 / 정식 강의안 카드의 「반에 연결하기」 버튼으로 기존 반에 연결하거나, 곧바로 새 반을 만들 때 강의안을 선택할 수 있어요.
final: ✅ 결과: 학기 전체 강의안 / 1 분 입력 → 15 주 syllabus + 각 주차 상세 계획 + 멀티미디어 모듈까지. 「지금 시작」을 누르면 바로 만들어 볼 수 있어요.
```

**courses (7 step)** — anchor: `courses-create`, modal×5, `final → [data-tour='courses-create']`.
```
🏫 새 반 만들기 / 작업실 「내 반」 섹션의 이 카드를 누르면 새 반 만들기 다이얼로그가 열려요. 이름·레벨·학기·학생 수·강의안을 입력하면 AI 가 소개·목표·커리큘럼을 자동 생성하고 학생 초대 링크까지 만들어줍니다 (≈ 30 초).
📑 반 안에는 5 개 탭 / 반을 만들고 입장하면: 홈 · 캘린더 · 강의 자료 · 알림 · 우리 반 — 한 반의 모든 운영을 한곳에서 처리합니다.
🏠 탭 A · 홈 (AI 블록 5 종) / 편집 모드에서 과정 소개 · 주차 한눈에 · 학생 동향 · 교사 메시지 · 자료 요약 블록을 자유롭게 조립. 각 블록은 AI 가 내용을 채워줍니다. 설정에서 「수강생에게 프로필 공개」를 켜면 교수 정보 블록도 자동으로 노출돼요.
📚 탭 C · 강의 자료 (3 서브탭) / 강의안 (주차 아코디언·잠금/공개) · 노래 (아카이브·YouTube 카드 추가) · 자료 (PDF/Word/PPT/이미지/오디오/비디오 ≤ 20 MB · 외부 링크 · 메모 · YouTube 임베드). 학생은 내용이 있는 서브탭만 보입니다.
📣 탭 D · 알림 — 양방향 / 교사 공지(AI 다듬기) + 학생 Q&A. 학생도 공지 카드를 펼쳐 답글을 달 수 있고 실시간으로 동기화됩니다.
👥 탭 E · 우리 반 / 「반 링크 복사」 → 학생에게 전달 → 학생이 클릭하면 프로필이 자동으로 채워지고 12 종 기본 아바타 중 선택만으로 입장 완료. 멤버는 카드 그리드로 한눈에 확인.
final: ✅ 결과: 완전한 수업 운영 허브 / 한 반을 만들면 학생 초대부터 자료 배포 · 공지 · 학습 흐름까지 모두 자동화. 「지금 시작」으로 첫 반을 만들어 볼까요?
```

**students (5 step)** — modal×2, anchor `students-csv`, modal, `final → [data-tour='students-csv']`.
```
👥 학생 관리 영역 / 작업실 「학생 관리」 섹션이에요. 「반별 보기」와 「전체 학생」 탭으로 학생을 한눈에 확인할 수 있어요.
📂 반별 아코디언 / 각 반을 펼치면 등록된 학생 목록이 보여요. 「반 링크 복사」 버튼으로 초대 링크를 바로 전달할 수 있습니다.
📥 CSV 일괄 등록 / 이 버튼으로 다이얼로그가 열려요. 이름·학번·학과·성별·레벨이 포함된 CSV 를 업로드 → 미리보기 → 한 번에 입학시킬 수 있어요.
🪪 학생 셀프 온보딩 / 초대 링크를 받은 학생은 클릭 시 회원가입 때 입력한 정보가 자동으로 채워져요. 확인·수정 후 12 종 기본 아바타 또는 직접 사진 업로드만으로 입장 완료.
final: ✅ 결과: 반 전체 학생 데이터 / CSV 일괄 등록 + 초대 링크만으로 입학 → 프로필 자동 → 우리 반 카드 그리드까지. 「지금 시작」으로 첫 학생을 등록해 볼까요?
```

**settings (5 step)** — anchors: `settings-tab-profile / -class / -notify / -security`, `final → [data-tour='settings-tab-profile']`.
```
🪪 프로필 탭 / 이름 · 아바타 · 자기소개 · 소속을 편집합니다. 학생은 첫 로그인 정보가 채워져 있고, 여기서 직접 수정합니다.
🏫 수업 환경 (교사 전용) / 기본 언어 · 레벨 · 테마 등 수업 운영 기본값을 지정합니다. 학생 화면에는 노출되지 않아요.
🔔 알림 (UI 미리보기) / 알림 옵션 UI 가 미리 노출되어 있어요. 실제 발송 기능은 추후 추가 예정.
🔒 계정 & 보안 / 비밀번호 변경과 계정 삭제 등 보안 동작. 비밀번호 변경은 지금도 사용 가능합니다.
final: ✅ 결과: 맞춤형 환경 / 프로필·기본값·보안을 정리해 두면 이후 모든 화면이 본인 설정대로 동작합니다.
```

### 4.3 `teacher/demo/data.ts` (Overview 스토리보드 14 슬라이드)

- 슬라이드 배열 순서: `intro → transition(1) → dashboard → transition(2) → songs → transition(3) → song_analysis → transition(4) → lessons → transition(5) → course_home → transition(6) → students → outro`.
- transition 카드 6 개 문구:
  1. `1. 대시보드에서 출발 · 위젯을 자유롭게 편집`
  2. `2. 곡 채집 · YouTube 한 줄이면 끝`
  3. `3. 곡 깊이 분석 · 6 개 탭으로 모든 학습 자료`
  4. `4. 강의안 만들기 · AI 가 15 주 syllabus 자동 생성`
  5. `5. 반 운영 · 홈을 블록으로 조립`
  6. `6. 학생 초대 · 초대 링크 → 자동 입장`
- `TEACHER_SLIDE_META[idx].ms` 는 콘텐츠 슬라이드 7500 ms, transition 3500 ms, intro/outro 6000 ms.
- `TEACHER_SLIDE_META[idx].ko` 문구 (콘텐츠 슬라이드 기준):
  - 0 `교사 가이드에 오신 걸 환영합니다 / 대시보드부터 학생 관리까지 6 분이면 충분합니다.`
  - 2 `위젯 편집 가능한 대시보드 / AI 초안·통계·내 반·Hero 인사이트 등 교사 전용 위젯을 자유롭게 배치.`
  - 4 `노래 아카이브 / YouTube 링크 하나로 가사·단어·문법을 AI 자동 분석.`
  - 6 `6 탭 심층 학습 / 영상·가사·단어·문법·읽기·탐구. 탐구에는 16편 AI 콘텐츠.`
  - 8 `강의안 제작 / 주제·레벨·주차만 입력하면 W1 오리엔테이션부터 W15 기말까지 자동.`
  - 10 `과정 홈 (블록 편집) / 공지·주차·학생·곡·텍스트·영상 등 20+ 블록을 드래그로 조립.`
  - 12 `학생 초대 & 자동 입장 / 초대 링크 → 학생 정보 자동 채움 → 확인·아바타 선택 → 입장.`
  - 13 `지금 시작해 보세요 / 첫 곡을 추가하는 데 1 분이면 충분합니다.`

---

## 5. Acceptance Criteria

- [ ] `/guide` 는 로그인 상태에서만 접근 가능. 미로그인 시 `/auth?redirect=/guide` 로 이동.
- [ ] 교사 계정: `TeacherGuide` 가, 학생 계정: `StudentGuide` 가 렌더. 관리자 계정에서만 `?as=teacher|student` 로 강제 전환 가능.
- [ ] Hero 카드 하단에 「총 7 개 모듈 · 45 개 세부 기능」 (feature 합계) 이 정확히 표기된다.
- [ ] 탭 초기값은 URL `?tab=` 파라미터를 존중. 기본값 `overview`.
- [ ] 「🎬 전체 둘러보기」 탭: Embla 캐러셀이 자동 재생 (첫 로드 시 `playing=true`), pause/restart/prev/next/slider 로 제어 가능. 각 슬라이드 표시 시간이 `TEACHER_SLIDE_META[idx].ms` 와 일치.
- [ ] 슬라이드는 정확히 14 장. `outro` 의 「지금 시작하기」 버튼이 `/songs` 로 이동.
- [ ] 「📖 설명서」 탭: 상단에 `ModuleCarousel`, 아래에 7 개 `ModuleCard`. 각 카드 헤더 우측의 「🚀 현장 체험」 버튼이 `${route}?tour=${id}${#hash?}` 로 네비게이션.
- [ ] `useGuideTour` 는 대상 페이지에서 anchor 를 200 ms 간격으로 최대 5 초 폴링. anchor 미존재 스텝은 자동 스킵되고, anchor 가 하나라도 있으면 즉시 시작.
- [ ] 각 시퀀스 마지막 popover 는 「🚀 지금 시작」/「← 사용 가이드로」 두 버튼을 노출한다. 「지금 시작」 → 현재 페이지 유지, 「사용 가이드로」/X 닫기 → `/guide?tab=docs` 로 replace.
- [ ] Materials / Activity / Lesson-plan-bookmark 관련 UI 는 어디에도 나타나지 않는다.
- [ ] 다크 모드에서도 gradient/텍스트가 세맨틱 토큰으로 자연스럽게 표시된다 (`text-white` 등 하드코딩 금지).
- [ ] `tsgo` 타입체크와 프로덕션 빌드 모두 통과.
