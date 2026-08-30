# P3p3 · 발음 연습 (Pronunciation Practice) 재현 프롬프트

> 본 문서는 `docs/thesis/appendix/prompts/00-template.md`가 정의한 5-Section 표준(Identity · Instructions · Examples · Context · Acceptance & Output)을 그대로 따른다. 이론적 근거는 00-template.md에 정리되어 있으며 본 문서에서 재게시하지 않는다.
> **범위**: 곡 분석 다이얼로그(P3j) → 「연습」 탭의 **발음 연습** 한 페이지. 소스: `src/components/songs/PronunciationPracticePage.tsx`(1849행) + 음성 관련 Edge Function 4종(`speech-evaluate` · `pronunciation-tone-change` · `pronunciation-coach` · TTS 2종 `xf-tts`/`typecast-tts`) + 공용 훅 `src/hooks/useXfTts.ts` + 개인정보 고지 컴포넌트 `src/components/songs/SpeechPrivacyNotice.tsx`. 다른 연습 모듈(퀴즈·받아쓰기·작문)은 P3p1·P3p2·P3p4에서 다룬다.
> **기준 시점**: 2026-08 현재 코드. 이전 판 대비 변경점 — ① 한국어 TTS가 讯飞에서 **Typecast**로 이관, ② 발음 평가 전 **1회 동의 다이얼로그 + 상시 ⓘ 아이콘** 추가, ③ `speech-evaluate`가 `word_scores` · `lang_profile` · `integrity`를 반환하고 **한국어는 단어도 `core: "sent"`** 로 평가, ④ 중국어 TTS 발음인이 `x4_xiaoyan`으로 고정, ⑤ 페이지 내부 채점 함수는 **의도적으로 mock 상태로 남아 있으며** 실채점 경로는 `useSpeechEvaluate`(P3k/P3l 소비)임.

---

## ① Identity

당신은 한중 이중언어 발음 교육 UX를 이해하는 시니어 프론트엔드 엔지니어이다. 기술 스택은 React 18 + Vite 5 + Tailwind CSS v3 + shadcn/ui + recharts + Supabase JS v2로 고정된다. 「멜로디 클래스」는 대한민국 대학의 중국어·한국어 학습자를 대상으로 하며, 본 페이지의 모든 UI 카피·해설은 **한국어**를 기본값으로 하고 학습 대상 텍스트만 `lang(zh|ko)`에 따라 전환된다.

---

## ② Instructions

### 2.1 산출물 (Prompt by Component, Not Page)

정확히 아래 7개 파일만 생성/수정한다.

1. `src/components/songs/PronunciationPracticePage.tsx` — 단일 파일(약 1850행). 3-Phase 상태기 · 문제 풀 구성 · 녹음 · 채점 · 성조 시각화 · 변조 해설 오버레이 · 상세 분석 · 결과 화면을 모두 포함한다. 내부 보조 컴포넌트(`StartScreen` · `PracticeScreen` · `WordContent` · `SentContent` · `EvalBars` · `Bar` · `CharAnalysis` · `HistoryList` · `DetailSection` · `FinalScreen` · `StatBox` · `BulbOverlay` · `SectionLabel` · `SelCard` · `OptBtn` · `PlayButton`)를 같은 파일에 둔다.
2. `src/components/songs/SpeechPrivacyNotice.tsx` — 음성 외부 전송 고지(1회 다이얼로그 + 상시 ⓘ 팝오버 + `localStorage` ack 유틸).
3. `src/hooks/useXfTts.ts` — 언어별 TTS 라우팅 · 캐시 · 문장 분할 · 브라우저 폴백. `speakTts` / `stopTts` / `useXfTts` / `toTtsLang`을 export.
4. `supabase/functions/xf-tts/index.ts` — 讯飞 온라인 음성 합성(중국어).
5. `supabase/functions/typecast-tts/index.ts` — Typecast 음성 합성(한국어).
6. `supabase/functions/speech-evaluate/index.ts` — 讯飞 ISE 발음 평가(중/한 엔드포인트 분기).
7. `supabase/functions/pronunciation-tone-change/index.ts` + `supabase/functions/pronunciation-coach/index.ts` — 변조 해설 · AI 코칭.

### 2.2 상태 모델 (Speak Atomic)

```text
Phase:  start ──▶ prac ──▶ final
          ▲                  │
          └──────────────────┘  (다시 연습하기 / 처음으로)
Lang:   zh | ko          Mode: word | sent        Order: seq | rand | inf
Rec:    idle → recording → evaluating → done
```

- `lang` · `mode` · `order` · `phase`는 서로 독립된 `useState`. 뒤로가기는 `prac|final`에서 누르면 페이지를 벗어나지 않고 `start`로 되돌리며 `current=0`, `evalData=null`, `detailOpen=false`, `coachData=null`로 초기화하고 진행 중인 TTS를 `stopTts()`로 정지한다.
- 문제 풀은 `wordPoolZh` / `wordPoolKo` / `sentPool` 세 갈래로 분리 보관한다. 단어 풀은 곡 단어장에서, 문장 풀은 가사 라인에서 만든다. `mergeWords`는 `${ko}|${zh}` 키로 중복을 제거하며 누적 병합한다.
- `order`: `seq`(순서대로) · `rand`(`shuffle`) · `inf`(무한 반복 — `current`를 `(c+1) % length`로 순환하며 `final`로 자동 이동하지 않고, 하단 「■ 연습 종료 및 결과 보기」 버튼으로만 종료).
- 문제 타입은 `WordQ`(zh/ko/pinyin 등) | `SentQ`의 유니언 `Q`. 채점 결과는 `ScoreData { overall, pron, tone, fluency?, rhythm?, integrity? }`, 누적 기록은 `ResultEntry { q, scoreData }[]`.

### 2.3 병음 음절 분해와 성조 판정 (핵심 알고리즘, 그대로 재현할 것)

1. `syllableTone(syll)` — 성조 부호 표(`āēīōūǖ`=1, `áéíóúǘ`=2, `ǎěǐǒǔǚ`=3, `àèìòùǜ`=4)로 1~4성을 판정하고, 부호가 없으면 **5(경성)** 를 반환한다.
2. `splitOneChunk(chunk)` — 붙어 있는 병음(`gǎnjué`)을 음절로 쪼개는 그리디 스캐너.
   - 초성 정규식 `INITIAL_RE = /^(zh|ch|sh|[bpmfdtnlgkhjqxrzcsyw])?/i`.
   - 초성 뒤 모음 클러스터(`VOWELS = "aeiouü" + 성조 부호 포함`)를 소비, 이어서 비음 코다 `n` / `ng`를 소비.
   - **`ng` 모호성 처리**: `g` 다음 글자가 모음이면 `g`를 다음 음절의 초성으로 넘긴다(`fángàn` 류 오분해 방지).
   - 초성도 모음도 소비하지 못하면 남은 문자열을 통째로 밀어 넣고 종료한다(무한 루프 방지).
3. `splitPinyinToSyllables(pinyin)` — 공백/어포스트로피로 1차 분할 후 각 청크에 `splitOneChunk`를 적용.
4. `extractTones(pinyin, expectedLen)` — 한자 개수만큼 성조 배열을 채우고, 부족분은 5로 패딩한다.

성조 팔레트는 고정 상수다: `TONE_NAMES = {1:"1성",2:"2성",3:"3성",4:"4성",5:"경성"}`, `TONE_COLORS` 1성 `hsl(217 91% 50%)` · 2성 `hsl(142 71% 38%)` · 3성 `hsl(35 90% 45%)` · 4성 `hsl(0 75% 50%)` · 경성 `hsl(262 83% 58%)`. `TONE_BADGE_CLASS`는 blue / emerald / amber / red / violet 100·700(다크 950/40·300) 조합.

**성조 곡선 SVG**(`getTonePath`)는 40×34 뷰박스 기준.

| 성조 | path |
| --- | --- |
| 1성 | `M6 17 L38 17` |
| 2성 | `M6 28 L38 6` |
| 3성 | `M6 10 C14 30 30 30 38 12` |
| 4성 | `M6 6 L38 28` |
| 경성 | `M6 20 L26 20` |

### 2.4 Phase 1 · 시작 화면 (`StartScreen`)

- 카드 `bg-card rounded-2xl border border-border shadow-sm p-6`, 진입 애니메이션 `animate-in fade-in slide-in-from-bottom-2 duration-300`.
- 헤더: 이모지 `🎙️`(`text-4xl`) → 제목 `발음 연습`(`text-xl font-bold`) → `{songTitle} — {artistName}`(`text-xs text-muted-foreground`).
- 선택 그룹 3개, 각 라벨은 `SectionLabel`(`text-[11px] font-bold text-muted-foreground uppercase tracking-wider`).
  - **학습 언어** — `SelCard` 2개: `🇨🇳 중국어 / 병음 · 성조`, `🇰🇷 한국어 / 발음 평가`.
  - **연습 모드** — `SelCard` 2개: `📝 단어 발음 / 단어 {wordCount}개`(분석 중이면 `단어 분석 중...`), `🎵 가사 문장 / 문장 {sentCount}개 · 정규화`.
  - **순서** — `OptBtn` 3개: `📋 순서대로` · `🔀 랜덤` · `♾️ 무한`.
- 데이터가 없는 조합(`noData`)이거나 `normalizing` / `analyzing` 중이면 시작 버튼을 비활성화하고 라벨을 각각 `데이터가 없습니다` / `가사 정규화 중...` / `단어 분석 중...`(스피너 동반)으로 바꾼다. 필요 시 `normalize-lyrics` → `analyze-song` 순으로 데이터를 보충한다.
- 시작 버튼: `w-full h-12 text-base font-bold`, 정상 라벨 `시작하기 →`.

### 2.5 Phase 2 · 연습 화면 (`PracticeScreen`)

**헤더** — 좌측에 `{current+1} / {total}`(무한 모드는 `∞`)와 `{mode==="word" ? "단어 발음" : "가사 문장"}`, 그 아래 `h-1.5` 진행 바(무한 모드는 50% 고정). 우측에 언어(blue) · 모드(amber) · 무한(violet) pill 뱃지.

**문제 카드** — `hasFeedback`(평가 완료)일 때만 하단 모서리를 펴서(`rounded-t-2xl border-b-0`) 아래 점수 카드와 하나로 이어 붙인다.

- `WordContent`(단어): 한자별로 병음 · 성조 뱃지 · 성조 곡선을 세로 정렬. `lang==="ko"`이면 성조 UI를 전부 감추고 한글 표기 + 「실제 발음」 행만 남긴다.
- `SentContent`(문장): 중국어 모드는 한국어 번역 → 중국어 원문 → `표준` / `실제` 병음 2행. 한국어 모드는 중국어를 참조행으로, 한국어를 학습 대상행으로 두고 `실제 발음` 1행만 표시한다.
- `실제` 행 우측의 `💡` 버튼은 `BulbOverlay`(전체 화면 `fixed inset-0 z-50 bg-black/45`)를 열어 변조 규칙·이유·변경 부분을 설명한다. 변경이 없으면 「변조 없음」 상태로 렌더한다.
- **시범 낭독 2개**: `🔊 표준 발음` / `🐢 천천히` → `speakTts(text, lang, { speed: slow ? 20 : 50 })`.

**녹음** — 원형 버튼 `w-16 h-16 rounded-full`: `idle` primary + `hover:scale-105`(Mic), `recording` `bg-red-500 animate-pulse`(Square), `evaluating` `bg-amber-500 animate-pulse`(Loader2 스핀, 버튼 disabled), `done` `bg-emerald-500`(Check). 아래에 12개 `.wbar` 웨이브 바를 `setInterval 110ms`로 흔들고(`startWave`/`stopWave`), 상태 문구는 `버튼을 눌러 녹음하세요` / `녹음 중... 다시 눌러 완료` / `평가 중...` / `평가 완료!`. 문구 옆에 **상시 `SpeechNoticeInfoButton`(ⓘ)** 을 둔다.

**개인정보 고지 게이트** — `startRecording()` 첫 줄에서 `isSpeechNoticeAcked()`를 확인하고, 미동의면 녹음을 시작하지 않고 `noticeOpen=true`로 `SpeechNoticeDialog`를 띄운다. 확인 → `ackSpeechNotice()` 후 녹음 재개, 취소 → 중단(다음 클릭 때 다시 표시). 고지 문구는 `SPEECH_NOTICE_TEXT = "음성이 발음평가를 위해 외부 서비스로 전송됩니다."`, `localStorage` 키는 `speech-eval-notice-ack`.

**채점** — `MediaRecorder`로 `audio/webm` blob을 만든 뒤 `evaluatePronunciation(blob, text, lang, mode)`을 호출하고, 결과를 `evalData` + `results`에 적재한다.
> **현행 코드 주의(반드시 그대로 재현)**: 페이지 내부 `evaluatePronunciation()`은 900ms 지연 후 난수를 돌려주는 **mock**이며 `// TODO: language-specific evaluation API` 주석이 남아 있다. 단어 모드는 `{ overall, pron, tone }`, 문장 모드는 `fluency` · `rhythm` · `integrity`를 추가로 반환한다. 실채점 경로는 `src/hooks/useSpeechEvaluate.ts` → `speech-evaluate`이며, 현재 가사 탭·단어장 탭(P3k·P3l)에서만 사용된다. 본 페이지를 실채점으로 승격할 때의 계약은 2.8에 명시한다.

**점수 카드** — 총점 링(`ringClass`: 90+ emerald · 75+ blue · 60+ amber · 그 미만 red, `w-[72px] h-[72px] border-[3px]`) + `EvalBars`. 막대는 `발음`(primary), `성조`(amber, **한국어 모드에서는 라벨이 `음운`**), 문장 모드에서만 `유창성`(emerald) · `운율`(violet) · `완성도`(orange).

**자세히 보기(`detailOpen`)** — 토글을 열 때만 아래를 렌더한다.
- `분석 차트`: recharts `RadarChart`. 단어 모드는 축 3개(`발음` · `성조|음운` · `정확도 = round((pron+tone)/2)`), 문장 모드는 축 5개(`발음` · `성조|음운` · `유창성` · `운율` · `완성도`). `PolarRadiusAxis domain={[0,100]}`, 그리드·축은 `hsl(var(--border))` / `hsl(var(--muted-foreground))` 토큰 사용.
- `글자별 분석`: **중국어 단어 모드에서만** `CharAnalysis` 렌더(글자별 안정 난수 점수).
- `🤖 AI 발음 코치`: 토글을 열 때 **지연 호출**. 캐시 키 `"{lang}:{mode}:{text}:{round(overall/5)*5}"`, 로딩 시 `피드백을 생성하고 있습니다...`, 결과는 본문 + 선택적 팁 박스.
- `📈 연습 기록`: 같은 문제의 이전 시도 목록(`HistoryList`).
- 하단 액션: `다시 녹음`(RotateCcw) / `다음 →`.

**변조 해설(`pronunciation-tone-change`)** — `{ has_change, standard, actual, changed_parts[], rule, reason }`.
- 캐시 키 `"{lang}:{text}"`, 메모리 `Map` + `localStorage` 프리픽스 `tc2:`(마운트 시 1회 하이드레이션).
- 동일 키 동시 요청은 `inFlightRef`로 합류.
- 429/402/`_rate_limited`는 **최대 3회**, `1500 * 2^attempt + jitter(0~500ms)` 지수 백오프.
- 다음 문제의 변조 해설을 백그라운드 프리페치한다.

**내비게이션** — `← 이전`(`current===0 && order!=="inf"`이면 disabled) / `다음 →`, 무한 모드에서는 중앙에 `무한 모드` 표기와 하단에 `■ 연습 종료 및 결과 보기`(빨강 아웃라인).

### 2.6 Phase 3 · 결과 화면 (`FinalScreen`)

- 평가 기록이 없으면 `🤔 평가된 문제가 없습니다.` + `← 처음으로` / `다시 연습하기`.
- 상단: 성취 이모지(90+ `🏆`, 75+ `🥈`, 60+ `🥉`, 그 미만 `💪`) + 평균 총점(`text-5xl font-bold text-primary`) + `평균 점수`.
- `StatBox` 3개: `총 연습`(총 시도 수) · `평가 완료`(채점된 문제 수) · `평균 발음`.
- 강약 카드 2개: `✅ 가장 잘한 단어`(emerald) / `⚠️ 더 연습이 필요해요`(red, 최고=최저면 숨김).
- `단어별 결과`: 텍스트 + 진행 막대(80+ emerald · 65+ amber · 그 미만 red) + 점수.
- 하단: `← 처음으로`(`onHome`) / `다시 연습하기`(`onRestart`).

### 2.7 TTS 라우팅 계약 (`useXfTts.ts`)

- `toTtsLang()`으로 `ko|korean|ko-*` → `"ko"`, 그 외 → `"zh"`로 정규화.
- **언어별 함수 분기**: `ko` → `typecast-tts`, `zh` → `xf-tts`. 요청 본문은 `{ text, lang, speed }` 공통, 응답은 `{ audio_base64 }` 공통(base64 MP3 → `data:audio/mpeg;base64,` URL).
- 모듈 레벨 `audioCache`(키 `"{lang}:{speed}:{text}"`, 200개 초과 시 전체 클리어)와 단일 `currentAudio`를 공유해 새 재생 시 이전 재생을 끊는다.
- `splitSentences(text, 300)`으로 8000바이트 제한 아래로 쪼개 순차 재생한다.
- **폴백**: 함수 오류·`audio_base64` 없음·재생 실패 시 브라우저 `speechSynthesis`(`zh-CN` / `ko-KR`, `rate 0.85`). 응답에 `fallback: true`(자격증명 미설정·쿼터 초과 등)가 있으면 콘솔 경고 없이 조용히 폴백한다.
- Typecast 계약: `POST https://api.typecast.ai/v1/text-to-speech`, 헤더 `X-API-KEY: TYPECAST_API_KEY`, 본문 `{ text, model: "ssfm-v30", language: "kor", voice_id: TYPECAST_VOICE_ID_KO ?? "tc_67db72eb93add6902ea41e5c", output: { audio_format: "mp3" } }`. 앱의 讯飞식 `speed`(0~100, 50=보통)는 `tempo = clamp(speed/50, 0.5, 2)`로 환산한다. 401/402/403/429는 `fallback: true`로 내려 브라우저 음성으로 강등한다.
- 讯飞 TTS 계약: `wss://tts-api.xfyun.cn/v2/tts`, 발음인 후보 배열의 **첫 값은 `x4_xiaoyan`**, 11200(발음인 미개통) 발생 시 다음 후보로 순차 재시도, `aue: "lame"`(MP3).

### 2.8 실채점 계약 (`speech-evaluate`, 승격 시 정답 스펙)

- 요청 `{ audio_base64(MP3 16k mono), text, category: "read_word" | "read_sentence", lang: "zh" | "ko" }`.
- 엔드포인트: `zh` → `/v1/private/s8e098720`(`lang: "cn"`), `ko` → `/v1/private/sffc17cdb`(`lang: "kr"`). 호스트 `cn-east-1.ws-api.xf-yun.com`, HMAC-SHA256 서명 URL.
- **`core` 규칙**: 중국어 단어만 `word`, **한국어는 단어라도 `sent`**(다국어 엔진이 `word`에서 빈 점수를 돌려주기 때문). 중국어가 11201로 실패하면 다국어 엔드포인트로 1회 재시도하며 이때도 `sent`로 강등한다.
- **결과 파싱**: `sent`로 평가된 단어는 `result.words[0].scores`를 우선 사용하고 없으면 최상위 필드로 폴백한다. 응답 `{ total_score, pronunciation, fluency, tone, integrity, lang_profile: "zh"|"ko", word_scores: [{ text, overall, pronunciation, tone }] }`(최대 20개).
- 타임아웃은 오디오 길이 기반(`clamp(estimatedMs*3, 45s, 180s)`)이며, 실패·타임아웃도 **HTTP 200 + `error` 필드**로 내려 클라이언트가 토스트만 띄우게 한다.

### 2.9 정리(cleanup) 규칙

언마운트 시 `stopTts()`, `waveTimerRef` 인터벌 해제, `MediaRecorder`가 `inactive`가 아니면 `stop()`을 호출한다. 마이크 트랙은 `onstop`에서 `stream.getTracks().forEach(t => t.stop())`으로 반드시 회수한다. 마이크 권한 거부 시 `마이크 권한이 필요합니다.` 안내를 띄우고 상태를 `idle`로 되돌린다.

---

## ③ Examples

### 3.1 확정 카피 표

| 위치 | 문자열 |
| --- | --- |
| 시작 화면 제목 | `발음 연습` |
| 언어 카드 | `중국어 / 병음 · 성조`, `한국어 / 발음 평가` |
| 모드 카드 | `단어 발음 / 단어 {n}개`, `가사 문장 / 문장 {n}개 · 정규화` |
| 순서 버튼 | `📋 순서대로` · `🔀 랜덤` · `♾️ 무한` |
| 시작 버튼 | `시작하기 →` / `데이터가 없습니다` / `가사 정규화 중...` / `단어 분석 중...` |
| 낭독 버튼 | `🔊 표준 발음` · `🐢 천천히` |
| 녹음 상태 | `버튼을 눌러 녹음하세요` · `녹음 중... 다시 눌러 완료` · `평가 중...` · `평가 완료!` |
| 고지 문구 | `음성이 발음평가를 위해 외부 서비스로 전송됩니다.` |
| 점수 항목 | `발음` · `성조`(한국어는 `음운`) · `유창성` · `운율` · `완성도` |
| 상세 섹션 | `분석 차트` · `글자별 분석` · `🤖 AI 발음 코치` · `📈 연습 기록` |
| 결과 화면 | `평균 점수` · `총 연습` · `평가 완료` · `평균 발음` · `✅ 가장 잘한 단어` · `⚠️ 더 연습이 필요해요` · `단어별 결과` |
| 결과 버튼 | `← 처음으로` · `다시 연습하기` |

### 3.2 컴포넌트 트리

```text
PronunciationPracticePage
├── SpeechNoticeDialog            (1회 동의 게이트)
├── BulbOverlay                   (변조 해설, 조건부 fixed 오버레이)
├── StartScreen
│   ├── SectionLabel × 3
│   ├── SelCard × 4
│   └── OptBtn × 3
├── PracticeScreen
│   ├── WordContent | SentContent (성조 뱃지 · 곡선 · 표준/실제 행 · 💡)
│   ├── PlayButton × 2
│   ├── 녹음 버튼 + .wbar × 12 + SpeechNoticeInfoButton
│   └── 평가 카드
│       ├── ringClass 총점 링 + EvalBars(Bar × 2~5)
│       └── detailOpen
│           ├── DetailSection「분석 차트」  → RadarChart
│           ├── DetailSection「글자별 분석」→ CharAnalysis (zh · word 전용)
│           ├── DetailSection「AI 발음 코치」
│           └── DetailSection「연습 기록」  → HistoryList
└── FinalScreen
    ├── StatBox × 3
    └── 단어별 결과 리스트
```

### 3.3 시나리오

**요청**: 「중국어 · 단어 발음 · 순서대로」로 `感觉`를 연습한다.

1. `extractTones("gǎnjué", 2)` → `[3, 2]` → 뱃지 `3성`(amber) · `2성`(emerald), 곡선 SVG는 3성 커브와 2성 상승선.
2. `🔊 표준 발음` 클릭 → `xf-tts`(vcn=`x4_xiaoyan`, speed=50) MP3 재생. 실패하면 브라우저 `zh-CN` 음성.
3. 마이크 첫 클릭 → 고지 다이얼로그 → 확인 → 녹음 시작(웨이브 바 애니메이션).
4. 정지 → `evaluating` → 점수 카드(`발음` · `성조`) + 총점 링.
5. `pronunciation-tone-change` → `has_change: false`(3성+2성은 변조 없음) → 💡 오버레이는 「변조 없음」.
6. `자세히 보기` → 레이더(발음 · 성조 · 정확도) + 글자별 분석 + `pronunciation-coach` 한국어 코칭.

**반례 1**: `你好`처럼 3성이 연속되면 `has_change: true`, `standard: "nǐ hǎo"`, `actual: "ní hǎo"`, `rule`에 「3성 연속 시 앞 3성이 2성으로」가 채워진다.
**반례 2**: 「한국어 · 단어 발음」에서는 성조 뱃지·곡선이 모두 사라지고 막대 라벨이 `음운`으로 바뀌며, TTS는 `typecast-tts`(ssfm-v30 · kor)로 나간다.

---

## ④ Context

- 데이터 출처: `songs` 테이블의 분석 결과(단어장, 가사). 부족하면 `normalize-lyrics` → `analyze-song` 순으로 보충하며, 그동안 시작 버튼은 로딩 상태로 잠긴다.
- 시크릿: 讯飞 `IFLYTEK_APP_ID` / `IFLYTEK_API_KEY` / `IFLYTEK_API_SECRET`(평가 + 중국어 합성 공유), Typecast `TYPECAST_API_KEY`(+ 선택 `TYPECAST_VOICE_ID_KO`). 미설정 시 TTS는 `fallback: true`로 브라우저 음성에 위임하고, 평가는 한국어 안내 문구를 반환한다.
- 讯飞 오류코드는 한국어로 번역해 노출한다: `11200` 발음인/기능 미개통, `11201` 서비스 권한·사용량 초과, `11202/11203` 유량 초과, `10005` 서비스 미개통.
- 모든 Edge Function은 CORS 헤더 필수, `supabase/config.toml`에 `verify_jwt = false`. 백엔드 오류도 200 + `error` 필드로 내려 프론트가 예외로 죽지 않게 한다.
- 개인정보: 녹음 음원은 서버에 저장하지 않고 평가 요청 본문으로만 전달한다. 1회 동의 사실만 브라우저 `localStorage`에 남긴다.
- 색상은 하드코딩 금지 원칙에 예외적으로 성조 팔레트만 상수 테이블로 허용한다(학습 규약상 성조 색은 고정 의미를 갖는다). 그 외 색은 semantic token(`bg-card` · `text-muted-foreground` · `hsl(var(--primary))`)만 사용한다.

---

## ⑤ Acceptance & Output

- [ ] `start | prac | final` 3-Phase가 모두 동작하고, 뒤로가기가 `current`·`evalData`·`detailOpen`·`coachData`를 초기화하며 TTS를 정지한다.
- [ ] `lang × mode × order` 12조합이 모두 진입 가능하거나, 데이터 부족 시 시작 버튼이 `데이터가 없습니다`로 잠긴다.
- [ ] `splitPinyinToSyllables("gǎnjué")` → `["gǎn","jué"]`, `ng` 모호 케이스에서 `g`가 다음 음절 초성으로 넘어간다.
- [ ] 성조 부호가 없는 음절은 5(경성)로 판정된다.
- [ ] 첫 녹음 시도에서 동의 다이얼로그가 정확히 1회 뜨고, 이후에는 뜨지 않으며 ⓘ 아이콘은 상시 노출된다(`localStorage.speech-eval-notice-ack === "1"`).
- [ ] `lang==="ko"`이면 성조 뱃지·곡선·`글자별 분석` 섹션이 렌더되지 않고 막대 라벨이 `음운`으로 바뀐다.
- [ ] TTS 요청이 `ko` → `typecast-tts`, `zh` → `xf-tts`로 라우팅되고, 실패 시 브라우저 음성으로 폴백해 버튼이 죽지 않는다.
- [ ] 동일 `text+lang+speed`의 2회차 낭독은 네트워크 요청 0건(캐시 히트).
- [ ] 변조 해설이 `localStorage(tc2:)`에 캐시되고, 429에서 최대 3회 지수 백오프 재시도하며, 다음 문제를 프리페치한다.
- [ ] 코칭 코멘트는 `자세히 보기`를 열 때만 호출된다(초기 렌더에서 호출 0건).
- [ ] 레이더 축 개수가 단어 3개 / 문장 5개로 달라진다.
- [ ] 무한 모드에서는 `final`로 자동 전환되지 않고 종료 버튼으로만 결과 화면에 진입한다.
- [ ] 언마운트 시 TTS·타이머·MediaRecorder·마이크 트랙이 모두 정리된다(트랙 `readyState === "ended"`).
- [ ] `grep -n "bg-\[#" PronunciationPracticePage.tsx` 결과 0건(성조 팔레트는 `hsl(...)` 상수 테이블로만 존재).

**출력 형식**: 위 ②2.1의 파일 순서(1→7)대로 전체 코드만 출력한다. 설명 산문·요약·사과·다른 파일 수정은 금지.
