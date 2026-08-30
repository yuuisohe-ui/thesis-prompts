# P3n · 탐구 탭 셸 + 곡 정보 4모듈 재현 프롬프트

> **본 프롬프트는 P3 시리즈의 14/17.**
> **적용 대상**:
> - `src/components/songs/ExploreTab.tsx` — 4-level 뷰 상태 머신 + 56px 사이드바 + 허브 + l2-info 카드 4개(카드 자체에서 생성).
> - `src/components/songs/ArtistStoryPage.tsx` · `ProductionBackgroundPage.tsx` · `ReleaseReactionPage.tsx` · `EraContextPage.tsx` — l3 상세(생성 결과 표시 + 재생성 + 저장).
> - `src/components/songs/artistStoryHelpers.ts` — 타입 · 검증 함수 · 캐시 · `generate*()` 래퍼.
> - `supabase/functions/generate-artist-story/index.ts` — 단일 함수, `action` 파라미터로 모든 곡 정보 작업 처리.
>
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* https://platform.openai.com/docs/guides/prompt-engineering (Retrieved July 12, 2026)
2. **Lovable. (n.d.).** *Prompting best practices.* https://docs.lovable.dev/prompting/prompting-one (Retrieved July 12, 2026)
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity

당신은 곡 분석 다이얼로그의 4번째 탭 "탐구"의 셸(사이드바 + 허브 + l2 서브그리드) 과, 그 첫 축인 "곡 정보" 4모듈(아티스트 스토리 / 제작 배경 / 발매 당시 반응 / 시대 배경) 을 구현하는 시니어 프론트엔드 엔지니어이자 학습 콘텐츠 IA(정보구조) 설계자입니다. 4-level 라우팅(hub → l2 → l3 → 일부 l4) 을 단일 컴포넌트 상태 머신으로 관리하고, 56 px 다크 네이비 사이드바로 상시 이동성을 보장합니다. **생성 트리거는 l2-info 카드 자체에 있고, l3 상세 페이지는 캐시된 결과의 표시·언어전환·재생성·저장만 담당합니다** — 이 원칙을 절대 반전시키지 마세요.

## ② Instructions

### 2.1 산출물(반드시 아래 8개 파일)

1. `src/components/songs/artistStoryHelpers.ts`
2. `supabase/functions/generate-artist-story/index.ts`(단일 함수, action 라우터)
3. `src/components/songs/ArtistStoryPage.tsx`
4. `src/components/songs/ProductionBackgroundPage.tsx`
5. `src/components/songs/ReleaseReactionPage.tsx`
6. `src/components/songs/EraContextPage.tsx`
7. `src/components/songs/ExploreTab.tsx`
8. 한국어 3줄 요약.

### 2.2 뷰 상태 머신

```ts
type ViewId =
  | "hub"
  | "l2-info" | "l2-lyric" | "l2-practice" | "l2-class"
  | "l3-artist" | "l3-background" | "l3-reaction" | "l3-era"
  | "l3-culture" | "l3-metaphor" | "l3-era-lang" | "l3-emotion"
  | "l3-quiz" | "l3-dictation" | "l3-pronunciation" | "l3-writing"
  | "l3-discussion" | "l3-writing-topic" | "l3-quiz-sheet"
  | "l4-word-quiz" | "l4-grammar-quiz" | "l4-lyrics-quiz";

type SidebarSection = "hub" | "info" | "lyric" | "practice" | "class";
```

`getActiveSection(view)` 는 반드시 `view.includes(...)` 패턴 매칭으로 구현(매핑 테이블 아님):
- `hub` → `hub`
- `info | artist | background | reaction | l3-era` → `info`
- `lyric | culture | metaphor | l3-era-lang | l3-emotion` → `lyric`
- `practice | quiz | dictation | pronunciation | writing | l4-*` → `practice`
- `class | discussion | l3-writing-topic | l3-quiz-sheet` → `class`

### 2.3 셸 — 사이드바 + 허브

**컨테이너**: 최상위 `flex h-full`. 왼쪽 사이드바 `w-14 shrink-0`, 오른쪽 `flex-1 overflow-y-auto`.

**사이드바 (`ExploreSidebar`)**:
- `w-14 bg-[#2e3d6b] py-[33px] px-0 my-0 mx-0 gap-[4px] flex flex-col items-center`.
- 상단 로고: `w-9 h-9 rounded-[10px] bg-amber-500` 안에 `♪` (text-base), `mb-3`.
- 로고 아래 구분선: `w-7 h-px bg-white/10 mb-1.5`.
- 5개 버튼: `w-10 h-10 rounded-[10px] text-[17px]`. 활성 = `bg-white/10 text-white`, 비활성 = `text-[#8fa3c8] hover:bg-white/[0.07] hover:text-white`.
- 각 버튼을 `<Tooltip side="right">` 로 감싼다. `TooltipProvider delayDuration={200}`.
- 순서/아이콘/라벨(고정):
  | id | icon | label | targetView |
  |---|---|---|---|
  | hub | 🏠 | `탐구 홈` | `hub` |
  | info | 📖 | `곡 정보` | `l2-info` |
  | lyric | 🔍 | `가사 심화` | `l2-lyric` |
  | practice | ✏️ | `연습` | `l2-practice` |
  | class | 🎓 | `수업 도구` | `l2-class` |

**허브 (view = "hub")**:
- 상단 `<PageHeader title="탐구" desc="이 곡을 더 깊게 파고들어보세요" />`.
- `grid grid-cols-2 gap-3.5`, 4장의 `HubCard`.
- 카드: `bg-card border rounded-[14px] p-6 cursor-pointer shadow-sm hover:shadow-md hover:-translate-y-0.5 border-b-[3px] border-b-transparent`. hover 시 하단 border 색을 accent 로. `text-[28px]` 아이콘, `text-[15px] font-bold` 제목, `text-xs` 설명, 하단 태그 pill `text-[11px] px-2.5 py-1 rounded-full`.
- 4개 카드(고정):

| icon | title | desc | tag | tagColor | accent | view |
|---|---|---|---|---|---|---|
| 📖 | `곡 정보` | `아티스트 생애, 제작 배경, 발매 반응, 시대 맥락을 탐구합니다` | `배경 탐구` | blue → `bg-primary/10 text-primary` | `border-b-primary` | `l2-info` |
| 🔍 | `가사 심화` | `문화 비교, 가사 속 비유와 상징, 시대 언어 특징, 감정 분석을 분석합니다` | `언어 분석` | purple → `bg-purple-100 text-purple-600` | `border-b-purple-500` | `l2-lyric` |
| ✏️ | `연습` | `퀴즈, 받아쓰기, 발음 연습, AI 작문으로 실력을 다집니다` | `학습 활동` | green → `bg-green-100 text-green-600` | `border-b-green-500` | `l2-practice` |
| 🎓 | `수업 도구` | `토론 질문, 핵심 단어 요약, 퀴즈 시트를 바로 생성합니다` | `선생님 전용` | gold → `bg-amber-100 text-amber-700` | `border-b-amber-500` | `l2-class` |

> `#2e3d6b`, `#8fa3c8`, `bg-amber-500` 은 **사이드바 브랜드 색 전용 예외**. 그 외는 semantic token.

### 2.4 l2-info 그리드 (핵심)

상단 `<BackButton label="탐구" />` (좌측 상단, `mb-5 inline-flex text-[13px] border bg-card px-3 py-1.5 rounded-lg hover:border-primary hover:text-primary`, 텍스트 `← 탐구`).
`<PageHeader icon="📖" title="곡 정보" desc="아티스트와 이 곡의 배경을 탐구해보세요" />`.
`grid grid-cols-2 gap-3.5` 안에 4개 카드. 4개 모두 동일한 `<AiGenCard>` 공통 컴포넌트를 사용한다(시대 배경도 예외 없이 동일 UX).

**`AiGenCard` — idle/generating/done/error 4상태 카드**(카드 자체가 생성 트리거):

```tsx
type GenState = "idle" | "generating" | "done" | "error";
interface AiCardState { state: GenState; progress: number; error?: string; }
```

- **컨테이너**: `bg-card border rounded-xl p-5 shadow-sm`.
- **본체**: `text-[22px] mb-2.5` 아이콘 + `text-sm font-bold mb-1.5` 제목 + `text-xs text-muted-foreground mb-3.5` 설명.
- **idle**: 파란 버튼 `inline-flex text-xs font-medium text-primary border border-primary bg-primary/5 px-3.5 py-1.5 rounded-lg hover:bg-primary hover:text-primary-foreground` → 라벨 `✦ AI로 생성하기`. 클릭 시 `onGenerate()`.
- **generating**: 상단에 `Loader2 h-3.5 w-3.5 animate-spin` + `생성 중... {Math.round(progress)}%`(색 primary), 하단 `<Progress value={progress} className="h-1.5" />`. 시뮬레이션은 §2.6 규칙.
- **error**: `AlertCircle` + `{error || "생성 실패"}` (destructive 색) + `↺ 다시 시도` 버튼 → 다시 `onGenerate()`.
- **done**: 초록 버튼 `text-green-700 border-green-300 bg-green-50 hover:bg-green-100` → `<CheckCircle2 />` + `생성 완료 · 페이지 열기`. 클릭 시 `onOpen()` → 해당 l3 뷰로 이동.

**4개 카드 파라미터**:

| icon | title | desc | onGenerate | onOpen |
|---|---|---|---|---|
| 👤 | `아티스트 스토리` | `가수의 생애와 활동 타임라인을 확인합니다` | `generateArtistStory()` | `nav("l3-artist")` |
| ✍️ | `제작 배경` | `이 곡은 왜, 누구를 위해 만들어졌는지 알아봅니다` | `generateProduction()` | `nav("l3-background")` |
| 📈 | `발매 당시 반응` | `차트 성적, 수상, 당시 사회적 반응을 살펴봅니다` | `generateReception()` | `nav("l3-reaction")` |
| 🕰️ | `시대 배경` | `이 곡이 발매된 시대의 사회·문화적 맥락을 이해합니다` | `handleGenerateEra("ko")` (기본 한국어) | `nav("l3-era")` |

### 2.5 시대 배경 — 기본 한국어 생성 + l3 내 언어 전환

시대 배경 카드는 다른 3개와 완전히 동일한 UX(진행바 · 완료 버튼 · 재시도)를 따르며, 카드 위 언어 피커는 두지 않는다.
- 카드에서 `✦ AI로 생성하기` 클릭 → `handleGenerateEra("ko")` 즉시 호출(기본값 한국어).
- 언어 전환은 `l3-era` 상세 페이지(§3.4 `EraContextPage`) 내부의 `한국어 / 中文` 토글에서만 수행. 아직 생성되지 않은 언어를 선택하면 그 자리에서 자동으로 `handleGenerateEra(targetLang)` 호출.
- error 재시도 버튼도 곧바로 `handleGenerateEra("ko")` 재실행.

### 2.5b `l2-lyric` 4카드 — 모두 `AiGenCard`, 기본 한국어

`가사 심화`(`l2-lyric`) 서브뷰 역시 `grid grid-cols-2 gap-3.5` 안에 4개 카드를 배치하며, **4개 모두 동일한 `<AiGenCard>`** 를 사용한다(카드 위 언어 피커/모드 스위처를 두지 않는다). 언어·모드 전환은 각 l3 상세 페이지 내부에서만 수행한다.

| icon | title | desc | onGenerate | onOpen |
|---|---|---|---|---|
| 🌏 | `문화 비교` | `중국어와 한국어에서 같은 주제를 어떻게 다르게 표현하는지 비교합니다` | `handleGenerateCulture("ko")` (기본 한국어) | `nav("l3-culture")` |
| 💡 | `가사 속 비유/상징` | `표면적 의미 너머 숨겨진 상징과 행간을 읽어냅니다` | `handleGenerateMetaphor()` | `nav("l3-metaphor")` |
| 📅 | `시대 언어 특징` | `이 곡이 발매된 시대의 유행어와 특유의 표현 방식을 살펴봅니다` | `handleGenerateEraLang()` | `nav("l3-era-lang")` |
| 💜 | `감정 분석` | `곡의 감정 흐름과 핵심 어휘, 한중 감정 표현을 비교 분석합니다` | `handleGenerateEmotion("ko")` (기본 한국어) | `nav("l3-emotion")` |

- 문화 비교 · 감정 분석은 이전 버전에 존재하던 **카드 내 `언어 선택: 한국어 / 中文` 피커를 제거**한다. 카드 클릭은 항상 한국어로 즉시 생성을 트리거하며, 언어(문화 비교) 또는 학습 방향(감정 분석)은 `l3-culture` / `l3-emotion` 상세 화면 내부의 토글에서만 전환한다. 아직 생성되지 않은 언어/모드로 전환 시에는 상세 화면에서 자동으로 재생성을 수행한다.
- error 재시도 버튼도 곧바로 동일한 기본 언어로 재실행한다(`handleGenerateCulture("ko")` / `handleGenerateEmotion("ko")`).
- 비유/상징과 시대 언어 특징은 언어 파라미터가 없거나 상세 페이지에서 자체적으로 관리하므로 카드 시그니처는 그대로 유지한다.




### 2.6 진행 시뮬레이션 규칙(엄격)

`startProgress(setter, ref)`:
1. 시작 시 즉시 `setter(5)`.
2. `setInterval` 300 ms 간격.
3. tick 마다 `p >= 85` 이면 return(멈춤). 그 외:
   - `p < 30` → `+3`
   - `p < 60` → `+2`
   - `else`    → `+0.5`
   - 상한 85로 clamp.
4. 성공 응답 도착 시 `stopProgress` + `setter(100)`.
5. 실패 시 `stopProgress` + 상태 error 로 전이(진행률 그대로 두어도 무방하지만 UI는 error 브랜치로 갈아탐).

### 2.7 캐시 & 상태 하이드레이션

- Table: `public.song_feature_cache(id uuid pk, song_id uuid fk, feature_type text, content jsonb)`. UNIQUE `(song_id, feature_type)`.
- `feature_type` enum(문자열): `artist_story | production_background | release_reaction | era_context | culture_compare | metaphor_symbol | era_language | emotion_analysis`.
- `loadCache<T>(songId, featureType)` → `content` 반환 or null.
- `saveCache(songId, featureType, content)` → 존재하면 update, 없으면 insert.
- `ExploreTab` 마운트 시 `Promise.all` 로 8개 캐시 병렬 로드. 검증 통과분만 `done` 으로 시딩.
- **artist/production/reception 은 단일 JSON 안에 `ko` 와 `zh` 를 함께 저장**(양언어 콘텐츠). era_context 는 `{ ko?: EraContextData, zh?: EraContextData }` 맵(언어별 독립).

### 2.8 데이터 계약(정확한 타입)

```ts
export type FeatureKey =
  | "artist_story" | "production_background" | "release_reaction"
  | "era_context" | "culture_compare" | "metaphor_symbol"
  | "era_language" | "emotion_analysis";
export type GenState = "idle" | "generating" | "done" | "error";

export interface ArtistStoryData {
  profile_timeline?: {
    type: "group" | "solo";
    verified_year?: string;
    ko: { profile: any; timeline: TimelineItem[] };
    zh: { profile: any; timeline: TimelineItem[] };
  };
  music_style?: { ko: { style: string; song_meaning: string }; zh: {...} };
  fun_facts?:   { ko: { facts: string[] }; zh: { facts: string[] } };
}
interface TimelineItem { year: string; event: string; description: string; highlight: boolean; }

export interface ProductionData {
  verified_year?: string;
  ko: { production: { intent: string; lyrics_origin: string } };
  zh: { production: { intent: string; lyrics_origin: string } };
}

export interface ReceptionData {
  verified_year?: string;
  ko: { reception: { stats: { chart?: string; sales?: string; year?: string }; public_reaction: string; career_position: string } };
  zh: { reception: {...} };
}

export interface EraContextData {
  verified_year?: string;
  basic: {
    music_culture: { korea: string; china: string };
    economy:       { korea: string; china: string };
    lifestyle:     { korea: string; china: string };
    media:         { korea: string; china: string };
    connection: string;
  };
  story: string;
}
```

**검증 함수(단순 truthy 체크, 실패 시 캐시 저장 금지)**:
- `isValidArtistStory`: `profile_timeline` 이 있으면 `ko.timeline`, `zh.timeline`, `type` 필수. 그 외에는 세 필드 중 하나라도 있으면 통과.
- `isValidProduction`: `d.ko.production.intent && d.zh.production.intent`.
- `isValidReception`: `d.ko.reception.public_reaction && d.zh.reception.public_reaction`.
- `isValidEraContext`: `basic.{music_culture|economy|lifestyle|media}.korea` 모두 truthy 및 `basic.connection` 및 `story` 필수.

### 2.9 Edge function `generate-artist-story` (단일, action 라우터)

- body: `{ action, song_title, artist_name, release_year, verified_year?, output_lang?, lyrics?, topic_name?, related_lyrics? }`
- 지원 action: `profile_timeline` · `music_style` · `fun_facts` · `production_reception` · `era_context` · `culture_topics` · `culture_detail` · `metaphor_expressions` · `metaphor_detail` · `era_language`.
- 모델: `gpt-4o-mini`, 429 에러 재시도 3회(지수 백오프 500→1000→2000 ms).
- `production_reception` 은 한 번의 호출로 `{ ko: { production, reception }, zh: { production, reception }, verified_year }` 반환 → 프론트에서 `generateProduction` / `generateReception` 이 각자 필요한 부분만 잘라 저장.
- `era_context` 는 `output_lang` 을 받아 해당 언어의 4카테고리 + story 를 반환(카테고리 내부 korea/china 문자열은 그 언어로).
- 응답 표준: `{ result: <object> }` 또는 실패 시 `{ error: string }`.

### 2.10 l3 페이지 4종 — 공통 뼈대

모두 다음을 준수:
- 상단에 뒤로가기 버튼(라벨 `← 곡 정보`), 아래 헤더 행: 좌측 `text-xl font-bold` 제목 + 서브텍스트 `{artistName} — {songTitle}`; 우측 `<LangToggle lang zh>` (데이터 있을 때만 표시). 
- `LangToggle`: `inline-flex rounded-lg border overflow-hidden`, 2 버튼(`한국어`, `中文`), 활성 = `bg-primary text-primary-foreground`, 비활성 = `bg-card text-muted-foreground hover:bg-accent`.
- 하단 액션 바: `flex justify-end gap-2.5 mt-6 pt-4 border-t`, `variant="outline"` [`↻ 전체 재생성`], `variant="default"` [`✓ 저장`]. 저장은 `saveCache` 호출 후 toast(`저장 완료`).
- draftData prop 존재 시 캐시 로드 스킵(l2-info 에서 방금 생성한 결과를 그대로 넘겨받는 경로).

**ArtistStoryPage** (`src/components/songs/ArtistStoryPage.tsx`, ~370줄):
- `<Accordion type="multiple">`, 4개 아이템: `profile / timeline / style / facts`. 기본 열림 `["profile", "timeline"]`.
- **레이지 생성**: `style` 또는 `facts` 아코디언이 열렸는데 데이터가 없으면 즉시 해당 action(`music_style` / `fun_facts`) 만 호출해서 부분 업데이트. 로컬 `loadingSection: Set<string>` / `errorSection: Record<string, string>` 로 섹션 단위 spinner/error.
- `ProfileSection`: `type` 이 `group` 이면 `debut_year / agency / active_period / members[]` 를, `solo` 면 `real_name / birth_year / debut_year / agency / other_activities` 를 표시. members 는 이름 + `— 역할`.
- `TimelineSection`: 좌측 `border-l-2` + 각 지점 `absolute -left-[31px] w-3.5 h-3.5 rounded-full border-2`. `highlight=true` 는 앰버(별표 접두 `★ `, `bg-amber-400 border-amber-500`, 텍스트 `text-amber-600/700`), 그 외 기본 색.
- `MusicStyleSection`: 🎶 음악적 특징 / 💎 이 곡의 의미 2 블록, 사이에 `border-t`.
- `FunFactsSection`: `<ul>`, 각 항목 `text-amber-500 ●` bullet.
- 헤더 아이콘 `👤`.

**ProductionBackgroundPage** (`src/components/songs/ProductionBackgroundPage.tsx`, ~150줄):
- 헤더 아이콘 `✍️` 제목 `제작 배경`.
- 콘텐츠: 단일 `border rounded-xl bg-card px-6 py-6` 안에 2 섹션:
  1. `🎯 창작 의도 / 创作意图` — `p.intent`.
  2. `📖 가사 탄생 배경 / 歌词诞生背景` — `p.lyrics_origin`.
  사이에 `border-t`.
- 로딩/에러/빈 상태 표준화(`<Loader2>` + `생성 중...`, `<AlertCircle>` + `다시 시도`, `EmptyState` 는 "아직 생성된 내용이 없습니다. 곡 정보 페이지에서 'AI로 생성하기'를 먼저 클릭해주세요.").
- 전체 재생성 시 `generateProduction()` 재호출.

**ReleaseReactionPage** (`src/components/songs/ReleaseReactionPage.tsx`, ~170줄):
- 헤더 아이콘 `📊` 제목 `발매 당시 반응`.
- `stats` 가 있으면 3개까지의 `StatCard`(`flex-1 bg-card border rounded-xl p-4 text-center`): `chart → 차트 순위/排行榜排名`, `sales → 판매량/销量`, `year → 발매연도/发行年份`. 있는 필드만 렌더.
- 아래 2섹션: `💬 당시 평가 / 当时的评价`(`public_reaction`), `🔗 커리어에서의 위치 / 在职业生涯中的地位`(`career_position`), 사이 `border-t`.
- 나머지 로딩/에러/저장 규격은 ProductionBackgroundPage 와 동일.

**EraContextPage** (`src/components/songs/EraContextPage.tsx`, ~290줄):
- 헤더 아이콘 `🕰️` 제목 `시대 배경 ({currentData.verified_year})`.
- 상태: `data: Record<"ko"|"zh", EraContextData>`, `lang`, `storyMode: boolean`, `isReading: boolean`.
- 헤더 우측: `[한국어 ✓]` `[中文 ✓]` (활성 표기), 그리고 **스토리 버튼**:
  - 아이들: `📖 스토리로 읽기`. storyMode 켜져 있으면 `📖 기본 모드`. 읽는 중이면 `🔊 읽는중...` + `<SoundWave/>` + `animate-pulse`.
  - **단일 클릭 = TTS 토글** (`speechSynthesis` 로 `story` 를 `lang="ko-KR" | "zh-CN"` rate 0.9 로 발화).
  - **더블 클릭 = storyMode 토글**. 구현은 `clickTimerRef` 를 두고 250 ms 안에 두 번째 클릭 오면 double, 아니면 single.
- 언어 스위칭: 대상 언어에 데이터 있으면 그냥 스위칭, 없으면 즉시 `handleGenerate(targetLang)` 실행(spinner 표시). 스위칭 시 진행 중이던 TTS 는 취소.
- 컨텐츠 렌더:
  - **기본 모드**: 4카테고리 카드(`music_culture / economy / lifestyle / media`), 각 카드 `p-5` 안에 `🎵 음악/대중문화` 등의 헤더 + `grid grid-cols-2 gap-5` (🇰🇷 한국 / 🇨🇳 중국). 이어서 `💡 이 노래와의 연결 / 这首歌的联系` 카드 하나.
  - **스토리 모드**: 단일 카드 `📖 스토리로 읽기 / 故事阅读` + `whitespace-pre-line` 텍스트.
- `SoundWave`: 4개의 `w-[3px] rounded-full bg-primary-foreground` 바가 `soundBar 0.6s ease-in-out ${i*0.15}s infinite alternate` 로 4→14 px 사이를 튀는 인라인 애니메이션(`<style>` 태그 임베드).
- unmount / 언어전환 / storyMode 전환 시 `window.speechSynthesis.cancel()` 필수.
- `handleSave` 는 기존 캐시와 병합해서 `{ ko?, zh? }` 를 유지.

### 2.11 강제 제약

- `#2e3d6b`, `#8fa3c8`, `bg-amber-500` 은 사이드바 브랜드 전용. 그 외 신규 색상 금지 — 반드시 semantic token 또는 카테고리별로 이미 지정된 tag/accent 클래스.
- 뷰 전환은 URL 을 건드리지 않는다(다이얼로그 내부 로컬 상태만).
- `isValid*` 를 통과하지 못한 결과는 절대 캐시에 저장하지 않는다(edge 응답이 부분적일 때 방어).
- `speechSynthesis` 는 페이지 이탈 시 반드시 `cancel()` — 리스너 누수 금지.
- 진행률 상한 85, tick 300 ms, 증분 3/2/0.5 를 임의로 바꾸지 않는다.
- l3 페이지는 자체적으로 처음부터 새로 생성하려 하지 않는다(전체 재생성 버튼 예외). 초기 진입은 `draftData` 또는 캐시로만 채워진다.

## ③ Examples

### 3.1 확정 카피 표

| 위치 | 카피 |
|---|---|
| 사이드바 5버튼 | `탐구 홈` / `곡 정보` / `가사 심화` / `연습` / `수업 도구` |
| 허브 헤더 | 제목 `탐구` / 서브 `이 곡을 더 깊게 파고들어보세요` |
| 허브 카드 4개 태그 | `배경 탐구` / `언어 분석` / `학습 활동` / `선생님 전용` |
| l2-info 헤더 | 제목 `📖 곡 정보` / 서브 `아티스트와 이 곡의 배경을 탐구해보세요` |
| l2-info 4카드 제목 | `아티스트 스토리` / `제작 배경` / `발매 당시 반응` / `시대 배경` |
| l2-info 4카드 아이콘 | 👤 · ✍️ · 📈 · 🕰️ |
| 시대 배경 l3 언어 토글 | `한국어` / `中文` (상세 페이지 내부에서만) |
| 생성 버튼 | `✦ AI로 생성하기` |
| 진행 중 | `생성 중... {n}%` |
| 완료 버튼 | `생성 완료 · 페이지 열기` |
| 오류 | `{에러메시지 || "생성 실패"}` / 재시도 `↺ 다시 시도` |
| l3 뒤로가기 | `← 곡 정보` |
| l3 상단 액션 | `↻ 전체 재생성` · `✓ 저장` |
| l3 저장 성공 토스트 | `저장 완료` (Production/Reception) / `저장되었습니다` (Era) |
| Empty state (Prod/Rec) | `아직 생성된 내용이 없습니다.` + `곡 정보 페이지에서 "AI로 생성하기"를 먼저 클릭해주세요.` |
| Empty state (Era) | `데이터가 없습니다. 곡 정보 페이지에서 먼저 생성해주세요.` |
| ArtistStory 아코디언 | `👤 아티스트 프로필` · `📅 타임라인` · `🎵 음악 스타일` · `💡 알고 계셨나요?` |
| Prod 섹션 | `🎯 창작 의도 / 创作意图` · `📖 가사 탄생 배경 / 歌词诞生背景` |
| Reception 섹션 | `💬 당시 평가 / 当时的评价` · `🔗 커리어에서의 위치 / 在职业生涯中的地位` |
| Reception StatCard 라벨 | `차트 순위/排行榜排名` · `판매량/销量` · `발매연도/发行年份` |
| Era 카테고리 | `🎵 음악/대중문화` · `💰 사회/경제` · `👗 라이프스타일` · `📺 미디어/기술` |
| Era 연결 카드 | `💡 이 노래와의 연결 / 这首歌的联系` |
| Era 스토리 버튼 | `📖 스토리로 읽기` / `📖 기본 모드` / `🔊 읽는중...` |

### 3.2 컴포넌트 트리

```text
<ExploreTab songId songTitle artistName>
  ├─ <ExploreSidebar activeSection onNavigate/>
  └─ <main flex-1 overflow-y-auto>
       switch(currentView):
        case "hub"     → <HubGrid/>  (4 HubCards)
        case "l2-info" → <BackButton "탐구"/> + <PageHeader/> + grid:
              ├─ <AiGenCard 👤 …/> onGenerate=handleGenerateArtist   onOpen=nav("l3-artist")
              ├─ <AiGenCard ✍️ …/> onGenerate=handleGenerateProd     onOpen=nav("l3-background")
              ├─ <AiGenCard 📈 …/> onGenerate=handleGenerateReaction onOpen=nav("l3-reaction")
              └─ <AiGenCard 🕰️ …/> (onGenerate → handleGenerateEra("ko"), 언어 전환은 l3에서)
        case "l3-artist"     → <ArtistStoryPage draftData={artistData}/>
        case "l3-background" → <ProductionBackgroundPage draftData={prodData}/>
        case "l3-reaction"   → <ReleaseReactionPage draftData={reactionData}/>
        case "l3-era"        → <EraContextPage draftData={eraData} draftLang={eraLang}/>
        case "l2-lyric" | "l2-practice" | "l2-class" → 각각 P3o / P3p / P3q
```

### 3.3 생성 → 열기 시나리오 (아티스트)

```
[초기] artistState = "idle"
    │  사용자가 [✦ AI로 생성하기] 클릭
    ▼
setArtistState("generating"); startProgress(setArtistProgress, artistIntervalRef);
    │  generateArtistStory(songTitle, artistName) 병렬 3 action 호출
    ▼
[성공] stopProgress; setArtistProgress(100); setArtistData(data); setArtistState("done");
    │  카드가 [✔ 생성 완료 · 페이지 열기] 로 스와핑
    ▼
사용자 클릭 → nav("l3-artist") → <ArtistStoryPage draftData={artistData}/> 렌더
```

## ④ Context

### 4.1 프로젝트 맥락

탐구 탭은 곡의 **배경 · 언어 · 연습 · 수업 도구** 4축으로 학습을 심화한다. 본 프롬프트는 그중 상단 셸(사이드바 + 허브 + l2 서브그리드) 과 첫 번째 축 "곡 정보"의 4모듈까지만을 다룬다. 나머지 3축(가사 심화 · 연습 · 수업 도구) 은 각각 P3o / P3p / P3q 에서 이어서 정의한다.

### 4.2 Lovable Cloud 후경

- 캐시 테이블 `song_feature_cache`, RLS 는 소유자 및 링크된 학생만 SELECT/INSERT/UPDATE 가능(P6 인프라 프롬프트의 표준 정책 상속).
- edge functions: `generate-artist-story`(단일, action 라우터) + `generate-emotion-analysis`(별도, 감정 흐름/어휘/비교). 본 프롬프트 범위는 전자 하나만.
- 모델: `gpt-4o-mini` 고정. 페르소나: "한국의 대학에서 중국어를 가르치는 교수". 응답은 반드시 JSON.
- 429 에러 시 3회 지수 백오프 재시도, 그래도 실패하면 `{ error: "..." }` 로 프론트에 넘긴다.

### 4.3 UX 원칙

- **낙관적 UI 이동 금지**: 생성이 끝나기 전에는 l3 로 이동시키지 않는다(카드에서 대기).
- **부분 재시도 우선**: ArtistStory 는 섹션(스타일/사실) 단위 lazy 생성 + 섹션별 에러/재시도.
- **양언어 병기**: artist/production/reception 은 한 번의 편집에 한/중 결과가 함께 저장된다. era 만 언어별 독립 캐시.
- **음성은 사용자가 통제**: 페이지 이탈 · 언어 스위칭 · storyMode 전환 · 언마운트 시 즉시 `speechSynthesis.cancel()`.

## ⑤ Acceptance & Output

### 5.1 Acceptance Criteria

- 5개 사이드바 버튼 모두 클릭 시 200 ms 내 뷰 전환(fetch 0회, 로컬 상태만).
- `getActiveSection` 이 22종 view 모두를 정확한 section 으로 분류(테이블 기반 단위 테스트 22 케이스 pass).
- 캐시가 있는 곡 재진입: fetch 8건 병렬 → 100 ms 내 카드 4장 모두 `done` 상태로 시딩되고, 열기 클릭 시 l3 페이지가 캐시로 즉시 렌더(추가 fetch 0회).
- 진행 시뮬레이션: 시작 300 ms 내 5→8% 관측, 3 초 시점에 45% ±5, 이후 85% 상한에서 정확히 정지. 응답 도착 300 ms 내 100% 표시 후 done 전환.
- `isValid*` 실패 시 캐시 미저장 — 다음 리로드 시 여전히 `idle`.
- 사이드바 정확히 56 px 폭, 활성 버튼 배경 `bg-white/10`, 로고 배경 `bg-amber-500`, 구분선 `w-7 h-px bg-white/10`.
- 시대 배경 카드: 다른 3개 카드와 완전히 동일하게 동작(idle 클릭 → 즉시 `handleGenerateEra("ko")`로 generating 진입). 언어 전환/재생성은 l3 `EraContextPage` 내부의 `한국어/中文` 토글에서 처리.
- EraContextPage 스토리 버튼: 단일 클릭 시 250 ms 후 TTS 시작, 그 안에 재클릭 시 storyMode 토글로 승격. 어느 경우든 이전 TTS 는 `cancel()`.
- ArtistStoryPage `style` / `facts` 아코디언을 처음 펼치면 부분 fetch 1회 실행, 두 번째 펼침에는 fetch 없음.

### 5.2 Output Format

반환 순서(파일 단위, 각 파일은 완결 코드):
1. `src/components/songs/artistStoryHelpers.ts`
2. `supabase/functions/generate-artist-story/index.ts`
3. `src/components/songs/ArtistStoryPage.tsx`
4. `src/components/songs/ProductionBackgroundPage.tsx`
5. `src/components/songs/ReleaseReactionPage.tsx`
6. `src/components/songs/EraContextPage.tsx`
7. `src/components/songs/ExploreTab.tsx`
8. 한국어 3줄 요약 — (a) 무엇을 만들었는지, (b) 생성이 l2 카드에서 일어나고 l3 는 표시 전용이라는 점, (c) 캐시/저장 정책.
