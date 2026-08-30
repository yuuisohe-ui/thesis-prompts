# S6 · 학생 설정 & 학생 휴지통

## 1. Identity

당신은 **멜로디 클래스(Melody Class)** 플랫폼의 학생 사이드 프론트엔드 엔지니어입니다. 이 프롬프트는 학생 계정으로 로그인했을 때 접근 가능한 두 개의 사이드바 항목 **「설정」(`/settings`)** 과 **「휴지통」(`/trash`)** 의 학생 전용 브랜치를 재현하기 위한 사양서입니다. 교사·관리자 전용 로직(수업 환경 탭, 강의안·학생 프로필 휴지통 등)은 이 문서의 범위 밖입니다.

## 2. Instructions

### 2.1 라우팅 & 사이드바 진입

- `AppSidebar.tsx` 의 `studentItems` 배열은 다음 순서를 유지합니다:
  1. 홈 `/`
  2. 내 공간 `/student-home`
  3. 노래 아카이브 `/songs`
  4. **휴지통 `/trash`** (`Trash2` 아이콘)
  5. 사용 가이드 `/guide`
  6. **설정 `/settings`** (`Settings` 아이콘)
- 두 경로 모두 `RequireAuth → AppLayout` 로 래핑됩니다.
- 관리자 뷰 전환(👩‍🏫 교사 / 🎓 학생)은 사이드바 하단에서 이루어지지만, 학생 계정 자체에서는 노출되지 않습니다.

### 2.2 학생 설정 (`SettingsPage.tsx`)

- 페이지 폭 `max-w-3xl`, 제목 「설정」(#2e3d6b, 22px semibold), 설명 「플랫폼 설정을 관리하세요.」
- `useAuth()` 로 `isTeacher` 를 얻어 다음과 같이 탭을 분기합니다:
  - 교사: `[프로필, 수업 환경, 계정 & 보안]`
  - **학생: `[프로필, 계정 & 보안]`**
- 알림(`notify`) 탭은 **양쪽 모두에서 완전히 제거**되었습니다. `NotificationsTab` 컴포넌트 파일도 존재하지 않습니다.
- 탭 스타일: 밑줄형 (`border-b-2`, 활성 시 `#2e3d6b`).
- 미로그인 상태에서는 로그인 유도 카드를 렌더링합니다.

#### 프로필 탭 — `StudentProfileTab`

- 데이터: `profiles.{full_name, avatar_url, department, student_id, hsk_level, topik_level, learning_duration, gender}`.
- 아바타
  - `avatars` 스토리지 버킷 업로드 (경로 `${user.id}/avatar-${ts}.${ext}`).
  - 2MB 이하, JPG/PNG/WEBP 만 허용.
  - 업로드 성공 시 `profiles.avatar_url` 갱신 및 `refreshProfile()` 호출.
- 편집 필드
  - 이름 (필수, 최대 50자)
  - 학번 (선택, 최대 30자, placeholder `2024xxxxxx`)
  - 학과 (선택, 최대 100자)
  - HSK 수준 `[없음, HSK 1-3, HSK 4-6, HSK 7-9]`
  - TOPIK 수준 `[없음, TOPIK 1-2, TOPIK 3-4, TOPIK 5-6]`
  - 학습 기간 `[6개월 미만, 1년, 2년, 3년, 4년, 5년 이상]`
  - 성별 라디오 `[남 / 여 / 기타]`
  - 이메일 (읽기 전용)
- 저장 시 `hsk_level="없음"` / `topik_level="없음"` 은 `null` 로 저장.

#### 계정 & 보안 탭 — `SecurityTab`

- 비밀번호 변경: `supabase.auth.updateUser({ password })` — 현재 실제로 동작.
- Google 로그인 사용자에게는 안내 문구 노출, 자체 비밀번호 없음.
- 로그아웃, 계정 삭제 요청(있다면) 등 기존 컴포넌트 유지.

### 2.3 학생 휴지통 (`Trash.tsx` — 학생 브랜치)

- 헤더: `Trash2` 아이콘 + 「휴지통」, 보관 안내 「삭제한 항목은 **7일**간 보관 후 자동으로 영구 삭제됩니다. 본인이 삭제한 항목만 표시됩니다.」
- 우상단 액션 버튼 2종: `전체 복원`(RotateCcw) · `휴지통 비우기`(Trash) — 현재 탭 항목이 0이면 비활성.
- **탭 구성 (역할별 분기)**
  - 교사/관리자: `songs, lesson_plans, courses, course_student_profiles` (T8 참조)
  - **학생: `songs, courses` — 정확히 2개 탭만 표시**
- 탭 라벨은 `TRASH_LABEL` 매핑: `songs → 노래`, `courses → 과정`.
- 각 탭 옆에 카운트 `Badge` (기본 회색, 곧 만료 시 destructive).
- 카드 그리드: `sm:grid-cols-2 lg:grid-cols-3`, 카드에 `primary`(제목) · `secondary`(부제) · 삭제일 · `X일 남음` 뱃지 (`daysRemaining ≤ 1` 이면 destructive).
- 카드 클릭 → 상세 다이얼로그(제목, 부제, 삭제일 + 남은 일수 + 복원/영구 삭제 버튼).
- 리스트 배경 우클릭 → `ContextMenu`: `전체 복원` / `휴지통 비우기`.
- 확인 다이얼로그
  - Bulk: 「{TRASH_LABEL[tab]} 전체 복원/영구 삭제 — N개 항목」, 영구 삭제 시 「이 작업은 되돌릴 수 없습니다.」 경고.
  - Single: 「‘{primary}’ 항목을 복원/영구 삭제합니다.」

### 2.4 RPC 계약 (SECURITY DEFINER)

학생 UI 는 아래 5개 RPC 만 호출하며, `_table` 인자는 반드시 `'songs'` 또는 `'courses'` 여야 합니다. 나머지 값은 서버에서 거부됩니다.

| RPC | 인자 | 반환 |
|---|---|---|
| `soft_delete_item(_table, _id)` | text, uuid | void |
| `restore_trash_item(_table, _id)` | text, uuid | void |
| `purge_trash_item(_table, _id)` | text, uuid | void |
| `restore_all_trash(_table)` | text | integer |
| `purge_all_trash(_table)` | text | integer |

권한 규칙 (RPC 내부):
- `owner_id = auth.uid()` 인 행만 대상.
- 관리자(`is_admin()`)는 `owner_id IS NULL` 공개 행까지 대상 — 학생에게는 해당 없음.

### 2.5 데이터 수집

- 클라이언트는 `TABLES.map` 으로 병렬 SELECT:
  - `songs`: `id, deleted_at, title, artist, title_bilingual, language`
  - `courses`: `id, deleted_at, name, level, semester`
- 필터: `.eq("owner_id", user.id).not("deleted_at","is",null).order("deleted_at",{ascending:false})`.
- 결과는 `Partial<Record<TrashTable, TrashRow[]>>` 로 저장(부분 셰이프 안전).

### 2.6 학생 휴지통 유입 경로

- **`songs`** — 학생이 공개 아카이브에서 `fork_public_item('songs', id)` 로 개인 사본을 만든 후 `move_to_trash('songs', id)` 를 호출한 경우. 원본 공개 노래는 소유자가 아니므로 학생 휴지통에 들어가지 않습니다.
- **`courses`** — 학생이 「내 수업」에서 본인이 소유한 과정 카드(포크된 사본)를 삭제한 경우. 초대 링크로 단순 참여한 과정은 `course_members` 에서 나갈 뿐 학생 휴지통에는 들어가지 않습니다.

### 2.7 상태 & 리로드

- 소프트 삭제/복원/영구 삭제 성공 후 `loadAll()` 재실행.
- 성공/실패 시 `useToast` 로 표준 문구 표시:
  - `복원 완료`, `영구 삭제 완료`, `전체 복원`, `전체 영구 삭제`, `오류` (destructive variant).

## 3. Examples

### 3.1 학생 사이드바 렌더 결과

```
[홈] [내 공간] [노래 아카이브] [휴지통] [사용 가이드] [설정]
```

### 3.2 설정 탭 셰이프

```
프로필 | 계정 & 보안
```

### 3.3 휴지통 탭 셰이프 (학생)

```
[노래 (3)]  [과정 (1)]
```

### 3.4 카드 예시

```
┌──────────────────────────────────┐
│ 明天会更好_내일은 더 나아진다  6일 남음 │
│ 群星                              │
│ 삭제일: 2026. 7. 18. 오후 3:22   │
└──────────────────────────────────┘
```

## 4. Context

- 관련 파일
  - `src/pages/SettingsPage.tsx`
  - `src/components/settings/StudentProfileTab.tsx`
  - `src/components/settings/SecurityTab.tsx`
  - `src/pages/Trash.tsx`
  - `src/lib/trash.ts` (`TRASH_LABEL`, `TRASH_RETENTION_DAYS = 7`, `daysRemaining`, RPC 래퍼)
  - `src/components/AppSidebar.tsx` (`studentItems`)
  - `src/hooks/useAuth.tsx` (`isTeacher`, `isAdmin`, `user`, `refreshProfile`)
- 스토리지: `avatars` 버킷 (공개 읽기, 소유자만 쓰기).
- 자동 청소: DB 함수 `purge_expired_trash()` 가 `deleted_at < now() - 7일` 인 행을 하드 삭제.
- 데이터 소스: `profiles`, `user_roles`, `songs`, `courses`, `user_preferences`(학생은 사용하지 않음).

### 4.1 다른 문서와의 경계

| 문서 | 범위 | S6 과의 관계 |
|---|---|---|
| **T8 — 교사 휴지통** | 4개 탭, 강의안·학생 프로필 포함 | S6 = 그 중 학생 접근 가능한 2개 탭만 재현 |
| **S2 — 학생 인증/온보딩** | Auth Gateway, 글로벌 온보딩, 반 온보딩 | 프로필 최초 세팅 흐름 — S6 는 이후의 편집 UI |
| **S4 — 학생 반 상세** | 반 홈/캘린더/자료/알림/커뮤니티 | 커뮤니티 탭에서 자기 카드 편집 시 온보딩 다이얼로그 재사용 |
| **T1a/T1b — 공용 위젯** | 대시보드 위젯 카탈로그 | 학생 대시보드는 S1 에서 다룸 — 이 문서와 무관 |

## 5. Acceptance

- [ ] 학생 계정으로 `/settings` 진입 시 탭이 정확히 `프로필, 계정 & 보안` 2개만 표시된다.
- [ ] `NotificationsTab` 임포트/렌더가 코드베이스 어디에도 없다.
- [ ] 학생 계정으로 `/trash` 진입 시 탭이 정확히 `노래, 과정` 2개만 표시되고 페이지가 정상적으로 렌더된다(빈 상태 카드 포함).
- [ ] 각 탭 카운트 뱃지가 실데이터 개수와 일치한다.
- [ ] 카드 클릭 → 상세 다이얼로그 → 복원/영구 삭제가 정상 동작하고, 성공 시 리스트가 즉시 갱신된다.
- [ ] 우클릭 컨텍스트 메뉴에서 `전체 복원` / `휴지통 비우기` 가 노출된다.
- [ ] `restore_all_trash('songs')` / `purge_all_trash('courses')` 등의 호출이 학생 소유 행만 대상으로 한다.
- [ ] 프로필 탭에서 아바타 업로드가 2MB / JPG·PNG·WEBP 제한을 지키고, 업로드 후 사이드바 프로필이 즉시 갱신된다.
- [ ] 프로필 저장 시 `hsk_level="없음"` / `topik_level="없음"` 이 DB 에는 `null` 로 기록된다.
- [ ] 계정 & 보안 탭의 비밀번호 변경이 실제로 성공한다(이메일/비밀번호 사용자 기준).
- [ ] 학생 계정으로는 `수업 환경` 탭 및 강의안/학생 프로필 휴지통에 어떤 경로로도 접근할 수 없다.

## Non-Goals

- 알림 탭, 알림 관련 저장·로직 일체.
- 교사 「수업 환경」 탭 (T-series 참조).
- 강의안(`lesson_plans`) · 학생 프로필(`course_student_profiles`) 휴지통 (T8 전용).
- 대시보드 · 반 상세 · 노래 아카이브 등 다른 학생 화면 (S1/S4 참조).
