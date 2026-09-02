# P3k · 가사 탭(Lyrics / Audio / Video) 재현 프롬프트

> **본 프롬프트는 P3 시리즈(P3a–P3q)의 11/17.**
> **적용 대상**: `src/components/songs/LyricsTab.tsx`, `AudioLyricsTab.tsx`, `VideoLyricsTab.tsx` — 세 가지 가사 뷰(정적 텍스트 · Suno 오디오 하이라이트 · YouTube 하이라이트). 각 뷰는 **독립적인 컴포넌트**이며 상위(P3j)의 `<SongAnalysisDialog>` 「가사」 서브탭이 song 데이터 형태에 따라 하나를 선택해 렌더한다.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**
> **개정 이력**: 초판은 낭독을 브라우저 `speechSynthesis` 로 서술했으나, 현재 플랫폼은 **한국어 Typecast · 중국어 讯飞(iFlytek)** 서버 TTS 를 `speakTts` 로 라우팅한다. 본 개정판은 현재 코드를 기준으로 한다.

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages: Identity, Instructions, Examples, Context.* OpenAI Platform Documentation. Retrieved July 12, 2026, from https://platform.openai.com/docs/guides/prompt-engineering
2. **Lovable. (n.d.).** *Prompting best practices.* Lovable Documentation. Retrieved July 12, 2026, from https://docs.lovable.dev/prompting/prompting-one — 5개 실천 원칙(Prompt by Component · Speak Atomic · Design with Real Content · Use Prompt Patterns for Layouts · Build with Lovable Cloud in Mind) 채택 근거.
3. **IEEE. (1998).** *IEEE Recommended Practice for Software Requirements Specifications* (IEEE Std 830-1998), §4.3.6 "Verifiable", p. 7. IEEE.
4. **Cohn, M. (2004).** *User Stories Applied: For Agile Software Development*, Ch. 6, pp. 67–74. Addison-Wesley.

---

## ① Identity (신원)

당신은 시니어 프론트엔드 엔지니어 겸 한중 이중언어 교육 UX 라이터입니다. React 18 + Vite 5 + Tailwind CSS v3 + shadcn/ui + Supabase(JS v2) 로, YouTube IFrame API · Web Audio · Suno `alignedWords` · 클라이언트-측 자막 추출 · **서버 TTS(Typecast / iFlytek) + 브라우저 음성 fallback** 을 조합해 세 가지 가사 렌더러를 만듭니다. 대상은 대한민국 대학의 K-Chinese/K-Korean 교사·학습자이며, 모든 UI 카피는 순수 한국어입니다.

## ② Instructions

### 2.1 산출물 (Prompt by Component, Not Page)

- **`src/components/songs/LyricsTab.tsx`** — 오디오/비디오 없는 **정적 가사 뷰**. 표시 모드·순서 · 병음 · 폰트 · 문체 분석 · 카드 저장 · 편집 담당.
- **`src/components/songs/AudioLyricsTab.tsx`** — Suno 로 생성된 **AI 곡 전용**(자체 `audio_url` + `alignedWords`). `SongPlayer` 임베드 + 라인 하이라이트 + 배경 영상 새로고침 + 정렬 복구 담당.
- **`src/components/songs/VideoLyricsTab.tsx`** — **YouTube 영상 곡 전용**(`video_id`). IFrame 플레이어 + 서버/클라이언트 자막 fallback + fuzzy time-map 매칭 담당.
- 세 컴포넌트는 **독립**이며 공유 훅을 만들지 않는다. 상위 `<SongAnalysisDialog>`(P3j) 가 song 형태로 분기한다:
  - `is_ai_generated && audio_url` → `<AudioLyricsTab>`
  - `video_id` → `<VideoLyricsTab>`
  - 그 외 → `<LyricsTab>`
- 의존 모듈(신규 생성 금지, 그대로 사용):
  - `src/hooks/useXfTts.ts` — `speakTts(text, lang)` · `stopTts()` · `useXfTts()` · `toTtsLang()`
  - `src/features/song-player/SongPlayer.tsx` (`embedded` 모드), `src/features/song-player/lines.ts` (`parseSunoLines`, `alignLyricsToSuno`)
  - `src/hooks/useYouTubePlayer.ts`, `src/hooks/useClientTranscript.ts`
  - `src/components/songs/LyricCardDialog.tsx`(카드 편집기), `src/components/songs/SongFeatureButtons.tsx`(P3n)

### 2.2 원자적 UI 규칙 (Speak Atomic)

#### 2.2.0 공통 TTS 라우팅 규칙 (세 컴포넌트 모두 동일)

- 낭독은 **반드시** `speakTts(text, lang)` 로만 호출한다. 컴포넌트에서 `new SpeechSynthesisUtterance` 를 직접 만들지 않는다.
- `speakTts` 내부 계약:
  - `toTtsLang(lang)` 이 `"ko" | "korean" | "ko-*"` → `"ko"`, 그 외 전부 → `"zh"` 로 정규화.
  - `"ko"` → Edge Function **`typecast-tts`**, `"zh"` → Edge Function **`xf-tts`** 호출. 요청 본문 `{ text, lang, speed }`.
  - 응답 `{ audio_base64 }` → `data:audio/mpeg;base64,…` 로 재생. 모듈-레벨 `Map` 캐시 키 = `` `${lang}:${speed}:${text}` ``, 200건 초과 시 전체 clear.
  - 함수 실패 또는 `audio_base64` 없음 → **브라우저 음성 fallback**(`lang = ko-KR | zh-CN`, `rate = 0.85`). 응답에 `fallback: true` 가 있으면 콘솔 경고도 남기지 않는다.
  - 긴 텍스트는 문장 경계(`。．.!?！？\n`) 기준 300자 이하로 분할 후 순차 재생. 새 재생 시작 전 이전 오디오와 `speechSynthesis` 큐를 모두 정지.

#### 2.2.1 `LyricsTab` (정적 뷰)

**Props**: `lyrics: LyricLine[]`, `onSave(lyrics)`, `readOnly?`, `language?: string`(기본 `"chinese"`), `songId?`, `songTitle?`, `songArtist?`, `songHskLevel?`.

**기본 순서 결정 규칙(중요)**:
```ts
const isZhSong = language !== "korean";
const isTopik  = !!songHskLevel && songHskLevel.toUpperCase().startsWith("TOPIK");
const koFirst  = isTopik || !isZhSong;         // TOPIK 분석이거나 한국어 곡이면 한국어 우선
const [order, setOrder] = useState<DisplayOrder>(koFirst ? "ko-zh" : "zh-ko");
```

**낭독 언어 결정 규칙(중요)**:
```ts
const speakLang = mode === "ko" ? "korean"
                : mode === "zh" ? "chinese"
                : order === "ko-zh" ? "korean" : "chinese";
```
버튼 클릭 시 우선 언어 텍스트가 비어 있으면 반대 언어 텍스트로 자동 fallback 하고, **실제 사용한 텍스트의 언어**를 `speakText` 에 넘긴다. 양쪽 다 비어 있으면 no-op.

**컨트롤 바**(`flex flex-wrap items-center gap-2 pb-3 border-b`), 좌→우 순서 고정:

1. **언어 모드 3-pill** (`mode: "zh" | "ko" | "both"`, 기본 `"both"`) — 라벨 `中文 / 한국어 / 둘 다`. 활성 pill `bg-[#243158] text-white`.
2. **표시 순서 2-pill** (`order: "zh-ko" | "ko-zh"`) — 라벨 `中→한 / 한→中`. `mode !== "both"` 이면 `disabled` + `opacity-40`.
3. **병음 토글** 단일 pill `병음` (`showPinyin`, 기본 `true`).
4. **폰트 크기** `A− {N} A+` — `MIN_FONT=11`, `MAX_FONT=22`, `DEFAULT_FONT=15`. 중앙 숫자 `bg-muted min-w-[28px]` 굵게.
5. **문체 분석 Popover**(`songId` 있을 때만) — 트리거 `✦ {현재 라벨} ▾`(기본 라벨 `스타일 분석`). 옵션 = `시적 번역`(poetic) / `직역`(literal) / `구어체 번역`(casual), 선택 상태일 때만 `해제` 추가. 선택 시 `translate-lyrics-style` 호출, 응답 `data.translations` 를 `styleCache[style]` 에 저장하고 **재선택 시 재호출 금지**. `"원문"` 옵션은 존재하지 않는다.
6. **카드 저장 버튼** `📤 카드로 저장`(`bg-[#3d6cb5] text-white`) → `selectMode` 진입 시 라벨 `✕ 취소` + `animate-pulse bg-red-500`.
7. **편집 버튼 그룹**(`!readOnly`, `ml-auto`): 정지 시 `수정`(Pencil), 편집 중 `취소` + `저장`. `저장` 은 `onSave(editLyrics)` 1회 호출 후 편집 종료.

**선택 모드 배너**(`selectMode === true`): `bg-[#3d6cb5]/10 border border-[#3d6cb5]/40 rounded-lg px-4 py-2.5`, 카피 `가사를 클릭해서 선택하세요 · <strong>{n}</strong>줄 선택됨` + `카드 만들기 →`(0줄 선택 시 disabled) + `취소`. `카드 만들기` 는 선택 인덱스를 오름차순 정렬해 `<LyricCardDialog selectedLines={[{kr, zh, py}]}>` 를 연다. 다이얼로그를 닫으면 `selectMode` 도 해제.

**가사 라인 카드**(`p-4 rounded-lg border bg-card space-y-1`):
- **읽기 모드** — `renderLines(line, mode, order, showPinyin, fontSize)`:
  - `zh`: `font-medium text-foreground`, `fontSize`
  - `pinyin`: `text-[#3d6cb5]`, `fontSize - 2` (`showPinyin` 이고 값이 있을 때만)
  - `ko`: `text-secondary-foreground`, `fontSize - 1`
  - 순서: `mode==="zh"` → zh+py / `"ko"` → ko / `"both" && zh-ko` → zh+py+ko / `"both" && ko-zh` → ko+zh+py.
  - 우측 `<Volume2>` 아이콘 버튼 → 위 낭독 규칙대로 `speakTts`.
  - 문체 결과가 있으면 `mt-2 pl-3 border-l-2 border-[#3d6cb5]/40 bg-[#243158]/5 rounded-r-md` 블록에 `{emoji} {styleResult}`(`fontSize - 1`). 로딩 중 `<Skeleton className="h-5 w-3/4 mt-2">`, 실패 시 `text-destructive` 로 `번역 실패. 다시 시도해 주세요.`.
- **편집 모드** — 3개 `<Input className="h-8">`(中文 / Pinyin / 한국어). 진입 시 `editLyrics = JSON.parse(JSON.stringify(lyrics))` 딥카피.
- **선택 모드** — 카드 전체 클릭 가능(`cursor-pointer hover:bg-muted/50`), 선택 시 `ring-2 ring-[#3d6cb5] bg-[#3d6cb5]/5`.

**문체 이모지**: `poetic → 🌸`, `literal → 📖`, `casual → 💬`.

#### 2.2.2 `AudioLyricsTab` (Suno AI 곡)

**Props**: `audioUrl`, `lyrics`, `alignedWords?: {word,start,end,success?}[]`, `bgVideoList?: string[] | {videoUrl}[]`, `songId?`, `analysisId?`, `language?`, `songTitle?`, `songArtist?`, `songHskLevel?`, `lyricsRaw?`, `isAiGenerated?`, `onNavigateToSong?`.

**레이아웃**: `flex gap-4`, 기본 `flex-col lg:flex-row`, `isExpanded` 이면 `flex-col`. 좌측은 `lg:w-1/2 shrink-0`(확장 시 `w-full`), 우측은 `flex-1`.

**좌측**: `<SongPlayer embedded audioUrl bgVideoList={bgVideos} onRefreshVideos title={songTitle} subtitle={songArtist || "AI 생성 노래"} onTimeUpdate onReady stageOverlay />` → 그 아래 순서로 (a) 타이밍 없음 안내, (b) 정렬 복구 배너, (c) `<SongFeatureButtons>`(songId 있을 때).

**우측**: `LangSwitcher` + 스크롤 라인 리스트.

**언어 스위처**: `Set<LangKey>`(`"korean" | "chinese" | "pinyin"`) 3-토글, 초기값 3개 모두 활성. **마지막 하나는 해제 불가**(`size <= 1` 이면 상태 유지). 라벨 `한국어 / 中文 / 拼音`. 활성 `bg-primary text-primary-foreground border-primary`. 우측 끝(`ml-auto`) `영상 확대 / 영상 축소` 토글(Maximize2/Minimize2).

**시간 정렬**:
- `sunoLines = parseSunoLines(alignedWords || [])` (Suno 항목은 라인 단위, `word` 가 `\n` 로 종료).
- `aligned = alignLyricsToSuno(lyrics, sunoLines)`.
- `hasTimed = sunoLines.length > 0`, `lineMismatch = hasTimed && sunoLines.length !== lyrics.length`.
- 활성 라인 = **이진 탐색**으로 `aligned[i].startS - 0.05 <= currentTime` 을 만족하는 최대 i.
- 활성 라인이 바뀌면 리스트 컨테이너를 `scrollTo({top: el.offsetTop - h/2 + el.clientHeight/2, behavior:"smooth"})` 로 중앙 정렬(페이지 스크롤 금지).

**타이밍 없음 안내**(`!hasTimed`): `⚠ 가사 타이밍 데이터가 없어 동기화가 제한됩니다.`

**정렬 복구 배너**(`lineMismatch && isAiGenerated && songId` 세 조건 동시 충족 시에만):
- 카피 `가사({lyrics.length}줄)와 타임스탬프({sunoLines.length}줄)가 일치하지 않습니다.` + `<Wrench> 정렬 복구` 버튼(진행 중 `<Loader2 animate-spin>` + `disabled`).
- `supabase.functions.invoke("sg-repair-aligned", { body: { song_id } })`. 성공 toast `정렬 복구 완료 / 페이지를 새로고침해 주세요.`, 실패 toast `복구 실패`(`destructive`).

**배경 영상 새로고침**(`onRefreshVideos`):
- `q = [songTitle, songArtist].filter(Boolean).join(" ").trim() || "nature sky sunlight"`.
- 1차 `sg-pixabay-videos` → `bgVideoList` 가 비면 `q = "nature sky sunlight people city"` 로 재시도.
- 결과가 있으면 `setBgVideos(list)` + `songs.bg_video_list` UPDATE(fire-and-forget).
- `refreshingBgRef` ref 뮤텍스로 재-엔트리 차단, 실패는 `console.warn` 만.

**stageOverlay(댄마쿠 자막)**: `absolute left-0 right-0 bottom-[12%] pointer-events-none`. 활성 라인의 `korean`(`bg-black/70 text-white text-base`) → `chinese`(`text-sm`) → `pinyin`(`bg-black/65 text-gray-200 text-[12px]`) 순으로 `activeLangs` 에 포함된 것만 표시.

**라인 리스트**: `overflow-y-auto`, `max-h-[60vh]`(축소) / `max-h-[40vh]`(확장).
- 좌측 `w-10 text-right font-mono text-[10px]` 에 `formatTime(startS)`(`m:ss`), 타임이 없으면 `{i+1}`.
- 활성 라인 `bg-primary/10 border-primary/30 shadow-sm scale-[1.01]`, 비활성 `border-transparent hover:bg-muted/50`.
- 라인 클릭 → 타임이 있을 때만 `apiRef.current?.seek(t)`.
- 라인 내부 primary 언어 = `language==="korean" ? ["korean","chinese","pinyin"] : ["chinese","korean","pinyin"]` 순서 중 `activeLangs` 에 있는 첫 언어. Primary 는 `text-base font-medium`(활성 시 `text-primary`), 나머지 `text-xs`(pinyin 은 `text-primary/70`).
- 우측 `🗣️ 낭독` 버튼: `e.stopPropagation()` 후 곡 언어(`language`) 기준 텍스트를 `speakTts`.

#### 2.2.3 `VideoLyricsTab` (YouTube 곡)

**Props**: `videoId`, `lyrics`, `timedEntries?`, `alignedWords?: {word,startS,endS}[]`, `songId?`, `analysisId?`, `language?`, `lyricsSource?`, `songTitle?`, `songArtist?`, `songHskLevel?`, `lyricsRaw?`, `onNavigateToSong?`.

**플레이어**: `const { currentTime, seekTo } = useYouTubePlayer(videoId, \`yt-player-${videoId}\`)`. 컨테이너 `relative aspect-video bg-black rounded-lg overflow-hidden`, 자막 오버레이는 `bottom-[52px] z-10 pointer-events-none`(한국어 `text-lg` · 중국어 `text-base` · 병음 `text-[13px]`).

**자막 소스 우선순위**:
1. `alignedWords` 가 있으면 `alignedTimed = alignedWords.map(w => ({start: w.startS, dur: max(0.1, endS-startS), text: word.replace(/\n+$/,"")}))`.
2. 아니면 `useClientTranscript(videoId, songId, analysisId, serverTimedEntries, language)`.
- `useAligned = alignedTimed.length > 0`, `isManualSRT = useAligned || lyricsSource === "custom_timed"`.
- `isManualSRT` 이면 `buildTimeMap` 을 **호출하지 않고**(`timeMap = []`) 인덱스 1:1 매핑을 쓴다.

**`buildTimeMap(lyrics, timed)` 규칙**(fuzzy 경로 전용):
- 자막 항목을 1.5초 초과 공백 기준으로 그룹핑(`{start, text}`).
- 각 가사 줄은 `normalize()`(공백·한중 문장부호 제거 + 소문자화)된 `line.chinese` 를 기준으로 최고 점수 그룹 선택. 포함 관계면 `shorter.length / target.length`, 아니면 문자 일치율 × 0.8. 이미 쓰인 그룹은 × 0.6 페널티.
- 최고 점수 ≥ 0.3 이면 해당 그룹 `start`, 아니면 `-1`.
- 매칭 수가 `lyrics.length * 0.3` 미만이면 **전체를 인덱스 비율 fallback**(`timed[round(ratio*(n-1))].start`)으로 대체한다.

**활성 라인**: `isManualSRT` 이면 `timedEntries[i].start <= currentTime` 인 최대 i, 아니면 `timeMap[i] >= 0 && <= currentTime` 인 최대 i. 활성 변경 시 `isExpanded` 면 컨테이너 내부만 `scrollTo`, 아니면 `el.scrollIntoView({block:"center"})`.

**동기화 상태 라벨**(플레이어 하단, 셋 중 하나만):
- 자막 로딩 중: `<Loader2 animate-spin> 자막 데이터 가져오는 중...`
- 타이밍 없음: `⚠ 이 영상에는 자막 타이밍 데이터가 없어 동기화가 불가합니다.`
- 그 외: manual → `⏱ {min(lyrics, entries)}/{entries} 구간 매핑`, fuzzy → `{icon} 추정 매핑 {m}/{n}줄`(비율 ≥0.7 `✅`, ≥0.3 `⚠`, 그 외 `❌`).

**리스트**: `AudioLyricsTab` 과 동일한 스타일·언어 스위처·확대 토글. manual 경로는 `formatTime(entry.start)`(`w-10 font-mono`), fuzzy 경로는 `{i+1}`(`w-5`). 낭독 버튼은 `speakTts(line.chinese, "zh")`.

### 2.3 강제 제약

- semantic token 우선. 예외로 허용하는 하드코드는 브랜드 hex `#243158`, `#3d6cb5`, `#2e3d6b` 와 자막 오버레이 전용 `bg-black/70` · `bg-black/65` · `text-white` · `text-gray-200` 뿐이다. 그 외 `bg-[#…]` / `text-white` / `bg-black` 금지.
- 낭독은 `speakTts` 단일 경로. 컴포넌트에서 TTS Edge Function 을 직접 `invoke` 하지 않는다.
- `<h1>` 은 페이지당 하나(가사 탭은 다이얼로그 내부이므로 `<h1>` 을 만들지 않는다).
- 카피는 순수 한국어(lorem ipsum 금지). 언어 pill 라벨(`中文`, `拼音`)은 학습 대상 표기이므로 예외.
- 세 컴포넌트는 자체적으로 song 형태 분기를 하지 않는다(분기는 P3j 책임).
- 모든 Edge Function 호출은 실패해도 UI 를 잠그지 않는다(배너·toast·fallback 중 하나로 처리).

## ③ Examples

### 3.1 확정 카피 표 (Design with Real Content)

| 위치 | 카피 |
|---|---|
| 언어 모드 pill | `中文` / `한국어` / `둘 다` |
| 표시 순서 pill | `中→한` / `한→中` |
| 병음 토글 | `병음` |
| 폰트 조절 | `A−` / `{n}` / `A+` |
| 문체 트리거(기본) | `✦ 스타일 분석 ▾` |
| 문체 옵션 | `시적 번역` / `직역` / `구어체 번역` / `해제` |
| 문체 실패 | `번역 실패. 다시 시도해 주세요.` |
| 카드 저장 진입/취소 | `📤 카드로 저장` / `✕ 취소` |
| 선택 배너 | `가사를 클릭해서 선택하세요 · {n}줄 선택됨` |
| 카드 만들기 | `카드 만들기 →` |
| 편집 버튼 | `수정` / `취소` / `저장` |
| Audio/Video 언어 pill | `한국어` / `中文` / `拼音` |
| 확대 토글 | `영상 확대` / `영상 축소` |
| 낭독 버튼 | `🗣️ 낭독` |
| Audio 타이밍 없음 | `⚠ 가사 타이밍 데이터가 없어 동기화가 제한됩니다.` |
| Audio 정렬 불일치 | `가사({a}줄)와 타임스탬프({b}줄)가 일치하지 않습니다.` |
| Audio 정렬 복구 버튼 | `정렬 복구` |
| Audio 정렬 성공 toast | `정렬 복구 완료` / `페이지를 새로고침해 주세요.` |
| Audio 정렬 실패 toast | `복구 실패` |
| Video 자막 로딩 | `자막 데이터 가져오는 중...` |
| Video 타이밍 없음 | `⚠ 이 영상에는 자막 타이밍 데이터가 없어 동기화가 불가합니다.` |
| Video manual 매핑 | `⏱ {m}/{n} 구간 매핑` |
| Video fuzzy 매핑 | `{icon} 추정 매핑 {m}/{n}줄` (`✅`/`⚠`/`❌`) |
| Audio 플레이어 부제 기본값 | `AI 생성 노래` |

### 3.2 컴포넌트 트리 (Use Prompt Patterns for Layouts)

```text
<SongAnalysisDialog>  ── 「가사」 서브탭 진입 시 song 분기 ──▶
   ├─ is_ai_generated && audio_url   → <AudioLyricsTab>
   │      ├─ <SongPlayer embedded stageOverlay={<Danmaku>} onRefreshVideos />
   │      ├─ 타이밍 없음 안내 (조건부)
   │      ├─ 정렬 복구 배너 (lineMismatch && isAiGenerated && songId)
   │      ├─ <SongFeatureButtons>            ── P3n
   │      └─ LangSwitcher + 가사 리스트 (binary-search 활성 라인, seek)
   │
   ├─ else if video_id                → <VideoLyricsTab>
   │      ├─ <div id="yt-player-{videoId}"/>  ── useYouTubePlayer
   │      │   └─ Danmaku overlay bottom-[52px]
   │      ├─ 동기화 상태 라벨 (로딩 / 없음 / manual / fuzzy)
   │      ├─ <SongFeatureButtons>            ── P3n
   │      └─ LangSwitcher + 가사 리스트 (manual index 또는 fuzzy buildTimeMap)
   │
   └─ else                            → <LyricsTab>
          ├─ 컨트롤 바 (mode + order + 병음 + 폰트 + 문체 + 카드저장 + 편집)
          ├─ 선택 모드 배너 (조건부)
          └─ 가사 카드 리스트 (읽기 / 편집 / 선택 상태)
                └─ 카드저장 → <LyricCardDialog>
```

**상태 스키마**:
```ts
// LyricsTab
type DisplayMode  = "zh" | "ko" | "both";
type DisplayOrder = "zh-ko" | "ko-zh";
type StyleType    = "none" | "poetic" | "literal" | "casual";
const MIN_FONT = 11, MAX_FONT = 22, DEFAULT_FONT = 15;

// Audio / Video 공통
type LangKey = "korean" | "chinese" | "pinyin";
const [activeLangs, setActiveLangs] = useState<Set<LangKey>>(new Set(["korean","chinese","pinyin"]));
const [isExpanded, setIsExpanded]   = useState(false);
```

## ④ Context (배경)

### 4.1 프로젝트 맥락
가사 뷰는 곡 학습의 근간이다. 세 렌더러는 (a) 텍스트만 있는 곡, (b) Suno 로 생성해 라인-레벨 정렬을 확보한 AI 곡, (c) YouTube 원본 자막에 의존하는 실제 곡 — 세 콘텐츠 원천을 하나의 UX 언어(활성 라인 하이라이트 · 언어 스위처 · 낭독)로 감싼다. 학습자가 TOPIK(한→중) 과 HSK(중→한) 어느 방향으로 들어와도 첫 화면의 언어 순서와 낭독 언어가 학습 방향과 일치해야 한다는 것이 P3k 의 핵심 요구다.

### 4.2 Lovable Cloud 후경 (Build with Lovable Cloud in Mind)

호출하는 Edge Function:

| 함수 | 호출 지점 | 실패 처리 |
|---|---|---|
| `typecast-tts` | `speakTts`(lang=ko) | 브라우저 음성 fallback, `fallback:true` 면 무로그 |
| `xf-tts` | `speakTts`(lang=zh) | 동일 |
| `translate-lyrics-style` | LyricsTab 문체 선택 | `styleError=true` → 라인별 실패 문구 |
| `sg-pixabay-videos` | AudioLyricsTab 배경 새로고침 | 일반 쿼리 재시도 → `console.warn` |
| `sg-repair-aligned` | AudioLyricsTab 정렬 복구 | `destructive` toast |

- `useClientTranscript`: 서버 `timedEntries` 가 없을 때 브라우저에서 자막을 가져오고 `songId + analysisId` 기준으로 캐시.
- `useYouTubePlayer`: IFrame API 로 `currentTime` 폴링 + `seekTo` 노출.
- 4-상태 렌더링: 로딩(`<Loader2 animate-spin>` / `<Skeleton>`) · 빈(`⚠` 안내 문구) · 에러(toast + `text-destructive`) · 성공(활성 하이라이트).

### 4.3 데이터 계약

```ts
export type LyricLine = { chinese: string; pinyin: string; korean: string };
export type TimedEntry = { start: number; dur: number; text: string };
export type AlignedWordSuno  = { word: string; start: number; end: number; success?: boolean };
export type AlignedWordVideo = { word: string; startS: number; endS: number };

// songs 컬럼 (스키마 원본은 P3h / B1 참조)
//   audio_url      text
//   aligned_words  jsonb   -- Suno 결과
//   bg_video_list  jsonb   -- Pixabay 후보 배열 (string[] 또는 {videoUrl}[])
//   lyrics_source  text    -- "custom_timed" 이면 index 1:1 매핑
//   language       text    -- "chinese" | "korean"
//   hsk_level      text    -- "TOPIK …" 로 시작하면 한국어 우선 표시
```

본 컴포넌트들은 DDL/RLS 를 생성하지 않는다. 관련 스키마·GRANT·정책은 P3h(songs) 및 B1/B2(edge functions) 에서 정의된다.

## ⑤ Acceptance & Output (IEEE 830 §4.3.6)

### 5.1 Acceptance Criteria

- **기본 순서**: `songHskLevel` 이 `TOPIK` 으로 시작하거나 `language === "korean"` 인 fixture 에서 초기 `order === "ko-zh"` (테스트 4 케이스: TOPIK+zh곡 / TOPIK+ko곡 / HSK+zh곡 / HSK+ko곡 → 앞 3개 `ko-zh`, HSK+zh곡만 `zh-ko`).
- **낭독 언어 분기**: `mode` × `order` 6 조합 전부에서 `speakTts` 인자 언어가 규칙과 일치(`ko`→ko, `zh`→zh, `both+ko-zh`→ko, `both+zh-ko`→zh). 우선 언어 텍스트가 빈 문자열인 fixture 에서 반대 언어로 1회 fallback, 양쪽 공백이면 호출 횟수 = **0**.
- **TTS 라우팅**: `lang="korean"` 호출 시 `typecast-tts` 1회 · `xf-tts` 0회, `lang="chinese"` 호출 시 그 반대. 동일 `(lang, speed, text)` 재호출 시 함수 호출 = **0**(캐시 적중).
- **활성 라인 지연** ≤ **80 ms**(currentTime 갱신 → 하이라이트 클래스 적용).
- **이진 탐색**: `aligned.length = 1000` 에서 활성 라인 결정 ≤ **1 ms**, 비교 횟수 ≤ **10**.
- **문체 캐시**: 동일 style 재선택 시 `translate-lyrics-style` 호출 = **0**, 최초 선택 = 1.
- **편집 저장**: `onSave(editLyrics)` 정확히 1회 + 저장 후 읽기 모드 복귀. 편집 취소 시 원본 `lyrics` 불변(딥카피 검증).
- **fuzzy 매핑**: 매칭 비율 < 30 % 이면 `timeMap.every(t => t >= 0)` 이 참(인덱스 비율 fallback 100 % 적용).
- **manual 경로**: `alignedWords` 가 있거나 `lyricsSource === "custom_timed"` 일 때 `buildTimeMap` 호출 = **0**.
- **정렬 복구**: `lineMismatch && isAiGenerated && songId` 세 조건 동시 충족 시에만 배너 렌더, `sg-repair-aligned` 호출은 클릭당 1회, `repairing` 중 재클릭 = 0회.
- **배경 새로고침 뮤텍스**: `onRefreshVideos` 를 100 ms 간격 5회 연속 호출해도 진행 중 재-엔트리 = **0**.
- **자동 스크롤**: `isExpanded === true` 인 VideoLyricsTab 에서 페이지 `window.scrollY` 변화 = **0**.
- **언어 스위처 하한**: `activeLangs.size ≥ 1` 항상 유지. 마지막 pill 클릭 시 상태 변화 = 0.
- **카드 저장**: `selectedIdx.size === 0` 이면 `카드 만들기 →` disabled, 클릭 = no-op. 다이얼로그 닫힘 시 `selectMode === false`.
- **폰트 경계**: `fontSize ∈ [11, 22]` 클램프, 경계 초과 클릭 시 상태 변화 = 0.
- **하드코드 색상 검사**: `rg -n "bg-\[#(?!243158|3d6cb5|2e3d6b)" src/components/songs/LyricsTab.tsx src/components/songs/AudioLyricsTab.tsx src/components/songs/VideoLyricsTab.tsx` = **0**.
- **TTS 직접 호출 금지 검사**: `rg -n "SpeechSynthesisUtterance|functions.invoke\(\"(xf|typecast)-tts" src/components/songs/{LyricsTab,AudioLyricsTab,VideoLyricsTab}.tsx` = **0**.
- **한국어 카피 커버리지**: 3.1 표의 모든 카피가 해당 컴포넌트에서 최소 1회 등장.

### 5.2 Output Format

LLM(또는 Lovable) 은 아래 파일을 이 순서대로 반환한다. 설명·사과·주석·마크다운 헤더 등 부수 텍스트는 금지한다.

1. `src/components/songs/LyricsTab.tsx`
2. `src/components/songs/AudioLyricsTab.tsx`
3. `src/components/songs/VideoLyricsTab.tsx`
4. 한국어 3줄 요약(무엇을 만들었는지, 어떤 edge function 을 호출하는지, 세 컴포넌트 분기 규칙).
