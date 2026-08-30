# P5d — 반 상세 페이지: 알림 & 우리 반 탭 (공용 골격)

> 공용(공통) 영역 마지막 문서. 반 상세 페이지의 남은 두 탭 **알림(NotificationsTab)** 과 **우리 반(CommunityTab)** 을 하나의 재현 프롬프트로 묶었다. 두 탭 모두 교사/학생 시점을 공유하고, `course_id` 기반 실시간 Postgres Changes 로 동기화되며, 반 초대 링크(`courses.share_token`) · `course_student_profiles` · `notification_replies` 스키마를 공유하기 때문에 함께 재현하는 것이 자연스럽다.

---

## 1. Identity

당신은 시니어 React + TypeScript + Supabase 프론트엔드 엔지니어이자 UX 라이터다. 아래 사양을 지켜 **한국어 UI** 로 동작하는 두 탭을 구현하라. 스택은 이미 다른 P5\*, P6 문서로 초기화되어 있으며, 다음 전역 자산을 그대로 사용한다.

- `@/integrations/supabase/client` — Supabase JS 클라이언트 (`localStorage` 세션 저장, `persistSession`, `autoRefreshToken`)
- `@/hooks/useAuth` — `{ user, isTeacher, isAdmin }`
- `@/hooks/use-toast` (shadcn `toast`)
- shadcn/ui: `Button`, `Input`, `Card/CardContent`, `Dialog…`, `Select…`, `Tabs…` 등
- lucide-react 아이콘
- `@/lib/utils` `cn`
- `@/components/courses/CourseModuleToolbar` 의 `COURSE_TOOLBAR_EVENT_NAME` + `CourseToolbarAddEventDetail` (P5a 참조)
- `@/components/courses/StudentOnboardingDialog` (재사용)

브랜드 색: navy `#2e3d6b`, 서브 배경 `#EEF2FF` / `#EEF1F8` / `#F5F7FB`, 카드 배경 `#fff` + `0.5px solid rgba(0,0,0,0.08)`, 라운드 14px. 이 두 탭은 tailwind semantic token 위에 이 “종이 카드” 스타일을 인라인 `style`로 얹는 형태를 유지한다.

---

## 2. Instructions

### 2.1 반 상세 페이지 안에서의 위치

P5a에서 정의한 `CourseDetail` 페이지에는 5개 탭(`home` / `calendar` / `materials` / `notifications` / `community`)이 있다. 본 문서는 그 중 **`notifications`** 와 **`community`** 두 탭을 다룬다. 두 탭은 편집 모드 유무와 상관없이 항상 마운트되고, 편집 모드는 P5a Home 탭 전용이므로 UI 잠금 로직이 필요 없다.

### 2.2 공통 DB 스키마

이 두 탭이 사용하는 테이블은 아래와 같다. 이미 P6 인프라 문서에서 정의되었다면 재정의하지 말고, 컬럼 시그니처가 다르면 마이그레이션 파일을 새로 추가하라.

| 테이블 | 주요 컬럼 | 용도 |
| --- | --- | --- |
| `course_notices` | `id`, `course_id`, `author_id`, `type` (`공지`/`과제`/`일정`), `content`, `created_at` | 교사가 학생에게 보내는 공지 |
| `course_student_posts` | `id`, `course_id`, `student_id`, `type` (`결석`/`지각`/`질문`/`기타`), `content`, `visibility` (`public`/`private`), `status` (`pending`/`approved`/`rejected`/`answered`), `created_at` | 학생 → 교사 게시물 |
| `notification_replies` | `id`, `parent_type` (`notice`/`student_post`), `parent_id`, `course_id`, `author_id`, `content`, `created_at` | 공지·게시물 각각에 달리는 스레드형 답글 |
| `course_student_profiles` | `id`, `course_id`, `member_user_id`, `full_name`, `student_number`, `department`, `avatar_url`, `emoji`, `language_level`, `study_years`, `gender`, `sort_order` | 반 학생 명단(온보딩으로 자동 채워짐) |
| `courses.share_token` | uuid | 학생 초대 링크(`/shared/course/:token`) 생성 |

RLS(요약, 실제 정책은 P6 문서 참조):

- **읽기**: `course_owner_id = auth.uid()` 이거나 `is_course_member(course_id)` 로 통과.
- **쓰기**: notices는 소유자 전용, student_posts는 자신(`student_id = auth.uid()`)+ member, replies는 member 전체(교사는 어떤 답글이든 삭제 가능), student_profiles 수정은 본인 프로필 or 소유자.
- Data API grant 는 P6 표준(`authenticated` 전체, `service_role` all)을 따른다.

### 2.3 알림 탭 — `NotificationsTab`

경로: `src/components/courses/NotificationsTab.tsx`. Props:
```ts
{ course: { id: string; owner_id?: string | null };
  editMode: boolean; // 사용하지 않지만 P5a의 통일된 시그니처 유지
  isStudentView: boolean;
  user: User | null; }
```

레이아웃(2열, `lg` 이하는 1열):
```text
┌───────────────────────────────┬──────────────────────────┐
│ TeacherNoticeList             │ 학생 시점: StudentComposer │
│ (교사 공지, 실시간 갱신)         │ 교사 시점: TeacherComposer │
│ StudentPostList               │                          │
│ (학생 게시물 + 승인/거절)        │                          │
└───────────────────────────────┴──────────────────────────┘
```

**시점 판정**:
```ts
const isOwner = !!(user && course.owner_id && course.owner_id === user.id);
const effectiveStudentView = isStudentView || !isTeacher || !isOwner;
```
학생 시점(`effectiveStudentView === true`)에서는:
- 학생 게시물 목록에서 `visibility === 'private'` 인 남의 게시물은 감춘다 (본인 게시물은 항상 노출).
- 우측 컴포저는 `StudentComposer`.
- 리스트에서 승인/거절/삭제 버튼 미노출.

**데이터 fetch**: `useEffect(() => …, [course.id])` 안에서 3개 쿼리를 `Promise.all` 로 병렬 로드하고, `supabase.channel(\`notif-${course.id}\`)` 로 `course_notices`, `course_student_posts` 의 postgres_changes 를 구독해 이벤트 발생 시 재-fetch. 컴포넌트 언마운트 시 `removeChannel`.

컨테이너 배경: `#EEF1F8`, 상단 여백 `mt-4`, 카드는 라운드 14px + hairline border.

#### 2.3.1 TeacherNoticeList

- 헤더: `Megaphone` 아이콘 + “교사 공지”.
- 필터 pill: `전체 / 공지 / 과제 / 일정` (활성 pill 은 navy 채움, 비활성은 `#EEF2FF`).
- 각 공지 카드:
  - 상단: type 배지(라운드 pill) + `M월 d일 HH:MM` 로컬 포맷.
  - 우측 액션: `답글` 토글(펼치기/접기) + (`canManage` 인 경우) `Pencil`(편집), `Trash2`(삭제).
  - 편집 모드: `type` 은 `공지/과제/일정` `<select>`, `content` 는 `<textarea>` (최소 80px). 저장 시 `course_notices.update({content, type}).eq('id', id)`. 저장 실패 → destructive toast.
  - `RepliesThread` 는 카드가 확장된 상태에서만 렌더(모두 접히면 unmount하여 채널 리소스 절약).
- 스크롤 컨테이너: `max-h-[420px] overflow-y-auto pr-1`.

#### 2.3.2 StudentPostList

- 헤더: `Users` 아이콘 + “학생 게시물”.
- 교사 시점(`canManage`)에서만 `전체 / 전체공개 / 교사전용` pill 필터 노출.
- 각 게시물 카드:
  - 좌: 원형 아바타(`emoji || full_name[0] || '👤'`), 학생 이름, 시간.
  - 우: `type` 배지, `Globe`(public) 또는 `Lock`(private) 아이콘.
  - 하단: 상태 배지(`pending`/`approved`/`rejected`/`answered`) — 색상 매핑:
    - pending `#F1F5F9`/`#475569`
    - approved `#DCFCE7`/`#166534`
    - rejected `#FEE2E2`/`#991B1B`
    - answered `#EEF2FF`/`#2e3d6b`
  - 하단 액션(교사만):
    - type ∈ {`결석`,`지각`} && pending → `승인` / `거절` 버튼 → `status` 업데이트.
    - type === `질문` && pending → `답변 완료` 버튼 → `status = 'answered'`.
  - 삭제 버튼: 본인(`isOwn`) 또는 교사(`canManage`) 노출.
  - `답글` 토글로 `RepliesThread` 마운트.
- 학생 시점에서는 필터 없이 `posts.filter(p => p.visibility === 'public' || p.student_id === user.id)` 결과만 노출.

#### 2.3.3 TeacherComposer

경로: `src/components/courses/notifications/TeacherComposer.tsx`.

- 상단: “새 공지 작성”.
- Type pill: `공지 / 과제 / 일정`.
- **템플릿 그리드**(3열): 6개 프리셋 문구 카드 클릭 → 본문 자동 채우기.
  ```ts
  const TEMPLATES = [
    { label: "교실 변경", text: "다음 수업의 교실이 변경되었습니다. 새 교실: " },
    { label: "수업 취소", text: "사정상 다음 수업이 취소되었습니다. 보강 일정은 추후 공지하겠습니다." },
    { label: "퀴즈 공지", text: "다음 시간에 짧은 퀴즈가 있습니다. 범위: " },
    { label: "과제 마감", text: "이번 주 과제 제출 마감일은 ___입니다. 잊지 말고 제출해 주세요." },
    { label: "자료 업로드", text: "수업 자료를 업로드했습니다. 강의 자료 탭에서 확인해 주세요." },
    { label: "일정 변경", text: "예정된 일정이 변경되었습니다. 새 일정: " },
  ];
  ```
- `<textarea rows={6}>` 라운드 16px, hairline border.
- 액션 pill:
  1. **AI로 다듬기** — `supabase.functions.invoke('polish-notice', { body: { role: 'teacher', type, content }})`.
     - 응답이 `{ polished }` 이면 하단에 `AIPolishPanel` 렌더.
     - 응답이 `{ error }` 또는 함수 error → destructive toast, description은 서버 메시지 그대로.
     - 429/402/네트워크 예외도 catch 해 사용자 친화 메시지.
  2. **미리보기** — 하단에 미리보기 카드 토글(수신: 전체 학생, 유형 표시, `발송 / 수정` 버튼).
  3. **발송** — `course_notices.insert({course_id, author_id: user.id, type, content: content.trim()})`. 로그인 필요.
- 발송 성공 → “발송 완료” 상태 화면(“새 공지 작성” 버튼으로 폼 리셋).
- **툴바 프리필**: 마운트 시 `window.addEventListener(COURSE_TOOLBAR_EVENT_NAME, handler)`. `detail.tab === 'notifications'` 이고 `detail.type ∈ TYPES` 이면 해당 type으로 폼 전환. 언마운트 시 remove.

#### 2.3.4 StudentComposer

경로: `src/components/courses/notifications/StudentComposer.tsx`.

- 상단: “선생님께 보내기”.
- Type pill: `결석 신청 / 지각 신청 / 질문 신청 / 기타 신청`.
- 공개 범위 pill: `교사에게만(private)` / `전체 공개(public)`.
- `type ∈ {결석, 지각}` 로 바뀌면 `visibility = 'private'` 강제.
- Placeholder 문구는 type 별로 다름(위 소스 `PLACEHOLDERS`).
- AI로 다듬기 = `role: 'student'` 로 동일 edge function 호출.
- 발송 → `course_student_posts.insert({course_id, student_id: user.id, type, content, visibility, status: 'pending'})`.
- 성공 시 “전송 완료” 상태 화면.

#### 2.3.5 AIPolishPanel

경로: `src/components/courses/notifications/AIPolishPanel.tsx`.
- Props: `{ polished, onReplace, onCancel }`.
- 배경 `#EEF2FF`, 헤더 `Sparkles` 아이콘 + “AI 다듬기 결과”, 본문 whitespace-pre-wrap.
- 액션: `이 내용으로 교체`(navy 채움) / `취소`(흰 배경 + border).

#### 2.3.6 RepliesThread — 공용 답글 스레드

경로: `src/components/courses/notifications/RepliesThread.tsx`. Props:
```ts
{ parentType: "notice" | "student_post";
  parentId: string;
  courseId: string;
  user: User | null;
  isTeacher: boolean;
  compact?: boolean; }
```

- `notification_replies` 에서 `parent_type + parent_id` 로 조회, `created_at asc`.
- 작성자 이름 매핑: `profiles.full_name` 을 기본으로 하고, `course_student_profiles`(같은 `member_user_id`) 가 있으면 학생 이름/이모지로 덮어쓴다.
- 실시간: `supabase.channel(\`replies-${parentType}-${parentId}\`)` 로 `parent_id=eq.${parentId}` 필터 구독.
- 입력: `<textarea rows={2}>` + 원형 send 버튼(`Send` 아이콘). 로그인 필요.
- 삭제: 본인 답글이거나 `isTeacher` 인 경우만 `Trash2` 노출 → `delete().eq('id', id)`.
- 스크롤 max-h: `compact ? 40 : 56` (Tailwind `max-h-40/56`).
- 이 컴포넌트는 P3(대시보드 학생용 알림 카드)에서도 재사용되므로 시그니처를 절대 변경하지 말 것.

### 2.4 폴리시 edge function 계약

`supabase/functions/polish-notice/index.ts` 는 P6에서 정의한다. 이 문서에서는 호출 규약만 보장한다.

- 입력: `{ role: 'teacher' | 'student', type: string, content: string }`
- 정상 응답: `{ polished: string }` (200)
- 오류 응답: `{ error: string }` (2xx 로도 리턴 가능 — 프론트는 두 방식 모두 처리)
- 모델: `google/gemini-2.5-flash` 또는 Lovable AI 기본. 429 → “요청이 너무 잦습니다.” 402 → “크레딧 부족”. 프론트는 서버가 준 문자열을 그대로 노출.

---

### 2.5 우리 반 탭 — `CommunityTab`

경로: `src/components/courses/CommunityTab.tsx`. Props:
```ts
{ course: { id: string; name?: string };
  editMode: boolean;
  user: User | null; }
```

목적: **초대 링크로 들어온 학생 카드 그리드** + 교사의 수동 입력/편집 + 학생 본인 프로필 편집.

#### 2.5.1 그리드 스펙 (참고 HTML → 여기서 CSS Grid 로 구현)

```css
display: grid;
grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
gap: 14px;
```

카드(높이 fluid, 세로 flex):
- 원형 아바타 52×52, `mb-[9px]`.
- 이름 13px medium, 학과·학번 10px muted (두 줄, 값 없을 때 “학과 미입력” 등 placeholder).
- 얇은 hairline divider 0.5px.
- 하단 배지 그룹(min-h 20px, 3px gap):
  - 언어 레벨: `#E6F1FB` 배경 + `#0C447C` 글자
  - `studyYearsLabel(study_years)`: muted 배지
  - `genderLabel(gender)`: muted 배지 (`남/여/기타`)

#### 2.5.2 아바타 우선순위

```ts
if (student.avatar_url) → <img/> (object-cover)
else if (student.emoji) → 원형 muted 배경에 26px 이모지
else → 원형 pastel(#E6F1FB / #0C447C) + 이름 앞 2글자
```

#### 2.5.3 “Ghost” 대기 카드

- `MIN_PLACEHOLDER_COUNT = 6`. 실제 학생 + placeholder 합계가 최소 6이 되도록 채운다.
- `editMode`가 true 이면 placeholder 최소 2 보장.
- 유령 카드: 흐린 배경 + `UserPlus` 아이콘 + “학생 대기 중 / 초대 후 자동 추가”.
- 편집 모드에서만 `직접 입력` 버튼 노출 → 인라인 에디터 카드로 전환.

#### 2.5.4 학생 “나” 상태

- `myProfile = students.find(s => s.member_user_id === user.id)`.
- 내 카드는 좌상단에 navy pill `나`, 우상단 hover 시 `Pencil` 버튼 → `StudentOnboardingDialog` (existing 프로필 pre-fill 후 오픈).
- 교사(`editMode` true) 는 다른 학생 카드에 대해서도 `Pencil` 로 인라인 에디터 오픈.

#### 2.5.5 인라인 에디터 카드 (교사 전용 수동 추가/수정)

- 그리드 안에서 해당 슬롯을 `gridColumn: span 2` 로 확장해 폼을 인라인 렌더.
- 필드: `full_name*`, `student_number`, `department`, `avatar_url` (URL 직접 입력).
- 저장(`Insert` 또는 `Update`)은 `supabase.from('course_student_profiles')` 로 수행. 저장 후 로컬 state 를 `sort_order` 기준 재정렬하고 toast.
- 이 에디터는 온보딩 다이얼로그와 다르게 이미지 업로드/이모지 선택은 하지 않는다(교사가 명단을 임시로 채우는 용도).

#### 2.5.6 초대 링크

- 상단 헤더에 `학생 초대` 버튼(UserPlus).
- `courses.share_token` 조회 후 `${origin}/shared/course/${token}` 를 `navigator.clipboard.writeText()`.
- Toast description: “학생이 링크로 들어와 Google 로그인 + 정보 입력을 완료하면 자동으로 명단에 추가됩니다.”

#### 2.5.7 fetch & 실시간

- `useEffect(() => { fetchStudentCards(); }, [course.id])`.
- `fetchStudentCards` 는 `course_student_profiles` (`sort_order asc`) 와 `courses.share_token` 을 `Promise.all` 로 함께 조회.
- **실시간 갱신은 필수가 아니지만, 온보딩 완료 직후 자동 반영이 필요하면 `postgres_changes`(`course_student_profiles`, `course_id=eq.${course.id}`) 를 채널에 추가**해 재-fetch 하라. 최소 요구는 “내 프로필 저장 → 콜백에서 `fetchStudentCards()` 재호출”.

---

## 3. Examples

### 3.1 Notification 탭 초기 상태

```
[교사 공지]                            [새 공지 작성]
공지가 없습니다.                        공지 | 과제 | 일정
                                       [교실 변경][수업 취소][퀴즈 공지]
[학생 게시물]                          [과제 마감][자료 업로드][일정 변경]
게시물이 없습니다.                       ┌──────────────────────┐
                                       │ 공지 내용을 입력해 주세요…│
                                       └──────────────────────┘
                                       [AI로 다듬기][미리보기][발송]
```

### 3.2 학생 게시물 — 결석 신청 승인 흐름

1. 학생: `결석 신청` + `교사에게만` + 내용 → 발송 → `status='pending', visibility='private'`.
2. 교사(우측 목록): pending pill + `승인`/`거절` 버튼.
3. 교사 `승인` → `status='approved'` → 학생 시점에서도 초록 배지로 갱신(실시간).

### 3.3 초대 후 카드 자동 등장

1. 교사 `학생 초대` → 링크 복사.
2. 학생: `/shared/course/:token` → 온보딩 다이얼로그(P5a 로직) → `course_members` + `course_student_profiles` insert.
3. 우리 반 탭: placeholder → 실제 카드 (내 카드에는 `나` pill).

---

## 4. Context

- 두 탭 모두 “교사와 학생이 하나의 UI 를 공유하는데 시점에 따라 액션이 달라진다”. `effectiveStudentView` 판정을 반드시 auth-state로 하고 UI 만으로 우회할 수 없게 하라 (RLS가 최종 방어선).
- 시간 포맷은 `${d.getMonth()+1}월 ${d.getDate()}일 HH:MM` (date-fns 없이 로컬 브라우저 시간). ko 로케일 필요 없음.
- 공지·답글·게시물 모두 **낙관적 업데이트 없이 realtime 재-fetch** 로 처리 — race condition을 피하고 삭제/편집 시 서버 상태를 소스 오브 트루스로 유지.
- 답글 스레드는 별도 컴포넌트로 분리해 학생 대시보드의 최신 공지 카드에서도 재활용된다(P3 대시보드 학생 모듈 문서에서 import).
- 우리 반 탭은 편집 모드 여부에 관계없이 항상 접근 가능. 편집 모드는 P5a Home 탭 전용이라 여기선 `editMode` 를 UI 노출 조건(placeholder 개수 등)에만 사용한다.

---

## 5. Acceptance

이 문서를 그대로 실행하고 나면 다음이 모두 만족되어야 한다.

- [ ] `NotificationsTab` 은 2열 레이아웃, 교사/학생 시점 자동 판정, 실시간 반영.
- [ ] 교사가 공지 작성 시 3개 카테고리(공지/과제/일정) 중 하나를 고를 수 있고, 6개 템플릿·AI 다듬기·미리보기·발송이 모두 동작한다.
- [ ] 학생 게시물의 `결석/지각` 는 자동 `private`, `승인/거절` 로 상태 전이; `질문` 은 `답변 완료` 로 마감.
- [ ] 학생 시점에서는 `private` 인 다른 학생 게시물이 절대 보이지 않는다(UI + RLS 둘 다).
- [ ] `RepliesThread` 는 공지·게시물 모두에서 확장/접기, 실시간 갱신, 본인 또는 교사만 삭제 가능.
- [ ] `polish-notice` edge function 이 429/402/서버 오류를 반환할 때 한국어 toast로 자연스럽게 노출된다.
- [ ] `CommunityTab` 그리드는 `repeat(auto-fill, minmax(160px, 1fr))`, gap 14px, 카드 라운드 12px + hairline border.
- [ ] 아바타 우선순위(`avatar_url > emoji > initial`) 대로 렌더링되고, 이모지가 없어도 파스텔 이니셜 카드가 뜬다.
- [ ] `MIN_PLACEHOLDER_COUNT = 6` 규칙에 따라 학생 수가 6명 미만일 때 유령 카드가 채워진다.
- [ ] `학생 초대` 버튼이 `courses.share_token` 기반 URL 을 복사하고, 학생 온보딩 완료 시 자동으로 카드가 등장한다.
- [ ] 내 카드에는 `나` pill 이 뜨고, `Pencil` 버튼이 `StudentOnboardingDialog` 를 프리필로 여는 반면, 교사는 다른 학생 카드에 대해 인라인 에디터를 연다.
- [ ] 두 탭 모두 컴포넌트 언마운트 시 `supabase.removeChannel` 로 realtime 리소스를 정리한다.
- [ ] 텍스트/문구는 모두 한국어이며, 브랜드 색(navy `#2e3d6b`, 서브 `#EEF2FF/#EEF1F8/#F5F7FB`)이 인라인 style 로 유지된다.
