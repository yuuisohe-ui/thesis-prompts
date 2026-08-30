# T2 — 교사 워크스페이스 `/workspace` (4 대 섹션 · 페이지네이션 · Fork · 드래그)

> 4.2.2 教师工作台。 논문 4.2 教师端 두 번째 재현 프롬프트. 한 파일 안에서 `Workspace` 페이지, 4 개 하위 섹션, `WorkflowBanner`, `AIDraftAlert`(비활성 잔재), `ConnectLessonPlanDialog`, 공용 `CoverImage` 카드, `wsBtn` 토큰, 그리고 공용 아이템 fork 헬퍼를 모두 재현할 수 있도록 설계되어 있다.

---

## 1. Identity — 이 프롬프트로 만드는 것

**이름**: 교사 워크스페이스 (Workspace).
**경로**: `/workspace` (`RequireAuth` + `AppLayout` 안).
**한 줄 정의**: "**공용 템플릿 → 내 반 → 내 강의안 → 학생**" 4 단 세로 스택으로, 교사가 자료를 고르고(fork), 반을 만들고, 강의안을 연결하고, 학생을 초대·관리하는 **모든 진입 허브**.
**대상 역할**: `teacher` / `admin` 만. 학생은 `RequireAuth` 통과해도 여기 오지 않는다 (사이드바에서 링크 자체를 감춘다).
**연결 대상**:
- 위쪽: `AppSidebar` "워크스페이스" 링크, `WorkflowBanner` 4 STEP 앵커.
- 아래쪽: 반 상세 `/courses/:id` (내부 `LessonPlanDetail` 임시 라우팅으로 강의안도 함께 연다), `SharedSong` 공유 링크, `CreateLessonPlanDialog`, `CreateCourseDialog`, `EditCourseDialog`, `BulkStudentDialog`.

---

## 2. Instructions — 만드는 규칙 (반드시 지킬 것)

### 2.1 페이지 골격 `src/pages/Workspace.tsx`

- `bg #f4f3f0` 위에 **큰 흰색 라운드 카드 4 장** (`rounded-[24px]`, `border 1px rgba(0,0,0,0.07)`, `box-shadow 0 1px 3px rgba(0,0,0,.06)`) 을 세로 스택.
- 각 카드 안 순서: **① PublicTemplatesSection → ② MyClassesSection → ③ MyLessonsSection → ④ StudentSection**.
- 카드 위 최상단에 `WorkflowBanner` (다크 그라디언트, 4 STEP 앵커 카드).
- URL hash 지원: `#tmpl-section` / `#class-section` / `#lessons-section` / `#student-section`. 진입 시 150ms 대기 후 `scrollIntoView({behavior:"smooth"})`.
- `dialogStep` state 로 `WorkflowBanner` 안의 "자세히" 모달 4 개를 열고, `preselectPlanId` state 로 강의안 → 반 생성 프리셀렉트 흐름을 관리.
- `LessonPlanDetail` 을 **모달이 아닌 페이지 대체**로 렌더한다: `selectedPlan` 이 있으면 `<LessonPlanDetail plan={} onBack={}/>` 만 리턴하고 나머지는 감춘다.
- 최초 mount 에서 `courses.cover_image_url` + `lesson_plans.cover_image_url` 을 한 번 뽑아 `seedUsedCoverUrls()` 로 Pixabay 재선택 캐시를 시딩 (같은 표지 중복 방지).
- `classesRefresh` counter 를 두고, 하위 섹션 fork/생성 시 이 값을 ++ 하여 `refreshKey` prop 으로 흘려 3 개 섹션을 동시에 재조회.

### 2.2 4 STEP 상단 배너 `WorkflowBanner.tsx`

- `linear-gradient(135deg, #1a2340 0%, #252d50 40%, #1e2d45 100%)` + 3 겹 radial-gradient 오버레이 + 24px 격자 도트 패턴.
- 4 개 STEP 카드는 `grid-cols-1 sm:grid-cols-2 lg:grid-cols-4`, 반투명 유리 `bg-white/[0.04] hover:bg-white/[0.08]`.
- 각 STEP: `n` (01–04), `title`, `sub`, `anchor`, `detail: string[3]`. 카드 클릭 → `scrollToAnchor(anchor)`. "자세히" 클릭 → `setDialogStep(i)` (버블 차단 `stopPropagation`).
- 4 STEP 텍스트 (한국어, 그대로 사용):
  1. `01 강의안 고르기` · `공용 템플릿 선택 또는 AI로 직접 제작` · anchor `tmpl-section`.
  2. `02 AI 초안 확정` · `확정해야 반에 연결할 수 있어요` · anchor `ai-alert`.
  3. `03 반 만들기` · `강의안을 반에 연결하고 반 링크 생성` · anchor `class-section`.
  4. `04 반 링크로 학생 초대` · `링크 공유 또는 CSV 일괄 등록` · anchor `student-section`.
- `Dialog` 는 3-bullet `detail` 리스트 + "해당 섹션으로 이동" 액션 하나.

### 2.3 공용 템플릿 섹션 `PublicTemplatesSection.tsx` — Fork 캐러셀

- 데이터: `lesson_plans` / `courses` 두 테이블에서 `owner_id IS NULL AND deleted_at IS NULL` 인 것을 최신순 20 개씩 병렬 조회 (`fetchWithRetry` 필수, `plan_type != 'ai_generated'`).
- 탭 2 개 (하나만 활성): `강의안` / `반 템플릿`. 스위치는 12px 패딩 `bg-[#eeecea]` 트랙 안의 흰 캡슐.
- 캐러셀:
  - 페이지당 보이는 카드 수 = `useVisibleCount()`: 모바일/`<640px` = 2, `<1280px` = 3, 그 외 = 4.
  - `STEP = 3` (한 번에 3 장씩 이동), `translateX(%)` 로 600ms ease 슬라이드, `overflow-hidden mx-11`, 좌우 원형 화살표 버튼 절대 배치.
  - 항목 폭 = `calc(${100/visible}% - ${12*(visible-1)/visible}px)` 로 12px gap 을 정확히 상쇄.
- **강의안 카드** `LessonCard`:
  - `CoverImage table="lesson_plans"` (h 108px), 좌상단 `공용` Badge, 하단 그라디언트 위 흰색 12px bold 제목 1 줄 clamp.
  - 하단 액션: `course_id` 가 있으면 `wsBtn.green` "반 링크 보기 →" (반 상세로 이동), 없으면 `wsBtn.indigo` "이 강의안으로 반 만들기 →" (`onMakeCourseFromPlan` → `preselectPlanId` 지정 후 `CreateCourseDialog` 열기).
  - 우상단 3-dot: **"내 강의안에 추가"** 메뉴 → `forkPublicItem("lesson_plans", id)` → 성공 시 toast `내 강의안에 추가되었습니다` + `onForked()`. `busy` 상태에서는 스피너 오버레이 + 3-dot 이 Loader2 로 교체.
- **반 카드** `CourseCard`:
  - 같은 커버 규격. 하단 2 버튼: `wsBtn.outline` "반 내용 보기" (반 상세로 이동) / `wsBtn.indigo` "공유 링크 복사" (`${origin}/shared/course/${share_token}` 를 클립보드 복사, toast).
  - 3-dot 메뉴 "내 반에 추가" → `forkPublicItem("courses", id)` → 반환된 `newId` 로 즉시 `/courses/:newId` 이동.
- 제목 우측 상단에 `wsBtn.outline` "✨ AI로 직접 제작" 버튼이 항상 노출 → `CreateLessonPlanDialog` 오픈.

### 2.4 내 반 섹션 `MyClassesSection.tsx` — 페이지 6 · 드래그 · 게시

- 데이터: `courses.*` where `deleted_at IS NULL`, 정렬 `sort_order asc, created_at desc`. `fetchHiddenIds("courses")` 로 걸러낸 뒤 `owner_id === user.id` 인 것만 = `mine`. 관리자여도 여기서는 자기 것만.
- 학생 수: `course_members` 를 `mine.id` in-list 로 뽑아 `role='student'` 만 count 해 `student_count` 를 실시간 덮어씀.
- 연결된 강의안·주차 진척도: `lesson_plans.course_id IN (mine.id)` → `lesson_weeks(lesson_plan_id, is_generated)` 를 그룹 카운트해 `{total, generated}` 로 축적, `planByCourse[course_id]` 에 저장.
- **빈 상태**: `mine` 이 0 이면 공용 반 (`owner_id IS NULL`) 중 랜덤 1 개를 `isRecommended: true` 로 붙여 표시하고, 그 카드는 편집/삭제/공개 토글 감춤.
- 그리드: `grid-cols-1 sm:grid-cols-2 xl:grid-cols-3 gap-4`.
- 페이지네이션: `PAGE_SIZE = 6`, 페이지 수 > 1 일 때만 우상단에 `< n / m >` 컴팩트 컨트롤. **마지막 페이지에서 아이템이 6 개 미만**이면 그 페이지 마지막 슬롯에 dashed "+ 새 반 만들기" 타일을 렌더 (`data-tour="courses-create"`).
- 드래그 (`@dnd-kit`): `PointerSensor activationConstraint.distance = 8`, `rectSortingStrategy`. drop 시 배열 재배열 → `courses.upsert([{id, sort_order}])` 로 순서 저장, 실패 시 `fetchCourses()` 롤백.
- 카드 (`SortableClassCard`):
  - `rounded-[18px]`, 커버 80px, 위에 `rgba(10,5,30,.5)` 어두운 오버레이, 좌하단에 흰색 14px extrabold 반 이름 + 그 아래 10px `class_time || semester || "시간 미설정"`.
  - 우상단 hover 노출 도구 3 개: 드래그 손잡이 `GripVertical`, "3-dot 메뉴" (`공개↔비공개 전환` / `수정` / `삭제`), 오른쪽 위 학생 수 pill `Users {count}`.
  - 본문:
    - **강의안 연결됨** → 인디고 박스 안에 `FileText` 아이콘 + 제목 + 얇은 진척도 바 `bg-[#4f52c8]`, 우측 `generated/total 주차` 텍스트.
    - **미연결** → dashed 박스 + `wsBtn.indigo` "연결하기 →" (`onConnectCourseToPlan(course)` → `ConnectLessonPlanDialog` 오픈).
    - 하단 2 버튼 flex-1: `wsBtn.outline` "반 상세 보기" (→ `/courses/:id`), `wsBtn.indigo` "반 링크 복사" (성공 시 1.5s 동안 `Check` 아이콘 + "복사됨").
- **공용 반 등록 (`is_public` toggle)**: 이미 공개면 즉시 `update {is_public:false}`. 비공개 → 공개는 확인 `AlertDialog` 를 먼저 띄우고, 공개 성공 후 별도 "감사합니다!" 다이얼로그로 사회 기여 감사 메시지.
- **공용 반 편집 시 자동 fork**: `handleEditClick` 에서 `!course.owner_id && !isAdmin` 이면 `forkPublicItem("courses", id)` 를 먼저 태우고, 새 반 row 를 `EditCourseDialog` 로 연다. 관리자는 원본을 그대로 편집.
- **삭제**: `AlertDialog` 확인 후 `rpc("move_to_trash", {_table:"courses", _id})` — 7 일 후 자동 파기. 성공 시 리스트에서 즉시 제거.

### 2.5 내 강의안 섹션 `MyLessonsSection.tsx` — 통합 리스트 · 페이지 10 · 인라인 편집

- 데이터: `lesson_plans` 를 `deleted_at IS NULL` 로 전량 최신순 조회 → `fetchHiddenIds("lesson_plans")` 로 감춰진 공용 항목 제거 → `owner_id === user.id` 만 `mine`.
- **드래그 순서 + localStorage 지속화**: `@dnd-kit/core` + `SortableContext` + `verticalListSortingStrategy`, `PointerSensor` `activationConstraint.distance = 6` (모바일 탭 오탐 방지). drop 시 `arrayMove` 후 `localStorage.setItem("lesson_plans_order_v1", JSON.stringify(ids))` 로 로컬 저장 (DB 저장 없음). 최초 로드 시 저장된 id 순서를 우선 적용하고, 없는 id 는 뒤로 밀어 `created_at desc` 로 이차 정렬. **정식 → 초안 순으로 합병한 뒤 순서를 적용**한다.
- 주차 통계: 내 plan.id 전체에 대해 `lesson_weeks(lesson_plan_id, is_generated)` 를 그룹 count → `weekStats[planId] = {total, generated}`.
- **분류 & 통합 렌더**: `plan_type === 'ai_generated'` = 초안 (`DraftPlanRow`, 옅은 amber), 그 외 = 정식 (`FormalPlanRow`, 회색). **동일한 세로 리스트에 정식 먼저, 초안 뒤로 합병**하여 하나의 `SortableContext` 안에 렌더한다.
- 페이지네이션: `PAGE_SIZE = 10`, `< n / m >` 컴팩트 컨트롤을 리스트 카드 헤더 내부 (섹션 헤더 밖이 아니라) 에 배치. 정식 + 초안 합병 후 통합 페이지네이션.
- **레이아웃**: `grid-cols-1 lg:grid-cols-[1fr_240px]` — 왼쪽 리스트 카드, 오른쪽 dashed "AI 초안 안내" 사이드박스 (배경 `#fffbeb`, 테두리 `#fde68a`, 텍스트 `#92400e`, 헤더에 `Sparkles`).
- 리스트 하단 **2 개 dashed 버튼 flex-1**: `+ 새 강의안 추가` (indigo dashed) / `✨ AI 강의안 생성` (amber dashed). 둘 다 지금은 `onOpenCreateLesson()` 을 호출한다 (`CreateLessonPlanDialog` 안에서 모드 분기).
- `FormalPlanRow`:
  - 좌측 `GripVertical` 손잡이 → 인디고 `FileText` 8×8 스퀘어 → 제목 (13px semibold) + `Badge outline` level + `추천` 배지 (빈 상태 추천 시) + 우측 회색 `${total}주차` 텍스트.
  - 우측 툴바: `course_id` 있으면 초록 pill `반 연결됨`, 없으면 `wsBtn.indigo` "반에 연결하기 →" (`onConnectPlanToCourse(id)` → `CreateCourseDialog` 로 preselect).
  - Hover 시 `Pencil` / `Trash2` 도구 노출. **Pencil → 인라인 편집** (`Input` 제목 + `Select` LEVELS 6 개: `HSK 1-3 / 4-6 / 7-9`, `TOPIK 1-2 / 3-4 / 5-6`). `Check` 저장 시:
    1. 공용 plan (`!owner_id`) 이면서 비-관리자면 먼저 toast `개인 사본 생성 중…` → `forkPublicItem("lesson_plans", id)` 로 사본 id 확보,
    2. `lesson_plans.update({title, level})` 을 사본(또는 원본) id 에 적용,
    3. 변경사항이 있으면 `pendingChange` state 를 채워 **재분석 확인 `AlertDialog`** 을 띄운다.
  - **삭제**: `confirm()` 확인 후 `rpc("move_to_trash", {_table:"lesson_plans", _id})` — 7 일 후 자동 파기. 성공 시 리스트에서 즉시 제거하고 `saveOrder` 를 갱신 id 배열로 다시 쓴다.
- **재분석 흐름** (`handleConfirmRegenerate`): 기존 `lesson_weeks` 전부 삭제 → 1주차의 `plan_meta` (`class_duration_min` / `mid_exam_week` / `final_exam_week` / `songs_per_week` / `week_count`) 로 `functions.invoke("generate-lesson-plan", { action:"generate_outline", … })` 호출 → 성공 시 옛 `lesson_plans` row 삭제 (edge func 이 새 row 로 재생성) → `fetchData()`. 로딩 중에는 `아니오/예` 버튼 disable + `Loader2`. AI 생성 파이프라인 자체의 상세 계약은 **T3b** 문서에서 별도 재현한다.
- `DraftPlanRow`:
  - 배경 `bg-amber-50/50`, 아이콘 `Sparkles`, 배지 `AI 초안` (amber).
  - 우측 `wsBtn.amber` **"정식으로 확정하기"** → `lesson_plans.update {plan_type:'custom'}` → 성공 toast, 리스트에서 amber → gray row 로 즉시 전환 (재조회 불필요).
  - Hover 시 `Trash2` 노출, 삭제는 동일하게 `move_to_trash` RPC.


### 2.6 학생 관리 섹션 `StudentSection.tsx` — 반별/전체 · 검색 · CSV

- 상단: 아이콘 + `학생 관리` + 우측 `전체 N명` 알약, 우상단 `wsBtn.indigo` "학생 목록 일괄 생성" 버튼 → `BulkStudentDialog` (`data-tour="students-csv"`).
- 탭 2 개 캡슐 스위치: `반별 보기` / `전체 학생`.
- **반별 보기**:
  - `내 반` 과 완전히 같은 코스 목록을 사용 (empty state 는 동일한 공용 추천 반).
  - `Accordion type="multiple"` 로 각 반이 접을 수 있는 행. 헤더: 반 이름 + `추천` 배지 + `Badge outline N명` + (있으면) `class_time`, 우측에 `반 링크 복사` (indigo) · `수정` · `삭제` (추천 반은 감춤).
  - 펼침 내용:
    - 추천 반 → "공용 반 템플릿이에요…" 안내 + `wsBtn.outline` "반 내용 보기".
    - 학생 0 명 → `EmptyClassCards` 2 열: **"반 링크 공유"** (인디고, 링크 복사) / **"학생 목록 일괄 생성"** (초록, `BulkStudentDialog`).
    - 학생 있음 → `StudentRowItem` 세로 리스트. 각 행: `Avatar` (avatar_url > emoji > initials) + 이름 + `학번 · 학과 · 수준` join + 우측 `가입`/`수동` 배지.
- **전체 학생**:
  - 좌상단 최대폭 sm 검색창 (이름/학번/학과/과정명 소문자 포함 매치).
  - 결과 0 → `EmptyStudents` 2 카드 (첫 반 링크 복사 / CSV 업로드).
  - 결과 있음 → `Table` (columns: 아바타 · 이름 · 반 · 학번 · 학과 · 수준 · 가입/수동).
- 데이터:
  - `course_student_profiles.*` `deleted_at IS NULL` 최신순, `fetchHiddenIds("course_student_profiles")` 필터.
  - `avatar_url` 이 있으면 `img rounded-full`, 없으면 `emoji`, 그것도 없으면 `👤`.
- 반 편집/삭제는 `MyClassesSection` 과 동일한 다이얼로그를 재사용 (독립 인스턴스).

### 2.7 강의안 ↔ 반 연결 `ConnectLessonPlanDialog.tsx`

- `courseId` prop 이 열림 트리거. 오픈 시 현재 코스의 `start_date` 를 프리로드해서 `Input type="date"` 프리셀렉트.
- 후보: `lesson_plans` `plan_type != 'ai_generated' AND deleted_at IS NULL AND (course_id IS NULL OR course_id === courseId)` (즉 다른 반에 이미 연결된 것은 제외).
- 선택된 plan 이 공용이면 먼저 `forkPublicItem` → 사본 id 를 `update {course_id: courseId}`. `start_date` 가 입력됐다면 `courses.update {start_date}`.
- 성공 시 toast + `onConnected()` + 다이얼로그 닫기. 저장 중에는 X 버튼과 배경 클릭 모두 무시.

### 2.8 공용 fork 헬퍼 `src/lib/ownership.ts`

- `forkPublicItem(table, itemId)` 는 Postgres `rpc("fork_public_item", {_table, _id})` 를 그대로 호출해 **새 row id 를 반환**한다.
- `fetchHiddenIds(table)` 는 `hidden_public_items` 를 `user_id` + `table_name` 으로 조회해 `Set<string>` 반환. 로그아웃 상태면 빈 Set.
- `hidePublicItem(table, id)` 는 `upsert(hidden_public_items, onConflict:"user_id,table_name,item_id")`.
- 이 파일은 워크스페이스 4 개 섹션 모두에서 재사용된다 (**추가/편집/렌더 필터** 세 지점에서 동일 함수 3 개만 쓰인다는 것이 재현 핵심).

### 2.9 표지 이미지 `CoverImage.tsx` — 재현 최소 스펙

- `existingUrl` 있으면 즉시 표시, 없으면 `ensureCoverUrl` 을 태워 Pixabay 에서 `primaryKeyword` → `fallbackKeywords` 순으로 검색한 뒤 signed URL 을 Storage 에 캐시.
- `onError` 리커버리: pixabay.com 원 URL 이면 `migrateLegacyPixabayUrl` 로 Storage 캐시로 이관, 그 외에는 `refreshCoverUrl` 로 다른 이미지 재선택. 같은 URL 을 두 번 리트라이하지 않도록 `recoveredRef: Set<string>` 로 가드.
- 관리자(`isAdmin`) 에게만 우상단 카메라 버튼 노출 → `refreshCoverUrl` 강제 재선택 (`hideRefresh` 로 죽일 수 있음).

### 2.10 시각 토큰 `tokens.ts` — **모든 워크스페이스 버튼의 유일한 소스**

- `wsBtnBase` = `inline-flex items-center justify-center gap-1 px-2.5 py-[5px] text-[11px] font-semibold rounded-[7px] transition select-none whitespace-nowrap disabled:opacity-50 disabled:pointer-events-none`.
- 변형: `outline` (흰 배경 + 회색 테두리), `indigo` (`#eeeffe / #3739a8 / #c7caff`), `indigoSolid` (`#4f52c8 white`), `green` (`#e7f7ee / #0a7f4d / #bbe9d0`), `amber` (`#fef3c7 / #92400e / #fde68a`), `ghost` (배경 없음).
- **크고 채도 높은 solid fill 은 금지** (`indigoSolid` 는 아주 드물게만 사용). 워크스페이스 카드는 모두 tinted 또는 outline 로 통일.
- `sectionTitleCls` = `text-[11px] uppercase tracking-[0.18em] font-semibold text-[#aaa8b8]` — 리스트 카드 내부 소제목 전용.

### 2.11 안전 & 접근성

- 모든 파괴적 액션(반 삭제, 게시)은 shadcn `AlertDialog` 를 반드시 통한다. `move_to_trash` RPC 로 소프트 삭제 → 7 일 후 자동 파기 (`/trash` 페이지에서 복원 가능).
- fork 중에는 카드에 `bg-white/70 backdrop-blur Loader2` 오버레이 + 3-dot → Loader2 로 교체하고, 중복 클릭을 `busy` state 로 완전 차단.
- 드래그 핸들은 항상 별도 `<button>` 으로 두고 `stopPropagation`. 카드 전체를 드래그 소스로 만들지 말 것 (모바일에서 스크롤과 충돌).
- 반 링크 복사 후 toast + 아이콘 스왑(`Check` 1.5s) 로 시각 피드백. `navigator.clipboard` 실패는 무시하지 말고 toast.

### 2.12 삭제/미사용 항목

- `AIDraftAlert.tsx` 파일은 리포지토리에 남아 있지만 **현재 워크스페이스에서 렌더하지 않는다.** (`Workspace.tsx` 안에 `AIDraftAlert removed — drafts now appear inline in MyLessonsSection` 이라는 주석만 남음.) 재현 시 이 파일을 만들 필요 없다. 초안은 `MyLessonsSection` 의 `DraftPlanRow` 로 통합되어 "정식으로 확정하기" 버튼을 그 자리에서 노출한다.
- `WorkflowBanner` STEP 02 의 앵커 `#ai-alert` 는 위 이유로 실제 target 이 없다. 스크롤은 미실행되지만 다이얼로그 안내는 유효하므로 STEP 자체는 유지.

---

## 3. Examples — 골격 스니펫

### 3.1 페이지 컨테이너

```tsx
// src/pages/Workspace.tsx
const cardCls = "bg-white rounded-[24px] p-4 sm:p-6";
const cardStyle = { border: "1px solid rgba(0,0,0,0.07)", boxShadow: "0 1px 3px rgba(0,0,0,0.06)" };

return (
  <div className="flex flex-col gap-5 ..." style={{ background: "#f4f3f0" }}>
    <WorkflowBanner dialogStep={dialogStep} setDialogStep={setDialogStep} />
    <div className={cardCls} style={cardStyle}><PublicTemplatesSection ... /></div>
    <div className={cardCls} style={cardStyle}><MyClassesSection    refreshKey={classesRefresh} ... /></div>
    <div className={cardCls} style={cardStyle}><MyLessonsSection    refreshKey={classesRefresh} ... /></div>
    <div className={cardCls} style={cardStyle}><StudentSection      refreshKey={classesRefresh} ... /></div>
    {/* CreateLessonPlanDialog · CreateCourseDialog · ConnectLessonPlanDialog */}
  </div>
);
```

### 3.2 fork 헬퍼 사용 (버튼 1 개)

```tsx
const [busy, setBusy] = useState(false);
const onFork = async () => {
  setBusy(true);
  try {
    const newId = await forkPublicItem("courses", course.id);
    toast({ title: "내 반에 추가되었습니다" });
    navigate(`/courses/${newId}`);
  } catch (e: any) {
    toast({ title: "추가 실패", description: e?.message, variant: "destructive" });
  } finally { setBusy(false); }
};
```

### 3.3 캐러셀 수식

```
visible = mobile || w<640 ? 2 : w<1280 ? 3 : 4
itemWidth = calc(${100/visible}% - ${12*(visible-1)/visible}px)
translate  = -(page * 100/visible)%
onPrev = setPage(max(0, page - 3))
onNext = setPage(min(maxPage, page + 3))    where maxPage = max(0, count - visible)
```

### 3.4 학생 count 재계산

```ts
const { data: members } = await supabase.from("course_members")
  .select("course_id, role").in("course_id", mineIds).eq("role","student");
const countMap: Record<string, number> = {};
for (const m of (members ?? [])) countMap[m.course_id] = (countMap[m.course_id] ?? 0) + 1;
for (const c of mine) c.student_count = countMap[c.id] ?? c.student_count ?? 0;
```

### 3.5 재분석 트리거

```ts
await supabase.from("lesson_weeks").delete().eq("lesson_plan_id", planId);
await supabase.functions.invoke("generate-lesson-plan", {
  body: { action: "generate_outline", title, level, additional_notes, output_lang: output_lang || "ko",
          week_count: totalWeeks, class_duration_min: meta.class_duration_min || 90,
          mid_exam_week: meta.mid_exam_week, final_exam_week: meta.final_exam_week,
          songs_per_week: meta.songs_per_week || 2 },
});
await supabase.from("lesson_plans").delete().eq("id", planId);
```

---

## 4. Context — 의존 파일과 데이터 계약

### 4.1 파일 트리 (신규/수정)

```
src/pages/Workspace.tsx
src/lib/ownership.ts          # forkPublicItem / fetchHiddenIds / hidePublicItem
src/components/workspace/
  ├─ WorkflowBanner.tsx
  ├─ PublicTemplatesSection.tsx
  ├─ MyClassesSection.tsx
  ├─ MyLessonsSection.tsx
  ├─ StudentSection.tsx
  ├─ ConnectLessonPlanDialog.tsx
  ├─ CoverImage.tsx
  └─ tokens.ts               # WS, wsBtn, sectionTitleCls
```

의존하는 기존 파일 (P 시리즈에서 이미 만든 것을 그대로 사용):
- `src/components/lessons/CreateLessonPlanDialog.tsx` (T3a/T3b 에서 상세 재현)
- `src/components/courses/CreateCourseDialog.tsx`, `EditCourseDialog.tsx` (T4/T5 에서 상세 재현, 여기서는 `open/onOpenChange/preselectPlanId/onCreated/onSaved` 만 계약)
- `src/components/students/BulkStudentDialog.tsx` (T6 에서 상세 재현)
- `src/components/lessons/LessonPlanDetail.tsx` (T3a 에서 상세 재현)
- `src/hooks/useAuth`, `src/hooks/use-toast`, `src/lib/fetchWithRetry`, `src/lib/pixabay`.

### 4.2 DB 계약

| 테이블 | 필요한 컬럼 | 필터 |
|---|---|---|
| `courses` | `id, name, level, semester, class_time, description, introduction, share_token, student_count, created_at, sort_order, owner_id, cover_image_url, is_public, start_date, deleted_at` | `deleted_at IS NULL` |
| `course_members` | `course_id, user_id, role` | `role='student'` 로 count |
| `lesson_plans` | `id, title, level, plan_type, status, created_at, course_id, owner_id, additional_notes, output_lang, deleted_at, cover_image_url` | `deleted_at IS NULL` |
| `lesson_weeks` | `lesson_plan_id, is_generated, week_number, content` | 그룹 count / 재분석 시 delete |
| `course_student_profiles` | `id, full_name, student_number, department, avatar_url, member_user_id, language_level, study_years, gender, emoji, course_id, created_at, deleted_at` | `deleted_at IS NULL` |
| `hidden_public_items` | `user_id, table_name, item_id` | fork 대신 감출 때 |

### 4.3 필요한 RPC / Edge Function

- `fork_public_item(_table text, _id uuid) returns uuid` — 소유자 없는 row 를 깊은 복사해 새 id 반환. `courses` fork 시 `course_home_blocks`, `lesson_plans` fork 시 `lesson_weeks` 도 복사되어야 UI 가 정상 동작.
- `move_to_trash(_table text, _id uuid) returns void` — `deleted_at = now()` 만 세팅. 7 일 후 `purge_expired_trash` cron 이 실제 삭제.
- Edge Function `generate-lesson-plan` (`action: "generate_outline"`) — **T3b** 에서 상세 계약.

### 4.4 RLS 필수 정책 요약

- `courses`, `lesson_plans`: `owner_id IS NULL` (공용) 은 모두 SELECT 가능; `owner_id = auth.uid()` 는 CRUD 가능; `is_public = true` 는 공용 목록 노출용.
- `course_members`: 학생은 자기 row 만, 교사는 자기가 소유한 `course_id` 에 속한 모든 row 를 SELECT.
- `course_student_profiles`: 교사는 소유한 `course_id` 만 CRUD, 학생은 자기 `member_user_id` 프로필만 CRUD (P4 에서 완전 재현). `hidden_public_items`: `user_id = auth.uid()` 만.

---

## 5. Acceptance — 재현이 끝났다고 판정하는 체크리스트

- [ ] `/workspace` 진입 시 다크 그라디언트 배너 + 4 개 흰 라운드 카드가 세로로 렌더된다.
- [ ] STEP 카드 클릭이 해당 앵커로 부드럽게 스크롤되고, "자세히" 는 3-bullet 다이얼로그를 연다.
- [ ] 공용 강의안 캐러셀이 데스크톱 4 · 태블릿 3 · 모바일 2 개씩 보이고, 3 개씩 이동하며 좌우 화살표가 첫/끝 페이지에서 자동 disable 된다.
- [ ] 공용 카드 3-dot "내 강의안에 추가" / "내 반에 추가" 클릭 시 스피너가 도는 동안 재클릭 불가, 성공 시 toast + `refreshKey` 증가로 3 개 섹션이 동시 갱신된다.
- [ ] `내 반` 이 0 개일 때 랜덤 공용 반 1 개가 `추천` 배지로 자리를 지키고, 그 카드는 편집/삭제/공개 토글이 감춰진다.
- [ ] 반 카드 드래그 순서가 새로고침 후에도 유지된다 (`sort_order` DB 저장).
- [ ] 반 카드 3-dot "공개" → 확인 → "감사합니다!" 2 단 다이얼로그가 순서대로 뜨고, 성공 후 카드에 공개 상태가 반영된다.
- [ ] "반 링크 복사" 클릭 시 클립보드에 `${origin}/shared/course/${share_token}` 이 실제로 들어가고 1.5 초간 `복사됨` 상태를 표시한다.
- [ ] 공용 반의 편집(연필) 클릭이 자동 fork 후 새 반에 대한 `EditCourseDialog` 를 열고, 관리자 계정에서는 원본을 직접 편집한다.
- [ ] `내 강의안` 리스트가 정식/초안 순서로 통합 렌더되고, 초안 행은 옅은 amber, 정식 행은 회색이며, drag 순서가 localStorage 에 저장된다.
- [ ] 정식 강의안 인라인 편집 후 제목/레벨이 바뀌면 "다시 분석할까요?" 다이얼로그가 뜨고, "예" 시 edge function 이 호출되어 새 15 주차가 생성된다.
- [ ] 초안 "정식으로 확정하기" 클릭 시 즉시 amber row 가 gray row 로 바뀌고, 그 강의안이 이제 "반에 연결하기" 대상이 된다.
- [ ] `학생 관리` 반별 탭이 `내 반` 과 동일한 반 목록을 (같은 추천 반까지) 보여주고, 각 반 아코디언 안 학생 리스트가 정확히 표시된다.
- [ ] 학생 0 명 반 아코디언 안에서 "반 링크 공유"와 "학생 목록 일괄 생성" 2 개 카드가 뜨고 각각 링크 복사 / `BulkStudentDialog` 오픈이 정상.
- [ ] "전체 학생" 탭의 검색이 이름/학번/학과/과정명 4 개 필드에 대해 소문자 포함 매치로 즉시 필터링된다.
- [ ] `ConnectLessonPlanDialog` 에서 공용 강의안 선택 후 "연결하기" 를 눌렀을 때 사본이 만들어지고, 옵션의 `start_date` 가 반에 저장되며 이후 주차 계산이 이 날짜부터 시작된다.
- [ ] `WorkflowBanner` STEP 02 앵커(`#ai-alert`) 는 no-op scroll 이지만 "자세히" 다이얼로그는 여전히 열린다 — 코드에서 이 위화감이 제거되어 있다면 감점.
- [ ] 데스크톱 1440px · 태블릿 768px · 모바일 375px 세 지점에서 4 카드 그리드가 각각 3/2/1 열로 재구성되고, 페이지네이션·캐러셀 조작이 모두 동작한다.

---

**끝.** 이 문서만으로도 `Workspace.tsx` + `src/components/workspace/*` + `src/lib/ownership.ts` 를 처음부터 다시 짤 수 있어야 한다. 이 문서에서 언급하지 않은 다이얼로그(`CreateLessonPlanDialog`, `CreateCourseDialog`, `EditCourseDialog`, `BulkStudentDialog`, `LessonPlanDetail`) 는 각각 T3a · T4 · T5 · T6 · T3a 재현 문서에서 완전한 스펙을 넘긴다.
