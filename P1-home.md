# P1 · 공용 홈페이지 (`/`) 재현 프롬프트

> **적용 대상**: `src/pages/Home.tsx` 및 상단 고정 NAV / 히어로 / WHY / 통계 / 피처(역할 스위처) / CTA / Footer 섹션.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 따른다.**

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages: Identity, Instructions, Examples, Context.* OpenAI Platform Documentation. Retrieved July 12, 2026, from https://platform.openai.com/docs/guides/prompt-engineering
2. **Lovable. (n.d.).** *Prompting best practices.* Lovable Documentation. Retrieved July 12, 2026, from https://docs.lovable.dev/prompting/prompting-one
3. **IEEE. (1998).** *IEEE Recommended Practice for Software Requirements Specifications* (IEEE Std 830-1998), §4.3.6 "Verifiable", p. 7. IEEE.
4. **Cohn, M. (2004).** *User Stories Applied: For Agile Software Development*, Ch. 6, pp. 67–74. Addison-Wesley.

---

## ① Identity (신원)
당신은 한중 이중언어 교육 SaaS 「멜로디 클래스」의 공용 랜딩 페이지를 담당하는 시니어 프론트엔드 엔지니어이자 UX 라이터입니다. 기술 스택은 React 18 + Vite 5 + Tailwind CSS v3 + shadcn/ui + Supabase JS v2 이며, 대한민국 대학 K-Chinese 교사·학습자의 감성적 톤을 이해합니다.

## ② Instructions (지시)

### 2.1 산출물 (Component-not-Page)
- `src/pages/Home.tsx` — 라우트 `/` 엔트리. 인증 상태별 라우팅 분기 포함.
- `TopNav` (동일 파일 내 지역 컴포넌트) — 로고 + 앵커 2개(왜 노래인가 · 주요 기능) + 로그인 CTA. `fixed top-0 z-50`, 다크네이비 배경 + 백드롭 블러.
- `HeroSection` — 배지 · H1(2줄, 골드 `<em>` 강조) · Sub · Primary/Secondary CTA · 하단 통계 3카드.
- `WhySection` — 골드 태그 + 큰따옴표 인용문(H2급) + 본문 2줄 + 출처 캡션.
- `FeaturesSection` — 캡슐형 역할 스위처(교사↔학생) + 각 4탭 미리보기.
- `CtaBanner`, `Footer` — 하단 전환 유도 및 공용 푸터.

### 2.2 원자적 UI 규칙 (Atomic Language)
- "탑 NAV" 가 아니라 "`h-16 fixed top-0 z-50` + `rgba(15,26,54,0.96)` 배경 + `backdrop-blur-xl` + `border-b border-white/[.06]` + 좌측 브랜드 버튼(`<img src="/brand-logo.png" alt="" className="w-[34px] h-[34px] object-contain rounded-[9px]" />` + `멜로디 클래스` 텍스트, → `/dashboard`) + 우측 앵커 버튼 2개(스무스 스크롤) + 우측 [로그인] outline 버튼(→ `/onboarding`)".
- 브랜드 로고는 emoji 가 아니라 `public/brand-logo.png` (업로드된 골드 라운드 스퀘어 + 음표 마크)를 `<img>` 로 렌더한다. 동일 원본에서 파생된 `public/favicon.png`(64×64) · `public/apple-touch-icon.png`(180×180) 은 `index.html` 에서 링크한다. 로고 텍스트 옆 이모지 병기 금지.
- **모바일 NAV 현행 동작**: 앵커 버튼 2개는 `hidden sm:inline` 이라 640 px 미만에서 감춰지고, 좌측 브랜드 버튼과 우측 [로그인] 버튼만 남는다. 별도의 햄버거 메뉴·드로어는 현재 존재하지 않는다(재현 시 임의로 추가하지 말 것).
- "히어로" 가 아니라 "다크네이비(`#0F1A36`) 풀-스크린 섹션 + 상단 골드 배지(pill) + H1 2줄(`font-serif`, `clamp(34px,5vw,58px)`, 두 번째 줄 골드 `<em>` 강조) + Sub 1줄(muted white/55) + Primary CTA(골드 그라디언트) + Secondary CTA(반투명 화이트) + 하단 통계 3카드(가운데 정렬, 구분선)".
- "WHY 섹션" 이 아니라 "흰 배경 `py-28`, 상단 골드 uppercase 태그(`tracking-[2.5px]`) + 3px 골드 언더라인 + 큰따옴표 인용문(`font-serif`, 골드 따옴표 강조) + 본문 2줄 + 출처 캡션(`italic`)".
- "역할 스위처" 가 아니라 "capsule pill 2 세그먼트, active 시 `bg-primary text-white`, 클릭 시 크로스페이드 200 ms".

### 2.3 강제 제약
- 색상은 반드시 semantic token(`--dash-gold`, `--dash-gold-border`, `--primary`, `--muted-foreground` 등)만 사용. `text-white`, `bg-black`, `bg-[#…]` 하드코드는 히어로/NAV 다크네이비 배경 등 브랜딩상 불가피한 3곳(nav bg, hero bg, features bg)에 한해 예외로 허용하며 나머지 색은 전부 token.
- 폰트는 프로젝트 전역 스택(`font-serif` 헤딩 · `font-sans` 본문). Inter/Poppins 강제 금지.
- 카피는 순수 한국어. 통계 라벨 "All"·"Live"·"1000+" 는 브랜딩 예외.
- `<h1>` 은 페이지에 단 하나(히어로 제목).
- 인증 상태 조회는 `useAuth()` 훅에만 위임. `supabase.auth.onAuthStateChange` 를 Home 내부에서 직접 구독하지 않는다.
- 앵커 ID 는 정확히 세 개만 사용: `#home`(hero), `#why`, `#features`.

## ③ Examples (예시)

### 3.1 확정 카피 표 (Real Content)
| 위치 | 한국어 카피 |
|---|---|
| NAV 로고 | `/brand-logo.png` 이미지(34×34, `rounded-[9px]`) + 멜로디 클래스 |
| NAV 앵커 1 | 왜 노래인가 (→ `#why` 스무스 스크롤) |
| NAV 앵커 2 | 주요 기능 (→ `#features` 스무스 스크롤) |
| NAV CTA | 로그인 (→ `/onboarding`) |
| Hero 배지 | 🎶 한중 노래 기반 언어 교육 플랫폼 |
| Hero H1 | 노래 한 곡이 &lt;br/&gt; **&lt;em&gt;교재 한 권보다&lt;/em&gt;** 오래 남습니다 (두 번째 줄 골드 `<em>` 강조) |
| Hero Sub | 가르치는 이에게는 준비를, 배우는 이에게는 즐거움을 |
| Primary CTA | 지금 시작하기 → (골드 그라디언트, → `/onboarding`) |
| Secondary CTA | 기능 살펴보기 (1500 ms 커스텀 이징 스무스 스크롤 → `#features`) |
| Stats 1 | **All** / HSK·TOPIK 전 등급 지원 |
| Stats 2 | **1000+** / 수록 곡 |
| Stats 3 | **Live** / 실시간 발음 평가 |
| WHY 태그 | 왜 노래로 가르치는가 |
| WHY 인용문(H2) | "노래는 단순한 흥미 유발 도구가 아닙니다" (좌우 따옴표 골드 강조) |
| WHY 본문 | 노래는 발음·리듬·어휘·문법을 자연스럽게 반복하게 만드는, &lt;br/&gt; 검증된 언어 입력 방식입니다. |
| WHY 출처 | — Richards, J. C. (1969) |
| Features(교사) H2 | 수업 설계는 플랫폼에 맡기고, 선생님은 가르치는 일에만 집중하세요 |
| Features(학생) H2 | 어려운 교재는 잊고, 좋아하는 노래로 배우세요 |
| CTA 배너 H2 | 더 나은 배움을 오늘부터 시작하세요 |

### 3.2 컴포넌트 트리
```text
<Home>
  ├─ <TopNav>  (fixed, h-16, dark navy)
  │   ├─ <img src="/brand-logo.png" /> 멜로디 클래스
  │   │     (앵커 2개는 sm 미만에서 hidden)
  │   └─ [왜 노래인가] [주요 기능] [로그인]
  ├─ <HeroSection id="home">  (bg #0F1A36)
  │   ├─ Badge · H1 · Sub
  │   ├─ [지금 시작하기 →] [기능 살펴보기]
  │   └─ StatsRow (3 cards: All · 1000+ · Live)
  ├─ <WhySection id="why">  (bg white)
  │   ├─ 골드 태그 "왜 노래로 가르치는가" + 3px underline
  │   ├─ H2 인용문
  │   ├─ 본문 2줄
  │   └─ — Richards, J. C. (1969)
  ├─ <FeaturesSection id="features">  (bg #F7F7F4)
  │   ├─ Capsule Toggle [교사 | 학생]
  │   ├─ 교사 뷰: 탭 4 (강의안 · 수업 홈 · 노래 분석 · 학생 관리)
  │   └─ 학생 뷰: 탭 4 (노래 아카이브 · AI 노래 제작 · 내 수업 · 공용 수업)
  ├─ <CtaBanner>
  └─ <Footer />
```

## ④ Context (배경)

### 4.1 프로젝트 맥락
「멜로디 클래스」는 한중 노래를 매개로 하는 언어 교육 SaaS. 홈 라우트 `/` 는 비로그인/로그인 모두 접근 가능한 공용 랜딩이며, (a) 서비스 정체성 전달, (b) 로그인·회원가입 CTA 유도, (c) 노래 기반 언어 교육의 학술적 근거 제시(WHY 섹션), (d) 교사·학생 두 유형의 핵심 가치 소개, (e) 로그인 상태이면 즉시 역할별 대시보드로 자동 전환을 담당한다. WHY 섹션의 Richards (1969) 인용은 페이지에 렌더되는 **콘텐츠**이며, 노래를 언어 학습의 검증된 입력 방식으로 규정한 초기 논문을 사용자에게 명시적으로 제시하기 위해 배치된다.

### 4.2 Lovable Cloud 후경
- `useAuth()` 훅에서 `{ user, isTeacher, isAdmin, needsOnboarding, loading }` 를 소비.
- 로딩 상태: 화면 중앙에 스피너만 렌더(랜딩을 미리 노출하지 않는다).
- 비로그인: 랜딩을 렌더 → NAV·히어로·CTA 클릭 시 `/onboarding`.
- 로그인 & `teacher`/`admin`: `/dashboard` 로 `replace` 이동.
- 로그인 & `student`: `/student-home` 으로 `replace` 이동.
- 로그인 & 역할 미배정: `/onboarding` 으로 `replace` 이동.

### 4.3 데이터 계약
홈 자체는 별도 테이블을 소유하지 않는다. 인증·역할·프로필 스키마는 P2(인증·온보딩) 문서를 참조.

## ⑤ Acceptance & Output (검증·산출)

### 5.1 Acceptance Criteria (정량 임계값)
- Lighthouse 모바일 성능 ≥ 85, 접근성 ≥ 95, CLS ≤ 0.05.
- 인증 상태 변화 감지 후 라우팅 전환까지 ≤ 100 ms(개발자 도구 Performance 계측 기준).
- `rg -n 'id="home"|id="why"|id="features"' src/pages/Home.tsx | wc -l` = 3.
- `rg -n "<h1" src/pages/Home.tsx | wc -l` = 1.
- `rg -n "Richards, J. C. \(1969\)" src/pages/Home.tsx | wc -l` ≥ 1 (WHY 섹션 출처 캡션 렌더 확인).
- `rg -n "노래는 단순한 흥미 유발 도구가 아닙니다" src/pages/Home.tsx | wc -l` = 1.
- 히어로 통계 카드 개수 = 3 (`querySelectorAll` 로 검증).
- NAV 브랜드 영역에 `<img src="/brand-logo.png">` 가 정확히 1개 존재하고 emoji 로고 문자열은 0건(`rg -n "/brand-logo.png" src/pages/Home.tsx | wc -l` = 1).
- `/brand-logo.png` · `/favicon.png` · `/apple-touch-icon.png` 요청이 각각 HTTP 200 · `content-type: image/png`.
- 뷰포트 390 px 에서 NAV 앵커 2개는 비표시, 브랜드 버튼과 [로그인] 버튼만 노출된다.
- 상단 NAV 는 `position: fixed` 이며 스크롤해도 항상 상단 노출(Playwright `getBoundingClientRect().top === 0`).
- `#why` / `#features` 앵커 클릭 시 스무스 스크롤 동작, `#features` 는 커스텀 이징 1500 ms.
- 다크모드 히어로 텍스트 대비비 ≥ 4.5 (axe DevTools).
- 역할 스위처 클릭 시 레이아웃 시프트 없음(CLS 델타 ≤ 0.01), 크로스페이드 200 ms.

### 5.2 Output Format
LLM은 아래 두 블록만 반환한다.
1. `src/pages/Home.tsx` 전체 파일 코드(TSX).
2. 변경 사항 요약 3줄(한국어).

설명·사과·주석·마크다운 헤더는 반환하지 않는다.
