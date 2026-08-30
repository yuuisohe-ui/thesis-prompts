# S1 · 학생 「내 공간」 (StudentHome) 재현 프롬프트

> 현재 플랫폼 실제 구현 기준. 존재하지 않는 기능은 포함하지 않는다.
>
> **범위**: `/student-home` 페이지 전체 = 대시보드 셸(그리드/편집 모드/드로어) + 학생 카탈로그 11종 완전 스펙 + `DashboardCalendar` 학생 시점 + 학생용 Hero·PlatformFlow·SongRecommendation variant + 이벤트 버스.
>
> **범위 밖**: 공용(clock/notes/image/youtube/link_card/quote/pomodoro/dday/sticky_note/divider) · 장식(aurora/floating_notes/space/ripple/music_visualizer/art_text/wave_deco) · 재미(fortune/tarot/fortune_cookie/gacha/weather/motivation) 위젯의 렌더 세부 — 이들은 role-agnostic 하므로 **T1a / T1b 프롬프트를 그대로 재사용**한다. 본 문서는 카탈로그 항목 등록·표시 조건만 지시한다.

---

## 1. Identity

너는 시니어 React 18 / TypeScript / Vite / Tailwind / shadcn/ui / @dnd-kit / Supabase(=Lovable Cloud) 개발자다. 「내 공간」이라는 학생 전용 드래그·리사이즈 대시보드를 처음부터 만든다. 상태는 localStorage(200ms 디바운스), 데이터는 Supabase RPC/테이블(`fetchWithRetry` 래핑), 실시간은 postgres_changes 채널 + `CustomEvent` 로컬 버스로 갱신한다. 모든 UI 문구는 한국어.

---

## 2. Instructions

### 2.1 라우팅 & 페이지 셸 (`src/pages/StudentHome.tsx`)

- App 라우터: `<Route path="/student-home" element={<RequireAuth><AppLayout><StudentHome /></AppLayout></RequireAuth>} />`.
- `RequireAuth` 는 `STUDENT_ALLOWED_PREFIXES` 화이트리스트를 강제. 학생(role=student)이 다른 경로에 접근하면 `<Navigate to="/student-home" replace />`.
- 페이지 컴포넌트는 다음만 렌더:
  ```tsx
  useGuideTour();
  useEffect(() => {
    const hash = window.location.hash?.slice(1);
    if (!hash) return;
    const t = setTimeout(() => {
      const el = document.getElementById(hash);
      el?.scrollIntoView({ behavior: "smooth", block: "start" });
    }, 400);
    return () => clearTimeout(t);
  }, []);
  return (
    <div className="-m-3 sm:-m-4 md:-m-6 p-3 sm:p-4 md:p-6 min-h-full" style={{ background: "#f4f3f0" }}>
      <DashboardGrid
        variant="student"
        storageKey="dashboard_layout_student"
        defaultLayout={DEFAULT_LAYOUT_STUDENT}
        title="내 공간"
        subtitle="나만의 학습 공간을 자유롭게 꾸며보세요."
      />
    </div>
  );
  ```
- 부모 여백 상쇄로 엣지-투-엣지, 배경 크림 `#f4f3f0`.
- `#public-courses`, `#my-courses` 해시로 진입 시 400ms 지연 후 smooth 스크롤(자식 위젯이 각각 `id`를 설정).

### 2.2 타입 & 기본 레이아웃 (`src/components/dashboard/types.ts`)

```ts
export type ModuleSize = "S" | "M" | "L";
export type ModuleType =
  | "hero_insight" | "platform_flow" | "calendar" | "song_recommendation"
  | "clock" | "notes" | "image" | "youtube" | "link_card" | "quote"
  | "pomodoro" | "dday" | "wave_deco" | "divider" | "sticky_note"
  | "aurora_deco" | "floating_notes" | "space_deco" | "ripple_interactive"
  | "music_visualizer" | "art_text"
  | "fortune" | "tarot" | "fortune_cookie" | "gacha" | "weather" | "motivation"
  | "student_my_courses" | "public_courses" | "learning_record"
  | "message_teacher" | "my_favorite_songs" | "learning_goal"
  | "daily_plan" | "weekly_plan" | "notice_check"
  | "student_week" | "student_assignments"
  // 카탈로그에 노출하지 않으나 타입은 유지(레거시/교사 재사용):
  | "particle_deco"
  | "ai_draft" | "stats" | "my_classes" | "recent_activity"
  | "student_messages" | "notices";

export interface ModuleInstance { id: string; type: ModuleType; size: ModuleSize; config?: Record<string, any>; }
export interface DashboardLayout { version: 1; modules: ModuleInstance[]; }

export const DEFAULT_LAYOUT_STUDENT: DashboardLayout = {
  version: 1,
  modules: [
    { id: "s-hero",   type: "hero_insight",        size: "L" },
    { id: "s-flow",   type: "platform_flow",       size: "L", config: { variant: "student" } },
    { id: "s-mine",   type: "student_my_courses",  size: "M" },
    { id: "s-public", type: "public_courses",      size: "M" },
    { id: "s-song",   type: "song_recommendation", size: "M" },
    { id: "s-cal",    type: "calendar",            size: "M" },
  ],
};
```

Size → 그리드 컬럼 매핑(고정):
- `S` = `col-span-12 md:col-span-4`
- `M` = `col-span-12 md:col-span-6`
- `L` = `col-span-12`
그리드 컨테이너: `grid grid-cols-12 gap-3.5 auto-rows-min`.

### 2.3 레이아웃 훅 (`useDashboardLayout`)

- 시그니처: `useDashboardLayout({ storageKey?, defaultLayout? })` — 학생은 `{ storageKey: "dashboard_layout_student", defaultLayout: DEFAULT_LAYOUT_STUDENT }`.
- 초기 로드: `localStorage.getItem(key)` → JSON 파싱 → `version === 1 && Array.isArray(modules)` 검증 → 실패 시 defaultLayout.
- 저장: `layout` 변경 시 200ms `setTimeout` 디바운스로 `localStorage.setItem`. cleanup 에서 `clearTimeout`.
- id 생성: `crypto.randomUUID()` 우선, 미지원 시 `` `m_${Date.now()}_${rand36(6)}` ``.
- API:
  - `addModule(type, size="M", insertIndex?)` — 유효 index 면 `splice(insertIndex, 0, item)`, 아니면 push.
  - `removeModule(id)` / `resizeModule(id, size)` / `reorder(from, to)` / `updateModule(id, patch)` — `patch.config` 는 얕은 병합 `{ ...m.config, ...patch.config }`.
  - `reset()` — `setLayout(fallback)`.
  - `setModules(modules)` — 마이그레이션용.
- **레거시 청소(필수)**: `DashboardGrid` 최초 마운트에서 `layout.modules.some(m => m.type === "particle_deco")` 이면 즉시 `setModules(layout.modules.filter(m => m.type !== "particle_deco"))` — 이전 파티클 배경 사용자의 localStorage 방어. `useEffect` deps 빈 배열.

### 2.4 DashboardGrid (`src/components/dashboard/DashboardGrid.tsx`)

Props: `variant?: "teacher"|"student"`, `storageKey?`, `defaultLayout?`, `title?`, `subtitle?`.

- `useAuth()` 로 `profile.role` 조회 → `isTeacher = variant === "teacher" && role !== "student"` (학생 홈에서는 항상 false).
- 상태: `editing: boolean`, `dragging: { kind: "catalog", type } | { kind: "module", id } | null`.
- Sensors: `useSensor(PointerSensor, { activationConstraint: { distance: 6 } })`.
- 최상위 div: `relative min-h-full overflow-hidden transition-[padding]`, 편집 중 `md:pr-[296px]` 추가, `style={{ background: "#f4f3f0" }}`.
- 내부 z-10 컨테이너에 다음 배치:

**헤더** (`flex items-center justify-between px-1 py-2 mb-3`):
- 좌측: `<h1>` = title || "대시보드" (`text-[18px] font-bold text-slate-800`), `<p>` = subtitle || "필요한 위젯만 골라 자유롭게 배치하세요.".
- 우측 그룹(`flex items-center gap-1.5 flex-nowrap shrink-0`):
  1. 편집 중일 때만: 「기본값」 버튼 (`RotateCcw` 아이콘 + text, `text-[11px]`, `border bg-white`, 클릭 → `reset()`).
  2. 「위젯 편집」 토글 (`data-tour="dash-edit"`, `text-[12px] font-bold px-3 py-1.5 rounded-full`, editing 시 `bg-indigo-600 text-white` + `Check` 아이콘 + "완료", 아니면 `bg-white border text-slate-700` + `Pencil` 아이콘 + "위젯 편집").

**DndContext** (closestCenter, sensors, onDragStart/End/Cancel):
- `onDragStart`: `active.data.current.source === "catalog"` 이면 `{ kind: "catalog", type: data.type }`, 아니면 `{ kind: "module", id: String(active.id) }`.
- `onDragEnd`:
  - catalog 인 경우: `over` 가 있으면 그 카드의 인덱스를 찾아 `addModule(type, size, idx)`, 없으면 `addModule(type, size)`.
  - module 인 경우: `active.id !== over.id` 일 때 `reorder(from, to)`.
- `onDragCancel`: `setDragging(null)`.

**SortableContext** (`rectSortingStrategy`, items = `gridModules.map(m => m.id)`):
- `gridModules = layout.modules.filter(m => m.type !== "particle_deco")` — 파티클은 그리드에서 완전 제외.
- grid: `<div className="grid grid-cols-12 gap-3.5 auto-rows-min">` → 각 인스턴스를 `<DashboardModuleFrame>` + `<ModuleRenderer>` 로 렌더.

**DragOverlay** (`dropAnimation={null}`):
- catalog dragging: 아이콘 박스(`bg-indigo-50 text-indigo-600`) + 라벨 (`text-[12px] font-bold`) 미니 카드.
- module dragging: 해당 카탈로그 라벨만 표시하는 카드.

**DashboardEditPanel** 슬라이드 드로어 렌더(다음 절).

### 2.5 DashboardModuleFrame (`DashboardModuleFrame.tsx`)

- `useSortable({ id: instance.id, disabled: !editing })` → `transform`/`transition` 을 `CSS.Transform.toString()` 으로 style 에 적용.
- 컨테이너: `relative bg-white rounded-[16px] border overflow-hidden`, size 클래스 첨가, `isDragging && "opacity-60"`, `editing && "ring-1 ring-indigo-200"`.
- 편집 모드 오버레이(우상단, `absolute top-2 right-2 z-20 flex items-center gap-1 bg-white/95 border rounded-md px-1 py-0.5 shadow-sm`):
  - `S / M / L` 토글 3버튼 — 활성 = `bg-indigo-600 text-white`, 비활성 = `text-slate-500 hover:bg-slate-100`, `text-[10px] font-bold px-1.5 py-0.5 rounded`. 클릭 → `onResize(id, size)`.
  - 우측 끝 `X` 삭제 버튼 → `onRemove(id)`.
- 편집 모드 좌상단 드래그 핸들: `GripVertical` 아이콘, `absolute top-2 left-2 z-20 bg-white/95 border rounded-md p-1 shadow-sm cursor-grab active:cursor-grabbing`, `{...attributes} {...listeners}`, aria-label="드래그".
- 자식 컨테이너: `h-full`, 편집 중일 때 `pt-2` 로 오버레이와 겹침 방지.

### 2.6 DashboardEditPanel (우측 드로어, variant="student")

- 위치: `fixed top-0 right-0 h-screen w-[280px] bg-white border-l z-40 shadow-xl transition-transform duration-300 flex flex-col`, 열림 = `translate-x-0`, 닫힘 = `translate-x-full`.
- 모바일 오버레이(md 이하): `fixed inset-0 bg-black/20 z-30 md:hidden`, 클릭 시 `onClose()`.
- 헤더 (`h-12 border-b`): 「위젯 추가」 (`text-sm font-bold`) + 우측 `X` 버튼.
- 안내 스트립 (`px-4 py-2 text-[11px] text-slate-500 border-b bg-slate-50/60`): "드래그해서 원하는 위치에 놓거나, + 를 눌러 끝에 추가하세요.".
- 섹션 필터(`variant === "student"` 규칙):
  - 항상 제외: `s.key === "teacher"`.
  - `s.key === "student"` 는 항상 표시 (학생이므로 `!isTeacher`).
  - 최종 노출 순서: 🧩 공용 / ✨ 장식 & 애니메이션 / 🔮 재미 & 운세 / 🎓 학생 전용.
- 섹션 아코디언 상태: `openMap: Record<CatalogCategory, boolean>` 초기 모두 false. 각 섹션 헤더:
  - `w-full flex items-center gap-2 px-3 py-2.5 hover:bg-slate-50 text-left`, `aria-expanded={isOpen}`.
  - 왼쪽: 이모지 (`text-base`), 라벨 (`text-[12.5px] font-extrabold text-slate-800 flex-1`).
  - 우측: 개수 뱃지 (`text-[10px] font-bold px-1.5 py-0.5 rounded-full bg-slate-100 text-slate-600`) + `ChevronDown` (열림 시 `rotate-180`).
- 각 카탈로그 행(`CatalogRow`):
  - `useDraggable({ id: "catalog:{type}", data: { source: "catalog", type, defaultSize } })`.
  - Layout: `w-full flex items-start gap-2 p-2.5 rounded-xl border bg-white hover:bg-indigo-50/40 hover:border-indigo-200 text-left transition`, dragging 시 `opacity-40`.
  - 좌측 `GripVertical` 핸들 버튼 (`{...attributes} {...listeners}`, aria-label="드래그하여 추가").
  - 아이콘 박스 (`w-8 h-8 rounded-lg bg-indigo-50 text-indigo-600 flex items-center justify-center shrink-0`).
  - 본문: 라벨 (`text-[12.5px] font-bold text-slate-800`) + 설명 (`text-[11px] text-slate-500 mt-0.5 leading-snug`). 본문 버튼 클릭 → `onAdd(type)`.
  - 우측 `+` 버튼 → `onAdd(type)`.

### 2.7 카탈로그 스펙 (`dashboardModuleCatalog.tsx`)

`CatalogItem = { type, label, description, icon: LucideIcon, defaultSize: "S"|"M"|"L", category: "common"|"deco"|"fun"|"teacher"|"student", teacherOnly?, studentOnly? }`.

학생 홈에 노출되는 항목만 열거(교사 전용은 학생 드로어에서 숨김이므로 값은 존재해도 무방):

**common (12)** — 세부는 T1a 참조. 이 문서에서는 등록만 지시한다.

| type | 라벨 | 설명 | 아이콘 | defaultSize |
|---|---|---|---|---|
| clock | 시계 | 디지털 시계 | AlarmClock | S |
| notes | 메모 | 간단 메모 | FileText | S |
| image | 이미지 | 업로드 또는 Pixabay | Image | S |
| youtube | 영상 임베드 | YouTube 임베드 | Youtube | M |
| link_card | 링크 카드 | 외부 링크 즐겨찾기 | Link2 | S |
| quote | 명언 카드 | 랜덤 명언 | Quote | S |
| pomodoro | 포모도로 타이머 | 25분 집중 | Timer | S |
| dday | D-day | 남은 날 카운트 | ListTodo | S |
| song_recommendation | 오늘의 노래 추천 | AI 교학 팁 포함 | Music | M |
| calendar | 캘린더 & 일정 | 수업 일정 보기 | CalendarDays | M |
| sticky_note | 스티커 메모 | 포스트잇 스타일 | StickyNote | S |
| divider | 구분선 | 섹션 분리 | Minus | L |

**deco (7)** — 세부는 T1b 참조. `aurora_deco, floating_notes, space_deco, ripple_interactive, music_visualizer, art_text, wave_deco`(레거시).

**fun (6)** — 세부는 T1b 참조. `fortune, tarot, fortune_cookie, gacha, weather, motivation`.

**student (11, `studentOnly: true`)** — 아래 §3 에서 각 모듈 완전 재현.

| type | 라벨 | 설명 | 아이콘 | defaultSize |
|---|---|---|---|---|
| student_my_courses | 내 수업 | 참여 중인 반 | BookOpen | M |
| public_courses | 공개 수업 | 참여 가능한 공개 반 | BookOpen | M |
| learning_record | 학습 기록 | 즐겨찾기/참여 반 카운트 | Activity | S |
| message_teacher | 선생님께 메시지 | 반 선택 후 전송 | MessageSquare | M |
| my_favorite_songs | 즐겨찾기 노래 | 내가 좋아한 노래 | Heart | M |
| learning_goal | 이번 달 학습 목표 | 체크리스트 형태 | ListTodo | S |
| daily_plan | 오늘의 계획 | 매일 리셋되는 체크리스트 | CalendarCheck | S |
| weekly_plan | 이번 주 계획 | 주 단위 체크리스트 | CalendarRange | S |
| notice_check | 공지 확인 | 가입한 반의 공지 모음 | Bell | M |
| student_week | 이번 주 강의 | 이번 주 일정 | CalendarDays | M |
| student_assignments | 과제 현황 | 선생님 과제 + 내가 추가 | GraduationCap | M |

Helper: `getCatalogItem(type) = DASHBOARD_CATALOG.find(c => c.type === type)`.

**카탈로그에 절대 노출 금지(학생 홈 기준)**:
- teacher 카테고리 전부 (`ai_draft, stats, my_classes, recent_activity, student_messages, notices, hero_insight, platform_flow`) — 단 `hero_insight` / `platform_flow` 는 기본 레이아웃에서는 사용하되 카탈로그 항목은 teacherOnly 로 남겨 학생 편집 패널에 노출되지 않게 한다.
- 레거시 `particle_deco` — 카탈로그 배열에서 완전 제거.

### 2.8 ModuleRenderer

`type` → 컴포넌트 switch. `default: return null`. Props: `{ instance, onConfigChange, editing }`.
`onConfigChange(id, config)` = `updateModule(id, { config })` (얕은 병합).

---

## 3. 학생 카탈로그 11종 완전 스펙

모든 모듈은 공용 껍데기 `_ModuleShell({ title, icon, right?, children, bodyClassName? })` 를 사용한다. 껍데기 규칙:
- 최상위: `h-full flex flex-col`.
- 헤더(`px-4 pt-3 pb-2 border-b`): 좌측 `icon + title` (`text-[13px] font-bold text-slate-800`), 우측 `right` 슬롯 (`text-[11px] text-slate-400`).
- 본문: `flex-1 min-h-0 ${bodyClassName ?? "p-4"}`.

데이터 조회는 모두 `fetchWithRetry((s) => query.abortSignal(s!))` 로 감싼다 (backoff 3회, base 2000ms, max 10000ms). 컴포넌트 언마운트 시 in-flight 요청 취소(`cancelled` 플래그).

### 3.1 `StudentMyCoursesModule` — 내 수업 (M)

- **랩퍼**: `<div id="my-courses">` — 해시 스크롤 앵커.
- **조회 순서**:
  1. `supabase.rpc("get_my_joined_courses")` → 배열에서 `!r.deleted_at` 필터 → `{ course_id, name, level }[]`.
  2. list 비어있지 않으면 병렬로 `supabase.from("courses").select("id, class_time").in("id", courseIds)` → `class_time` 을 각 row 에 병합.
- **상태**: `rows`, `loading`, `pendingLeave: Row | null`.
- **렌더**:
  - 로딩: `"불러오는 중…"` (`text-[12px] text-slate-400 text-center py-6`).
  - 빈 상태: `"아직 참여한 반이 없어요.\n공개 수업을 둘러보세요!"`.
  - 카드(최대 6개): `group px-3 py-2 rounded-xl border bg-white hover:bg-indigo-50/50 hover:border-indigo-200 transition flex items-center justify-between gap-2`.
    - 왼쪽 버튼: 이름 (`text-[12.5px] font-bold truncate`) + 서브라인 (`text-[10.5px] text-slate-400 truncate`): `{level}` + `" · "` + `{class_time}` (있는 것만). 클릭 → `navigate("/courses/${course_id}")`.
    - 오른쪽 hover 액션: `Trash2`(반 나가기, `opacity-0 group-hover:opacity-100`, hover 시 `text-red-600 bg-red-50`) → `setPendingLeave(row)`. 우측 `ArrowRight` 아이콘 버튼 → 같은 경로 이동.
- **탈퇴 다이얼로그** (`AlertDialog`):
  - 제목: "이 반에서 나가시겠습니까?".
  - 설명: "`{name}` 에서 나갑니다. 언제든 다시 참여할 수 있어요.".
  - 확인 클릭 → `supabase.from("course_members").delete().eq("course_id", …).eq("user_id", uid)`.
    - 실패 → `toast.error("나가기 실패: " + error.message)`.
    - 성공 → `toast.success("반에서 나갔어요.")`, 로컬 rows 필터 제거, `window.dispatchEvent(new Event("student-courses:refresh"))`.
- **실시간 갱신**:
  - `supabase.channel("student-courses-<uid>").on("postgres_changes", { event: "UPDATE", schema: "public", table: "courses" }, refetch)`.
  - `window.addEventListener("student-courses:refresh", refetch)` + `"course:updated"` 도 리슨. 언마운트 시 모두 해제.

### 3.2 `PublicCoursesModule` — 공개 수업 (M)

- **랩퍼**: `<div id="public-courses">`.
- **페이지 크기**: 4.
- **조회**: `supabase.from("courses").select("id, name, level", { count: "exact" }).or("owner_id.is.null,is_public.eq.true").is("deleted_at", null).order("sort_order", { ascending: true }).range(from, to)`.
- **가입 상태 프리로드**: `supabase.from("course_members").select("course_id").eq("user_id", uid)` → `joinedIds: Set<string>`. `student-courses:refresh` 리스너로 재조회.
- **헤더 우측(right)**: `page+1 / totalPages` 뱃지 사이에 `ChevronLeft` / `ChevronRight` 버튼. `page === 0 || loading` 시 이전 disabled, 마지막 페이지도 유사. `text-[11px] tabular-nums`.
- **본문**:
  - 로딩/빈 상태 → 안내 문구.
  - 각 행: `w-full text-left px-3 py-2 rounded-xl border bg-white hover:bg-emerald-50/50 hover:border-emerald-200 flex items-center justify-between gap-2`. 왼쪽: 이름 + `level` 뱃지 (`bg-emerald-50 text-emerald-700 border-emerald-200 text-[10px] font-bold rounded-full px-1.5 py-0.5`). 오른쪽 `+` 원형 버튼 (`h-7 w-7 rounded-full`):
    - `joined` → `Check` 아이콘, `bg-emerald-100 border-emerald-300 text-emerald-700`, aria-label "이미 참여 중".
    - 아니면 `Plus`, `bg-white hover:bg-emerald-50 hover:border-emerald-300 text-emerald-600`, aria-label "내 수업에 추가".
- **가입 액션 `handleAdd`**:
  - `e.stopPropagation()` (카드 네비 방지).
  - `joinedIds.has(courseId)` → `toast.info("이미 참여 중인 반이에요.")`.
  - 아니면 `supabase.from("course_members").insert({ course_id, user_id: uid, role: "student" })`. 실패 → `toast.error`. 성공 → `joinedIds` 낙관 갱신 + `toast.success("내 수업에 추가됐어요!")` + `student-courses:refresh` 브로드캐스트.
- 카드 본문 클릭 → `navigate("/courses/${r.id}")`.

### 3.3 `LearningRecordModule` — 학습 기록 (S)

- 2컬럼 그리드 (`grid grid-cols-2 gap-3 py-2`):
  - 즐겨찾기 노래 카드: `rounded-xl border bg-gradient-to-br from-rose-50 to-white p-3 text-center`, 숫자 `text-2xl font-extrabold text-rose-600`.
  - 참여 반 카드: `from-indigo-50 to-white`, 숫자 `text-indigo-600`.
- 카운트:
  - `supabase.from("song_favorites").select("id", { count: "exact", head: true }).eq("user_id", uid)` → `songs`.
  - `supabase.from("course_members").select("id", { count: "exact", head: true }).eq("user_id", uid)` → `classes`.
- `<CountUp value={n} />` 이징: `requestAnimationFrame`, `dur = 800ms`, `progress = 1 - Math.pow(1 - p, 3)` (cubic-out). 값 변경 시 새 rAF.

### 3.4 `MessageTeacherModule` — 선생님께 메시지 (M)

- 상태: `courses: {course_id, course_name}[]`, `selected`, `content`, `loading`, `sending`.
- **참여 반 조회**:
  ```
  supabase.from("course_student_profiles")
    .select("course_id, courses:course_id ( id, name )")
    .eq("member_user_id", uid)
    .is("deleted_at", null)
  ```
  → `course_id` 로 중복 제거. 1개면 `selected` 자동 지정.
- **UI 상태**:
  - loading → 중앙 `Loader2 animate-spin`.
  - 참여 반 0 → Textarea disabled + placeholder "먼저 강의에 참여해 주세요.", 오른쪽 Send 버튼 disabled + Tooltip("먼저 강의에 참여해 주세요").
  - 참여 반 ≥ 2 → shadcn `Select` (h-8 text-[12px]) 로 반 선택, placeholder "반을 선택하세요".
  - 참여 반 = 1 → 반 이름을 `text-[11px] text-slate-400` 로 표시.
- Textarea: `flex-1 min-h-[80px] text-[12.5px] resize-none`, placeholder "선생님께 전하고 싶은 말을 적어 보세요.".
- Send 버튼: `self-end inline-flex items-center gap-1.5 text-[12px] font-bold px-3 py-1.5 rounded-full bg-indigo-600 text-white hover:bg-indigo-500 disabled:opacity-50 disabled:cursor-not-allowed`, sending 시 `Loader2 animate-spin`.
- `canSend = !!user?.id && !!content.trim() && !!selected && !sending`.
- 전송:
  ```
  supabase.from("course_student_posts").insert({
    course_id: selected,
    student_id: user.id,
    content: content.trim(),
    type: "메시지",
    visibility: "private",
    status: "pending",
  })
  ```
  실패 → `toast.error("메시지 전송에 실패했어요")`. 성공 → `setContent("")` + `toast.success("선생님께 메시지를 보냈어요")`.

### 3.5 `MyFavoriteSongsModule` — 즐겨찾기 노래 (M)

- **조회 순서**:
  1. `song_favorites.select("song_id").eq("user_id", uid).order("created_at", { ascending: false }).limit(6)`.
  2. `songs.select("id, title, artist, video_id, youtube_url, language, lyrics_raw, deleted_at").in("id", favoriteIds).is("deleted_at", null)`.
  3. `favoriteIds` 순서를 유지하도록 Map lookup 으로 재정렬.
- `extractVideoId(url)` 정규식: `(?:v=|youtu\.be\/|embed\/|shorts\/)([A-Za-z0-9_-]{11})`. 없으면 `song.video_id` fallback.
- 그리드: `grid grid-cols-2 gap-2` (총 최대 6장).
- 카드: `text-left rounded-lg border bg-white overflow-hidden hover:border-pink-300 hover:shadow-md hover:-translate-y-0.5 transition`.
  - 상단 썸네일 영역: `relative w-full aspect-video bg-gradient-to-br from-pink-100 to-amber-100 flex items-center justify-center overflow-hidden`.
    - `video_id` 있으면 `<img src="https://i.ytimg.com/vi/${id}/mqdefault.jpg" loading="lazy" onError={style.display='none'} />`.
    - 없으면 `Music h-6 w-6 text-pink-400` 아이콘.
  - 하단: `px-2 py-1.5` — 제목 `text-[12px] font-bold truncate`, 부제(artist) `text-[10.5px] text-slate-400 truncate`.
- 빈 상태: "아직 즐겨찾기한 노래가 없어요.".
- **클릭 → `openAnalysis(row)`**:
  - `setSelectedSong(row.raw)`, `setDialogOpen(true)`, `setAnalysisLoading(true)`, reset `analysisNotFound`.
  - `supabase.from("song_analyses").select("*").eq("song_id", row.song_id).order("created_at", { ascending: false }).limit(1).maybeSingle()`.
  - error 또는 없음 → `setAnalysisNotFound(true)`, error 만 `toast.error("분석을 불러오지 못했습니다")`.
  - 성공 → `setSelectedAnalysis(data)`.
  - `finally` → `setAnalysisLoading(false)`.
- `<SongAnalysisDialog ... readOnly onReanalyze={() => {}} onAnalysisUpdated={setSelectedAnalysis} />` — 재분석/편집 버튼 숨김.

### 3.6 `NoticeCheckModule` — 공지 확인 (M)

- **조회 순서**:
  1. `course_student_profiles.select("course_id").eq("member_user_id", uid).is("deleted_at", null)` → 중복 제거로 `courseIds`.
  2. `courseIds` 없으면 빈 상태.
  3. 병렬:
     - `courses.select("id, name, owner_id").in("id", courseIds)`.
     - `course_notices.select("*").in("course_id", courseIds).order("created_at", { ascending: false }).limit(50)`.
  4. `profiles.select("id, full_name").in("id", ownerIds)` 로 teacher_name 매핑.
  5. `notice_reads.select("notice_id").eq("user_id", uid).in("notice_id", noticeIds)` → `readIds: Set`.
- **헤더 right**: `unreadCount > 0` 이면 `"{n} 새 글"` 뱃지 (`bg-indigo-500 text-white`).
- **본문 (`bodyClassName="p-0 overflow-y-auto"`)**: `<ul className="divide-y">`, 각 `<li>` 는 접혀있으면 `opacity-60` (읽음).
  - 헤더 버튼: `w-full text-left px-4 py-2.5 hover:bg-slate-50`.
    - 좌측: 미읽음 도트 (`w-1.5 h-1.5 bg-indigo-500 rounded-full`).
    - 본문: 카테고리 뱃지 (`bg-amber-50 text-amber-700`) + 첫 줄(`n.content.split("\n")[0]`, 미읽음 `text-slate-800 font-semibold`, 읽음 `text-slate-500`).
    - 서브: `{course_name} · {teacher_name} 선생님 · {shortTime(created_at)}`.
    - 우측: `ChevronDown/ChevronUp`.
  - 펼쳤을 때: `px-4 py-3 bg-slate-50 border-t` — 본문 `whitespace-pre-wrap text-[13px]` + `<RepliesThread parentType="notice" parentId={n.id} courseId={n.course_id} user={user} isTeacher={isTeacher} compact />`.
- `shortTime(iso)` 규칙: <60분 → "N분 전" (min 1), <24시간 → "N시간 전", <7일 → "N일 전", 이후 → "M/D".
- `markRead(id)` (옵티미스틱):
  1. 로컬 `readIds` 에 즉시 추가.
  2. `supabase.from("notice_reads").upsert({ user_id, notice_id }, { onConflict: "user_id,notice_id", ignoreDuplicates: true })`.
  3. 실패 시 로컬 상태 롤백.
- `toggle(id)` = `markRead(id) + setExpandedId(cur => cur === id ? null : id)`.

### 3.7 `LearningGoalModule` / `DailyPlanModule` / `WeeklyPlanModule`

모두 공용 `ChecklistShell` 사용.

**공용 shell 스펙 (`_ChecklistShell.tsx`)**:
- Props: `{ title, icon, storageKey, placeholder?, accent?: "amber"|"sky"|"violet", emptyText? }`.
- accent 팔레트:
  - `amber`: 버튼 `bg-amber-500 hover:bg-amber-600`, focus ring `focus:ring-amber-300`.
  - `sky`: `bg-sky-500`, `focus:ring-sky-300`.
  - `violet`: `bg-violet-500`, `focus:ring-violet-300`.
- 아이템: `{ id, text, done }[]`. id = `crypto.randomUUID()` fallback.
- 초기 로드: `localStorage.getItem(storageKey)`. JSON 배열이면 그대로. 문자열이면 legacy 마이그레이션 → `[{ id, text: parsed.trim(), done: false }]`.
- 저장: `localStorage.setItem(storageKey, JSON.stringify(items))` — 매 변경마다.
- 헤더 아래 카운트: `text-[10.5px] text-slate-400 font-semibold` → `{done}/{total} 완료` (아이템 있을 때만).
- 리스트: `max-h-[180px] overflow-auto pr-1` — 각 항목 `Checkbox` + 텍스트(`text-[12.5px] break-words`, done 시 `line-through text-slate-400`) + hover 시 `X` 삭제 버튼.
- 하단 인풋: `Input h-8 text-[12px] + focus:ring-*` + `Plus` 정사각 버튼(`h-8 w-8 rounded-md text-white`, disabled 시 opacity 40). Enter 또는 + 클릭으로 추가, `draft.trim()` 필요.
- 빈 상태: `text-[11.5px] text-slate-400 text-center py-3`, `emptyText || "아직 항목이 없어요. 아래에서 추가해 보세요."`.

**모듈별 인스턴스화**:

| 모듈 | title | icon | storageKey | accent | placeholder | emptyText |
|---|---|---|---|---|---|---|
| `learning_goal` | "이번 달 학습 목표" | `Target h-4 w-4 text-amber-500` | `student_learning_goal_items` | amber | "예) 매주 노래 한 곡 외우기" | "이번 달 목표를 자유롭게 추가해 보세요." |
| `daily_plan` | `` `오늘의 계획 (${todayKey()})` `` | `CalendarCheck h-4 w-4 text-sky-500` | `` `student_daily_plan_${todayKey()}` `` | sky | "오늘 할 일을 적어 주세요" | "오늘의 계획을 추가해 보세요." |
| `weekly_plan` | `` `이번 주 계획 (${isoWeekKey()})` `` | `CalendarRange h-4 w-4 text-violet-500` | `` `student_weekly_plan_${isoWeekKey()}` `` | violet | "이번 주 목표를 추가" | "이번 주 계획을 추가해 보세요." |

- `todayKey()` = `YYYY-MM-DD` (로컬 시간). 매일 자정 지나면 새 키 → 이전 날 항목은 자동 사라짐(과거 키에 여전히 존재하지만 UI 미노출).
- `isoWeekKey()` = ISO 주 계산 (`YYYY-Www`): `d.setDate(d.getDate() + 3 - ((d.getDay()+6)%7))` → Thursday, `jan4 = new Date(year, 0, 4)`, 주 번호 = `1 + Math.round((diff - 3 + ((jan4.getDay()+6)%7)) / 7)`.

### 3.8 `CalendarModule` → `DashboardCalendar` (M)

`CalendarModule` 은 껍데기 없이 `<div className="h-full"><DashboardCalendar /></div>` 만 렌더한다. `DashboardCalendar` 는 학생/교사가 공유하되 데이터 소스와 편집 권한이 다르다.

#### 3.8.1 상태

- `today = startOfDay(new Date())`.
- `month`(현재 보이는 달), `selectedDate: Date | null`.
- `courses: { id, name, class_time, weekdays: number[], color, start_date }[]`.
- `events: { id, title, description, starts_at, course_id, item_type, event_type, all_day }[]`.
- `customTypes: { id, label, color }[]`.
- `hiddenIds: string[]` — localStorage `hidden_calendar_courses`.
- 편집 패널 상태: `panelOpen, editingId, newTitle, selType, ampm ("am"|"pm"), hour (1..12), minute (0/15/30/45), allDay, saving`.
- 커스텀 타입 폼: `showCustomForm, customName, customColor`.
- `confirmDelete: { kind: "event", eventId, courseId?, canHideCourse } | { kind: "course", courseId, courseName } | null`.
- `containerRef` — outside-click 감지.

#### 3.8.2 상수

- `weekdayMap = { 일:0, 월:1, 화:2, 수:3, 목:4, 금:5, 토:6 }`.
- `weekdayLabels = ["일","월","화","수","목","금","토"]`.
- `COURSE_COLORS = ["bg-dash-blue","bg-dash-teal","bg-dash-gold","bg-dash-purple"]`.
- `COURSE_HEX = ["#2557a7","#0e7a5a","#b8882a","#6b4faa"]`.
- Built-in 타입 3종:
  - `class` "수업" `#2557a7`
  - `memo` "개인 메모" `#b8882a`
  - `task` "할 일" `#6b4faa`
- 커스텀 색상 팔레트 12색: `["#2557a7","#0e7a5a","#b8882a","#6b4faa","#c94040","#1a7070","#c05090","#507830","#705018","#404880","#8a5028","#308898"]`.

#### 3.8.3 데이터 로드 (`loadCourses`)

- `useAuth()` 의 `isTeacher / isAdmin` 로 분기 — 학생 홈에서는 항상 학생 흐름:
  - 학생: `supabase.rpc("get_my_joined_courses")` → `!deleted_at` 필터 → `joinedIds`.
  - `courses.select("id, name, class_time, start_date").in("id", joinedIds).is("deleted_at", null)` 로 상세 병합.
- `hidden_calendar_courses` localStorage 값으로 숨김 필터.
- 각 코스 index 로 `COURSE_COLORS[i%4]` 라운드-로빈 배정, `weekdays = parseClassWeekdays(class_time)` (`/[일월화수목금토]/g` 정규식 → 매핑 → 중복 제거 → 오름차순).
- Realtime: `supabase.channel("dash-cal-courses-<uid>").on("postgres_changes", { event: "*", schema: "public", table: "courses" }, loadCourses)`. `window.addEventListener("course:updated", loadCourses)`. 언마운트 시 채널·리스너 해제.

#### 3.8.4 이벤트 & 커스텀 타입 로드

- `course_calendar_items.select("id, title, description, starts_at, course_id, item_type, event_type, all_day").eq("user_id", uid).order("starts_at")` → `events`.
- `custom_event_types.select("id, label, color").eq("user_id", uid).order("created_at")` → `customTypes`.
- `allTypes = [...BUILTIN_TYPES, ...customTypes]`, `typeById(id)` 헬퍼.

#### 3.8.5 렌더 — 헤더 + 월 그리드

- 카드 컨테이너: `bg-card border border-border rounded-[14px] overflow-hidden`, ref = `containerRef`.
- 헤더 (`px-[18px] pt-[14px] pb-3 border-b`):
  - 좌: 📅 아이콘 박스(`w-6 h-6 rounded-md bg-dash-blue-bg`) + "캘린더 & 일정" (`text-[13.5px] font-bold`).
  - 우: `+ 일정 추가` 버튼 (`text-xs text-dash-blue font-semibold`) → `openPanel()`.
- 월 네비 (`px-[18px] pb-3`):
  - 좌: `format(month, "yyyy년 M월", { locale: ko })`.
  - 우: `‹` / `›` 버튼 (`w-[26px] h-[26px] rounded-[7px] border`) → `subMonths` / `addMonths`.
- 셀 계산: `firstOfMonth`, `startOffset = firstOfMonth.getDay()`, `daysInMonth`. 빈 셀 + 실제 날짜 셀 배치.
- 요일 헤더: 일요일·토요일 `text-destructive`, 나머지 `text-dash-text4`.
- 각 날짜 셀:
  - 오늘: 날짜 텍스트를 `bg-dash-navy text-white w-[22px] h-[22px] rounded-full` 원 안에.
  - `selectedDate && isSameDay` 이고 오늘 아님 → `bg-dash-blue-bg`.
  - 일요일(선택·오늘 아님) → `text-destructive`.
  - 각 날짜에 dot 최대 3개: 해당 날짜의 코스(`getCoursesForDate`) HEX + 이벤트 타입 컬러(`typeById(e.event_type).color`).
- `getCoursesForDate(date)`:
  - `c.weekdays.includes(date.getDay())` 이고, `c.start_date` 가 있으면 `startOfDay(date) >= startOfDay(new Date(start_date+'T00:00:00'))`.
- `getEventsForDate(date)`:
  - `e.starts_at` 이 같은 날 & (course_id 없거나 placeholder `00000000-0000-0000-0000-000000000000` 또는 visible courseIds).

#### 3.8.6 아젠다 (아래 리스트)

- `weekStart / weekEnd = startOfWeek/endOfWeek(today, { weekStartsOn: 0 })`.
- `weekSchedule` 또는 `daySchedule` (`selectedDate` 여부).
- 각 아이템: 컬러 3px 바 (`hexColor`) + 코스명/이벤트 제목 + 타입 뱃지 (`background: color+"18", color: color`) + 시간 (`format(HH:mm)` 또는 "종일").
- 이벤트 hover 액션: `편집` 버튼(→ `openPanel(eventId)`) + `삭제` 버튼(→ confirmDelete = event).
- 반복 수업(`isClass`) hover 액션: `숨기기` (→ confirmDelete = course).
- 상단 라벨: `selectedDate` 있으면 "당일 일정" + `format(selectedDate, "M월 d일 (E)", { locale: ko })`, 없으면 "이번 주 일정".
- `← 이번 주로` 되돌리기 버튼 (selectedDate 일 때).
- 하단 `+ 일정 추가` 버튼 (`openPanel()`).

#### 3.8.7 편집 패널 (하단 슬라이드)

- 조건부 렌더 `panelOpen && ...`, 애니메이션 `animate-in slide-in-from-bottom-2 duration-200`.
- 헤더: `editingId` 면 "일정 편집", 아니면 "M월 d일 새 일정" or "새 일정" + `✕` 닫기.
- 입력: 제목 인풋 (Enter → `handleSave`, autoFocus).
- 유형 칩 (`allTypes.map`): 선택 시 `background: t.color, color: white`. 옆에 `+ 유형 추가` 대시 버튼 → 커스텀 폼 토글.
- 커스텀 유형 폼: 이름 인풋 + 12색 스와치 (선택 시 `scale-110 border-foreground`) + 취소/추가 버튼 → `custom_event_types.insert({ user_id, label, color })`.
- 시간: AM/PM · 시(1~12) · 분(0/15/30/45) 3개 셀렉트 + `종일` 체크박스 (체크 시 셀렉트 disabled).
- 저장 / 취소 버튼.
- `handleSave`:
  - `targetDate = selectedDate || today`.
  - allDay 이면 00:00, 아니면 12h → 24h 변환 (`am+12=0`, `pm+non-12+12`) → `Date.setHours(h24, minute, 0, 0)` → ISO.
  - editingId 있음: `course_calendar_items.update({ title, starts_at, event_type: selType, all_day }).eq("id", editingId)`.
  - 없음: `.insert({ course_id: courses[0]?.id ?? placeholder, title, starts_at, item_type: "custom", event_type: selType, all_day, user_id: uid })`.
  - 성공 → toast "일정 추가 완료" / "일정 수정 완료", 실패 → destructive toast.
  - 성공 후 `closePanel()` + `fetchEvents()`.

#### 3.8.8 삭제 흐름

- `confirmDelete` 모달 (`fixed inset-0 z-50 bg-black/40` + 320px 카드):
  - `event` 모드: "이 일정만 삭제" (=`deleteEventOnly` → `.delete().eq("id")`), `canHideCourse` 이면 "이 수업 캘린더에서 숨기기" 추가.
  - `course` 모드: "숨기기" 확인 → `hideCourse(id)` = `hidden_calendar_courses` 에 추가 + toast("캘린더에서 숨김").

#### 3.8.9 범례 & 숨김 관리

- 코스 있으면 하단 범례: `flex flex-wrap gap-2.5 px-[18px] py-[10px] border-t` — 각 코스 컬러 도트 + "이름 (요일들)".
- `hiddenIds.length > 0` 이면 하단에 "숨겨진 수업 N개" + "모두 다시 표시" 링크 (`saveHidden([])` + `setHiddenIds([])` + toast).

#### 3.8.10 학생 권한 규칙 (RLS 로도 이중 방어, 클라이언트 UX)

- `class` 반복 일정(가상 항목, DB 미저장) — 편집 불가. hover 시 오직 「숨기기」만 노출.
- `course_calendar_items` 행 중 `user_id === auth.uid()` 인 것만 편집/삭제 UI 노출 (다른 사람 것은 애초 조회되지 않음).
- 학생은 반 소유 이벤트를 추가할 수 있으나 실질적으로는 개인 memo/task 용도. `handleSave` 의 `course_id` 는 `courses[0]?.id` fallback 을 그대로 사용해 학생의 첫 참여 반에 붙거나 placeholder 로 저장.
- 외부 클릭 (`mousedown` on document) → `selectedDate = null`.

### 3.9 `StudentWeekModule` — 이번 주 강의 (M)

- **아이콘**: `CalendarDays h-4 w-4 text-indigo-500`, 제목 "이번 주 강의".
- **주 계산**: `startOfWeek(now)` = 월요일 00:00 (`day=(getDay()+6)%7` 만큼 뒤로) → `nextMonday = monday+7일`.
- **조회 순서**:
  1. `course_student_profiles.select("course_id").eq("member_user_id", uid).is("deleted_at", null)` → 중복 제거 `courseIds`. 빈 배열이면 즉시 종료.
  2. 병렬:
     - `course_calendar_items.select("id, title, starts_at, event_type, all_day, course_id").in("course_id", courseIds).gte("starts_at", monday.ISO).lt("starts_at", nextMonday.ISO).order("starts_at").limit(20)`.
     - `courses.select("id, name, class_time, start_date, deleted_at").in("id", courseIds)` → `!deleted_at` 필터로 name 맵 생성.
- **반복 수업 확장**: 각 코스의 `class_time` 문자열을 파싱:
  - `KO_TO_IDX = { 일:0, 월:1, 화:2, 수:3, 목:4, 금:5, 토:6 }`.
  - 요일 = `class_time.match(/[일월화수목금토]/g)`, 시간 = `/(\d{1,2}):(\d{2})/` 매치 (기본 10:00).
  - 각 요일 → `offset = (dayIdx-1+7)%7` (월요일=0) → `meet = monday + offset 일 + HH:mm`. `start_date` 존재 & `meet < startDate` 이면 스킵. 이번 주 범위 밖도 스킵.
  - `{ id: "class-<courseId>-<ISO>", title: "수업", starts_at: meet.ISO, event_type: "class", all_day: false, course_id, course_name }` 로 추가.
- **정렬 & 표시**: `starts_at` 오름차순, 상위 5개만 렌더.
- **날짜 배지**: `${M}/${D} ${KO_DOW[getDay()]}` + 시간(`HH:mm` 또는 all_day 면 "종일"). `w-14 py-1 rounded-md bg-indigo-50 text-indigo-700`, 2줄 (badge/time).
- **이모지 매핑**: `class/lesson=📚`, `assignment/homework=📝`, `exam/test=📊`, `event=📅`, 기본 `📌`.
- **행 스타일**: `flex items-start gap-2 px-2 py-1.5 rounded-md hover:bg-slate-50`. 본문: 제목(`text-[12.5px] font-semibold text-slate-800 truncate` + 이모지) + 코스명(`text-[11px] text-slate-400 truncate`).
- **상태**: 로딩 → "불러오는 중…"(`text-[12px] text-slate-400 text-center py-6`), 빈 → "이번 주 예정된 일정이 없어요.".
- 언마운트 취소 플래그(`cancelled`) 준수. `fetchWithRetry` 필수.

### 3.10 `StudentAssignmentsModule` — 과제 현황 (M)

- **아이콘**: `GraduationCap h-4 w-4 text-emerald-500`, 제목 "과제 현황".
- **하이브리드 소스**: 선생님 과제(공지) + 학생 로컬 과제 2종을 하나의 체크리스트에 병합.
- **선생님 과제 조회**:
  1. `supabase.rpc("get_my_joined_courses")` → `!deleted_at` 필터, `courseIds` + `nameMap`.
  2. `course_notices.select("id, title, category, course_id, created_at").in("course_id", courseIds).order("created_at", { ascending: false }).limit(30)`.
  3. 필터: `(category ?? "").toLowerCase().includes("assign") || category === "과제"`.
  4. 결과 → `{ id, title, course_name: nameMap[course_id]||"수업", created_at, source:"teacher" }`.
- **로컬 상태 스토리지**:
  - `LOCAL_KEY = "student_assignments_local"` — `{ id, text, done, source:"local" }[]`.
  - `DONE_KEY = "student_assignments_done_teacher"` — 완료 처리된 선생님 과제 `id[]` (teacher 항목은 DB 를 수정하지 않고 로컬에만 기록).
  - `nid()` = `` `a_${Date.now()}_${rand36(4)}` ``.
- **핸들러**:
  - `toggleTeacher(id)`: `doneIds` 토글 + `saveDone`.
  - `toggleLocal(id)`: `local` 배열 map + `saveLocal`.
  - `removeLocal(id)`: 필터 + `saveLocal`.
  - `addLocal()`: `draft.trim()` 비어있지 않으면 push + clear.
- **집계**:
  - `totalDone = doneIds.filter(id => teacher.some(t=>t.id===id)).length + local.filter(l=>l.done).length`.
  - `total = teacher.length + local.length`.
- **UI**:
  - 상단 카운터: `text-[10.5px] text-slate-400 font-semibold` → `{totalDone}/{total} 완료`.
  - 둘 다 빈 상태: "가입한 반의 과제 공지가 없어요. 아래에서 직접 추가할 수 있어요." (`text-[11.5px] text-slate-400 text-center py-3`).
  - 리스트: `space-y-1 max-h-[180px] overflow-auto pr-1`.
  - 각 항목: 체크박스 `w-4 h-4 rounded border`, 완료 시 `bg-emerald-500 border-emerald-500 text-white` + `Check` 아이콘. 텍스트 완료 시 `line-through text-slate-400`.
  - teacher 행: 서브라인 `text-[10px] text-slate-400 truncate` → `{course_name} · 선생님 과제`.
  - local 행: hover 시 우측 `X` 삭제 버튼(`opacity-0 group-hover:opacity-100`, hover 시 `text-red-500 bg-red-50`).
  - 하단 입력: `Input h-8 text-[12px]` + `Enter` 로 추가 + emerald `Plus` 버튼(`h-8 w-8 rounded-md bg-emerald-500 hover:bg-emerald-600`, 빈 draft 시 `opacity-40`).
- **정책**: 이 모듈은 DB `course_notices` 를 수정하지 않는다 — 완료 상태는 순수 로컬(브라우저별). 학생이 로컬 과제를 추가해도 다른 기기/선생님에게 전달되지 않는다.

### 3.11 학생용 재사용 위젯 (기본 레이아웃 전용)

기본 레이아웃에 포함되지만 카탈로그에는 노출되지 않는 항목 — 학생 편집 패널에서 다시 추가할 방법이 없으므로 삭제 후 「기본값」 버튼으로만 복원 가능하다.

#### 3.11.1 `HeroInsightModule` (L)

세부 스펙은 T1 §2.9(1) 를 그대로 재사용. 학생 홈에서도 동일 컴포넌트, 동일 `generate-hero-insight` edge function, 동일 저장 위치(`profiles.hero_config / hero_cache`, localStorage fallback). 학생도 태그·직접 입력·빈도 설정을 사용할 수 있다.

#### 3.11.2 `PlatformFlowModule` — 학습 흐름 (`config.variant === "student"`, L)

- 헤더 좌측 라벨: "학습 흐름" / 서브: "각 단계를 클릭해 바로 시작하세요".
- 노드 3개(각 flex-1, 사이 커넥터 flex-1):
  1. `s_songs` "노래 아카이브" (Music, 초록 `#0a9b5e`, delay 0s, 라벨 "듣기") → `navigate("/songs")`.
  2. `s_public` "공개 수업 찾기" (BookOpen, 남색 `#4f52c8`, delay .6s, "둘러보기") → `navigate("/student-home#public-courses")`.
  3. `s_join` "반 참여" (GraduationCap, 앰버 `#d97706`, delay 1.2s, "내 반") → `setJoinOpen(true)` (JoinCourseDialog).
- 각 노드 카운트:
  - `s_songs`: `songs count where deleted_at is null` (공개 카탈로그 전체).
  - `s_public`: `courses count where owner_id is null and deleted_at is null`.
  - `s_join`: `course_members count where user_id = uid`.
- 노드 원형 아이콘 박스 `w-14 h-14 rounded-2xl` + 2px border + `dash-ripple` 애니메이션(2.5s ease-out delay infinite: scale 0.9→1.2, opacity 0.8→0).
- 커넥터: `h-[2px] bg:#eeeffe` + 60% 폭 그라디언트가 `dash-flowglow` (2s linear infinite, `i*0.4s` delay) 로 좌→우 흐름.
- 카드 배경/보더 색상은 각 노드의 `bg/border/text/ripple` 색 세트를 사용.
- `<JoinCourseDialog open={joinOpen} onOpenChange={setJoinOpen} />` 은 variant="student" 일 때만 렌더.

#### 3.11.3 `SongRecommendationModule` (M)

T1 참조. 학생 홈에서도 그대로 사용 — GPT 로 매일 자정 갱신되는 랜덤 노래 추천 카드. 학생/교사 차별 로직 없음.

---

## 4. Context — 데이터 계약 & 이벤트 버스

### 4.1 사용 RPC

- `get_my_joined_courses()` → `{ id uuid, name text, level text, deleted_at timestamptz }[]`. `SECURITY DEFINER`, 자기 자신 소속 반만 반환.

### 4.2 사용 테이블 (전부 RLS 보호)

| 테이블 | 학생 접근 | 용도 |
|---|---|---|
| `courses` | SELECT (본인 소속·공개) | 이름/레벨/`class_time`/`start_date` 조회 |
| `course_members` | SELECT self, INSERT self, DELETE self | 공개 반 가입·탈퇴 |
| `course_student_profiles` | SELECT (본인 `member_user_id`) | 참여 반 조회 |
| `course_calendar_items` | SELECT (본인 반), INSERT/UPDATE/DELETE self only | 캘린더 이벤트 |
| `custom_event_types` | SELECT/INSERT self | 캘린더 타입 팔레트 |
| `course_notices` | SELECT (본인 반) | 공지 |
| `notice_reads` | SELECT/UPSERT self | 읽음 상태 |
| `notification_replies` | via RepliesThread | 공지·글 답글 |
| `course_student_posts` | INSERT self | 선생님께 메시지 |
| `song_favorites` | SELECT/INSERT/DELETE self | 즐겨찾기 |
| `songs`, `song_analyses` | SELECT public | 즐겨찾기 카드 + 분석 다이얼로그 |
| `profiles` | SELECT (제한된 컬럼) | Hero 캐시, teacher_name |

### 4.3 실시간 & 이벤트 버스

- Postgres channel: `student-courses-<uid>` (courses UPDATE), `dash-cal-courses-<uid>` (courses *).
- Window 커스텀 이벤트:
  - `"student-courses:refresh"` — `StudentMyCoursesModule` 탈퇴/재가입, `PublicCoursesModule` 가입 후 브로드캐스트. 리스너는 두 모듈 모두.
  - `"course:updated"` — 반 정보 변경 시 캘린더 재로드 트리거.

### 4.4 네트워크 규칙

- 모든 Supabase 조회는 `fetchWithRetry(op, 3, 2000, 10000)` 로 래핑, `AbortSignal` 전파.
- 429 응답 → toast "요청이 너무 잦아요. 잠시 후 다시 시도해주세요.".
- 402 응답 → toast "크레딧이 부족합니다.".
- 실시간 채널·`window.addEventListener` 는 useEffect cleanup 에서 반드시 해제.

---

## 5. Non-Goals (S1 에서 다루지 않음)

- 공용 위젯 12종의 렌더 세부 → **T1a 재사용** (본 문서는 카탈로그 등록만).
- 장식 7종 + 재미 6종의 세부 → **T1b 재사용**.
- 교사 전용 위젯 8종(`ai_draft, stats, my_classes, recent_activity, student_messages, notices, hero_insight 편집 관리자 뷰, platform_flow 교사 variant`) — 학생 편집 패널에서 완전 비노출.
- 레거시 위젯 `particle_deco` — 카탈로그·기본 레이아웃 어디에도 노출 금지.
- 코스 상세 페이지 `/courses/:id` 의 학생 뷰 (별도 문서 S2).
- 온보딩 · 반 가입 초대 · 프로필 초기 세팅 (별도 문서 S3).
- 공용 노래 아카이브 페이지 (`/songs`) (별도 문서 S4).
- 학생 설정 (`/settings` 학생 분기) (별도 문서 S5).
- 학생 사용 가이드 (`/guide` 학생 분기) (별도 문서 S6).
- 학생 휴지통 (`/trash` 학생 분기) (별도 문서 S7).
- 홈 & 인증 흐름 (별도 문서 S8).

---

## 6. Acceptance Criteria

1. `/student-home` 미로그인 접근 → `/auth?redirect=%2Fstudent-home` 리다이렉트.
2. 학생 로그인 후 기본 6모듈이 순서대로 렌더: **Hero(L) → 학습 흐름(L, student variant) → 내 수업(M) → 공개 수업(M) → 오늘의 노래 추천(M) → 캘린더(M)**.
3. `#public-courses` 또는 `#my-courses` 해시로 진입 시 400ms 내에 해당 카드가 화면 상단으로 smooth 스크롤된다.
4. 「위젯 편집」 클릭 → 우측 280px 슬라이드 드로어 오픈, 각 프레임 우상단에 `S/M/L` 스위치 + `X` 삭제, 좌상단에 `GripVertical` 핸들, ring-1 indigo 강조가 나타난다.
5. 학생 편집 드로어에는 **🧩 공용 / ✨ 장식 & 애니메이션 / 🔮 재미 & 운세 / 🎓 학생 전용** 4개 섹션만 노출되고, 「👩‍🏫 교사 전용」 은 절대 렌더되지 않는다. `ai_draft` 등 교사 위젯은 아예 검색·드래그·추가 불가.
6. 카탈로그 카드 드래그 → 캔버스 아무 카드 위에 드롭 시 그 카드 인덱스 앞으로 삽입. 빈 영역에 드롭 또는 `+` 클릭 시 캔버스 끝에 추가.
7. 크기 변경 / 삭제 / 드래그 이동 / 추가 / 「기본값」 모두 200ms 이내 `localStorage.dashboard_layout_student` 에 반영된다. localStorage 에 `type === "particle_deco"` 인 인스턴스가 남아있으면 첫 렌더 시 자동 제거된다.
8. **내 수업** — 최대 6개 반. hover 시 `Trash2` 노출 → `AlertDialog` 확인 → `course_members.delete` → 로컬 즉시 제거 + `student-courses:refresh` 브로드캐스트. 실시간 채널로 반 정보 변경이 반영된다.
9. **공개 수업** — 4개씩 페이지네이션, 좌/우 화살표 + `N/M` 뱃지. 이미 가입한 반은 초록 `Check` 뱃지 표시. `Plus` 클릭 → `course_members.insert({role:"student"})` → 낙관 UI + toast + 리프레시 이벤트. 중복 가입 시 info toast.
10. **학습 기록** — 즐겨찾기 노래 / 참여 반 각각 실제 count 를 800ms cubic-out 이징으로 CountUp.
11. **선생님께 메시지** — 참여 반 자동 로드(중복 제거). 반 0 → Textarea disabled + 「먼저 강의에 참여해 주세요」 Tooltip. 반 1 → Select 숨김, 이름 노출. 반 ≥2 → shadcn Select. 전송 성공 시 `course_student_posts.insert({type:"메시지", visibility:"private", status:"pending"})` + Textarea clear + toast.
12. **즐겨찾기 노래** — 최대 6장 2열 카드, YouTube 썸네일(`mqdefault.jpg`) + 제목/artist. 카드 클릭 시 `song_analyses.maybeSingle()` 로드 후 `SongAnalysisDialog readOnly` 오픈. 분석 없음 → `analysisNotFound=true` 상태 명확 표시.
13. **공지 확인** — 참여 반의 최근 50개, `notice_reads` 로 읽음 관리. 미읽음 수 뱃지. 카드 클릭 시 옵티미스틱 read upsert (`onConflict:"user_id,notice_id"`) + 펼침 → `RepliesThread compact` 로 답글 가능. `shortTime` 포맷 준수.
14. **체크리스트 3종** — 각각 지정된 localStorage 키에 저장. `daily_plan` 은 자정 지나면 새 키(`student_daily_plan_YYYY-MM-DD`)로 자동 초기화. `weekly_plan` 은 ISO 주 키. `learning_goal` 은 월/주 무관 지속.
15. **캘린더** — 학생이 참여한 반만 자동 로드. 월 그리드에 dot 최대 3개 표시(코스 색 + 이벤트 타입 색). 오늘 = `bg-dash-navy` 원, 일요일 = `text-destructive`. 아젠다에서 `class` 반복 일정은 편집 불가·「숨기기」만 가능, `course_calendar_items` 자기 소유 행은 편집·삭제 가능. 커스텀 유형 추가는 12색 팔레트로. `hidden_calendar_courses` localStorage 로 반별 숨김/복원. 실시간 채널(`dash-cal-courses-<uid>`) 로 반 변경 즉시 반영.
16. **이번 주 강의** — `course_student_profiles` 로 소속 반 산출 후 이번 주(월~다음 월요일) `course_calendar_items` + `courses.class_time` 반복 수업을 확장·병합해 시간순 상위 5개를 렌더. 배지 = `M/D 요일`, 시간 = `HH:mm` 또는 "종일". 이모지 매핑 (`class=📚, assignment=📝, exam=📊, event=📅`).
17. **과제 현황** — 선생님 과제(`course_notices` 중 `category` 가 assign/과제 포함)와 로컬 과제(`student_assignments_local`)를 한 리스트로 병합. teacher 항목 완료는 `student_assignments_done_teacher` 로컬 배열에만 저장하고 DB 를 건드리지 않는다. 상단 `{done}/{total} 완료` 카운터, 하단 `Enter`/`+` 로 로컬 과제 추가, hover 시 로컬 항목만 `X` 삭제 가능.
18. 모든 Supabase 조회는 `fetchWithRetry` 로 래핑되고 언마운트 시 취소된다. 실시간 채널·window 이벤트 리스너는 반드시 정리된다.
