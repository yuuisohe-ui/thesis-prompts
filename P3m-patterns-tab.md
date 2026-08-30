# P3m · 문형(Patterns) 탭 재현 프롬프트

> **본 프롬프트는 P3 시리즈의 13/17.**
> **적용 대상**: `src/components/songs/PatternsTab.tsx`, `src/components/songs/LyricCardDialog.tsx`(pattern 모드 재사용), edge functions `supabase/functions/evaluate-sentence/index.ts`, `supabase/functions/regenerate-patterns/index.ts`.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* https://platform.openai.com/docs/guides/prompt-engineering (Retrieved July 12, 2026)
2. **Lovable. (n.d.).** *Prompting best practices.* https://docs.lovable.dev/prompting/prompting-one (Retrieved July 12, 2026)
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity

당신은 중국어 문형 학습(구문·연결어·강조·시태) UI 를 설계하는 시니어 프론트엔드 엔지니어이자 대학 초·중급 중국어 교재 편집자입니다. 학습 루프는 다음 5-스텝이다: (1) 문형 표시 → (2) 한국어 상세 설명 → (3) 중/한 병렬 예문 → (4) 학습자 따라 쓰기 입력 → (5) AI 채점(0–100점). 한 곡당 문형은 3–8개(백엔드 시스템 프롬프트가 강제하는 범위)로 렌더한다.

## ② Instructions

### 2.1 산출물

- `src/components/songs/PatternsTab.tsx` — 문형 카드 목록, 인라인 편집, 카드 저장 선택 모드, 따라 쓰기 채점.
- `supabase/functions/evaluate-sentence/index.ts` — 학습자 문장을 `{numeric_score, score, strengths, improvements, corrected}` 5필드로 반환하는 단일-곡 채점 엔드포인트.
- `supabase/functions/regenerate-patterns/index.ts` — 관리자용 **배치 재분석 엔드포인트**(프론트에서 직접 호출하는 UI 트리거는 존재하지 않는다). `{song_id}` 또는 `{batch: true}` 로 호출.

### 2.2 원자적 UI 규칙

**Props**: `{ patterns: SentencePattern[]; onSave: (patterns) => void; readOnly?: boolean; songTitle?: string; songArtist?: string|null; songHskLevel?: string|null; }`.

**상단 툴바** (`flex items-center gap-2 flex-wrap`):
- 왼쪽: `📤 카드로 저장` 토글 버튼. 활성 시 라벨 `✕ 취소`, 클래스 `bg-red-500 text-white animate-pulse`. 비활성 시 `bg-[#3d6cb5] text-white hover:bg-[#2e3d6b]`(브랜드 상수, semantic 예외로 허용).
- 오른쪽(`ml-auto`, `!readOnly && !selectMode` 조건): 편집 아닐 때 `수정`(outline, `Pencil` 아이콘). 편집 중 `취소`(ghost, `X`) + `저장`(solid, `Save`) 2버튼.

**Select 배너**(selectMode 진입 시): `bg-[#3d6cb5]/10 border border-[#3d6cb5]/40 rounded-lg px-4 py-2.5`, 좌측 `문형을 클릭해서 선택하세요 · <strong>N</strong>개 선택됨`, 우측 `카드 만들기 →`(`disabled`=선택 0개) + `취소`(ghost).

**PatternCard**(`<Card>` × N, 3 ≤ N ≤ 8):
- selectMode 이면 카드 전체 클릭 가능(`cursor-pointer hover:bg-muted/50`), 내부 버튼은 `[&_button]:pointer-events-none` 로 무력화. 선택 시 `ring-2 ring-[#3d6cb5] bg-[#3d6cb5]/5`.
- **CardHeader**(pb-2): `pattern` (편집 모드 `<Input h-8>`, 아닐 때 `text-base flex-1`) + 우측 `따라 쓰기` 토글(`PenLine` 아이콘, 활성 시 variant `secondary`, 비활성 `outline`). 편집 중에는 우측 버튼 렌더 금지.
- **CardContent**(space-y-3):
  - `explanation` — 편집 모드 `<Textarea rows={2}>`, 아닐 때 `<p class="text-sm text-muted-foreground">`.
  - **Examples 렌더**: `p.examples.flatMap(ex => ex.includes('\n') ? ex.split('\n').filter(Boolean) : [ex])` 로 legacy `\n` 항목 flatten 후, 각 라인을 `trim()` 하여:
    - `isKoreanLine(trim)` (한글 존재 && 한자 부재) → `text-xs text-muted-foreground pl-3 border-l-2 border-muted/40 -mt-1 pb-0.5` 로 한국어 번역 라인.
    - 그 외 → 중국어 예문 `<button>` 행: `border-l-2 border-primary/30 hover:bg-muted/50 rounded-r px-2 py-1 group w-full text-left`. 클릭 시 `speakChinese(trimmed)` (SpeechSynthesis, `zh-CN`, rate 0.85). hover 시 우측 `Volume2 h-3 w-3` opacity 0→60.
  - 편집 모드에서는 예문을 `<Input h-8>` 로 순차 렌더(줄바꿈 flatten 미적용, 원본 문자열 유지).
- **따라 쓰기 확장**(`isWriting && !editing`):
  - `flex gap-2`: `<Input placeholder="이 문형을 사용하여 중국어 문장을 만들어 보세요...">` (Enter 키로 제출, `disabled={evaluating}`) + `평가하기` 버튼. 로딩 중 `<Loader2 animate-spin />평가 중`.
  - **현재 피드백 카드** (feedback 존재 시): tier 3-색 강제 —
    - `numeric_score ≥ 80` → `bg-emerald-500/10 border-emerald-500/20`, 점수 `text-emerald-600`.
    - `≥ 60` → `bg-amber-500/10 border-amber-500/20`, `text-amber-600`.
    - 그 외 → `bg-red-500/10 border-red-500/20`, `text-red-600`.
    - 내부: `{numeric_score}점` (text-2xl font-bold) + `{score}` 문자열 라벨 + `교정: <button onClick={speakChinese(corrected)}>{corrected}<Volume2 inline/></button>` (원문과 다를 때만) + `👍 잘한 점: {strengths}` + `💡 개선할 점: {improvements}`.
  - **이전 피드백 스택**: `Object.entries(feedbacks)` 를 순회하여 `k.startsWith('${pattern}-') && k !== currentKey` 인 것들을 `opacity-70 bg-muted/20` 로 나열. 각 항목은 `이전 문장: {키의 pattern 뒤 부분}` + `{fb.score}` + (`fb.corrected` 있으면) `교정: {fb.corrected}` 만 노출(strengths/improvements 생략).

**LyricCardDialog 연동**: `mode="pattern"`, `selectedPatterns: CardPatternItem[]` (`{ pattern, explanation, examples }`), `songTitle`/`artist`/`hskLevel` 전달. `onOpenChange(false)` 시 `cancelSelectMode()` 호출.

### 2.3 강제 제약

- **하드코드 색 규칙**: `#3d6cb5`/`#2e3d6b` (브랜드 primary) 및 tier 3색(`emerald/amber/red-500`) 는 예외. 그 외 임의 `bg-[#...]` 금지.
- `speechSynthesis.cancel()` 를 매 TTS 호출 전 실행(중첩 재생 차단).
- `evaluate-sentence` 응답이 `{feedback}` 이 아니면 `평가 실패`(server 에러 문구) 또는 `평가 오류`(네트워크) toast, 폼 값은 유지.
- 편집 모드에서는 `따라 쓰기` 버튼과 확장 UI 를 렌더하지 않는다(상태 오염 방지).
- `feedbacks` 는 `${pattern}-${userSentence}` 키의 flat map — 다른 pattern 의 히스토리는 절대 노출하지 않는다.
- 프론트에서 `regenerate-patterns` 를 호출하는 UI 는 만들지 않는다(관리자 도구 전용).

## ③ Examples

### 3.1 확정 카피 표

| 위치 | 카피 |
|---|---|
| 카드 저장 진입 | `📤 카드로 저장` |
| 카드 저장 취소 | `✕ 취소` |
| Select 배너 | `문형을 클릭해서 선택하세요 · N개 선택됨` |
| 카드 생성 CTA | `카드 만들기 →` |
| Select 배너 취소 | `취소` |
| 편집 진입 | `수정` |
| 편집 저장 | `저장` |
| 편집 취소 | `취소` |
| 따라 쓰기 토글 | `따라 쓰기` |
| Input placeholder | `이 문형을 사용하여 중국어 문장을 만들어 보세요...` |
| 평가 버튼 idle | `평가하기` |
| 평가 버튼 loading | `평가 중` |
| 점수 접미 | `점` (예: `85점`) |
| 교정 프리픽스 | `교정: ` |
| 강점 프리픽스 | `👍 잘한 점: ` |
| 개선점 프리픽스 | `💡 개선할 점: ` |
| 이전 문장 프리픽스 | `이전 문장: ` |
| Toast 서버 오류 | `평가 실패` / `{server error message}` |
| Toast 네트워크 오류 | `평가 오류` / `{err.message}` |

### 3.2 컴포넌트 트리

```text
<PatternsTab>
  ├─ Toolbar: [📤 카드로 저장 | ✕ 취소]  ·  (수정 · 저장 · 취소)
  ├─ SelectBanner (selectMode)
  ├─ <Card> × N  (3..8)
  │    ├─ Header: pattern · [따라 쓰기]
  │    └─ Content
  │         ├─ explanation  (또는 Textarea)
  │         ├─ ExampleRows (KR-line vs ZH-button, TTS)
  │         └─ WritingArea (isWriting && !editing)
  │              ├─ Input + [평가하기]
  │              ├─ CurrentFeedback  (tier 색상)
  │              └─ PrevFeedback × M  (opacity-70)
  └─ <LyricCardDialog mode="pattern"/>
```

## ④ Context

### 4.1 프로젝트 맥락
문형 탭은 `SongAnalysisDialog` 의 6-Tabs (`영상 / 가사 / 단어장 / 문법·표현 / 읽기 / 탐구`) 중 `문법·표현` 슬롯에 마운트된다. 어휘 탭(P3l)과 대칭 구조지만 학습 단위가 "문형" 이므로 카드 수는 3–8개 범위이며(백엔드 시스템 프롬프트의 요구사항), UI 상 상·하한 강제 검증은 렌더 시점에 하지 않고 백엔드 응답을 그대로 신뢰한다. `regenerate-patterns` 는 배치 관리자 도구로만 존재하고, 사용자용 트리거 UI 는 존재하지 않는다.

### 4.2 Lovable Cloud 후경

- **4-상태**: 로딩(부모 다이얼로그가 관리, 본 탭은 배열이 이미 로드된 후 마운트) / 빈(`patterns.length === 0` 이면 카드 0개 → 빈 컨테이너, 별도 empty state 문구 없음) / 에러(toast) / 성공.
- **`evaluate-sentence` 계약**:
  - Endpoint: `POST /functions/v1/evaluate-sentence`.
  - 입력: `{ pattern: string; explanation: string; examples: string[]; user_sentence: string }`.
  - 백엔드 시스템 프롬프트는 중국어로 작성되나 학생 대상 필드(`score`, `strengths`, `improvements`)는 **한국어**로 반환. `numeric_score` 는 0–100 정수, `corrected` 는 교정된 중국어 문장(맞으면 원문 그대로).
  - 모델: OpenAI-호환 endpoint 를 통해 `google/gemini-3-flash-preview` 호출, `OPENAI_API_KEY` 사용.
  - HTTP 상태: 200 `{feedback}`, 400 `{error:"Missing sentence or pattern"}`, 429 `{error:"요청이 너무 많습니다. 잠시 후 다시 시도해주세요."}`, 402 `{error:"AI 크레딧이 부족합니다."}`, 500 `{error:"AI 평가에 실패했습니다."}` / `{error:"평가 결과를 파싱할 수 없습니다."}`.
- **`regenerate-patterns` 계약**:
  - Endpoint: `POST /functions/v1/regenerate-patterns`.
  - 입력: `{ song_id?: string; batch?: boolean }`. `batch=true` 면 `language='chinese'` AND `lyrics_raw NOT NULL` AND `song_analyses` 행 존재하는 모든 곡을 순차 처리.
  - 시스템 프롬프트가 3–8개 문형을 강제하고, `pattern`(한국어 문법 용어, 예: `把자문`, `是...的 강조구문`, `겸어문`), `explanation`(3–5문장 한국어, 의미·한국어와 차이·자주 하는 실수·팁 포함), `examples`(중국어와 한국어가 교대 배치된 배열)의 3필드 JSON 배열을 반환한다.
  - 모델: `gpt-4o-mini`, `temperature: 0.3`, `response_format: json_object`.
  - 응답: `{ processed: number, results: [{song_id, title, status: 'ok'|'skipped'|'error', count?, reason?, error?}] }`.

### 4.3 데이터 계약

`song_analyses.sentence_patterns` 컬럼(jsonb): `SentencePattern[]` = `{ pattern: string; explanation: string; examples: string[] }[]`. 저장 흐름은 `<PatternsTab onSave={patterns => parent.updateAnalysis({sentence_patterns: patterns})} />`. RLS·GRANT 는 P3j / P3h 에서 정의한 `song_analyses` 정책을 그대로 재사용한다(본 프롬프트는 DDL 재선언 없음).

## ⑤ Acceptance & Output

### 5.1 Acceptance Criteria

- `p.examples` 배열 내부에 `"a\nb"` legacy 항목이 섞여도 렌더 결과의 예문 라인 수는 `flatten(examples).filter(Boolean).length` 와 정확히 일치.
- 한글만/혼합/한자만 3-케이스 fixture 에 대해 KR 라인 자동 구분 정확도 100 %(`isKoreanLine` 회귀 테스트).
- `evaluate-sentence` 왕복 p95 ≤ 4500 ms (edge function 로그 기반, `numeric_score` 렌더까지의 총 지연 포함).
- 동일 pattern 의 이전 채점 히스토리는 최신 아래(현재 카드 하단)로 스택되며 `opacity-70` 클래스가 DOM 에 실제로 존재(`document.querySelectorAll('.opacity-70').length === M`).
- selectMode 진입 후 카드를 클릭했을 때 카드 내부 `<button>` 의 click 이벤트가 실행되지 않는다(pointer-events-none 검증, `getComputedStyle` 확인).
- `grep -nE "bg-\[#(?!3d6cb5|2e3d6b)" src/components/songs/PatternsTab.tsx` 매칭 = 0(브랜드 상수 2건만 예외 허용).
- 편집 모드에서 `따라 쓰기` 토글 및 확장 UI 의 DOM 노드 수 = 0.
- `evaluate-sentence` 500/429/402 응답 시 폼 `userSentence` 값이 유지된다(스냅샷 회귀 테스트).

### 5.2 Output Format

반환 순서:
1. `supabase/functions/evaluate-sentence/index.ts`
2. `supabase/functions/regenerate-patterns/index.ts`
3. `src/components/songs/PatternsTab.tsx`
4. 한국어 3줄 요약(그 외 사과·주석·마크다운 헤더 금지).
