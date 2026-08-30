# P3j · 노래 분석 다이얼로그 셸 · 상단 도구모음 · 4-기능 진입 재현 프롬프트

> **본 프롬프트는 P3 시리즈(P3a–P3q)의 10/17.**
> **적용 대상**: `src/components/songs/SongAnalysisDialog.tsx` (다이얼로그 셸 · 헤더 도구모음 · 6-Tabs 프레임 · 오디오/비디오/정적 3-모드 선택), `src/components/songs/SongFeatureButtons.tsx` (2×2 4-기능 그리드 + 인라인 확장 패널: 아티스트 소개 / 학습 가이드 / 관련 곡 추천 / 댓글).
> 각 탭 본문(가사/단어/문법/읽기/탐구)은 P3k~P3q 에서, 공유·임베드는 P3i 에서 정의한다. 본 문서는 그 껍데기·헤더·4-기능 오케스트레이터만 다룬다.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* Retrieved July 12, 2026.
2. **Lovable. (n.d.).** *Prompting best practices.* Retrieved July 12, 2026 — 5개 실천 원칙 채택 근거.
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity (신원)

당신은 곡 상세 학습 화면을 하나의 `Dialog`(`max-w-5xl h-[90vh]`) 안에 조립하는 시니어 프론트엔드 엔지니어입니다. 헤더 도구모음(제목·배지·가사 편집·재분석·공유), 4-기능 그리드(아티스트 소개 / 학습 가이드 / 관련 곡 추천 / 댓글), 6-Tabs(영상 / 가사 / 단어장 / 문법·표현 / 읽기 / 탐구)를 shadcn `Dialog`, `Tabs`, `Popover` 로 조합하되, 곡 데이터 형태에 따라 첫 탭(영상)의 하위 뷰를 `AudioLyricsTab`(AI 곡) · `VideoLyricsTab`(YouTube 곡) 로 자동 분기합니다.

## ② Instructions

### 2.1 산출물 (Lovable 실천 원칙 "Prompt by Component, Not Page")

- `src/components/songs/SongAnalysisDialog.tsx` — 셸 · 헤더 도구모음 · 6-Tabs 프레임 · 인라인 가사 편집기 · 붙여넣기로 분석 UI · 재분석 progress 시뮬레이션.
- `src/components/songs/SongFeatureButtons.tsx` — 2×2 4-기능 그리드 + 인라인 확장 패널 4종(`ArtistPanel`, `GuidePanel`, `RelatedPanel`, `CommentsPanel`) + `ComposerDialog`.
- `src/components/songs/EmbedDialog.tsx` — **P3i 를 그대로 재사용**(본 문서에서는 재정의 금지, `import` 만 함).
- Edge Functions: `generate-song-feature`(artist/guide 통합), `generate-song-comments`(초기 댓글 시드 + 사용자 append) — 존재를 전제로 호출만 정의, 구현은 P4d.

### 2.2 원자적 UI 규칙 (Lovable 실천 원칙 "Speak Atomic")

**Dialog 컨테이너** = `<DialogContent className="max-w-5xl h-[90vh] flex flex-col p-0">`. 닫기 X는 shadcn `Dialog` 기본 제공(별도 아이콘 추가 금지).

**Header 도구모음** (`DialogHeader px-6 pt-6 pb-2 shrink-0`) — **단 한 줄, `flex flex-wrap gap-2 items-center` 로 좌→우 순서 고정**:

1. `<Music className="h-5 w-5 text-primary" />` 아이콘.
2. `<span class="truncate">` 제목 — 최대 1줄, 초과 시 말줄임.
3. `— {artist}` — `text-sm text-muted-foreground font-normal`, 값 있을 때만.
4. `({year})` — 존재 시만, `text-xs text-muted-foreground`.
5. **level 배지** — `hsk_level` 존재 시 `hskColor[level]` 매핑, `text-xs px-2 py-0.5 rounded-full border`.
6. **가사 소스 배지** — `lyrics_source` 값에 따라 정확히 아래 6종 중 1개(없을 수도 있음):

   | `lyrics_source` | 배지 |
   |---|---|
   | `subtitle_lrclib` | `🎵 LRCLIB 가사` (emerald) |
   | `subtitle_*`(그 외) | `▶️ YouTube 자막` (emerald) |
   | `ai_search` | `⚠ AI 추정 가사` (amber) |
   | `ai_generated` | `🤖 AI 생성 가사` (purple) |
   | `custom` | `✏️ 직접 입력` (blue) |
   | `custom_timed` | `⏱ 자막 입력` (blue) |

7. **재분석 진행 배지**(`reanalyzing === true` 인 동안만): `<Badge variant="outline" class="gap-1.5 border-primary/40 text-primary bg-primary/5"><Loader2 spin/>분석중 {Math.floor(analyzeProgress)}%</Badge>` — 나머지 편집·재분석·공유 버튼은 이 동안 숨긴다.
8. **가사 고치기 버튼** — `variant="outline" size="sm" h-6 gap-1 text-xs`, `<Pencil/> 가사 고치기`. 클릭 시 인라인 편집기(§2.2 하단) 오픈.
9. **재분석 버튼**(Popover) — 트리거 `<RefreshCw/> 재분석`. 콘텐츠(w-44 p-2):
   - `학습 언어 선택` label.
   - 옵션 2개: `🇨🇳 중국어 배우기`(→ `onReanalyze("chinese")`), `🇰🇷 한국어 배우기`(→ `onReanalyze("korean")`). 국기는 `flag-icon-css` CDN SVG.
10. **공유 버튼**(Popover) — 트리거 `🔗 공유`. 콘텐츠(w-44 p-1.5)에 정확히 2 항목:
    - `🔗 링크 복사` → `navigator.clipboard.writeText("${origin}/shared/${share_token || id}")` + toast `링크가 복사되었습니다.` + Popover 닫기.
    - `📺 퍼가기 (임베드)` → `EmbedDialog` 오픈(P3i).

   **금지**: 헤더에 별도의 편집(⋮)/삭제(🗑)/즐겨찾기(♥)/닫기(✕) 아이콘 버튼을 추가하지 말 것. 편집·삭제·즐겨찾기·조회는 카드(P3h) 액션바에서만 노출한다.

11. **`readOnly === true` 인 경우** — 위 8·9·10 세 버튼을 DOM 에서 렌더하지 않는다(존재 자체가 없어야 함).

**Body** (`flex-1 overflow-y-auto px-6 pb-6 min-h-0`):

- **가사 편집기**(`showLyricsEditor`)가 열려 있으면 최상단에 `border-2 border-primary/30 bg-primary/5 rounded-lg p-4 space-y-3` 카드로 렌더: `<Textarea rows={10}>` + `[이 가사로 재분석]`(진행 중 `Loader2 재분석 중...`) + `[취소]`. 저장 규칙:
  - `full_analysis.manual_lyrics_source_raw` (SRT 원문)가 존재하면 그것을 초기값으로, 없으면 각 라인의 학습언어(중국어 곡→`chinese`, 한국어 곡→`korean`) 텍스트를 `\n` 조인한 값으로 초기화.
  - `[이 가사로 재분석]` 클릭 시 `onReanalyzeWithLyrics(text, learn_language)` 호출 후 편집기 닫힘.

- **로딩**(`analysisLoading`): `<Loader2 h-8 w-8 animate-spin>` 중앙.

- **분석 없음**(`analysisNotFound || !analysis`, `readOnly=false`): 정중앙 stack —
  - `<Music h-12 w-12 opacity-40>` + `분석이 아직 없습니다.`
  - `[지금 분석하기]` (진행 중 `분석 중... (1-2분 소요)`).
  - 접힘 상태의 `[가사 직접 입력하여 분석]` → 클릭 시 max-w-lg `<Textarea rows={8}>` + `[이 가사로 분석]` / `[취소]`.

- **분석 존재**: 6-Tabs 렌더(§ 아래).

- **`readOnly=true` + 분석 없음**: 안내 문구만(`분석 데이터가 없습니다.`), 어떤 CTA 도 렌더하지 않는다.

**6-Tabs 프레임** — `<Tabs defaultValue="video" className="space-y-4">`:

| value | icon | label | 하위 컴포넌트 (프롬프트) |
|---|---|---|---|
| `video` | `▷` | `영상` | `AudioLyricsTab`(AI 곡: `audio_url && !video_id`) 또는 `VideoLyricsTab`(기타) — P3k |
| `lyrics` | `▤` | `가사` | `LyricsTab` — P3k |
| `words` | `✦` | `단어장` | `WordListTab` — P3l |
| `patterns` | `◈` | `문법·표현` | `PatternsTab` — P3m |
| `reading` | `♪` | `읽기` | `ReadingTab` — 별도 프롬프트 없음, `lyrics_with_pinyin` 재사용 |
| `explore` | `◎` | `탐구` | `ExploreTab` — P3n~q |

`TabsList` 는 `flex w-full h-auto p-0 bg-card border-b border-border rounded-none`, 각 `TabsTrigger` 는 `flex-1 rounded-none border-b-2 border-transparent py-3 px-4 text-[13px] text-muted-foreground data-[state=active]:border-b-primary data-[state=active]:text-primary data-[state=active]:font-medium data-[state=active]:bg-transparent data-[state=active]:shadow-none hover:text-foreground/70`. 아이콘은 `<span class="mr-1">` 로 라벨 앞에 렌더.

**SongFeatureButtons — 4-기능 진입**(다이얼로그의 카드/뷰 안에서 사용, 위치는 상위 뷰가 결정하되 본 컴포넌트는 자기 완결적 UI 를 제공):

- 컨테이너 `<div class="mt-3 space-y-2.5">`.
- **2×2 그리드**(`grid grid-cols-2 gap-2`), 버튼 4개 순서·라벨·이모지 고정:

  | type | emoji | label(비활성) | label(active) |
  |---|---|---|---|
  | `artist` | 🎤 | 아티스트 소개 | 아티스트 소개 |
  | `guide` | 📖 | 학습 가이드 | 학습 가이드 |
  | `related` | 🎵 | 관련 곡 추천 | 관련 곡 추천 |
  | `comments` | 💬 | 댓글 보기 | 댓글 닫기 |

  각 버튼 `py-3 px-3 rounded-[10px] border text-[13px] font-medium transition-all duration-200 relative`. 활성 = `bg-[hsl(var(--dash-navy))] border-[hsl(var(--dash-navy))] text-primary-foreground shadow-md`. 비활성 = `bg-card border-border text-foreground/70 hover:bg-muted/50 hover:border-muted-foreground/30 hover:-translate-y-0.5 hover:shadow-sm`. `comments` 버튼만 우측에 `ChevronDown` (활성 시 180° 회전). 로딩 중 버튼 우측에 `Loader2 animate-spin` 절대배치.
- **인라인 확장 패널**(그리드 바로 아래, Sheet 아님) — `activePanel` 이 있으면 `rounded-xl border border-border bg-card overflow-hidden animate-in slide-in-from-top-2 duration-200`:
  - `loading === activePanel` → `<Loader2/> {activePanel==='comments'?'댓글 불러오는 중...':'생성 중...'}`.
  - `activePanel==='comments'` → `<CommentsPanel/>`(핀 고정 정렬, 좋아요 토글, 답글 펼침, `renderTextWithTags` 로 `#태그` 인라인 하이라이트, 역할 3종 `선생님/학생/익명` 별 아바타·테두리 스타일).
  - 그 외 → `<PanelHeader type/>` + 내용(`ArtistPanel` / `GuidePanel` / `RelatedPanel`).
- **캐시 정책**: `cache[type]` 존재 시 재호출 없이 즉시 렌더. `songId` 변경 시 `cache`/`comments`/`activePanel`/`loading` 를 모두 리셋.
- **댓글 작성**: `[댓글 쓰기]` → `<ComposerDialog>`(익명 스위치 + 태그 chip 6종 + 본문 `Textarea`). 성공 시 `comments` 낙관적 삽입 후 `generate-song-comments`(action:`append`) 로 서버 반영, 실패는 `console.warn` 만.
- **관련 곡**: 로컬 `fetchRelatedSongs({songId, artist, hskLevel, language})` — 같은 아티스트 또는 같은 언어·인접 등급 상위 6곡을 점수화, 각 카드 클릭 시 `onNavigateToSong(id)`.

**Edge Function 호출 계약**:

```ts
// artist / guide / related(관련 곡은 로컬 계산이므로 제외)
supabase.functions.invoke("generate-song-feature", {
  body: { song_id, feature_type: "artist" | "guide", title, artist,
          hsk_level, language, lyrics_snippet: (lyricsSnippet ?? "").slice(0, 300),
          is_ai_generated }
})
// 댓글 시드
supabase.functions.invoke("generate-song-comments", {
  body: { song_id, title, artist, lyrics: (lyricsSnippet ?? "").slice(0, 800) }
})
// 댓글 append
supabase.functions.invoke("generate-song-comments", {
  body: { song_id, action: "append", new_comment }
})
```

### 2.3 강제 제약

- 헤더에 **모국어/학습언어/이중** 세 갈래 학습 언어 스위처를 두지 않는다. 학습 언어 선택은 오직 `재분석` Popover 안에서만 발생한다.
- 헤더에 **작품 특징 / 부르기(잠금)** 등 존재하지 않는 항목을 넣지 않는다. 진입 버튼은 정확히 4종(아티스트 · 학습 가이드 · 관련 곡 · 댓글).
- 확장은 **인라인**(그리드 바로 아래 카드)이며 shadcn `Sheet`(우측 슬라이드)로 대체하지 않는다.
- semantic token 사용. 활성 버튼의 남색 배경은 `hsl(var(--dash-navy))` 토큰을 참조하고, `bg-[#…]` / `text-white` / `bg-black` 하드코드는 금지한다.
- `song?.id` 변경 및 `open === false` 전이 시 인라인 편집기·붙여넣기·재분석 progress·기능 캐시 상태를 전부 리셋한다(이전 곡 상태 누수 금지).
- `readOnly=true` 인 경우 헤더의 `가사 고치기`/`재분석`/`공유` 3버튼과 `SongFeatureButtons` 의 댓글 작성/좋아요 저장이 모두 no-op 이거나 DOM 미렌더.
- 재분석 progress 는 실제 진행률이 아닌 시각적 안내: 800 ms 간격 tick, 30% 이하 +2, 60% 이하 +1, 85% 이하 +0.5, 그 이후 +0.2, 상한 95%. `reanalyzing=false` 로 전환 시 0 으로 초기화.
- 공유 링크는 `song.share_token || song.id` 를 우선하고, `EmbedDialog` 는 P3i 정의를 그대로 사용(재정의 금지).

## ③ Examples (Lovable 실천 원칙 "Design with Real Content" & "Use Prompt Patterns for Layouts")

### 3.1 확정 카피 표

| 위치 | 카피 |
|---|---|
| 헤더 편집 버튼 | `가사 고치기` |
| 헤더 재분석 트리거 | `재분석` |
| 재분석 Popover 라벨 | `학습 언어 선택` |
| 재분석 옵션 | `중국어 배우기` / `한국어 배우기` |
| 재분석 진행 배지 | `분석중 {N}%` |
| 공유 트리거 | `🔗 공유` |
| 공유 옵션 1 | `🔗 링크 복사` |
| 공유 옵션 2 | `📺 퍼가기 (임베드)` |
| 링크 복사 토스트 | `링크가 복사되었습니다.` |
| 편집기 안내 | `가사를 수정하면 단어장·문형·읽기가 모두 새로 분석됩니다.` |
| 편집기 저장 | `이 가사로 재분석` |
| 편집기 재분석중 | `재분석 중...` |
| 분석 없음 · 제목 | `분석이 아직 없습니다.` |
| 분석 없음 · CTA | `지금 분석하기` |
| 분석 없음 · 소요 안내 | `분석 중... (1-2분 소요)` |
| 붙여넣기 진입 | `가사 직접 입력하여 분석` |
| 붙여넣기 안내 | `가사를 붙여넣고 분석 버튼을 눌러주세요.` |
| 붙여넣기 실행 | `이 가사로 분석` |
| readOnly 빈 상태 | `분석 데이터가 없습니다.` |
| Tabs | `영상 / 가사 / 단어장 / 문법·표현 / 읽기 / 탐구` |
| 4-기능 버튼 | `아티스트 소개 / 학습 가이드 / 관련 곡 추천 / 댓글 보기` |
| 댓글 버튼(활성) | `댓글 닫기` |
| 4-기능 로딩(생성) | `생성 중...` |
| 4-기능 로딩(댓글) | `댓글 불러오는 중...` |
| 댓글 헤더 | `💬 댓글 · {N}개` |
| 댓글 비어있음 | `아직 댓글이 없습니다.` |
| 댓글 역할 | `선생님 / 학생 / 익명` |
| 댓글 핀 라벨 | `고정` |
| 댓글 답글 라벨 | `답글 {N}` |
| 댓글 작성 CTA | `댓글 쓰기` |
| Composer placeholder | `이 노래에 대해 이야기해보세요…` |
| Composer 익명 스위치 | `익명으로 게시` |
| 생성 실패 토스트 | `생성 실패` |

### 3.2 컴포넌트 트리

```text
<Dialog max-w-5xl h-[90vh] p-0 flex flex-col>
  ├─ <DialogHeader px-6 pt-6 pb-2 shrink-0>
  │     └─ <DialogTitle flex flex-wrap gap-2 items-center>
  │           Music · title · — artist · (year) · [level] · [lyrics_source] ·
  │           (reanalyzing? [분석중 N%] : [가사 고치기] [재분석▾] [🔗 공유▾])
  │
  ├─ <Body flex-1 overflow-y-auto px-6 pb-6 min-h-0>
  │     ├─ if showLyricsEditor  → <Textarea + [이 가사로 재분석] [취소]>
  │     ├─ if analysisLoading   → <Loader2 center>
  │     ├─ elif !analysis       → <EmptyState + [지금 분석하기] + [가사 직접 입력하여 분석]>
  │     └─ else <Tabs defaultValue="video">
  │           <TabsList flex w-full border-b>
  │             ▷ 영상 · ▤ 가사 · ✦ 단어장 · ◈ 문법·표현 · ♪ 읽기 · ◎ 탐구
  │           </TabsList>
  │           video   → AudioLyricsTab | VideoLyricsTab   (P3k)
  │           lyrics  → LyricsTab                          (P3k)
  │           words   → WordListTab                        (P3l)
  │           patterns→ PatternsTab                        (P3m)
  │           reading → ReadingTab
  │           explore → ExploreTab                         (P3n~q)
  │
  └─ <EmbedDialog open={embedOpen} token={share_token || id}/>  ← P3i 재사용

<SongFeatureButtons>   ← 상위 뷰가 원하는 위치에 삽입
  ├─ grid grid-cols-2 gap-2
  │     [🎤 아티스트 소개] [📖 학습 가이드]
  │     [🎵 관련 곡 추천]  [💬 댓글 보기 / 댓글 닫기 ▾]
  ├─ if activePanel: rounded-xl border animate-in slide-in-from-top-2
  │     ├─ loading? Loader2 + 라벨
  │     ├─ comments? <CommentsPanel/>
  │     └─ else <PanelHeader/> + <ArtistPanel|GuidePanel|RelatedPanel/>
  └─ <ComposerDialog/>
```

## ④ Context

### 4.1 프로젝트 맥락

본 셸은 아카이브(P3a) 카드(P3h) 클릭, 공유 페이지(P3i) 삽입, 워크스페이스 최근 활동 등 다수 진입점에서 열린다. 콘텐츠 실체(가사/단어/문법/읽기/탐구)는 하위 프롬프트가 담당하고, 본 문서는 **껍데기·헤더·6-Tabs 분기·4-기능 진입** 만 정의한다. 공유·임베드는 P3i, 편집·삭제·즐겨찾기 액션은 P3h 액션바에 이미 존재하므로 헤더에 중복 배치하지 않는다.

### 4.2 Lovable Cloud 후경 (Lovable 실천 원칙 "Build with Lovable Cloud in Mind")

- 4-상태 렌더: **로딩**(`Loader2` 중앙) / **빈**(분석 없음 CTA 스택) / **에러**(edge function 실패 시 `생성 실패` toast) / **성공**(6-Tabs).
- Edge Functions 사용: `generate-song-feature`, `generate-song-comments`. Deno 함수, `deno.json` 에 등록됨. 재분석 트리거(`onReanalyze`)와 가사 수정 재분석(`onReanalyzeWithLyrics`)은 상위 컨테이너가 주입.
- `EmbedDialog` 는 P3i 에서 정의된 컴포넌트를 그대로 import — 재구현 금지.

### 4.3 데이터 계약

```ts
interface SongAnalysisDialogProps {
  song: Song | null;
  analysis: SongAnalysis | null;
  analysisLoading: boolean;
  analysisNotFound: boolean;
  reanalyzing: boolean;
  open: boolean;
  onOpenChange: (open: boolean) => void;
  onReanalyze: (learnLanguage?: "chinese" | "korean") => void;
  onReanalyzeWithLyrics?: (lyrics: string, learnLanguage?: string) => void;
  onAnalysisUpdated: (analysis: SongAnalysis) => void;
  onNavigateToSong?: (songId: string) => void;
  readOnly?: boolean;
}

// video 탭 분기 규칙
const useAudioMode = !!song.audio_url && !song.video_id;

// SongFeatureButtons
interface SongFeatureButtonsProps {
  songId: string;
  title?: string | null;
  artist?: string | null;
  hskLevel?: string | null;
  language?: "chinese" | "korean";
  lyricsSnippet?: string;   // 최대 800자 (댓글은 800, feature는 300으로 절단)
  isAiGenerated?: boolean;
  onNavigateToSong?: (songId: string) => void;
}
```

`song_analyses.full_analysis.manual_lyrics_source_raw` (`text | null`) — 사용자가 SRT/타임코드 포함 가사를 붙여넣은 경우 원문 보존용. `full_analysis.learn_language`(`"chinese" | "korean"`) — 마지막 재분석 학습 언어. 두 필드 모두 본 문서에서 신규 생성 금지, 이미 존재함을 전제.

## ⑤ Acceptance & Output

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6)

- 다이얼로그 첫 페인트 p95 ≤ **200 ms**(캐시 있음), 콜드 ≤ **600 ms**.
- 헤더 도구모음의 DOM 순서가 §2.2 순서와 100% 일치 (`document.querySelectorAll` 결과의 순차 매칭).
- 헤더에 `모국어 / 학습언어 / 이중` 문자열이 렌더되지 않음(`rg -n "이중|모국어" src/components/songs/SongAnalysisDialog.tsx` = **0**).
- 헤더에 별도의 편집(⋮)·삭제(🗑)·즐겨찾기(♥) 아이콘 버튼 DOM 이 존재하지 않음(`querySelectorAll('[aria-label="삭제"], [aria-label="즐겨찾기"]')` = 0).
- `readOnly=true` 시 `가사 고치기` / `재분석` / `공유` 3버튼이 DOM 에 존재하지 않음(자동 검증 3건 = 0).
- 6-Tabs `TabsTrigger` 개수 = **6**, 라벨 순서 = `영상 / 가사 / 단어장 / 문법·표현 / 읽기 / 탐구`(정규식 매칭 100%).
- `SongFeatureButtons` 는 정확히 4개의 버튼을 렌더, 라벨 = `아티스트 소개 / 학습 가이드 / 관련 곡 추천 / 댓글 보기(또는 댓글 닫기)`. `Sheet` 컴포넌트 사용 0건.
- 캐시: 같은 곡·같은 기능을 두 번 클릭 시 `supabase.functions.invoke` 호출 횟수 = **1**.
- `song.id` 전이 시 `activePanel === null`, `cache === {}`, `comments === []` 스냅샷 검증.
- 재분석 진행 배지는 `reanalyzing === true` 인 동안 800 ms 마다 최소 1회 리렌더, 상한 95%.
- `rg -n "bg-\[#|text-white|bg-black" src/components/songs/SongAnalysisDialog.tsx src/components/songs/SongFeatureButtons.tsx` — `hsl(var(...))` 관련 참조를 제외하고 하드코드 매칭 = **0**.
- 카피 표(§3.1) 문자열 렌더 매칭률 = **100 %**.

### 5.2 Output Format

정확히 아래 순서로 파일을 반환한다. 각 파일은 완전한 소스 코드로, 설명·사과·주석 헤더 없이 코드 블록만 출력한다.

1. `src/components/songs/SongFeatureButtons.tsx`
2. `src/components/songs/SongAnalysisDialog.tsx`
3. 한국어 3줄 요약(다이얼로그 셸·헤더 도구모음·4-기능 진입의 구현 요지).
