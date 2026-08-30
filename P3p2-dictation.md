# P3p2 · 받아쓰기 (Dictation) 재현 프롬프트

> 본 문서는 `docs/thesis/appendix/prompts/00-template.md`가 정의한 5-Section 표준(Identity · Instructions · Examples · Context · Acceptance & Output)을 그대로 따른다. 이론적 근거(OpenAI 개발자 메시지 4-요소, Lovable 5 실천 원칙, IEEE 830 §4.3.6, Cohn 2004 §6)는 00-template.md에 정리되어 있으며 본 문서에서 재게시하지 않는다.
> **범위**: 곡 분석 다이얼로그(P3j) → 「연습」 탭의 **받아쓰기(Dictation)** 게임 한 페이지. 소스: `src/components/songs/DictationPage.tsx`(802행) + Edge Function `supabase/functions/dictation-explain/index.ts`. 다른 연습 모듈(퀴즈/발음/작문)은 P3p1·P3p3·P3p4에서 다룬다.

---

## ① Identity

당신은 한중 이중언어 교육 UX 라이터를 겸하는 시니어 프론트엔드 엔지니어이다. 기술 스택은 React 18 + Vite 5 + Tailwind CSS v3 + shadcn/ui + Supabase JS v2로 고정된다. 「멜로디 클래스(멜로디 클래스)」는 대한민국 대학의 중국어·한국어 학습자를 대상으로 하며, 본 페이지의 카피·오디오·해설 언어는 모두 한국어를 기본값으로 하되 학습 대상 언어(정답 텍스트)만 학습자가 선택한 `lang(ko|zh)`에 따라 전환된다.

---

## ② Instructions

### 2.1 산출물 (Prompt by Component, Not Page)

정확히 아래 2개 파일만 생성/수정한다. 파일 트리 외의 파일은 절대 손대지 않는다.

1. `src/components/songs/DictationPage.tsx` — 802행 단일 컴포넌트. 3-Phase 상태기(`start | game | final`) · 데이터 로딩 · YouTube 숨김 플레이어 · TTS · 힌트 3-단계 · 콤보 · 해설 페치·캐시를 모두 포함.
2. `supabase/functions/dictation-explain/index.ts` — `gpt-4o-mini` 기반 해설 생성 Edge Function. `{main, point, cross}` 3-필드 JSON 반환. CORS 필수, 429/402 한국어 메시지 매핑.

### 2.2 원자적 UI 규칙 (Speak Atomic)

**전체 컨테이너**: `max-w-[600px] mx-auto`. 세 화면 공통 `animate-in fade-in duration-200`.

**Phase 1 · Start Screen** (`phase === "start"`):
- 상단 좌측 뒤로가기 버튼: `ArrowLeft(15) + " 연습"`, `text-muted-foreground → hover:text-foreground`.
- 카드: `bg-card border border-border rounded-2xl p-6 shadow-sm`.
- 헤더 블록: 이모지 `🎧`(text-4xl mb-2.5) → 제목 `받아쓰기`(text-xl font-bold) → 서브라인 `{songTitle} — {artistName}`(text-sm text-muted-foreground).
- 섹션 라벨 스타일: `text-[11px] font-bold text-muted-foreground uppercase tracking-wider mb-2.5`.
- **모드 선택**: 2열 그리드(`grid-cols-2 gap-2.5`), 카드 두 장.
  - `lyric` 카드: 이모지 `🎵`, 제목 `가사 받아쓰기`, 설명 `노래 구간을 듣고\n가사를 입력하세요`.
  - `word` 카드: 이모지 `📝`, 제목 `단어 받아쓰기`, 설명 `발음을 듣고\n단어를 입력하세요`.
  - 카드 스타일: `rounded-xl p-4 border-2`, 선택 시 `border-primary bg-primary/5`, 미선택 `border-border bg-muted/30 hover:border-primary/50`.
  - `disabled`(해당 데이터 0건) 카드는 `opacity-40 cursor-not-allowed`이며 클릭이 무효.
- **문제 수(난이도)**: 3개의 pill 버튼 `flex-1 py-2.5 px-2 rounded-lg border-[1.5px]`.
  - `5` → `쉬움 · 5문제`, `10` → `보통 · 10문제`, `15` → `어려움 · 15문제`.
  - 재생 제한은 난이도에 따라 결정: `maxPlay = difficulty===5 ? 999 : difficulty===10 ? 3 : 1`. 999는 무제한.
- **언어 선택**: `ko` → `한국어`, `zh` → `中文`. 동일 pill 스타일.
- 시작 버튼: `w-full py-3 text-[15px] font-bold`, 라벨 `시작하기 →`.

**Phase 2 · Game Screen** (`phase === "game"`):
- 상단 헤더 좌측: `current+1 / total`과 모드 라벨(`가사 받아쓰기` | `단어 받아쓰기`) — `text-[11px] text-muted-foreground`, 아래 `<Progress value={(current/total)*100} className="h-1.5" />`.
- 상단 헤더 우측: 두 개의 스탯 셀. `점수`(text-lg font-bold) / `콤보`(text-lg font-bold **text-amber-500**, 값은 `x{combo}`).
- **문제 카드**: `bg-card border border-border p-5 shadow-sm`. 미제출 시 `rounded-[14px]`, 제출 후에는 `rounded-t-[14px] rounded-b-none`(피드백 카드가 아래에 붙기 위함).
- 메타 로우: 좌측 pill `가사 | 단어` (`text-[11px] font-bold text-primary bg-primary/10 px-2.5 py-0.5 rounded-full`) + 우측 `renderPlayDots()`.
- **재생 도트**: 라벨 `재생` + 최대 3개의 도트(`w-[7px] h-[7px] rounded-full`). 남은 재생 수는 `bg-primary`, 소진된 것은 `bg-border`. `maxPlay ≥ 999`(무제한)이면 전체 컨테이너 `opacity-30`.
- **재생 버튼 2개**(`flex gap-2.5`):
  - `노래로 듣기` — `Music(14)` + 라벨. 조건: `mode==="lyric" && q.start!=null && videoId` 일 때만 활성. 클릭 시 `seekTo(q.start)` 실행 후 `q.dur ?? 5`초 뒤 `pauseYT()`. 재생 중 라벨은 `재생 중...`. 재생 카운트 +1.
  - `일반 朗読` — `Volume2(14)` + 라벨. `SpeechSynthesisUtterance` 사용, 단어 모드는 항상 `zh-CN`, 가사 모드는 `lang==="ko" ? "ko-KR" : "zh-CN"`, `rate=0.85`. 재생 중 라벨 `읽는 중...`. 노래 재생 중이면 먼저 `stopSongPlayback()` 실행.
  - 활성 상태 색: `border-primary text-primary bg-primary/5`. 비활성: `border-border bg-muted/30 text-muted-foreground hover:border-primary hover:text-primary`.
- **힌트 pill 3개** (`flex gap-1.5 flex-wrap`):
  - Lv1 `첫 글자 −10점` → `첫 글자: {answer[0]}...`, 점수 감산 10.
  - Lv2 `글자 수 −20점` → `글자 수: {answer.length}글자`, 감산 20.
  - Lv3 `정답 보기 −50점` → `정답: {answer}`, 감산 50, 동시에 `inputVal`을 정답으로 채운다.
  - pill 스타일: `px-2.5 py-1 rounded-full border text-[11px] font-medium`, hover `border-amber-500 text-amber-500`, 사용됨/제출후 `opacity-30 cursor-not-allowed`.
- 힌트 표시 박스: `bg-amber-50 dark:bg-amber-500/10 border border-amber-200 dark:border-amber-500/20 rounded-lg px-3.5 py-2.5 text-sm text-amber-600 font-medium mb-3.5 tracking-wide`.
- 입력창: `<Input>` `mb-3 text-[15px] py-3 bg-muted/30`, placeholder `들은 내용을 입력하세요...`. Enter 키로 제출. 제출 후 정답이면 `border-green-500 bg-green-50/dark:bg-green-500/10`, 오답이면 `border-red-500 bg-red-50/dark:bg-red-500/10`.
- 액션 로우(`flex gap-2.5`): `제출`(flex-1 py-3 text-sm font-bold) + `건너뛰기 →`(px-4 py-3 border-[1.5px] border-border rounded-[10px] text-[13px]).
- **피드백 카드** (제출 후 슬라이드다운, `animate-in slide-in-from-top-2 duration-300`, `border border-t-0 rounded-b-[14px] p-5`):
  - 배경 · 테두리: 정답 `bg-green-50/dark:bg-green-500/5 border-green-200/dark:border-green-500/20`; 오답 `bg-red-50/dark:bg-red-500/5 border-red-200/dark:border-red-500/20`; 건너뜀 `bg-muted/30 border-border`.
  - 상단: 이모지(정답 `✅` · 오답 `❌` · 건너뜀 `⏭️`) + 텍스트(`정답` / `오답` / `건너뜀`, 각각 `text-green-600 / text-red-600 / text-muted-foreground`, `text-sm font-bold`). 정답이며 `combo>2`이면 우측에 `🔥 {combo}연속 콤보!`(text-xs text-amber-500). 맨 오른쪽 `+{feedbackScore}점`(text-lg font-bold text-primary).
  - **비교 렌더**: 건너뜀이면 라벨 `정답` + 정답 텍스트만 표시. 그 외에는 두 줄로 표시:
    1. 라벨 `정답`(green pill) + `renderComparison(inputVal, answer)` — 정답 문자열을 문자 단위로 순회하며 `input[i]===answer[i]` 이면 `text-foreground`, 아니면 `text-red-500 underline underline-offset-2`.
    2. 학습자 입력이 있으면 라벨 `입력`(red pill) + 원문 입력.
  - **해설 블록**(`border-t border-black/[0.07] dark:border-white/[0.07] pt-3.5`): 헤더 `📖 해설`(text-[11px] font-bold uppercase). 로딩 중 `Loader2(16) + "해설을 생성하고 있습니다..."`. 결과는 3-필드:
    - `main` — 본문(text-[13px] leading-relaxed text-muted-foreground).
    - `point` — 좌측 border 3px: 정답이면 `border-l-green-500`, 오답이면 `border-l-red-500`. 카드 스타일 `bg-card rounded-lg px-3.5 py-2.5 mt-2.5`.
    - `cross` — 좌측 border 3px `border-l-green-500`, 카드 스타일 동일.
  - 하단: `현재 문제 인덱스+1 >= 총 문항 수` 여부에 따라 `결과 보기 →` 또는 `다음 문제 →` 버튼.

**Phase 3 · Final Screen** (`phase === "final"`):
- 카드: `bg-card border border-border rounded-2xl p-6 shadow-sm`.
- 헤더: 최종 아이콘(text-[44px]) → 총점(text-5xl font-bold text-primary) → 라벨 `총점`.
  - 아이콘 임계: `totalAcc >= 90 → 🏆`, `>=70 → 🥈`, `>=50 → 🥉`, `그 외 → 💪`.
- 통계 3-열(`grid grid-cols-3 gap-2.5`): `정확도({totalAcc}%)`, `정답({totalCorrect}/{results.length})`, `최고 콤보(🔥{maxCombo} · text-amber-500)`. 각 셀 `bg-muted/30 rounded-[10px] p-3.5 text-center`.
- 문제별 결과 리스트: `results.map`로 각 행 `flex items-center justify-between px-3.5 py-2.5 rounded-lg text-[13px]`. 정답 `bg-green-50/dark:bg-green-500/5`, 오답 `bg-red-50/dark:bg-red-500/5`, 건너뜀 `bg-muted/30`. 좌측 상태 마커(`⏭️ | ✓ | ✗`) + 정답 텍스트(`max-w-[220px] truncate`), 우측 `+{score}점` 또는 `건너뜀`.
- 하단 액션(`flex gap-2.5`): `전체 다시하기`(variant outline, phase를 `start`로 되돌리며 `expCache.current={}`), `틀린 문제만`(내부에서 `retryWrong()`; 모든 결과가 `accuracy>=90`이면 disabled).

### 2.3 원자적 데이터·로직 규칙

- **데이터 소스**: `songs.video_id`와 `song_analyses.{lyrics_with_pinyin, word_list, full_analysis.timed_entries}`.
- **가사 문항 구축**: `lyrics_with_pinyin`에서 `chinese && korean`이 모두 있는 항목만 사용. 각 라인에 대해 `full_analysis.timed_entries` 중 정규식 `/[\s,，。！？、；：""''（）《》\-–—…·.!?;:'"()\[\]]/g`으로 punctuation을 제거한 뒤 `zh.includes(text) || text.includes(zh) || ko.includes(text) || text.includes(ko)` 매칭. 매칭되면 `{start, dur}`를 저장, 없으면 `start=undefined`(→ 해당 문항의 `노래로 듣기` 버튼은 자동 disabled).
- **단어 문항 구축**: `word_list` 항목에서 `{ko: w.meaning_ko, zh: w.word, pinyin: w.pinyin}`. 문항 셔플 후 `min(difficulty, pool.length)`개 사용.
- **정확도 계산**: `calcAccuracy(input, answer)` — trim 후 정확히 일치하면 100. 아니면 `min(a.length,b.length)` 만큼 순회하며 위치 일치 카운트, `round((ok/b.length)*100)`. 90 이상을 정답으로 판정.
- **점수·콤보**: `base = round(accuracy*combo)` (accuracy 백분위 × combo). `gain = max(0, base - hintPenalty)`. 정답이면 `combo = min(combo+1, 5)`, 오답이면 `combo = 1`. `maxCombo`는 게임 전체 최댓값.
- **재생 카운트**: `노래로 듣기`·`일반 朗読` 각각 1회로 계산. `playCount >= maxPlay`이면 재생 버튼 disabled(무제한이면 예외).
- **YouTube 숨김 플레이어**: `<div id="dictation-yt-player" className="w-0 h-0 overflow-hidden absolute" />`를 항상 렌더링. `useYouTubePlayer(videoId, ytContainerId)` 훅으로 `seekTo/pause` 확보. 노래 재생은 `q.dur ?? 5`초 후 자동 pause. 페이지 언마운트/모드 전환 시 `stopSongPlayback()`.
- **해설 페치**: 문항 인덱스+언어 조합을 캐시 키(`${current}_${lang}`)로 사용. `supabase.functions.invoke("dictation-explain", { body })`. 실패 시 `{main:"해설을 불러오지 못했습니다.", point:"", cross:""}` 폴백을 캐시.
- **강제 제약**: `bg-[#…]`, `text-white`, `bg-black`, HEX 하드코딩 전면 금지. 모든 색·경계·라운드는 shadcn 토큰(primary, muted, border, muted-foreground, foreground, card, amber-500, green-500/600, red-500/600) 또는 tailwind 표준 팔레트만 사용. 페이지 내 `<h1>`은 두지 않는다(부모 다이얼로그에 이미 존재). 순수 한국어 카피 유지, lorem ipsum 절대 금지.

---

## ③ Examples

### 3.1 확정 카피 표 (Design with Real Content)

| 위치 | 텍스트 |
|---|---|
| 뒤로가기 | ` 연습` (아이콘 `ArrowLeft` 뒤) |
| 헤더 이모지 | `🎧` |
| 페이지 제목 | `받아쓰기` |
| 서브라인 | `{songTitle} — {artistName}` |
| 섹션 · 모드 | `모드 선택` |
| 모드 카드 1 (lyric) | `🎵` / `가사 받아쓰기` / `노래 구간을 듣고\n가사를 입력하세요` |
| 모드 카드 2 (word) | `📝` / `단어 받아쓰기` / `발음을 듣고\n단어를 입력하세요` |
| 섹션 · 난이도 | `문제 수` |
| 난이도 버튼 | `쉬움 · 5문제` / `보통 · 10문제` / `어려움 · 15문제` |
| 섹션 · 언어 | `언어` |
| 언어 버튼 | `한국어` / `中文` |
| 시작 CTA | `시작하기 →` |
| 게임 진행 라벨 | `{current+1} / {total}` · `가사 받아쓰기` / `단어 받아쓰기` |
| 스탯 라벨 | `점수` · `콤보` |
| 문항 pill | `가사` / `단어` |
| 재생 라벨 | `재생` |
| 재생 버튼 | `노래로 듣기` · `재생 중...` · `일반 朗読` · `읽는 중...` |
| 힌트 pill | `첫 글자 −10점` · `글자 수 −20점` · `정답 보기 −50점` |
| 힌트 표시 | `첫 글자: {ch}...` · `글자 수: {n}글자` · `정답: {answer}` |
| 입력 placeholder | `들은 내용을 입력하세요...` |
| 액션 | `제출` · `건너뛰기 →` |
| 피드백 상태 | `정답` · `오답` · `건너뜀` |
| 콤보 배지 | `🔥 {combo}연속 콤보!` (combo > 2) |
| 점수 배지 | `+{gain}점` |
| 비교 라벨 | `정답` · `입력` |
| 해설 헤더 | `📖 해설` |
| 해설 로딩 | `해설을 생성하고 있습니다...` |
| 해설 실패 폴백 | `해설을 불러오지 못했습니다.` |
| 다음 CTA | `다음 문제 →` / `결과 보기 →` |
| 결과 화면 라벨 | `총점` · `정확도` · `정답` · `최고 콤보` · `문제별 결과` |
| 결과 마커 | `⏭️` · `✓` · `✗` |
| 최종 아이콘 임계 | `🏆(≥90)` · `🥈(≥70)` · `🥉(≥50)` · `💪(<50)` |
| 결과 하단 | `전체 다시하기` · `틀린 문제만` · `건너뜀` · `+{score}점` |

### 3.2 파일 트리 (Use Prompt Patterns for Layouts)

```
src/
└── components/
    └── songs/
        └── DictationPage.tsx           # 802행 · 3-Phase 상태기 단일 파일
supabase/
└── functions/
    └── dictation-explain/
        └── index.ts                    # gpt-4o-mini · {main,point,cross} JSON
```

컴포넌트 트리:

```
DictationPage
├── HiddenYouTubePlayer (div#dictation-yt-player · w-0 h-0)
├── StartScreen
│   ├── BackLink
│   ├── HeaderBlock (emoji · title · subline)
│   ├── ModePickerGrid (lyric card · word card)
│   ├── DifficultyPills (5 · 10 · 15)
│   ├── LanguagePills (ko · zh)
│   └── StartButton
├── GameScreen
│   ├── HeaderRow (progress + score/combo)
│   ├── QuestionCard
│   │   ├── MetaRow (pill + PlayDots)
│   │   ├── PlayButtons (노래로 듣기 · 일반 朗読)
│   │   ├── HintPills × 3
│   │   ├── HintDisplay (조건부)
│   │   ├── Input (Enter=submit)
│   │   └── ActionRow (제출 · 건너뛰기)
│   └── FeedbackCard (조건부, submit 후 슬라이드다운)
│       ├── StatusRow (icon · label · combo · +score)
│       ├── ComparisonBlock (renderComparison 문자단위)
│       ├── ExplanationBlock (📖 해설 · main · point · cross)
│       └── NextButton
└── FinalScreen
    ├── HeaderBlock (finalIcon · totalScore · '총점')
    ├── StatsGrid × 3 (정확도 · 정답 · 최고 콤보)
    ├── ResultList (문제별)
    └── ActionRow (전체 다시하기 · 틀린 문제만)
```

### 3.3 Edge Function 요청·응답 예시

요청 body:

```json
{
  "song_title": "月亮代表我的心_달이 내 마음을 대신해요",
  "artist_name": "邓丽君",
  "language": "ko",
  "mode": "lyric",
  "answer": "달이 내 마음을 대신해요",
  "input": "달이 나의 마음을 대신해요",
  "accuracy": 83,
  "result": "오답"
}
```

응답 body(반드시 유효 JSON):

```json
{
  "main": "'대신하다'는 다른 대상을 대체한다는 뜻으로, 이 문장에서는 화자의 감정을 달이 상징적으로 표현한다는 것을 나타냅니다.",
  "point": "'나의'와 '내'는 같은 뜻이지만, 이 노래에서는 리듬을 위해 '내'가 사용되었습니다.",
  "cross": "중국어 '代表'는 한국어의 '대신하다·상징하다' 두 뉘앙스를 모두 담습니다."
}
```

---

## ④ Context

### 4.1 프로젝트 맥락

「멜로디 클래스」는 대학 강의실에서 사용하는 K-Chinese/K-Korean 노래 학습 플랫폼이다. 곡 하나의 분석 다이얼로그(P3j)는 6개의 탭을 노출하며, 그 중 「연습」 탭은 4개의 서브 카드(퀴즈 · 받아쓰기 · 발음 · AI 작문)를 진입점으로 갖는다. 본 문서(P3p2)의 범위는 **받아쓰기 카드 클릭 이후 로드되는 페이지 한 개**로 국한되며, 이는 `DictationPage` props `{songId, songTitle, artistName, onBack}`로 다이얼로그에서 마운트된다.

### 4.2 Lovable Cloud 후경 (Build with Lovable Cloud in Mind)

- **인증**: 이 페이지는 로그인 사용자만 접근하는 상위 라우트 아래에서 열린다. 별도의 세션 체크는 불필요하나 `supabase.functions.invoke`가 자동으로 Authorization 헤더를 붙인다.
- **로딩 상태**: `loading=true`이면 페이지 중앙에 `Loader2` 스피너만 렌더링, 숨김 YT 플레이어는 유지.
- **빈 상태**: `lyrics.length===0`이면 lyric 카드가 disabled, `words.length===0`이면 word 카드가 disabled. 두 데이터가 모두 없으면 사용자가 시작할 수 없다(현재 사양은 별도 empty screen을 렌더하지 않는다 — 시작 카드가 모두 disabled인 상태가 empty의 역할을 겸함).
- **에러 상태**: 해설 페치 실패는 폴백 문자열로 흡수(캐시에 저장하여 재시도 방지). 429/402는 Edge Function이 한국어 메시지로 매핑하여 반환하며, 프론트는 실패 시 폴백 카드만 노출.
- **성공 상태**: 통상 진행 흐름은 위 Phase 2 → Phase 3.

### 4.3 데이터 계약

본 페이지는 신규 테이블을 만들지 않는다. 기존 테이블만 읽는다:

- `songs (id, video_id)` — YouTube 재생용.
- `song_analyses (song_id, lyrics_with_pinyin, word_list, full_analysis)` — 문항 소스. `full_analysis.timed_entries: {start:number(초), dur:number(초), text:string}[]`.

이 두 테이블의 RLS·GRANT는 P3a/P3j에서 이미 정의되어 있으며 본 문서 범위에서는 변경하지 않는다.

### 4.4 Edge Function 계약

`supabase/functions/dictation-explain/index.ts` 사양:

- 요청: `{song_title, artist_name, language: "ko"|"zh", mode: "lyric"|"word", answer, input, accuracy, result}`.
- 시스템 프롬프트(고정 문자열): `"당신은 음악 교육 플랫폼의 언어 학습 해설 AI입니다. 반드시 유효한 JSON만 출력하고 다른 텍스트는 절대 포함하지 마세요."`.
- 사용자 프롬프트는 `langLabel(한국어|중국어)`·`modeLabel(가사|단어)`을 치환한 뒤 3-필드 JSON 스키마를 명시적으로 지시한다.
- 모델: `gpt-4o-mini`, `max_tokens: 1000`, temperature 기본.
- 응답 파싱: markdown 코드펜스(```json … ```)를 제거한 후 `JSON.parse`. 파싱 실패 시 `{main:"해설을 생성하지 못했습니다.", point:"", cross:""}` 반환.
- 오류 매핑: HTTP 429 → `요청이 너무 많습니다. 잠시 후 다시 시도해주세요.` · HTTP 402 → `AI 크레딧이 부족합니다. 충전 후 다시 시도해주세요.` · 그 외 → 500 + `{main:"해설을 불러오지 못했습니다.", point:"", cross:""}`.
- CORS 헤더는 모든 응답(성공·에러·OPTIONS 프리플라이트)에 포함.

---

## ⑤ Acceptance & Output

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6 · 정량 임계값)

1. `wc -l src/components/songs/DictationPage.tsx` = **802 ± 20**.
2. `wc -l supabase/functions/dictation-explain/index.ts` ≤ **120**.
3. `grep -E "bg-\[|text-white|bg-black" src/components/songs/DictationPage.tsx | wc -l` = **0**.
4. `grep -c "phase === " src/components/songs/DictationPage.tsx` ≥ **3** (start · game · final 분기).
5. 정확도 임계: `calcAccuracy` 결과 ≥ 90인 케이스가 100% `feedbackType==="correct"`로 분기해야 함.
6. 콤보 상한: `combo`는 어떤 진행에서도 **5를 초과하지 않는다**(min(combo+1, 5)).
7. 힌트 감점: Lv1=10, Lv2=20, Lv3=50 — 세 값의 합계 감점을 사용하면 gain은 **0 이하로 내려가지 않는다**(`Math.max(0, base - hintPenalty)`).
8. 재생 도트: `maxPlay < 999`일 때 화면에 렌더되는 도트 수 ≤ **3**.
9. 최종 아이콘 임계: `totalAcc>=90→🏆`, `>=70→🥈`, `>=50→🥉`, 그 외 `💪` 4개 케이스가 정확히 존재한다.
10. 해설 캐시 재적중률: 동일 `${current}_${lang}` 키로 두 번째 이상 호출 시 네트워크 요청 건수 = **0**(캐시 히트).
11. Edge Function 응답 JSON 필드: 정확히 `{main, point, cross}` **3개** 문자열 키만 존재.
12. Lighthouse Best Practices ≥ **90**, CLS ≤ **0.05**(다이얼로그 내부 스크롤 컨테이너 안에서 측정).
13. 카피의 한국어 순도: `docs/thesis/appendix/prompts/P3p2-dictation.md` 「3.1 확정 카피 표」의 어느 라벨도 영문/lorem/중문(단, `中文` 언어 스위치 라벨과 `일반 朗読`의 `朗読`는 예외)으로 대체되지 않는다 — `grep -E "lorem|TODO|FIXME|placeholder텍스트" src/components/songs/DictationPage.tsx | wc -l` = **0**.

### 5.2 Output Format

LLM(또는 Lovable)은 다음 두 파일을 이 순서로만 반환한다. 파일 사이·앞뒤에 설명·사과·마크다운 헤더·주석 요약을 절대 포함하지 않는다.

1. `src/components/songs/DictationPage.tsx`
2. `supabase/functions/dictation-explain/index.ts`
