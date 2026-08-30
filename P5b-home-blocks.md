# P5b · 홈 탭 블록 카탈로그·렌더러·인라인 에디터 (Home Block Catalog / Renderer / Inline Editor)

> 본 문서는 부록 프롬프트 표준 템플릿(`00-template.md`) 5-Section 골격을 따른다. 이론적 근거(OpenAI 4-요소, Lovable 실천 원칙, IEEE 830 §4.3.6, Cohn 2004)는 `00-template.md`를 참조.
>
> 상위 문서: **P5a**(반 상세 shell + 홈 탭 편집 모드). 본 문서는 P5a의 편집 툴바가 실제로 조작하는 **개별 블록의 카탈로그·렌더링·인라인 편집 UI**를 재현 가능한 수준으로 확정한다.

---

## ① Identity (신원)

당신은 대한민국 대학의 K-Chinese/K-Korean 교육 UX를 담당하는 시니어 프론트엔드 엔지니어다. 스택은 **React 18 + Vite 5 + TypeScript 5 + Tailwind CSS v3 + shadcn/ui + Supabase JS v2**. 본 작업의 대상은 「멜로디 클래스」 반 상세 페이지의 **홈 탭(course-home)**을 구성하는 개별 **블록(course_home_blocks 행)**의 세 축 — (1) **카탈로그**(추가 가능한 블록 타입 목록과 기본값), (2) **렌더러**(학생·교사 모두에게 보이는 최종 UI), (3) **인라인 에디터**(편집 모드에서 각 블록을 그 자리에서 수정) — 를 한 파일 단위로 재구성한다. 카피는 순수 한국어이며 이모지 아이콘은 카드 헤더 좌측 이모지 1개(예: `🎯`, `📌`)만 허용한다.

## ② Instructions (지시)

### 2.1 산출물 (Prompt by Component, Not Page)

정확히 아래 6개 파일을 생성한다. 다른 파일은 만들지 않는다.

```
src/components/courses/home-blocks/
├── types.ts                       # 블록 타입 문자열 유니온 + 관련 인터페이스
├── quizPresets.ts                 # 3개 퀴즈 프리셋 상수
├── blockCatalog.tsx               # 7개 카탈로그 정의 + createBlockInsert + buildBlockInsertPrefilled
├── defaultBlocks.ts               # 반 생성 직후 삽입되는 7개 시드 블록 (async)
├── FocusedHomeBlocks.tsx          # WeeklyPreviewBlock (핵심). 레거시 CourseHeader 등은 미사용
└── CourseHomeBlockRenderer.tsx    # switch(block.block_type) 진입점

src/components/courses/
└── CourseHomeBlockEditorItem.tsx  # dnd-kit 정렬 가능한 인라인 편집 카드
```

### 2.2 원자적 UI 규칙 (Speak Atomic)

- **카드 헤더**: 이모지 1개(`🎯 📌 📅 🎵 📖 👨‍🏫` 등) + 블록 제목 문자열 → `p.text-sm.font-bold`, `gap-1.5`.
- **course_info 카드**: `grid.grid-cols-2.gap-2.5`, 셀은 `rounded-lg.bg-muted/60.px-3.5.py-3`, 라벨은 `text-[11px].font-semibold.uppercase.tracking-wider.text-muted-foreground`, 값은 `text-sm.font-semibold`.
- **goals 항목**: `rounded-lg.bg-emerald-50 dark:bg-emerald-950/30.px-3.5.py-2.5`, 앞에 `1.5×1.5.rounded-full.bg-emerald-500` 도트.
- **curriculum 항목**: `border-l-[3px].border-primary`, 좌측에 `W1`/`W2` 라벨 `text-[11px].font-bold.text-primary`.
- **vocab_grid**: `grid-cols-3`, 셀은 `rounded-lg.bg-amber-50 dark:bg-amber-950/30`, 汉字는 `text-lg.font-bold.text-amber-600`, pinyin/한국어는 `text-[11px]`.
- **weekly_preview**: 강의안·이번 주 노래(최대 2개, `grid.sm:grid-cols-2`)를 카드로. `courseSchedule.ts`의 상태(`no_start_date` / `before_start` / `in_progress`)에 따라 안내 문구 분기.
- **CourseHomeBlockEditorItem**: 상단 바(드래그 핸들 `GripVertical` + `Badge`{block.block_type} + 제목 인풋 + 삭제 아이콘), 하단은 **좌: 폼 / 우: 실시간 미리보기** 2단(`lg:grid-cols-[minmax(0,1fr)_minmax(320px,360px)]`).
- **드래그**: `@dnd-kit/sortable` `useSortable({ id: block.id })`, 드래깅 중 `opacity-60`.

### 2.3 강제 제약 (Design with Real Content)

- semantic token만 사용. `bg-[#…]`, `text-white`, `bg-black`, `text-black` 하드코드 금지. `emerald-50/500`, `amber-50/500`처럼 Tailwind 팔레트는 허용(다크 모드 병기 필수: `dark:bg-emerald-950/30`).
- `<h1>` 금지(페이지 h1은 P5a `CourseDetail`이 가진다). 블록 제목은 모두 `<p>` 또는 `<h3>`(선택)로.
- lorem ipsum 금지. 모든 placeholder는 실제 한국어(예: "예: 15주", "예: 20명", "汉字", "pinyin", "한국어 뜻").
- 모든 Supabase 호출은 `AbortController` 없이 짧게 유지하되 `.eq('course_id', courseId)`로 스코프 필수.
- `block.content`는 항상 `Record<string, any>`로 취급하고 각 case에서 안전한 기본값을 반환.

## ③ Examples (예시)

### 3.1 확정 카피 표 (한국어 원문, Design with Real Content)

**카탈로그(`blockCatalog.tsx`) — 순서 고정 (편집 툴바에서 이 순서로 노출됨)**

| type | label | description | icon |
|---|---|---|---|
| `course_header` | 상단 헤더 | Hero 텍스트 제어 (과정당 1개) | `LayoutTemplate` |
| `course_info` | 과정 정보 | 기간·학기·시간 요약 | `Info` |
| `goals` | 학습 목표 | 학기 학습 목표 리스트 | `Target` |
| `weekly_preview` | 이번 주 학습 | 이번 주 강의안과 노래 | `ListMusic` |
| `curriculum` | 주차별 커리큘럼 | 강의안의 주차 목록 | `CalendarDays` |
| `songs_list` | 수록 노래 | 강의안에 연결된 곡 | `Music` |
| `vocab_grid` | 핵심 어휘 | 강조할 어휘 카드 | `BookOpen` |

**`course_header` 특수 규칙:** 툴바에 노출되지만 **과정당 최대 1개**만 존재. 이미 있으면 재추가 시 토스트 `상단 헤더는 과정당 1개만 추가할 수 있습니다.` 렌더러의 `course_header` case 는 `return null` 이며 실제 Hero 는 P5a의 `CourseDetail` 이 이 블록의 content 를 읽어 렌더한다. 표지 이미지 편집은 별개의 `CourseHeroEditor` (P5a).

**전체 `CourseHomeBlockType` 유니온(카탈로그 외 legacy/시스템 타입 포함, 렌더러 switch는 모두 지원):**

```
course_header | weekly_preview |
text | image | video | file | lyrics | notice_banner | divider |
quiz | students | lesson_plan_overview | weekly_lessons |
course_info | goals | curriculum | songs_list | vocab_grid | teacher_info
```

> 이전 버전에 존재하던 `latest_announcement` / `question_comments` 는 **제거**되었다. 유니온·카탈로그·렌더러·에디터 어디에도 존재해서는 안 되며, 레거시 데이터가 남아 있을 경우 렌더러는 `null` 을 반환해 안전하게 무시한다.

**시드 블록(`defaultBlocks.ts`) — 반 생성 직후 sort_order 0~6**

| sort_order | block_type | title |
|---|---|---|
| 0 | `course_header` | 상단 헤더 |
| 1 | `course_info` | 과정 정보 |
| 2 | `goals` | 학습 목표 |
| 3 | `weekly_preview` | 이번 주 학습 |
| 4 | `curriculum` | 주차별 커리큘럼 |
| 5 | `songs_list` | 수록 노래 |
| 6 | `vocab_grid` | 핵심 어휘 |

`createDefaultCourseHomeBlocks` 는 **async** 이며 각 항목을 `buildBlockInsertPrefilled` 로 만든다 → 연결된 강의안이 있으면 `curriculum` 의 weeks, `songs_list` 의 songs, `weekly_preview` 의 `selected_plan_id`, `course_info` 의 학기/시간이 자동 채워진다.

**퀴즈 프리셋(`quizPresets.ts`)**

| id | title | questions | attempts | type |
|---|---|---|---|---|
| `moon-vocab` | 月亮代表我的心 어휘 퀴즈 | 10 | 25 | 어휘 |
| `apple-listening` | 小苹果 듣기 퀴즈 | 8 | 18 | 듣기 |
| `friend-pattern` | 朋友 문법 퀴즈 | 12 | 30 | 문법 |

**course_info 라벨(고정)**: `총 기간 / 수강 대상 / 학기 / 수업 시간 / 시작일`.
값이 비면 그 셀은 렌더하지 않는다(빈 카드 방지).

**WeeklyPreview 안내 문구(상태별, 정확히 이 문자열)**

| state | text |
|---|---|
| `no_start_date` | 수업 시작일이 설정되지 않았습니다. 과정 정보에서 시작일을 입력해주세요. |
| `before_start` | 수업은 `{yyyy년 M월 d일 (EEE)}`에 시작됩니다. 시작 전에는 이번 주 학습이 표시되지 않습니다. |
| `in_progress` & !selectedPlan | 연결된 강의안이 없어서 이번 주 학습을 표시할 수 없습니다. |
| `in_progress` & !currentWeek | 이번 주는 예정된 강의가 없습니다. |

**LatestAnnouncement 빈 상태**: `아직 등록된 공지가 없습니다.`
**LatestAnnouncement 더보기 버튼**: `더보기` + `ArrowRight` → `/courses/${courseId}?tab=community&filter=announcement`.

**Editor 상단 바**: `[⋮⋮] [Badge:{block_type}] [Input:블록 제목] ─────────── [🗑]`.
**Editor placeholder**: `블록 제목`.

### 3.2 파일 트리 / 컴포넌트 트리 (Use Prompt Patterns for Layouts)

```
CourseHomeBlockEditorItem
├── (상단 바) GripVertical · Badge · Input(title) · Button(Trash2)
├── (좌) 폼 영역 — block_type 별 switch
│    ├── text          : Select(style) + Textarea(text)
│    ├── notice_banner : Select(tone) + Textarea(text)
│    ├── divider       : Input(label)
│    ├── image         : Button(업로드) + Button(URL) + Input(url,alt,caption)
│    ├── file          : Button(업로드) + Input(label,file_name) + Textarea(desc)
│    ├── video|lyrics  : Button(SongPickerDialog 열기) + Badge(선택된 곡)
│    ├── quiz          : Select(quizPresets)
│    ├── lesson_plan_overview|weekly_lessons : Select(lessonPlans) + Textarea(desc)
│    ├── students      : Textarea(desc)
│    ├── course_info   : Input(duration,students_target)+SemesterPicker+ClassTimePicker+Input[type=date](start_date)
│    ├── goals         : items[] Input + "목표 추가"
│    ├── curriculum    : weeks[] Input×3(week,title,desc) + "강의안에서 채우기" + "주차 추가"
│    ├── songs_list    : songs[] Input×3(title,artist,point) + "노래 추가"
│    ├── vocab_grid    : vocab[] Input×3(chinese,pinyin,korean) + "어휘 추가"
│    └── weekly_preview: Select(선택 강의안)
└── (우) CourseHomeBlockRenderer  ← 좌측 편집 결과가 실시간 반영

CourseHomeBlockRenderer(block)
└── switch(block.block_type)
     ├── course_header               → null (Hero는 CourseDetail이 별도 렌더)
     ├── weekly_preview              → <WeeklyPreviewBlock/>
     ├── text                        → 스타일(title/subtitle/body) 별 <div>
     ├── notice_banner               → tone-based tint 배너
     ├── divider                     → 좌우 hr + 중앙 라벨
     ├── image | video | file        → 각 카드
     ├── lyrics                      → 상위 6줄 (中/pinyin/韓)
     ├── quiz | students             → 카드 + 이동 버튼
     ├── lesson_plan_overview        → 강의안 그리드
     ├── weekly_lessons              → 상위 8주차
     ├── course_info / goals /
     │   curriculum / songs_list /
     │   vocab_grid / teacher_info   → 신규 홈 카드 (§ 2.2 스타일)
     ├── (legacy) latest_announcement | question_comments → null
     └── default                     → "지원되지 않는 블록입니다."
```

### 3.3 핵심 API 시그니처 (변경 금지)

```ts
// blockCatalog.tsx
export interface BlockDefinition {
  type: CourseHomeBlockType;
  label: string;
  description: string;
  icon: LucideIcon;
}
export const blockDefinitions: BlockDefinition[]; // §3.1 표 순서

export function createBlockInsert(params: {
  courseId: string;
  createdBy?: string | null;
  type: CourseHomeBlockType;
  sortOrder: number;
}): TablesInsert<"course_home_blocks">;

export async function buildBlockInsertPrefilled(params: {
  courseId: string;
  createdBy?: string | null;
  type: CourseHomeBlockType;
  sortOrder: number;
  lessonPlans: CourseLessonPlanSummary[];
  course: { semester?: string | null; class_time?: string | null; level?: string | null; student_count?: number | null };
}): Promise<TablesInsert<"course_home_blocks">>;
```

**Prefill 규칙(반드시 이 로직 유지)**

- `course_info` → `{ duration: "{weeks}주" | "", students_target: "{n}명" | "", semester, class_time }`.
- `curriculum` → primary plan의 `weeks[]`를 `{ week: "W{n}", title, desc: week_type==="regular" ? "정규 수업" : week_type }`로.
- `songs_list` → primary plan의 모든 `song_ids`를 합집합으로 뽑아 `songs` 테이블에서 `id,title,artist,teaching_point` 조회 후 `{ title, artist, point: teaching_point }`.
- `weekly_preview` / `lesson_plan_overview` / `weekly_lessons` → `content.selected_plan_id = primary.id`.
- 그 외 → 기본 content 그대로.

## ④ Context (배경)

### 4.1 프로젝트 맥락

「멜로디 클래스」의 반 상세 홈 탭은 **course_home_blocks** 행들을 `sort_order` 오름차순으로 렌더링하는 위젯 시스템이다. 본 문서(P5b)는 P5a가 관리하는 편집 세션(Undo/Redo·Save)의 조작 단위인 **블록** 자체를 정의한다. P5a는 배열 상태를, P5b는 배열의 원소 하나하나의 스키마·렌더·에디터를 담당한다.

### 4.2 Lovable Cloud 후경

- 인증된 교사만 편집. 학생은 렌더러만 사용. RLS는 P5a에서 정의된 `course_home_blocks` 정책을 재사용.
- 로딩/빈/에러/성공 4-상태:
  - **로딩**: 편집 카드 우측 미리보기가 데이터를 기다릴 때 `p.text-sm.text-muted-foreground` 안내.
  - **빈**: 각 리스트형 블록은 "아직 등록된 …이 없습니다." 계열의 순수 한국어 문구.
  - **에러**: 미리보기 default case → "지원되지 않는 블록입니다."
  - **성공**: 카드 헤더 + 콘텐츠 그리드.

### 4.3 데이터 계약

`course_home_blocks` 테이블은 P5a에서 이미 정의되었다고 가정한다. 본 문서는 **content JSON 스키마**만 추가로 확정한다.

```jsonc
// course_info
{ "duration": "15주", "students_target": "20명", "semester": "2026년 1학기", "class_time": "월 10:00~12:00", "start_date": "2026-03-02" }
// goals
{ "items": ["매주 노래 1곡을 완창한다", "핵심 어휘 60개를 습득한다"] }
// curriculum
{ "weeks": [{ "week": "W1", "title": "오리엔테이션", "desc": "정규 수업" }] }
// songs_list
{ "songs": [{ "title": "月亮代表我的心", "artist": "邓丽君", "point": "성조 대비" }] }
// vocab_grid
{ "vocab": [{ "chinese": "月亮", "pinyin": "yuèliang", "korean": "달" }] }
// weekly_preview / lesson_plan_overview / weekly_lessons
{ "selected_plan_id": "<uuid|null>", "description": "…" }
// notice_banner
{ "tone": "primary|accent|muted|destructive", "text": "…" }
// video / lyrics
{ "song_id": "<uuid>", "title": "…", "artist": "…", "video_id": "<yt id>", "youtube_url": "…", "lyrics": [ {chinese,pinyin,korean} ] }
// quiz
{ "quiz_id": "moon-vocab" | "apple-listening" | "friend-pattern" }
// teacher_info (P5a가 세팅에서 자동 삽입)
{ "full_name": "…", "organization": "…", "avatar_url": "…", "bio": "…" }
// course_header (Hero 텍스트 제어; 렌더러는 null, CourseDetail 이 소비)
{
  "title_override": "",        // 비면 course.name
  "subtitle_override": "",     // 비면 아래 항목 자동 조합
  "show_date": true,           // start_date 있으면 "YYYY년 M월 D일 시작", 없으면 semester
  "show_level": true,
  "show_class_time": true,
  "extra_line": ""             // 선택. 세 번째 줄로 표시
}
```

**Hero 소비 규칙 (CourseDetail):**
- 제목: `title_override?.trim() || course.name`
- 부제목: `subtitle_override?.trim() || [dateOrSemester, level, class_time].filter(Boolean).join(" · ")`
  - `dateOrSemester = show_date ? (start_date ? "YYYY년 M월 D일 시작" : semester) : ""`
  - `show_level === false` 면 level 제외, `show_class_time === false` 면 class_time 제외
- 세 번째 줄: `extra_line?.trim()` 이 있으면 `text-white/75` 로 렌더
- 표지 이미지는 이 블록과 무관 — `courses.cover_image_url` + `CourseHeroEditor` (P5a) 담당

## ⑤ Acceptance & Output (검증·산출)

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6 — 정량 임계값)

1. `blockDefinitions.length === 7` 이고 그 `type` 배열은 §3.1 표의 순서(course_header → course_info → goals → weekly_preview → curriculum → songs_list → vocab_grid)와 정확히 일치한다.
2. `CourseHomeBlockRenderer` 의 `switch (block.block_type)` 는 §3.1 유니온의 모든 활성 case + legacy(`latest_announcement`, `question_comments` → null) + `default` 를 모두 다룬다.
3. `createDefaultCourseHomeBlocks({...})` 는 **Promise** 를 반환하며, resolve 배열의 `length === 7` · `block_type` 순서는 `["course_header","course_info","goals","weekly_preview","curriculum","songs_list","vocab_grid"]`.
4. `buildBlockInsertPrefilled({type: "course_info", ...})` 호출 시 `content.semester === params.course.semester`, `content.class_time === params.course.class_time` 이다.
5. `WeeklyPreviewBlock` 은 `getCourseSchedule` 반환의 4가지 상태(`no_start_date`, `before_start`, `in_progress`+플랜없음, `in_progress`+주차없음) 각각에서 §3.1 표의 문자열을 렌더한다(정확 일치, 문자 단위 diff 0).
6. `CourseHomeBlockEditorItem` 은 `useSortable({ id: block.id })` 를 호출하고, `isDragging === true` 인 프레임에서 컨테이너 클래스가 `"opacity-60"` 을 포함한다.
7. `grep -R "bg-\[#" src/components/courses/home-blocks src/components/courses/CourseHomeBlockEditorItem.tsx` 결과 **0건**.
8. `grep -RE "(text-white|text-black|bg-black|bg-white)\b" src/components/courses/home-blocks src/components/courses/CourseHomeBlockEditorItem.tsx` 결과 **0건**(`bg-background`/`text-foreground` 등 semantic token만 사용).
9. `pnpm tsc --noEmit` 오류 **0건**.
10. `vitest run` 로컬 스냅샷(있는 경우) 통과율 100 %.

### 5.2 Output Format

LLM 이 반환하는 파일 순서는 다음과 같으며, 각 파일은 **완전한** 소스만 포함한다. 설명·마크다운 헤더·주석 요약·사과 문구·`// ...` ellipsis 를 금지한다.

1. `src/components/courses/home-blocks/types.ts`
2. `src/components/courses/home-blocks/quizPresets.ts`
3. `src/components/courses/home-blocks/blockCatalog.tsx`
4. `src/components/courses/home-blocks/defaultBlocks.ts`
5. `src/components/courses/home-blocks/FocusedHomeBlocks.tsx`
6. `src/components/courses/home-blocks/CourseHomeBlockRenderer.tsx`
7. `src/components/courses/CourseHomeBlockEditorItem.tsx`
