# 00 · 프롬프트 표준 템플릿 (Prompt Standard Template)

> **본 파일은 논문 부록 「프롬프트 재현 자료(Prompt Reproduction Appendix)」 전체 50개 파일(본 템플릿 1 · README 1 · 공용 P 계열 27 · 교사 T 계열 13 · 학생 S 계열 5 · 백엔드 B 계열 3)가 공통으로 따르는 골격을 정의한다.** 본 템플릿은 Lovable 플랫폼 상에서 가장 안정적으로 해석·실행되는 구조를 채택하며, 그 이론적 근거는 아래 네 건의 일차 자료에 기반한다.

---

## 이론적 근거 (Theoretical Grounding)

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages: Identity, Instructions, Examples, Context.* OpenAI Platform Documentation. Retrieved July 12, 2026, from https://platform.openai.com/docs/guides/prompt-engineering
   → 외곽 4-요소 골격(①Identity · ②Instructions · ③Examples · ④Context) 채택 근거.
2. **Lovable. (n.d.).** *Prompting best practices.* Lovable Documentation. Retrieved July 12, 2026, from https://docs.lovable.dev/prompting/prompting-one
   → 본 부록에서 실제로 인용된 5개 실천 원칙(Prompt by Component, Not Page · Design with Real Content · Speak Atomic · Use Prompt Patterns for Layouts · Build with Lovable Cloud in Mind) 채택 근거. 원문 문서에는 이 외에도 다수의 실천 팁이 있으나, 본 부록에서는 위 5개만 명시적으로 원용한다.
3. **IEEE. (1998).** *IEEE Recommended Practice for Software Requirements Specifications* (IEEE Std 830-1998), §4.3.6 "Verifiable", p. 7. IEEE.
   → ⑤Acceptance & Output 절의 "정량 임계값으로만 서술한다" 규칙의 근거. 요구사항이 검증 가능하려면 반드시 측정 가능한 수치로 표현되어야 한다는 원칙을 그대로 수용.
4. **Cohn, M. (2004).** *User Stories Applied: For Agile Software Development*, Chapter 6 "Acceptance Testing User Stories", pp. 67–74 (특히 p. 68). Addison-Wesley.
   → 각 모듈별 Acceptance Criteria가 "해당 기능이 완결되었는지 판단하는 기본 기준"을 제공한다는 원칙의 근거.

---

## 골격 (5-Section Skeleton)

본 부록의 모든 프롬프트 문서는 정확히 아래 다섯 절로 구성된다. 앞 네 절(①–④)은 OpenAI 개발자 메시지 4-요소를 그대로 따르며, ⑤절은 IEEE 830 §4.3.6 및 Cohn(2004)에 근거하여 완결 판정을 위해 추가한 절이다. Lovable 실천 원칙 중 아래 5개가 각 절 내부에서 명시적으로 원용된다: Prompt by Component, Not Page(②2.1) · Speak Atomic(②2.2) · Design with Real Content(②2.3, ③3.1) · Use Prompt Patterns for Layouts(③3.2) · Build with Lovable Cloud in Mind(④4.2).

### ① Identity (신원)
한 문단으로 다음을 확정한다.
- 역할: 시니어 프론트엔드 엔지니어 겸 한중 이중언어 교육 UX 라이터.
- 기술 스택: React 18 + Vite 5 + Tailwind CSS v3 + shadcn/ui + Supabase(JS v2).
- 언어·문화 정위: 대한민국 대학의 K-Chinese/K-Korean 교사·학습자.

### ② Instructions (지시)
2.1 **산출물** — Lovable 실천 원칙 "Prompt by Component, Not Page"에 따라, 페이지가 아닌 파일·컴포넌트 단위로 나열한다.
2.2 **원자적 UI 규칙 (Atomic Language)** — Lovable 실천 원칙 "Speak Atomic"에 따라, "카드"가 아니라 "커버(16:9) + 제목 2줄 clamp + 하트 토글이 있는 카드"처럼 요소 단위로 서술한다.
2.3 **강제 제약** — semantic token만 사용, `bg-[#…]` / `text-white` / `bg-black` 하드코드 금지, `<h1>` 페이지당 하나, 모든 fetch는 `fetchWithRetry` + AbortController, 순수 한국어 카피(lorem ipsum 금지, Lovable 실천 원칙 "Design with Real Content").

### ③ Examples (예시)
3.1 **확정 카피 표** — Lovable 실천 원칙 "Design with Real Content"에 따라, 실제 렌더링될 한국어 문자열을 위치별로 표로 확정한다.
3.2 **파일 트리 / 컴포넌트 트리** — ASCII 트리로 구조를 예시한다(Lovable 실천 원칙 "Use Prompt Patterns for Layouts" 반영).

### ④ Context (배경)
4.1 **프로젝트 맥락** — 「멜로디 클래스」의 위치와 해당 모듈의 시스템 내 역할.
4.2 **Lovable Cloud 후경** — Auth, RLS, GRANT, 로딩/빈/에러/성공 4-상태 렌더링(Lovable 실천 원칙 "Build with Lovable Cloud in Mind").
4.3 **데이터 계약** — 관련 테이블의 `CREATE TABLE` + `GRANT` + `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` + `CREATE POLICY`를 이 순서로 명시(백엔드가 관여하지 않는 문서는 생략 가능).

### ⑤ Acceptance & Output (검증·산출)
5.1 **Acceptance Criteria** — IEEE 830 §4.3.6에 따라 **정량 임계값으로만 기술한다**. 예: Lighthouse 성능 ≥ 85, CLS ≤ 0.05, `grep 'bg-\['` 결과 0건, p95 응답 ≤ 300 ms.
5.2 **Output Format** — LLM(또는 Lovable) 이 반환할 파일 순서를 번호로 명시하고, 설명·사과·주석·마크다운 헤더 등 부수 텍스트를 명시적으로 금지한다.

---

## 사용 지침

1. 새로운 프롬프트 문서를 생성할 때는 본 파일의 5-Section 골격을 그대로 복제한다.
2. 상단에는 반드시 위 4건의 이론적 근거를 재게시하거나 본 파일을 명시적으로 참조한다.
3. Acceptance Criteria의 모든 항목은 자동/수동 검증이 가능한 수치·명령·정규식으로 표현한다. "잘 보인다", "빠르다" 같은 정성적 서술은 금지한다.
4. Output Format은 재현 시 파일 순서가 바뀌지 않도록 번호를 유지한다.
