# T1 · 교사 대시보드 복현 프롬프트

> 논문 4.2.1 대응. `/dashboard` 라우트 (교사 진입 시) 를 처음부터 재구현하기 위한 단일 복현 프롬프트.
> 공용 shell/auth 는 P6-infra, P2-auth 참조. 본 문서는 「대시보드 위젯 시스템 + 교사 전용 8 위젯 + Dashboard 캘린더」 에 집중한다.
>
> ⚠️ 공용(12) · 장식(7) · 재미(6) 위젯의 세부 렌더 로직은 방대하므로 본 문서에서는 **카탈로그 스펙 + 3~5 줄 요약**만 담고, 세부 구현은 부록 T1a/T1b/T1c 로 분리한다. 이 세 파일은 role-agnostic 하므로 학생 대시보드(`/student-home`) 구축 후에도 그대로 재사용 가능하다.

---

## 1. Identity

당신은 시니어 프론트엔드 엔지니어이다. React 18 + Vite + TypeScript + Tailwind + shadcn/ui + `@dnd-kit/core` + `@dnd-kit/sortable` + Supabase(=Lovable Cloud) 조합으로,
「교사용 위젯 대시보드」를 처음부터 만든다. 위젯은 총 44 종, 카테고리 5 개(공용 12 · 장식 7 · 재미 6 · 교사 전용 8 · 학생 전용 11), 사이즈 S/M/L, 드래그로 재배치·삽입·크기 변경·삭제가 가능하며 localStorage 에 200ms 디바운스로 저장된다. 파티클(전체 화면 배경) 위젯은 UX 혼선 이슈로 제거되었으며, 카탈로그·헤더·기본 레이아웃 어디에도 노출하지 않는다.

---

## 2. Instructions

### 2.1 라우팅 / 페이지 컨테이너

- `RequireAuth` 통과 후 `/dashboard` = `src/pages/Dashboard.tsx`.
- 페이지 내용은 다음과 같이 최소로 유지: 배경 `#f4f3f0`, 상하좌우 음수 마진(`-m-3 sm:-m-4 md:-m-6`)+같은 값 패딩으로 AppLayout 내부 여백을 상쇄, `useGuideTour()` 호출, `<DashboardGrid />` 를 렌더.
- 학생 라우트 `/student-home` 에서도 같은 `<DashboardGrid variant="student" storageKey="dashboard_layout_student" defaultLayout={DEFAULT_LAYOUT_STUDENT} title="학생 홈" subtitle="…" />` 형태로 재사용 가능. 본 문서에서는 교사 진입만 요구 스펙.

### 2.2 타입 & 상수 (`src/components/dashboard/types.ts`)

```ts
export type ModuleSize = "S" | "M" | "L";
export type ModuleType =
  | "hero_insight" | "platform_flow" | "calendar" | "song_recommendation"
  | "clock" | "notes" | "image" | "youtube" | "link_card" | "quote"
  | "pomodoro" | "dday" | "wave_deco" | "divider"
  | "ai_draft" | "stats" | "my_classes" | "recent_activity"
  | "student_messages" | "notices"
  | "student_my_courses" | "student_week" | "student_assignments"
  | "aurora_deco" | "floating_notes" | "space_deco"
  | "ripple_interactive" | "music_visualizer" | "art_text"
  | "fortune" | "tarot" | "fortune_cookie" | "gacha" | "weather" | "motivation"
  | "sticky_note"
  | "message_teacher" | "notice_check" | "public_courses"
  | "learning_record" | "my_favorite_songs" | "learning_goal"
  | "daily_plan" | "weekly_plan";

export interface ModuleInstance { id: string; type: ModuleType; size: ModuleSize; config?: Record<string, any>; }
export interface DashboardLayout { version: 1; modules: ModuleInstance[]; }

export const DEFAULT_LAYOUT: DashboardLayout = {
  version: 1,
  modules: [
    { id: "default-hero", type: "hero_insight", size: "L" },
    { id: "default-flow", type: "platform_flow", size: "L" },
    { id: "default-calendar", type: "calendar", size: "M" },
    { id: "default-song", type: "song_recommendation", size: "M" },
  ],
};

export const DEFAULT_LAYOUT_STUDENT: DashboardLayout = {
  version: 1,
  modules: [
    { id: "s-hero", type: "hero_insight", size: "L" },
    { id: "s-flow", type: "platform_flow", size: "L", config: { variant: "student" } },
    { id: "s-mine", type: "student_my_courses", size: "M" },
    { id: "s-public", type: "public_courses", size: "M" },
    { id: "s-song", type: "song_recommendation", size: "M" },
    { id: "s-cal", type: "calendar", size: "M" },
  ],
};
```

### 2.3 레이아웃 훅 (`useDashboardLayout`)

- 시그니처: `useDashboardLayout({ storageKey?, defaultLayout? })` → `{ layout, addModule, removeModule, updateModule, resizeModule, reorder, reset }`.
- storageKey 기본값 `"dashboard_layout"`. 학생 라우트는 `"dashboard_layout_student"` 를 명시적으로 전달.
- 초기 로드: `localStorage.getItem(key)` → JSON 파싱 → `version === 1 && Array.isArray(modules)` 검증 → 실패 시 defaultLayout.
- 저장: 매 `layout` 변경마다 200ms `setTimeout` 디바운스로 `localStorage.setItem`. cleanup 에서 `clearTimeout`.
- 새 인스턴스 id 생성: `crypto.randomUUID()` 우선, 미지원 시 `m_${Date.now()}_${rand36(6)}`.
- `addModule(type, size="M", insertIndex?)` — 인덱스가 유효 범위면 `splice`, 아니면 뒤에 push.
- `updateModule(id, patch)` — `config` 는 얕은 병합 `{ ...m.config, ...patch.config }`.
- `resizeModule`, `reorder(from, to)`, `reset()` (defaultLayout 로 초기화) 는 상태 교체.
- **레거시 청소**: `DashboardGrid` 최초 마운트 시 `layout.modules` 에서 `type === "particle_deco"` 인 인스턴스가 발견되면 즉시 필터해서 `setModules` 로 재저장(이전 버전에서 파티클을 사용했던 사용자의 localStorage 방어).

### 2.4 카탈로그 (`dashboardModuleCatalog.tsx`)

`CatalogItem = { type, label, description, icon: LucideIcon, defaultSize, category, teacherOnly?, studentOnly? }`.

| category | items |
|---|---|
| common | clock, notes, image, youtube, link_card, quote, pomodoro, dday, song_recommendation, calendar, sticky_note, divider (12) |
| deco | aurora_deco, floating_notes, space_deco, ripple_interactive, music_visualizer, art_text, wave_deco(legacy) (7) |
| fun | fortune, tarot, fortune_cookie, gacha, weather, motivation (6) |
| teacher (teacherOnly) | ai_draft, stats, my_classes, recent_activity, student_messages, notices, hero_insight, platform_flow (8) |
| student (studentOnly) | student_my_courses, public_courses, learning_record, message_teacher, my_favorite_songs, learning_goal, daily_plan, weekly_plan, notice_check, student_week, student_assignments (11) |

각 항목의 defaultSize·label·description·icon 은 아래 표를 그대로 사용(발췌; 전체는 `DASHBOARD_CATALOG` 배열로 하드코딩).

- `hero_insight` L / "Hero 인사이트" / "오늘의 언어·음악 지식" / Sparkles / teacher
- `platform_flow` L / "플랫폼 흐름" / "4단계 워크플로우" / Workflow / teacher
- `stats` M / "현황 통계" / "핵심 카운트" / BarChart3 / teacher
- `my_classes` M / "내 반 현황" / "반 + 링크 복사" / Users / teacher
- `recent_activity` S / "최근 활동" / "최근 8개 이벤트" / Lightbulb / teacher
- `student_messages` S / "학생 메시지" / MessageSquare / teacher
- `notices` S / "공지" / Bell / teacher
- `ai_draft` M / "AI 초안 확인" / "AI 생성 강의안" / Sparkles / teacher
- `calendar` M / "캘린더 & 일정" / CalendarDays / common
- `song_recommendation` M / "오늘의 노래 추천" / Music / common
- (나머지는 위 카테고리 표의 아이콘·설명을 유지)

Helper: `getCatalogItem(type)` → `find(c => c.type === type)`.

### 2.5 DashboardGrid (`src/components/dashboard/DashboardGrid.tsx`)

Props: `variant?: "teacher" | "student"`, `storageKey?`, `defaultLayout?`, `title?`, `subtitle?`.

- `useAuth()` 에서 `profile.role` 확인 → `isTeacher = variant === "teacher" && role !== "student"`.
- `editing: boolean` state, `dragging: { kind: "catalog", type } | { kind: "module", id } | null`.
- `sensors = useSensors(useSensor(PointerSensor, { activationConstraint: { distance: 6 } }))`.
- **파티클 특수 처리 없음**: 이전 버전의 fullscreen 파티클 배경은 제거되었다. 모든 모듈은 grid 카드로만 렌더된다.
- 상단 헤더(`z-10` 컨테이너 내부):
  - 좌측: `<h1>` = title || "대시보드", `<p>` = subtitle || "필요한 위젯만 골라 자유롭게 배치하세요.".
  - 우측 그룹(`flex items-center gap-1.5 flex-nowrap shrink-0`):
    1. 편집 중일 때 `기본값` 버튼(`RotateCcw`, `reset()` 호출).
    2. `위젯 편집` 토글 (`data-tour="dash-edit"`). editing 시 `<Check/>완료`, 아니면 `<Pencil/>위젯 편집`.
- 편집 중일 때 컨테이너에 `md:pr-[296px]` 를 붙여 우측 드로어와 겹치지 않게.
- DnD:
  - `onDragStart` — `active.data.current.source === "catalog"` 면 catalog dragging, 아니면 module dragging.
  - `onDragEnd` — catalog 인 경우: `e.over` 가 있으면 그 카드의 인덱스 앞에 삽입, 없으면 push. module 인 경우: `reorder(from, to)`.
  - `DragOverlay` — catalog 이면 아이콘+라벨 미니 카드, module 이면 라벨 카드.
  - `SortableContext items={gridModules.map(m => m.id)} strategy={rectSortingStrategy}`.
  - grid: `<div className="grid grid-cols-12 gap-3.5 auto-rows-min">`.
- 각 카드는 `<DashboardModuleFrame>` 로 감싸고 그 내부에 `<ModuleRenderer>`.
- `<DashboardEditPanel open={editing} onClose={()=>setEditing(false)} onAdd={type => addModule(type, getCatalogItem(type)?.defaultSize || "M")} variant={variant} isTeacher={isTeacher} />`.

### 2.6 DashboardModuleFrame

- `useSortable({ id: instance.id, disabled: !editing })`.
- Size class: `S → col-span-12 md:col-span-4`, `M → col-span-12 md:col-span-6`, `L → col-span-12`.
- 컨테이너 클래스: `relative bg-white rounded-[16px] border overflow-hidden`; dragging 시 `opacity-60`; editing 시 `ring-1 ring-indigo-200`.
- 편집 모드 오버레이:
  - 우상단 `S/M/L` 토글(활성 = `bg-indigo-600 text-white`) + `X` 삭제(→ `onRemove`).
  - 좌상단 `GripVertical` 드래그 핸들 (aria-label="드래그").
  - 자식 컨테이너 상단 여백 `pt-2`.

### 2.7 DashboardEditPanel (우측 드로어)

- `w-[280px]`, `fixed top-0 right-0 h-screen`, transform 슬라이드 인/아웃 300ms.
- 모바일에서만 배경 오버레이(`fixed inset-0 bg-black/20 z-30 md:hidden`).
- 헤더 h-12: 「위젯 추가」 + X.
- 안내: 「드래그해서 원하는 위치에 놓거나, + 를 눌러 끝에 추가하세요.」
- 섹션(순서 · 이모지 · 라벨):
  1. 🧩 공용
  2. ✨ 장식 & 애니메이션
  3. 🔮 재미 & 운세
  4. 👩‍🏫 교사 전용 — `variant !== "student" && isTeacher` 일 때만 노출
  5. 🎓 학생 전용 — `variant === "student" || !isTeacher` 일 때만 노출
  - 학생 variant 에서는 아예 teacher 카테고리를 제외.
- 각 섹션은 아코디언(state `openMap: Record<CatalogCategory, boolean>`). 헤더에 이모지·라벨·개수 뱃지·`ChevronDown`(열림 시 `rotate-180`).
- 각 카탈로그 행:
  - `useDraggable({ id: "catalog:{type}", data: { source: "catalog", type, defaultSize } })`.
  - 좌측 `GripVertical` 드래그 핸들, 아이콘 박스(`bg-indigo-50 text-indigo-600`), 라벨/설명, 우측 `+` 버튼(`onAdd(type)`).
  - Dragging 시 `opacity-40`.

### 2.8 ModuleRenderer

44 개 타입 → 컴포넌트 매핑 switch. `default: return null`. props 로 `{ instance, onConfigChange, editing }` 전달.

`onConfigChange(id, config)` 은 `updateModule(id, { config })` 를 호출하여 `config` 얕은 병합을 수행한다.

### 2.9 교사 전용 8 위젯 — 완전 스펙

모두 공용 껍데기 `_ModuleShell({ title, icon, children })` 을 사용(카드 상단에 아이콘+타이틀, 하단에 콘텐츠). 데이터 조회는 `fetchWithRetry`(exp backoff + AbortController) 로 감싼다.

#### (1) `HeroInsightModule` (기본 L)

- 자체 카드(껍데기 없이) — 어두운 mesh 배경 canvas + 다중 radial gradient 애니메이션.
- 라이트박스 배경: `linear-gradient(135deg,rgba(26,35,64,.88) 0%,…65% 100%)` + 흰 도트 그리드 오버레이.
- 상태
  - `config: HeroConfig | null` = `{ tags: string[], custom: string, frequency: "daily"|"alt"|"weekly" }`.
  - `cache: HeroCache | null` = `{ lastGenerated: "YYYY-MM-DD", content: { title, content, tags[] } }`.
  - `displayed`, `loading`, `popoverOpen`, `fadeKey`(재렌더용 int).
- 초기 로드
  - `supabase.from("profiles").select("hero_config, hero_cache").eq("id", user.id).maybeSingle()`.
  - 실패 시 `localStorage.getItem("hero_config_" + uid)` / `"hero_cache_" + uid` 로 fallback.
- fallback pool `KNOWLEDGE`: 언어학·음악·중한문화·시사·영화·드라마·역사 6 태그, 각 2~4 항목. `hashStr(오늘 날짜)` 로 base 인덱스 결정 → 「다음」 버튼은 `defaultOffset++`.
- 만료 규칙 `shouldRegenerate(freq, lastGenerated)` — daily=1일, alt=2일, weekly=7일. 없으면 항상 재생성.
- 표시 로직
  - `config` 없음 → fallback pool.
  - `config` 있고 캐시 유효 → 캐시 컨텐츠 그대로.
  - `config` 있고 캐시 만료 → 캐시 즉시 표시하되 백그라운드로 API 호출 → 완료 시 페이드 인.
  - `config` 있고 캐시도 없음 → 「오늘의 교학 영감을 준비해드릴게요」 CTA 화면 + `지금 생성하기` 버튼.
- API 호출 `supabase.functions.invoke("generate-hero-insight", { body: { tags, custom } })` → `{title, content, tags[]}`.
  - 파싱 실패/502 → 1회 재시도 → 실패 시 sonner `toast.error("인사이트 생성에 실패했어요. …")`.
  - 429 → 「요청이 너무 잦아요…」, 402 → 「크레딧이 부족합니다…」.
- 저장: `supabase.from("profiles").update({ hero_config, hero_cache }).eq("id", uid)`. 실패 시 localStorage fallback + `toast("오프라인 모드로 저장되었습니다.")`.
- Popover 설정 UI(shadcn Popover, w-320, side="top", align="end")
  - 관심 분야 태그 6개 토글(선택 시 `bg-indigo-600 text-white`).
  - 「직접 입력」 Input(placeholder: "예: 오늘의 영화 추천, 중국어 사자성어...").
  - 「업데이트 빈도」 RadioGroup: 매일 / 격일 / 주 1회.
  - 「마지막 업데이트: YYYY년 M월 D일 / 다음 업데이트: …」 안내.
  - `저장하기` 클릭 시: 태그·custom 둘 다 비어있으면 에러, 아니면 config 저장 후 즉시 API 호출.
- 상단 우측 「다음 인사이트」 버튼(loading 이면 `Loader2 animate-spin`).
- 로딩 중일 때 카드 전체에 shimmer 오버레이 애니메이션 (1.5s infinite).

#### (2) `PlatformFlowModule` (기본 L)

- 노드 4개(교사): 노래 아카이브 / 강의안 제작 / 반 운영 / 학생 초대. `Music/BookText/Users/UserPlus` 아이콘.
- 각 노드 색상 세트(bg/border/text/ripple) — 초록/보라/앰버/바이올렛. `delay` 0/0.6/1.2/1.8s 로 ripple 애니메이션 stagger.
- 노드 사이 커넥터: `#eeeffe` 라인 + `linear-gradient(90deg,transparent,rgba(99,102,241,.8),transparent)` 60% 폭 스팬이 좌→우로 흐르는 flowglow 애니메이션(2s linear infinite, 각 커넥터마다 `${i*0.4}s` delay).
- 카운트 조회
  - songs: `count(*)` from `songs` where `deleted_at is null`.
  - lessons: `lesson_plans` owner_id=me & deleted_at is null.
  - classes: `courses` owner_id=me & deleted_at is null.
  - students: `course_student_profiles` owner_id=me & deleted_at is null.
- 클릭 시 `navigate(path)`. `path` 는 `/songs`, `/workspace#lessons-section`, `/workspace#class-section`, `/workspace#student-section`.
- 학생 variant(`config.variant === "student"`)에서는 3 노드(노래 아카이브 / 공개 수업 찾기 / 반 참여), `s_join` 클릭 시 `JoinCourseDialog` 오픈.

#### (3) `StatsModule` (기본 M)

- 4 개 통계 카드 grid (`grid-cols-2 sm:grid-cols-4`):
  - 노래(from-blue-500 to-cyan-400), 학생, 과정, 강의안.
- 값은 count-up 훅 (600ms) 으로 부드럽게 증가.
- 각 카드 상단 3px 그라디언트 바 + 라벨(10px) + 숫자(24px).
- 쿼리는 위의 PlatformFlow 와 동일.

#### (4) `MyClassesModule` (기본 M)

- `courses` owner_id=me, deleted_at is null, `order created_at desc`, `limit(6)`, select `id, name, share_token`.
- 각 반의 학생 수는 `course_student_profiles` head-only count 를 병렬 호출.
- 각 행: 반 이름 + 「학생 N명」 + 「복사」 버튼(`navigator.clipboard.writeText(`${origin}/c/${share_token}`)` + `toast({ title: "링크 복사됨" })`).
- 반이 없으면 「아직 반이 없습니다.」.

#### (5) `RecentActivityModule` (기본 S)

- 5 소스 병렬 조회(각 limit 5):
  1. `songs` (owner_id=me): `🎵 {title} 추가`, dot blue.
  2. `lesson_plans`: `📋 강의안 {title}`, dot violet.
  3. `courses`: `📚 반 {name}`, dot amber.
  4. `course_student_profiles` join `courses:course_id(name)`: `👤 {full_name} 등록 · {course.name}`, dot emerald.
  5. `materials`: `📎 자료 {title} 업로드`, dot pink.
- 병합 후 `created_at desc` 정렬 → 상위 8 개.
- 시간 포맷: `formatDistanceToNow(new Date(x), { addSuffix: true, locale: ko })`.
- 각 행: 좌측 점 + 텍스트(트렁케이션) + 상대 시간.

#### (6) `StudentMessagesModule` → `StudentMessagesCard`

- 데이터
  - `courses` owner_id=me → 반 id 목록·이름 맵.
  - `course_student_posts` where `course_id in (…)` order desc limit 8 → 상위 5 노출.
  - `course_student_profiles` where `member_user_id in (studentIds)` → 프로필 맵.
- 실시간: `supabase.channel("dash-student-posts").on("postgres_changes", { event: "*", schema: "public", table: "course_student_posts" }, refetch)`.
- 각 행: 아바타(emoji or 이름 첫자, 컬러 4택 rotation) + 이름 + 반 이름 + 타입 뱃지 + Lock/Globe(visibility) + 본문 프리뷰 + 상대시간 + unread(pending) 파란 점.
- 클릭 시 확장 → 본문 전문 + `<RepliesThread parentType="student_post" parentId={m.id} courseId={m.course_id} user isTeacher compact />`.

#### (7) `NoticesModule` → `NoticesCard`

- 데이터: 내 반들의 `course_notices` 최근 5.
- 실시간 postgres_changes 구독(`course_notices`).
- 태그 색상 맵: 공지 / 과제 / 일정.
- 첫 줄만 프리뷰. 확장 시 전문 + `RepliesThread parentType="notice"`.

#### (8) `AIDraftReviewModule` (기본 M)

- `lesson_plans` where `owner_id = me AND plan_type = 'ai_generated' AND deleted_at is null` order desc limit 5.
- 각 초안 행: 제목 + 레벨 뱃지 + `확정` 버튼.
- 확정: `update lesson_plans set plan_type = 'custom' where id`. 성공 시 toast + 로컬 리스트에서 제거.
- 비어있으면 「검토할 AI 초안이 없습니다.」.

### 2.10 공용/장식/재미 위젯 — 요약 (세부는 T1a/T1b/T1c)

각 위젯은 `_ModuleShell` 또는 자체 카드로 렌더. props 는 `{ instance, onConfigChange, editing }` 통일. **세 카테고리는 role-agnostic** 하므로 학생 라우트에서도 그대로 재사용된다.

- **공용(12)** — 세부는 T1a-common-widgets.md
  - `clock` — 매 초 setState, 24h 시:분:초 대형 표시.
  - `notes` — controlled textarea, `instance.config.text` 저장.
  - `image` — 업로드 or Pixabay 검색으로 URL 저장, `object-cover`.
  - `youtube` — URL 입력 후 embed iframe.
  - `link_card` — 타이틀+URL+favicon.
  - `quote` — 명언 pool 랜덤.
  - `pomodoro` — 25분 집중 타이머 + Start/Reset.
  - `dday` — 목표 날짜 입력 → 남은 일 카운트.
  - `song_recommendation` — 오늘의 노래 1곡 AI 추천.
  - `calendar` — `<DashboardCalendar />` (아래 2.11 절 완전 스펙).
  - `sticky_note` — 포스트잇 스타일 controlled 메모, 색상 5택.
  - `divider` — 스타일 5택(solid/dashed/dotted/gradient/wave) 구분선, 전폭.

- **장식(7)** — 세부는 T1b-decor-widgets.md. 모두 캔버스/CSS 애니메이션. 편집 모드에서만 style 변경 가능. 파티클 fullscreen 배경은 제거됨.
  - `aurora_deco` — SVG 필터 오로라 그라데이션.
  - `floating_notes` — 어두운 배경에 음표 위로 떠오름.
  - `space_deco` — 별+유성.
  - `ripple_interactive` — 클릭 시 파문 생성.
  - `music_visualizer` — 40 개 막대 파형 랜덤 애니.
  - `art_text` — 네온/레인보우/글리치 3 스타일 텍스트.
  - `wave_deco` — SVG 파도(레거시 유지).

- **재미(6)** — 세부는 T1c-fun-widgets.md
  - `fortune` — 12별자리 선택 + 한줄 운세 pool.
  - `tarot` — 3장 뽑기(과거·현재·미래) 카드 flip.
  - `fortune_cookie` — 클릭 시 열림 애니 + 랜덤 메시지.
  - `gacha` — 가챠 머신 이미지, Roll 시 랜덤 아이템 이모지.
  - `weather` — 서울 5일 예보(외부 API or 로컬 스텁).
  - `motivation` — 「따뜻한 한마디」 pool 랜덤.

### 2.11 DashboardCalendar (`calendar` 위젯의 실체)

`src/components/dashboard/DashboardCalendar.tsx` 독립 컴포넌트.

- 라이브러리: `date-fns`, `date-fns/locale/ko`.
- 상태: `month`(현재), `selectedDate`, `courses[]`(id/name/class_time/weekdays/color), `events[]`, `customTypes[]`, 그리고 입력 패널 상태(`panelOpen`, `editingId`, `newTitle`, `selType`, `ampm`, `hour`, `minute`, `allDay`, `saving`), 삭제 확인 상태(`confirmDelete: {id, courseId?} | null`).
- 데이터 로드(사용자 진입 시):
  - 교사(variant 기본): `courses` `owner_id = me`, `deleted_at is null`, select `id, name, class_time`.
  - 학생(`variant === "student"`): RPC 또는 `course_student_profiles` join 으로 참여 중인 반 id 집합 → `courses` in-list 조회. 삭제된 반은 자동 제외.
  - `parseClassWeekdays(class_time)` 은 문자열에서 `[일월화수목금토]` 문자를 추출해 요일 index 배열로 변환.
  - 색상은 미리 정의된 4 팔레트(`#2557a7 / #0e7a5a / #b8882a / #6b4faa`) 를 순환 배정.
  - `custom_event_types`: 사용자 정의 타입 목록.
  - `course_calendar_items`: 소속 반들의 이벤트를 `deleted_at is null` 필터로 로드.
  - `localStorage.hidden_calendar_courses` (JSON array) 에 담긴 course_id 는 렌더링에서 제외 → 사용자가 "이 수업 전체 숨기기" 를 선택했을 때 사용.
- 캘린더 그리드
  - `startOfWeek(startOfMonth(month), { weekStartsOn: 0 })` ~ `endOfWeek(endOfMonth(month))`.
  - 요일 헤더: 일 월 화 수 목 금 토.
  - 각 날짜 셀: 오늘(`isSameDay(day, today)` → 파란 링), 같은 요일에 수업이 있으면 반 색상 dot 표시, 이벤트가 있으면 최대 2 개 미리보기 뱃지 + `+N`.
  - 클릭 시 `selectedDate = day` → 우측/하단 상세 리스트에 그 날짜 이벤트 전부 표시.
- 이벤트 CRUD
  - `+` 버튼 → 입력 패널 오픈:
    - 타입 선택: 빌트인 3종(class/memo/task) + 커스텀 타입들. 마지막에 「+ 커스텀 추가」 옵션.
    - 커스텀 추가 서브폼: 라벨 + 색상 12팔레트 스와치 → `custom_event_types.insert`.
    - 종일 스위치.
    - 종일이 아니면: AM/PM 세그먼트 + 시(1~12) + 분(00/15/30/45 4택) 별도 dropdown.
    - 제목 Input.
    - 저장: `course_calendar_items.upsert({ …, owner_id: uid })`.
  - 기존 이벤트 클릭 → 편집 모드(editingId 세팅) → 저장/삭제.
  - **삭제 확인 다이얼로그**: 삭제 클릭 시 shadcn `AlertDialog` 오픈.
    - 옵션 A 「이 일정만 삭제」 → `course_calendar_items.update({ deleted_at: now() })`.
    - 옵션 B 「이 수업 전체 캘린더에서 숨기기」 → `localStorage.hidden_calendar_courses` 에 course_id 추가(교사가 아직 반을 운영 중일 때 시야에서만 제외하는 용도).
- 반의 정규 수업 시간(course.class_time) 자체는 이벤트가 아니지만, 캘린더의 해당 요일 셀에 자동 하이라이트 dot 를 추가한다(`parseClassWeekdays` 결과 사용).
- 실시간 동기화는 없음(패널 저장 후 로컬 state 만 갱신).

### 2.12 학생 전용 위젯 목록(교사 라우트에서는 카탈로그에 노출되지 않음)

`student_my_courses`, `public_courses`, `learning_record`, `message_teacher`, `my_favorite_songs`, `learning_goal`, `daily_plan`, `weekly_plan`, `notice_check`, `student_week`, `student_assignments` (총 11) — 세부 스펙은 학생端 T-문서에서 다룬다. 본 문서에서는 `ModuleType` 열거·`ModuleRenderer` 매핑만 유지.

---

## 3. Examples

### 예시 A — 사용자가 「위젯 편집」 → 「학생 메시지」 카드를 특정 카드 앞에 드롭

1. 우상단 편집 토글 → `editing = true`, 우측 드로어 슬라이드 인.
2. 「교사 전용」 아코디언 확장 → 「학생 메시지」 행 드래그(핸들 `useDraggable(id: "catalog:student_messages")`).
3. 격자 안의 「내 반 현황」 카드 위에서 놓음 → `onDragEnd` 에서 `active.data.source === "catalog"` 이므로 `over.id` 인덱스를 찾아 `addModule("student_messages", "S", idx)`.
4. localStorage `dashboard_layout` 200ms 후 저장.
5. 새로고침 후에도 같은 위치 유지.

### 예시 B — Hero Insight 첫 설정

1. Popover 오픈 → 「음악」 + 「중한문화」 태그 선택, 빈도 「매일」 → 저장.
2. `profiles.hero_config` upsert 후 즉시 `generate-hero-insight` 호출.
3. 응답 `{title:"발라드 호흡", content:"…", tags:["음악"]}` → 카드에 페이드 인, `profiles.hero_cache = { lastGenerated: "2026-07-20", content }`.
4. 다음 날 재방문 → `shouldRegenerate("daily", "2026-07-20") === true` → 캐시 표시 + 백그라운드 재생성.

### 예시 C — 초안 확정

1. `AIDraftReviewModule` 카드에 「W1 오리엔테이션 초안」 등 5개 표시.
2. 「확정」 클릭 → `update lesson_plans set plan_type='custom' where id='…'`.
3. 성공 시 리스트에서 제거 + toast「정식 강의안으로 전환됨」.
4. 이후 `MyLessonsSection`(교사 워크스페이스) 에 정식 강의안으로 표시되어 반 연결 가능.

---

## 4. Context

### 4.1 DB 스키마 요약(관련 컬럼만)

- `profiles(id uuid PK, hero_config jsonb, hero_cache jsonb, role text, …)`.
- `songs(id, title, owner_id, deleted_at, created_at, …)`.
- `lesson_plans(id, title, level, plan_type, owner_id, course_id, deleted_at, created_at)`.
- `courses(id, name, class_time, share_token, owner_id, deleted_at, created_at)`.
- `course_student_profiles(id, course_id, owner_id, member_user_id, full_name, emoji, deleted_at, created_at)`.
- `course_student_posts(id, course_id, student_id, type, content, visibility, status, created_at)`.
- `course_notices(id, course_id, author_id, type, content, created_at)`.
- `materials(id, title, owner_id, created_at)`.
- `course_calendar_items(id, course_id, owner_id, title, description, starts_at, item_type, event_type, all_day, …)`.
- `custom_event_types(id, owner_id, label, color, created_at)`.
- `course_members(id, course_id, user_id, …)` (학생 variant 카운트용).

RLS 는 P6-infra 정의를 따름. Hero 위젯은 `profiles.hero_config/hero_cache` 컬럼에 대해 owner-only update, self-select 정책 필요.

### 4.2 Edge Function 계약

- `generate-hero-insight`
  - Request: `{ tags: string[], custom: string }`.
  - Response: `{ title: string, content: string, tags: string[] }` — JSON 파싱 실패 시 재시도 대상.
  - Error mapping: 429 → 「요청이 너무 잦아요…」, 402 → 「크레딧이 부족합니다…」, no_topic → 조용히 무시, 그 외 → 「인사이트 생성에 실패했어요…」.

### 4.3 사용 라이브러리 / 스타일

- `@dnd-kit/core` + `@dnd-kit/sortable` — DnD.
- `date-fns` + `date-fns/locale/ko` — 상대 시간 · 캘린더.
- `sonner` (Hero 위젯) 및 `@/hooks/use-toast` (그 외).
- `lucide-react` 아이콘.
- shadcn/ui: Popover / RadioGroup / Input / Button / Label.
- 공통 유틸: `fetchWithRetry`(exp backoff + AbortController) 로 모든 Supabase 조회 감쌈.

### 4.4 접근성 & 반응형

- 편집 컨트롤 버튼 모두 `aria-label` 지정(드래그/삭제/기본값).
- PointerSensor 활성화 거리 6px 로 우발적 드래그 방지.
- Grid 는 12-col, 데스크톱에서 S=4col / M=6col / L=12col. 모바일은 전체 12col.
- 편집 드로어는 모바일에서 배경 오버레이 + 슬라이드 인, 데스크톱은 컨테이너 padding-right 로 겹침 방지.

---

## 5. Acceptance

- [ ] `/dashboard` 최초 진입 시 `DEFAULT_LAYOUT` 4 위젯(hero_insight/platform_flow/calendar/song_recommendation) 노출, 배경 `#f4f3f0`.
- [ ] 「위젯 편집」 클릭 → 우측 드로어 슬라이드 인, 각 카드에 드래그 핸들·S/M/L·X 아이콘 노출.
- [ ] 카탈로그 행 드래그 → 카드 위에 드롭 시 해당 인덱스에 삽입, 빈 곳에 드롭 시 뒤에 추가.
- [ ] 카드 재정렬(드래그) 시 `reorder(from, to)` 정확 반영.
- [ ] S/M/L 토글 시 grid col-span 변경.
- [ ] X 삭제 시 카드 제거.
- [ ] 「기본값」 클릭 시 DEFAULT_LAYOUT 로 초기화 후 200ms 이내 localStorage 반영.
- [ ] 새로고침해도 마지막 배치 유지.
- [ ] `ModuleType` 열거·카탈로그·헤더 어디에도 `particle_deco` 가 존재하지 않으며, 이전 버전에서 저장된 파티클 인스턴스는 진입 시 자동 필터링된다.
- [ ] Hero 위젯 — config 미설정 시 fallback pool 순환, 설정 후 edge function 호출·cache 기록·페이드 인, 다음날 재방문 시 만료 감지 후 백그라운드 재생성.
- [ ] Hero 위젯 429/402/parse 실패 각각 정확한 토스트.
- [ ] Stats 4 카운트 정확, 600ms count-up.
- [ ] MyClasses 「복사」 → 클립보드에 `${origin}/c/${share_token}` 저장, toast.
- [ ] RecentActivity 5 소스 병합, ko locale 상대시간.
- [ ] StudentMessages / Notices `postgres_changes` 실시간 반영, 확장 시 `RepliesThread` 표시.
- [ ] AIDraft 「확정」 시 `plan_type` 이 `custom` 으로 업데이트되어 리스트에서 사라짐.
- [ ] Calendar — 반 요일에 자동 dot, 이벤트 생성 시 종일/AM·PM/15분 단위 저장, 커스텀 타입 12색 팔레트 추가 가능, 삭제 시 「이 일정만」/「이 수업 전체 숨기기」 옵션 다이얼로그 노출.
- [ ] `variant="student"` 로 재사용 시 카탈로그에 「교사 전용」 카테고리 미노출, 캘린더는 참여 중인 반만 표시.
- [ ] 모든 데이터 조회는 `fetchWithRetry` 로 감싸져 있으며 AbortController 로 언마운트 시 취소.

---

## 부록 · 남은 문서

- **T1a-common-widgets.md** — 공용 12 위젯 내부 렌더 상세. role-agnostic, 학생 대시보드에서도 그대로 사용.
- **T1b-decor-widgets.md** — 장식 7 위젯 (aurora / floating_notes / space / ripple / music_visualizer / art_text / wave_deco).
- **T1c-fun-widgets.md** — 재미 6 위젯 (fortune / tarot / fortune_cookie / gacha / weather / motivation).
- **T2 워크스페이스** — `AIDraftReviewModule` 과 짝을 이루는 `AIDraftAlert`(작업실 상단 배너).
