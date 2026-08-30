# P2 · 인증 · 온보딩 · 역할 부여 재현 프롬프트

> **적용 대상**: `src/pages/Auth.tsx`, `src/pages/Onboarding.tsx`, `src/hooks/useAuth.tsx`, `src/components/RequireAuth.tsx`, Supabase `profiles` · `user_roles` 테이블과 RLS.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 따른다.**

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages: Identity, Instructions, Examples, Context.* OpenAI Platform Documentation. Retrieved July 12, 2026, from https://platform.openai.com/docs/guides/prompt-engineering
2. **Lovable. (n.d.).** *Prompting best practices.* Lovable Documentation. Retrieved July 12, 2026, from https://docs.lovable.dev/prompting/prompting-one
3. **IEEE. (1998).** *IEEE Recommended Practice for Software Requirements Specifications* (IEEE Std 830-1998), §4.3.6 "Verifiable", p. 7. IEEE.
4. **Cohn, M. (2004).** *User Stories Applied: For Agile Software Development*, Ch. 6, pp. 67–74. Addison-Wesley.

---

## ① Identity (신원)
당신은 Supabase(Postgres + RLS) 기반 SaaS 인증 흐름을 설계하는 풀스택 엔지니어이자 보안 감사자입니다. React 18 + TypeScript + Vite 5 + Tailwind CSS v3 + shadcn/ui + `@supabase/supabase-js` 를 능숙히 다루며, 대한민국 대학 K-Chinese/K-Korean 교사·학습자를 대상으로 하는 온보딩 UX 를 설계합니다. OWASP A01(Broken Access Control) 관점에서 역할 저장 위치와 셀프-부트스트랩 정책을 감사합니다.

## ② Instructions (지시)

### 2.1 산출물 (Component-not-Page)
- `src/hooks/useAuth.tsx` — 세션·프로필·역할·`needsOnboarding` 파생 상태를 노출하는 컨텍스트 훅.
- `src/components/RequireAuth.tsx` — 미인증 리다이렉트, 온보딩 강제, 학생 라우트 화이트리스트.
- `src/pages/Auth.tsx` — Google/이메일 로그인 UI, 초대 링크 감지, 로그인 성공 시 역할 기반 라우팅.
- `src/pages/Onboarding.tsx` — 프로필(성명·프로필 사진·소속·역할) 입력 후 `profiles`·`user_roles` upsert.
- Postgres 마이그레이션 — `app_role` enum, `profiles`, `user_roles`, GRANT, RLS, `has_role()` SECURITY DEFINER 함수, 셀프-부트스트랩 정책 1개.

### 2.2 원자적 UI 규칙 (Atomic Language)
- "로그인 화면" 이 아니라 "카드(max-w-md, `bg-card`, 라운드 shadow) + 상단 로고 + [Google 로 계속하기] 버튼(Google 컬러 아이콘) + 구분선 + 이메일 인풋 + 비밀번호 인풋 + [로그인] 버튼 + 하단 [회원가입] 링크".
- "온보딩 화면" 이 아니라 "2단계 스텝퍼(1/2 → 2/2) · 진행 표시 바 · 각 단계 상단 제목 + 설명 · 하단 [이전] · [다음] · 최종 [시작하기]".
- "프로필 사진 선택" 이 아니라 "12개 프리셋 아바타 그리드(4열×3행) + [사진 업로드] 버튼 · 선택된 아바타에 골드 링 강조".
- "역할 선택" 이 아니라 "라디오 카드 2개: [교사] · [학생], 각 카드에 아이콘 + 설명 1줄 + 활성 시 primary 링".

### 2.3 강제 제약
- **역할은 절대 `profiles`에 저장 금지** — `user_roles`만 신뢰. (OWASP A01)
- `has_role()` 은 `SECURITY DEFINER`, `SET search_path = public`, `stable`.
- `user_roles` RLS 는 반드시 세 정책으로 분리: SELECT(본인) · INSERT(셀프-부트스트랩, `admin` 제외) · 나머지는 service_role 만.
- `profiles`·`user_roles` 에 `GRANT`를 반드시 명시(Supabase 는 기본 grant 없음).
- `supabase.auth.onAuthStateChange` 콜백 내부에서 `supabase.from(...)` 을 직접 호출하지 말고 `setTimeout(0)` 으로 지연시켜 auth deadlock 회피.
- `RequireAuth` 학생 화이트리스트: `/student-home`, `/songs`, `/guide`, `/settings`, `/shared/`, `/embed/`, `/courses/`.
- 초대 링크(`?redirect=/shared/course/...`)로 진입한 경우 OAuth redirect_uri 는 **현재 페이지 전체 URL** 이어야 하며, 세션 복원 후 그 경로로 이동한다.
- 카피는 순수 한국어. 실패 시 사유를 담은 한국어 토스트.
- 색상은 semantic token 만. `bg-[#…]` / `text-white` / `bg-black` 금지.

## ③ Examples (예시)

### 3.1 확정 카피 표 (Real Content)
| 위치 | 카피 |
|---|---|
| Auth 페이지 H1 | 멜로디 클래스에 로그인 |
| Google 버튼 | Google 로 계속하기 |
| 이메일 인풋 placeholder | 이메일 |
| 비밀번호 인풋 placeholder | 비밀번호 |
| 로그인 버튼 | 로그인 |
| 회원가입 링크 | 계정이 없으신가요? 회원가입 |
| 온보딩 스텝 1 제목 | 프로필을 완성해 주세요 |
| 온보딩 스텝 2 제목 | 어떤 역할로 시작하시겠어요? |
| 역할 카드 (교사) | 교사 · 강의안·수업을 만들고 학생을 관리합니다 |
| 역할 카드 (학생) | 학생 · 수업에 참여하고 노래로 배웁니다 |
| 최종 CTA | 시작하기 |
| 저장 실패 토스트 | 저장에 실패했습니다: {reason} |
| 권한 오류 토스트 | 권한이 없습니다. 다시 로그인해 주세요. |

### 3.2 인증·라우팅 흐름 트리
```text
[Landing/Invite] → /auth
      │
      ├─ Google OAuth → Supabase → onAuthStateChange(SIGNED_IN)
      │       └─ login_events insert (provider, ua)
      │
      └─ Email/Password → signInWithPassword
              │
              ▼
      useAuth.syncUserState(user)
              │
              ├─ profiles select → 없으면 upsert(fallbackName)
              ├─ user_roles select → isTeacher/isAdmin/needsOnboarding 계산
              │       needsOnboarding = !onboarded_at && roles.length === 0
              ▼
      RequireAuth 판단
              ├─ !user            → /auth?redirect=...
              ├─ needsOnboarding  → /onboarding
              ├─ student only     → 화이트리스트 외 접근 시 /student-home
              └─ else             → 요청 경로 렌더
```

## ④ Context (배경)

### 4.1 프로젝트 맥락
「멜로디 클래스」는 교사·학생·관리자 3역할 SaaS. 인증 경로는 두 가지: (a) Google OAuth(랜딩 또는 수업 초대 링크에서 진입), (b) 이메일+비밀번호(백오피스/관리자용). 첫 로그인 후 사용자는 반드시 `/onboarding` 을 통해 프로필과 역할을 확정해야 하고, 확정 이후에만 대시보드/학생 홈에 접근할 수 있다. 수업 초대 링크(`/shared/course/:id`)로 진입한 학생은 로그인 직후 초대 링크로 되돌아가야 한다.

### 4.2 Lovable Cloud 후경 (4-상태)
- **loading**: `useAuth.loading === true` → 화면 중앙 스피너. 하위 컴포넌트를 미리 렌더하지 않는다.
- **empty(미인증)**: `RequireAuth` 가 `/auth?redirect=<currentPath>` 로 replace.
- **error**: `signInWithOAuth`/`signInWithPassword` 실패 → 한국어 토스트 + 인풋 유지.
- **success**: 세션 확보 → `profiles` upsert → `user_roles` 조회 → 역할별 라우팅.
- Google Provider 설정: redirect_uri 는 `window.location.origin` 기반 동적 문자열이며, 초대 링크의 경우 현재 페이지 전체 URL.

### 4.3 데이터 계약
```sql
-- 1) enum
create type public.app_role as enum ('admin','teacher','student');

-- 2) profiles
create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  full_name text not null,
  email text,
  avatar_url text,
  school text, title text, bio text,
  onboarded_at timestamptz
);
grant select, insert, update on public.profiles to authenticated;
grant all on public.profiles to service_role;
alter table public.profiles enable row level security;
create policy "own profile select" on public.profiles for select using (auth.uid() = id);
create policy "own profile insert" on public.profiles for insert with check (auth.uid() = id);
create policy "own profile update" on public.profiles for update using (auth.uid() = id);

-- 3) user_roles
create table public.user_roles (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  role public.app_role not null,
  unique(user_id, role)
);
grant select on public.user_roles to authenticated;
grant all on public.user_roles to service_role;
alter table public.user_roles enable row level security;

create policy "own roles select" on public.user_roles
  for select using (auth.uid() = user_id);

-- 셀프-부트스트랩: 최초 1회, teacher/student 만 허용, admin 금지
create policy "self bootstrap non-admin role" on public.user_roles
  for insert with check (
    auth.uid() = user_id
    and role in ('teacher','student')
    and not exists (
      select 1 from public.user_roles ur where ur.user_id = auth.uid()
    )
  );

-- 4) has_role — SECURITY DEFINER, stable, search_path 고정
create or replace function public.has_role(_user_id uuid, _role public.app_role)
returns boolean
language sql
stable
security definer
set search_path = public
as $$
  select exists (
    select 1 from public.user_roles
    where user_id = _user_id and role = _role
  )
$$;
```

## ⑤ Acceptance & Output (검증·산출)

### 5.1 Acceptance Criteria (정량 임계값)
- 신규 학생 Google 최초 로그인 → `/onboarding` → 역할 선택 → `/student-home` 도달까지 클릭 수 ≤ 3, 총 소요 시간 ≤ 30 s(로컬 계측).
- 신규 교사 Google 최초 로그인 → `/onboarding` → 교사 필드 입력 → `/dashboard` 도달까지 클릭 수 ≤ 5.
- 학생이 `/dashboard` 직접 접근 → `/student-home` 으로 100% replace(수동 5회 시도 전부 성공).
- 로그아웃 상태에서 `/settings` 접근 → `/auth?redirect=/settings` 이동, 로그인 후 `/settings` 복귀. 왕복 성공률 = 100%.
- `INSERT INTO public.user_roles(user_id, role) VALUES (auth.uid(), 'admin')` 를 SQL Playground 에서 authenticated 로 실행 시 반드시 `permission denied` 또는 RLS violation 반환. 성공하면 즉시 실패로 판정.
- 애플리케이션 로그에 `permission denied for table profiles` 또는 `permission denied for table user_roles` 발생 건수 = 0.
- `rg -n "role.*=.*'admin'" src/pages/Onboarding.tsx src/pages/Auth.tsx src/components/RequireAuth.tsx src/hooks/useAuth.tsx | wc -l` = 0 (클라이언트에서 admin 을 하드코드로 부여하지 않음).
- `rg -n "bg-\[#|text-white|bg-black" src/pages/Auth.tsx src/pages/Onboarding.tsx src/components/RequireAuth.tsx | wc -l` = 0.
- Lighthouse 접근성 ≥ 95, CLS ≤ 0.05.
- `onAuthStateChange` 콜백에서 `supabase.from(...)` 직접 호출 grep 결과 = 0 (`rg -nA5 "onAuthStateChange" src | rg "supabase\.from"` 로 검증).

### 5.2 Output Format
반환 순서(그 외 텍스트 금지):
1. Postgres 마이그레이션 SQL (enum · profiles · user_roles · GRANT · RLS · `has_role()`).
2. `src/hooks/useAuth.tsx`
3. `src/components/RequireAuth.tsx`
4. `src/pages/Auth.tsx`
5. `src/pages/Onboarding.tsx`
6. 한국어 3줄 요약.
