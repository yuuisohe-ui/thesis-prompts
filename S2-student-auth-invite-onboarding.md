# S2 · 학생 인증 · 초대 · 온보딩 (Auth · Invite · Onboarding)

> **재현 목표**: 학생이 플랫폼과 처음 만나는 지점 전부. 초대 링크를 받은 순간부터, 로그인 게이트 통과, 글로벌 온보딩(계정 최초 1회), 코스별 온보딩(반마다 프로필 확인)까지의 완결된 진입 플로우를 하나의 프롬프트로 재현한다.
>
> **범위**: `src/pages/Auth.tsx`, `src/pages/Onboarding.tsx`, `src/components/GlobalOnboardingDialog.tsx`, `src/components/courses/StudentOnboardingDialog.tsx`, `src/components/courses/JoinCourseDialog.tsx`, `src/pages/CourseDetail.tsx` 의 학생 게이트 부분, `src/hooks/useAuth.tsx`, `src/App.tsx` 의 `/shared/course/:token` 라우트, `src/components/RequireAuth.tsx`, `src/components/courses/studentAvatars.ts`.
>
> **비고**: S1「내 공간」·S4「반 상세 학생 뷰」·P3 계열「노래 아카이브·곡 학습」의 진입 지점만 제공하고 그 안의 UI 는 각 문서에 위임한다. 교사 온보딩·교사 대시보드는 포함하지 않는다.

---

## 1. Identity

너는 학생 사용자 여정의 "0단계 ~ 1단계" 를 담당하는 진입 게이트를 만든다. 이 문서가 재현하는 결과물은 다음 세 개의 시나리오를 모두 통과해야 한다:

1. **직접 방문 시나리오**: 학생이 URL 을 직접 입력해 `/auth` 로 들어와 Google 로 로그인 → 역할 판정 → `/onboarding` 또는 `/student-home` 으로 라우팅.
2. **초대 링크 시나리오**: 교사가 보낸 `https://<origin>/shared/course/<share_token>` 링크 클릭 → 비로그인이면 `/auth?redirect=...&courseName=...` 로 강제 리다이렉트 → 로그인 후 원래 링크로 복귀 → 코스 진입 전 `StudentOnboardingDialog` 강제 노출.
3. **재입장 시나리오**: 이미 프로필이 저장된 학생이 같은 코스로 재진입 → 다이얼로그 스킵, 곧바로 홈 탭.

Google 로그인이 학생 기본값이다. 이메일/비밀번호는 초대 링크 진입에서는 숨기고, 직접 방문 시에만 하단에 노출한다.

## 2. Instructions

### 2.1 라우트 · 접근 제어

App 라우트에 다음 세 개를 등록한다.

```tsx
// src/App.tsx
<Route path="/auth" element={<Auth />} />
<Route path="/onboarding" element={<Onboarding />} />
<Route path="/shared/course/:token" element={<RequireAuth><CourseDetail /></RequireAuth>} />
```

- `/auth` 와 `/onboarding` 은 인증 없이도 접근 가능. 이미 로그인된 상태로 재방문 시 자동 라우팅한다.
- `/shared/course/:token` 은 `RequireAuth` 로 감싼다. 비로그인 세션이면 컴포넌트 마운트 전에 `/auth?redirect=<원래경로>&course=<id>&courseName=<encoded>` 로 replace 리다이렉트한다.
- `RequireAuth` 는 `useAuth().loading` 을 존중하고, 로딩 중엔 스켈레톤을 보여준다.

### 2.2 인증 페이지 (`Auth.tsx`)

**쿼리 파라미터 파싱**:
- `redirect`: 로그인 후 돌아갈 경로.
- `courseName`: 초대 링크 진입 시 헤더에 노출할 반 이름.
- `isCourseInvite = redirect.startsWith("/shared/course/")` 로 계산한다.

**최상단 카드 헤더**:
- `isCourseInvite && courseName` 이면 반 이름을 큰 글씨로. 아니면 "멜로디 클래스".
- 부제: 초대 진입 시 "수업에 참여하려면 Google 계정으로 로그인해주세요", 일반 진입 시 "로그인하여 계속하세요".

**Google 로그인 버튼 (최상단)**:
- 초대 진입 시 `variant="default"`, 일반 진입 시 `variant="outline"`.
- `redirect_uri` 는 초대 진입 시 현재 URL(`origin + pathname + search`), 일반 진입 시 `origin`.
- 실패 시 destructive 토스트.

**이메일/비밀번호 폼**:
- `!isCourseInvite` 인 경우에만 렌더. 초대 링크 사용자는 반드시 Google 로만.
- 폼 하단에 "계정이 없으신가요? 회원가입" → `/onboarding` 로 이동.

**로그인 후 라우팅** (`onAuthStateChange` + `getSession` 병행):
```
async function routeAfterAuth(userId) {
  if (redirect && redirect.startsWith("/shared/course/")) return navigate(redirect, { replace: true });
  const roles = await select("user_roles", { user_id: userId });
  if (roles.includes("admin") || roles.includes("teacher")) return navigate("/dashboard", { replace: true });
  if (roles.includes("student")) return navigate("/student-home", { replace: true });
  return navigate("/onboarding", { replace: true });
}
```
- `setTimeout(..., 0)` 로 지연 호출해 auth-state 데드락을 회피한다.
- `SIGNED_IN` · `TOKEN_REFRESHED` 이벤트 모두 처리.

### 2.3 전역 온보딩 (`Onboarding.tsx` + `GlobalOnboardingDialog.tsx`)

**목적**: 계정 생애 최초 1회, 역할과 기본 프로필을 입력받는다. 이후 어떤 반에 참여하든 이 정보가 프리필 소스로 재사용된다.

**두 가지 진입점**:
- (a) `/onboarding` 라우트 = 회원가입 마법사(4-step wizard). 역할 선택 → 계정 연결(Google/이메일) → 프로필 입력 → 완료.
- (b) 다른 라우트에서 로그인된 상태로 `needsOnboarding=true` 를 만나면 `<GlobalOnboardingDialog>` 가 자동 오픈(닫기 불가). App 루트에서 항상 마운트되어 있다.

**`useAuth` 조건**: `needsOnboarding = !profiles.onboarded_at && !hasAnyRole`. 즉 (프로필의 `onboarded_at` 이 NULL) AND (`user_roles` 행이 하나도 없을 때) 강제 온보딩. 레거시 유저(역할만 있고 `onboarded_at` NULL)는 통과시킨다.

**수집 필드** (GlobalOnboardingDialog 기준):

| 필드 | 저장 위치 | 필수 | 옵션 |
|---|---|---|---|
| 이름 | `profiles.full_name` | ✓ | user_metadata.full_name 프리필 |
| 역할 | `user_roles.role` | ✓ | `student` / `teacher` (기본 student) |
| 국적 | `profiles.nationality` | ✓ | 대한민국·中国·日本·United States·기타(직접 입력) |
| 모국어 | `profiles.native_language` | ✓ | 한국어·中文·日本語·English·기타 |
| TOPIK | `profiles.topik_level` | | 없음·TOPIK 1~6 |
| HSK | `profiles.hsk_level` | | 없음·HSK 1~6 |
| 영어 | `profiles.english_level` | | 없음·초급·중급·고급·원어민 |
| 소속/직장 | `profiles.affiliation` | ✓ | |
| 직책/학년 | `profiles.job_title` | | |
| 나이 | `profiles.age` | | 숫자 |
| 성별 | `profiles.gender` | | 선택 안 함·남·여·기타 |

- 저장 시 `profiles.upsert({ id: user.id, ..., onboarded_at: now() })` + `user_roles.upsert({ user_id, role }, { onConflict: "user_id,role" })` 순서 실행. 실패 시 destructive 토스트.
- 저장 완료 후 `refreshProfile()` 를 호출해 `useAuth` 를 즉시 갱신 → App 이 정상 라우팅으로 이어감.
- 다이얼로그는 `onEscapeKeyDown` · `onPointerDownOutside` · `onInteractOutside` 를 모두 preventDefault 로 막아 사용자가 닫을 수 없어야 한다.

**`/onboarding` 4-Step Wizard 세부** (Onboarding.tsx):
- Step 1 역할 선택: 큰 카드 두 개(학생/교사). 쿼리 `?role=student` 로 프리셋 가능.
- Step 2 계정 연결: Google 원클릭 or 이메일/비밀번호(로그인·회원가입 토글, 비밀번호 확인 필드, 표시/숨김 토글). Google 성공 시 `redirect_uri = origin + "/onboarding?role=" + role`.
- Step 3 프로필 입력: 역할에 따라 다른 필드 세트(학생: 이름·학번·학과·입학년도·학습기간·HSK·TOPIK, 교사: 이름·학교·학과·직책·담당과목·소개). 저장 시 `profiles` upsert + `user_roles` upsert + `onboarded_at` 스탬프.
- Step 4 완료: 요약 카드 + "시작하기" 버튼 → 역할에 맞는 홈으로 replace 이동.

### 2.4 초대 링크 → 코스 진입 (`CourseDetail.tsx` 학생 게이트)

`/shared/course/:token` 로 진입한 경우:

1. **코스 조회**: `courses.select("*").eq("share_token", token).maybeSingle()`. 실패 시 "과정을 찾을 수 없습니다" 토스트 + 로딩 종료.
2. **인증 게이트** (isShared && !user 이면):
   ```
   const redirect = encodeURIComponent(pathname + search);
   navigate(`/auth?redirect=${redirect}&course=${course.id}&courseName=${encodeURIComponent(course.name)}`, { replace: true });
   ```
3. **방문 추적**: `useTrackShareVisit({ linkType: "course", token, resourceTitle: course.name, enabled: isShared && !!course })` 로 `share_visits` 에 기록.
4. **학생 프로필 게이트**:
   - `course.owner_id === user.id` 이면 skip (교사가 자기 반을 shared 링크로 열어보는 케이스).
   - `!isShared` 진입인데 `user_roles` 에 teacher/admin 이면 skip.
   - 그 외엔 `course_student_profiles.select("id").eq("course_id", course.id).eq("member_user_id", user.id).maybeSingle()` 조회.
   - `course_members.upsert({ course_id, user_id, role: "student" }, { onConflict: "course_id,user_id" })` 로 멤버 행 자동 유지.
   - 프로필 행이 없으면 `setOnboardingOpen(true)`, 있으면 다이얼로그 스킵.

### 2.5 코스별 온보딩 (`StudentOnboardingDialog.tsx`)

**모드**:
- 최초 진입(`existing=null`): 제목 "{반이름} 에 오신 것을 환영합니다 🎉", 취소 시 이전 페이지로 되돌아감.
- 편집(`existing` 있음, CommunityTab 자기 카드 연필 아이콘): 제목 "내 정보 수정", 취소 버튼 노출.

**프리필 소스** (`existing=null` 케이스):
- `profiles.select("full_name, department, student_id, hsk_level, topik_level, learning_duration, gender, avatar_url")` 로 계정 전역 정보를 끌어와 각 필드에 채운다.
- HSK/TOPIK 는 자유 문자열(`HSK 3`)을 버킷(`HSK 1-3`)으로 매핑:
  - `HSK 1~3` → `HSK 1-3`, `HSK 4~6` → `HSK 4-6`, `HSK 7+` → `HSK 7-9`, 없음 → `없음`.
  - TOPIK 도 동일 규칙(`1~2`, `3~4`, `5+`).
- `learning_duration` → `study_years` 매핑: `"6개월"` → `0.5`, 숫자 파싱 후 1~5 사이로 클램프.
- 프리필된 경우 설명 문구를 "가입 시 입력한 정보를 미리 채워뒀습니다. 확인하고 수정한 뒤 입장해주세요." 로 바꾼다.

**수집 필드**:

| 필드 | 저장 위치 | 필수 |
|---|---|---|
| 아바타 | `course_student_profiles.avatar_url` | ✓ (기본값 있음) |
| 이름 | `full_name` | ✓ |
| 학번 | `student_number` | |
| 학과 | `department` | |
| HSK 수준 | `language_level` (합성) | |
| TOPIK 수준 | `language_level` (합성) | |
| 학습 기간 | `study_years` (숫자) | |
| 성별 | `gender` | ✓ (기본 `other`) |

- `language_level` 은 `[hsk, topik]` 중 "없음" 이 아닌 값들을 `" / "` 로 join. 둘 다 없음이면 NULL.
- `emoji` 필드는 항상 NULL 로 저장(레거시 이모지 시스템은 폐기).

**아바타 위젯**:
- 좌측 원형 프리뷰(64px) + 우측 12칸 그리드(`grid-cols-6`, `aspect-square rounded-full`).
- 12칸 중 실제 프리셋이 있는 슬롯은 `PRESET_STUDENT_AVATARS[idx]` 이미지, 없는 슬롯은 점선 dashed placeholder.
- 선택된 프리셋은 `border-primary shadow-sm`. 호버 시 `hover:scale-105`.
- "내 사진 업로드" 버튼 → `avatars` 버킷에 `{user.id}/student-avatar-{ts}.{ext}` 경로로 업로드, publicURL 을 `avatarUrl` 로 설정. 이미지 파일 아님/5MB 초과 시 destructive 토스트.
- 프리셋 목록은 `src/assets/student-avatars/avatar-*.png.asset.json` 을 `import.meta.glob(..., { eager: true })` 로 로드, `url` 필드를 추출해 정렬. 최대 12장이며 없는 슬롯은 자동 skip.

**저장 흐름**:
- INSERT 케이스: 같은 반의 `sort_order` 최댓값을 조회해 `+1` 로 넣어 카드 정렬 안정성을 확보.
- UPDATE 케이스: `.eq("id", existing.id)` 로 갱신.
- 성공 시 `"{반이름}에 입장했습니다"` (또는 "정보가 수정되었습니다") 토스트, `onSaved(profileId)` 콜백, `onOpenChange(false)`.
- 실패 시 destructive 토스트.

**닫기 시 안전장치** (CourseDetail 의 `handleOnboardingOpenChange`):
- 다이얼로그가 저장 없이 닫히면 `course_student_profiles` 를 재조회해 여전히 행이 없으면 "입장이 취소되었습니다" 토스트 + `navigate(-1)`.

### 2.6 반 참여 다이얼로그 (`JoinCourseDialog.tsx`)

- 헤더: "반 참여" / "교사가 보내준 수업 링크나 코드를 입력하세요."
- 입력값에서 UUID 정규식 `[0-9a-f]{8}-[0-9a-f]{4}-...` 로 첫 매치를 추출 → 링크·raw ID 어느 쪽이든 대응.
- `supabase.rpc("get_course_preview_by_token", { _token: id })` 호출. 응답이 배열이면 첫 원소 사용. 실패 시 "수업을 찾을 수 없어요."
- 프리뷰 카드: BookOpen 아이콘 + 반 이름, level 배지, introduction 3줄 클램프.
- "참여하기" 클릭:
  - `course_members.select("id").eq("course_id").eq("user_id").maybeSingle()` 로 중복 확인. 이미 있으면 `toast.info("이미 참여 중인 반이에요.")` + 다이얼로그 닫기 + `student-courses:refresh` 이벤트 + `/courses/:id` 이동.
  - 없으면 `course_members.insert({ course_id, user_id, role: "student" })` 후 성공 토스트 + 이벤트 + 이동.
- 다이얼로그 닫힘 시 상태 리셋(`input, preview, error, loading, joining`).

### 2.7 `useAuth` 훅 계약

```ts
{
  user, profile: { id, full_name, avatar_url, onboarded_at },
  isTeacher, isAdmin, needsOnboarding, loading,
  signOut(), refreshProfile()
}
```

- 마운트 시 `getSession()` + `onAuthStateChange()` 병행 등록.
- 콜백 내 supabase 호출은 반드시 `setTimeout(..., 0)` 지연.
- `SIGNED_IN` 이벤트에서 `login_events` 에 `{ user_id, provider, user_agent(500자 clamp) }` 삽입.
- profiles 행이 없으면 `{ id, full_name: fallbackName, email }` 로 upsert. `email` 이 비어 있으면 backfill.
- `needsOnboarding = !profile.onboarded_at && !hasAnyRole`.

### 2.8 상태 · 이벤트 버스

- `window.dispatchEvent(new Event("student-courses:refresh"))` — 반 참여 성공 후 「내 공간」 학생 카드 갱신을 유도.
- `window.dispatchEvent(new CustomEvent("course:updated", { detail: { id } }))` — 코스 정보 변경 시 CourseDetail 새로고침.

## 3. Examples

### 3.1 초대 링크 첫 진입 시퀀스

```text
학생 브라우저
  │  URL 클릭: https://app/shared/course/abc-123
  ▼
RequireAuth → user==null → /auth?redirect=%2Fshared%2Fcourse%2Fabc-123&courseName=test기초%20中国语
  │  Google 버튼(default variant, 최상단) 클릭
  ▼
lovable.auth.signInWithOAuth("google", { redirect_uri: 현재 URL })
  │  콜백 후 SIGNED_IN
  ▼
routeAfterAuth() → redirect 이 /shared/course/ 로 시작 → navigate(redirect)
  │
  ▼
CourseDetail 마운트 → isShared=true → useEffect: profile 조회 → 없음
  │  course_members.upsert (role: student)
  │  StudentOnboardingDialog(open=true, existing=null) 표시
  │  profiles 조회 → HSK/TOPIK/이름 프리필
  │  안내 문구: "가입 시 입력한 정보를 미리 채워뒀습니다."
  ▼
학생 확인/수정 후 "확인하고 입장하기"
  │  course_student_profiles.insert (sort_order=max+1, avatar_url=선택본)
  │  토스트: "test기초 中国语에 입장했습니다"
  ▼
HomeTab 렌더 완료
```

### 3.2 재입장 시퀀스

```text
CourseDetail 마운트 → course_student_profiles.select → id 반환
  │  onboardingOpen 유지 false → 다이얼로그 스킵
  ▼
HomeTab 즉시 렌더
```

### 3.3 다이얼로그 취소 안전장치

```text
학생이 실수로 다이얼로그 배경 대신 닫기 버튼 클릭
  │  onOpenChange(false)
  │  handleOnboardingOpenChange → profiles 재조회 → 여전히 없음
  ▼
토스트: "입장이 취소되었습니다"
navigate(-1)
```

## 4. Context

### 4.1 스키마 · RPC 의존성

- `profiles(id, full_name, avatar_url, onboarded_at, nationality, native_language, topik_level, hsk_level, english_level, affiliation, job_title, age, gender, email, department, student_id, learning_duration)`.
- `user_roles(user_id, role)` — RLS 우회는 `has_role()` security definer 함수 사용. 학생 온보딩에서는 `role = 'student'` 를 upsert(onConflict: user_id,role).
- `courses(id, name, share_token, owner_id, is_public, ...)` — `share_token` 은 익명 조회용 UUID.
- `course_members(course_id, user_id, role)` — 학생 자동 upsert.
- `course_student_profiles(id, course_id, member_user_id, full_name, student_number, department, language_level, study_years, gender, avatar_url, emoji, sort_order)` — `(course_id, member_user_id)` 부분 unique 인덱스.
- `share_visits(link_type, token, ...)` — 초대 링크 방문 로그.
- `login_events(user_id, provider, user_agent)` — SIGNED_IN 시 삽입.
- RPC `get_course_preview_by_token(_token uuid)` → `{ id, name, level, introduction }`. anon 접근 가능해야 프리뷰가 뜬다.

### 4.2 RLS 요구사항

- `profiles`: 본인 SELECT/UPDATE/INSERT. 서비스롤 ALL.
- `user_roles`: SELECT authenticated 본인만, UPDATE/DELETE 금지(권한 상승 방지). INSERT 는 self-signup 시 자기 자신에 한해 허용.
- `course_student_profiles`:
  - SELECT: 코스 오너 OR 코스 멤버 OR 자기 자신(member_user_id = auth.uid()).
  - INSERT/UPDATE: `auth.uid() = member_user_id` OR 코스 오너.
- `course_members`: SELECT authenticated (본인 또는 오너). INSERT self.

### 4.3 스토리지

- `avatars` 버킷 public read. 업로드 경로 `{user.id}/...` 로 자기 폴더 강제. RLS 정책: `bucket_id = 'avatars' AND (storage.foldername(name))[1] = auth.uid()::text`.

### 4.4 어셈블링된 12장 프리셋 아바타

- `src/assets/student-avatars/avatar-{01..12}.png.asset.json` 슬롯. 현재 6장(01~06)만 존재.
- `PRESET_STUDENT_AVATARS` 배열은 `import.meta.glob(..., { eager: true })` 로 정렬된 순서로 URL 만 추출. 슬롯 부재는 UI 에서 점선 placeholder 로 렌더.

### 4.5 UI 토큰

- 초대 진입 Google 버튼: `variant="default"`, 아이콘 svg 인라인(4색 Google 로고).
- 다이얼로그 최대폭 `max-w-lg` / `max-h-[90vh] overflow-y-auto`.
- 프리뷰 카드 배경 `bg-slate-50`, 반 이름 `text-[14px] font-bold text-slate-800`.
- 아바타 그리드 `grid-cols-6 gap-2`, 선택 보더 `border-primary shadow-sm`.

### 4.6 비-목표

- 교사 온보딩 마법사(Step 3 교사 필드)는 완전한 스펙만 언급하고 UI 상세는 다루지 않는다. 교사 관련 detail 은 T2·T5 에서 처리.
- 이메일 인증 메일 템플릿, HIBP 비밀번호 정책은 백엔드 설정이라 이 문서 범위 밖.
- 소셜 로그인은 Google 만. Apple/GitHub 등은 미지원.
- 초대 링크를 통한 대량 학생 자동 등록(교사 CSV)은 T7 소관.

## 5. Acceptance Criteria

1. 비로그인 상태에서 `/shared/course/<token>` 접근 시 `/auth?redirect=...&courseName=...` 로 replace 되고, Auth 페이지 헤더에 반 이름이 노출된다.
2. Auth 페이지에서 초대 진입일 때 이메일/비밀번호 폼이 숨겨지고, Google 버튼이 default variant 로 최상단에 위치한다.
3. Google 성공 후 원래 초대 URL 로 자동 복귀한다.
4. 계정 최초 로그인 학생은 `/onboarding` 또는 `GlobalOnboardingDialog` 를 만나며, ESC·바깥 클릭으로 닫히지 않는다.
5. 온보딩 저장이 성공하면 `profiles.onboarded_at` 이 채워지고 `user_roles` 에 role 이 upsert 된다.
6. 온보딩 완료 후 학생은 `/student-home` 으로, 교사는 `/dashboard` 로 자동 이동한다.
7. 코스 첫 진입 시 `StudentOnboardingDialog` 가 열리고, 기본 프로필 값(이름·HSK·TOPIK·학과·아바타)이 프리필된다.
8. HSK 3 / TOPIK 5 등의 원시값이 각각 `HSK 1-3` / `TOPIK 5-6` 버킷으로 정확히 매핑된다.
9. 12칸 아바타 그리드가 렌더되고, 실제 자산이 없는 슬롯은 점선 placeholder 로 보인다.
10. 5MB 초과 이미지·비이미지 파일 업로드 시 destructive 토스트가 뜨고 저장되지 않는다.
11. 저장 후 `course_student_profiles` 에 `sort_order = max+1`, `avatar_url` 세팅, `emoji=null` 로 행이 생긴다.
12. 재입장 시 다이얼로그는 열리지 않고 곧바로 홈 탭이 표시된다.
13. 다이얼로그를 저장 없이 닫으면 "입장이 취소되었습니다" 토스트와 함께 이전 페이지로 돌아간다.
14. `JoinCourseDialog` 에서 링크·raw UUID 어느 쪽을 넣어도 UUID 추출 후 프리뷰가 뜨고, 이미 참여한 반이면 info 토스트 + 이동만 발생한다.
15. `student-courses:refresh` 이벤트가 참여 성공 시 항상 dispatch 되어 「내 공간」 모듈이 자동 갱신된다.
16. `SIGNED_IN` 시 `login_events` 에 provider · user_agent(500자 clamp) 가 기록된다.
17. 교사가 자기 소유 반을 `/shared/course/<token>` 으로 열어도 학생 온보딩 다이얼로그가 뜨지 않는다.
