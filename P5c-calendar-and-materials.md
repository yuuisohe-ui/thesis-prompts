# P5c — 캘린더 & 강의 자료 (재현 프롬프트)

> 이 문서는 반 상세페이지의 두 번째·세 번째 탭인 **캘린더(`CalendarTab`)** 와 **강의 자료(`LessonMaterialsTab` = 강의안 + 노래 + 자료)** 를 하나의 재현 프롬프트로 통합한다. 두 탭은 `parseClassTime` / `getCourseSchedule` / `getWeekDateRange` 스케줄 헬퍼를 공유하며, 학생 뷰(`isStudentView`) 및 편집 모드(`editMode`) 규칙이 완전히 일치한다. P5a(반 껍데기 & Home) 와 P5b(홈 블록) 이 이미 존재한다고 가정한다.

---

## 1. Identity

`P5a` 에서 만든 `CourseDetail` 페이지의 두 개 하위 탭을 구현한다.

- 파일 위치
  - `src/components/courses/CalendarTab.tsx`
  - `src/components/courses/LessonMaterialsTab.tsx`
  - `src/components/courses/materials/LessonPlanPanel.tsx`
  - `src/components/courses/materials/CourseSongPanel.tsx`
  - `src/components/courses/materials/CourseAttachmentPanel.tsx`
  - `src/lib/courseSchedule.ts` (P5a에서 이미 생성)
- 부모에서 넘어오는 공용 props
  ```ts
  interface Props {
    course: { id: string; name?: string; class_time?: string | null; start_date?: string | null };
    editMode: boolean;                 // isOwner && editMode
    isStudentView?: boolean;           // shared-link 또는 non-owner
  }
  ```
- 학생 뷰 원칙: 편집/삭제/업로드 UI 는 모두 감춘다. 잠긴 주차는 여닫기가 disabled 되고, 자료 탭의 빈 서브탭은 자동으로 숨긴다.

## 2. Instructions

### 2.1 캘린더 탭

레이아웃: `Card` 안 `grid gap-6 lg:grid-cols-[minmax(0,1fr)_360px]` — 왼쪽 shadcn `<Calendar>`, 오른쪽 사이드 패널.

상태:
- `month`: 현재 표시 월 (기본 `startOfMonth(new Date())`).
- `selectedDate`: 클릭한 날짜.
- `items: CourseCalendarItem[]`: 해당 월의 커스텀 일정 (`course_calendar_items`).

정규 수업일 계산:
- `parseClassTime(course.class_time)` 로 `{ weekdayIndices, start, end }` 를 얻는다. "월/수 10:00~11:30" 같은 문자열을 지원.
- `isClassDay(date)` = 해당 요일이 `weekdayIndices` 에 있고, `course.start_date` 이후.
- 월 안의 정규 수업일 목록 → `classEntries`, 오늘 이전이면 `status="completed"`, 이후면 `"upcoming"`.

공휴일:
- `@hyunbinseo/holidays-kr` 의 `getHolidayNames(date)` 를 `try/catch` 로 감싼 `getHolidayNamesSafe` 로 호출.

`<Calendar>` 커스터마이징 요점 (그대로 사용):
- 셀 크기 `h-16`, day 클래스 `flex-col rounded-2xl`, `showOutsideDays`, `locale={ko}`.
- `modifiers = { classDay: 수업일, holiday: 공휴일, hasEvent: 커스텀 일정 }`.
- `modifiersClassNames`: 수업일 `bg-primary/15 text-primary`, 공휴일 `text-destructive`, 사용자 일정 `ring-2 ring-secondary`.
- `day_today` 는 `border-primary`, `day_selected` 는 `bg-primary text-primary-foreground`.

상단 요약 문구:
- 수업 요일이 있으면 `수업 요일: 월, 수 · 10:00 ~ 11:30 · 시작일 YYYY-MM-DD · 한국 공휴일 자동 표시`.
- 파싱 실패 시 `수업 시간에 요일 정보가 없어 공휴일만 먼저 표시하고 있습니다.`.
- 아래 배지 4개: `수업일` / `공휴일` / `사용자 일정` / `예정`.

오른쪽 사이드 패널:
- `selectedDate` 없음 → **"M월 수업 일정"** 목록 (버튼 클릭 시 그 날로 이동, 완료/예정 배지). 수업 요일이 없으면 안내문 "수업 시간에서 요일을 읽지 못했습니다. 예: 월/수 10:00~11:30".
- `selectedDate` 있음 → 헤더 `"M월 d일 (EEE)"` + `<X>` 해제 버튼.
  - 정규 수업일이면 `정규 수업일` 카드 + 시간.
  - 공휴일이면 destructive 카드 + 공휴일 이름.
  - `items` 중 같은 날짜인 것들을 렌더 (제목, 유형 배지, 시각 `HH:mm`, 설명). 본인이 만든 항목(`owner_id === user.id || user_id === user.id`)에만 삭제 아이콘.
  - 아무것도 없으면 "등록된 일정이 없습니다."
  - 하단에 `<Plus/> 일정 추가` — 로그아웃 상태면 토스트 후 return.

`일정 추가` 다이얼로그 (`AddCalendarItemDialog`):
- 필드: 제목, 유형(`class` / `exam` / `notice` / `assignment` / `other`), 종일 스위치, 시각(`type="time"`, 종일이면 숨김), 설명.
- 저장 시 `starts_at` = 종일이면 00:00, 아니면 선택 시각. `all_day`, `event_type`, `item_type: "lesson"`, `course_id`, `user_id` 를 함께 insert.
- 실패 시 error message 를 그대로 토스트.

데이터 로딩: `useEffect([course.id, month])` 에서 `fetchItems()` 가 해당 월 범위 `starts_at` 필터로 select.

### 2.2 강의 자료 탭 (LessonMaterialsTab)

`Tabs` 3개: `강의안(plans)` / `노래(songs)` / `자료(attachments)`. 각 트리거는 아이콘(`BookOpen`/`Music`/`Paperclip`) + 라벨 + 개수 배지.

- 서브 컴포넌트에 `onCountChange` 콜백을 넘겨 `planCount / songCount / attachCount` 를 실시간으로 부모가 보관한다.
- **학생 뷰 규칙**: 개수가 0 인 서브탭은 트리거 자체를 감춘다. 세 개가 모두 0 이면 카드 하나로 `아직 강의 자료가 등록되지 않았습니다.` 를 보여주되, 개수 콜백을 유지하기 위해 세 패널은 `hidden` div 안에서 계속 마운트한다.
- 현재 서브탭이 학생 뷰에서 숨겨졌다면 다른 서브탭으로 자동 스위치.
- 세 패널은 **항상 마운트**하고 비활성 서브탭은 `hidden` 로만 감춘다 (실시간 카운트 유지).

### 2.3 강의안 패널 (LessonPlanPanel)

로직:
- `lesson_plans` 에서 `course_id === course.id` 인 모든 강의안을 최신순으로 로드 → `plans`.
- 강의안이 여러 개면 상단에 `Select` 로 선택. 초기 선택은 첫 번째.
- 선택된 강의안의 `lesson_weeks` 를 `week_number` 오름차순으로 로드.
- `supabase.channel("course-lesson-plans-<id>")` 로 `lesson_plans (filter=course_id)` 와 `lesson_weeks` 변경을 구독해 자동 재조회.
- 빈 상태: `연결된 강의안이 없습니다` + 안내문.

주차 스케줄 표시 (매우 중요):
- `getCourseSchedule(course.start_date, planWeeks.length)` 결과에 따라 카드 위 안내 배너를 다르게 렌더.
  - `no_start_date` → amber 배너 "수업 시작일이 설정되지 않았습니다. 홈 > 과정 정보에서 시작일을 입력하면 매주 학습 일정이 표시됩니다."
  - `before_start` → primary 배너 "수업은 YYYY년 M월 d일 (EEE)에 시작됩니다. 시작 전에는 주차별 내용이 잠금 상태로 표시됩니다."
  - `in_progress` → 배너 없음.

주차 아코디언 (`Accordion type="single" collapsible`):
- 각 아이템의 왼쪽 뱃지: 시작일 있으면 `getWeekDateRange` 로 `M.d – M.d` 표시.
- 상태 판별
  - `isLocked = week.content?.is_locked === true`
  - `notStartedYet = weekRange && new Date() < weekRange.start`
  - `blockedForStudent = isStudentView && (isLocked || notStartedYet)`
- `blockedForStudent` 이면 트리거 `disabled`, 왼쪽 원 안 아이콘을 `Lock` 으로, 캡션을 "아직 시작되지 않은 주차입니다" 또는 "강사가 이 주차를 잠갔습니다" 로 스와핑.
- 오른쪽 배지: `week_type` 라벨 (`오리엔테이션 / 중간고사 / 기말고사 / 정규 수업`) 와 variant.
- 교사에게만 잠금 토글 버튼 노출. 클릭 시 `content.is_locked` 를 flip 하여 `lesson_weeks.update` — 낙관적 업데이트 + 실패 시 재조회.
- 열려있는 주차의 `AccordionContent` 는 잠긴 주차일 때 교사에게 amber info 배너 노출 후 `WeekDetailView` 를 `embedded` + `isStudentView` 로 렌더.

### 2.4 노래 패널 (CourseSongPanel)

- `course_material_items` 에서 `material_type='song'` 필터로 조회, `sort_order` → `created_at` 순.
- 교사에게만 `노래 추가` 버튼 → `SongPickerDialog(moduleType="course", moduleTitle="강의 자료")`.
- 추가 시 payload: `{ course_id, material_type:"song", song_id, title, description: "artist|hsk|videoId", url: youtube_url, sort_order: items.length }`.
- 그리드 3열 (sm 2, lg 3). 카드 = YouTube 썸네일 `mqdefault.jpg` + 우상단 HSK 배지 + 제목/가수. 카드 클릭 → `navigate("/songs/" + song_id)`.
- `description` 에서 videoId 를 파싱, 없으면 `url` 에서 `extractVideoId` 로 추출.
- 삭제 버튼(교사만) `confirm()` 후 delete.
- 빈 상태: `Music` 아이콘 + "아직 추가된 노래가 없습니다".

### 2.5 자료 패널 (CourseAttachmentPanel)

지원 타입 4개: `file` / `link` / `text` / `embed`.

교사 툴바 (`!isStudentView` 일 때만):
- `파일 업로드` 버튼 → `<input type=file accept={ACCEPTED_FILE_EXT}>`.
- 드롭다운 `추가` → 링크 / 메모 / 영상 임베드 세 항목.

파일 업로드:
- 크기 검증 `file.size > MAX_FILE_BYTES` → 토스트.
- 파일명 안전화: `name.replace(/[^\w.\-]+/g, "_")`.
- 스토리지 경로 **반드시** `${user.id}/courses/${course.id}/${Date.now()}_${safeName}` (RLS: 첫 폴더 = auth.uid()).
- 버킷 `materials`, `upsert:false`, contentType 지정. `getPublicUrl` 로 URL 획득.
- 인서트: `{ course_id, material_type:"file", title:file.name, description:"${mime}|${size}|${path}", url: publicUrl, sort_order: items.length }`.
- 삭제 시: `description` 에서 3번째 필드(path) 를 꺼내 `storage.from('materials').remove([path])` 후 DB delete.

링크/메모/임베드 다이얼로그:
- 공통 필드: 제목, URL(메모 제외), 설명/내용.
- 유효성: 링크/임베드는 URL 필수, 메모는 내용 필수.
- 저장 payload: `{ material_type: dialogType, title: title || url || "메모", description, url, sort_order }`.

렌더링:
- 각 항목은 좌측 아이콘 원 + 본문 + 우측 액션 카드.
- **file**: `getFileMetaByName(title)` 로 아이콘/라벨. 우측에 `<Download/> 다운로드` 링크 (`download` attribute + `target=_blank`). 부제 `${label} · ${formatBytes(size)}`.
- **link**: 우측에 `<ExternalLink/> 열기` 링크. 본문에 URL 과 설명 (line-clamp-2).
- **embed**: `getYoutubeId(url)` 로 videoId 추출 후 `<iframe src="https://www.youtube.com/embed/{id}" aspect-video>` 임베드 카드. 본문 아래에 설명.
- **text**: 아이콘 원 = 메모 색. 본문에 제목 + `whitespace-pre-wrap` 설명, URL 은 `linkify(text)` 로 `<a target=_blank>` 자동 변환.
- 교사 hover 시 우측에 `Trash2` 삭제 아이콘 노출.

실시간 반영: 세 패널 모두 `supabase.channel("course-<name>-<id>").on postgres_changes` 로 자기 테이블을 구독하고 refetch.

## 3. Examples

**시간표 파싱 → 캘린더 표시**
- `class_time = "월/수 10:00~11:30"`, `start_date = "2026-03-02"` → 3월 2일(월)부터 매주 월/수요일이 primary/15 배경으로 하이라이트, 시작일 이전 월/수는 표시되지 않음.
- 사이드 패널 "3월 수업 일정" 리스트에 총 8회 (완료 6, 예정 2) 가 나열, 8월 15일은 destructive 색 + "공휴일 · 광복절" 라벨.

**주차 잠금 → 학생 뷰**
- 교사가 5주차를 잠그면 `lesson_weeks.content = { ..., is_locked: true }`. 학생 계정에서 강의안 탭을 열면 5주차 아코디언은 회색 opacity-70, 자물쇠 아이콘, 캡션 "강사가 이 주차를 잠갔습니다", 클릭 불가.

**파일 업로드 경로**
- 로그인 유저 `abc-123` 이 `강의계획서.pdf` 를 반 `xyz-789` 에 업로드하면 저장 경로는 `abc-123/courses/xyz-789/1747-강의계획서.pdf`. RLS 는 첫 폴더가 auth.uid() 인지로 write 를 허용한다.

## 4. Context

### 4.1 스키마

`course_calendar_items`
- `id uuid`, `course_id uuid`, `user_id uuid` (작성자), `owner_id uuid nullable` (수업 오너 캐시), `title text`, `description text nullable`, `starts_at timestamptz`, `all_day boolean`, `event_type text` (`class|exam|notice|assignment|other`), `item_type text` (본 화면에선 `"lesson"` 고정).
- RLS: 수업 오너 또는 작성자만 update/delete, 반원은 select.

`course_material_items`
- `id`, `course_id`, `material_type` (`song|file|link|text|embed`), `song_id` nullable, `title`, `description`, `url`, `sort_order`, `created_at`.
- 노래는 `song_id` + `description="artist|hsk|videoId"` 로 힌트를 보관. 파일은 `description="mime|size|path"`.

`lesson_plans` / `lesson_weeks`
- `lesson_plans.course_id` 로 반과 연결. `lesson_weeks.content` 는 JSON, 잠금 정보는 `content.is_locked`.

Storage 버킷 `materials`
- Public read 는 켜지 않음. `getPublicUrl` 로 signed-less URL 을 얻지만 RLS 로 접근 제한.
- 정책: `bucket_id='materials' AND auth.uid()::text = (storage.foldername(name))[1]` — write/read.

### 4.2 헬퍼

`src/lib/courseSchedule.ts` 은 P5a 에 이미 정의되어 있어 그대로 재사용한다.
- `parseClassTime`, `filterClassMeetingDates`, `getCourseSchedule`, `getWeekDateRange`.

`src/lib/file-icons.tsx` 는 `getFileMetaByName / linkMeta / noteMeta / embedMeta / ACCEPTED_FILE_EXT / MAX_FILE_BYTES / formatBytes` 를 export.

`WeekDetailView` (P5a 외부, `src/components/lessons/WeekDetailView.tsx`) 는 강의안 상세 렌더러이며, 여기서는 `embedded + isStudentView` 프롭으로 재사용한다.

### 4.3 상호작용 규약

- `LessonMaterialsTab` 는 P5a 의 `CourseModuleToolbar` 이벤트와 무관하다. 홈 탭에서만 툴바가 뜨므로 여기서는 참조하지 않는다.
- `CalendarTab` 은 편집 모드와 무관하게 동일하게 동작한다 (일정 추가/삭제 권한은 항상 작성자 기준).
- 세 자료 서브패널의 `onCountChange` 는 배지와 학생 뷰 자동-스위칭에만 쓰인다.

## 5. Acceptance

- [ ] `class_time` 문자열을 파싱해 요일별 수업일이 캘린더에 primary/15 로 표시되고, `start_date` 이전 날짜는 정규 수업일로 표시되지 않는다.
- [ ] 공휴일이 `text-destructive` 로 마킹되고, 선택 시 오른쪽 패널에 공휴일 이름이 노출된다.
- [ ] `일정 추가` 다이얼로그로 등록한 커스텀 일정이 캘린더에 `ring-2 ring-secondary` 로 표시되고, 본인만 삭제 아이콘이 보인다.
- [ ] 강의안 탭은 여러 강의안일 때 Select 로 전환되고, 각 주차 아코디언 캡션에 `M.d – M.d` 범위가 표시된다.
- [ ] `getCourseSchedule` 결과에 따라 상단 배너 3가지 상태(없음/시작 전/진행 중)가 정확히 스위칭된다.
- [ ] 학생 뷰에서 잠긴 주차 또는 아직 시작 안 된 주차는 아코디언이 disabled 되고 캡션이 대체 문구로 바뀐다.
- [ ] 교사만 주차 잠금 토글, 노래 추가/삭제, 파일 업로드/링크/메모/임베드 추가, 자료 삭제 UI 가 노출된다.
- [ ] 파일 업로드 경로가 `${uid}/courses/${courseId}/...` 로 생성되고, 삭제 시 스토리지 파일도 함께 제거된다.
- [ ] YouTube URL 을 `embed` 로 추가하면 카드 내부에 `<iframe>` 임베드 플레이어가 렌더된다.
- [ ] 학생 뷰에서 세 서브탭이 모두 비어 있으면 대체 카드가 표시되고, 카운트가 채워지면 자동으로 해당 탭이 노출된다.
- [ ] `course_calendar_items`, `lesson_plans`, `lesson_weeks`, `course_material_items` 각각의 postgres_changes 를 구독하여 다른 세션의 편집이 실시간 반영된다.
