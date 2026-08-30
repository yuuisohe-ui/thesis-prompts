# P3o · 탐구 · 가사 심화 4모듈 재현 프롬프트

> **본 프롬프트는 P3 시리즈의 15/17.**
> **적용 대상**:
> - `src/components/songs/CultureComparePage.tsx`
> - `src/components/songs/MetaphorSymbolPage.tsx`
> - `src/components/songs/EraLanguagePage.tsx`
> - `src/components/songs/EmotionAnalysisPage.tsx`
> - `src/components/songs/artistStoryHelpers.ts` (부분 — 아래 함수만)
> - edge function `generate-artist-story` (actions: `culture_topics`, `culture_detail`, `metaphor_expressions`, `metaphor_detail`, `era_language`)
> - edge function `generate-emotion-analysis` (actions: `flow`, `vocab`, `compare`)
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**
> **주의**: 본 4개 페이지는 모두 P3n(탐구 셸) 의 `LyricPickerPage` 에서 카드 클릭으로 진입하는 l3 상세이다. 진입 시점에 부모(l2)가 `draftData/draftFlow/draftLang` 로 초기 데이터를 넘겨줄 수 있다.
> **l2 카드 규약 (P3n §2.5b 와 동일)**: `가사 속 비유`·`시대 언어 특징`·`문화 비교`·`감정 분석` 4장 모두 `AiGenCard` 로 렌더되어 클릭 시 **즉시 한국어(`ko`) 로 생성 + 진행바** 를 노출한다. l2 카드 안에는 언어/모드 선택 UI 가 없으며, 언어(문화·비유·시대) 또는 학습 방향(감정) 전환은 **오직 l3 상세 페이지 내부의 스위처** 에서만 수행된다. 따라서 l3 진입 시 `draftLang`/`draftFlow` 은 항상 `"ko"` 기준으로 도달한다고 가정해도 무방하다.

---

## 이론적 근거

1. **OpenAI. (n.d.).** https://platform.openai.com/docs/guides/prompt-engineering (Retrieved July 12, 2026)
2. **Lovable. (n.d.).** https://docs.lovable.dev/prompting/prompting-one (Retrieved July 12, 2026)
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity

당신은 가사 텍스트를 언어·문화 축으로 심화 분석하는 4개 상세 페이지를 만드는 시니어 프론트엔드 엔지니어이자, 응용언어학·문화연구·정서분석을 겸비한 UX 라이터입니다. 두 언어(한국어·중국어) 사이의 대응·차이를 시각적으로 대비해 보여주고, 문화·비유·시대성·정서 흐름을 학습자에게 이해하기 쉬운 카드·아코디언·표 구조로 재구성합니다.

## ② Instructions

### 2.1 산출물 파일 목록 (정확한 경로)

1. `supabase/functions/generate-artist-story/index.ts` — 아래 5개 action 을 라우팅해야 함(다른 action 은 P3n 소관, 병존해도 됨).
2. `supabase/functions/generate-emotion-analysis/index.ts` — 3개 action.
3. `src/components/songs/artistStoryHelpers.ts` — `callEdge` / `callEmotionEdge` 및 6개 generator 함수, 4개 `isValid*` 검증 함수, 대응 타입.
4. `src/components/songs/CultureComparePage.tsx`
5. `src/components/songs/MetaphorSymbolPage.tsx`
6. `src/components/songs/EraLanguagePage.tsx`
7. `src/components/songs/EmotionAnalysisPage.tsx`

### 2.2 공통 UI/UX 규약

- 상단 좌측: `← 가사 심화` 뒤로가기 버튼(`text-[13px] text-muted-foreground border border-border bg-card px-3 py-1.5 rounded-lg`, hover 시 primary).
- 상단 우측: 언어/모드 스위처. **버튼 2개**(`Button size="sm" variant={active ? "default" : "outline"} className="text-xs h-8"`). 활성 버튼 라벨 끝에 `✓` 접미사.
- 하단: `flex justify-end gap-2 mt-6 pt-3.5 border-t border-border` 안에 `↺ 재생성` 과 `✓ 저장` 두 버튼. 저장 중이면 `Loader2 animate-spin`.
- 상세 카드 컨테이너: `border border-border rounded-xl overflow-hidden bg-card shadow-sm`.
- 로딩: `Loader2 h-4 w-4 animate-spin` + `분석 중...` 텍스트 (중앙 정렬, `py-8`).
- 오류: `mb-3 px-3 py-2 rounded-lg border border-destructive/30 bg-destructive/5 text-xs text-destructive`.

### 2.3 CultureComparePage — 아코디언 기반, 토픽 단위 지연 로딩

- Props: `{ songId, songTitle, artistName, onBack, draftData?: CultureTopicsData, draftLang?: "ko"|"zh" }`.
- 상태: `lang`(초기 `draftLang || "ko"`), `topics`(초기 `draftData?.topics || []`), `details: Record<string, CultureDetailData>`, `loadingTopics: Set<string>`, `openItems: string[]`, `saving`.
- Mount 시 `loadCache(songId, "culture_compare")` → `{[lang]: {topics, details}}` 구조로 저장되므로 `lang` 축으로 하나만 hydrate.
- 자동 확장: `topics.length > 0 && openItems.length === 0` 이면 `topics[0].topic_name` 을 자동 open.
- 지연 로딩: `openItems` 배열의 각 topicName 에 대해 `!details[name] && !loadingTopics.has(name)` 이면 `generateCultureDetail(...)` 호출 후 결과 저장. 재진입 시 캐시 hit.
- 언어 스위칭 `handleLangSwitch`:
  - 상태 초기화 후 `loadCache` 재조회 → 새 언어 데이터 있으면 즉시 표시, 없으면 topics 만 남고 detail 은 open 시 lazy 생성.
- **상세 카드 내부 구조** (`AccordionContent px-5 pb-5 space-y-6`):
  1. `📝 관련 가사` — `topic.related_lyrics` 를 `bg-primary/5 border border-primary/20 text-primary text-xs px-3 py-1.5 rounded-lg` chip.
  2. `🇨🇳 중국어 문화권` panel — `bg-red-50/50 dark:bg-red-950/20 border border-red-200/50 dark:border-red-800/30 rounded-xl p-4`. 3섹션: `언어적 표현 방식` · `문화적 배경` · `현대적 변화`.
  3. `🇰🇷 한국어 문화권` panel — 동일 스타일이나 blue 계열. 3섹션: `언어적 표현 방식` · `문화적 배경` · `이 곡에서의 표현`.
  4. `🔄 두 언어 비교표` — `<table>` (`w-full text-sm`, `border border-border rounded-xl overflow-hidden`). 헤더 `bg-muted/50`, 3열(빈 라벨 / `중국어` / `한국어`). 행 4개: `emotion_expression → 감정표현`, `grammar → 문법형태`, `cultural_background → 문화배경`, `intensity → 강도표현`. 값 없으면 `-`.
  5. `💡 학습 포인트` — amber panel (`bg-amber-50/50 dark:bg-amber-950/20 border border-amber-200/50 dark:border-amber-800/30 rounded-xl p-4`).
- 헤더 카피: `🌏 문화 비교` · `가사 속 주제를 중국어·한국어 문화권에서 비교합니다`.
- 서브 헤더 카피: `핵심 주제` (`text-sm font-semibold text-muted-foreground mb-3`).
- 재생성: `details = {}`, `openItems = []`, 50 ms 뒤 첫 topic 을 다시 open (자동 지연 로딩 트리거).
- 저장: 기존 캐시와 병합 → `{...existing, [lang]: {topics, details}}` 로 `saveCache`.

### 2.4 MetaphorSymbolPage — 아코디언, ko/zh 상세 병존 저장

- Props: `{ songId, songTitle, artistName, onBack, draftData?: MetaphorExpressionsData }`.
- 상태: `lang`("ko" 기본), `expressions`, `koDetails`, `zhDetails`(둘 다 독립 캐시), `loadingExprs`, `openItems`, `saving`. `details = lang==="ko" ? koDetails : zhDetails`.
- Mount 시 `loadCache(songId, "metaphor_symbol")` → `{expressions, ko:{details}, zh:{details}}` 구조.
- 자동 확장: 첫 expression 자동 open. `openItems` × `lang` 조합으로 지연 로딩(`generateMetaphorDetail(songTitle, artistName, expr.expression_name, expr.lyrics_ko, expr.lyrics_zh, lang)`).
- **상세 카드 내부 구조** (`space-y-6`):
  1. `📝 실제 가사` — `expr.lyrics_ko` 과 `expr.lyrics_zh` 두 줄, 이탤릭, `"..."` 로 감쌈.
  2. `표면적 의미` / `실제 의미` — `grid grid-cols-2 gap-3`. 좌: `bg-muted/30 border` + 👁️, 우: `bg-primary/5 border-primary/20` + 💡.
  3. `사용된 기법` — `detail.techniques[]` 가 있을 때만. `Badge variant="secondary" cursor-help` + Tooltip(`side="top" max-w-[240px]`)으로 `t.description` 노출.
  4. 한국어/중국어 뉘앙스 — `grid grid-cols-2 gap-3`, KR blue panel + CN red panel, `text-sm leading-relaxed`.
  5. `💬 학습 포인트` — amber panel.
- 헤더 카피: `💡 가사 속 비유/상징` · `표면적 의미 너머 숨겨진 상징과 행간을 읽어냅니다`.
- 서브 헤더: `비유/상징 표현`.
- 언어 스위칭: `openItems` 리셋 후 50 ms 뒤 첫 expression 재 open. `koDetails`/`zhDetails` 는 각각 유지되므로 이미 생성된 언어는 즉시 표시.
- 저장: `{...existing, expressions, ko: {details: {...existing.ko.details, ...koDetails}}, zh: {...}}`.

### 2.5 EraLanguagePage — 단일 정보 페이지, 자동 생성

- Props: `{ songId, songTitle, artistName, onBack, draftData?: EraLanguageData }`.
- 상태: `lang`, `koCache`, `zhCache`, `loading`, `regenerating`. `currentData = lang==="ko" ? koCache : zhCache`.
- Mount 시 `loadCache(songId, "era_language")` → `{ko?, zh?}`. `draftData` 는 ko 캐시가 없을 때만 ko 로 hydrate.
- 자동 생성 트리거: `useEffect([lang, currentData])` 안에서 `!currentData && !loading` 이면 즉시 `generate(lang)` 호출. 내부에서 `songs.lyrics_raw` 조회 후 `generateEraLanguage(...)`.
- **최초 로딩 화면**: `loading && !currentData` → 전용 스켈레톤 (뒤로가기 + 중앙 스피너 + `시대 언어 특징을 생성하고 있습니다...`).
- 언어 스위칭 중 다른 쪽 생성 중일 때: 상단 아래에 `mb-4 text-xs text-primary` 로 `다른 언어 버전을 생성하고 있습니다...` 표시.
- **본문 구조**:
  - H2 `📅 {verified_year}년의 언어`.
  - Card 1: `🇰🇷 한국어 시대 특징` — `era_expressions[]` 를 `Badge variant="secondary" cursor-help` + Tooltip(설명, `max-w-[250px]`). 하단 `era_description` 문단.
  - Card 2: `🇨🇳 중국어 시대 특징` — 동일 구조.
  - Card 3 (조건부): `has_lyric_features === true && expressions.length > 0` → `📝 가사 속 시대 언어` Accordion `type="single" collapsible defaultValue="expr-0"`. 각 아이템:
    - Trigger: `expression_name`.
    - Body: 이탤릭 인용 `"{lyrics}"`, `grid grid-cols-2 gap-3` — 좌 `그때 표현`, 우 `지금 표현` (`bg-muted/50 rounded-lg p-3`), 그 뒤 `description`.
  - Card 4: `💬 학습 포인트`.
- 하단: 재생성(스피너 표시) / 저장. 저장 시 `merged = {ko?: koCache, zh?: zhCache}` 만 포함.
- 저장 후 toast `저장 완료 / 시대 언어 특징이 저장되었습니다.`.

### 2.6 EmotionAnalysisPage — 학습 모드 시스템 (핵심)

**모드 정의** (반드시 이 매핑을 유지):

| Mode | 학습 언어 | 출력 표시 언어 | 대상 학습자 |
|---|---|---|---|
| `ko` | 한국어(learning) | **中文** | 중국인 학생이 한국어를 학습 |
| `zh` | 중국어(learning) | **한국어** | 한국인 학생이 중국어를 학습 |

- Props: `{ songId, songTitle, artistName, onBack, draftFlow?: EmotionFlowData, draftLang?: "ko"|"zh" }`.
- 상태: `mode`(초기 `draftLang || "ko"`), `koData: EmotionAnalysisLangData`, `zhData: ...` 각각 `{flow?, vocab?, compare?}` 부분 저장. `openItems`(초기 `["flow"]`), `flowLoading`/`vocabLoading`/`compareLoading`, `saving`, `error`.
- 파생값:
  - `outputLang = mode === "ko" ? "zh" : "ko"`
  - `learningLang = mode === "ko" ? "korean" : "chinese"`
  - `t = T[outputLang]` — 아래 카피 표 참조.
- Mount 시 `loadCache(songId, "emotion_analysis")` → `{ko?, zh?}`.

**섹션 1: 🌊 전체 감정 흐름** (`value="flow"`, 기본 open)
- `flow[]` 각 아이템을 렌더. **단계 색 매핑**은 index → `["intro","develop","climax","end"][min(i,3)]` 로 결정.
- 색 팔레트(각 stage bg/border/text/dot):
  - `intro`: blue-50/60 → blue-500 dot.
  - `develop`: orange-50/60 → orange-500 dot.
  - `climax`: red-50/60 → red-500 dot.
  - `end`: green-50/60 → green-500 dot.
- 각 카드: `w-2.5 h-2.5 rounded-full` dot + `stage` 굵은 텍스트 + `Badge` 로 `emotion`. `border-l-2 pl-3` 로 `lyric` 인용(이탤릭) + `lyric_translated`(있으면 muted). 아래 `desc` 문단.
- 자동 트리거: mode 변경 시 `!data.flow && !flowLoading` 이면 `loadFlow()`.

**섹션 2: ✦ 핵심 감정 어휘** (`value="vocab"`)
- Accordion open 시에 lazy 로딩(`data.vocab` 없으면 `loadVocab()`).
- `loadVocab()` 흐름:
  1. `song_analyses.word_list` 조회.
  2. `generateEmotionVocab(songTitle, wordList, learningLang, outputLang)` 시도.
  3. AI 실패 또는 빈 결과 → **fallback 빌더** 사용:
     - `learningLang === "korean"` → `word = w.meaning_ko`, `meaning = w.word`(한자), `attr = "neu"`, `attr_label = t.neutralLabel`.
     - `learningLang === "chinese"` → `word = w.word`, `pinyin = w.pinyin`, `meaning = w.meaning_ko`, `attr = "neu"`.
     - 최대 12개.
- 렌더: `<table w-full text-sm>` 헤더 4열(`word` `pinyin`(있을 때만) `meaning` `attr`). `attr` 는 pos(green) / neg(red) / neu(muted) 색 pill(`ATTR_COLORS`), 라벨은 `attr_label` 사용.
- `showPinyinCol = data.vocab.vocab.some(v => v.pinyin)`.
- 빈 결과: `t.none` 문구.

**섹션 3: 🔀 한중 감정 표현 비교** (`value="compare"`)
- Accordion open 시 lazy 로딩(`generateEmotionCompare(...)`).
- 각 `compare[]` 항목:
  - `H4`: `{emoji || "💭"} {topic}`.
  - `grid grid-cols-2 gap-3` — 좌 KR blue panel(`🇰🇷 한국어`, `ko_expressions` bullet), 우 CN red panel(`🇨🇳 汉语`, `zh_expressions` bullet).
  - 아래 purple 박스(`bg-purple-50/60 border-purple-200/50 rounded-xl p-4`): `{t.diff} — {difference}`.

**카피 표 T** (반드시 이 정확한 카피 유지):

```ts
const T = {
  // outputLang === "zh" → 학생=중국인, 한국어 배우기, 화면 카피는 중국어
  ko: {
    title: "💜 情感分析",
    desc: "查看歌曲的情感流动、核心词汇与韩中表达对比",
    back: "歌词探究",
    s1: "整体情感流动",
    s2: "核心情感词汇",
    s3: "韩中情感表达对比",
    regen: "↺ 重新生成",
    save: "✓ 保存",
    loading: "分析中...",
    diff: "💡 差异",
    word: "韩语词", pinyin: "罗马音", meaning: "中文释义", attr: "情感",
    none: "未找到情感相关词汇。",
    btnLearnKo: "学韩语",
    btnLearnZh: "学汉语",
    neutralLabel: "中性",
  },
  // outputLang === "ko" → 학생=한국인, 중국어 배우기, 화면 카피는 한국어
  zh: {
    title: "💜 감정 분석",
    desc: "곡의 감정 흐름과 핵심 어휘, 한중 표현 비교를 살펴봅니다",
    back: "가사 심화",
    s1: "전체 감정 흐름",
    s2: "핵심 감정 어휘",
    s3: "한중 감정 표현 비교",
    regen: "↺ 재생성",
    save: "✓ 저장",
    loading: "분석 중...",
    diff: "💡 차이점",
    word: "단어", pinyin: "병음", meaning: "뜻", attr: "감정",
    none: "감정 관련 어휘를 찾지 못했습니다.",
    btnLearnKo: "한국어 배우기",
    btnLearnZh: "중국어 배우기",
    neutralLabel: "중립",
  },
};
```

**모드 스위칭**: `openItems = ["flow"]` 리셋. 각 언어 데이터(`koData`/`zhData`)는 유지되므로 이미 생성된 섹션은 즉시 표시.

**재생성**: `setData(() => ({}))` 로 현재 mode 데이터 전체 비우기, `openItems = ["flow"]`, 50 ms 뒤 `loadFlow()`.

**저장**: `existing = loadCache(...)`, `merged = {...existing, ko: mode==="ko" ? data : existing.ko, zh: mode==="zh" ? data : existing.zh}`.

### 2.7 강제 제약

- 하드코드 색 문자열은 위 팔레트 표 안에서만 사용, 그 외 컴포넌트 내부에서는 semantic token(`text-muted-foreground`, `border-border`, `bg-primary/5`, `text-primary` 등)만 사용.
- **Recharts / d3 / 차트 라이브러리 절대 사용 금지** — 감정 흐름은 컬러 카드 리스트로만 표현한다.
- 편의를 위해 dynamic `import("@/integrations/supabase/client")` 로 조회하는 부분(가사 원문 로드)은 EraLanguagePage 에서만 허용, EmotionAnalysisPage 는 상단 정적 import 사용.
- 캐시 저장 키는 `song_feature_cache.feature_type` 을 `culture_compare`/`metaphor_symbol`/`era_language`/`emotion_analysis` 로 고정.
- `generateEmotionVocab` 실패 시 fallback 은 반드시 **word_list 가 존재할 때만** 사용, 그 외에는 원래 오류를 상위로 전파.
- 각 재생성/저장 버튼은 저장 진행 중 disable + 스피너.

## ③ Examples

### 3.1 확정 카피 표 (Culture / Metaphor / Era)

| 위치 | 카피 |
|---|---|
| Culture 뒤로가기 | `← 가사 심화` |
| Culture H2 / desc | `🌏 문화 비교` / `가사 속 주제를 중국어·한국어 문화권에서 비교합니다` |
| Culture 섹션 헤더 | `핵심 주제` |
| Culture 카드 아이콘/제목 | `📝 관련 가사` · `🇨🇳 중국어 문화권` · `🇰🇷 한국어 문화권` · `🔄 두 언어 비교표` · `💡 학습 포인트` |
| Culture 비교표 컬럼 | `중국어` · `한국어` |
| Metaphor H2 / desc | `💡 가사 속 비유/상징` / `표면적 의미 너머 숨겨진 상징과 행간을 읽어냅니다` |
| Metaphor 카드 헤더 | `📝 실제 가사` · `👁️ 표면적 의미` · `💡 실제 의미` · `사용된 기법` · `🇰🇷 한국어` · `🇨🇳 중국어` · `💬 학습 포인트` |
| Era H2 | `📅 {verified_year}년의 언어` |
| Era 카드 헤더 | `🇰🇷 한국어 시대 특징` · `🇨🇳 중국어 시대 특징` · `📝 가사 속 시대 언어` · `💬 학습 포인트` |
| Era 좌/우 라벨 | `그때 표현` · `지금 표현` |
| 공통 하단 버튼 | `↺ 재생성` · `✓ 저장` |
| 공통 언어 스위처 | `한국어` · `中文` (활성 시 뒤에 `✓`) |

### 3.2 컴포넌트 트리

```text
<CultureComparePage>
  ├─ BackButton "← 가사 심화"
  ├─ Header + LangToggle(ko|zh)
  ├─ SubHeader "핵심 주제"
  └─ <Accordion type="multiple">
       └─ AccordionItem(topic_name)  × topics.length
            ├─ Trigger topic_name
            └─ Content:
                 ├─ 관련 가사 chips
                 ├─ 🇨🇳 panel (expression / culture / modern)
                 ├─ 🇰🇷 panel (expression / culture / song_expression)
                 ├─ 비교표(4행 × 2열)
                 └─ 학습 포인트

<MetaphorSymbolPage>
  ├─ Header + LangToggle
  └─ Accordion(multiple) → 각 표현 상세(가사·표/실 의미·기법·뉘앙스·학습)

<EraLanguagePage>
  ├─ Header "📅 {year}년의 언어" + LangToggle
  ├─ Card KR era features (badges + description)
  ├─ Card CN era features
  ├─ (조건부) Card 가사 속 시대 언어 (Accordion single)
  └─ Card 학습 포인트

<EmotionAnalysisPage>
  ├─ Header + ModeToggle(learn ko | learn zh)
  └─ <Accordion type="multiple" defaultValue=["flow"]>
       ├─ flow: stage 색 카드 리스트
       ├─ vocab: table(word·pinyin·meaning·attr)
       └─ compare: topic 단위 KR/CN grid + diff purple
```

### 3.3 최소 endpoint 계약

```ts
// generate-artist-story
action: "culture_topics"      → { topics: {topic_name, related_lyrics[]}[] }
action: "culture_detail"      → CultureDetailData { chinese:{expression,culture,modern}, korean:{expression,culture,song_expression}, comparison:{emotion_expression:{chinese,korean}, grammar, cultural_background, intensity}, learning_point }
action: "metaphor_expressions"→ { expressions: {expression_name, lyrics_ko, lyrics_zh}[] }
action: "metaphor_detail"     → MetaphorDetailData { surface_meaning, real_meaning, techniques:{name,description}[], korean_nuance, chinese_nuance, learning_point }
action: "era_language"        → EraLanguageData { verified_year, has_lyric_features, korean:{era_expressions:{expression,description}[], era_description}, chinese:{...}, expressions:{expression_name,lyrics,then,now,description}[], learning_point }

// generate-emotion-analysis
action: "flow"    → { flow: { stage, emotion, lyric, lyric_translated?, desc }[] }
action: "vocab"   → { vocab: { word, pinyin?, meaning, attr:"pos"|"neg"|"neu", attr_label }[] }
action: "compare" → { compare: { topic, emoji?, ko_expressions[], zh_expressions[], difference }[] }
```

## ④ Context

### 4.1 프로젝트 맥락

- 4개 페이지는 P3n(탐구 셸)의 l3 상세이다. `LyricPickerPage`(l2)에서 카드 클릭 시 각 페이지로 전환.
- 부모(l2)가 사전 생성한 topics/expressions/flow 데이터를 `draftData`/`draftFlow` 로 넘겨받아 초기 렌더를 즉시 수행하고, 상세는 lazy 생성한다(문화·비유). 시대 언어는 마운트 시 즉시 자동 생성, 감정 분석은 flow 만 자동 생성 후 vocab/compare 는 사용자가 아코디언을 열 때 lazy 생성.
- 감정 분석은 P3o 4모듈 중 유일하게 **학습 방향(mode)** 개념을 갖는다. 이는 대상 학습자를 한국인 학생과 중국인 학생 양쪽에 맞추기 위한 UX 결정으로, 다른 3모듈의 `lang` 스위처와는 의미가 다르다(다른 3모듈은 결과 표기 언어만 바뀜). 단, **l2 카드에서는 학습 방향/언어 선택을 노출하지 않는다**. 4장 모두 `AiGenCard` 클릭 → 한국어(`ko`) 기준 자동 생성 → l3 진입 후 상세 페이지 내부에서만 스위칭한다.

### 4.2 Lovable Cloud 후경

- 캐시 테이블: `song_feature_cache(song_id, feature_type, content)`, unique key `(song_id, feature_type)`.
- `feature_type` 매핑:
  - `culture_compare` → `{[lang]: {topics, details}}`
  - `metaphor_symbol` → `{expressions, ko:{details}, zh:{details}}`
  - `era_language` → `{ko?: EraLanguageData, zh?: EraLanguageData}`
  - `emotion_analysis` → `{ko?: EmotionAnalysisLangData, zh?: EmotionAnalysisLangData}`
- helper 계약:
  - `loadCache<T>(songId, feature_type) → Promise<T | null>`
  - `saveCache(songId, feature_type, content) → Promise<void>`
- 참조 메모: `mem://features/explore-culture-comparison-logic`, `mem://features/explore-metaphor-analysis-logic`, `mem://features/explore-era-linguistic-logic`, `mem://features/explore-generation-workflow`, `mem://features/explore-content-modules`.

### 4.3 데이터 계약 (TypeScript)

```ts
export interface CultureTopicItem { topic_name: string; related_lyrics: string[]; }
export interface CultureTopicsData { topics: CultureTopicItem[]; }
export interface CultureDetailData {
  chinese: { expression: string; culture: string; modern: string };
  korean:  { expression: string; culture: string; song_expression: string };
  comparison: {
    emotion_expression: { chinese: string; korean: string };
    grammar:            { chinese: string; korean: string };
    cultural_background:{ chinese: string; korean: string };
    intensity:          { chinese: string; korean: string };
  };
  learning_point: string;
}

export interface MetaphorExpressionItem { expression_name: string; lyrics_ko: string; lyrics_zh: string; }
export interface MetaphorExpressionsData { expressions: MetaphorExpressionItem[]; }
export interface MetaphorDetailData {
  surface_meaning: string;
  real_meaning: string;
  techniques: { name: string; description: string }[];
  korean_nuance: string;
  chinese_nuance: string;
  learning_point: string;
}

export interface EraExpressionItem { expression: string; description: string; }
export interface EraLyricExpression { expression_name: string; lyrics: string; then: string; now: string; description: string; }
export interface EraLanguageData {
  verified_year: string;
  has_lyric_features: boolean;
  korean:  { era_expressions: EraExpressionItem[]; era_description: string };
  chinese: { era_expressions: EraExpressionItem[]; era_description: string };
  expressions: EraLyricExpression[];
  learning_point: string;
}

export interface EmotionFlowItem { stage: string; emotion: string; lyric: string; lyric_translated?: string; desc: string; }
export interface EmotionFlowData { flow: EmotionFlowItem[]; }
export interface EmotionVocabItem { word: string; pinyin?: string; meaning: string; hsk?: string; attr: "pos"|"neg"|"neu"; attr_label: string; }
export interface EmotionVocabData { vocab: EmotionVocabItem[]; }
export interface EmotionCompareItem { topic: string; emoji?: string; ko_expressions: string[]; zh_expressions: string[]; difference: string; }
export interface EmotionCompareData { compare: EmotionCompareItem[]; }
export interface EmotionAnalysisLangData { flow?: EmotionFlowData; vocab?: EmotionVocabData; compare?: EmotionCompareData; }
```

## ⑤ Acceptance & Output

### 5.1 Acceptance Criteria

- Culture/Metaphor 페이지는 **부모가 준 draftData 의 topics/expressions 배열만으로도** 언어 스위칭 없이 첫 카드가 자동 open 되어야 하며, 그 상세만 지연 로드된다.
- 언어(문화·비유) 스위칭 후 이미 생성된 언어 상세 캐시가 있으면 재요청 없이 즉시 표시.
- 시대 언어 페이지는 `!currentData` 이면 반드시 자동 `generateEraLanguage(lang)` 을 트리거하고, 첫 도착 전까지 전용 스켈레톤을 노출한다.
- 감정 분석 mode 매핑이 정확해야 한다. 스크린 카피는 `T[outputLang]` 로 결정되며, mode==="ko" 일 때 절대 한국어 카피가 화면에 노출되어서는 안 된다(반대도 동일).
- 감정 어휘 fallback 은 `word_list.length > 0` 일 때만 동작하고, pinyin 컬럼은 vocab 안에 pinyin 이 하나라도 있을 때만 렌더된다.
- 재생성 → 저장 시나리오에서 다른 언어/모드의 캐시가 유실되지 않는다(반드시 `{...existing}` 로 병합).
- Recharts 등 그래프 라이브러리 import 가 어디에도 존재하지 않아야 한다.
- 각 페이지 상단 뒤로가기 클릭 시 부모 `onBack()` 만 호출(라우팅 조작 없음).

### 5.2 Output Format

반환 순서:
1. `supabase/functions/generate-artist-story/index.ts` (5개 action 라우팅 부분)
2. `supabase/functions/generate-emotion-analysis/index.ts` (3개 action)
3. `src/components/songs/artistStoryHelpers.ts` (타입 + helper + 6 generator + isValid*)
4. `src/components/songs/CultureComparePage.tsx`
5. `src/components/songs/MetaphorSymbolPage.tsx`
6. `src/components/songs/EraLanguagePage.tsx`
7. `src/components/songs/EmotionAnalysisPage.tsx`
8. 한국어 3줄 요약.
