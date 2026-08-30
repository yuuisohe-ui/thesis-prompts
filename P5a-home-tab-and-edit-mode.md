# P5a · 반 상세페이지 shell + 홈 탭 + 편집 모드 재현 프롬프트

> **적용 대상**
> - `src/pages/CourseDetail.tsx` — 라우트 진입점 (소유자·로그인 학생·공유 링크 3 경로 통합)
> - `src/components/courses/CourseModuleToolbar.tsx` — 편집 모드 sticky 툴바 dispatcher
> - `src/components/courses/HomeTab.tsx` — 홈 탭 본체 + 편집 모드 총괄
> - `src/components/courses/CourseHeroEditor.tsx` — Hero 표지 편집 다이얼로그
> - `src/components/courses/home-blocks/blockCatalog.tsx` — 블록 정의 + prefill
> - `src/components/courses/home-blocks/defaultBlocks.ts` — 신규 코스 시딩
> - `src/components/courses/home-blocks/types.ts` — 블록 타입 정의
> - `src/App.tsx` — 라우트 두 줄 추가
>
> 블록 렌더러(`CourseHomeBlockRenderer`) / 인라인 에디터(`CourseHomeBlockEditorItem`) 는 **P5b**, 캘린더/자료/알림/우리 반 탭은 **P5c~P5e** 로 위임한다. `SemesterPicker`, `ClassTimePicker`, `lib/courseSchedule.ts` 는 본 프롬프트에서 인터페이스만 확정하고 상세 구현은 워크스페이스 다이얼로그 프롬프트(P4)와 공유한다.
>
> 본 프롬프트는 `00-template.md` 의 5-Section 골격을 따른다.

## 이론적 근거
1. OpenAI. (n.d.). *Prompt engineering — developer messages: Identity, Instructions, Examples, Context.* OpenAI Platform Documentation.
2. Lovable. (n.d.). *Prompting best practices — Meta prompting & scoped rewrites.* Lovable Documentation.
3. IEEE. (1998). *IEEE Std 830-1998 §4.3.6 Verifiable*, p. 7.
4. Nielsen, J. (1993). *Usability Engineering, Ch. 5.4 — Support undo.* Academic Press.
5. Cohn, M. (2004). *User Stories Applied*, Ch. 6, pp. 67–74. Addison-Wesley.

---

## ① Identity (신원)

당신은 React 18 + React Router v6 + `@dnd-kit` + shadcn/ui + Supabase 기반의 프론트엔드 아키텍트입니다. 반 상세페이지의 **공용 shell** 과 **홈 탭 + 편집 모드** 를 하나의 응집된 산출물로 완성합니다.

- **Shell**: `/courses/:id` 와 `/courses/shared/:token` 을 하나의 컴포넌트로 흡수. 소유자(교사) 뷰 · 로그인 학생 뷰 · 공유 링크 학생 뷰 3 경로에 대해 온보딩 게이트, Hero, 5 탭 레이아웃, 편집 모드 스위치, 방문 로깅을 담당.
- **홈 탭 읽기 모드**: `course_home_blocks` 를 순서대로 렌더링. 소유 교사의 `is_public_to_students=true` + `bio` 존재 시 가상 `teacher_info` 블록을 삽입.
- **홈 탭 편집 모드**: 툴바 CustomEvent 소비 → 블록 추가, DnD 순서 이동, 인라인 편집 patch(디바운스 저장), 20 단 Undo/Redo, Hero 표지 편집(직접 업로드 / Pixabay / 제거) 을 담당.

## ② Instructions (지시)

### 2.1 산출물 파일

1. `src/pages/CourseDetail.tsx` — 라우트 진입점.
2. `src/components/courses/CourseModuleToolbar.tsx` — 편집 모드 sticky 툴바 (홈 탭 전용 dispatcher).
3. `src/components/courses/HomeTab.tsx` — 홈 탭 통합.
4. `src/components/courses/CourseHeroEditor.tsx` — Hero 편집 Dialog.
5. `src/components/courses/home-blocks/types.ts` — 블록 타입 열거 · lesson plan/week 서머리.
6. `src/components/courses/home-blocks/blockCatalog.tsx` — `blockDefinitions[]`, `createBlockInsert`, `buildBlockInsertPrefilled`, 블록별 default `content`.
7. `src/components/courses/home-blocks/defaultBlocks.ts` — `createDefaultCourseHomeBlocks(...)`.
8. `src/App.tsx` 에 라우트 두 개 추가:
   - `<Route path="/courses/:id" element={<CourseDetail />} />`
   - `<Route path="/courses/shared/:token" element={<CourseDetail />} />`

> `CourseHomeBlockRenderer`, `CourseHomeBlockEditorItem` 은 P5b 소관. 본 프롬프트는 두 컴포넌트의 **호출 계약(props)** 만 §2.14 에서 확정한다.

---

### 2.2 라우트·데이터 흐름 (CourseDetail)

- `useParams<{ id?: string; token?: string }>()` 로 두 경로를 하나로 흡수. `isShared = !!token`.
- `courses` 테이블 fetch: `token` 이면 `.eq("share_token", token)`, 아니면 `.eq("id", id)` 로 `.maybeSingle()`.
- 필요 컬럼: `id, name, level, semester, class_time, description, introduction, share_token, student_count, owner_id, cover_image_url, start_date`.
- 데이터 없음 → destructive toast `{ title: "오류", description: "과정을 찾을 수 없습니다." }` + 화면에 `과정을 찾을 수 없습니다.` + `돌아가기` 버튼(`navigate(-1)`).

### 2.3 인증 게이트 (공유 링크 전용)

- 조건: `isShared && !authLoading && !loading && course && !user`.
- 동작: `navigate("/auth?redirect=" + encodeURIComponent(pathname+search) + "&course=" + course.id + "&courseName=" + encodeURIComponent(course.name), { replace: true })`.
- `/courses/:id` 경로에서는 미로그인 상태를 상위 `RequireAuth` 가 처리하므로 여기서는 리다이렉트 금지.

### 2.4 역할 판정

```
isOwner        = !!(course && user && course.owner_id === user.id)
isStudentView  = isShared || (!!course && !isOwner)
effectiveEditMode = isOwner && editMode
```

### 2.5 학생 온보딩 게이트

`authLoading || loading || !course || !user || profileChecked` 이면 skip. 아니면:
1. `course.owner_id === user.id` → `setProfileChecked(true)` 종료.
2. `isShared === false` 인 비-소유자: `user_roles` 조회 → `teacher | admin` 이면 즉시 `setProfileChecked(true)` 종료.
3. `course_student_profiles` 에서 `(course_id, member_user_id=user.id)` 로 `.maybeSingle()`.
4. **동시에** `course_members` upsert:
   ```ts
   supabase.from("course_members").upsert(
     { course_id: course.id, user_id: user.id, role: "student" },
     { onConflict: "course_id,user_id" }
   )
   ```
5. 프로필 없음 → `setOnboardingOpen(true)`. 마지막에 `setProfileChecked(true)`.
6. `useEffect` 정리 함수에서 `cancelled = true` 로 경합 방지.

### 2.6 온보딩 닫힘 처리

`handleOnboardingOpenChange(next)`: `next === false` 이고 프로필 여전히 없으면 → toast `{ title: "입장이 취소되었습니다" }` + `navigate(-1)`.

### 2.7 로딩 스피너 조건

`loading || (isShared && (authLoading || (user && !profileChecked)))` → `<div class="animate-pulse text-muted-foreground">로딩 중...</div>` 를 60vh 중앙에 표시.

### 2.8 탭 상태

- 초기값: `new URLSearchParams(window.location.search).get("tab")` 을 `CourseTabValue` 로 캐스팅 후 없으면 `"home"`.
- `Tabs value onValueChange` 로 5 탭 제어. `TabsList` 는 `w-full grid grid-cols-5` + `data-tour="course-tabs"`.
- 각 `TabsTrigger` 는 `gap-1.5` + 아이콘 + `<span class="hidden sm:inline">` 라벨.
- 탭 목록: `home` 홈 · `calendar` 캘린더 · `materials` 강의 자료 · `notifications` 알림 · `community` 우리 반. 아이콘: `Home`, `Calendar`, `FolderOpen`, `Bell`, `Users` (lucide-react).

### 2.9 편집 모드 레이아웃

- 스위치는 소유자만 노출. `Switch id="edit-mode"` + `<Label htmlFor="edit-mode">편집 모드</Label>`, pill 컨테이너 `rounded-full border border-white/20 bg-black/20 px-3 py-1.5 backdrop-blur-sm`.
- ON + `activeTab === "home"` 일 때만 본문을 `<div class="grid gap-6 xl:grid-cols-[minmax(0,1fr)_320px] xl:items-start">` 로 감싸고 우측에 `<CourseModuleToolbar />`. **다른 탭에서는 편집 모드 스위치가 ON 이어도 툴바 미노출** — 툴바는 홈 탭 전용.
- OFF 이면 tabs 만 렌더.

### 2.10 Hero 섹션 (읽기)

- 컨테이너: `relative flex min-h-[200px] flex-col justify-between overflow-hidden rounded-[28px] border p-6 text-primary-foreground shadow-sm sm:p-8`.
- `cover_image_url` 있음 → `border-white/10`, `<img class="absolute inset-0 h-full w-full object-cover">` + 오버레이 `<div class="absolute inset-0" style="background: rgba(10,5,30,0.5)">` (`// intentional: hero cover overlay`).
- 없음 → `border-primary/20 bg-gradient-to-br from-primary via-primary to-primary/70` + 보조 오버레이 `bg-gradient-to-tr from-transparent via-transparent to-background/10 bg-[#2e3d6b]` (`// intentional: hero gradient fallback`).
- 좌상단: `!isShared` 인 경우 `ArrowLeft` ghost icon 버튼 (`border border-white/20 bg-black/20 text-white hover:bg-black/30 hover:text-white`). 공유 뷰는 `<div />` 자리 유지.
- 우상단: `isOwner` 만 편집 스위치 pill. 편집 모드 ON 이면 스위치 좌측에 `이미지 편집` ghost 버튼 (Hero Editor Dialog 트리거).
- 하단: `<h1 class="text-3xl font-bold tracking-tight text-white drop-shadow sm:text-4xl">{course.name}</h1>` + `<p class="text-sm text-white/85 sm:text-base">{[semester, level, class_time].filter(Boolean).join(" · ")}</p>`.

### 2.11 방문 로깅

```ts
useTrackShareVisit({
  linkType: "course",
  token: token ?? null,
  ownerId: null,
  resourceTitle: course?.name ?? null,
  enabled: isShared && !!course,
});
```

### 2.12 CourseModuleToolbar 규칙

- 파일 상단에 반드시 export:
  - `export type CourseTabValue = "home" | "calendar" | "materials" | "notifications" | "community"`
  - `export const COURSE_TOOLBAR_EVENT_NAME = "course-module-toolbar:add"`
  - `export interface CourseToolbarAddEventDetail { tab: CourseTabValue; type: string }`
  - `export function dispatchCourseToolbarAdd(detail: CourseToolbarAddEventDetail)` → `window.dispatchEvent(new CustomEvent(COURSE_TOOLBAR_EVENT_NAME, { detail }))`
- **홈 탭에서만 렌더**. props: `{ activeTab: CourseTabValue }`. `activeTab !== "home"` 이면 `return null` (편집 모드가 홈 이외 탭으로 이동하더라도 툴바 자체가 사라짐).
- 카드 컨테이너: `Card` + `xl:sticky xl:top-6 xl:z-20`. 내부 `CardContent space-y-4 p-4`.
- 헤더: `<p class="font-semibold">홈 모듈</p><p class="text-sm text-muted-foreground">홈 화면에 보여줄 블록을 추가하세요.</p>`.
- 액션 카드: `blockDefinitions` (from `./home-blocks/blockCatalog`) 을 `.map` 하여 자동 생성.
  - 스타일: `rounded-2xl border bg-card p-3 hover:bg-accent/30`.
  - 좌측 아이콘 pill: `rounded-xl bg-secondary p-2 text-secondary-foreground`.
  - 우측 `Plus` ghost icon 버튼 → `dispatchCourseToolbarAdd({ tab: "home", type: block.type })`.

### 2.13 홈 탭 데이터 라이프사이클

Mount(`useEffect` on `course.id`) 순서:

```ts
1) fetchBlocks(): course_home_blocks WHERE course_id ORDER sort_order ASC
   - rows.length === 0 → insert(createDefaultCourseHomeBlocks({...})).select() → setBlocks
   - 아니면 setBlocks(rows)
2) fetchLessonPlans(): lesson_plans WHERE course_id ORDER created_at DESC
   → lesson_weeks WHERE lesson_plan_id IN (planIds), week_number ASC
   → 각 plan.weeks 로 그룹핑
3) fetchOwnerProfile(): profiles WHERE id = course.owner_id
   .select("full_name, avatar_url, organization, bio, is_public_to_students")
4) supabase.channel(`course-home-${courseId}`)
   postgres_changes(event="*", table="course_home_blocks", filter=course_id=eq.{id})
   → 변경 감지 시 fetchBlocks() 재실행
5) cleanup: 모든 pendingPatch timer clearTimeout + removeChannel
```

### 2.14 블록 mutate 계약 (Undo/Redo 근간)

- 두 개의 스택 `history: Row[][]`, `future: Row[][]` (각 상한 20).
- `pushHistory(snap)`: `setHistory((prev) => [...prev.slice(-19), snap])` + `setFuture([])`.
- **큰 변형**(`addBlock`, `deleteBlock`, `handleDragEnd`, `undo`, `redo`): `pushHistory(currentBlocks)` → `saveOrderedBlocks(next)` 로 전체 upsert.
- **인라인 필드 변경**은 히스토리 미포함. `queueBlockUpdate(id, updates)`:
  - `setBlocks` 즉시 로컬 반영
  - `pendingPatchRef.current[id] = { ...prev, ...updates }`
  - `patchTimersRef.current[id]` 디바운스 350 ms → `persistBlockPatch(id)` (개별 `update().eq('id', id)`).
- `flushSave()` — 저장 버튼: 모든 pending timer clear + 즉시 persist + toast `저장되었습니다`.

### 2.15 `saveOrderedBlocks(next)`

```
setBlocks(next); setSavingOrder(true)
payload = next.map((b, idx) => ({
  id: b.id, course_id, created_by, block_type, title, content, sort_order: idx
}))
supabase.from("course_home_blocks").upsert(payload)
실패 → toast 오류 + fetchBlocks 재실행 (강한 롤백)
성공 → setSavingOrder(false)
```

### 2.16 `addBlock(type, insertIndex = blocks.length)`

```
pushHistory(blocks)
inserted = await buildBlockInsertPrefilled({
  courseId, createdBy: user?.id, type, sortOrder: insertIndex,
  lessonPlans,
  course: { semester, class_time, level, student_count, start_date }
})
next = [...blocks]; next.splice(insertIndex, 0, inserted)
resequenced = next.map((b, i) => ({ ...b, sort_order: i }))
await saveOrderedBlocks(resequenced)
```

### 2.17 툴바 이벤트 소비

```ts
useEffect(() => {
  const handler = (e: Event) => {
    const detail = (e as CustomEvent<CourseToolbarAddEventDetail>).detail;
    if (!detail || detail.tab !== "home") return;
    void addBlock(detail.type as any);
  };
  window.addEventListener(COURSE_TOOLBAR_EVENT_NAME, handler);
  return () => window.removeEventListener(COURSE_TOOLBAR_EVENT_NAME, handler);
}, [blocks, lessonPlans, course.id]);
```

### 2.18 DnD 순서 이동

- `@dnd-kit/core` `DndContext` + `PointerSensor(distance: 8)`, `closestCenter`.
- `SortableContext items=blocks.map(b => b.id) strategy=verticalListSortingStrategy`.
- `BlockCanvas` = `useDroppable({ id: "block-canvas" })`. `isOver` → 테두리 강조.
- `handleDragEnd`: `activeId === overId` 무시. `overId === "block-canvas"` → `newIndex = blocks.length - 1`. `arrayMove` → `sort_order` 재부여 → `pushHistory` → `saveOrderedBlocks`.

### 2.19 Undo / Redo

```
undo:
  if history.length === 0 return
  prev = history.pop()
  future.unshift(currentBlocks); future = future.slice(0, 20)
  saveOrderedBlocks(prev)

redo:
  if future.length === 0 return
  next = future.shift()
  history.push(currentBlocks)
  saveOrderedBlocks(next)
```

### 2.20 편집 모드 sticky 헤더 (HomeTab 상단)

- `sticky top-2 z-10 flex flex-wrap items-center justify-between gap-3 rounded-3xl border bg-card/95 px-4 py-3 backdrop-blur`.
- 좌측: `<PencilLine class="h-4 w-4 text-primary" /> 홈 편집 모드` + 서브라인 `오른쪽 툴바에서 블록을 추가하거나, 블록을 클릭해 편집하세요.`.
- 우측 순서: `savingOrder` 시 `저장 중...` → `실행 취소`(disabled=`history.length===0`) → `다시 실행`(disabled=`future.length===0`) → 주 버튼 `저장`(→ `flushSave`).
- 아이콘 lucide-react: `PencilLine`, `Undo2`, `Redo2`, `Save`.

### 2.21 인라인 에디터 · 자산 업로드 훅 (P5b 로 위임되는 계약)

`<CourseHomeBlockEditorItem>` (P5b) 이 소비하는 props:

```ts
onUpdate=(blockId, updates) => queueBlockUpdate(...)
onDelete=(blockId) => void deleteBlock(blockId)
onUploadAsset=(blockId, file, kind: "image" | "file") => handleUploadAsset(...)
onPickSong=(blockId, song) => handlePickSong(...)
onUpdateCourseStartDate=async (value) =>
  supabase.from("courses").update({ start_date: value || null }).eq("id", courseId)
```

- `handleUploadAsset` 경로: `${course.id}/${blockId}/${Date.now()}.${ext}` @ bucket `course-home-assets`. 성공 시 `getPublicUrl` → `queueBlockUpdate({ content: { ...prev, url, file_name, alt|label } })`.
- `handlePickSong` — `block_type === "lyrics"` 이면 `song_analyses.lyrics_with_pinyin` 를 추가 조회해 `content.lyrics` 까지 채운다. 그 외 비디오/파일/일반은 `song_id · title · artist · video_id · youtube_url` 만.
- `onUpdateCourseStartDate` 는 `course_info` 블록 인라인에서 `start_date` 수정 시 호출 → 상위 course state 도 setter 로 동기화.

### 2.22 학생/기본 렌더 순서

```
showTeacherInfo = ownerProfile?.is_public_to_students && ownerProfile?.bio?.trim()
list = [...blocks]
if (showTeacherInfo) {
  virtualTeacher = { id:"virtual-teacher-info", block_type:"teacher_info",
                     title:"교수 정보",
                     content:{full_name, avatar_url, organization, bio}, sort_order:-1 }
  idx = list.findIndex(b => b.block_type === "course_info")
  idx >= 0 ? list.splice(idx, 0, virtualTeacher) : list.unshift(virtualTeacher)
}
list.map(b => <CourseHomeBlockRenderer block={b} course={course} ... />)
```

### 2.23 CourseHeroEditor (Dialog)

3-tab: `upload | pixabay | remove`. `defaultValue="upload"`.

- 저장 확정 `persist(url|null)`: `courses.update({ cover_image_url: url }).eq('id', courseId)` → toast `표지가 업데이트되었습니다.` → `onChanged(url)` → 닫기.
- `upload`: `<Input type=file accept=image/*>` → `course-home-assets/${courseId}/hero-${Date.now()}.${ext}` 업로드(`upsert:true`) → `getPublicUrl` → `persist`.
- `pixabay`: 키워드 → `supabase.functions.invoke("pixabay-search", { body: { keywords:[k], lang:"ko", perKeyword:8, totalLimit:8 } })`. 결과 `data.assets: {id,url,thumb,user}[]` 를 3열 그리드. 클릭 시 `persist(a.url)`. 빈 상태: `키워드로 검색해보세요.`.
- `remove`: 현재 이미지 미리보기 + `variant="destructive"` 버튼 `표지 제거` → `persist(null)`.
- 하단 안내: `<Upload class="h-3 w-3" /> 이미지는 학생에게도 표시됩니다. 저작권을 확인하세요.`.

Hero 편집 트리거는 §2.10 의 우상단 `이미지 편집` 버튼. `currentUrl / onChanged` props 로 상위 course state 를 갱신.

### 2.24 코스 일정 계약 (start_date · class_time · semester)

- `courses.start_date date null`. `courses.class_time text`(형식 `요일 HH:MM~HH:MM`, 예: `월 10:00~12:00`). `courses.semester text`(형식 `YYYY년 X학기`).
- `SemesterPicker`(외부): 연도(현재 년도 default) + 학기(1|2 default 1) 두 select → 값 `${year}년 ${term}학기`.
- `ClassTimePicker`(외부): 요일 select + 시작 시(00-23) + 시작 분(00|15|30|45) + 종료 시 + 종료 분. 기본 `월 10:00~12:00`.
- `src/lib/courseSchedule.ts`: `getCurrentWeekIndex(startDate, now)` (0-based, 1주=7일), `parseClassTime(class_time)` → `{ weekday, startHour, startMin, endHour, endMin }`.
- 홈 탭은 이 계약을 **소비만** 한다: `course_info` 블록의 인라인 편집에서 3 필드 노출 + `weekly_preview` 블록이 `getCurrentWeekIndex` 로 이번 주 계산.

### 2.25 강제 제약

- semantic token 만 사용. 예외 두 곳(오버레이 `rgba(10,5,30,0.5)`, fallback `bg-[#2e3d6b]`) 만 `// intentional:` 주석과 함께 허용.
- `<h1>` 은 Hero 안 정확히 1개 (`CourseDetail.tsx`). HomeTab 내부 0.
- Supabase 임포트는 항상 `@/integrations/supabase/client`.
- URL 파라미터 파싱은 `CourseDetail` 최상단 1회.
- 실시간 채널 이름: `course-home-${courseId}` (다른 탭과 충돌 방지).
- 툴바 이벤트는 `detail.tab === "home"` 만 처리.
- `CourseModuleToolbar` 는 `activeTab !== "home"` 이면 `null` 반환 — 다른 탭에서 편집 툴바가 나타나지 않도록 하드 게이트.

---

## ③ Examples (예시)

### 3.1 확정 카피 표

| 위치 | 카피 |
|---|---|
| fetch 실패 toast title | 오류 |
| fetch 실패 toast desc | 과정을 찾을 수 없습니다. |
| 로딩 인디케이터 | 로딩 중... |
| 코스 없음 문구 | 과정을 찾을 수 없습니다. |
| 코스 없음 액션 | 돌아가기 |
| 편집 스위치 라벨 | 편집 모드 |
| Hero 편집 버튼 | 이미지 편집 |
| 온보딩 취소 toast | 입장이 취소되었습니다 |
| 탭(home) | 홈 |
| 탭(calendar) | 캘린더 |
| 탭(materials) | 강의 자료 |
| 탭(notifications) | 알림 |
| 탭(community) | 우리 반 |
| Toolbar 헤더 | 홈 모듈 / 홈 화면에 보여줄 블록을 추가하세요. |
| 편집 헤더 타이틀 | 홈 편집 모드 |
| 편집 헤더 서브라인 | 오른쪽 툴바에서 블록을 추가하거나, 블록을 클릭해 편집하세요. |
| 저장 진행 텍스트 | 저장 중... |
| Undo 버튼 | 실행 취소 |
| Redo 버튼 | 다시 실행 |
| 저장 버튼 | 저장 |
| 저장 성공 toast | 저장되었습니다 |
| 블록 fetch 실패 | 오류 / 홈 블록을 불러오지 못했습니다. |
| 초기 시딩 실패 | 오류 / 기본 홈 구성을 만들지 못했습니다. |
| 순서 저장 실패 | 오류 / 블록 순서를 저장하지 못했습니다. |
| 삭제 실패 | 오류 / 블록을 삭제하지 못했습니다. |
| 인라인 저장 실패 | 오류 / 블록 저장에 실패했습니다. |
| 자산 업로드 실패 | 업로드 실패 / (원 에러 메시지) |
| Hero 다이얼로그 제목 | 표지 이미지 편집 |
| Hero 탭 라벨 | 직접 업로드 · Pixabay 검색 · 제거 |
| Hero 업로드 라벨 | 이미지 파일 (JPG/PNG, 권장 1600×900) |
| Hero 업로드 진행 | 업로드 중... |
| Hero Pixabay placeholder | 예: 중국 풍경, 서예, 학교... |
| Hero Pixabay 빈 상태 | 키워드로 검색해보세요. |
| Hero 제거 안내 | 현재 표지를 제거하면 기본 그라디언트로 되돌아갑니다. |
| Hero 제거 없음 | 현재 표지가 설정돼 있지 않습니다. |
| Hero 제거 버튼 | 표지 제거 |
| Hero 저장 성공 | 표지가 업데이트되었습니다. |
| Hero 하단 안내 | 이미지는 학생에게도 표시됩니다. 저작권을 확인하세요. |
| Hero 저장 실패 | 저장 실패 / (원 에러 메시지) |
| Hero Pixabay 검색 실패 | 검색 실패 / (원 에러 메시지) |

### 3.2 `blockDefinitions` 표 (툴바가 자동 순회)

| type | label | description | icon |
|---|---|---|---|
| course_info | 과정 정보 | 기간·학기·시간 요약 | Info |
| goals | 학습 목표 | 학기 학습 목표 리스트 | Target |
| curriculum | 주차별 커리큘럼 | 강의안의 주차 목록 | CalendarDays |
| songs_list | 수록 노래 | 강의안에 연결된 곡 | Music |
| vocab_grid | 핵심 어휘 | 강조할 어휘 카드 | BookOpen |
| weekly_preview | 이번 주 학습 | 이번 주 강의안과 노래 | ListMusic |
| latest_announcement | 최신 공지 | 가장 최근 공지 1개 | Megaphone |
| question_comments | 질문 및 댓글 | 학생 질문 공간 | FileQuestion |

### 3.3 블록별 default `content`

```ts
{
  course_header: {},
  weekly_preview: { selected_plan_id: null, description: "연결된 강의안의 이번 주 학습 내용을 자동으로 보여줍니다." },
  latest_announcement: {},
  question_comments: {},
  text: { style: "body", text: "새 텍스트를 입력하세요." },
  image: { url: "", alt: "", caption: "" },
  video: { song_id: "", title: "대표 영상을 선택하세요", artist: "노래 관리에서 연결", video_id: "", youtube_url: "" },
  file:  { url: "", label: "새 파일 자료", description: "학생에게 배포할 PDF 또는 음원을 업로드하세요.", file_name: "" },
  lyrics:{ song_id: "", song_title: "", title: "가사 카드", artist: "", video_id: "", youtube_url: "", lyrics: [] },
  notice_banner: { tone: "primary", text: "중요한 공지나 안내를 여기에 적어주세요." },
  divider: { label: "Section" },
  quiz: { quiz_id: "moon-vocab" },
  students: { description: "학생 참여 현황과 명단을 확인하세요." },
  lesson_plan_overview: { selected_plan_id: null, description: "과정과 연결된 강의안을 보여줍니다." },
  weekly_lessons: { selected_plan_id: null, description: "주차별 수업 계획을 학생에게 안내합니다." },
  course_info: { duration: "", students_target: "", semester: "", class_time: "", start_date: "" },
  goals: { items: [] },
  curriculum: { weeks: [] },
  songs_list: { songs: [] },
  vocab_grid: { vocab: [] },
  teacher_info: {},
}
```

### 3.4 `buildBlockInsertPrefilled` 프리필 규칙

- `createBlockInsert` 결과를 base 로. `lessonPlans[0]` 을 primary 로 취급.
- `course_info` → `{ duration: primary ? "${weeks.length}주" : "", students_target: student_count ? "${n}명" : "", semester, class_time, start_date: course.start_date || "" }`.
- `curriculum` (primary?.weeks) → `weeks: weeks.map(w => ({ week: "W${w.week_number}", title: w.title || "${w.week_number}주차", desc: w.week_type === "regular" ? "정규 수업" : String(w.week_type || "") }))`.
- `songs_list` (song_ids 존재) → `song_ids` 유니크 수집 → `songs.select("id,title,artist,teaching_point").in("id", songIds)` → `songs: rows.map(r => ({ title, artist, point: teaching_point || "" }))`.
- `weekly_preview | lesson_plan_overview | weekly_lessons` (primary 존재) → `{ ...defaultContent, selected_plan_id: primary.id }`.
- 그 외 → base 그대로.

### 3.5 초기 시딩 (`createDefaultCourseHomeBlocks`)

```
[
  { type: "course_info",    sort_order: 0, content: prefill.course_info },
  { type: "goals",          sort_order: 1, content: { items: [] } },
  { type: "weekly_preview", sort_order: 2, content: prefill.weekly_preview },
  { type: "curriculum",     sort_order: 3, content: prefill.curriculum },
  { type: "songs_list",     sort_order: 4, content: prefill.songs_list },
  { type: "latest_announcement", sort_order: 5, content: {} },
]
```

### 3.6 골격 스니펫

```tsx
// CourseDetail.tsx
const { id, token } = useParams<{ id?: string; token?: string }>();
const isShared = !!token;
const isOwner = !!(course && user && course.owner_id === user.id);
const isStudentView = isShared || (!!course && !isOwner);
const effectiveEditMode = isOwner && editMode;

{effectiveEditMode && activeTab === "home" ? (
  <div className="grid gap-6 xl:grid-cols-[minmax(0,1fr)_320px] xl:items-start">
    <div>{tabs}</div>
    <CourseModuleToolbar activeTab={activeTab} />
  </div>
) : tabs}
```

```tsx
// HomeTab.tsx — mutate 스택
const [blocks, setBlocks] = useState<CourseHomeBlockRow[]>([]);
const [history, setHistory] = useState<CourseHomeBlockRow[][]>([]);
const [future, setFuture] = useState<CourseHomeBlockRow[][]>([]);
const pendingPatchRef = useRef<Record<string, TablesUpdate<"course_home_blocks">>>({});
const patchTimersRef = useRef<Record<string, ReturnType<typeof setTimeout>>>({});

const pushHistory = (snap: CourseHomeBlockRow[]) => {
  setHistory((prev) => [...prev.slice(-19), snap]);
  setFuture([]);
};
```

---

## ④ Context (배경)

### 4.1 라우팅 계약
- `RequireAuth` 는 학생 화이트리스트에 `/courses/` prefix 포함, 두 라우트 모두 통과.
- 공유 링크(`/courses/shared/:token`) 는 미로그인 시 본 컴포넌트가 자체적으로 `/auth?redirect=…` replace 이동.

### 4.2 데이터 계약
- `courses`: §2.2 컬럼. Hero 편집 결과는 `cover_image_url`, 일정 편집 결과는 `start_date`, 소유자만 UPDATE.
- `course_home_blocks(id uuid, course_id uuid, created_by uuid null, block_type text, sort_order int, title text, content jsonb, created_at, updated_at)`. Realtime 활성. RLS: 소유자 CRUD + 학생 SELECT.
- `course_student_profiles`: `(course_id, member_user_id)` 존재 여부만 확인. 스키마 관리는 P4/T3.
- `course_members`: `(course_id, user_id, role)` upsert onConflict `course_id,user_id`.
- `user_roles`: `(user_id, role)` — `teacher | admin` 판별.
- `lesson_plans + lesson_weeks` — 프리필 소스. `lesson_weeks.song_ids uuid[]` 로 `songs` 조인.
- `song_analyses.lyrics_with_pinyin` — `lyrics` 블록 자동 채움.
- `profiles.is_public_to_students & bio & full_name & avatar_url & organization` — 가상 `teacher_info` 소스.
- `share_link_visits` — `useTrackShareVisit` 훅이 insert/update.
- Storage bucket `course-home-assets` (public read, 소유자 write). 자산 경로 규칙은 §2.21 / §2.23.
- Edge function `pixabay-search` — payload/response 스펙은 P3f/T-EDGE-PIXABAY 참조.

### 4.3 이벤트 프로토콜
- `window` 레벨 `CustomEvent("course-module-toolbar:add", { detail: { tab, type } })`.
- HomeTab 은 `detail.tab === "home"` 만 처리. 다른 탭 리스너는 각각 P5c~P5e 내부에서 자체 등록·해제.
- `CourseModuleToolbar` 자체는 상태를 갖지 않음 — 순수 dispatcher.

### 4.4 Guide 훅
- `TabsList` 의 `data-tour="course-tabs"` 는 사용 가이드 온보딩 투어 앵커.

### 4.5 관련 문서
- **P5b** — 블록 렌더러 & 인라인 에디터.
- **P5c** — 캘린더 탭 (start_date + parseClassTime 소비).
- **P5d** — 강의 자료 탭 + ConnectLessonPlanDialog(start_date).
- **P5e** — 알림 탭 + 우리 반 탭.
- **P4** — 워크스페이스 다이얼로그 (`CreateCourseDialog`, `EditCourseDialog`, `SemesterPicker`, `ClassTimePicker`).
- **P3f / T-EDGE-PIXABAY** — `pixabay-search` edge function.

---

## ⑤ Acceptance & Output (검증·산출)

### 5.1 정량 Acceptance Criteria

**Shell (CourseDetail.tsx)**
1. `rg -n "useParams" src/pages/CourseDetail.tsx` = 1 건, 반환 타입에 `id?` 와 `token?` 둘 다 포함.
2. `rg -n "isShared" src/pages/CourseDetail.tsx` ≥ 4 건, 최소 1 곳은 미로그인 → `/auth?redirect=` replace 이동.
3. 소유자 계정으로 `/courses/:id` 100 회 진입 시 `course_student_profiles` 조회 0 회, 온보딩 다이얼로그 open 0 회.
4. 로그인한 다른 교사(role=teacher) 가 열면 온보딩 미표시(`user_roles` 1 회 조회).
5. 학생 계정으로 `/courses/shared/:token` 첫 진입 시 온보딩 정확히 1 회 오픈.
6. 온보딩을 저장 없이 닫으면 `입장이 취소되었습니다` toast + `navigate(-1)` 1 회.
7. `?tab=community` 로 진입 시 초기 `activeTab === "community"`.
8. 편집 모드 ON + `activeTab === "home"` 일 때만 xl 이상에서 우측 320px sticky 툴바 노출. 다른 탭에서는 편집 ON 이어도 툴바 미노출.
9. `useTrackShareVisit` 호출: 공유 링크 접근·이탈 시 `share_link_visits` insert 1 + duration update 1.
10. Hero 하드코드 색상은 오버레이 `rgba(10,5,30,0.5)` 와 fallback `bg-[#2e3d6b]` 두 곳만(`rg -n "bg-\[#|rgba\(" src/pages/CourseDetail.tsx` ≤ 2 라인).
11. `<h1>` 개수 = 1 (`rg -n "<h1" src/pages/CourseDetail.tsx | wc -l` = 1).
12. `TabsList` 에 `data-tour="course-tabs"` 존재.

**Toolbar**
13. `activeTab !== "home"` 일 때 `CourseModuleToolbar` 렌더 결과 = `null` (DOM 노드 0).
14. Home 툴바의 각 액션 클릭 → `course-module-toolbar:add` CustomEvent 정확히 1 회, `detail.tab === "home"`.

**HomeTab 편집 · 데이터**
15. `rg -n "COURSE_TOOLBAR_EVENT_NAME" src/components/courses/HomeTab.tsx` ≥ 1 회 + `removeEventListener` cleanup 존재.
16. `detail.tab !== "home"` 이벤트 dispatch 시 `addBlock` 호출 0 회.
17. Undo 20 회 연속 → history 상한 20 유지, 21 회째 no-op.
18. `pushHistory` 실행 시 `future` 즉시 clear (`setFuture([])`).
19. 새 코스 첫 진입: `course_home_blocks` insert 정확히 6 행(`course_info, goals, weekly_preview, curriculum, songs_list, latest_announcement`), sort_order 0..5.
20. 기존 코스: rows.length > 0 이면 seed insert 0 회.
21. 인라인 편집: 350 ms 내 추가 입력 없으면 1 회 `update`. 15 회 폭주 시 최종 update 1 회.
22. `flushSave` 실행 시 pending timer 전부 clear + persist + toast `저장되었습니다` 1 회.
23. Realtime: 다른 브라우저에서 blocks 수정 시 현재 창이 300 ms 이내 `fetchBlocks` 재실행.
24. `blockDefinitions.length === 8` 이고 순서 §3.2 표와 정확히 동일.
25. HomeTab 내부 `<h1>` 개수 = 0.
26. HomeTab / CourseHeroEditor 하드코드 색상 0 (`rg -n "bg-\[#|text-\[#|#[0-9a-fA-F]{6}" src/components/courses/HomeTab.tsx src/components/courses/CourseHeroEditor.tsx` = 0).

**Hero Editor**
27. `upload` 탭 파일 선택 → `course-home-assets/${courseId}/hero-*.ext` upload → `courses.update({ cover_image_url })` 1 회 → 다이얼로그 자동 close.
28. `pixabay` 탭 이미지 클릭 → 같은 update 흐름.
29. `remove` 탭 버튼 → `courses.update({ cover_image_url: null })`.

**Teacher info 가상 블록**
30. `is_public_to_students=true` & `bio` 존재 시 `teacher_info` 가상 블록이 `course_info` 바로 앞. 아니면 부재.

**일정 계약**
31. `course_info` 인라인 편집에서 `start_date` 수정 → `onUpdateCourseStartDate` 1 회 → `courses.update({ start_date })` 1 회 + 상위 course state 동기화.
32. `class_time` 파싱은 `parseClassTime` 만 사용(HomeTab 내부에 정규식 하드코드 0).

### 5.2 Output Format

반환 순서(그 외 텍스트 금지):
1. `src/pages/CourseDetail.tsx` 전체.
2. `src/components/courses/CourseModuleToolbar.tsx` 전체.
3. `src/components/courses/home-blocks/types.ts` 전체.
4. `src/components/courses/home-blocks/blockCatalog.tsx` 전체.
5. `src/components/courses/home-blocks/defaultBlocks.ts` 전체.
6. `src/components/courses/CourseHeroEditor.tsx` 전체.
7. `src/components/courses/HomeTab.tsx` 전체.
8. `src/App.tsx` 에 추가할 라우트 두 줄 스니펫.
9. 한국어 3줄 요약.
