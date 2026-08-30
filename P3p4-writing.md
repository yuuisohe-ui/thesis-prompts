# P3p4 · AI 작문 챌린지 (Writing Challenge) 재현 프롬프트

> 본 문서는 `docs/thesis/appendix/prompts/00-template.md`가 정의한 5-Section 표준(Identity · Instructions · Examples · Context · Acceptance & Output)을 그대로 따른다. 이론적 근거(OpenAI 개발자 메시지 4-요소, Lovable 5 실천 원칙, IEEE 830 §4.3.6, Cohn 2004 §6)는 00-template.md에 정리되어 있으며 본 문서에서 재게시하지 않는다.
> **범위**: 곡 분석 다이얼로그(P3j) → 「탐구」 4-Level 네비게이션의 **L3 `l3-writing`** — AI 작문 챌린지 한 페이지. 소스: `src/components/songs/WritingChallengePage.tsx`(772행) + Edge Function `supabase/functions/writing-challenge-feedback/index.ts`. 다른 연습 모듈(퀴즈·받아쓰기·발음)은 P3p1·P3p2·P3p3에서 다룬다.

---

## ① Identity

당신은 한중 이중언어 교육 UX 라이터를 겸하는 시니어 프론트엔드 엔지니어이다. 기술 스택은 React 18 + Vite 5 + Tailwind CSS v3 + shadcn/ui + Supabase JS v2로 고정된다. 「멜로디 클래스」는 대한민국 대학의 중국어·한국어 학습자를 대상으로 한다. 본 페이지의 시스템 카피·안내 문구는 모두 한국어이며, AI 피드백 문구 언어만 `lang(cn|kr)`에 따라 교차 전환된다: `cn` 모드(중국어 작문)의 피드백은 한국어, `kr` 모드(한국어 작문)의 피드백은 중국어.

---

## ② Instructions

### 2.1 산출물 (Prompt by Component, Not Page)

정확히 아래 2개 파일만 생성/수정한다. 파일 트리 외의 파일은 절대 손대지 않는다.

1. `src/components/songs/WritingChallengePage.tsx` — 772행 단일 컴포넌트. 2-Phase 상태기(`select | write`) · 단어 풀 분류·배치 페치 · 5-단어 랜덤 픽 · 작문·AI 피드백 · HTML 결과 다운로드를 포함.
2. `supabase/functions/writing-challenge-feedback/index.ts` — `gpt-4o-mini` + Function Calling(`provide_writing_feedback`) 기반 피드백 Edge Function. `{score, comment, good, fix, better}` 5-필드 반환. CORS 필수, 429/402 한국어 메시지 매핑.

### 2.2 데이터 · 상태

- **Props**: `songId: string`, `songTitle: string`, `artistName: string`, `onBack: () => void`.
- **상수**: `PICK_COUNT = 5`(회당 뽑는 단어 수), `BATCH_LINES = 20`(재분석 시 한 번에 처리하는 가사 라인 수).
- **타입**:
  - `Lang = "cn" | "kr"`
  - `WordItem = { word, pinyin?, meaning_ko?, hsk_level?, topik_level?, example_sentence? }`
  - `Feedback = { score: 1~5, comment, good, fix, better }`
- **상태 변수**: `loading`, `cnPool: WordItem[]`, `krPool: WordItem[]`, `analyzing: Lang | null`, `step: "select" | "write"`, `lang: Lang`, `picked: WordItem[]`, `activeIdx: number | null`, `text: string`, `submitting: boolean`, `feedback: Feedback | null`, `usedWords: Record<Lang, Set<string>>`, `analysisOffset: Record<Lang, number>`, `hasMoreLyrics: Record<Lang, boolean>`.

### 2.3 단어 분류 (`classifyWord`)

- 정규식 `HANGUL_RE = /[\uAC00-\uD7AF\u1100-\u11FF\u3130-\u318F]/`, `HANZI_RE = /[\u4E00-\u9FFF\u3400-\u4DBF]/`.
- 단어에 한글이 있으면 `"kr"`, 한자가 있으면 `"cn"`. 문자로 판정 실패 시 `topik_level != null → "kr"`, `hsk_level != null → "cn"`. 그 외 `null`(어느 풀에도 담기지 않음).
- 마운트 시 `song_analyses.word_list`를 로드해 `normalizeWord` → `classifyWord`로 `cnPool` / `krPool` 분리.

### 2.4 부족한 단어 재추출 (`fetchNextBatch`)

- 조건: 해당 언어 풀에서 미사용 단어가 `PICK_COUNT` 미만이고 `hasMoreLyrics[l]`이 `true`일 때.
- `analyzing=l`로 표시하고 `toast({ title: "단어 새로 추출 중..." })`.
- Edge Function 호출: `supabase.functions.invoke("reanalyze-wordlist", { body: { song_id, lang: l==="cn"?"zh":"ko", level: 1, lines_offset, lines_limit: BATCH_LINES } })`.
- 응답에서 `words[]`, `next_offset`, `has_more` 추출. `classifyWord`로 언어 일치 필터.
- **DB 병합**: `song_analyses.word_list`를 다시 읽어 `word`로 중복 제거 후 새 단어 append, `update`로 저장.
- **로컬 풀 병합**: 상태 갱신 시 이미 있는 `word`는 제외.
- 실패 시 `toast({ title: "단어 분석 실패", variant: "destructive" })`. `finally`에서 `analyzing=null`.

### 2.5 5-단어 픽 (`pickUnusedWords`)

- `usedWords[l]`에 포함되지 않은 단어부터 후보. 부족하면 `fetchNextBatch`를 반복(최대 `hasMoreLyrics[l]===false` 될 때까지).
- 여전히 부족하면 이미 사용한 단어를 재활용하며 토스트: `"이 곡의 모든 단어를 추출했어요 / 단어가 반복될 수 있어요."`.
- 최종 후보가 0이면 `enterWrite`에서 `toast({ title: "단어 추출 실패", variant: "destructive" })` 후 `step` 유지.
- 채택된 단어들은 `usedWords[l]`에 추가.

### 2.6 화면 전환

- `enterWrite(l)`: `pickUnusedWords(l)` → 성공 시 `lang, picked, activeIdx=null, text="", feedback=null, step="write"` 설정.
- `redrawWords()`: 같은 언어에서 다시 5개 픽, `activeIdx=null`. (텍스트·피드백 유지)
- `retry()`: `feedback=null, text=""` → `redrawWords()` → `window.scrollTo({top:0, behavior:"smooth"})`.
- `select`로 복귀 시 (Back 버튼·상단 배지 클릭 모두): `step="select", feedback=null, text=""`.

### 2.7 원자적 UI 규칙

**Root**: `max-w-3xl mx-auto`.

**로딩**: `flex items-center justify-center min-h-[400px] text-muted-foreground` + `Loader2 w-5 h-5 animate-spin mr-2` + `"단어 불러오는 중..."`.

**뒤로가기 버튼**(select/write 공통 스타일):
- `inline-flex items-center gap-1.5 text-sm text-muted-foreground bg-card border border-border rounded-lg px-3 py-1.5 hover:bg-muted transition mb-5` + `ArrowLeft(3.5)`.
- select: 라벨 `"탐구"`, `onClick={onBack}`.
- write: 라벨 `"언어 선택"`, `onClick`은 `step="select"`로 복귀 + feedback·text 초기화.

**Song info**: `flex items-center gap-2.5 mb-6`, 제목 `text-[15px] font-bold`, 아티스트 `text-[13px] text-muted-foreground`. 제목 fallback `(제목 없음)`.

**Phase 1 · Select** (`bg-card rounded-2xl border border-border p-8 text-center`):
- 헤드라인 `어떤 언어로 작문할까요?` (text-lg font-bold mb-1.5).
- 서브라인 `이 노래로 중국어 또는 한국어 작문을 연습할 수 있어요` (text-[13px] text-muted-foreground mb-8).
- 그리드 `grid-cols-1 sm:grid-cols-2 gap-3.5`, `LangOption` 카드 2장.
- **LangOption 카드**: `border-2 rounded-2xl px-4 py-6 bg-card`, hover `border-primary bg-primary/5 -translate-y-0.5`, loading `opacity-70 cursor-wait`. flag `text-4xl`, name `text-base font-bold`, desc `text-xs text-muted-foreground leading-relaxed`(3줄 `<br/>`). 풀이 비어있으면 `hint`: `"단어장이 비어있어 가사로 새로 분석합니다"` (`text-[11px] text-muted-foreground/80 italic`). CTA 알약: 일반 `bg-primary/10 text-primary`, loading `bg-muted text-muted-foreground` + `Loader2 + "분석 중..."`; 일반 시 `"시작하기 →"`.
- 두 카드 내용:
  - CN: flag `🇨🇳`, name `중국어 작문`, desc `이 노래의 중국어 단어로 / 중국어 문장을 써요 / 피드백은 한국어로 받아요`.
  - KR: flag `🇰🇷`, name `한국어 작문`, desc `이 노래의 한국어 표현으로 / 한국어 문장을 써요 / 피드백은 중국어로 받아요`.

**Phase 2 · Write**:
- **언어 배지 버튼** (클릭 시 select 복귀): `inline-flex items-center gap-1.5 text-xs font-bold px-3 py-1 rounded-full border mb-4 hover:opacity-80`. `cn`: `bg-rose-50 text-rose-600 border-rose-200` + `🇨🇳 중국어 작문`. `kr`: `bg-indigo-50 text-indigo-700 border-indigo-200` + `🇰🇷 한국어 작문`.
- **Words 섹션 카드** `bg-card rounded-2xl border border-border p-5 mb-4`:
  - 헤더 좌: 도트 `w-1.5 h-1.5 rounded-full bg-primary` + `"이 노래의 단어로 작문해보세요"` (text-[13px] font-bold).
  - 헤더 우: `<Button variant="outline" size="sm" className="h-7 text-xs gap-1">` + `Shuffle(w-3 h-3) + "다시 뽑기"` → `redrawWords`.
  - 빈 경우: `"이 곡의 단어장에 단어가 없습니다."` (text-sm text-muted-foreground py-6 text-center).
  - 그리드 `grid-cols-2 sm:grid-cols-3 md:grid-cols-5 gap-2`.
  - 단어 카드: `border rounded-xl p-2.5 text-center bg-card`, active `border-primary bg-primary/5`, 비-active hover `border-primary/50 bg-muted/50`. 내부: 단어 `text-[15px] font-bold`, 레벨 배지(있을 때) `text-[10px] font-semibold px-1.5 py-0.5 rounded-full bg-indigo-50 text-indigo-700`, 하단 힌트 `text-[11px] text-muted-foreground`: `▼ 예문` / active 시 `▲ 닫기`.
  - **레벨 배지 로직**: `hsk_level != null → "HSK {n}"`, 아니면 `topik_level != null → "TOPIK {n}"`, 아니면 배지 없음.
  - **예문 패널** (activeIdx가 존재하고 example_sentence 또는 meaning_ko 있을 때만): `bg-muted/40 rounded-xl p-3 mt-3 border-l-[3px] border-primary`. 상단 라벨 `text-[11px] font-bold text-primary` + `BookOpen(w-3 h-3) + " 예문"`. 예문 `text-[13px] font-semibold leading-relaxed`. 뜻 `text-[12px] text-muted-foreground mt-1`.
- **Write 영역 카드** `bg-card rounded-2xl border border-border p-5 mb-4`:
  - 헤더 도트 + `"자유롭게 작문하세요"` (text-[13px] font-bold).
  - 안내 `text-xs text-muted-foreground mb-2.5`:
    - cn: `위 단어를 활용해 자유롭게 중국어 문장을 작성해보세요.`
    - kr: `위 단어를 활용해 자유롭게 한국어 문장을 작성해보세요.`
  - `<Textarea>` `min-h-[140px] text-[14px] leading-relaxed resize-y`. 플레이스홀더:
    - cn: `예) 我希望以后能去中国旅行...`
    - kr: `예) 저는 다음에 한국 여행을 가고 싶어요...`
  - 하단 좌: `{charCount}자` (text-xs text-muted-foreground). 하단 우: `<Button onClick={submit} disabled={submitting || !text.trim()}>` + (submitting ? `<Loader2 w-3.5>` : 없음) + `"AI 피드백 받기"`.

**Phase 2 · Feedback** (`feedback != null`인 경우 아래 블록을 write 영역 하단에 추가):
- 컨테이너 `id="writing-feedback" className="bg-card rounded-2xl border border-border p-5"`. `submit` 성공 후 50ms 뒤 `scrollIntoView({behavior:"smooth", block:"start"})`.
- 헤더: 좌 `AI 피드백` (text-[15px] font-bold), 우 두 개 버튼 `h-8 text-xs gap-1`:
  - `<Button variant="outline" size="sm" onClick={exportHTML}>` + `Download(w-3.5) + "저장"`.
  - `<Button size="sm" onClick={retry}>` + `RotateCcw(w-3.5) + "다시 도전"`.
- 내 글 블록 `bg-muted/40 rounded-xl p-3 mb-3 border-l-[3px] border-border`: 라벨 `text-[11px] font-bold text-muted-foreground` = `내가 쓴 글`, 본문 `text-[13px] whitespace-pre-wrap`.
- 점수 바 `bg-muted/40 rounded-xl px-4 py-3.5 mb-3 flex items-center gap-4`:
  - 좌: `<span class="text-[32px] font-bold text-primary leading-none">{score}</span>` + `<span class="text-muted-foreground text-base">/ 5점</span>`.
  - 우 상: 별 5개, `i < score ? "⭐" : "☆"` (text-lg leading-none).
  - 우 하: `comment` (text-[13px] text-muted-foreground mt-1.5).
- **FeedbackBlock 3장** (content 비어있으면 `return null`), `rounded-xl border px-4 py-3.5 mb-2.5`, 라벨 `text-[11px] font-bold tracking-wider mb-2`, 본문 `text-[13px] whitespace-pre-wrap`:
  - good: `bg-emerald-50 border-emerald-200`, 라벨 `text-emerald-700`, 텍스트 `✅ 잘 쓴 부분`, content=`feedback.good`.
  - fix: `bg-amber-50 border-amber-200`, 라벨 `text-amber-700`, 텍스트 `📝 문법·표현 교정`, content=`feedback.fix`.
  - better: `bg-indigo-50 border-indigo-200`, 라벨 `text-indigo-700`, 텍스트 `💡 더 자연스러운 표현`, content=`feedback.better`.

### 2.8 피드백 페치 (`submit`)

1. `text.trim()` 빈 문자열이면 `toast({ title: "작문 내용을 입력해주세요" })` 후 return.
2. `submitting=true; feedback=null`.
3. `supabase.functions.invoke("writing-challenge-feedback", { body: { lang, words: picked.map(p=>p.word), text: trimmed } })`.
4. 성공 시 `data.feedback` 5-필드 검증:
   - `score = clamp(Number(fb.score) || 0, 1, 5)`. 문자열 필드는 모두 `String(fb.x || "")`.
5. 실패 시 `toast({ title: "AI 피드백 실패", description: e.message, variant: "destructive" })`.
6. `finally { submitting = false }`.

### 2.9 HTML 결과 다운로드 (`exportHTML`)

- 파일명 고정: `작문챌린지_결과.html`.
- 헤더: `🎵 {songTitle} · {artistName} | (중국어 작문|한국어 작문) | {toLocaleDateString("ko-KR")}`.
- 렌더 순서: 헤더 → 제시 단어 pill 박스 → 내 글 박스(파란색 좌측 border `#1a3a6b`) → 점수 큰 숫자 → comment 박스 → good 초록 박스 → fix 주황 박스 → better 파란 박스.
- 인라인 스타일 시트 사용(외부 CSS/자산 참조 금지). 폰트 `Apple SD Gothic Neo, sans-serif`.
- `escapeHTML`로 `& < > " '` 5개 문자 이스케이프 — XSS 방지.
- `URL.createObjectURL(new Blob([html], {type:"text/html"}))` → `a.click()` → 1000ms 후 `revokeObjectURL`.

### 2.10 Edge Function (`writing-challenge-feedback`)

- **CORS 헤더**: `Access-Control-Allow-Origin: *` 및 `Access-Control-Allow-Headers`에 `authorization, x-client-info, apikey, content-type, x-supabase-client-platform, x-supabase-client-platform-version, x-supabase-client-runtime, x-supabase-client-runtime-version` 포함. `OPTIONS`에서 204 반환.
- **입력 검증**: `lang ∈ {"cn","kr"}` 아니면 400 `"lang must be 'cn' or 'kr'"`. `text`가 없거나 trim 후 빈 문자열이면 400 `"text is required"`.
- **시스템 프롬프트** (두 문자열 상수 그대로 유지):
  - `SYSTEM_CN` = `당신은 친절한 중국어 선생님입니다. 학생(한국인)이 중국어로 작문을 썼습니다. 반드시 한국어로 피드백을 주세요. 다음 JSON 형식으로만 답하세요(다른 텍스트 없이): {"score":1~5숫자, "comment":"한줄평가", "good":"잘쓴부분", "fix":"문법교정", "better":"더자연스러운표현"}`
  - `SYSTEM_KR` = `당신은 친절한 한국어 선생님입니다. 학생(중국인)이 한국어로 작문을 썼습니다. 반드시 중국어로 피드백을 주세요. 다음 JSON 형식으로만 답하세요(다른 텍스트 없이): {"score":1~5숫자, "comment":"一句话评价", "good":"写得好的地方", "fix":"语法纠正", "better":"更自然的表达"}`
- **User 메시지**: `제시 단어: {words.join(", ")}\n학생 작문: {text}`.
- **모델**: `gpt-4o-mini`, `tools=[provide_writing_feedback]`, `tool_choice=function`.
- **Tool Schema**:
  ```json
  {
    "type":"object",
    "properties":{
      "score":{"type":"integer","minimum":1,"maximum":5},
      "comment":{"type":"string"},
      "good":{"type":"string"},
      "fix":{"type":"string"},
      "better":{"type":"string"}
    },
    "required":["score","comment","good","fix","better"],
    "additionalProperties":false
  }
  ```
- **응답 파싱**: 우선 `choices[0].message.tool_calls[0].function.arguments`를 `JSON.parse`. 없으면 `choices[0].message.content`에서 백틱 제거 후 파싱.
- **파싱 실패**: 500 `"피드백 결과를 파싱할 수 없습니다."`.
- **에러 매핑**:
  - `response.status === 429` → 429 `"요청이 너무 많습니다. 잠시 후 다시 시도해주세요."`.
  - `response.status === 402` → 402 `"AI 크레딧이 부족합니다."`.
  - 그 외 게이트웨이 오류 → 500 `"AI 피드백 생성에 실패했습니다."`.
- **환경변수**: `OPENAI_API_KEY` 없으면 500. 인증 요구하지 않음(익명 호출 가능하도록 `verify_jwt = false` 등록).

---

## ③ Examples

### 3.1 학생 실행 예 · Happy Path (CN)

1. 학생이 "생일 축하 노래" 곡 분석 → 탐구 → 연습 → AI 작문 챌린지 진입.
2. `select` 화면에서 "🇨🇳 중국어 작문" 카드 클릭 → `cnPool.length ≥ 5`이므로 즉시 5개 카드 표시.
3. 학생이 첫 단어 카드 클릭 → `activeIdx=0`, 예문 패널 슬라이드 오픈.
4. Textarea에 `我今天很开心，因为朋友给我买了蛋糕。` 입력, 카운터 `20자`.
5. `AI 피드백 받기` 클릭 → Edge Function 호출 → 반환 예시:
   ```json
   { "feedback": { "score": 4, "comment": "자연스럽습니다.", "good": "'因为... '로 이유를 잘 설명했어요.", "fix": "'给我买了蛋糕' 대신 '给我买了一个蛋糕'가 더 자연스러워요.", "better": "我今天很开心，因为朋友给我买了一个大大的蛋糕。" } }
   ```
6. 결과 카드가 아래로 슬라이드, `writing-feedback`으로 스크롤. `저장` 클릭 시 `작문챌린지_결과.html` 다운로드.

### 3.2 단어장이 비어있는 경우

- 마운트 시 `word_list=[]` → `cnPool=krPool=[]`. LangOption에 `단어장이 비어있어 가사로 새로 분석합니다` 힌트.
- 클릭 시 `pickUnusedWords("cn")` → `fetchNextBatch("cn")` 반복 호출로 최대 `PICK_COUNT` 이상 확보.
- 가사가 아예 없으면 최종 후보 0 → 토스트 `"단어 추출 실패 / 이 곡 가사에서 해당 언어 단어를 찾지 못했어요."`, `step`은 `select` 유지.

### 3.3 재시도 반복으로 풀 고갈

- 세 번 `redrawWords` 후 `usedWords.cn` 크기가 `cnPool.length`에 가까워짐.
- 새 batch도 소진(`hasMoreLyrics.cn = false`)되면 토스트: `"이 곡의 모든 단어를 추출했어요 / 단어가 반복될 수 있어요."` 표시하고 이미 사용한 단어를 섞어 재활용.

### 3.4 잘못된 입력 (Edge Function 방어)

- `lang="jp"` → 400 `{ "error": "lang must be 'cn' or 'kr'" }`.
- `text=""` → 400 `{ "error": "text is required" }`.
- 429/402 → 위 매핑대로 한국어 메시지.

---

## ④ Context

### 4.1 부모 라우팅

- `WritingChallengePage`는 `ExploreTab` 4-Level 라우터에서 `case "l3-writing"`로 렌더링된다. `onBack={() => nav("l2-practice")}`.

### 4.2 데이터 원천

- `song_analyses` 테이블의 `word_list` JSONB(`WordItem[]`) 컬럼. 학생이 재분석을 트리거할 수 있으므로 `fetchNextBatch`는 read-then-write 병합으로 다른 클라이언트의 append를 덮어쓰지 않도록 세심하게 다룬다.
- `reanalyze-wordlist` Edge Function이 실제 가사 라인을 분석해 새 단어를 반환한다(별도 프롬프트).

### 4.3 디자인 토큰

- 색상은 shadcn/Tailwind 시맨틱 토큰(`bg-card`, `text-foreground`, `text-muted-foreground`, `border-border`, `text-primary`, `bg-primary/5|/10`)을 우선한다.
- 강조 팔레트(`rose`, `indigo`, `emerald`, `amber`)는 각각 CN 배지·KR 배지·good·fix에만 국한하여 사용한다. 다른 위치에서 하드코딩된 HEX를 사용하지 않는다.

### 4.4 접근성

- 뒤로가기 버튼과 언어 배지는 `<button>` 요소. 단어 카드는 키보드 포커스 가능해야 하며 예문 토글은 `aria-expanded`를 옵션으로 부여할 수 있다(현행 구현은 텍스트 힌트 `▲/▼`로 대체).
- 피드백 로딩 중 `Loader2 animate-spin`을 사용, 스크린리더 안내는 버튼 텍스트 `AI 피드백 받기`가 유지되어 disabled 상태로 시그널링된다.

### 4.5 성능·오류 격리

- 단어 재분석은 20 라인 단위 배치이므로 첫 호출이 무거워도 이후 스크롤형 확장으로 완만하다.
- Textarea·피드백 상태는 이 컴포넌트 내부에만 존재. 다이얼로그 밖 라우팅에 영향을 주지 않는다.
- HTML 다운로드는 클라이언트 사이드 Blob이므로 서버 부하 0. `escapeHTML`을 반드시 통과시켜 XSS 차단.

---

## ⑤ Acceptance & Output

### 5.1 Acceptance Criteria (하나라도 실패 시 재작업)

- [ ] 파일은 정확히 `src/components/songs/WritingChallengePage.tsx` 1개 + `supabase/functions/writing-challenge-feedback/index.ts` 1개만 생성/수정한다.
- [ ] `select` → `write` 전환은 `pickUnusedWords`가 성공한 후에만 이루어진다. 실패 시 destructive 토스트 후 `select` 유지.
- [ ] 단어 카드에 `HSK n` 또는 `TOPIK n` 배지가 정확히 표시되고, 두 값이 모두 없으면 배지 자체가 렌더되지 않는다.
- [ ] `redrawWords`는 미사용 단어가 부족할 때만 `fetchNextBatch`를 호출하며, `hasMoreLyrics[l]===false`가 되면 재활용 토스트를 표시한다.
- [ ] AI 피드백 요청 중 `submit` 버튼은 disabled + 스피너, `text`가 비어있어도 disabled.
- [ ] Feedback JSON은 5-필드(`score`, `comment`, `good`, `fix`, `better`) 모두 문자열/정수로 정규화된 뒤 렌더된다. `score`는 1~5 사이로 clamp.
- [ ] 별 렌더는 5개 고정, `⭐` 개수 = `score`, 나머지는 `☆`.
- [ ] `저장` 클릭 시 `작문챌린지_결과.html` 파일이 다운로드되고, 파일 안의 모든 사용자 입력은 `escapeHTML`을 거친다.
- [ ] Edge Function이 CN 모드에서 반드시 한국어로, KR 모드에서 반드시 중국어로 피드백을 반환한다(시스템 프롬프트 문구를 임의로 변경하지 않는다).
- [ ] 429는 429 상태 + 한국어 안내, 402는 402 상태 + 한국어 안내로 매핑한다.

### 5.2 Output Format (에이전트가 IDE에 커밋할 형태)

- `WritingChallengePage.tsx`: 함수 컴포넌트 1개(기본 export) + `LangOption`·`FeedbackBlock` 두 개 서브 컴포넌트. 외부 상태 저장소 사용 금지, `useCallback`으로 안정화된 핸들러 사용.
- `writing-challenge-feedback/index.ts`: Deno `serve` 진입점, 상수 프롬프트 2개, function-calling 응답 파싱, JSON fallback, 4가지 에러 상태(400·402·429·500).
- 자산·아이콘·색상은 본 문서에서 지정한 것만 사용. `lovable-tagger`나 여타 외부 자산 참조를 신설하지 않는다.
