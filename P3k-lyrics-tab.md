# P3k · 가사 탭(Lyrics / Audio / Video) 재현 프롬프트

> **본 프롬프트는 P3 시리즈(P3a–P3q)의 11/17.**
> **적용 대상**: `src/components/songs/LyricsTab.tsx`, `AudioLyricsTab.tsx`, `VideoLyricsTab.tsx` — 세 가지 가사 뷰(정적 텍스트 · Suno 오디오 하이라이트 · YouTube 하이라이트). 각 뷰는 **독립적인 컴포넌트**이며 상위(P3j)의 `<SongAnalysisDialog>` 의 「가사」 서브탭 라우터에서 song 데이터 형태에 따라 하나가 선택되어 렌더된다.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* Retrieved July 12, 2026.
2. **Lovable. (n.d.).** *Prompting best practices.* Retrieved July 12, 2026 — 5개 실천 원칙(Prompt by Component · Speak Atomic · Design with Real Content · Use Prompt Patterns for Layouts · Build with Lovable Cloud in Mind) 채택 근거.
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity (신원)

당신은 시니어 프론트엔드 엔지니어 겸 한중 이중언어 교육 UX 라이터입니다. React 18 + Vite 5 + Tailwind v3 + shadcn/ui + Supabase(JS v2) 를 사용해, YouTube IFrame API · Web Audio · `speechSynthesis` · Suno alignedWords · 클라이언트-측 자막 추출을 다뤄 세 가지 가사 렌더러를 만듭니다. 대상은 대한민국 대학의 K-Chinese/K-Korean 교사·학습자입니다.

## ② Instructions

### 2.1 산출물 (Prompt by Component)

- **`src/components/songs/LyricsTab.tsx`** — 오디오/비디오 없는 **정적 가사 뷰**. 폰트 · 문체 번역 · 카드 저장 · 편집을 담당한다.
- **`src/components/songs/AudioLyricsTab.tsx`** — Suno 로 생성된 **AI 곡 전용**(자체 `audio_url` + `alignedWords`). `SongPlayer` 임베드 + 라인 하이라이트 + 배경 영상 새로고침 + 정렬 복구를 담당한다.
- **`src/components/songs/VideoLyricsTab.tsx`** — **YouTube 영상 곡 전용**(`video_id`). IFrame 플레이어 + 서버/클라이언트 자막 fallback + fuzzy time-map 매칭을 담당한다.
- 세 컴포넌트는 **독립**이며 공유 훅을 만들지 않는다. 상위 `<SongAnalysisDialog>`(P3j) 의 「가사」 탭이 `song` 형태로 분기한다:
  - `is_ai_generated && audio_url` → `<AudioLyricsTab>`
  - `video_id` → `<VideoLyricsTab>`
  - 그 외 → `<LyricsTab>`
- 관련 라이브러리:
  - `src/features/song-player/SongPlayer.tsx` (`embedded` 모드로 사용)
  - `src/features/song-player/lines.ts` — `parseSunoLines`, `alignLyricsToSuno`
  - `src/hooks/useYouTubePlayer.ts` — IFrame API wrap
  - `src/hooks/useClientTranscript.ts` — 서버 `timedEntries` 없을 때 브라우저에서 자막 추출
  - `src/components/songs/LyricCardDialog.tsx` — 카드 편집기(P3l 문서에서 상세)
  - `src/components/songs/SongFeatureButtons.tsx` — 2×2 하단 기능 버튼(P3n 참조)

### 2.2 원자적 UI 규칙 (Speak Atomic)

#### 2.2.1 `LyricsTab` (정적 뷰)

**Props**: `lyrics: LyricLine[]`, `onSave(lyrics)`, `readOnly?: boolean`, `language?: "chinese"|"korean"`, `songId?`, `songTitle?`, `songArtist?`, `songHskLevel?`.

**컨트롤 바**(상단, `flex flex-wrap items-center gap-2 pb-3 border-b`):

1. **언어 모드 그룹**(3-pill segmented, `mode: "zh" | "ko" | "both"`, 기본 `"both"`) — 라벨 `中文 / 한국어 / 둘 다`. 활성 pill 은 `bg-[#243158] text-white`.
2. **표시 순서 그룹**(2-pill, `order: "zh-ko" | "ko-zh"`, 기본은 `language==="korean"` 이면 `"ko-zh"` 아니면 `"zh-ko"`). `mode !== "both"` 이면 disabled + 40% opacity. 라벨 `中→한 / 한→中`.
3. **병음 토글** 단일 pill `병음` — `showPinyin: boolean`, 기본 `true`.
4. **폰트 크기 조절**: `A− {N} A+` (`fontSize` state, MIN=11, MAX=22, 기본 15). 중앙 숫자는 `bg-muted min-w-[28px]` 로 현재 값을 굵게 표시.
5. **문체 분석 Popover**(`songId` 있을 때만 렌더): 트리거는 `✦ {현재라벨} ▾` pill. Popover 옵션 = `시적 번역 / 직역 / 구어체` 3개 + (선택 상태일 때만) `해제`. `"원문"` 옵션은 존재하지 않는다. 선택 시 `translate-lyrics-style` edge function 호출, 성공 결과는 로컬 `styleCache: Record<StyleType, Record<lineIdx, string>>` 에 저장하고 재선택 시 재호출하지 않는다.
6. **카드 저장 버튼** `📤 카드로 저장` — `bg-[#3d6cb5] text-white`. 클릭 시 `selectMode=true` 진입, 버튼 라벨은 `✕ 취소` + `animate-pulse bg-red-500`.
7. **편집 버튼 그룹**(`!readOnly` 시 우측 정렬 `ml-auto`):
   - 정지 상태: `수정` 아이콘 버튼(`<Pencil>`).
   - 편집 상태: `취소` + `저장` 두 버튼. `저장` 클릭 시 `onSave(editLyrics)` 호출 후 편집 종료.

**선택 모드 배너**(`selectMode === true` 일 때만 표시):
- 컨테이너 `bg-[#3d6cb5]/10 border border-[#3d6cb5]/40 rounded-lg px-4 py-2.5`.
- 카피 `가사를 클릭해서 선택하세요 · <strong>{n}</strong>줄 선택됨` + `카드 만들기 →`(선택 0줄 disabled) + `취소`.
- `카드 만들기` 클릭 → 선택된 index 를 오름차순 정렬 후 `<LyricCardDialog>` 를 `selectedLines={ordered.map(i => {kr, zh, py})}` 로 연다.
- 다이얼로그가 닫히면 selectMode 도 함께 해제한다.

**가사 라인 카드**(순차 렌더, `p-4 rounded-lg border bg-card`):
- **읽기 모드**: `renderLines(line, mode, order, showPinyin, fontSize)` 로 3중 텍스트 배치.
  - `zh`: `font-medium text-foreground` 크기 = `fontSize`
  - `pinyin`: `text-[#3d6cb5]` 크기 = `fontSize - 2`
  - `ko`: `text-secondary-foreground` 크기 = `fontSize - 1`
  - 순서 규칙: `mode==="zh"` → zh+py / `mode==="ko"` → ko / `mode==="both" && order==="zh-ko"` → zh+py+ko / `mode==="both" && order==="ko-zh"` → ko+zh+py.
  - 우측 `<Volume2>` 아이콘 버튼 → `speakText(zhSong ? line.chinese : line.korean, language)` 호출, `speechSynthesis` `lang = "ko-KR"` 또는 `"zh-CN"`, `rate = 0.85`.
  - 문체 결과가 있으면 `mt-2 pl-3 border-l-2 border-[#3d6cb5]/40 bg-[#243158]/5` 블록에 `{emoji} {styleResult}` 로 표시. 로딩 중은 `<Skeleton h-5 w-3/4>`, 실패 시 `text-destructive` 로 `번역 실패. 다시 시도해 주세요.`.
- **편집 모드**: 3개 `<Input h-8>` (중국어 / 병음 / 한국어) 로 대체. `editLyrics` 는 진입 시 `JSON.parse(JSON.stringify(lyrics))` 로 딥카피.
- **선택 모드**: 카드 전체가 클릭 가능(`cursor-pointer`), 선택 시 `ring-2 ring-[#3d6cb5] bg-[#3d6cb5]/5`.

**문체 이모지 매핑**: `poetic → 🌸`, `literal → 📖`, `casual → 💬` (라인 결과 왼쪽 접두어).

#### 2.2.2 `AudioLyricsTab` (Suno AI 곡)

**Props**: `audioUrl`, `lyrics`, `alignedWords?: {word,start,end}[]`, `bgVideoList?: string[]|{videoUrl}[]`, `songId?`, `analysisId?`, `language?`, `songTitle?`, `songArtist?`, `songHskLevel?`, `lyricsRaw?`, `isAiGenerated?`, `onNavigateToSong?`.

**레이아웃**: `flex gap-4`, 기본 `flex-col lg:flex-row`, `isExpanded === true` 이면 `flex-col`.
- **좌측(50 %)** — `<SongPlayer embedded audioUrl bgVideoList onRefreshVideos title subtitle onTimeUpdate onReady stageOverlay />`.
- **우측(1fr)** — 언어 스위처 + 스크롤 가사 목록 + 하단 정보.

**언어 스위처**(`<LangSwitcher>`): `Set<"korean"|"chinese"|"pinyin">` 로 **3개 독립 토글**(단, 마지막 하나는 해제 불가). Pill 형태 `bg-primary text-primary-foreground` 활성 / `bg-muted text-muted-foreground` 비활성. 우측 끝(`ml-auto`) 에 `<Maximize2/Minimize2> 영상 확대 / 영상 축소` 토글.

**시간 정렬**:
- `sunoLines = parseSunoLines(alignedWords || [])` — Suno 는 라인 단위 항목을 주며 `word` 가 `\n` 로 끝나면 한 줄.
- `aligned = alignLyricsToSuno(lyrics, sunoLines)` — lyrics.length 와 sunoLines.length 이 다를 수 있으므로 인덱스 정렬.
- `hasTimed = sunoLines.length > 0`, `lineMismatch = hasTimed && sunoLines.length !== lyrics.length`.
- 현재 활성 라인은 **이진 탐색**으로 `aligned[i].startS - 0.05 <= currentTime` 만족하는 가장 큰 i.

**정렬 복구 밴드**(`lineMismatch && isAiGenerated && songId` 일 때만):
- 카피 `가사({lyrics.length}줄)와 타임스탬프({sunoLines.length}줄)가 일치하지 않습니다.` + `<Wrench> 정렬 복구` 버튼.
- 버튼 클릭 → `supabase.functions.invoke("sg-repair-aligned", { body: { song_id } })`. 성공 시 toast `정렬 복구 완료` / 실패 시 `복구 실패` `destructive`.

**배경 영상 새로고침** (`SongPlayer` 의 `onRefreshVideos` 콜백):
- `q = [title, artist].filter(Boolean).join(" ") || "nature sky sunlight"`.
- 1차 `sg-pixabay-videos` 호출 → 결과 없으면 fallback `q="nature sky sunlight people city"`.
- 결과가 있으면 상태 갱신 + `songs.bg_video_list` UPDATE(fire-and-forget).
- 재-엔트리 방지 `refreshingBgRef` ref 로 뮤텍스.

**stageOverlay** (댄마쿠 자막, 플레이어 하단 `bottom-[12%]` 에 배치):
- 활성 라인의 `korean/chinese/pinyin` 을 각각 `bg-black/70 text-white` 블록으로 겹쳐 표시(`activeLangs` 에 있는 언어만).
- `pointer-events-none` 로 클릭 방해 안 함.

**라인 리스트**:
- 컨테이너 `overflow-y-auto` + `max-h-[60vh]`(축소) / `max-h-[40vh]`(확장).
- 각 라인: `formatTime(startS)` (mm:ss) 를 좌측에 monospace 로. 활성 라인은 `bg-primary/10 border-primary/30 shadow-sm scale-[1.01]`.
- 라인 클릭 → `apiRef.current?.seek(t)`.
- 라인 내부 primary 언어 결정: `language==="korean" ? ["korean","chinese","pinyin"] : ["chinese","korean","pinyin"]` 순서로 `activeLangs` 에 있는 첫 언어. Primary 는 `text-base font-medium`, 나머지는 `text-xs` (pinyin 은 `text-primary/70`).
- 라인 우측: `🗣️ 낭독` 버튼 — 곡 언어에 맞춰 TTS. `e.stopPropagation()` 로 seek 방지.

**자동 스크롤**: `currentActiveIndex` 변화 시 라인 요소를 컨테이너 내 중앙으로 부드럽게 스크롤(`container.scrollTo`, `behavior:"smooth"`). 페이지 뷰포트는 움직이지 않는다.

**하단 배지**: `hasTimed === false` 이면 `⚠ 가사 타이밍 데이터가 없어 동기화가 제한됩니다.`.

**하단 기능 그리드**: `songId` 있으면 `<SongFeatureButtons>` 렌더(P3n 정의).

#### 2.2.3 `VideoLyricsTab` (YouTube 영상)

**Props**: `videoId`, `lyrics`, `timedEntries?`, `alignedWords?: {word,startS,endS}[]`, `songId?`, `analysisId?`, `language?`, `lyricsSource?`, `songTitle?`, `songArtist?`, `songHskLevel?`, `lyricsRaw?`, `onNavigateToSong?`.

**레이아웃**: AudioLyricsTab 와 동일한 좌/우 2-Column + `isExpanded` 토글.

**좌측**: `<div id="yt-player-{videoId}">` — `useYouTubePlayer(videoId, containerId)` 훅으로 `{ currentTime, seekTo }` 를 얻는다. `aspect-video bg-black rounded-lg overflow-hidden`. 하단 `bottom-[52px]` 위치에 댄마쿠 3-라인 오버레이(pointer-events-none).

**자막 소스 결정**:
- **Manual SRT 경로**(`isManualSRT`): `alignedWords` 가 있거나 `lyricsSource === "custom_timed"`. `timedEntries` 는 `alignedWords.map(w => ({ start: w.startS, dur: max(0.1, w.endS-w.startS), text: w.word.replace(/\n+$/,"") }))`. 라인-타임 매핑은 **index-1:1**.
- **Fuzzy 경로**: `useClientTranscript(videoId, songId, analysisId, serverTimedEntries, language)` 로 `timedEntries` 확보 → `buildTimeMap(lyrics, timedEntries)` 로 매핑.
  - `normalize()`: 공백·`,，。！？、；：""''（）《》-–—…·.!?;:'"()[]` 제거 + toLowerCase.
  - **그룹화**: `timedEntries` 를 1.5 s 이상 간격이 있을 때 자름.
  - **매칭**: 각 lyric 라인 대해 모든 그룹 검사, `longer.includes(shorter)` 면 `score = shorter.length / target.length`, 아니면 문자 단위 매칭률 × 0.8. 사용된 그룹은 score × 0.6 감점. threshold **0.3** 이상만 채택.
  - **비상 fallback**: 매칭된 라인 수가 lyrics.length × 30 % 미만이면 index-비율 균등 분포로 대체.

**동기화 상태 라벨**(가사 리스트 상단):
- `!hasTimed` → `⚠ 이 영상에는 자막 타이밍 데이터가 없어 동기화가 불가합니다.`
- `isManualSRT` → `⏱ {min(lyrics.length, entries.length)}/{entries.length} 구간 매핑`
- 그 외: syncRatio = matched/lyrics.length, `✅`(≥0.7) / `⚠`(≥0.3) / `❌`(<0.3) + `추정 매핑 {n}/{lyrics.length}줄`.
- fetching 중이면 `<Loader2 animate-spin> 자막 데이터 가져오는 중...`.

**활성 라인 계산**:
- Manual: 선형 스캔 `timedEntries[i].start <= currentTime` 중 마지막 i.
- Fuzzy: 선형 스캔 `timeMap[i] >= 0 && timeMap[i] <= currentTime` 중 마지막 i.

**자동 스크롤**:
- `isExpanded` → 컨테이너 내부만 스크롤(`container.scrollTo`).
- 축소 상태 → `el.scrollIntoView({block:"center"})`.

**라인 카드**: AudioLyricsTab 와 동일한 primary/secondary 렌더 규칙. 좌측 시간 라벨: manual 은 `formatTime(entry.start)`, fuzzy 는 `i+1` 인덱스만. 클릭 시 `seekTo(entry.start)` 또는 `seekTo(timeMap[i])`. `🗣️ 낭독` 은 항상 **중국어**(`speakChinese`, `zh-CN`) — 이 컴포넌트는 K-Chinese 곡 전용이므로.

**하단 기능 그리드**: `songId` 있으면 `<SongFeatureButtons>` 렌더.

### 2.3 강제 제약 (Speak Atomic + Design with Real Content)

- 세 컴포넌트는 **공유 훅을 만들지 않는다**. 상태는 각자 관리한다.
- 컨트롤 활성 색은 프로젝트 dark-navy 팔레트에서 `#243158`(LyricsTab pill), `#3d6cb5`(강조), `bg-primary`(Audio/Video pill) 로 통일. 하드코드된 6자리 hex 는 위 3개만 예외적으로 허용하며 나머지는 semantic token(`bg-primary`, `text-muted-foreground`, `border-border`, `bg-card`, `bg-muted`).
- `rg -n "bg-\[#|text-white|bg-black" src/components/songs/{LyricsTab,AudioLyricsTab,VideoLyricsTab}.tsx` 결과는 위에서 허용한 오버레이 자막 배경(`bg-black/70`, `bg-black/65`)과 카드 저장 pill 색 3개(`#243158`, `#3d6cb5`, `#243158`) 이외에 새 항목이 **추가되어서는 안 된다**.
- `readOnly === true` 이면 편집 버튼과 저장 액션은 숨긴다. TTS · 문체 분석 · 카드 저장 · seek 는 유지.
- 모든 edge function 호출은 `supabase.functions.invoke(...)` 로 하고, 실패 시 toast 는 반드시 한국어 카피를 사용한다.
- lorem ipsum · placeholder 텍스트 사용 금지. 모든 사용자-facing 문자열은 아래 3.1 표의 확정 카피만 사용한다.
- `<h1>` 은 만들지 않는다(모달 내부이므로).

## ③ Examples

### 3.1 확정 카피 표 (Design with Real Content)

| 위치 | 카피 |
|---|---|
| LyricsTab 언어 모드 | `中文 / 한국어 / 둘 다` |
| LyricsTab 순서 | `中→한 / 한→中` |
| LyricsTab 병음 토글 | `병음` |
| LyricsTab 폰트 | `A− {N} A+` |
| LyricsTab 문체 트리거 | `✦ 스타일 분석 ▾` (선택 시 라벨 교체) |
| LyricsTab 문체 옵션 | `시적 번역 / 직역 / 구어체 / 해제` |
| LyricsTab 문체 이모지 | `🌸 / 📖 / 💬` |
| LyricsTab 문체 실패 | `번역 실패. 다시 시도해 주세요.` |
| LyricsTab 편집 진입 | `수정` |
| LyricsTab 편집 저장/취소 | `저장 / 취소` |
| LyricsTab 카드 저장 진입 | `📤 카드로 저장` |
| LyricsTab 선택 취소 | `✕ 취소` |
| LyricsTab 선택 배너 | `가사를 클릭해서 선택하세요 · {n}줄 선택됨` |
| LyricsTab 카드 생성 CTA | `카드 만들기 →` |
| Audio/Video 언어 스위처 | `한국어 / 中文 / 拼音` |
| Audio/Video 확대 토글 | `영상 확대 / 영상 축소` |
| Audio/Video 낭독 | `🗣️ 낭독` |
| Audio 정보 부제(artist 없을 때) | `AI 생성 노래` |
| Audio 타이밍 없음 | `⚠ 가사 타이밍 데이터가 없어 동기화가 제한됩니다.` |
| Audio 정렬 불일치 | `가사({a}줄)와 타임스탬프({b}줄)가 일치하지 않습니다.` |
| Audio 정렬 복구 버튼 | `정렬 복구` |
| Audio 정렬 성공 toast | `정렬 복구 완료 / 페이지를 새로고침해 주세요.` |
| Audio 정렬 실패 toast | `복구 실패` |
| Video 자막 로딩 | `자막 데이터 가져오는 중...` |
| Video 타이밍 없음 | `⚠ 이 영상에는 자막 타이밍 데이터가 없어 동기화가 불가합니다.` |
| Video manual 매핑 라벨 | `⏱ {m}/{n} 구간 매핑` |
| Video fuzzy 매핑 라벨 | `{icon} 추정 매핑 {m}/{n}줄` (icon = `✅`/`⚠`/`❌`) |

### 3.2 컴포넌트 트리 (Use Prompt Patterns for Layouts)

```text
<SongAnalysisDialog>  ── 「가사」 서브탭 진입 시 song 분기 ──▶
   ├─ is_ai_generated && audio_url   → <AudioLyricsTab>
   │      ├─ <SongPlayer embedded stageOverlay={<Danmaku>} onRefreshVideos />
   │      ├─ 정렬 복구 배너 (조건부)
   │      ├─ <SongFeatureButtons>            ── P3n
   │      └─ 가사 리스트 (binary-search 활성 라인)
   │
   ├─ else if video_id                → <VideoLyricsTab>
   │      ├─ <div id="yt-player-{videoId}"/>  ── useYouTubePlayer
   │      │   └─ Danmaku overlay bottom-[52px]
   │      ├─ 동기화 상태 라벨
   │      ├─ <SongFeatureButtons>            ── P3n
   │      └─ 가사 리스트 (manual index 또는 fuzzy buildTimeMap)
   │
   └─ else                            → <LyricsTab>
          ├─ 컨트롤 바 (mode + order + 병음 + 폰트 + 문체 + 카드저장 + 편집)
          ├─ 선택 모드 배너 (조건부)
          └─ 가사 카드 리스트 (읽기 / 편집 / 선택 상태)
                └─ 카드저장 → <LyricCardDialog>   ── P3l
```

**상태 스키마**:
```ts
// LyricsTab 내부
type DisplayMode = "zh" | "ko" | "both";
type DisplayOrder = "zh-ko" | "ko-zh";
type StyleType = "none" | "poetic" | "literal" | "casual";

// Audio/Video 내부
type LangKey = "korean" | "chinese" | "pinyin";
const [activeLangs, setActiveLangs] = useState<Set<LangKey>>(new Set(["korean","chinese","pinyin"]));
const [isExpanded, setIsExpanded] = useState(false);
```

**폰트 상수**: `MIN_FONT=11`, `MAX_FONT=22`, `DEFAULT_FONT=15`.

## ④ Context (배경)

### 4.1 프로젝트 맥락
가사 뷰는 곡 학습의 근간이다. 세 렌더러는 각각 (a) 텍스트만 있는 곡(신곡·낭독용), (b) Suno 로 생성해 라인-레벨 정렬을 이미 확보한 AI 곡, (c) YouTube 원본 자막에 의존하는 실제 곡 — 세 콘텐츠 원천을 통일된 UX 언어(활성 라인 하이라이트 · TTS · 언어 스위처)로 감싼다. 상위 P3j 다이얼로그가 song 형태에 따라 자동 분기하므로 각 컴포넌트는 자체적으로 조건 분기를 하지 않는다.

### 4.2 Lovable Cloud 후경 (Build with Lovable Cloud in Mind)

- Edge Function `translate-lyrics-style`: OpenAI 로 3-스타일 번역. LyricsTab 내부 `styleCache` 는 프로세스-내 캐시(모달을 다시 열면 재호출). 실패 시 toast/텍스트로 명시.
- Edge Function `sg-pixabay-videos`: Pixabay 검색. 결과가 비면 일반 fallback 쿼리 재시도.
- Edge Function `sg-repair-aligned`: Suno alignedWords 재정렬. AI 곡 전용.
- `useClientTranscript` 훅: 서버 `timedEntries` 미제공 시 브라우저에서 자막 fetch, `songId + analysisId` 기준 캐시.
- `useYouTubePlayer` 훅: IFrame API 로 currentTime 폴링 + seekTo 노출.
- 4-상태 렌더링: 로딩(`<Loader2 animate-spin>`) / 빈(`⚠` 배너) / 에러(toast + destructive 텍스트) / 성공(활성 하이라이트).

### 4.3 데이터 계약

```ts
export type LyricLine = { chinese: string; pinyin: string; korean: string };
export type TimedEntry = { start: number; dur: number; text: string };
export type AlignedWordSuno = { word: string; start: number; end: number; success?: boolean };
export type AlignedWordVideo = { word: string; startS: number; endS: number };

// songs 컬럼 (P3h 스키마 참조)
//   audio_url        text
//   aligned_words    jsonb           -- Suno 결과 저장
//   bg_video_list    jsonb           -- Pixabay 후보 배열
//   lyrics_source    text            -- "custom_timed" 이면 index-1:1 매핑
```

이 컴포넌트들은 DDL/RLS 를 직접 생성하지 않는다. 필요한 스키마는 P3h(songs) · P4a(edge functions) 에서 이미 정의된다.

## ⑤ Acceptance & Output (IEEE 830 §4.3.6)

### 5.1 Acceptance Criteria

- **활성 라인 지연** ≤ **80 ms**(재생 currentTime 갱신 → 하이라이트 클래스 적용).
- **AudioLyricsTab 이진탐색** 은 O(log n): `aligned.length = 1000` 기준 활성 라인 결정 ≤ **1 ms**.
- **문체 캐시**: 동일 style 재선택 시 `translate-lyrics-style` 호출 수 = **0**. 최초 선택 시 = 1.
- **LyricsTab 편집 저장**: `onSave(editLyrics)` 정확히 1 회 호출 + `setEditing(false)` 후 카드가 읽기 모드로 복귀.
- **VideoLyricsTab fuzzy 매핑**: 매칭 비율 < 30 % 이면 100 % 라인이 index-비율 fallback 시간을 가진다(즉 `timeMap.every(t => t>=0)`).
- **Manual SRT 경로**: `alignedWords` 가 있거나 `lyricsSource === "custom_timed"` 일 때 `buildTimeMap` 을 호출하지 않는다(호출 = 0회).
- **정렬 복구**: `lineMismatch === true && isAiGenerated === true && songId` 세 조건 모두일 때만 배너 렌더. `sg-repair-aligned` 호출은 클릭당 1 회, `refreshingBgRef` 를 이용한 뮤텍스로 재-엔트리 = 0.
- **자동 스크롤**: `isExpanded === true` 인 VideoLyricsTab 에서는 페이지 `window.scrollY` 변화 = 0(내부 컨테이너만 스크롤).
- **언어 스위처 하한**: `activeLangs.size` 는 항상 ≥ 1. 마지막 남은 언어 pill 클릭 시 무시.
- **카드 저장**: `selectedIdx.size === 0` 이면 `카드 만들기` 버튼 disabled 이며 클릭 = no-op.
- **폰트 크기 경계**: `fontSize ∈ [11, 22]` 로 클램프. 경계 넘어서는 클릭은 상태 변화 = 0.
- **하드코드 색상 검사**: `rg -n "bg-\[#(?!243158|3d6cb5)|text-white(?![^\s])" src/components/songs/{LyricsTab,AudioLyricsTab,VideoLyricsTab}.tsx` 결과 = **0**(위 2 hex + 자막 오버레이용 `bg-black/70`, `bg-black/65` 제외).
- **한국어 카피 커버리지**: 3.1 표의 모든 카피가 각 컴포넌트에서 최소 1회 등장.

### 5.2 Output Format

LLM(또는 Lovable) 은 아래 파일을 이 순서대로 반환한다. 설명·사과·주석·마크다운 헤더 등 부수 텍스트는 금지한다.

1. `src/components/songs/LyricsTab.tsx`
2. `src/components/songs/AudioLyricsTab.tsx`
3. `src/components/songs/VideoLyricsTab.tsx`
4. 한국어 3줄 요약(무엇을 만들었는지, 어떤 edge function 을 호출하는지, 세 컴포넌트 분기 규칙).
