# S6 · 학생 설정 & 학생 휴지통

## 1. Identity

당신은 **멜로디 클래스(Melody Class)** 플랫폼의 학생 사이드 프론트엔드 엔지니어입니다. 이 프롬프트는 학생 계정으로 로그인했을 때 접근 가능한 두 개의 사이드바 항목 **「설정」(`/settings`)** 과 **「휴지통」(`/trash`)** 의 학생 전용 브랜치를 재현하기 위한 사양서입니다. 교사·관리자 전용 로직(수업 환경 탭, 강의안·학생 프로필 휴지통 등)은 이 문서의 범위 밖입니다.

기술 스택: React 18 + Vite 5 + Tailwind CSS v3 + shadcn/ui + Supabase(JS v2).

## 2. Instructions

### 2.1 라우팅 & 사이드바 진입

- `AppSidebar.tsx` 의 `studentItems` 배열은 다음 순서를 유지합니다:
  1. 홈 `/` (`Home`)
  2. 내 공간 `/student-home` (`Sparkles`)
  3. 노래 아카이브 `/songs` (`Music`)
  4. **휴지통 `/trash`** (`Trash2`)
  5. 사용 가이드 `/guide` (`BookOpenCheck`)
  6. **설정 `/settings`** (`Settings`)
- 사이드바 항목 분기 기준은 `isStaff = isTeacher || isAdmin` 이며, 학생은 `studentItems` 를 봅니다.
- 두 경로 모두 `RequireAuth → AppLayout` 로 래핑됩니다.
- 두 페이지 모두 최상단에서 `useGuideTour()` 를 호출하여 사용 가이드의 현장 체험(`student-settings`, `student-trash` 시나리오)을 받습니다.

### 2.2 학생 설정 (`SettingsPage.tsx`)

- 페이지 폭 `max-w-3xl`, 제목 「설정」(#2e3d6b, `text-[22px] font-semibold`), 설명 「플랫폼 설정을 관리하세요.」
- `useAuth()` 로 `isTeacher` 를 얻어 탭을 분기합니다:
  - 교사: `[프로필, 수업 환경, 계정 & 보안]`
  - **학생: `[프로필, 계정 & 보안]`**
- 알림(`notify`) 탭은 **양쪽 모두에서 완전히 제거**되었습니다. `NotificationsTab` 컴포넌트 파일도 존재하지 않습니다.
- 탭 스타일: 밑줄형 (`border-b-2`, 활성 시 `#2e3d6b`), 각 트리거에 `data-tour="settings-tab-{profile|class|security}"`, `TabsList` 에 `data-tour="settings-tabs"`.
- `loading` 중에는 스켈레톤(제목 바 + `h-64` 카드), 미로그인 상태에서는 「설정을 관리하려면 로그인이 필요합니다.」 안내와 `/auth` 이동 버튼을 렌더링합니다.
- 프로필 탭 본문은 역할에 따라 `isTeacher ? <ProfileTab/> : <StudentProfileTab/>` 로 분기됩니다.

#### 프로필 탭 — `StudentProfileTab`

- 데이터: `profiles.{full_name, avatar_url, department, student_id, hsk_level, topik_level, learning_duration, gender}`.
- 토스트는 `sonner` 의 `toast.success/error` 를 사용합니다.
- 아바타
  - `avatars` 스토리지 버킷 업로드 (경로 `${user.id}/avatar-${Date.now()}.${ext}`, `upsert: true`, `cacheControl: "3600"`).
  - 2MB 초과 시 「이미지는 2MB 이하여야 합니다」, MIME 이 `image/jpeg|png|jpg|webp` 가 아니면 「JPG, PNG 파일만 업로드 가능합니다」.
  - 업로드 성공 시 `getPublicUrl` 결과로 `profiles.avatar_url` 갱신 → `refreshProfile()` → 「프로필 사진이 변경되었습니다」.
  - 버튼 라벨은 「사진 변경」/업로드 중 「업로드 중...」, 보조 문구 「JPG, PNG 최대 2MB」.
- 편집 필드
  - 이름 (필수, 최대 50자) — 비어 있으면 「이름을 입력하세요」
  - 학번 (선택, 최대 30자, placeholder `2024xxxxxx`)
  - 학과 (선택, 최대 100자, placeholder `중어중문학과`)
  - HSK 수준 `[없음, HSK 1-3, HSK 4-6, HSK 7-9]`
  - TOPIK 수준 `[없음, TOPIK 1-2, TOPIK 3-4, TOPIK 5-6]`
  - 학습 기간 `[6개월 미만, 1년, 2년, 3년, 4년, 5년 이상]` (내부 값 `0.5/1/2/3/4/5`)
  - 성별 라디오 `[남 / 여 / 기타]` (내부 값 `male/female/other`, 기본 `other`)
  - 이메일 (읽기 전용)
- 값 정규화 헬퍼: `hskToBucket`, `topikToBucket`(DB 의 자유 문자열을 구간 라벨로 환원), `durationToYears`/`yearsToDuration`(연수 ↔ 한국어 라벨).
- 저장 시 `hsk_level="없음"` / `topik_level="없음"` 은 `null` 로 저장, 학번·학과의 빈 문자열도 `null` 로 저장. 성공 시 「저장되었습니다」 + `refreshProfile()`.

#### 계정 & 보안 탭 — `SecurityTab`

- **비밀번호 변경 카드**
  - 입력 3개: 현재 비밀번호 / 새 비밀번호 / 새 비밀번호 확인.
  - 검증: 새 비밀번호 6자 이상(「새 비밀번호는 6자 이상이어야 합니다」), 두 입력 일치(「새 비밀번호가 일치하지 않습니다」).
  - 절차: `supabase.auth.signInWithPassword({ email, password: cur })` 로 현재 비밀번호를 재확인(실패 시 「현재 비밀번호가 올바르지 않습니다」) → `supabase.auth.updateUser({ password })` → 「비밀번호가 변경되었습니다」 후 입력 초기화.
  - `isOAuthOnly`(= `user.identities` 에 `provider === "email"` 이 없음)인 경우 입력 폼 대신 「소셜 로그인 계정은 비밀번호 변경이 지원되지 않습니다.」 만 노출.
- **계정 탈퇴(danger zone) 카드** — `bg-red-50 border-red-200`, 안내 「계정을 탈퇴하면 모든 데이터(강의안, 학생 정보, 노래 아카이브)가 영구적으로 삭제됩니다. 이 작업은 되돌릴 수 없습니다.」
  - `AlertDialog` 확인(제목 「정말 탈퇴하시겠습니까?」, 확인 버튼 「탈퇴 확인」) 후 `supabase.functions.invoke("delete-account")` → `signOut()` → `/` 이동.

### 2.3 학생 휴지통 (`Trash.tsx` — 학생 브랜치)

- 레이아웃: 페이지 여백을 상쇄한 전면 배경 `-m-3 sm:-m-4 md:-m-6 p-3 sm:p-4 md:p-6`, 배경색 `#f4f3f0`, 흰색 카드 `rounded-[24px]` + `1px solid rgba(0,0,0,0.07)` + 미세 그림자 (Workspace 톤과 동일).
- 헤더 카드: 인디고 아이콘 칩(`#eeeffe` 배경 / `#c7caff` 테두리 / `Trash2 #4f52c8`) + 「휴지통」, 안내 「삭제한 항목은 **7일간** 보관 후 자동으로 영구 삭제됩니다. 본인이 삭제한 항목만 표시됩니다.」(`7일간` 만 `#4f52c8` 강조).
- 우상단 액션 2종: `전체 복원`(RotateCcw, `wsBtn.outline`) · `휴지통 비우기`(Trash, 붉은 톤 `#fee2e2/#991b1b/#fecaca`) — 현재 탭 항목이 0이면 `disabled`.
- **탭 구성 (역할별 분기)**
  - 교사/관리자(`isStaff`): `songs, lesson_plans, courses, course_student_profiles` (T8 참조)
  - **학생: `songs, left_courses` — 정확히 2개 탭**
- 탭 라벨은 `TRASH_LABEL` 매핑: `songs → 노래`, `left_courses → 가입한 반`.
- `TabsList` 는 `#faf9f7` 배경의 pill 그룹, 활성 트리거는 흰색 + `#4f52c8` 글자 + `#c7caff` 테두리. 각 트리거 우측에 카운트 뱃지.
- 카드 그리드: `sm:grid-cols-2 lg:grid-cols-3`, 카드는 `rounded-[18px]` 흰색, hover 시 `-translate-y-0.5` + 인디고 그림자. 카드 구성 = `primary`(제목, truncate) · `days일 남음` 뱃지 · `secondary`(부제) · `삭제일: {toLocaleString("ko-KR")}`.
- 남은 일수 뱃지 색: `≤1` 붉은색(`#fee2e2/#991b1b`), `≤3` 호박색(`#fef3c7/#92400e`), 그 외 인디고(`#eeeffe/#3739a8`).
- 상태 렌더링 4종: 로딩 시 `h-40` 펄스 블록, 빈 상태는 원형 아이콘 + 「휴지통이 비어 있어요」 + 「{라벨}에서 삭제한 항목은 7일간 보관 후 자동으로 영구 삭제돼요.」 + 「7일 보관 정책」 칩, 데이터가 있으면 카드 그리드, 모든 탭 합계가 0이면 하단에 「모든 휴지통이 비어 있어요.」.
- 카드 클릭 → 상세 `Dialog`(제목 = primary, 설명 = secondary + 삭제일 + 「N일 후 영구 삭제」, 하단 `복원` / `영구 삭제` 버튼).
- 리스트 영역 우클릭 → `ContextMenu`: `전체 복원` / `전체 영구 삭제`(destructive) — 해당 탭이 비어 있으면 비활성.
- 확인 `AlertDialog`
  - Bulk: 「{TRASH_LABEL[tab]} 전체 복원」 / 「{TRASH_LABEL[tab]} 전체 영구 삭제」 + 「… 모든 항목(N개)을 복원/영구 삭제합니다.」, 영구 삭제 시 `AlertTriangle` 아이콘과 「이 작업은 되돌릴 수 없습니다.」 경고.
  - Single: 「‘{primary}’ 항목을 복원/영구 삭제합니다.」

### 2.4 RPC 계약 (SECURITY DEFINER)

학생 UI 는 두 종류의 경로를 사용합니다.

| 구분 | RPC | 인자 | 반환 |
|---|---|---|---|
| 실제 테이블(`songs`) | `soft_delete_item(_table, _id)` | text, uuid | void |
| | `restore_trash_item(_table, _id)` | text, uuid | void |
| | `purge_trash_item(_table, _id)` | text, uuid | void |
| | `restore_all_trash(_table)` | text | integer |
| | `purge_all_trash(_table)` | text | integer |
| 가상 테이블(`left_courses`) | `get_my_left_courses()` | – | course_id, name, level, semester, left_at |
| | `restore_course_membership(_course_id)` | uuid | void |
| | `purge_course_membership(_course_id)` | uuid | void |

- `src/lib/trash.ts` 는 `REAL_TABLES` 집합으로 실제 테이블만 `_table` 인자에 넘기고, `left_courses` 는 위 3개 전용 RPC 로 라우팅합니다. 지원하지 않는 조합은 `unsupported trash table` 로 즉시 예외.
- `left_courses` 는 전용 일괄 RPC 가 없으므로 `restoreAll`/`purgeAll` 이 목록을 조회한 뒤 클라이언트에서 순차 호출하고 처리 건수를 반환합니다.
- 권한 규칙 (RPC 내부): `owner_id = auth.uid()` 인 행만 대상. 관리자(`is_admin()`)는 `owner_id IS NULL` 공개 행까지 대상 — 학생에게는 해당 없음.

### 2.5 데이터 수집

- 클라이언트는 `TABLES.map` 으로 병렬 수집:
  - `songs`: `select("id, deleted_at, title, artist, title_bilingual, language")`, 필터 `.eq("owner_id", user.id).not("deleted_at","is",null).order("deleted_at",{ascending:false})`
    - `primary = title_bilingual || title || artist || "(제목 없음)"`, `secondary = artist`
  - `left_courses`: `fetchLeftCourses()`(RPC) 결과를 `deleted_at ← left_at` 으로 매핑해 동일한 보관 타이머 로직을 재사용
    - `primary = name || "(이름 없음)"`, `secondary = "{level} · {semester}"`
- 결과는 `Partial<Record<TrashTable, TrashRow[]>>` 로 저장(부분 셰이프 안전). 조회 실패 시 해당 탭은 빈 배열로 처리하여 다른 탭 렌더를 막지 않습니다.
- `TrashRow = { id, deleted_at, primary, secondary?, raw }`.

### 2.6 학생 휴지통 유입 경로

- **`songs`** — 학생이 공개 아카이브에서 `fork_public_item('songs', id)` 로 개인 사본을 만든 뒤 소프트 삭제한 경우. 원본 공개 노래는 소유자가 아니므로 학생 휴지통에 들어가지 않습니다.
- **`left_courses`** — 학생이 「내 공간」의 내 반 위젯에서 `leave_course(_course_id)` 로 반을 나간 경우. `course_members.left_at` 이 기록되며 7일 내 「가입한 반」 탭에서 복원할 수 있습니다.

### 2.7 상태 & 리로드

- 소프트 삭제/복원/영구 삭제 성공 후 `loadAll()` 재실행, 상세·확인 다이얼로그는 닫힘.
- 역할이 바뀌어 현재 탭이 허용 목록에서 벗어나면 `TABLES[0]` 으로 자동 복귀.
- 토스트(`useToast`) 표준 문구: `복원 완료`, `영구 삭제 완료`, `전체 복원`, `전체 영구 삭제`, 실패 시 `오류`(destructive, 기본 설명 「다시 시도해주세요.」).

## 3. Examples

### 3.1 학생 사이드바 렌더 결과

```text
[홈] [내 공간] [노래 아카이브] [휴지통] [사용 가이드] [설정]
```

### 3.2 설정 탭 셰이프

```text
프로필 | 계정 & 보안
```

### 3.3 휴지통 탭 셰이프 (학생)

```text
[노래 (3)]  [가입한 반 (1)]
```

### 3.4 카드 예시

```text
┌──────────────────────────────────────┐
│ 明天会更好_내일은 더 나아진다  6일 남음 │
│ 群星                                  │
│ 삭제일: 2026. 7. 18. 오후 3:22        │
└──────────────────────────────────────┘
```

## 4. Context

- 관련 파일
  - `src/pages/SettingsPage.tsx`
  - `src/components/settings/StudentProfileTab.tsx`
  - `src/components/settings/SecurityTab.tsx`
  - `src/pages/Trash.tsx`
  - `src/lib/trash.ts` (`TRASH_LABEL`, `TRASH_RETENTION_DAYS = 7`, `daysRemaining`, RPC 래퍼, `fetchLeftCourses`)
  - `src/components/workspace/tokens.ts` (`wsBtn` 버튼 토큰)
  - `src/components/AppSidebar.tsx` (`studentItems`)
  - `src/components/guide/student/tourSteps.ts` (`student-settings`, `student-trash`)
  - `src/hooks/useAuth.tsx` (`isTeacher`, `isAdmin`, `user`, `refreshProfile`)
  - `supabase/functions/delete-account` (계정 탈퇴)
- 스토리지: `avatars` 버킷 (공개 읽기, 소유자만 쓰기).
- 자동 청소: DB 함수 `purge_expired_trash()` 가 `deleted_at < now() - 7일` 인 행을 하드 삭제.
- 데이터 소스: `profiles`, `user_roles`, `songs`, `course_members`, `courses`.

### 4.1 다른 문서와의 경계

| 문서 | 범위 | S6 과의 관계 |
|---|---|---|
| **T8 — 교사 휴지통** | 4개 탭, 강의안·학생 프로필 포함 | S6 = 학생이 접근 가능한 2개 탭만 재현 |
| **S2 — 학생 인증/온보딩** | Auth Gateway, 글로벌 온보딩, 반 온보딩 | 프로필 최초 세팅 흐름 — S6 는 이후의 편집 UI |
| **S1 — 학생 내 공간** | 위젯 대시보드, 반 나가기(`leave_course`) | 나간 반이 S6 의 「가입한 반」 탭으로 유입 |
| **S4 — 학생 반 상세** | 반 홈/캘린더/자료/커뮤니티 | 커뮤니티 탭에서 자기 카드 편집 시 온보딩 다이얼로그 재사용 |

## 5. Acceptance

- [ ] 학생 계정으로 `/settings` 진입 시 탭이 정확히 `프로필, 계정 & 보안` 2개만 표시된다.
- [ ] `NotificationsTab` 임포트/렌더가 코드베이스 어디에도 없다(`rg -n "NotificationsTab" src` 결과 0건).
- [ ] 학생 계정으로 `/trash` 진입 시 탭이 정확히 `노래, 가입한 반` 2개만 표시되고 빈 상태 카드까지 정상 렌더된다.
- [ ] 각 탭 카운트 뱃지가 실데이터 개수와 일치한다.
- [ ] 카드 클릭 → 상세 다이얼로그 → 복원/영구 삭제가 정상 동작하고, 성공 시 리스트가 즉시 갱신된다.
- [ ] 우클릭 컨텍스트 메뉴에서 `전체 복원` / `전체 영구 삭제` 가 노출되고, 빈 탭에서는 비활성이다.
- [ ] 「가입한 반」 탭의 복원/영구 삭제가 `restore_course_membership` / `purge_course_membership` 을 호출하며, `_table` 인자를 가진 RPC 로는 호출되지 않는다.
- [ ] `restore_all_trash('songs')` / `purge_all_trash('songs')` 호출이 학생 소유 행만 대상으로 한다.
- [ ] 남은 일수 뱃지가 1일 이하 붉은색, 3일 이하 호박색, 그 외 인디고로 표시된다.
- [ ] 프로필 탭에서 아바타 업로드가 2MB / JPG·PNG·WEBP 제한을 지키고, 업로드 후 사이드바 프로필이 즉시 갱신된다.
- [ ] 프로필 저장 시 `hsk_level="없음"` / `topik_level="없음"` 이 DB 에는 `null` 로 기록된다.
- [ ] 계정 & 보안 탭에서 현재 비밀번호 재인증 후 비밀번호 변경이 실제로 성공하고, 소셜 로그인 전용 계정에서는 폼 대신 안내 문구만 보인다.
- [ ] 학생 계정으로는 `수업 환경` 탭 및 강의안/학생 프로필 휴지통에 어떤 경로로도 접근할 수 없다.

## Non-Goals

- 알림 탭, 알림 관련 저장·로직 일체.
- 교사 「수업 환경」 탭 (T-series 참조).
- 강의안(`lesson_plans`) · 반(`courses`) · 학생 프로필(`course_student_profiles`) 휴지통 (T8 전용).
- 대시보드 · 반 상세 · 노래 아카이브 등 다른 학생 화면 (S1/S4 참조).
