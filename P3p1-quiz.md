# P3p1 · 탐구 · 연습 · 퀴즈 재현 프롬프트

> **본 프롬프트는 P3 시리즈의 16/17. 연습(l2-practice) 4모듈 중 첫 번째 — 퀴즈 전용.**
> **적용 대상**: `src/components/songs/QuizPage.tsx`, edge functions `quiz-word-explain`, `quiz-grammar-generate`.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**
> **연습 축은 총 4파일로 분리(P3p1=퀴즈 / P3p2=받아쓰기 / P3p3=발음 / P3p4=작문). 본 문서는 퀴즈만 다룬다.**

---

## 이론적 근거

1. **OpenAI. (n.d.).** https://platform.openai.com/docs/guides/prompt-engineering (Retrieved July 12, 2026)
2. **Lovable. (n.d.).** https://docs.lovable.dev/prompting/prompting-one (Retrieved July 12, 2026)
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity

당신은 곡별 학습 데이터(단어장·문형·가사)로부터 3종 객관식 퀴즈를 실시간 조립하는 시니어 프론트엔드 엔지니어이자 언어평가 전공자입니다. 문항 은행은 원본 데이터(단어·문형·가사)의 파생물이며, 오답 해설과 문법 문항 본문만 AI 로 보강합니다. 단일 컴포넌트 안에서 `start → game → final` 3-phase 상태 기계로 문항 단위를 관리합니다.

## ② Instructions

### 2.1 산출물

- `src/components/songs/QuizPage.tsx` — 3-phase 컴포넌트(시작/게임/결과), `initialType`(word/grammar/lyric)로 L4 진입 시 유형 프리셋.
- `supabase/functions/quiz-word-explain/index.ts` — 단어 상세 설명(오답 시 자동, 정답 시 온디맨드 버튼).
- `supabase/functions/quiz-grammar-generate/index.ts` — 문형 → 빈칸형 문항 1개 생성(엄격한 언어·길이 검증).

### 2.2 원자적 UI · 상태 규칙

**3-Phase 상태 기계** (`phase: "start" | "game" | "final"`):

1. **StartScreen** (`max-w-[600px] mx-auto`, 카드 `bg-white border border-[#e2e8f4] rounded-2xl p-6 shadow-sm`):
   - 상단: 뒤로 pill `← 탐구`.
   - 헤더: `📝` 이모지 + `퀴즈` 타이틀 + `곡명 — 아티스트` 서브.
   - 섹션 라벨 `학습 언어` (`text-[11px] font-bold text-[#8a9bbb] uppercase tracking-wider`).
   - `<LangCard>` 2 개 그리드: `中文`(strong `#e03131`, 서브 `한자·병음 학습`) / `한국어`(strong `#1971c2`, 서브 `한글 학습`). 선택 시 border 2px 컬러화.
   - 섹션 라벨 `퀴즈 유형` + `<TypeCard>` 그리드: `단어 퀴즈`(`📖`), `문법 퀴즈`(`📋`) 2열 + `가사 퀴즈`(`🎵`, `col-span-2`).
   - CTA 풀폭 버튼 `시작하기 →` (`bg-[#2563eb]`), 생성 중이면 `Loader2` + `퀴즈 생성 중...` + `disabled:opacity-60`.

2. **GameScreen**:
   - 상단바 3-slot: 뒤로 pill `← 퀴즈` / 중앙 `progress {current+1} / {total}` + `퀴즈 유형` 라벨 + 1.5px 진행 바(`bg-[#2563eb]`, `width` transition 500ms) / 우측 점수 chip `{score}점` (`bg-[#eff4ff]`, `text-[#2563eb]`).
   - 문항 카드: `key={current}` + `animate-in fade-in slide-in-from-bottom-2` 로 문항 전환 시 리마운트.
   - 상단 chip: `📖 단어 퀴즈` / `📋 문법 퀴즈` / `🎵 가사 퀴즈` (11px uppercase).
   - **word**: 큰 5xl `question` + 서브 `pinyin` + `questionType`(예: `한국어 뜻은?` / `중국어 단어는?` / `中文意思是？` / `韩语意思是？`).
   - **fill**: 프롬프트 `빈칸에 알맞은 말을 고르세요` + `<BlankRenderer>` (회색 `bg-[#f4f6fb]` 박스, 답변 후 정답 텍스트 노출) + 하단 회색 `translation`.
   - **lyric**: 프롬프트 `빈칸에 알맞은 가사를 고르세요` + `<BlankRenderer>` + `힌트: {반대언어 가사}`.
   - 선택지 리스트(세로 스택 `gap-2.5`): 각 옵션 `26px` 원형 번호 뱃지 `① ② ③ ④` + 텍스트, hover 시 파랑 하이라이트. 정답/오답 확정 후:
     - 선택 정답: `border-[#16a34a] bg-[#f0fdf4]` + 번호 뱃지 `bg-[#16a34a] text-white`.
     - 선택 오답: `border-[#dc2626] bg-[#fef2f2]`.
     - 미선택 정답도 초록으로 하이라이트, 나머지는 회색 `opacity-70`.
   - 답변 확정 후 결과 배너(정답=초록/오답=빨강): `✅ 정답!` 또는 `❌ 오답` + 해설.
     - **word 오답**: 자동으로 `quiz-word-explain` 호출 → 로딩 스피너 → 결과 표시.
     - **word 정답**: 배너 안에 `💡 상세 설명 보기` pill 버튼(온디맨드 호출).
     - **fill 오답**: `explanation` 즉시 표시.
     - **fill 정답**: `💡 상세 설명 보기` pill(캐시된 explanation 표시).
     - **lyric**: 정답/오답 모두 `explanation` 즉시 표시(로컬 생성 문자열이므로 API 없음).
   - 하단 CTA `다음 문제 →` (마지막 문항이면 `결과 보기 →`).

3. **FinalScreen**:
   - 상단 뒤로 pill `← 퀴즈`.
   - 히어로: 3-tier 아이콘(≥90% `🏆`, ≥70% `🥈`, ≥50% `🥉`, else `💪`) + `{score}` 5xl + `/ {totalScore}점`.
   - `<StatBox>` 3그리드: 정답(초록) / 오답(빨강) / 정답률(파랑).
   - `문제 리뷰` 리스트: ✅/❌ 아이콘 + 문제 텍스트(word=`question → ?`, fill=`sentence`, lyric=`text`) + 우측 정답 chip.
   - 하단 2버튼: `↺ 다시 풀기` (border 2px, `bg-[#eff4ff]`) / `홈으로 →` (풀 파랑).

### 2.3 문항 조립 규칙 (핵심 · 순수 프론트)

- 상수: `QUESTION_COUNT = 10`, `POINTS_PER_Q = 10`.
- 컴포넌트 마운트 시 `song_analyses` 에서 `word_list, sentence_patterns, lyrics_with_pinyin` 을 `.limit(1).maybeSingle()` 로 1회 로드.
- 단어 필터: `word_list` 는 `isHanzi(word)`+`isHangul(meaning_ko)` 통과 항목만 유지(50 % 이상 문자 비율 판정).
- `pickN` = 셔플 후 상위 N. `shuffle` = `[...arr].sort(() => Math.random() - 0.5)`.

**buildWordQuestions** (`wordList.length >= 4` 필요):
- 각 픽에 대해 `dirA = Math.random() > 0.5` 로 질문 방향 결정.
- `lang="zh"` + dirA: 한자 질문(pinyin 서브) → `한국어 뜻은?` (오답 3개는 `allHangul` 풀에서, `isHangul` 통과분).
- `lang="zh"` + !dirA: 한국어 질문 → `중국어 단어는?` (오답 3개는 `allHanzi` 풀).
- `lang="ko"` 는 questionType 만 `中文意思是？` / `韩语意思是？` 로 변경.
- 옵션 4 개 `shuffle([answer, ...3distractors])`.

**buildLyricQuestions** (`lyrics.length >= 1`):
- 각 라인에서 `wordList` 에 있는 단어가 라인에 등장하면 그 단어를 정답으로 선택, 없으면 폴백 토큰화(zh: 2글자 청크, ko: 공백 분리 후 길이≥2 토큰) 후 랜덤 픽.
- 오답 3 개는 `wordList` 에서 우선, 부족하면 `lyrics` 전체 토큰으로 패딩. 최종 3 개 미만이면 이 라인은 `null` 반환.
- 정답 자리를 `___` 로 치환하여 `text`, 반대 언어를 `hint`. explanation 은 로컬 문자열.

**buildGrammarQuestions** (`patterns.length >= 1`, 유일한 AI 호출 경로):
- 각 픽에 대해 캐시 키 `hashKey(lang|pattern|example)`.
- 캐시 조회: `song_feature_cache` 에서 `feature_type = "quiz_grammar_v3"` 로 `{ [key]: FillQuestion }` map 저장.
- 미스 시 `supabase.functions.invoke("quiz-grammar-generate", { songTitle, artistName, pattern, example, language: lang })`.
- 정상 응답 `{ sentence, answer, options[≥2], explanation, translation }` → `out` 에 push + `newlyGenerated[key]` 축적.
- 루프 종료 후 캐시가 존재하면 UPDATE, 아니면 INSERT (`feature_type: "quiz_grammar_v3"`).

**데이터 부족 폴백 흐름** (`startQuiz`):
- word 선택 + wordList<4 → toast "단어가 부족합니다 · 문법 퀴즈로 진행" → 자동 grammar 시도.
- grammar 선택 + patterns=0 → toast "문형이 없습니다 · 단어 퀴즈로 진행" → wordList<4 면 최종 실패 toast.
- lyric 결과 0 문항 → `가사 퀴즈를 만들 수 없습니다` destructive toast + 시작 취소.

### 2.4 답변 처리 · 해설 로딩

- `handleAnswer(option)`: 이미 답변한 경우 무시. 정답이면 `score += POINTS_PER_Q`, `correctCount++`. 오답이면 `wrongCount++`.
- **word 오답만** `loadWordExplanation` 호출(정답이면 explanation="" 로 두고 pill 버튼으로 온디맨드).
- **fill 오답만** `q.explanation` 즉시 세팅. 정답이면 pill 로 온디맨드.
- **lyric** 은 항상 로컬 explanation 표시.
- `loadWordExplanation`: `song_feature_cache.feature_type = "quiz_word_explain"` map 캐시(`wordKey = "{zh}|{ko}"`). 미스 시 `quiz-word-explain` 호출 → 캐시 upsert. 실패 시 로컬 폴백 `${zh}(${pinyin})는 "${ko}"를 의미합니다.`
- `showDetailExplanation` (정답 pill 클릭): 위와 동일한 캐시 경로, `results` 배열의 마지막 항목 `explanationOverride` 갱신.

### 2.5 강제 제약

- 문항 데이터 캐시는 `song_feature_cache`(`feature_type in {"quiz_grammar_v3", "quiz_word_explain"}`)에만 저장. `song_analyses.data.quizzes` 같은 경로 사용 금지.
- 학습자 응답·점수·기록은 DB 미저장. 세션 로컬만.
- `initialType` prop 은 L4 (`l4-word-quiz` / `l4-grammar-quiz` / `l4-lyrics-quiz`) 진입 시 유형만 프리셋하고, 사용자가 시작화면에서 변경 가능.
- 카드 컨테이너 폭 고정 `max-w-[600px] mx-auto`.
- 문항 카드 전환은 `key={current}` + `animate-in slide-in-from-bottom-2` 로 리마운트, 새 `<audio>` 등 생성 없음.

## ③ Examples

### 3.1 확정 카피 표

| 위치 | 카피 |
|---|---|
| 시작 화면 뒤로 pill | `← 탐구` |
| 게임/결과 뒤로 pill | `← 퀴즈` |
| 시작 CTA | `시작하기 →` / `퀴즈 생성 중...` |
| 학습 언어 카드 | `中文` · `한자·병음 학습` / `한국어` · `한글 학습` |
| 유형 카드 | `단어 퀴즈` · `단어↔뜻 매칭` / `문법 퀴즈` · `문형 빈칸 채우기` / `가사 퀴즈` · `가사 빈칸 채우기` |
| 진행 표시 | `{current+1} / {total}` |
| 점수 chip | `{score}점` |
| 유형 라벨 | `단어 퀴즈` / `문법 퀴즈` / `가사 퀴즈` |
| word 질문타입 (zh 모드) | `한국어 뜻은?` / `중국어 단어는?` |
| word 질문타입 (ko 모드) | `中文意思是？` / `韩语意思是？` |
| fill 프롬프트 | `빈칸에 알맞은 말을 고르세요` |
| lyric 프롬프트 | `빈칸에 알맞은 가사를 고르세요` |
| lyric 힌트 | `힌트: {반대언어}` |
| 결과 배너 | `✅ 정답!` / `❌ 오답` |
| 상세 설명 pill | `💡 상세 설명 보기` |
| 로딩 텍스트 | `설명을 불러오는 중...` |
| 다음 CTA | `다음 문제 →` / `결과 보기 →` |
| 결과 stat 라벨 | `정답` / `오답` / `정답률` |
| 결과 리뷰 헤더 | `문제 리뷰` |
| 결과 하단 버튼 | `↺ 다시 풀기` / `홈으로 →` |
| 폴백 toast | `단어가 부족합니다` / `문형이 없습니다` / `퀴즈 데이터 부족` / `문법 퀴즈 생성 실패` / `가사 퀴즈를 만들 수 없습니다` |
| lyric 로컬 해설 (zh) | `정답은 "{word}" 입니다. 가사에서 이 단어가 사용되었습니다.` |
| lyric 로컬 해설 (ko) | `正确答案是"{word}"。歌词中使用了这个词。` |
| word 로컬 폴백 해설 (zh) | `{zh}({pinyin})는 "{ko}"를 의미합니다.` |
| word 로컬 폴백 해설 (ko) | `"{ko}"의 중국어 표현은 {zh}({pinyin})입니다.` |

### 3.2 컴포넌트 트리

```text
<QuizPage>
  ├─ phase=start
  │    └─ <StartScreen>
  │          ├─ ← 탐구
  │          ├─ 헤더(📝 · 곡명 — 아티스트)
  │          ├─ <LangCard× 2>
  │          ├─ <TypeCard× 3>  (가사=col-span-2)
  │          └─ CTA "시작하기 →"
  ├─ phase=game
  │    └─ <GameScreen>
  │          ├─ 상단바(뒤로 + progress + 점수)
  │          ├─ 문항 카드(word/fill/lyric 3분기)
  │          │    ├─ (fill|lyric) <BlankRenderer>
  │          │    ├─ 옵션 4× (번호뱃지 + 텍스트)
  │          │    └─ 결과 배너 + "다음 문제 →"
  │          └─ 상세설명 pill (온디맨드)
  └─ phase=final
        └─ <FinalScreen>
              ├─ 히어로(icon + score/total)
              ├─ <StatBox× 3>
              ├─ 문제 리뷰 리스트
              └─ [↺ 다시 풀기] [홈으로 →]
```

### 3.3 edge function I/O 스키마

```ts
// quiz-word-explain — POST
Req  = { zh: string; ko: string; pinyin?: string; language: "zh" | "ko" }
Res  = { explanation: string; example: string } | { error: string }
// gpt-4o-mini · temperature 0.3 · response_format json_object.

// quiz-grammar-generate — POST
Req  = { songTitle: string; artistName: string; pattern: string; example: string; language: "zh"|"ko" }
Res  = { sentence: string; answer: string; options: string[4]; translation: string; explanation: string }
      | { error: string }
// 검증 5중: (a) 학습언어(hangul/hanzi ≥2), (b) 정확히 1개 ___ 마커,
//          (c) 정답 길이(ko<12자·종결어미 금지 / zh<8자),
//          (d) 오답 길이 ±60% 이내(min tol 2),
//          (e) options 4개 셔플·중복 제거.
```

## ④ Context

### 4.1 프로젝트 맥락
연습 축은 다른 모듈(정보·가사·수업 도구)이 "이해" 를 목표로 하는 것과 달리 "재생산" 을 요구한다. 퀴즈는 그중 가장 낮은 인지 부담(재인) 층으로, 곡 데이터에서 파생된 문항만 사용해 학습자가 이미 학습한 범위 안에서 즉시 자기 검증할 수 있게 한다. 문항 은행은 원본 데이터 파생물이므로 별도 저장하지 않고, AI 로 생성되는 부분(문법 문항 본문·오답 해설)만 캐시한다.

### 4.2 Lovable Cloud 후경

- edge functions:
  - `quiz-word-explain` — 단어 오답 해설(품사·용법·예문). gpt-4o-mini · temperature 0.3 · JSON only.
  - `quiz-grammar-generate` — 문형 → 빈칸 채우기 문항 1개. 매우 엄격한 자체 검증(언어 순도, 종결어미 금지, 선택지 길이 균일).
- 429/402 → 한국어 표준 문구(`AI 호출이 일시적으로 제한되었습니다.` / `AI 사용 한도가 소진되었습니다.`).
- 학습자 응답·점수는 DB 미저장. 캐시는 곡 단위·언어 별로만 재사용.

### 4.3 데이터 계약

**입력(로컬)**: `song_analyses`

```ts
word_list: { word: string; pinyin?: string; meaning_ko: string }[]
sentence_patterns: { pattern: string; examples?: string[] }[]
lyrics_with_pinyin: { chinese: string; pinyin?: string; korean: string }[]
```

**AI 캐시(공유)**: `song_feature_cache(song_id, feature_type, content)`

- `feature_type = "quiz_grammar_v3"` · `content: { [hashKey]: FillQuestion }`
- `feature_type = "quiz_word_explain"` · `content: { ["zh|ko"]: { explanation, example } }`

**세션(휘발성)**: `Question[]`, `ResultItem[]`, `current`, `score`, `correctCount`, `wrongCount`, `answered`, `selectedAnswer`, `activeExplanation`.

## ⑤ Acceptance & Output

### 5.1 Acceptance Criteria

- `QUESTION_COUNT` 정확히 10, `POINTS_PER_Q` 정확히 10, `totalScore = questions.length × POINTS_PER_Q`.
- 3-tier 아이콘 임계값: ≥90 `🏆`, ≥70 `🥈`, ≥50 `🥉`, else `💪`.
- word 문항 옵션 4개는 언어별 분리 풀(`isHanzi` / `isHangul`)만으로 구성, 혼종 옵션 0.
- fill 문항의 `sentence` 는 `___` 마커 정확히 1개, 정답 길이 ko<12자 & 종결어미(다/요/까/니다) 금지, zh<8자.
- lyric 문항의 정답 단어는 `text.includes(answer)` 필수, `___` 치환 후 원문 완전 복원 가능.
- word 정답/오답 처리에서 `loadWordExplanation` 호출은 **오답에서만 자동**, 정답은 pill 클릭 시에만.
- `song_feature_cache` upsert 는 루프 종료 후 1회만 발생(문항별 개별 INSERT 금지).
- 초기 `loading` → 데이터 fetch 완료 전까지 스피너 표시, `wordList/patterns/lyrics` state 는 필터링 후 값이 반영됨.
- StartScreen 카드 폭 정확히 `max-w-[600px]`, 문항 카드 전환 시 `key={current}` 리마운트로 `slide-in-from-bottom-2` 애니메이션 트리거.

### 5.2 Output Format

반환 순서:
1. `supabase/functions/quiz-word-explain/index.ts`
2. `supabase/functions/quiz-grammar-generate/index.ts`
3. `src/components/songs/QuizPage.tsx`
4. 한국어 3줄 요약(파일 수·핵심 상태 기계·캐시 키 규약).
