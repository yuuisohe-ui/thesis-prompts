# P3l · 단어장 탭(Word List) 재현 프롬프트

> **본 프롬프트는 P3 시리즈(P3a–P3q)의 12/17.**
> **적용 대상**: `src/components/songs/WordListTab.tsx`, `StrokeOrderPanel.tsx`, `RelatedWordsPanel.tsx`, `SpeechPrivacyNotice.tsx`, `src/hooks/useSpeechEvaluate.ts`, `src/lib/speechAudio.ts` — 단어 리스트 · HSK/TOPIK 필터 · 재분석 · 편집 · 카드 저장 · 획순 애니메이션 · 연관어 · **실제 음성 발음평가**.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**
> **개정 이력**: 초판은 평가 응답을 `{ total_score, pronunciation, fluency }` 로만 서술했고 TTS 를 브라우저 음성으로 기술했다. 현재 플랫폼은 讯飞(iFlytek) ISE 실측 평가(`tone` · `integrity` · `lang_profile` · `word_scores`), 개인정보 2단 고지, Typecast/讯飞 서버 TTS 를 사용한다. 본 개정판은 현재 코드를 기준으로 한다.

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* Retrieved July 12, 2026, from https://platform.openai.com/docs/guides/prompt-engineering
2. **Lovable. (n.d.).** *Prompting best practices.* Retrieved July 12, 2026, from https://docs.lovable.dev/prompting/prompting-one
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6 "Verifiable", p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity (신원)

당신은 시니어 프론트엔드 엔지니어 겸 한중 이중언어 교육 UX 라이터입니다. React 18 + Vite 5 + Tailwind CSS v3 + shadcn/ui + Supabase(JS v2) 위에서, HanziWriter(획순) · MediaRecorder + Web Audio + lamejs(MP3 인코딩) · 讯飞 ISE(발음평가) · Typecast/讯飞(TTS) · LLM 기반 재분석/연관어를 조합해 단어장 탭을 구현합니다. 대상은 대한민국 대학의 K-Chinese/K-Korean 교사·학습자이며, UI 카피는 순수 한국어입니다.

## ② Instructions

### 2.1 산출물 (Prompt by Component, Not Page)

- `src/components/songs/WordListTab.tsx` — 단어장 본체(툴바 · 배너 · 필터 · 카드 리스트 · 평가 결과).
- `src/components/songs/StrokeOrderPanel.tsx` — HanziWriter 획순 패널(중국어 전용).
- `src/components/songs/RelatedWordsPanel.tsx` — 유의어/반의어/관용표현 패널 + 단어장 추가.
- `src/components/songs/SpeechPrivacyNotice.tsx` — 발음평가 개인정보 고지(1회 다이얼로그 + 상시 ⓘ 버튼) 및 `isSpeechNoticeAcked` / `ackSpeechNotice`.
- `src/hooks/useSpeechEvaluate.ts` — 녹음 → MP3 변환 → `speech-evaluate` 호출 → 결과/피드백 상태 관리.
- `src/lib/speechAudio.ts` — `encodeMp3Base64(pcm, sampleRate)` · `blobToMp3Base64(blob)`.
- `src/components/songs/songTagUtils.ts` 의 `buildTagsFromWordList(words, mode)` 재사용(신규 작성 금지).
- 낭독은 `src/hooks/useXfTts.ts` 의 `speakTts` 재사용, 취약 부분 칩은 `ReadingTab.tsx` 의 `WeakPoints` 재사용.

### 2.2 원자적 UI 규칙 (Speak Atomic)

#### 2.2.1 Props 및 언어 체계

```ts
interface WordListTabProps {
  words: WordItem[]; onSave(words: WordItem[]): void; readOnly?: boolean;
  songLanguage?: string; songId?: string; shareToken?: string | null;
  lyricsRaw?: string | null; songTitle?: string; songArtist?: string | null; songHskLevel?: string | null;
}
type LangSystem = "zh" | "ko";
```
- **언어 자동 감지**: `words.some(w => w.topik_level != null) ? "ko" : "zh"` 를 `langSystem` 초기값으로 사용한다.
- `langSystem === "zh"` → 등급 라벨 `HSK`, 등급 배열 `[1..9]`, 병음 표시 O, 획순 버튼 O.
- `langSystem === "ko"` → 라벨 `TOPIK`, 등급 배열 `[1..6]`, 병음 표시 X, 획순 버튼 X, 등급은 `topik_level ?? hsk_level`.

#### 2.2.2 툴바 (`flex items-center justify-between mb-3`)

- 좌: `{filtered.length}개 단어`(`text-sm font-medium`).
- 우(`!readOnly` 일 때만, 좌→우 고정):
  1. **수정 토글** — `<Pencil>` + `수정` ⇄ `완료`. 활성 시 `border-red-400/50 text-red-600 bg-red-50`.
  2. **단어장 재분석 Popover**(`w-80 p-4`, `align="end"`) — 제목 `단어장 재분석`, `학습 언어` 2-pill(`중국어` / `한국어`, 전환 시 `reanaLevel` 초기화), `현재 나의 수준` 등급 pill 목록(`{라벨} {n}`), 안내 `선택한 등급 이상의 단어가 표시됩니다`, 실행 버튼 `이 설정으로 재분석`(등급 미선택 또는 진행 중 disabled, 진행 중 `<Loader2 animate-spin> 재분석 중...`). 활성 pill = `bg-[hsl(220,40%,23%)] text-white`.
  3. **저장** — `<Save> 저장` ⇄ 성공 시 2초간 `<Check> 저장됨`(`border-emerald-500/30 text-emerald-600`).
  4. **카드로 저장** — `📤 카드로 저장` ⇄ `✕ 취소`(`bg-red-500 animate-pulse`).

#### 2.2.3 배너

- **재분석 임시 상태 배너**(`showBanner && isReanalyzed`): `bg-amber-50 border-amber-300/40 rounded-lg px-4 py-2.5`. 카피 `⚠ 재분석 결과입니다. 저장하지 않으면 원본으로 돌아갑니다.` + 인라인 링크 `원본으로 복귀` + 우측 `<X>` 닫기(배너만 숨김, 상태 유지).
- **선택 모드 배너**(`selectMode`): `bg-[#3d6cb5]/10 border-[#3d6cb5]/40`. 카피 `단어를 클릭해서 선택하세요 · {n}개 선택됨` + `카드 만들기 →`(0개면 disabled) + `취소`.

#### 2.2.4 필터 행

- 좌측 라벨 `HSK` 또는 `TOPIK` → `전체` pill → 등급 pill `{n}급`. 활성 pill = navy fill.
- 필터 규칙: `filter === "all"` 이면 전부, 아니면 `(langSystem === "ko" ? topik_level ?? hsk_level : hsk_level) === parseInt(filter)`.
- 우측(`ml-auto`): **글꼴 Popover**(`<AArrowUp> 글꼴`, `글꼴 크기: {n}px`, `<Slider min=11 max=22 step=1>`, 기본 14) + **`<SpeechNoticeInfoButton>`** 상시 ⓘ.

#### 2.2.5 단어 카드 (`overflow-y-auto flex-1 space-y-3 min-h-[300px]`)

한 행 = `flex items-start gap-2`:
- **삭제 버튼**(`editMode && !readOnly && !selectMode`): 좌측 원형 `×`(`w-7 h-7 border-red-300`, hover 시 red fill) → `handleDeleteWord(actualIndex)`.
- **카드 본체**(`flex-1 rounded-xl border bg-card overflow-hidden hover:shadow-sm`):
  - `fromRelated` 단어는 `border-l-[3px] border-l-[hsl(220,40%,23%)]`.
  - `editMode && !selectMode` 이면 `border-red-200`.
  - `selectMode` 이면 카드 전체 클릭 선택(`[&_button]:pointer-events-none` 으로 내부 버튼 비활성), 선택 시 `ring-2 ring-[#3d6cb5] bg-[#3d6cb5]/5`.
  - **메인 행**(`p-4 flex items-center gap-3 flex-wrap`): 단어(`font-bold`, `fontSize + 10`px, `min-w-[60px]`) → 병음(zh 전용, `fontSize`) → 뜻(`meaning_ko`, `fontSize`) → `flex-1` 스페이서 → `연관어` 배지(`fromRelated` 일 때) → 등급 배지(값이 없거나 0이면 `—`) → `✍ 획순`(zh 전용) → `✦ 연관어`(`fromRelated` 아닌 단어만) → `<Volume2>` 낭독 → 마이크 버튼.
  - **예문 행**: `예: {example_sentence}` 전체가 버튼이며 클릭 시 예문을 낭독.
  - 획순/연관어 패널은 카드 하단에 인라인 확장(동시에 하나씩만: `strokeOpenWord`, `relatedOpenWord` 는 단일 문자열 상태, 같은 단어 재클릭 시 닫힘).
  - 평가 결과 카드는 `mx-4 mb-3 rounded-lg border px-3 py-2`.

**낭독 규칙**: 단어·예문 모두 `speakTts(text, langSystem)` — `"ko"` → Edge Function `typecast-tts`, `"zh"` → `xf-tts`, 실패 시 브라우저 음성 fallback(`ko-KR`/`zh-CN`, rate 0.85). 컴포넌트에서 `SpeechSynthesisUtterance` 를 직접 만들지 않는다.

#### 2.2.6 발음평가(실측) — `useSpeechEvaluate`

**개인정보 2단 고지**
- 상수 `SPEECH_NOTICE_TEXT = "음성이 발음평가를 위해 외부 서비스로 전송됩니다."`, `localStorage` 키 `speech-eval-notice-ack`.
- `startRecording(text)` 진입 시 `isSpeechNoticeAcked()` 가 false 이면 **녹음을 시작하지 않고** 대상 텍스트를 `pendingTextRef` 에 보관 후 `noticeOpen = true`.
- `<SpeechNoticeDialog>`(`max-w-sm`): 제목 `발음 평가 안내`, 본문 = 고지 문구, 보조 문구 `이 안내는 최초 1회만 표시됩니다.`, 버튼 `취소` / `확인`. `확인` → `ackSpeechNotice()` + 보류된 녹음 자동 시작. `취소` → 보류 폐기(다음 클릭 시 다시 표시).
- `<SpeechNoticeInfoButton>`: 상시 노출되는 `<Info className="h-3.5 w-3.5">` 버튼, `aria-label="발음 평가 안내"`, Popover(`w-64 p-3 text-xs`)로 같은 문구 재확인.

**녹음 → 인코딩 → 평가 파이프라인**
1. `getUserMedia({ audio: { sampleRate: 16000, channelCount: 1 } })` → `MediaRecorder` 로 청크 수집. 권한 거부 시 toast `마이크 접근 실패 / 마이크 권한을 허용해주세요.`(destructive).
2. 정지 시 트랙 종료 → `Blob` → `AudioContext({ sampleRate: 16000 })` 로 디코드 → 채널 0 Float32 PCM 추출.
3. `encodeMp3Base64(pcm, sampleRate)`: Float32 → Int16 클램프 변환 → `new Mp3Encoder(1, sampleRate, 32)`(모노 32 kbps) → `blockSize = 1152` 단위 인코딩 + `flush()` → `0x8000` 청크 단위 base64(스택 오버플로 방지).
4. `supabase.functions.invoke("speech-evaluate", { body: { audio_base64, text, category, lang } })`
   - `category: "read_word" | "read_sentence"`(단어장은 항상 `"read_word"`), `lang = langSystem`.
5. 응답 처리:
   - `typeof data.total_score === "number" && data.total_score > 0` 일 때만 성공. 결과를 `results[text]` 에 저장하고 toast `{emoji} 발음 점수: {n}점`(≥80 `🎉`, ≥60 `👍`, 그 외 `💪`).
   - `data.error` 존재 → toast `평가 실패` + 서버 메시지. 그 외 실패 → `평가 실패 / 음성을 인식하지 못했습니다. 다시 녹음해 주세요.`
   - 예외 → toast `평가 오류` + `err.message`. **실패한 시도는 결과에 기록하지 않는다**(모의 점수 생성 금지).

**응답 계약**
```ts
type WordScore = { text: string; overall: number; pronunciation: number; tone?: number };
type EvaluationResult = {
  total_score: number; pronunciation: number; fluency: number;
  tone?: number; integrity?: number;
  lang_profile?: "zh" | "ko";   // "ko" 프로필에는 성조 점수가 없다
  word_scores?: WordScore[];
  feedback: string;             // 클라이언트에서 생성한 한국어 피드백
};
```
서버(`speech-evaluate`)는 `lang` 에 따라 讯飞 ISE 엔드포인트를 분기한다: `zh → /v1/private/s8e098720 (lang: "cn")`, `ko → /v1/private/sffc17cdb (lang: "kr")`. 한국어는 `core: "word"` 에서 빈 점수가 반환되므로 **단어 평가도 `core: "sent"`** 로 보내고 `words[0].scores` 에서 점수를 추출한다. 프론트엔드는 이 분기를 알 필요 없이 `lang` 만 전달한다.

**한국어 피드백 생성 규칙**(`generateFeedback`, 총점→발음→유창성 순 문장 결합):
- 총점 ≥90 `🎉 훌륭합니다! 발음이 매우 정확합니다.` / ≥75 `👍 잘했습니다! 전반적으로 좋은 발음입니다.` / ≥60 `💪 괜찮습니다. 조금 더 연습하면 좋아질 거예요.` / 그 외 `📖 원어민 발음을 듣고 따라 해보세요.`
- 발음 ≥85 `발음 정확도가 높습니다.` / ≥65 → ko `받침과 억양에 조금 더 신경 써보세요.`, zh `성조와 발음에 조금 더 신경 써보세요.` / 그 외 → ko `받침을 또렷하게 발음해 보세요.`, zh `성조를 정확히 구분하여 발음해 보세요.`
- 유창성 문장은 `category === "read_sentence"` 일 때만 추가(단어장에서는 미출력).

**결과 카드 표시**(`scoreBg`/`scoreColor`: ≥80 emerald · ≥60 amber · 그 외 red)
- `{총점}점`(굵게) · `발음 {n}` · `유창성 {n}`(>0 일 때) · `완성도 {n}`(`integrity` 있을 때) · `성조 {n}`(**`langSystem === "zh"` 이고 `tone` 있을 때만**).
- `<WeakPoints result={result} />`: `word_scores` 중 `0 < overall < 60` 인 항목을 최대 8개 칩으로(`취약 부분` 라벨 + `{text} {점수}`), 없으면 렌더하지 않음.
- 하단에 `feedback` 문장.

**마이크 버튼 상태**: 평가 중 `<Loader2 animate-spin>` + `disabled`, 녹음 중 `<MicOff>` + `variant="destructive"`, 대기 `<Mic>`. **다른 단어가 녹음 중이면 모든 마이크 버튼 disabled**(`!!recording && !isRecording`).

#### 2.2.7 재분석 · 편집 · 연관어 · 카드 저장

**재분석**(`handleReanalyze`)
- `supabase.functions.invoke("reanalyze-wordlist", { body: { song_id, lang: langSystem, level: reanaLevel } })`.
- 응답이 배열이 아니면 `Invalid response format` 오류. 배열이면 언어별 필드 정리: `zh` → `topik_level` 제거, `ko` → `hsk_level` · `pinyin` 제거.
- `currentWords = cleaned`, `isReanalyzed = true`, `showBanner = true`. 최초 재분석 직전 `originalWords = [...words]` 백업.
- 성공 후 `buildTagsFromWordList(cleaned, langSystem)` 결과가 비어 있지 않으면 `songs.tags` 를 UPDATE(실패는 `console.warn` 만, 사용자 흐름 차단 금지).
- 실패 toast `재분석 실패: {메시지}`. **재분석 결과는 `저장` 을 눌러야 영속화된다**(낙관적 저장 금지).
- `원본으로 복귀` → `currentWords = originalWords`, `originalWords = null`, `isReanalyzed = false`, 배너 숨김.

**저장**(`handleSave`): `onSave(isReanalyzed ? currentWords : [...words])` → `저장 완료` toast → `isReanalyzed/showBanner/originalWords` 초기화 → 2초 후 버튼 라벨 복귀.

**삭제**(`handleDeleteWord`): 현재 표시 배열에서 splice 후 `currentWords` 로 승격하고 `isReanalyzed = true`(미저장 변경 표시).

**획순 패널**(`StrokeOrderPanel`)
- 입력 단어에서 `[\u4e00-\u9fff]` 문자만 추출, 없으면 `null` 반환.
- 글자당 `window.HanziWriter.create(el, ch, { width:86, height:86, padding:8, showOutline:true, strokeColor:"#243158", outlineColor:"#dde3ef", strokeAnimationSpeed:0.8, delayBetweenStrokes:200 })`.
- 컨테이너 `w-[88px] h-[88px]` + 22px 격자 배경. 마운트 100 ms 후 자동 애니메이션, 버튼 `<Play> 재생` / `<RotateCcw> 초기화`(`hideCharacter()` + `showOutline()`).

**연관어 패널**(`RelatedWordsPanel`)
- 마운트 시 `generate-related-words`(`{ word, lang }`) 1회 호출, 언마운트 시 `cancelled` 가드.
- 로딩: 3점 pulse + `연관어 불러오는 중...`. 실패: `연관어를 불러올 수 없습니다.` + toast `연관어 불러오기 실패`.
- 섹션 라벨 `유의어`(syn) / `반의어`(ant) / `자주 쓰는 표현`(col), 각 항목: 단어 · 병음(zh 전용) · 뜻 · `<Volume2>` 낭독 · `+` 추가 버튼(진행 중 스피너, 완료/기존 단어면 `✓` + `이미 추가됨`).
- 추가 흐름(`handleAddRelatedWord`): `generate-example-sentence`(`{ word, lang }`) 호출 → `zh` 는 `example_zh`, `ko` 는 `example_ko` 사용 → `{ word, pinyin(zh만), meaning_ko, hsk_level: 0, example_sentence, fromRelated: true }` 를 **부모 단어 바로 뒤**에 삽입(부모 없으면 맨 끝) → `isReanalyzed = true` → toast `연관어가 추가되었습니다`. 실패 시 toast `추가 실패: {메시지}` 후 에러를 다시 throw(패널이 버튼 상태 롤백).

**카드 저장**(`openCardEditor`): 선택 인덱스의 단어를 `{ word, pinyin(zh만), meaning, example, level }` 로 매핑(`level = grade > 0 ? "{라벨} {grade}" : ""`) 후 `<LyricCardDialog mode="word" selectedWords={items}>` 오픈. 닫히면 선택 모드 해제.

### 2.3 강제 제약

- semantic token 우선. 브랜드 hex `hsl(220,40%,23%)`, `#243158`, `#3d6cb5`, `#2e3d6b`, `#e8ecf5`, `#c5ceea`, `#fafbfd`, `#dde3ef`, `#f0f2f8` 만 예외 허용, 그 외 `bg-[#…]` / `text-white` / `bg-black` 금지.
- 발음 점수는 **반드시 `speech-evaluate` 실측값**만 사용한다. `Math.random()` 기반 모의 점수, 임의 보정, 실패 시 대체 점수 생성 모두 금지.
- 마이크 접근 전 반드시 고지 게이트를 통과해야 한다(`isSpeechNoticeAcked()` 없이 `getUserMedia` 호출 금지).
- TTS 는 `speakTts` 단일 경로. 컴포넌트가 `xf-tts` / `typecast-tts` 를 직접 invoke 하지 않는다.
- 재분석·연관어 추가·삭제는 모두 **미저장 상태**이며 `onSave` 이전에 DB 를 갱신하지 않는다(예외: `songs.tags` 동기화).
- `readOnly === true` 이면 툴바 전체(수정·재분석·저장·카드 저장)를 렌더하지 않는다.
- 카피는 순수 한국어(lorem ipsum 금지). `중국어`/`한국어` 등 언어 라벨은 한국어 표기를 사용한다.

## ③ Examples

### 3.1 확정 카피 표 (Design with Real Content)

| 위치 | 카피 |
|---|---|
| 개수 표시 | `{n}개 단어` |
| 수정 토글 | `수정` / `완료` |
| 재분석 트리거 | `단어장 재분석` |
| 재분석 팝오버 | `학습 언어` · `중국어` · `한국어` · `현재 나의 수준` · `선택한 등급 이상의 단어가 표시됩니다` |
| 재분석 실행 | `이 설정으로 재분석` / `재분석 중...` |
| 재분석 실패 | `재분석 실패: {메시지}` |
| 저장 | `저장` / `저장됨` / `저장 완료` |
| 카드 저장 | `📤 카드로 저장` / `✕ 취소` / `카드 만들기 →` |
| 임시 상태 배너 | `⚠ 재분석 결과입니다. 저장하지 않으면 원본으로 돌아갑니다.` / `원본으로 복귀` |
| 선택 배너 | `단어를 클릭해서 선택하세요 · {n}개 선택됨` |
| 필터 | `전체` / `{n}급` |
| 글꼴 | `글꼴` / `글꼴 크기: {n}px` |
| 등급 배지 | `HSK {n}` / `TOPIK {n}` / `—` |
| 연관어 배지 | `연관어` |
| 획순 버튼 | `✍ 획순` |
| 획순 컨트롤 | `재생` / `초기화` |
| 연관어 버튼 | `✦ 연관어` |
| 연관어 섹션 | `유의어` / `반의어` / `자주 쓰는 표현` |
| 연관어 상태 | `연관어 불러오는 중...` / `연관어를 불러올 수 없습니다.` / `연관어 불러오기 실패` |
| 연관어 추가 | `단어장에 추가` / `이미 추가됨` / `연관어가 추가되었습니다` / `추가 실패: {메시지}` |
| 예문 | `예: {문장}` |
| 개인정보 고지 | `음성이 발음평가를 위해 외부 서비스로 전송됩니다.` |
| 고지 다이얼로그 | `발음 평가 안내` · `이 안내는 최초 1회만 표시됩니다.` · `취소` · `확인` |
| 마이크 실패 | `마이크 접근 실패` / `마이크 권한을 허용해주세요.` |
| 평가 성공 toast | `{emoji} 발음 점수: {n}점` |
| 평가 실패 | `평가 실패` / `음성을 인식하지 못했습니다. 다시 녹음해 주세요.` / `평가 오류` |
| 점수 지표 | `{n}점` · `발음 {n}` · `유창성 {n}` · `완성도 {n}` · `성조 {n}` |
| 취약 부분 | `취약 부분` |

### 3.2 컴포넌트 트리 (Use Prompt Patterns for Layouts)

```text
<WordListTab flex flex-col h-full>
  ├─ Toolbar
  │    ├─ "{n}개 단어"
  │    └─ [수정] [단어장 재분석 ▾] [저장] [📤 카드로 저장]      (!readOnly)
  ├─ ReanalyzeBanner (showBanner && isReanalyzed)
  ├─ SelectBanner    (selectMode)
  ├─ FilterRow  [HSK|TOPIK] [전체][1급]…  ── ml-auto ─▶ [글꼴 ▾] [ⓘ SpeechNoticeInfoButton]
  ├─ WordCards (overflow-y-auto min-h-[300px])
  │    └─ per word
  │         ├─ [×] 삭제           (editMode)
  │         ├─ MainRow: 단어 · 병음(zh) · 뜻 · [연관어] · 등급배지
  │         │            · [✍ 획순](zh) · [✦ 연관어] · [🔊] · [🎤]
  │         ├─ "예: …"  (클릭 → 낭독)
  │         ├─ <StrokeOrderPanel>    (isStrokeOpen)
  │         ├─ <RelatedWordsPanel>   (isRelatedOpen && !fromRelated)
  │         └─ ScoreCard: 점수 지표 + <WeakPoints> + feedback
  ├─ <LyricCardDialog mode="word">
  └─ <SpeechNoticeDialog open={noticeOpen} onConfirm onCancel />
```

**호출 흐름**:
```text
🎤 클릭 ─▶ isSpeechNoticeAcked()?
             ├─ false ─▶ SpeechNoticeDialog ─(확인)─▶ ack + 녹음 시작
             └─ true  ─▶ getUserMedia(16k mono) ─▶ MediaRecorder
🎤 재클릭 ─▶ stop ─▶ decodeAudioData(16k) ─▶ encodeMp3Base64(mono, 32kbps)
          ─▶ speech-evaluate { audio_base64, text, category:"read_word", lang }
          ─▶ { total_score, pronunciation, fluency, tone?, integrity?, lang_profile, word_scores[] }
          ─▶ generateFeedback() ─▶ results[text] ─▶ ScoreCard + WeakPoints
```

## ④ Context (배경)

### 4.1 프로젝트 맥락
단어장 탭은 「멜로디 클래스」에서 곡 → 어휘 학습으로 이어지는 전이 지점이다. 교사는 학급 수준에 맞춰 HSK/TOPIK 등급으로 단어를 재추출하고(재분석), 불필요한 단어를 지우고, 연관어를 덧붙여 자기 수업용 단어장을 만든다. 학습자는 같은 화면에서 획순을 보고, 예문을 듣고, 마이크로 실제 발음 점수를 받는다. 발음평가는 외부 서비스로 음성을 전송하므로 최초 1회 고지와 상시 재확인 수단을 함께 제공한다.

### 4.2 Lovable Cloud 후경 (Build with Lovable Cloud in Mind)

| 함수 | 호출 지점 | 실패 처리 |
|---|---|---|
| `speech-evaluate` | 마이크 정지 후 | toast(destructive), 결과 미기록 → 재녹음 유도 |
| `reanalyze-wordlist` | 재분석 실행 | toast `재분석 실패: …`, 기존 목록 유지 |
| `generate-related-words` | 연관어 패널 마운트 | 패널 내 오류 문구 + toast |
| `generate-example-sentence` | 연관어 추가 | toast `추가 실패` + 버튼 상태 롤백 |
| `typecast-tts` / `xf-tts` | `speakTts`(ko/zh) | 브라우저 음성 fallback |
| `songs.tags` UPDATE | 재분석 성공 직후 | `console.warn` 만, 흐름 차단 없음 |

- `reanalyze-wordlist` 는 `song_id`, `lang`, `level` 필수이며 `lines_offset` / `lines_limit` 로 가사 분할 처리를 지원한다(배치 응답에는 `next_offset`, `has_more`, `total_lines` 포함). 프론트엔드는 **배열 응답 경로**를 사용하며 배열이 아니면 오류로 처리한다.
- 4-상태 렌더링: 로딩(스피너/펄스) · 빈(`—` 배지, 패널 미표시) · 에러(toast + 인라인 문구) · 성공(점수 카드/추가 완료 체크).
- RLS: 단어장은 `song_analyses` 를 통해 접근하며 저장은 상위 다이얼로그(P3j)의 `onSave` 가 담당한다. 본 컴포넌트는 `songs.tags` 외 직접 DB 쓰기를 하지 않는다.

### 4.3 데이터 계약

```ts
export interface WordItem {
  word: string;
  pinyin?: string;            // 중국어 전용
  meaning_ko: string;         // zh곡: 한국어 뜻 / ko곡: 중국어 뜻
  hsk_level?: number;         // 1~9, 0 = 미지정
  topik_level?: number;       // 1~6 (한국어 단어장)
  example_sentence?: string;
  fromRelated?: boolean;      // 연관어에서 추가된 단어
}

// speech-evaluate 요청/응답
type EvalRequest  = { audio_base64: string; text: string; category: "read_word" | "read_sentence"; lang: "zh" | "ko" };
type EvalResponse = {
  total_score: number; pronunciation: number; fluency: number;
  tone?: number; integrity?: number; lang_profile: "zh" | "ko";
  word_scores: { text: string; overall: number; pronunciation: number; tone?: number }[];
  error?: string;
};
```

스키마·GRANT·RLS 원본은 P3h(`songs`, `song_analyses`) 및 B2(분석 Edge Functions) 에서 정의한다.

## ⑤ Acceptance & Output (IEEE 830 §4.3.6)

### 5.1 Acceptance Criteria

- **언어 자동 감지**: `topik_level` 을 가진 단어가 1개 이상인 fixture → `langSystem === "ko"`, 라벨 `TOPIK`, 등급 pill 6개, 병음/획순 버튼 렌더 = **0**. 그 반대 fixture → `HSK`, pill 9개.
- **필터 정확도**: 등급 pill 클릭 시 표시 개수 = 해당 등급 단어 수(오차 0). `전체` 클릭 시 = 전체 개수.
- **실측 평가 강제**: `rg -n "Math.random" src/hooks/useSpeechEvaluate.ts src/components/songs/WordListTab.tsx` = **0**. `speech-evaluate` 응답 `total_score = 0` 또는 오류 fixture 에서 `results` 항목 증가 = **0**, toast 1회.
- **평가 요청 페이로드**: 단어장에서 호출 시 `category === "read_word"` 100 %, `lang === langSystem` 100 %, `audio_base64.length > 0`.
- **오디오 인코딩**: 16 kHz 모노 3초 입력에 대해 `encodeMp3Base64` 결과가 유효 MP3 프레임으로 시작하고 크기 ≤ **16 KB**(32 kbps 기준), 100만 샘플 입력에서 `RangeError`(스택 오버플로) 발생 = **0**.
- **고지 게이트**: `localStorage` 비운 상태에서 첫 마이크 클릭 시 `getUserMedia` 호출 = **0**, 다이얼로그 노출 1회. `확인` 후 자동 녹음 시작 1회, `speech-eval-notice-ack === "1"`. `취소` 후 재클릭 시 다이얼로그 재노출.
- **상시 ⓘ**: 필터 행에 `SpeechNoticeInfoButton` 1개 존재, Popover 문구가 `SPEECH_NOTICE_TEXT` 와 문자열 동일.
- **점수 표시 분기**: `langSystem === "ko"` 결과에서 `성조` 문자열 출현 = **0**. `zh` + `tone > 0` 이면 1회.
- **취약 부분**: `word_scores` 중 `0 < overall < 60` 항목만, 최대 **8개** 칩. 해당 항목 0개면 `취약 부분` 렌더 = 0.
- **동시 녹음 차단**: 한 단어 녹음 중 다른 단어 마이크 버튼 `disabled` = 100 %.
- **재분석 비영속성**: 재분석 후 `저장` 없이 언마운트하면 `song_analyses` UPDATE 호출 = **0**(단, `songs.tags` UPDATE 는 1회 허용). `원본으로 복귀` 시 목록이 원본과 완전 일치.
- **재분석 필드 정리**: `lang="zh"` 결과에 `topik_level` 키 = 0건, `lang="ko"` 결과에 `hsk_level`·`pinyin` 키 = 0건.
- **연관어 삽입 위치**: 추가된 단어의 인덱스 = 부모 단어 인덱스 + 1(부모가 없을 때만 마지막), `fromRelated === true`, `hsk_level === 0`.
- **연관어 요청 횟수**: 같은 단어 패널을 열고 닫고 다시 열 때 `generate-related-words` 호출 ≤ **2**(패널은 마운트마다 1회), 열려 있는 동안 추가 호출 = 0.
- **획순**: 비한자 단어(예: `사랑`)에 대해 `StrokeOrderPanel` 렌더 = **null**, HanziWriter 인스턴스 생성 = 0. 한자 2글자 단어는 캔버스 2개 + 마운트 후 자동 애니메이션 1회.
- **TTS 라우팅**: `langSystem === "ko"` 낭독 시 `typecast-tts` 1회 · `xf-tts` 0회, `zh` 는 반대. 동일 텍스트 재클릭 시 함수 호출 = 0(캐시 적중).
- **readOnly**: `readOnly === true` 에서 툴바 버튼 렌더 = **0**, 삭제 버튼 = 0.
- **폰트**: 슬라이더 범위 `[11, 22]`, 단어 글자 크기 = `fontSize + 10`px.
- **리스트 최소 높이**: 단어 0개일 때도 리스트 컨테이너 높이 ≥ **300 px**(레이아웃 점프 방지, CLS ≤ 0.05).
- **하드코드 색상 검사**: `rg -n "bg-\[#(?!3d6cb5|2e3d6b|243158|e8ecf5|c5ceea|fafbfd|f0f2f8|dde3ef)" src/components/songs/WordListTab.tsx src/components/songs/StrokeOrderPanel.tsx src/components/songs/RelatedWordsPanel.tsx src/components/songs/SpeechPrivacyNotice.tsx` = **0**.
- **직접 TTS 호출 금지**: `rg -n "SpeechSynthesisUtterance|invoke\(\"(xf|typecast)-tts" src/components/songs/{WordListTab,RelatedWordsPanel}.tsx` = **0**.
- **한국어 카피 커버리지**: 3.1 표의 모든 카피가 해당 파일에서 최소 1회 등장.

### 5.2 Output Format

LLM(또는 Lovable) 은 아래 파일을 이 순서대로 반환한다. 설명·사과·주석·마크다운 헤더 등 부수 텍스트는 금지한다.

1. `src/lib/speechAudio.ts`
2. `src/components/songs/SpeechPrivacyNotice.tsx`
3. `src/hooks/useSpeechEvaluate.ts`
4. `src/components/songs/StrokeOrderPanel.tsx`
5. `src/components/songs/RelatedWordsPanel.tsx`
6. `src/components/songs/WordListTab.tsx`
7. 한국어 3줄 요약(무엇을 만들었는지, 어떤 edge function 을 호출하는지, 발음평가 고지·실측 규칙).
