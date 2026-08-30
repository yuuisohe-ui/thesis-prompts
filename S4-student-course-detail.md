# S4 · 학생 반 상세 뷰 (Student Course Detail)

> **범위**: 학생이 `/courses/:id` 혹은 `/shared/course/:token` 로 진입한 이후 보게 되는 **반 상세 화면 전체**의 학생용 렌더링·상호작용 재현 프롬프트.
> 반의 골격(라우팅·Hero·5탭 전환·오너 판별·Realtime)은 공용 P5x 및 T4/T6 에서 이미 정의됨. **S4 는 그 위에서 학생이 실제로 사용하는 부분**(읽기 전용 홈 렌더러, 학생 시점 캘린더, 학생 시점 강의 자료 3-서브탭, 학생 알림 컴포저 + 답글 스레드, 우리 반 카드 그리드 상호작용)만 반복 없이 정의한다.
> 초대 링크 진입·학생 프로필 온보딩·`course_members` upsert 은 **S2**, 온보딩 이후 진입하는 학생 대시보드는 **S3(=`/student-home`)** 담당. S4 는 "학생이 반 안에 들어와서 하는 모든 것".

---

## 1. Identity

- **Route**: `/courses/:id`, `/shared/course/:token` (둘 다 `RequireAuth` 필수). 학생은 두 경로 모두 도달 가능.
- **Component 경로**: `src/pages/CourseDetail.tsx` (공용 shell), `src/components/courses/{HomeTab, CalendarTab, LessonMaterialsTab, NotificationsTab, CommunityTab}.tsx` + 하위 `home-blocks/*`, `materials/*`, `notifications/*`.
- **역할 판정** (반드시 동일하게 구현):
  ```ts
  const isShared = !!token;
  const isOwner  = !!(course && user && course.owner_id === user.id);
  const isStudentView = isShared || (!!course && !isOwner);
  ```
  → S4 는 **`isStudentView === true`** 인 모든 사용자에게 적용된다. `editMode` 는 항상 false 로 강제(오너만 스위치 노출).
- **선행 조건**: 학생 프로필 (`course_student_profiles.member_user_id = auth.uid()`) 이 있어야 정상 진입. 없으면 S2 의 `StudentOnboardingDialog` 가 열리며, dismiss 시 `navigate(-1)`.
- **Non-goals**:
  - 홈 블록 편집 UI, 툴바 이벤트 수신, undo/redo, 블록 추가/삭제/드래그 — 모두 **오너 전용**(T4 소관).
  - 캘린더 새 일정 추가 · 삭제(오너·본인만) 이외의 관리 — T6 소관.
  - 자료 3-서브탭의 업로드 / 노래 추가 / 링크 · 메모 · 임베드 추가 — T6 소관.
  - 학생 프로필 카드 최초 등록 / 초대 링크 발급 · 직접 입력 슬롯 — T6 우리 반 편집층에서 처리.

---

## 2. Instructions

### 2.1 진입 게이트 (CourseDetail 학생 분기)

- `useEffect` 로 `course` 를 fetch(`share_token` 또는 `id`)한다. 실패 시 "과정을 찾을 수 없습니다." 토스트.
- `isShared && !user` → `/auth?redirect=<현재 경로>&course=<id>&courseName=<name>` 로 replace.
- `authLoading || loading || (isShared && user && !profileChecked)` 동안 애니메이션 로더("로딩 중...", `min-h-[60vh]`).
- 학생 여부 확인 뒤:
  - `user_roles` 조회 → `teacher` 또는 `admin` 이면 온보딩 스킵.
  - 그 외에는 `course_student_profiles` 에서 `member_user_id = user.id` 존재 여부 확인, `course_members` 를 `role='student'` 로 upsert(`onConflict: "course_id,user_id"`).
  - 프로필 없으면 `StudentOnboardingDialog` 열기(S2 재사용).
- Hero 영역: 학생은 편집 모드 스위치·`표지 편집` 버튼이 렌더되지 않는다. `isShared` 이면 뒤로가기 버튼도 숨김(빈 `<div />` 로 자리만 차지).

### 2.2 5-탭 진입 파라미터

- URL `?tab=home|calendar|materials|notifications|community` 를 초기값으로 사용.
- 모든 탭에 학생용 `data-tour` 는 유지(가이드 S7 재사용). 단 편집 툴바(`CourseModuleToolbar`)는 학생 뷰에서 렌더되지 않음.

### 2.3 홈 탭 (`HomeTab`) — 학생 렌더 경로

- 학생은 반드시 `editMode === false` 분기(§HomeTab.tsx 480-527)로 흐른다. DnD·에디터·툴바 리스너는 오너 전용.
- `fetchData()` 는 학생에게도 동일하게 실행되어 다음 3개를 병렬 로드:
  1. `course_home_blocks` (초기 배열이 비어있으면 `createDefaultCourseHomeBlocks` 로 자동 시드 후 insert — 학생도 트리거될 수 있으므로 정책상 `INSERT` 를 학생 role 에게도 허용해야 함, 반대로 시드가 없어도 렌더가 깨지지 않도록 방어).
  2. `lesson_plans` + `lesson_weeks` (해당 course_id).
  3. `profiles` 오너 정보(`full_name, avatar_url, organization, bio, is_public_to_students`).
- Realtime 채널 `course-home-${id}` 을 학생에게도 붙여 블록 변경을 즉시 반영.
- **교수 정보 가상 블록 삽입**: `ownerProfile.is_public_to_students === true && bio.trim()` 일 때만 `virtual-teacher-info` 를 만들고,
  - `course_info` 블록이 있으면 그 바로 앞에,
  - 없으면 리스트 맨 앞에 unshift.
  - 이 블록은 절대 DB 에 저장하지 않는다(가상 id: `virtual-teacher-info`).
- 각 블록은 `CourseHomeBlockRenderer` 로 위임하고, 다음 prop 을 전달: `courseId, courseName, introduction, studentCount, lessonPlans, user, startDate, semester, classTime`.
- 블록 타입별 학생 시점 렌더 규칙(§CourseHomeBlockRenderer 및 FocusedHomeBlocks 기반):

  | block_type              | 학생 렌더 규칙                                                                                          |
  |-------------------------|---------------------------------------------------------------------------------------------------------|
  | `course_header`         | `return null`. Hero 는 상위 `CourseDetail` 가 그림.                                                     |
  | `latest_announcement`, `question_comments` | 현재 렌더러에서 `return null` — 이 자리는 알림 탭·커뮤니티 탭으로 이동됨. 남아있어도 화면에 안 그려짐. |
  | `text`                  | style=title/subtitle/body 3종, `whitespace-pre-wrap`.                                                   |
  | `notice_banner`         | `tone ∈ {primary, accent, muted, destructive}` → tailwind 클래스 매핑. Megaphone 아이콘.                |
  | `divider`               | 얇은 구분선 + 선택적 대문자 라벨.                                                                        |
  | `image`                 | `content.url` 있으면 `<img>`(최대 420px 높이) + caption. 없으면 안내 카드.                              |
  | `video`                 | `content.video_id` 로 YouTube 임베드(16:9). 없으면 안내 카드. 학생은 편집 불가.                          |
  | `file`                  | 카드 + `다운로드` 버튼(새 탭). `content.label || file_name`.                                            |
  | `lyrics`                | `song_analyses.lyrics_with_pinyin` 을 최대 6행 프리뷰. 원가사 · 병음 · 한국어 3줄.                       |
  | `quiz`                  | `quizPresets` 에서 매칭, `퀴즈 관리 열기` 버튼은 학생 노출 유지(권한은 라우팅 단에서 걸림).              |
  | `students`              | 학생 수 표기 + `학생 관리` 버튼. 학생 클릭 시 `/students` 는 접근 권한 없음(라우팅 단에서 처리).         |
  | `lesson_plan_overview`  | 연결된 강의안 카드 그리드.                                                                                |
  | `weekly_lessons`        | 선택 강의안의 8개 주차까지 나열, 오버플로우 안내 문구.                                                    |
  | `course_info`           | 총 기간 · 대상 · 학기 · 수업 시간 · 시작일(포맷 `YYYY년 M월 D일`).                                       |
  | `goals`                 | 이모지 🎯 + 초록색 pill 리스트.                                                                          |
  | `curriculum`            | 이모지 📅 + 좌측 primary border, week/title/desc.                                                        |
  | `songs_list`            | 이모지 🎵 + artist · point.                                                                              |
  | `vocab_grid`            | 이모지 📖 + 3-column grid(중문·병음·한글).                                                               |
  | `weekly_preview`        | **`FocusedHomeBlocks.WeeklyPreviewBlock`** — `getCourseSchedule(course.start_date, weeks.length)` 로 상태 계산: `no_start_date` / `before_start` / `in_progress`. 진행 중이면 현재 주차의 `title`, `week_number`, `song_ids` 상위 2곡(`songs` 테이블 fetch)까지 카드로 표시. |
  | `teacher_info`(가상)    | 위 §Home 규칙에 따라 학생 뷰에서만 삽입. 렌더러에는 정식 case 가 없으므로, **본 문서 요구사항**: 렌더러에 `case "teacher_info"` 를 추가하여 아바타 + 성함 + 소속 + bio 를 카드로 표시하도록 구현할 것. 존재하지 않는 case 로 떨어지면 렌더 미실행 → 반드시 오너의 `is_public_to_students` 토글이 켜졌을 때 학생 화면에 노출되어야 한다. |

### 2.4 캘린더 탭 (`CalendarTab`) — 학생 뷰

- 학생/오너 모두 동일한 컴포넌트 사용, 편집 권한만 row-level 로 다름.
- 좌측 대형 캘린더 + 우측 3-영역 사이드 패널.
- 좌측: `date-fns` + `@hyunbinseo/holidays-kr` 로 공휴일 자동 표기.
  - modifier: `classDay` (수업요일, `parseClassTime(course.class_time)` 로 파싱, `course.start_date` 이전은 제외) → primary/15 배경.
  - modifier: `holiday` → destructive 텍스트.
  - modifier: `hasEvent` (`course_calendar_items.starts_at` 매핑) → secondary ring.
- 우측 상단(선택 없음): "N월 수업 일정" + 총 회차 배지 + 스크롤 리스트(예정/완료 상태 badge).
- 우측 하단(날짜 선택 시): 정규 수업 카드, 공휴일 카드, `course_calendar_items` 카드 목록.
- **일정 추가 버튼**: 학생에게도 노출. 눌러도 `user` 있으면 `AddCalendarItemDialog` 로 진입 가능. 저장 시 payload:
  ```json
  { "course_id": "...", "user_id": "<auth.uid()>", "title": "...", "description": null, "starts_at": "ISO", "all_day": false, "event_type": "class|exam|notice|assignment|other", "item_type": "lesson" }
  ```
  RLS 는 학생 본인의 `user_id` 만 insert 가능하도록 이미 구성되어 있다.
- **삭제 버튼**: `item.owner_id === user.id || item.user_id === user.id` 인 경우에만 휴지통 아이콘 노출. 학생은 자기가 추가한 일정만 지울 수 있고, 교사 일정은 지울 수 없다.
- 월 이동은 `month` state 로 관리, 월 변경 시 `fetchItems` 재실행(월 범위 필터).
- 로그아웃 상태 시 `일정 추가` 클릭 → "로그인이 필요합니다" destructive 토스트.

### 2.5 강의 자료 탭 (`LessonMaterialsTab`) — 학생 뷰

- `isStudentView=true` 로 자식 3-패널 렌더링, 3-서브탭 구조: `plans` · `songs` · `attachments`.
- **자동 서브탭 숨김**: 학생에게는 `count > 0` 인 서브탭만 노출. 현재 선택 탭이 숨겨지면 존재하는 첫 탭으로 자동 전환. 세 서브탭이 모두 비면 "아직 강의 자료가 등록되지 않았습니다" 빈 카드(단, 3-패널을 `hidden` 으로 mount 하여 실시간 count 유지).
- 탭 트리거는 아이콘(BookOpen/Music/Paperclip) + 라벨 + 카운트 배지.
- **강의안 패널(`LessonPlanPanel`)** — 학생:
  - 강의안 다중일 때 Select 로 전환.
  - 각 주차는 Accordion. 학생에게 **잠금** 규칙: `week.content.is_locked === true` **또는** `course.start_date` 기준 `getWeekDateRange(week_number)` 의 시작일이 미래일 때 → `disabled`, 자물쇠 아이콘, `[&>svg]:hidden`, opacity-70. 잠금 텍스트: "🔒 잠김" 대신 학생 문구는 "강사가 이 주차를 잠갔습니다" / "아직 시작되지 않은 주차입니다".
  - 잠금/시작 전 주차는 열 수 없다. 정상 주차는 `WeekDetailView` 를 `embedded isStudentView` 로 렌더.
  - 학생에게는 잠금 토글 버튼(자물쇠 아이콘)이 렌더되지 않는다.
- **노래 패널(`CourseSongPanel`)** — 학생:
  - 상단 "노래 추가" 툴바 숨김.
  - 카드 그리드(sm:2, lg:3): YouTube 썸네일(`img.youtube.com/vi/<id>/mqdefault.jpg`), HSK 레벨 배지, title/artist. 카드 클릭 시 `it.song_id` 있으면 `/songs/${song_id}` 로 이동(공용 곡 분석 화면). 삭제 버튼 미노출.
- **첨부 자료 패널(`CourseAttachmentPanel`)** — 학생:
  - 상단 "파일 업로드"·"추가" 드롭다운 숨김.
  - `material_type ∈ {file, link, text, embed}` 4종 렌더:
    - `file`: 아이콘(파일 종류별) + 이름/사이즈 + `다운로드` 링크(새 탭). 삭제 버튼 미노출.
    - `link`: 제목/URL/설명 + `열기` 새 탭 링크.
    - `embed`: YouTube ID 추출 후 16:9 iframe. 없으면 URL 텍스트.
    - `text`: 제목 + 본문(줄바꿈 유지) + URL 자동 링크화(`URL_REGEX`).
  - 빈 상태: 종이집게 아이콘 + "아직 추가된 자료가 없습니다".
  - Realtime: `course-attachments-${id}` 채널로 즉시 갱신.

### 2.6 알림 탭 (`NotificationsTab`) — 학생 뷰

- `effectiveStudentView = isStudentView || !isTeacher || !isOwner`. 학생은 항상 true.
- 좌측 컬럼: `TeacherNoticeList` + `StudentPostList`.
- 우측 컬럼: `StudentComposer` (오너면 `TeacherComposer` — S4 범위 외).
- **`visiblePosts` 필터**: `visibility === "public"` 이거나 `student_id === user.id` 인 것만.
- Realtime: `notif-${courseId}` 채널로 `course_notices`, `course_student_posts` 모두 구독, `course_student_profiles` 는 이니셜 로드에서 `{member_user_id → {full_name, avatar_url, emoji}}` 맵 구성.
- 배경 `#EEF1F8`, 카드 배경 `#fff`, 강조 텍스트 `#2e3d6b`, 태그 배경 `#EEF2FF`.

#### 2.6.1 교사 공지 리스트 (`TeacherNoticeList`, canManage=false)

- 상단 필터 pill: `전체 / 공지 / 과제 / 일정` (학생도 필터 사용 가능).
- 카드: 유형 pill + 시각(`M월 D일 HH:MM`) + `답글` 토글. 편집·삭제 버튼은 canManage=false 이므로 미노출.
- 토글 열림 시 `RepliesThread(parentType="notice")` 렌더.

#### 2.6.2 학생 게시물 리스트 (`StudentPostList`, canManage=false)

- 필터 pill(전체공개/교사전용)은 canManage=false 이므로 숨김. 학생은 자기 글 + 전체공개 글만 본다.
- 카드: 아바타(emoji/이니셜) + 이름 + 시간 + 유형 pill(결석/지각/질문/기타) + 공개 여부 아이콘(Globe/Lock).
- 상태 chip: `pending(대기)`, `approved(승인됨)`, `rejected(거절됨)`, `answered(답변 완료)`.
- 승인/거절/답변 완료 버튼은 canManage=false 이므로 미노출.
- **자기 글 삭제**: `isOwn = p.student_id === currentUserId` 인 경우 휴지통 아이콘 노출, DELETE `course_student_posts`.
- `답글` 토글 → `RepliesThread(parentType="student_post")`.

#### 2.6.3 답글 스레드 (`RepliesThread`)

- `notification_replies` 를 `parent_type` + `parent_id` 로 fetch, `created_at ASC`.
- 작성자 이름은 `profiles.full_name` 로 기본, `course_student_profiles.member_user_id` 매칭 시 이모지+`full_name` 로 덮어씀.
- Realtime: `replies-${parentType}-${parentId}` 채널.
- 학생은 자신의 답글만 삭제 가능(`user.id === r.author_id`). 교사(`isTeacher`)는 모든 답글 삭제 가능.
- 입력: 2행 textarea + `Send` 버튼(빈 draft 시 disable). 로그아웃 시 안내 문구만.
- 등록 payload: `{ parent_type, parent_id, course_id, author_id, content }`.

#### 2.6.4 학생 컴포저 (`StudentComposer`)

- 유형 pill 4종: `결석 / 지각 / 질문 / 기타`. 초깃값 `질문`.
- 유형별 placeholder 텍스트(각 안내문 필수 유지).
- 공개 pill 2종: `교사에게만(private)` / `전체 공개(public)`. **유형이 `결석` 또는 `지각` 이면 자동으로 `private` 로 강제(useEffect)**.
- 6행 textarea. 하단 버튼 3종:
  1. `AI로 다듬기` — `supabase.functions.invoke("polish-notice", { body: { role: "student", type, content } })`. 결과 `polished` state 로 저장, `AIPolishPanel` 로 교체/취소 선택.
  2. `미리보기` — 토글.
  3. `보내기` — 미리보기 모드로 전환 후 확인 시 insert.
- 전송 payload: `{ course_id, student_id: user.id, type, content, visibility, status: "pending" }`.
- 성공 시 "전송 완료" 상태 카드, `새 메시지 작성` 버튼으로 초기화.
- 에러 처리: `data.error` → `AI 다듬기 실패` destructive 토스트, `insert` 에러 → `발송 실패` 토스트.
- 로그아웃 상태에서 `보내기` 클릭 → "로그인이 필요합니다" destructive 토스트.

### 2.7 우리 반 탭 (`CommunityTab`) — 학생 뷰

- `editMode=false` 로 전달됨(오너만 true 가능).
- CSS Grid: `repeat(auto-fill, minmax(160px, 1fr))`, gap 14px.
- 헤더: 제목 + 설명 + `학생 초대` 버튼(초대 링크 복사, 학생에게도 노출).
- **학생 카드**: `StudentOnboardingDialog` 를 통해 저장된 `course_student_profiles` 를 `sort_order ASC` 로 나열.
  - 아바타 우선순위: `avatar_url` > `emoji` > 이름 이니셜 2자.
  - 이름 · 학과 · 학번 · 구분선 · 뱃지(레벨 / 학습기간 / 성별).
  - 본인 카드에는 `나` 뱃지 + primary/40 ring.
- **본인 카드 편집**: `member_user_id === user.id` 이면 hover 시 연필 아이콘 노출 → 클릭 시 `StudentOnboardingDialog(existing=myProfile)` 를 재사용해 프로필 편집. 저장 후 `fetchStudentCards()` 재실행.
- **다른 학생 카드**: 학생 뷰(editMode=false)에서는 hover 편집 아이콘·직접 입력 슬롯이 표시되지 않는다.
- Placeholder ghost 카드: 학생 뷰에서는 `MIN_PLACEHOLDER_COUNT(6) - students.length` 만큼만(음수면 0). 카드 안 "직접 입력" 버튼은 `editMode` 에서만 렌더.
- 로딩 시 6개 스켈레톤 카드.

### 2.8 데이터 계약 요약

| 테이블 / 스토리지                | 학생 권한          | 사용처                                     |
|---------------------------------|--------------------|--------------------------------------------|
| `courses`                       | SELECT (member/share_token) | Hero, 홈 오너 profile, start_date       |
| `course_home_blocks`            | SELECT + (필요 시) INSERT(시드) | HomeTab 렌더                       |
| `course_calendar_items`         | SELECT / INSERT(본인) / DELETE(본인) | CalendarTab                    |
| `course_material_items`         | SELECT (course member)      | 자료 3-서브탭                            |
| `lesson_plans`, `lesson_weeks`  | SELECT (연결된 course)      | 강의안 패널, weekly_preview 블록          |
| `songs`, `song_analyses`        | SELECT (public)             | weekly_preview, lyrics 블록, `/songs/:id` |
| `course_notices`                | SELECT (course member)      | 알림 좌측 상단                            |
| `course_student_posts`          | SELECT (public + own) / INSERT(own) / DELETE(own) | 알림 좌측 하단·컴포저 |
| `notification_replies`          | SELECT (course member) / INSERT(own) / DELETE(own) | 답글 스레드            |
| `course_student_profiles`       | SELECT (course member) / UPDATE(own via member_user_id) | 우리 반, 알림 프로필 조인 |
| `profiles`                      | SELECT `(is_public_to_students=true)` 열만 노출 필요 | 교수 정보 블록          |
| `course_members`                | UPSERT(own)                 | 진입 시 role='student' 등록              |
| `storage: materials`            | SELECT (public url)         | 첨부 다운로드                             |
| `supabase.functions/polish-notice` | invoke                   | 학생 컴포저 AI 다듬기                     |

### 2.9 에러 처리 규칙

- 모든 fetch 실패 → 한국어 destructive 토스트.
- 로그인 필요 시 → "로그인이 필요합니다" destructive 토스트(캘린더/컴포저/답글 공통).
- 온보딩 취소 시 → "입장이 취소되었습니다" 토스트 + `navigate(-1)`.
- `polish-notice` 429/402 등 상위 에러는 서버가 이미 한국어 메시지로 내려주므로 그대로 표시.

---

## 3. Examples

### 3.1 CourseDetail 학생 진입 판정

```ts
const isShared = !!token;
const isOwner = !!(course && user && course.owner_id === user.id);
const isStudentView = isShared || (!!course && !isOwner);
const effectiveEditMode = isOwner && editMode; // 학생은 항상 false
```

### 3.2 HomeTab 학생 렌더 (교수 정보 삽입)

```tsx
const showTeacherInfo = !!ownerProfile?.is_public_to_students && !!ownerProfile?.bio?.trim();
const list = [...blocks];
if (showTeacherInfo) {
  const teacherBlock = { id: "virtual-teacher-info", block_type: "teacher_info", ... };
  const idx = list.findIndex(b => b.block_type === "course_info");
  idx >= 0 ? list.splice(idx, 0, teacherBlock) : list.unshift(teacherBlock);
}
return list.map(b => <CourseHomeBlockRenderer block={b} ... />);
```

### 3.3 학생 컴포저 전송

```ts
await supabase.from("course_student_posts").insert({
  course_id, student_id: user.id, type: "질문",
  content: text, visibility: "private", status: "pending",
});
```

### 3.4 자기 게시물 삭제

```ts
if (p.student_id === currentUserId) {
  await supabase.from("course_student_posts").delete().eq("id", p.id);
}
```

### 3.5 캘린더 자기 일정 삭제 조건

```tsx
{user && (item.owner_id === user.id || item.user_id === user.id) && (
  <Button onClick={() => handleDelete(item.id)}><Trash2 /></Button>
)}
```

### 3.6 강의안 학생 잠금 판정

```ts
const isLocked = !!week.content?.is_locked;
const weekRange = course.start_date ? getWeekDateRange(course.start_date, week.week_number) : null;
const notStartedYet = !!(weekRange && new Date() < weekRange.start);
const blockedForStudent = isStudentView && (isLocked || notStartedYet);
```

---

## 4. Context

### 4.1 재현 대상 파일

- `src/pages/CourseDetail.tsx` (학생 분기 · 프로필 확인 · Hero 학생 뷰)
- `src/components/courses/HomeTab.tsx` (읽기 전용 렌더 경로 + `teacher_info` 가상 블록)
- `src/components/courses/home-blocks/CourseHomeBlockRenderer.tsx` + `FocusedHomeBlocks.tsx` (블록별 렌더링)
- `src/components/courses/CalendarTab.tsx` (수업요일·공휴일·본인 일정 CRUD)
- `src/components/courses/LessonMaterialsTab.tsx` + `materials/{LessonPlanPanel,CourseSongPanel,CourseAttachmentPanel}.tsx`
- `src/components/courses/NotificationsTab.tsx` + `notifications/{TeacherNoticeList,StudentPostList,StudentComposer,RepliesThread,AIPolishPanel}.tsx`
- `src/components/courses/CommunityTab.tsx` + `StudentOnboardingDialog.tsx`(S2 재사용)
- `src/lib/courseSchedule.ts` (`parseClassTime`, `getCourseSchedule`, `getWeekDateRange`)

### 4.2 기존 문서와의 차이

- **P5a (홈 탭 골격 · 편집 모드)**: 5탭 shell·오너용 편집 모드 전체 스펙. S4 는 그 shell 위에서 학생이 실제로 보는 렌더 경로(교수 정보 자동 삽입, weekly_preview 상태별 카드, 잠금/공휴일)만 강조.
- **P5b (홈 블록 세트)**: 블록 렌더러의 편집 폼·필드 스키마 중심. S4 는 학생이 보는 최종 카드 형태(어떤 필드가 어떻게 노출되는지)를 명세.
- **P5c (캘린더 · 자료 골격)**: 두 탭의 공용 shell + 오너/학생 공용 렌더 규칙. S4 는 학생 시점의 자동 서브탭 숨김, 잠금 주차 처리, 본인만 CRUD 가능한 캘린더 규칙에 집중.
- **P5d (알림 · 커뮤니티 공용 골격)**: `TeacherNoticeList/StudentPostList/RepliesThread` 골격 + `StudentComposer` 기본. S4 는 `effectiveStudentView` 로 canManage=false 인 상태의 UI 변형(필터 pill 축소, 승인/거절 버튼 미노출, 자기 글만 삭제), `visibility` 자동 강제, AI 다듬기·미리보기·전송 상태 전이.
- **T4 (홈 편집 툴바)**: 툴바에서 `CustomEvent` 로 addBlock 호출. S4 는 그 이벤트 **리스너를 등록하지 않는 학생 브랜치**임을 명시.
- **T6 (교사 자료·알림·커뮤니티 쓰기층)**: 자료 업로드/노래 추가/링크·메모·임베드 추가/공지 작성/승인·거절/카드 편집 등 쓰기 경로. S4 는 그 쓰기 UI 가 학생에게 **숨겨지는 규칙**과 학생 자기 데이터에 대해서만 열리는 CRUD 만 정의한다.
- **S2 (학생 인증·초대·온보딩)**: `/auth?redirect=&course=`, `StudentOnboardingDialog` 프로필 최초 등록. S4 는 그 dialog 를 본인 카드 편집 트리거로도 재사용.
- **S3 (학생 내 공간)**: `/student-home` 캘린더·알림 요약 카드 등. 실제 반 내부 상호작용은 S4 로 위임.

---

## 5. Acceptance

1. 학생이 `/courses/:id` 또는 `/shared/course/:token` 로 진입하면 편집 스위치·`표지 편집` 버튼·`CourseModuleToolbar` 가 렌더되지 않는다.
2. `isShared && !user` 이면 `/auth?redirect=...&course=...&courseName=...` 로 즉시 replace.
3. 학생 프로필이 없으면 `StudentOnboardingDialog` 가 자동으로 열리고, 취소 시 `navigate(-1)` 로 나간다. 이 흐름 중에는 `course_members` 가 `role='student'` 로 upsert 된다.
4. HomeTab 학생 렌더에서 `ownerProfile.is_public_to_students === true && bio.trim()` 이면 교수 정보 블록이 `course_info` 앞(없으면 최상단)에 자동 삽입되고, DB 에는 저장되지 않는다.
5. `weekly_preview` 블록은 `course.start_date` 상태에 따라 세 가지 안내(미설정 / 시작 전 / 진행 중) 중 하나를 정확히 노출하고, 진행 중이면 현재 주차 카드와 상위 2곡을 렌더한다.
6. 캘린더 탭에서 학생은 자기가 만든 일정(`user_id === auth.uid()` 또는 `owner_id === auth.uid()`)에 대해서만 삭제 아이콘을 본다.
7. 캘린더 `일정 추가` 는 로그아웃 시 destructive 토스트, 로그인 시 `AddCalendarItemDialog` 로 진입해 저장하면 새 일정이 즉시 카드에 나타난다.
8. 강의 자료 탭에서 학생은 `count > 0` 인 서브탭만 보고, 전체가 0 이면 빈 카드가 노출된다. 서브탭 선택은 존재하는 첫 탭으로 자동 조정된다.
9. `LessonPlanPanel` 학생 뷰에서 `week.content.is_locked` 이거나 `getWeekDateRange` 시작일이 미래인 주차는 자물쇠 아이콘 + 비활성 Accordion 으로 잠기고, 잠금 토글 버튼은 렌더되지 않는다.
10. `CourseSongPanel` · `CourseAttachmentPanel` 학생 뷰에서 상단 액션 툴바(파일 업로드 / 노래 추가 / 링크·메모·영상 추가) 와 삭제 아이콘이 모두 숨겨진다.
11. 첨부 카드에서 `file` 은 새 탭 다운로드, `link` 는 새 탭 열기, `embed` 는 YouTube iframe(16:9), `text` 는 본문 + URL 자동 링크로 각각 정확히 렌더된다.
12. 알림 탭에서 `effectiveStudentView === true` 이면 `TeacherNoticeList` 필터 pill 4종만 보이고 승인/거절/편집/삭제 버튼은 모두 사라진다.
13. `StudentPostList` 는 `visibility === "public"` 이거나 본인 글만 보여주고, 본인 글에만 삭제 버튼이 나타난다.
14. `StudentComposer` 는 유형을 `결석`/`지각` 으로 바꾸면 즉시 `visibility` 가 `private` 로 자동 전환된다.
15. `AI로 다듬기` 실패 시 서버 `data.error` 메시지가 destructive 토스트로 노출되고, 성공 시 `AIPolishPanel` 이 결과를 보여주며 `교체`/`취소` 를 선택할 수 있다.
16. `RepliesThread` 는 자기 답글만 삭제 가능하며, `notification_replies` realtime 채널을 통해 새 답글이 즉시 반영된다.
17. `CommunityTab` 학생 뷰에서 본인 카드에는 `나` 뱃지가 붙고 hover 시 연필 아이콘이 노출되어 `StudentOnboardingDialog(existing=myProfile)` 로 자기 프로필을 편집할 수 있다. 다른 학생 카드의 연필 아이콘·직접 입력 버튼은 노출되지 않는다.
18. 모든 `postgres_changes` 채널(`course-home-*`, `course-lesson-plans-*`, `course-songs-*`, `course-attachments-*`, `notif-*`, `replies-*`) 이 학생 뷰에서도 정상적으로 구독되어 새 데이터가 즉시 반영된다.
