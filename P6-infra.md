# P6 · 공용 인프라 · 레이아웃 · 사이드바 재현 프롬프트

> **적용 대상**: `src/App.tsx`, `src/main.tsx`, `src/components/AppLayout.tsx`, `src/components/AppSidebar.tsx`, `src/components/RequireAuth.tsx`, `src/index.css`, `tailwind.config.ts`, `src/lib/utils.ts`, `src/lib/fetchWithRetry.ts`.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 따른다.**

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages: Identity, Instructions, Examples, Context.* OpenAI Platform Documentation. Retrieved July 12, 2026, from https://platform.openai.com/docs/guides/prompt-engineering
2. **Lovable. (n.d.).** *Prompting best practices.* Lovable Documentation. Retrieved July 12, 2026, from https://docs.lovable.dev/prompting/prompting-one
3. **IEEE. (1998).** *IEEE Recommended Practice for Software Requirements Specifications* (IEEE Std 830-1998), §4.3.6 "Verifiable", p. 7. IEEE.
4. **Cohn, M. (2004).** *User Stories Applied: For Agile Software Development*, Ch. 6, pp. 67–74. Addison-Wesley.

---

## ① Identity (신원)
당신은 React 18 + Vite 5 + Tailwind CSS v3 + shadcn/ui 기반의 대규모 SaaS 프론트엔드 아키텍트입니다. 디자인 토큰 시스템, 라우팅, 역할 기반 접근 제어, 네트워크 재시도 정책, 접근성을 종합 설계합니다.

## ② Instructions (지시)

### 2.1 산출물 (Component-not-Page)
- `src/index.css` — HSL 기반 semantic token(`--background`, `--foreground`, `--card`, `--primary`, `--sidebar-*` 등) + Tailwind directives.
- `tailwind.config.ts` — token → HSL var 매핑.
- `src/App.tsx` — `BrowserRouter`, 모든 Route 정의, lazy loading, `RequireAuth` 래핑, 404.
- `src/components/AppLayout.tsx` — `SidebarProvider` + 헤더(트리거·태그라인·아바타·로그아웃) + main + Footer.
- `src/components/AppSidebar.tsx` — 역할별 메뉴(교사/학생), 축소 상태, admin view switcher.
- `src/components/RequireAuth.tsx` — 미인증 → `/auth?redirect=…`, 온보딩 필요 → `/onboarding`, 학생 접근 화이트리스트.
- `src/lib/fetchWithRetry.ts` — Exponential Backoff + AbortController + 429/402 한국어 토스트.

### 2.2 원자적 UI 규칙 (Atomic Language)
- "헤더"가 아니라 "h-14 border-b bg-card, 좌측 SidebarTrigger + '한중 노래 기반 교육 플랫폼' 태그라인, 우측 Avatar(fallback 이니셜) + 이름(sm 이상 노출) + 로그아웃 ghost 아이콘 버튼".
- "사이드바"가 아니라 "좌측 세로 네비, 축소 시 아이콘만 표시, aria-label 유지, 역할별 메뉴 배열".
- "재시도 정책"이 아니라 "최대 3회, 초기 400 ms, backoff 2배, `429` → '요청이 많습니다. 잠시 후 다시 시도해주세요.', `402` → 'AI 크레딧이 부족합니다.' 토스트".

### 2.3 강제 제약
- 하드코드 색상 유틸리티 금지(`text-white`, `bg-black`, `bg-[#…]`). 반드시 semantic token.
- 모든 내부 관리 페이지는 반드시 `AppLayout` 으로 감싼다(`SidebarProvider` 컨텍스트 확보).
- `<h1>` 은 각 페이지에 단 하나.
- lazy 로딩 페이지는 `Suspense fallback={<PageLoader/>}` 로 감싸 CLS 방지.
- 라우트 전환 시 스크롤 top으로 초기화.
- 모바일 breakpoint(`sm`, `md`) 에서 사이드바 자동 접힘.
- `supabase.auth.onAuthStateChange` 는 `useAuth` 훅 내부에서만 구독하며, 다른 컴포넌트는 파생 상태만 소비.

## ③ Examples (예시)

### 3.1 확정 카피 표 (Real Content)
| 위치 | 카피 |
|---|---|
| 헤더 태그라인 | 한중 노래 기반 교육 플랫폼 |
| 로그아웃 버튼 title | 로그아웃 |
| 미로그인 CTA | 로그인 |
| 429 토스트 | 요청이 많습니다. 잠시 후 다시 시도해주세요. |
| 402 토스트 | AI 크레딧이 부족합니다. |
| 404 페이지 | 요청하신 페이지를 찾을 수 없습니다. |
| 교사 사이드바 | 홈 · 대시보드 · 노래 아카이브 · 수업 관리 · 휴지통 · 사용 가이드 · 설정 |
| 학생 사이드바 | 홈 · 내 공간 · 노래 아카이브 · 휴지통 · 사용 가이드 · 설정 |
| 숨김(라우트 유지) | 활동 기록 · 자료실 |

### 3.2 AppLayout 마크업 (반드시 유지)
```tsx
<SidebarProvider>
  <div className="min-h-screen flex w-full min-w-0 overflow-x-hidden">
    <AppSidebar />
    <div className="flex-1 flex flex-col min-w-0 overflow-x-hidden">
      <header className="h-14 flex items-center border-b bg-card px-4 justify-between min-w-0">
        <div className="flex items-center">
          <SidebarTrigger className="mr-4" />
          <span className="text-sm text-muted-foreground">한중 노래 기반 교육 플랫폼</span>
        </div>
        {/* Avatar · 이름 · 로그아웃 */}
      </header>
      <main className="flex-1 p-3 sm:p-4 md:p-6 min-w-0 overflow-x-hidden">{children}</main>
      <Footer />
    </div>
  </div>
</SidebarProvider>
```

## ④ Context (배경)

### 4.1 프로젝트 맥락
SPA 구조. 라우트는 3층으로 구분:
- **공용**: `/`, `/auth`, `/onboarding`, `/shared/*`, `/embed/*`.
- **교사**: `/dashboard`, `/workspace`, `/lessons/*`, `/courses/*`, `/students`, `/materials`, `/activity`, `/trash`.
- **학생**: `/student-home`, `/courses/:id`(뷰어), `/settings`.

### 4.2 Lovable Cloud 후경
- `useAuth()` 훅에서 `{ user, profile, isTeacher, isAdmin, needsOnboarding, signOut, loading }` 소비.
- `RequireAuth` 는 4가지 상태를 처리: loading → 스피너, 미인증 → `/auth?redirect=`, 온보딩 필요 → `/onboarding`, 학생이 화이트리스트 밖 경로 접근 시 → `/student-home`.
- 학생 접근 화이트리스트: `/student-home`, `/songs`, `/guide`, `/settings`, `/shared/`, `/embed/`, `/courses/`.

### 4.3 데이터 계약
본 문서 자체는 스키마를 소유하지 않는다. 프로필·역할 스키마는 P2, 코스·수업은 P5/T3 문서를 참조.

## ⑤ Acceptance & Output (검증·산출)

### 5.1 Acceptance Criteria (정량 임계값)
- 다크/라이트 토글 시 사이드바·헤더·본문 색상이 semantic token 만으로 전환(하드코드 grep 결과 0건).
- `rg -n "bg-\[#|text-white|bg-black" src/components/AppLayout.tsx src/components/AppSidebar.tsx src/App.tsx` = 0 건.
- 축소 상태 사이드바에서 라벨 숨김 + `aria-label` 유지, axe 검사 위반 0건.
- admin 계정 로그인 후 사이드바 하단 뷰 스위처(교사/학생 프리뷰) 노출.
- `fetchWithRetry` 는 429 응답 시 정확히 3회 지수 재시도(초기 400 ms · 배수 2), 로그로 검증.
- `AppLayout` 없이 렌더된 내부 페이지 수 = 0 (`rg -L "AppLayout" src/pages` 로 화이트리스트만 남음).
- Lighthouse 접근성 ≥ 95, CLS ≤ 0.05.
- 라우트 전환 후 `window.scrollY === 0` 만족률 = 100 %.

### 5.2 Output Format
반환 순서(그 외 텍스트 금지):
1. `src/index.css` (CSS 변수 + Tailwind directives)
2. `tailwind.config.ts`
3. `src/App.tsx`
4. `src/components/AppLayout.tsx`
5. `src/components/AppSidebar.tsx`
6. `src/components/RequireAuth.tsx`
7. `src/lib/fetchWithRetry.ts`
8. 한국어 3줄 요약.
