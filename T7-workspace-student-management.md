# T7 · 워크스페이스 「학생 관리」 섹션 (교사 전용)

> 대상 컴포넌트: `src/components/workspace/StudentSection.tsx`
> 부속 다이얼로그: `src/components/students/BulkStudentDialog.tsx`
> 부속 패널: `src/components/students/StudentCsvImportPanel.tsx`
> 상위 진입: `src/pages/Workspace.tsx` 의 `<StudentSection />` (스크롤 앵커 `#student-section`)

이 문서는 워크스페이스 하단 「학생 관리」 섹션을 재현하기 위한 프롬프트다. 현재 플랫폼에 **실제로 존재하는 입구/화면만** 포함한다.

---

## 1. Identity

`StudentSection` 은 워크스페이스 홈에서 교사가 자신이 소유한 반의 학생 명단을 한 곳에서 조망·등록·관리하는 섹션이다. 두 가지 표시 모드(**반별 보기 / 전체 학생**) 와 한 개의 등록 진입점(**학생 목록 일괄 생성**) 을 가진다.

- 진입: `/workspace` 페이지의 마지막 섹션. `refreshKey` prop 으로 상위(반 생성 후)에서 강제 리로드.
- 대상 데이터:
  - `courses` (교사가 소유한 반 = `owner_id === user.id`, `deleted_at is null`)
  - `course_student_profiles` (`deleted_at is null`)
  - `fetchHiddenIds("courses")` · `fetchHiddenIds("course_student_profiles")` 로 숨김 필터
- 빈 상태: 소유한 반이 0개일 때 공개 반 하나를 **추천(`isRecommended: true`)** 으로 보여준다.

## 2. Instructions

### 2.1 헤더

```
[Users icon(#4f52c8)] 학생 관리   [pill: 전체 {rows.length}명 · bg #eeeffe / text #3739a8]
반에 등록된 학생을 반별 또는 전체로 확인할 수 있어요. 반 링크 공유 또는 학생 목록 일괄 생성으로 학생을 등록하세요.
                                                          [Upload] 학생 목록 일괄 생성  ← wsBtn.indigo, data-tour="students-csv"
```

우측 버튼은 `BulkStudentDialog` 를 연다(§4).

### 2.2 탭 스위치 (`반별 보기 / 전체 학생`)

`bg-[#eeecea]` inline-flex pill. 선택 탭은 `bg-white shadow-sm text-[#1a1825]`, 비선택은 `text-[#6b6880]`. 상태 `tab: "byclass" | "all"`, 초기값 `"byclass"`.

### 2.3 반별 보기 (`tab === "byclass"`)

- 데이터: `displayCourses = courses.length > 0 ? courses : (recommended ? [{...recommended, isRecommended: true}] : [])`.
- 반 하나 = shadcn `AccordionItem` (`type="multiple"`, 병렬 오픈).
- 헤더 행 (accordion trigger, `[&>svg]:hidden` 로 기본 화살표 제거하고 오른쪽 `ChevronRight` 를 직접 렌더):
  - 반 이름 (13px bold) + (추천 pill) + `Badge outline`: `{students.length}명`
  - 서브라인: `class_time` (11px muted)
  - 우측 액션:
    - `반 링크 복사` (wsBtn.indigo) → `navigator.clipboard.writeText(`${origin}/shared/course/${share_token}`)`, toast `"링크 복사 완료"`
    - 소유 반일 때만 (`!isRecommended`): `Pencil` (수정 → `EditCourseDialog`) / `Trash2` (삭제 → `AlertDialog`)
  - **stopPropagation 필수**: 액션 아이콘 클릭이 아코디언을 접었다 폈다 하지 않도록 `role="button" tabIndex={0}` span 으로 감싸고 `onClick`, `onKeyDown` 모두 `stopPropagation`.
- Accordion Content 3분기:
  1. `isRecommended` → 안내문 + `반 내용 보기` (`navigate(/courses/${c.id})`).
  2. 학생 0명 → `<EmptyClassCards>` (반 링크 공유 / 학생 목록 일괄 생성 2개의 큰 카드).
  3. 학생 있음 → `students.map(<StudentRowItem>)` (아바타·이모지·이니셜 fallback + 학번·학과·수준 요약 + `가입/수동` 배지).
- 섹션 하단: `+ 새 반 만들기` 점선 카드 → `onOpenCreateCourse()` (부모가 `CreateCourseDialog` 오픈).

### 2.4 반 삭제

`AlertDialog` → `supabase.rpc("move_to_trash", { _table: "courses", _id })` → toast `"휴지통으로 이동되었습니다"` → `load()` 재조회. 실제 삭제 아님(휴지통 7일 후 자동).

### 2.5 전체 학생 (`tab === "all"`)

- 검색 인풋 (`이름, 학번, 학과, 과정명으로 검색...`), 클라이언트 필터.
- 결과 0명 → `<EmptyStudents>` (반 링크 초대 / CSV 일괄 등록 2카드; 반 링크는 `courses[0].share_token`).
- 결과 있음 → shadcn `Table` 렌더: 아바타 · 이름 · 반 · 학번 · 학과 · 수준 · 가입/수동.

### 2.6 데이터 로드 (`load()`)

```
1) courses: SELECT * FROM courses WHERE deleted_at IS NULL
             ORDER BY sort_order ASC, created_at DESC
   → hiddenCourses = fetchHiddenIds("courses") 로 필터
   → myCourses = allCourses.filter(owner_id === user.id)
2) recommended = myCourses.length === 0 이면 owner_id IS NULL 중 랜덤 1개
3) course_student_profiles: SELECT id, full_name, student_number, department,
     avatar_url, member_user_id, language_level, study_years, gender, emoji,
     course_id, created_at WHERE deleted_at IS NULL ORDER BY created_at DESC
   → hidden 필터, course_name / share_token 매핑
```

### 2.7 자원 재사용

- `EditCourseDialog` 는 T5 참고.
- `BulkStudentDialog` (§4) 는 여기 T7 소유.
- `wsBtn.indigo / ghost / outline / green` 은 `src/components/workspace/tokens.ts` 의 통일 버튼 토큰.

## 3. Examples (UI 사양)

```text
─────────────────────────────────────────────────────────
👥 학생 관리  전체 4명                    [⬆ 학생 목록 일괄 생성]
반에 등록된 학생을 반별 또는 전체로 확인할 수 있어요. …
─────────────────────────────────────────────────────────
[반별 보기]  전체 학생

▽ 노래로 배우는 중국 역사   [1명]   [🔗 반 링크 복사] [✎] [🗑] >
    수 10:00~12:00
    ├ 홍길동   2024001 · 중어중문학과 · HSK 4-6         [가입]
▽ 한국 역사                [0명]   [🔗 반 링크 복사] [✎] [🗑] >
    월 18:00~20:00
    ┌ 반 링크 공유           ┐ ┌ 학생 목록 일괄 생성        ┐
    └ (링크 복사)           ┘ └ (CSV 업로드)               ┘

+ 새 반 만들기 (점선 카드)
```

## 4. `BulkStudentDialog` (모달 · 3단계 위저드)

**상태**: `step: 1|2|3` , `randomEmoji: boolean(true)`. 임포트 중 (`panelRef.current.isImporting()`) 이면 다이얼로그 닫기 무시.

### Step 1 · 기능 소개
- `CSV 작성 → 업로드 → 학생 카드 생성` 3단 아이콘 스트립.
- `이 기능으로 할 수 있는 일` bullet 4개.
- 버튼: `취소` / `다음 →`.

### Step 2 · 템플릿 다운로드
- 컬럼 스펙 표 (필수/선택 배지):
  | 컬럼 | 필수 | 설명 |
  |------|------|------|
  | `course_name` | 필수 | 학생을 배정할 과정 이름 (정확히 일치) |
  | `full_name` | 필수 | 학생 이름 |
  | `student_number` | 선택 | 학번 |
  | `department` | 선택 | 학과 |
  | `hsk_level` | 선택 | `HSK 1-3` / `HSK 4-6` / `HSK 7-9` |
  | `topik_level` | 선택 | `TOPIK 1-2` / `TOPIK 3-4` / `TOPIK 5-6` |
  | `study_years` | 선택 | 학습 기간 (`0.5, 1, 2, 3, 4, 5`) |
  | `gender` | 선택 | `male / female / other` |
  | `emoji` | 선택 | 단일 이모지. 비우면 랜덤 배정 |
- `예시 템플릿 다운로드 (.csv)` (점선 카드):
  - CSV 는 `BOM(\ufeff) + CRLF`, 헤더 순서는 위 표와 동일, 예시 2행 포함(`중국어 회화 1 / 홍길동 / …`, `중국어 회화 1 / 김영희 / …`).
  - `,`, `"`, `\n` 포함 값은 큰따옴표로 감싸고 `"` 는 이스케이프.
- `이모지 자동 랜덤 배정` 토글(기본 ON).
- 버튼: `이전` / `업로드하러 가기 →`.

### Step 3 · 업로드
`<StudentCsvImportPanel randomEmoji={} userId={} onImportComplete={} onClose={} />` 를 마운트.

## 5. `StudentCsvImportPanel` — 파싱·중복·삽입 파이프라인

### 5.1 파싱
- `papaparse` 로 `header:true, skipEmptyLines:true`.
- 헤더 정규화: `Object.entries` → `key.toLowerCase().trim()`.
- `full_name` 이 비어있는 행은 버린다.

### 5.2 코스 매칭
- 사용 가능한 반: `SELECT id, name FROM courses WHERE deleted_at IS NULL`.
- 우선순위: `row.course_id` → `nameToId.get(row.course_name.trim())`.
- 매칭 실패: `results.push({status:"error", message:"과정을 찾을 수 없습니다"})`.

### 5.3 중복 판별
같은 `course_id + full_name` (학번이 있으면 `+ student_number`) 이 이미 `deleted_at IS NULL` 로 존재하면 `status:"duplicate", message:"이미 등록된 학생"`.

### 5.4 값 정규화
- `normalizeLevel(hsk_level, HSK_VALID)` : 대소문자·공백 무시 매칭, 실패 시 `""`.
- `normalizeLevel(topik_level, TOPIK_VALID)` 동일.
- `composeLanguageLevel(hsk, topik)` : `"HSK 4-6 / TOPIK 3-4"` 형태, 둘 다 비면 `null`.
- `parseStudyYears`: 숫자 그대로 → `6개월` 문구는 `0.5` → 아니면 정규식 `(\d+(?:\.\d+)?)`.
- `parseGender`: `male|남|남자 → "male"`, `female|여|여자 → "female"`, 그 외 `"other"`.
- `emoji`: 값이 있으면 그대로, 비어있고 `randomEmoji` 이면 `STUDENT_EMOJIS` 배열에서 랜덤.

### 5.5 sort_order
반별 카운터를 lazy 초기화: 첫 삽입 직전 `MAX(sort_order)` 를 조회하여 `sortCounters` 에 저장, 이후 `+1` 씩 증가.

### 5.6 삽입
```
INSERT INTO course_student_profiles (
  course_id, full_name, student_number, department,
  language_level, study_years, gender, emoji, sort_order
) VALUES (...);
```

### 5.7 UI
- 파일 선택 후 상단에 파일명 + 유효 행 수 미리보기(`(코스명)` 포함).
- 진행 중 `Progress` 바 + `{progress}/{total}`.
- 완료 요약 배지: 성공/중복/오류 카운트, 각 행별 아이콘(`CheckCircle2 / AlertTriangle / XCircle`) 리스트.
- 완료 후 `onImportComplete()` 호출 → `StudentSection.load()` 재실행.

## 6. Context (RLS / 데이터 소유)

- `courses.owner_id = auth.uid()` 만 자기 반. RLS 정책상 다른 교사 소유 반은 조회 자체가 안 되지만 공개 템플릿(`owner_id IS NULL`) 은 열람 허용 → 빈 상태 추천에 사용.
- `course_student_profiles` INSERT 는 대상 `courses` 의 owner 여야 통과. CSV 임포트에서 남의 반 이름을 넣으면 코스 매칭 단계에서 걸러진다.
- `move_to_trash` RPC 는 SECURITY DEFINER 로 owner-only 검증.
- `share_token` 은 반 링크 (`/shared/course/{token}`) 발급 UUID. 학생은 이 링크로 `StudentOnboardingDialog` (T5) 를 거쳐 자기 프로필을 만든다.

## 7. Acceptance

- [ ] 상단 헤더에 `전체 N명` pill 이 표시되고 N은 로드된 학생 수와 일치한다.
- [ ] `학생 목록 일괄 생성` 클릭 → `BulkStudentDialog` step 1 이 뜨고 `data-tour="students-csv"` 가 붙어 있다.
- [ ] `반별 보기` 에서 링크/수정/삭제 클릭이 아코디언 열림 상태를 바꾸지 않는다 (`stopPropagation`).
- [ ] 학생 0명 반은 `EmptyClassCards` 2카드가 뜨며, 반 링크 카드는 `share_token` 없을 때 disabled.
- [ ] 삭제는 항상 `move_to_trash` 를 호출 (물리 삭제 금지).
- [ ] 소유 반 0개인 신규 교사에게는 공개 반 1개가 `추천` 배지와 함께 등장하고, 액션 버튼(수정/삭제) 은 숨겨진다.
- [ ] CSV 임포트 시 `course_name` 미매칭·중복·성공 각각의 결과 배지가 개별 행에 표시된다.
- [ ] 임포트 완료 후 `refreshKey` 변화 없이도 섹션 데이터가 자동 재조회된다 (내부 `load()`).

---

## 8. P·T 시리즈와의 차이점

- **T5 (course-basics-and-membership)** : `courses` 행 자체의 생성/편집/조인/온보딩 다이얼로그를 다룬다. T7 은 이미 존재하는 반에 대해 **학생 명단을 대량으로 채우는 워크스페이스 뷰**만 담당한다. `EditCourseDialog` 호출부만 공유하고 구현은 T5 소유.
- **T2 (workspace)** : 워크스페이스 상단 배너/내 반 섹션/내 강의안 섹션의 전체 레이아웃을 다룬다. T7 은 그 하위 한 섹션(`#student-section`)만 단독으로 재현한다. 워크스페이스 전체 CSS 토큰(`wsBtn`) 은 T2 소유.
- **P5c (calendar-and-materials)** : `LessonMaterialsTab` 등 반 상세 내부 탭을 설명한다. 학생 명단과 무관하다.
- **P5d (notifications-and-community)** : 반 상세의 「우리 반」 탭에서 학생 카드 그리드를 렌더한다(`CommunityTab`). T7 의 「전체 학생」 표와는 목적이 다르다 — P5d 는 학생 페르소나 표현/편집, T7 은 교사가 여러 반을 가로질러 조망·등록.
- **T6 (course-workspaces-teacher-edits)** : `CommunityTab` 편집 모드에서 교사가 개별 학생 카드를 직접 입력한다. 그 UI 는 학생 1명 단위 편집이지 CSV 대량 등록이 아니다. **CSV 일괄 등록/템플릿/파싱 파이프라인은 오직 T7 소유**.
- **`src/pages/Students.tsx`** : 별도의 `/students` 페이지도 존재하나 현재 사이드바 라우팅에서 노출되는 진입점은 워크스페이스 섹션이 우선이며, 그 페이지는 동일 `BulkStudentDialog` + `course_student_profiles` 조회를 카드/표로 단순 렌더할 뿐이다. 재현 프롬프트는 T7 하나로 충분하다 (해당 페이지가 필요하면 이 문서 §5 파이프라인을 그대로 재사용).
