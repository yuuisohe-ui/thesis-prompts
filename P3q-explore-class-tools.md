# P3q · 탐구 · 수업 도구 3모듈 재현 프롬프트

> **본 프롬프트는 P3 시리즈의 17/17 (마지막).**
> **적용 대상 (실측)**:
> - `src/components/songs/ClassToolLangDialog.tsx` (103행) — 공용 언어 선택 게이트, `mode: "output" | "study"` 2 모드.
> - `src/components/songs/DiscussionPage.tsx` (162행) — 토론 질문 4개, contentEditable 인라인 편집, `window.open` + `window.print()` 인쇄.
> - `src/components/songs/WritingTopicPage.tsx` (268행) — 난이도 3단계(`easy/mid/hard`) 카드, 힌트 3개 + lyric 인용 + 예문 3종(short/mid/long) 아코디언 탭, HTML 다운로드.
> - `src/components/songs/QuizSheetPage.tsx` (255행) — 4개 유형(all/word/grammar/lyric), 편집 모드, `@media print` 인쇄, HTML 다운로드.
> - `supabase/functions/generate-discussion-questions/index.ts` (142행).
> - `supabase/functions/generate-writing-topics/index.ts` (174행).
> - `supabase/functions/generate-quiz-sheet/index.ts` (239행) — v2 캐시 키, 규칙 위반 문항 서버측 정규식 필터.
>
> 본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다. **1개 파일로 전량 커버**되며 누락은 없다(총 코드 1,343행, 공용 게이트를 3 모듈이 재사용하므로 반복 서술 없이 통합 서술 가능).

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering.* https://platform.openai.com/docs/guides/prompt-engineering (Retrieved July 12, 2026)
2. **Lovable. (n.d.).** *Prompting best practices.* https://docs.lovable.dev/prompting/prompting-one (Retrieved July 12, 2026)
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6 "Verifiable", p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity

당신은 한국 대학의 교사가 30초 내에 "오늘 수업에 그대로 쓸 수 있는 3종 산출물(토론 질문·작문 주제·인쇄용 퀴즈 시트)"을 생성·편집·인쇄·공유할 수 있게 만드는 시니어 프론트엔드 엔지니어이자 한중 이중언어 교수-학습 자료 편집자입니다. React 18 + Vite 5 + Tailwind CSS v3 + shadcn/ui + Supabase(JS v2) 스택 위에서, Lovable Cloud Edge Functions(gpt-4o-mini, `response_format: json_object`)를 호출해 `song_feature_cache` 테이블에 언어별로 캐싱하며, 한국어(ko) / 중국어(zh) 두 언어를 완전히 분리해 관리합니다.

## ② Instructions

### 2.1 산출물 (Prompt by Component)

프론트엔드 컴포넌트 (4):
- `src/components/songs/ClassToolLangDialog.tsx` — 공용 언어 게이트. props: `open, onOpenChange, toolTitle, toolIcon?, onConfirm(lang), description?, mode?: "output" | "study"`. mode 별 카피가 달라짐(§3.1).
- `src/components/songs/DiscussionPage.tsx` — 토론 질문 4개 (문자열 배열, 카드 구조 아님). 인라인 contentEditable 편집. 인쇄.
- `src/components/songs/WritingTopicPage.tsx` — 난이도 3장 카드 (easy/mid/hard), 힌트 3개 + lyric 인용 + 3-탭 예문 아코디언(short/mid/long). HTML 저장 + 링크 공유.
- `src/components/songs/QuizSheetPage.tsx` — 상단 유형 토글 (전체/단어/문법/가사) → 시트 미리보기 + 편집 모드 + `@media print` 인쇄 + HTML 저장 + 링크 공유.

엣지 함수 (3):
- `supabase/functions/generate-discussion-questions/index.ts`.
- `supabase/functions/generate-writing-topics/index.ts`.
- `supabase/functions/generate-quiz-sheet/index.ts` (언어 정규식 검증 포함).

### 2.2 원자적 UI 규칙 (Speak Atomic)

**ClassToolLangDialog** (shadcn `Dialog`, `max-w-md`):
- 제목: `{toolIcon}{toolTitle}` (아이콘은 이모지 문자열, `mr-1`).
- 설명: `mode="study"` 이면 "학습 대상 언어를 선택하세요. (선택한 언어로 학습자가 풀게 됩니다)", `mode="output"` (기본) 이면 "생성할 자료의 언어를 선택하세요.". `description` prop 이 있으면 우선.
- 섹션 라벨(`text-[11px] font-bold uppercase tracking-wider text-muted-foreground`): `언어 선택`.
- 언어 카드 2개 `grid-cols-2 gap-3`:
  - 한국어 카드: `text-lg font-bold` 로 `한국어` + `text-[11px] opacity-80` 부라벨. output 모드 = `한글로 생성`, study 모드 = `한국어를 학습하는 시트`. 선택 시 `border-blue-600 bg-blue-50 text-blue-700`, 비선택 시 `border-border bg-muted/40 text-muted-foreground hover:border-blue-300`.
  - 중국어 카드: `中文` + 부라벨. output = `중국어(한자)로 생성`, study = `중국어를 학습하는 시트`. 선택 시 `border-red-600 bg-red-50 text-red-600`.
- 카드 공통 shape: `rounded-[10px] border-2 p-4 text-center transition-all`. 초기 선택은 `ko`. dialog `open` 이 true 로 전이할 때마다 `ko` 로 리셋 (`useEffect`).
- CTA 버튼: 텍스트 `✦ 생성하기`, `w-full h-11 text-sm font-bold`, 클릭 시 `onConfirm(lang)` 후 `onOpenChange(false)`.

**공용 뒤로가기 버튼(모든 3 페이지 공통)**:
- 텍스트 `← 수업 도구` (앞에 `← ` 유니코드 화살표), `text-[13px] text-muted-foreground border border-border bg-card px-3 py-1.5 rounded-lg hover:border-primary hover:text-primary`.

**DiscussionPage**:
- 제목행: `<h2 className="text-lg font-bold">💬 토론 질문 생성</h2>` + 우측에 `text-xs text-muted-foreground` 로 `({language === "ko" ? "한국어" : "中文"})`.
- 부제(`text-[13px] text-muted-foreground`): "수업 중 활용할 토론 질문을 AI가 생성합니다. 텍스트를 직접 수정할 수 있습니다."
- 카드 래퍼: `bg-card border border-border rounded-2xl p-5 shadow-sm`.
- 4-상태:
  - loading: `Loader2 animate-spin` + "토론 질문을 생성하고 있습니다..." (`text-sm text-primary`, `py-12`).
  - error: `AlertCircle` + 서버 메시지 + `↺ 다시 시도` 버튼(빨간 테두리).
  - success: 헤더행 `"토론 질문"` (`text-sm font-bold`) 우측에 `↺ 재생성` (테두리) / `🖨 인쇄` (`border-amber-300 text-amber-700 bg-amber-50`).
  - empty: success 조건 `questions.length > 0` 미충족 시 아무것도 렌더 X (loading/error 로 커버).
- 질문 항목: `flex gap-3 bg-muted/40 border border-border rounded-[10px] p-3.5` — 좌측 번호 원(`w-7 h-7 rounded-full bg-primary text-primary-foreground text-xs font-bold`), 우측 `contentEditable suppressContentEditableWarning`, blur 시 `handleEdit(i, textContent)` 로 로컬 state 업데이트. 포커스 시 `focus:border-b focus:border-dashed focus:border-primary` 로 힌트.
- 인쇄: `window.open("", "_blank")` 새 창에 인라인 CSS 를 포함한 HTML 을 `document.write`, `setTimeout(() => w.print(), 200)`. `<h2>{title} — {artist}</h2>` + subhead "토론 질문" (zh 모드에서는 `讨论问题`). 번호 원(`#1a2540` 다크 네이비 배경) + 질문 텍스트. 모든 사용자 문자열은 `esc()` 로 HTML 이스케이프 (`& < > " '`).

**WritingTopicPage**:
- 제목: `<h2>✍️ 작문 주제 생성</h2>` + 언어 뱃지.
- 부제: "난이도별 작문 주제, 힌트, 예문을 AI가 생성합니다."
- 성공 상태 헤더 우측 3버튼: `↺ 재생성` / `💾 HTML 저장` (`border-purple-300 text-purple-600 bg-purple-50`) / `🔗 링크 공유` (`border-emerald-300 text-emerald-600 bg-emerald-50`).
- 각 카드(3개, `border border-border rounded-xl overflow-hidden bg-background`):
  1. 상단 스트립(`border-b border-border`): 난이도 pill (`easy=쉬움 bg-emerald-50 text-emerald-600`, `mid=보통 bg-amber-50 text-amber-600`, `hard=어려움 bg-red-50 text-red-600`, `text-[11px] font-bold px-2.5 py-0.5 rounded-full`) + `t.length` (예: `3~5문장`).
  2. 주제(`text-[15px] font-bold`).
  3. `💡 쓰기 힌트` 섹션(`text-[11px] font-bold text-muted-foreground`) — 리스트 3항 (`•` 프라이머리 색).
  4. lyric 인용 블록(존재 시): `bg-primary/5 border-l-[3px] border-primary rounded text-[13px] text-primary`, 텍스트는 `🎵 {t.lyric}`.
  5. 예문 토글: `📖 예문 보기` (`ChevronDown/Up 3.5w`).
  6. 열림 시 내부 탭 3개(`짧은 예문 / 중간 예문 / 긴 예문`) — 활성 탭 `text-primary border-primary`, 본문 `whitespace-pre-wrap text-[13px] leading-relaxed`.
- 상태 `openIdx: Record<number, boolean>`, `tabIdx: Record<number, "short"|"mid"|"long">`, 기본 탭 `short`.
- `💾 HTML 저장`: 순수 HTML 문자열 생성 → `Blob([html], "text/html")` → `URL.createObjectURL` → `<a download="작문주제_{title}_{lang}.html">` 클릭 후 `revokeObjectURL`. HTML 안에 인라인 CSS(카드/뱃지/lyric/예문 블록) 포함. 성공 시 `toast.success("HTML 파일을 저장했습니다")`.
- `🔗 링크 공유`: `navigator.clipboard.writeText(window.location.href)` 후 `toast.success("링크가 복사되었습니다")` / 실패 시 `toast.error("링크 복사에 실패했습니다")`.

**QuizSheetPage**:
- 제목: `<h2>📄 퀴즈 시트 생성</h2>` + 언어 뱃지.
- 부제: "단어·문법·가사 퀴즈를 선택해 편집 가능한 문제지를 생성합니다."
- 유형 토글행(`text-[11px] font-bold uppercase tracking-wider text-muted-foreground` 라벨 `퀴즈 유형`):
  - `전체 / 단어 퀴즈 / 문법 퀴즈 / 가사 퀴즈` 4버튼 (`quizType: "all" | "word" | "grammar" | "lyric"`).
  - 활성 스타일 `border-primary bg-primary/10 text-primary`.
  - 변경 시 `useEffect` 로 재요청 (force=false → 캐시 활용).
- 성공 헤더 우측 2버튼: `↺ 재생성` (force=true) / `✎ 편집 모드` (토글, 활성 시 `border-amber-400 text-amber-700 bg-amber-50`, 텍스트 `✎ 편집 완료`).
- 시트 프리뷰(`sheetRef`, `quiz-sheet-print` 클래스, `bg-white text-slate-900`):
  - 헤더 (`bg-[#1a2540] text-white text-center px-5 py-4`): `{title} — {artist}` + 유형 라벨 (`typeLabel[quizType]`).
  - 메타행 (`bg-slate-100 border-b border-slate-200 text-xs text-slate-500 px-5 py-3 flex gap-5`): `이름：` (`min-w-[120px] border-b border-slate-400` 밑줄 빈칸), `날짜：`(80px), `점수：`(60px) `/ 100`.
  - 본문 (`px-5 py-4`): 각 문항 `border-b border-dashed`, 마지막은 `last:border-0`. `q-num` 텍스트 `{i+1}.`. 질문 `q-text` + `opts grid grid-cols-2 gap-1.5 text-[13px] text-slate-600`. 옵션 마커 `optionMarks = ["①","②","③","④"]`.
  - `editMode` 시 질문/옵션 모두 `contentEditable`.
- 아래 액션 바(`no-print`): `💾 HTML 저장` (`border-purple-300 text-purple-600 bg-purple-50`) / `🔗 링크 공유` (`border-emerald-300 text-emerald-600 bg-emerald-50`).
- 인쇄 CSS (컴포넌트 상단 `<style>` 인라인):
  ```
  @media print {
    body * { visibility: hidden; }
    .quiz-sheet-print, .quiz-sheet-print * { visibility: visible; }
    .quiz-sheet-print { position: absolute; left: 0; top: 0; width: 100%; }
    .no-print { display: none !important; }
  }
  ```
- HTML 저장: `sheetRef.current.outerHTML` 을 인쇄용 스타일과 함께 감싸 `Blob → download`. 파일명 `{title}_{quizType}_{language}.html`.

### 2.3 강제 제약

- semantic token(`primary`, `muted-foreground`, `card`, `border`, `destructive`) 만 사용. 예외로 시트 헤더 다크 네이비 `#1a2540` (인쇄 프리셋 브랜드 상수)와 인쇄용 HTML 문자열 내부 인라인 CSS 는 허용(외부 DOM 이 아니라 새 창/파일 문서라 tailwind 접근 불가).
- 폰트: 인쇄용 HTML/새 창 문서 `font-family: 'Malgun Gothic', sans-serif`.
- 모든 사용자 데이터를 인쇄/저장 HTML 로 삽입할 때 `esc()` 헬퍼로 `& < > " '` 이스케이프 필수.
- 클립보드: `navigator.clipboard.writeText` 만 사용, 실패 시 catch 로 `toast.error` (fallback DOM `execCommand` 는 불필요, 현행 코드는 catch-only).
- 언어 스위처 값(ko/zh) 은 완전 분리 캐시 (`feature_type` suffix `_ko` / `_zh`). 재요청 시 `force=true` 파라미터로 우회.
- 학습자 응답은 저장하지 않음 (수업 도구는 read-only 산출물). 편집 결과는 로컬 state 만.
- 순수 한국어 UI 카피, lorem ipsum 금지.

## ③ Examples

### 3.1 확정 카피 표

| 위치 | 카피 |
|---|---|
| Dialog 설명 · output | `생성할 자료의 언어를 선택하세요.` |
| Dialog 설명 · study | `학습 대상 언어를 선택하세요. (선택한 언어로 학습자가 풀게 됩니다)` |
| 언어 섹션 라벨 | `언어 선택` |
| ko 카드 부라벨 (output/study) | `한글로 생성` / `한국어를 학습하는 시트` |
| zh 카드 부라벨 (output/study) | `중국어(한자)로 생성` / `중국어를 학습하는 시트` |
| Dialog CTA | `✦ 생성하기` |
| 공용 뒤로가기 | `← 수업 도구` |
| Discussion 제목 | `💬 토론 질문 생성` |
| Discussion 부제 | `수업 중 활용할 토론 질문을 AI가 생성합니다. 텍스트를 직접 수정할 수 있습니다.` |
| Discussion 로딩 | `토론 질문을 생성하고 있습니다...` |
| Discussion 리스트 헤더 | `토론 질문` |
| Discussion 인쇄 subhead (ko/zh) | `토론 질문` / `讨论问题` |
| WritingTopic 제목 | `✍️ 작문 주제 생성` |
| WritingTopic 부제 | `난이도별 작문 주제, 힌트, 예문을 AI가 생성합니다.` |
| WritingTopic 리스트 헤더 | `작문 주제 (난이도별 3개)` |
| WritingTopic 힌트 섹션 | `💡 쓰기 힌트` |
| WritingTopic lyric 프리픽스 | `🎵 ` |
| WritingTopic 예문 토글 | `📖 예문 보기` |
| WritingTopic 난이도 라벨 | `쉬움` / `보통` / `어려움` |
| WritingTopic 예문 탭 | `짧은 예문` / `중간 예문` / `긴 예문` |
| WritingTopic 다운로드 파일명 | `작문주제_{title}_{lang}.html` |
| QuizSheet 제목 | `📄 퀴즈 시트 생성` |
| QuizSheet 부제 | `단어·문법·가사 퀴즈를 선택해 편집 가능한 문제지를 생성합니다.` |
| QuizSheet 유형 라벨 | `퀴즈 유형` |
| QuizSheet 유형 토글 | `전체` / `단어 퀴즈` / `문법 퀴즈` / `가사 퀴즈` |
| QuizSheet 시트 유형 표시 | `단어 + 문법 + 가사 퀴즈` / `단어 퀴즈` / `문법 퀴즈` / `가사 퀴즈` |
| QuizSheet 시트 메타 | `이름：` / `날짜：` / `점수：` `/ 100` |
| QuizSheet 편집 토글 | `✎ 편집 모드` ↔ `✎ 편집 완료` |
| 공용 재생성 | `↺ 재생성` |
| 공용 재시도 | `↺ 다시 시도` |
| 공용 인쇄 | `🖨 인쇄` |
| 공용 HTML 저장 | `💾 HTML 저장` |
| 공용 링크 공유 | `🔗 링크 공유` |
| 링크 복사 성공/실패 | `링크가 복사되었습니다` / `링크 복사에 실패했습니다` |
| HTML 저장 성공 | `HTML 파일을 저장했습니다` |
| 429 오류 | `AI 호출이 일시적으로 제한되었습니다. 잠시 후 다시 시도해주세요.` |
| 402 오류 | `AI 사용 한도가 소진되었습니다. 워크스페이스 사용량을 확인해주세요.` |
| 502 (형식) | `AI 응답 형식이 올바르지 않습니다.` |
| 퀴즈 검증 실패 | `퀴즈 생성에 실패했습니다. 다시 시도해주세요.` |

### 3.2 컴포넌트 트리 (Use Prompt Patterns for Layouts)

```text
<ClassToolLangDialog open onOpenChange toolTitle toolIcon onConfirm mode>
  Dialog(max-w-md)
    ├─ DialogHeader → 아이콘 + 제목 + 설명(mode별)
    └─ Body
         ├─ 라벨 "언어 선택"
         ├─ grid-cols-2 → [한국어 카드][中文 카드]
         └─ Button "✦ 생성하기"

<DiscussionPage songId songTitle artistName language onBack>
  ├─ 뒤로가기 (← 수업 도구)
  ├─ 헤더 (💬 토론 질문 생성 · 언어)
  ├─ 부제
  └─ Card(rounded-2xl p-5)
       ├─ Loading | Error(↺ 다시 시도) | Empty(no-render)
       └─ Success
            ├─ 헤더행 [토론 질문] · [↺ 재생성][🖨 인쇄]
            └─ 4× 질문(번호 원 + contentEditable)
       └─ 인쇄: window.open → document.write(esc(html)) → w.print()

<WritingTopicPage>
  └─ Card
       └─ Success
            ├─ 헤더 [작문 주제 (난이도별 3개)] · [↺ 재생성][💾 HTML 저장][🔗 링크 공유]
            └─ 3× 카드
                 ├─ pill(난이도) · length
                 ├─ topic
                 ├─ 💡 쓰기 힌트 (3항)
                 ├─ 🎵 lyric 인용(선택)
                 ├─ 📖 예문 보기 토글
                 └─ 열림 시 [짧은][중간][긴] 탭 → 본문

<QuizSheetPage>
  ├─ (no-print) 헤더 · 부제 · 유형 토글 · 재생성/편집 모드
  ├─ (quiz-sheet-print) 시트
  │     ├─ 헤더(#1a2540) 곡명 · 유형 라벨
  │     ├─ 메타행 이름/날짜/점수
  │     └─ 본문 문항 리스트(옵션 ①②③④)
  └─ (no-print) 액션바 [💾 HTML 저장][🔗 링크 공유]
```

## ④ Context

### 4.1 프로젝트 맥락

수업 도구 3종은 「멜로디 클래스」 곡 상세 다이얼로그의 `탐구` 탭 → L2 `수업 도구` 하위에 배치. 진입 전에 `ClassToolLangDialog` 로 언어(도구별 mode 상이)를 반드시 선택해야 하며, 선택 후 각 페이지로 라우팅. 페이지는 진입 즉시 캐시 확인 → 있으면 표시, 없으면 자동 생성. 재생성/유형 전환 시 `force=true`.

### 4.2 Lovable Cloud 후경 (Build with Lovable Cloud in Mind)

- 3 엣지 함수 공통: `deno.land/std@0.168.0/http/server.ts` 대신 `Deno.serve` 사용, `createClient` 는 `esm.sh/@supabase/supabase-js@2.49.4`, service-role 키로 백엔드 접근.
- 모델 `gpt-4o-mini`, `response_format: { type: "json_object" }` 강제.
- 온도: discussion 0.8, writing 0.7, quiz 0.4 (재현성/검증성 강화).
- 4-상태(loading/error/empty/success) 표준 렌더링. 429·402·기타 상태 코드는 서버가 한국어 메시지로 변환해 반환.
- 캐시 테이블: `song_feature_cache(song_id, feature_type, content jsonb)` — upsert `onConflict: "song_id,feature_type"`.
- `feature_type` 키:
  - `discussion_questions_ko` / `discussion_questions_zh`.
  - `writing_topics_ko` / `writing_topics_zh`.
  - `quiz_sheet_v2_{all|word|grammar|lyric}_{ko|zh}` (v2 접두어로 이전 버전 무효화).

### 4.3 데이터 계약

**엣지 함수 I/O**:

```jsonc
// generate-discussion-questions
POST { song_id, language: "ko"|"zh", force?: boolean }
→ { questions: string[4], cached: boolean }
// 실패 시 502 error: "AI 응답 형식이 올바르지 않습니다." (파싱 실패 or length<3)
```

```jsonc
// generate-writing-topics
POST { song_id, language, force? }
→ {
    topics: [
      { level: "easy"|"mid"|"hard",
        topic: string, length: string,
        hints: string[3],
        lyric: string,
        examples: { short: string, mid: string, long: string } }
    ] // 항상 정확히 3개 (level 순서 강제)
    cached: boolean
  }
```

```jsonc
// generate-quiz-sheet
POST { song_id, language, quiz_type: "all"|"word"|"grammar"|"lyric", force? }
→ {
    questions: [{ type: "word"|"grammar"|"lyric", q, options: string[4], answer }],
    quiz_type, language, cached
  }
```

**출제 수량 매트릭스** (`quiz-sheet` 내부):
- `all` → word 4, grammar 4, lyric 4.
- `word` / `grammar` / `lyric` → 해당 유형 5문항, 나머지 0.

**언어 정규식 검증 (quiz-sheet 서버측 필수)**:

```ts
const HANGUL_RE = /[\uAC00-\uD7AF]/;
const HANZI_RE  = /[\u4E00-\u9FFF]/;

// study language = 학습 대상 (lang=ko → 학습자가 한국어를 배움)
// native language = 학습자 모국어 (lang=ko → 중국어)
// word:      options + answer 는 native 문자만 (반대 스크립트 금지)
// grammar/lyric: q + options + answer 는 study 문자만 (반대 스크립트 금지)
// 조건 미충족 문항은 최종 응답에서 필터.
```

**AI 시스템 프롬프트 개요**:
- Discussion: "당신은 한국 대학교의 중국어 교사입니다(ko) / 你是中国大学的韩语教师(zh). 이 곡을 활용한 수업용 토론 질문 4개를 생성. 가사·시대 배경·문화·개인 경험 각도를 고루 반영." 출력 언어 ko/zh 강제.
- Writing: 페르소나 + "난이도 3단계(easy=3~5문장·mid=10문장 내외·hard=단락). 각 카드: 주제 1 · 힌트 3 · 관련 가사 인용 1줄 · 예문 short/mid/long. 모든 텍스트를 출력 언어로."
- Quiz: 학습 대상 언어(studyLangLabel) vs 학습자 모국어(nativeLangLabel)를 명시. 유형별 문자 제약(word=native, grammar/lyric=study), `자체 검증` 절 및 서버측 정규식 필터로 이중 검증.

**캐시 조회 SQL (참고)**:

```sql
SELECT content
FROM public.song_feature_cache
WHERE song_id = $1 AND feature_type = $2
LIMIT 1;
```

## ⑤ Acceptance & Output

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6)

기능/데이터:
- Discussion 성공 응답 `questions.length === 4` (실측: 서버는 `slice(0,5)` + `length<3` 이면 502; 정상 케이스는 4).
- Writing 성공 응답 `topics.length === 3`, level 배열은 정확히 `["easy","mid","hard"]` (서버가 index 로 재라벨).
- Quiz 성공 응답 각 문항 `options.length === 4`, `answer` 는 options 중 하나와 문자열 일치.
- Quiz word 유형: 옵션·정답에 `HANGUL_RE` (ko 모드) 또는 `HANZI_RE` (zh 모드) 매치 0건.
- Quiz grammar/lyric: 문제·옵션·정답에 study 반대 스크립트 매치 0건.
- 언어 스위치(ko↔zh) 후 캐시 히트 시 OpenAI 재요청 0회 (network 탭에서 `openai.com` 요청 없음).
- `force=true` 시 OpenAI 재요청 정확히 1회, 응답 후 `song_feature_cache` upsert 1회.

UI:
- 3 페이지 모두 4-상태(loading/error/empty/success) 커버, error 상태에서 `↺ 다시 시도` 클릭 시 재요청 1회.
- QuizSheet 인쇄(`window.print()`) 시 `.no-print` 요소가 실제로 렌더 트리에서 비가시(`visibility: hidden` 상속).
- Discussion contentEditable 편집 후 blur → state 반영 → 인쇄 새 창에도 편집본 반영.
- WritingTopic 예문 아코디언 초기 상태 닫힘, 클릭 시 short 탭이 기본 활성.
- 클립보드 API 실패 시 `toast.error` 정확히 1회.

성능/품질:
- 캐시 히트 응답 p95 ≤ 500 ms (edge cold start 제외).
- OpenAI 미스 응답 p95 ≤ 8 s.
- `grep -rE "bg-\[#(?!1a2540)]" src/components/songs/{DiscussionPage,WritingTopicPage,QuizSheetPage,ClassToolLangDialog}.tsx` 결과 0건 (허용된 시트 헤더 상수 제외).
- Lighthouse Accessibility ≥ 90 (편집 가능 요소에 대한 focus outline 확보).

### 5.2 Output Format

반환 순서(부수 텍스트·마크다운 헤더·사과·설명 금지, 각 파일은 그대로 저장 가능한 코드):

1. `supabase/functions/generate-discussion-questions/index.ts`
2. `supabase/functions/generate-writing-topics/index.ts`
3. `supabase/functions/generate-quiz-sheet/index.ts`
4. `src/components/songs/ClassToolLangDialog.tsx`
5. `src/components/songs/DiscussionPage.tsx`
6. `src/components/songs/WritingTopicPage.tsx`
7. `src/components/songs/QuizSheetPage.tsx`
8. 한국어 3줄 요약(작업 파일 수·핵심 변경·검증 방법).
