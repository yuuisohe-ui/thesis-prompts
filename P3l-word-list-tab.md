# P3l · 단어(WordList) 탭 재현 프롬프트

> **본 프롬프트는 P3 시리즈(P3a–P3q)의 12/17.**
> **적용 대상**: `src/components/songs/WordListTab.tsx`, `src/components/songs/StrokeOrderPanel.tsx`, `src/components/songs/RelatedWordsPanel.tsx`, `src/components/songs/songTagUtils.ts`, `src/hooks/useSpeechEvaluate.ts`, `supabase/functions/reanalyze-wordlist/index.ts`. `LyricCardDialog`(단어 모드) 는 P3k 정의를 재사용.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* Retrieved July 12, 2026, from https://platform.openai.com/docs/guides/prompt-engineering
2. **Lovable. (n.d.).** *Prompting best practices.* Retrieved July 12, 2026, from https://docs.lovable.dev/prompting/prompting-one — 본 부록의 5개 실천 원칙 채택 근거.
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity

당신은 한중 이중언어 어휘 학습 UI 를 담당하는 시니어 프론트엔드 엔지니어이자 CEFR/HSK/TOPIK 교재 편집 경험이 있는 UX 라이터입니다. Web Speech API, HanziWriter, Supabase Edge Function 을 조합해 한 카드 안에서 "듣기 → 획순 → 발음 채점 → 연관어 확장 → 카드 저장" 학습 루프를 완결하며, 재분석/편집/선택 세 가지 모드가 서로 간섭하지 않도록 상태를 격리합니다.

## ② Instructions

### 2.1 산출물 (Lovable 실천 원칙 "Prompt by Component, Not Page")

- `src/components/songs/WordListTab.tsx` — 2 행 툴바 · 필터 · 재분석 · 편집 · 선택→카드 · 획순/연관어 토글 · 발음 평가 결과 뷰.
- `src/components/songs/StrokeOrderPanel.tsx` — HanziWriter 기반 획순 애니메이션 패널(88×88, 22px 그리드).
- `src/components/songs/RelatedWordsPanel.tsx` — **세로 3 섹션(유의어/반의어/자주 쓰는 표현)** 확장 패널. Grid 아님.
- `src/components/songs/songTagUtils.ts` — `buildTagsFromWordList(words, mode)` : **상위 3 개 급수를 `"HSK 5 (12)"` 형식으로 반환**(카운트 내림차순, top-3).
- `src/hooks/useSpeechEvaluate.ts` — MediaRecorder(webm) → AudioContext 16 kHz 리샘플 → Int16 PCM → base64 → `speech-evaluate` 호출. **피드백 텍스트는 프런트에서 `generateFeedback()` 로 조립**.
- `supabase/functions/reanalyze-wordlist/index.ts` — `song_id/lang/level` 필수, 선택적 `lines_offset/lines_limit` 로 **배치 모드**. 청크 10 줄 × 동시 3 개 처리, `word` 기준 중복 제거.

### 2.2 원자적 UI 규칙 (Lovable 실천 원칙 "Speak Atomic")

**툴바 — 2 행 구조**:

**1 행 (`flex items-center justify-between`)**:
- 좌: `{filtered.length}개 단어` (text-sm font-medium).
- 우(readOnly 아닐 때 4 개 버튼, 순서 고정): `수정` ↔ `완료` (Pencil, editMode 시 red-50 배경) → `단어장 재분석` (RefreshCw, Popover) → `저장` ↔ `저장됨` (Save/Check, 성공 시 2 초 emerald 상태) → `📤 카드로 저장` ↔ `✕ 취소` (selectMode 시 `bg-red-500 animate-pulse`, 아닐 때 `bg-[#3d6cb5]`).

**2 행 (`flex flex-wrap`, level filter + font)**:
- 좌: `{levelLabel}` 라벨(HSK 또는 TOPIK) + `전체` 칩 + 급수 칩 `{n}급`. 활성 = `bg-[hsl(220,40%,23%)] text-white`. `hskLevels = 1..9`, `topikLevels = 1..6`.
- 우(`ml-auto`): `글꼴` Popover(AArrowUp) → `Slider min=11 max=22 step=1`. 라벨 `글꼴 크기: {n}px`.

**HSK/TOPIK 스위처는 툴바가 아니라 "단어장 재분석" Popover 내부** 의 "학습 언어" 두 pill(`중국어` / `한국어`) 로만 노출. 초기값: `words.some(w => w.topik_level != null) ? "ko" : "zh"`.

**단어장 재분석 Popover** (`w-80 p-4`, `align="end"`):
1. 제목 `단어장 재분석`.
2. `학습 언어` — `중국어` / `한국어` pill(활성 시 짙은 남색 배경). 선택 시 `reanaLevel` null 리셋.
3. `현재 나의 수준` — 언어에 맞춰 1..9(HSK) 또는 1..6(TOPIK) 칩. `{levelLabel} {n}` 표기.
4. 하단 힌트 `선택한 등급 이상의 단어가 표시됩니다`.
5. CTA `이 설정으로 재분석` — `reanaLevel === null || reanalyzing` 이면 disabled, 진행 중 `<Loader2 spin/> 재분석 중...`.

**배너**:
- 재분석 성공 시(`showBanner && isReanalyzed`): amber 배경, 카피 `⚠ 재분석 결과입니다. 저장하지 않으면 원본으로 돌아갑니다.` + 인라인 `원본으로 복귀` 링크 + `✕` 닫기.
- 선택 모드 시(`selectMode`): `bg-[#3d6cb5]/10`, `단어를 클릭해서 선택하세요 · {n}개 선택됨` + `카드 만들기 →` (`selectedSet.size===0` 이면 disabled) + `취소`.

**단어 행** (좌측에 조건부 삭제 버튼 + 카드 본체):
- **삭제 버튼**: `editMode && !readOnly && !selectMode` 일 때만 카드 좌측에 원형 `×` (`w-7 h-7 rounded-full border-red-300`). 클릭 시 로컬 상태에서 제거 후 자동 `isReanalyzed=true` 로 임시태 진입.
- **카드 본체** (`rounded-xl border bg-card`, hover shadow):
  - `w.fromRelated=true` → 왼쪽 3px `border-l-[hsl(220,40%,23%)]` 액센트.
  - `editMode && !selectMode` → 카드 테두리 `border-red-200`.
  - `selectMode` → `cursor-pointer hover:bg-muted/50`, 내부 버튼 이벤트 차단(`[&_button]:pointer-events-none`).
  - `isSelected` → `ring-2 ring-[#3d6cb5] bg-[#3d6cb5]/5`.
- **메인 라인**(p-4, flex-wrap):
  - 단어(`font-bold min-w-[60px]`) — 크기 = `fontSize + 10`.
  - 병음(zh 만) — 크기 = `fontSize`.
  - `meaning_ko` — 크기 = `fontSize`.
  - `flex-1` 스페이서.
  - `fromRelated` 배지 `연관어` (`#e8ecf5` 배경).
  - 급수 배지: 등급 0 이면 `—`, 있으면 `{levelLabel} {n}` — 색상은 `hskBadgeColor` 또는 `topikBadgeColor`(1–2 emerald, 3–4 amber, 5 purple, 6–9 red).
  - `✍ 획순` 버튼(zh 만) — 열림 시 amber-500/10 배경.
  - `✦ 연관어` 버튼 — `!w.fromRelated` 일 때만. 열림 시 짙은 남색 배경 + 흰색 텍스트.
  - TTS 아이콘(`Volume2 h-7 w-7 ghost`).
  - Mic 아이콘 — 녹음 중 `variant=destructive` + `MicOff`, 평가 중 `Loader2 spin` + disabled, 그 외 `Mic`. 다른 단어 녹음 중이면 이 버튼도 disabled.
- **예문**: `w.example_sentence` 존재 시 별도 라인, 클릭하면 TTS 재생. 크기 = `fontSize`.
- **획순 패널**: `isStrokeOpen` 시 `<StrokeOrderPanel word={w.word}/>` 인라인 렌더.
- **연관어 패널**: `isRelatedOpen && !fromRelated` 시 `<RelatedWordsPanel key={w.word} .../>` 인라인 렌더.
- **평가 결과**(`results[w.word]` 존재 시): `mx-4 mb-3 rounded-lg border`, 배경/문자색 = `scoreBg/scoreColor` (≥80 emerald, ≥60 amber, else red). 표시: `{total_score.toFixed(0)}점` + `발음 {p}` + `유창성 {f}` (f>0 만) + 하단 `feedback` 2 줄.

**RelatedWordsPanel — 세로 3 섹션 구조**:
- 컨테이너: `border-t border-border bg-[#fafbfd] p-3.5 animate-in fade-in slide-in-from-top-1`.
- 로딩: 3 개 5×5 점 `bg-[hsl(220,40%,23%)]/30 animate-pulse` (delay 0/0.2/0.4s) + `연관어 불러오는 중...`.
- 에러: `연관어를 불러올 수 없습니다.` + `toast.error("연관어 불러오기 실패")`.
- 성공: `syn / ant / col` 순회. 섹션 라벨 = `유의어 / 반의어 / **자주 쓰는 표현**` (`col` 은 "연어" 가 아니라 "자주 쓰는 표현").
- 각 항목 라인(`flex items-center py-1.5 border-b border-[#f0f2f8]`): 단어(font-bold min-w-[56px]) + (zh 만) 병음 `#3d6cb5 min-w-[86px]` + 뜻(flex-1 muted) + speak 버튼 + add 버튼(6×6 `rounded-[5px]`).
- add 버튼 3 태: 기본 `+`, 로딩 `Loader2 spin` (disabled), 이미 추가됨 `✓` (`bg-emerald-50 border-emerald-600 cursor-default`). `existingWords.includes(item.word)` 도 추가됨으로 판정.

**StrokeOrderPanel**:
- `word.split("").filter(ch => /[\u4e00-\u9fff]/.test(ch))` 한자만 통과, 비한자면 아무것도 렌더 안 함.
- 각 글자 컨테이너: `88×88 rounded-lg border bg-white`, 배경에 22px 그리드(선형 그라디언트 `#e5e7eb`).
- HanziWriter 옵션: `width:86, height:86, padding:8, showOutline:true, strokeColor:"#243158", outlineColor:"#dde3ef", strokeAnimationSpeed:0.8, delayBetweenStrokes:200`.
- 마운트 후 100 ms 지연 자동 재생.
- 버튼: `재생` (Play) / `초기화` (RotateCcw). 프리픽스 이모지/화살표 없음.

**LyricCardDialog(단어 모드)**: P3k 정의 재사용. `mode="word"`, `selectedWords: CardWordItem[]` (`{word, pinyin, meaning, example, level}`), `songTitle`, `artist`, `hskLevel` 전달. 다이얼로그 닫힘(`onOpenChange(false)`)과 동시에 `cancelSelectMode()`.

### 2.3 강제 제약

- semantic token 우선. 예외 허용 hardcoded 색: 프로젝트 브랜드 톤 `hsl(220,40%,23%)`, `#3d6cb5`, `#2e3d6b`, `#e8ecf5`, `#c5ceea`, `#f0f2f8`, `#fafbfd`, `#243158`, `#dde3ef` 및 점수용 emerald/amber/red.
- 모든 TTS 호출 전 `window.speechSynthesis.cancel()` 로 큐 초기화. `rate = 0.85`, lang `ko-KR | zh-CN`.
- 재분석 결과는 **낙관적으로 반영 금지**. 성공 후에만 `setCurrentWords(cleanedData)` + `setIsReanalyzed(true)`. 첫 재분석 진입 시 `originalWords = [...words]` 백업.
- 재분석 응답을 **언어에 맞춰 필드 정제**: `zh` → `topik_level` 제거; `ko` → `hsk_level` 과 `pinyin` 제거.
- 성공 후 `buildTagsFromWordList` 로 `songs.tags` 를 비동기 업데이트하되 실패해도 사용자 흐름 유지(`console.warn`).
- `fromRelated=true` 카드는 `✦ 연관어` 버튼을 렌더하지 않는다.
- 삭제 버튼은 `editMode && !selectMode` 에서만 표시. 삭제 후 즉시 `isReanalyzed=true` 임시태로 전환, 저장 전에는 DB 반영 금지.
- 연관어 추가: `generate-example-sentence` 호출 → `WordItem{fromRelated:true, hsk_level:0}` 생성 → **부모 단어 인덱스 + 1 위치에 삽입**. 실패 시 toast + 예외 전파.
- `speech-evaluate` body: `{ audio_base64, text, category: "read_word"|"read_sentence" }`. 응답의 숫자 필드로 `generateFeedback()` 실행 후 결과 캐시.
- 마이크 권한 거부 시 `toast.error("마이크 접근 실패", "마이크 권한을 허용해주세요.")`, UI 잠금 없이 이전 상태 복귀.
- `reanalyze-wordlist` 는 `lines_limit` 이 존재하면 `{words, next_offset, has_more, total_lines}` 객체, 없으면 **레거시 배열** 을 반환한다. 두 경로 모두 유지.

## ③ Examples

### 3.1 확정 카피 표

| 위치 | 카피 |
|---|---|
| 좌측 카운터 | `{n}개 단어` |
| 편집 진입 | `수정` (Pencil) |
| 편집 종료 | `완료` |
| 재분석 트리거 | `단어장 재분석` (RefreshCw) |
| 재분석 팝오버 제목 | `단어장 재분석` |
| 학습 언어 라벨 | `학습 언어` · pill `중국어` / `한국어` |
| 수준 라벨 | `현재 나의 수준` |
| 수준 힌트 | `선택한 등급 이상의 단어가 표시됩니다` |
| 재분석 CTA | `이 설정으로 재분석` |
| 재분석 진행 | `재분석 중...` |
| 재분석 배너 | `⚠ 재분석 결과입니다. 저장하지 않으면 원본으로 돌아갑니다.` |
| 원본 복귀 | `원본으로 복귀` |
| 저장 | `저장` (Save) |
| 저장 성공 | `저장됨` (Check, emerald 2s) |
| 저장 토스트 | `저장 완료` |
| 재분석 실패 토스트 | `재분석 실패: {msg}` |
| 카드 저장 진입 | `📤 카드로 저장` |
| 카드 저장 취소 | `✕ 취소` |
| 선택 배너 | `단어를 클릭해서 선택하세요 · {n}개 선택됨` |
| 카드 만들기 CTA | `카드 만들기 →` |
| 급수 필터 프리픽스 | `HSK` 또는 `TOPIK` · `전체` · `{n}급` |
| 폰트 트리거 | `글꼴` (AArrowUp) |
| 폰트 슬라이더 라벨 | `글꼴 크기: {n}px` |
| 획순 버튼 | `✍ 획순` |
| 연관어 버튼 | `✦ 연관어` |
| 연관어 섹션 라벨 | `유의어` / `반의어` / `자주 쓰는 표현` |
| 연관어 로딩 | `연관어 불러오는 중...` |
| 연관어 실패 | `연관어를 불러올 수 없습니다.` / toast `연관어 불러오기 실패` |
| 연관어 추가 성공 토스트 | `연관어가 추가되었습니다` |
| 연관어 추가 실패 토스트 | `추가 실패: {msg}` |
| 연관어 배지 | `연관어` |
| 급수 없음 배지 | `—` |
| 예문 프리픽스 | `예: {sentence}` |
| 획순 재생/초기화 | `재생` (Play) / `초기화` (RotateCcw) |
| 마이크 실패 토스트 | `마이크 접근 실패` — `마이크 권한을 허용해주세요.` |
| 점수 토스트 | `🎉 발음 점수: {n}점` / `👍 …` / `💪 …` |
| 점수 카드 | `{n}점` + `발음 {p}` + (f>0) `유창성 {f}` + feedback |

### 3.2 컴포넌트 트리

```text
<WordListTab>
  ├─ Toolbar Row 1  (justify-between)
  │    ├─ "{n}개 단어"
  │    └─ [수정|완료] [단어장 재분석 ▾] [저장|저장됨] [📤 카드로 저장|✕ 취소]
  │            │
  │            └─ ReanalyzePopover
  │                 ├─ "학습 언어" pill (중국어 / 한국어)
  │                 ├─ "현재 나의 수준" chips (1..9 or 1..6)
  │                 ├─ 힌트: "선택한 등급 이상의 단어가 표시됩니다"
  │                 └─ CTA "이 설정으로 재분석"
  ├─ ReanalyzeBanner  (isReanalyzed && showBanner)
  ├─ SelectBanner     (selectMode)
  ├─ Toolbar Row 2
  │    ├─ "HSK"/"TOPIK" + [전체][1급]...[9급|6급]
  │    └─ (ml-auto) [글꼴 ▾ → Slider 11..22]
  ├─ Rows (overflow-y-auto min-h-[300px])
  │    └─ WordRow × N
  │          ├─ (editMode) 좌측 원형 × 삭제
  │          └─ Card (fromRelated=left-border, selected=ring)
  │                ├─ Main Line: 단어 · 병음(zh) · 뜻 · [연관어 배지] · 급수 배지 · [✍ 획순] · [✦ 연관어] · TTS · Mic
  │                ├─ 예문 (클릭 → TTS)
  │                ├─ <StrokeOrderPanel/>       (isStrokeOpen)
  │                ├─ <RelatedWordsPanel/>      (isRelatedOpen && !fromRelated)
  │                └─ SpeechResultCard          (results[word])
  └─ <LyricCardDialog mode="word" .../>
```

### 3.3 상태 다이어그램

```text
idle ──[재분석 성공]──▶ reanalyzed(임시)
  ▲                            │
  │◀─[원본으로 복귀]────────────┤
  │◀─[저장]─────────────────────┘   (DB 반영 + isReanalyzed=false)

idle ──[수정]──▶ editMode ──[삭제]──▶ reanalyzed(임시)
                    └─[완료]─▶ idle

idle ──[📤 카드로 저장]──▶ selectMode ──[카드 만들기]──▶ LyricCardDialog
                                └─[✕ 취소]─▶ idle
```
세 모드(reanalyze / edit / select)는 동시에 활성화될 수 있으나 UI 상호작용 우선순위는 **selectMode > editMode > 일반**.

## ④ Context

### 4.1 프로젝트 맥락
단어 탭은 곡 분석 다이얼로그 6-탭 중 학습 밀도가 가장 높은 화면. 어휘 한 개당 최대 6 상호작용(TTS/녹음평가/획순/연관어확장/편집/카드화)을 제공하되 초기 렌더는 목록만 보이도록 progressive disclosure 를 지킨다.

### 4.2 Lovable Cloud 후경 (Lovable 실천 원칙 "Build with Lovable Cloud in Mind")

- 4-상태: 로딩(스켈레톤 8 행) / 빈(`분석된 단어가 아직 없습니다`) / 에러(toast + retry) / 성공(목록).
- `useSpeechEvaluate` 훅:
  - `getUserMedia({ audio:{ sampleRate:16000, channelCount:1 } })`, `MediaRecorder(mimeType:"audio/webm")`.
  - 정지 시 Blob → arrayBuffer → `AudioContext({sampleRate:16000}).decodeAudioData` → `Float32Array` → `Int16Array` (clamp -1..1, `<0 ? *0x8000 : *0x7FFF`).
  - Uint8 뷰 → 문자열 → `btoa` → base64.
  - `supabase.functions.invoke("speech-evaluate", { body:{ audio_base64, text, category } })`.
  - 응답에 `total_score` 숫자 존재 시 `generateFeedback(score, pronunciation, fluency, category)` 로 한국어 피드백 조립 후 `setResults(prev => { ...prev, [text]: {...} })`.
- `speech-evaluate` 응답 스키마: `{ total_score, pronunciation, fluency, error? }`.
- `reanalyze-wordlist` 엔드포인트:
  - 입력: `{ song_id, lang: "zh"|"ko", level: 1..9|1..6, lines_offset?, lines_limit? }`.
  - 출력(레거시): `WordItem[]`; 출력(배치): `{ words, next_offset, has_more, total_lines }`.
  - 내부: `songs.lyrics_raw` 조회 → 줄 단위 슬라이스 → 10 줄 청크 → 동시 3 개 `gpt-4o-mini` 호출(fallback: Lovable Gateway `google/gemini-2.5-flash`) → `word` 기준 dedupe.
  - 프롬프트 규칙: `word` 는 학습 언어 원문, `meaning_ko` 는 반대 언어 번역(zh→한국어, ko→중국어), `example_sentence` 는 반드시 `word` 를 포함하는 자연스러운 원문.
- `generate-related-words({ word, lang })` → `{ syn, ant, col }` 각 `RelatedItem[]`.
- `generate-example-sentence({ word, lang })` → `{ example_zh?, example_ko? }`.

### 4.3 데이터 계약

`song_analyses.data.words: WordItem[]` — 필드:
- `word: string` (필수)
- `pinyin?: string` (zh 만, `""` 허용)
- `meaning_ko: string` (반대 언어 번역 저장 슬롯 — 이름은 legacy)
- `hsk_level?: number` (zh)
- `topik_level?: number` (ko)
- `example_sentence: string`
- `fromRelated?: boolean` (UI-only 마커)

`songs.tags: text[]` — `buildTagsFromWordList(words, mode)` 결과:
- 각 항목의 `mode==="ko" ? (topik_level ?? hsk_level) : (hsk_level ?? topik_level)` 값을 카운트.
- `${"HSK"|"TOPIK"} ${lvl} (${count})` 형식으로 카운트 내림차순 top-3 반환.
- 데이터가 비어 있으면 `[]`.

RLS 는 P3h 정의를 재사용(`songs`, `song_analyses` 각각 소유자 또는 공용).

## ⑤ Acceptance & Output

### 5.1 Acceptance Criteria

- HSK 필터 `3` 선택 시 DOM 카운트 = fixture 중 `hsk_level === 3` 항목 수.
- 재분석(ko) 성공 후 각 항목에 `hsk_level`, `pinyin` 키 부재 (assert `Object.keys(row).indexOf("hsk_level") === -1`). zh 재분석 후 `topik_level` 키 부재.
- 재분석 저장 완료 시 `songs.tags` 는 정규식 `/^(HSK|TOPIK) \d+ \(\d+\)$/` 를 만족하는 항목만 포함하고 개수 ≤ 3.
- 첫 재분석 진입 시 `originalWords` 스냅샷이 생성되고, `원본으로 복귀` 클릭 → `currentWords === originalWords`, `isReanalyzed=false`, 배너 사라짐.
- 스피커/예문 클릭 시 SpeechSynthesis queue length ≤ 1(연타 시 이전 발화 자동 취소).
- HanziWriter 는 한자 이외 문자만 있는 word 입력 시 컨테이너를 하나도 렌더하지 않음(비한자 fixture 로 검증).
- `speech-evaluate` 호출 페이로드는 `{ audio_base64:string, text:string, category:"read_word"|"read_sentence" }` 세 키만 포함.
- 마이크 권한 거부 시 `toast.error` 발생 후 `recording === null`, 이전 UI 상태 유지.
- editMode 활성 & selectMode 비활성 → 각 카드 좌측에 원형 `×` 렌더. selectMode 활성이면 `×` 미렌더.
- `fromRelated=true` 카드는 `✦ 연관어` 버튼을 렌더하지 않고 좌측 3 px `border-l` 액센트를 가진다.
- 연관어 추가 성공 시 새 항목 위치 = 부모 인덱스 + 1, `fromRelated=true`, `hsk_level=0`.
- `RelatedWordsPanel` 은 컨테이너 안에 `<div className="grid grid-cols-3">` 노드를 포함하지 않는다(세로 배열 강제).
- `rg -n "bg-\[#(?!3d6cb5|2e3d6b|e8ecf5|c5ceea|f0f2f8|fafbfd|243158|dde3ef)" src/components/songs/{WordListTab,StrokeOrderPanel,RelatedWordsPanel}.tsx` 매칭 = 0.
- `rg -n "text-white" src/components/songs/{WordListTab,StrokeOrderPanel,RelatedWordsPanel}.tsx` 는 활성 pill/버튼 컨텍스트 외의 매칭 = 0.

### 5.2 Output Format

반환 순서:
1. `src/components/songs/songTagUtils.ts`
2. `src/hooks/useSpeechEvaluate.ts`
3. `src/components/songs/StrokeOrderPanel.tsx`
4. `src/components/songs/RelatedWordsPanel.tsx`
5. `src/components/songs/WordListTab.tsx`
6. `supabase/functions/reanalyze-wordlist/index.ts`
7. 한국어 3 줄 요약(2 행 툴바 / 세 모드 격리 / 배치 재분석 지원).
