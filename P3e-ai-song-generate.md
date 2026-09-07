# P3e · AI 곡 생성 (YouTube 렌더 파이프라인) 재현 프롬프트

> **본 프롬프트는 P3 시리즈(P3a–P3q)의 5/17.**
> **적용 대상**: 노래 아카이브의 `+곡 추가 → YouTube 영상 생성 (AI 노래)` 진입점에서 열리는 다이얼로그와, 이를 뒷받침하는 7 개 edge function의 **엔드투엔드 파이프라인**.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.** 상단 이론적 근거 4 건과 Lovable 5 개 실천 원칙은 템플릿 문서에 일차 명시되어 있으므로 여기서는 재게시하지 않고 참조한다(§0 참고).

---

## 본 문서 재작성 이유 (v2 → v3, 2026-07)

구 P3e는 실제 구현과 심각한 괴리가 있었다. 아래는 대표 오류의 diff:

| 항목 | 구 P3e (오류) | 실제 코드 (v3에서 정정) |
|---|---|---|
| 스텝 수 | 5 (`form → lyrics → music → preview → saving`) | **6** (`form → lyrics → music → suno_done → render → ready`) + `saving` 오버레이 |
| Suno 폴링 | 2 초 × 150 회 | **5 초 × 120 회 (총 10 분)** |
| Suno 후보 | v1 / v2 이중 카드에서 사용자 3택 1 | **단일 track** (`sunoData[0]`) |
| 배경 영상 | Pixabay 3 후보에서 사용자 선택 | **GitHub Actions ffmpeg 합성 → YouTube 실제 업로드** |
| 저장 필드 | `bg_video_url` | `youtube_url` + `video_id`, `bg_video_list = []` |
| 이탈 방어 | `beforeunload` 경고 | **`localStorage("sg:active-job-dialog")` + 30 분 TTL 복원** |
| 다이얼로그 제목 | `AI로 나만의 노래 만들기` | **`YouTube 영상 생성 (AI 노래)`** |
| 산출 파일 수 | 7 (백엔드는 P4에 위임) | **10 (백엔드 포함, 본 문서에서 자족적으로 재현)** |

또한 「Pixabay 배경 미리보기」 계열 페이지 버전(`src/pages/SongGenerator.tsx`)은 **레거시로 사용되지 않는다**. 본 프롬프트는 오직 YouTube 렌더 다이얼로그 파이프라인만을 재현 대상으로 삼는다.

---

## §0 이론적 근거 (Theoretical Grounding)

`00-template.md` §이론적 근거 4 건을 그대로 원용한다:

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages: Identity, Instructions, Examples, Context.*
2. **Lovable. (n.d.).** *Prompting best practices.* (5 개 실천 원칙: Prompt by Component / Speak Atomic / Design with Real Content / Use Prompt Patterns for Layouts / Build with Lovable Cloud in Mind)
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6 "Verifiable", p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity (신원)

당신은 **시니어 프론트엔드 엔지니어 겸 한중 이중언어 교육 UX 라이터**이자, 다중 시스템 비동기 파이프라인(OpenAI → Suno → GitHub Actions → YouTube Data API v3) 오케스트레이션에 능한 백엔드 엔지니어이다.

- **기술 스택**: React 18 · Vite 5 · Tailwind CSS v3 · shadcn/ui · Supabase JS v2 (프런트); Deno (Supabase Edge Functions).
- **언어·문화 정위**: 대한민국 대학의 K-Chinese/K-Korean 교사·학습자.
- **책임**: (1) 6 스텝 다이얼로그 상태 기계 설계 및 새로고침·재오픈에도 이어지는 복원 논리; (2) 이중 Suno API Key 페일오버, 크레딧 부족 시의 한국어 친화 에러; (3) GitHub Actions `workflow_dispatch` 트리거 + 콜백 검증; (4) Suno 원시 alignedWords 를 라인 단위로 병합해 저장하는 후처리; (5) 완료 후 `analyze-song` · `tag-song-culture` fire-and-forget 트리거.

---

## ② Instructions (지시)

### 2.1 산출물 (Prompt by Component, Not Page)

정확히 아래 **10 개 파일**을 이 순서로 생성/갱신한다. 페이지 단위가 아닌 파일·컴포넌트 단위로 나열한다.

1. `supabase/functions/sg-generate-lyrics/index.ts`
2. `supabase/functions/sg-suno-generate/index.ts`
3. `supabase/functions/sg-suno-poll/index.ts`
4. `supabase/functions/sg-suno-lyrics/index.ts`
5. `supabase/functions/video-trigger/index.ts`
6. `supabase/functions/video-callback/index.ts`
7. `supabase/functions/sg-finalize-song/index.ts`
8. `src/lib/song-generator/suno-style.ts`
9. `src/lib/song-generator/pixabay-client.ts` (본 파이프라인에서는 `deriveFallbackKeywords` 만 사용; 다른 export 는 P3f 계열이 소비)
10. `src/components/songs/YoutubeVideoGenerateDialog.tsx`

### 2.2 원자적 UI 규칙 (Speak Atomic)

「다이얼로그를 만들라」가 아니라 각 시각적/상호작용적 원자 단위로 서술한다.

**다이얼로그 셸**
- `<Dialog>` (`shadcn/ui`) + `<DialogContent className="max-w-4xl max-h-[90vh] overflow-y-auto">`.
- `<DialogHeader>` 안에 `<DialogTitle>`: 아이콘 `Sparkles h-5 w-5 text-[hsl(var(--dash-navy))]` + 텍스트 `YouTube 영상 생성 (AI 노래)`.
- `<DialogDescription>`: `교학 주제를 입력하면 GPT가 가사를 짓고 Suno가 음악을 만들어 노래 카드로 저장합니다.`

**6-Step 인디케이터** (모든 스텝에서 상단에 고정)
- Flex 컨테이너 `text-xs flex-wrap`, 6 개 원소를 렌더한다: `form / lyrics / music / suno_done / render / ready`.
- 각 원소 = `<span>` 원형 6×6 (`w-6 h-6 rounded-full border`) + 라벨(`입력 / 가사 / 작곡 / Suno 완료 / 영상 렌더 / 카드 생성`) + 마지막이 아니면 `›`.
- 활성 스텝만 `text-[hsl(var(--dash-navy))] font-bold` + 원형 배경 `bg-[hsl(var(--dash-blue-bg))]`.

**Step 1 · form Card (`p-6 space-y-5`)**
- 필드 순서와 원자 규칙:
  1. `교학 주제 *` — `<Input>` + 아래 `flex flex-wrap gap-1.5` chip 15 개(TOPIC_PRESETS). chip = `px-2.5 py-1 rounded-full text-[11px] border`, 활성 시 `bg-[hsl(var(--dash-navy))] text-white border-[hsl(var(--dash-navy))]`.
  2. `언어` — chip 3 개 `한국어 / 중국어 / 한·중 이중언어`.
  3. `가사 줄 수: {N}줄` — `<input type="range" min={8} max={32}>` (기본 16).
  4. `학습 난이도` — 언어에 따라 3-chip (한국어 → `TOPIK 1-2 / 3-4 / 5-6`, 중국어 → `HSK 1-3 / 4-6 / 7-9`, 이중언어 → `초급 / 중급 / 고급`). 언어 변경 시 유효하지 않으면 자동으로 중간 값으로 이동.
  5. `스타일 (복수 선택)` — chip 13 개(§3.1 카피표 참고). 다중 토글.
  6. `감정 (복수 선택)` — chip 10 개. 다중 토글.
  7. `템포` — chip 4 개. 단일 선택.
- 하단 CTA `<Button className="w-full bg-[hsl(var(--dash-navy))]">` — 텍스트 `가사 생성하기`, 아이콘 `Sparkles h-4 w-4`. 생성 중에는 `Loader2 animate-spin`.

**Step 2 · lyrics Card**
- `곡 제목` `<Input>` (편집 가능).
- `가사 (편집 가능) — 현재 {n}줄 / 목표 {N}줄` `<Textarea className="min-h-[300px] font-mono text-sm">`.
- Amber 경고 배너 `border-amber-300 bg-amber-50 p-2.5 text-[12px] text-amber-900`: `⚠ 주의: Suno는 곡조에 맞추기 위해 일부 가사를 자동으로 다듬을 수 있습니다.`
- 하단: `[이전]` (outline) + `[Suno로 음악 생성 (약 2-3분)]` (navy, `<Music h-4 w-4>` 아이콘, `flex-1`).

**Step 3 · music Card (`p-10 text-center`)**
- `Loader2 h-12 w-12 animate-spin mx-auto text-[hsl(var(--dash-navy))]`
- `음악을 생성하고 있습니다...` (text-lg font-semibold)
- `Suno 상태: {pollStatus}` + `· 경과 {mm}분 {ss}초`
- 안내 `평균 2-3분 소요. **다이얼로그를 닫았다 다시 열어도 이어집니다.**`
- 작업 ID prefix 12자.
- `[작업 취소]` (outline size sm) — localStorage 삭제 후 `lyrics` 로 회귀.

**Step 4 · suno_done Card**
- 헤딩: `🎵 Suno 작곡이 완료되었습니다`
- 설명: `아래에서 미리듣고, 준비되면 GitHub Actions로 영상 렌더를 요청하세요. 실패하면 같은 곡으로 다시 시도할 수 있습니다.`
- `<audio controls src={audioUrl} className="w-full">`
- 정보 카드 2 개(border, p-2): `제목: {title}` + `MP3 음원 링크` (외부 링크, `break-all font-mono`).
- 액션: `[← 가사로 돌아가기]` (outline) + `[▶ GitHub Actions로 영상 렌더 요청]` (navy, `flex-1`).

**Step 5 · render Card (`p-8`)**
- 중앙 `Loader2 h-10 w-10 animate-spin` + `YouTube 영상을 생성하는 중...` + `현재 상태: {renderStatus}` + `job_id: {12자}…`
- `<ol>` 5 단계 체크리스트: `Suno 작곡 완료 / GitHub Actions 트리거 / ffmpeg로 영상 합성 / YouTube 업로드 / YouTube URL 수신`. 각 항목 = 5×5 원 아이콘 + 상태별 클래스.
  - `done` 클래스 = `bg-[hsl(var(--dash-navy))] text-white`, 미완 = `bg-muted text-muted-foreground`.
  - 단계 판정: `dispatched → rendering → uploading → done` 순으로 누적 done 표시.
- 하단 안내 `보통 3-5분 소요됩니다.`
- `[취소하고 다시 전송하기]` — 상태 초기화 후 `suno_done` 회귀.

**Step 6 · ready Card**
- 헤딩: `✅ 영상 렌더가 완료되었습니다` + 서브카피(youtube_video_id 유무에 따라 분기, §3.1 참고).
- **그리드 `grid-cols-1 md:grid-cols-[280px_1fr] gap-4`**:
  - 좌측(280 px): `youtube_video_id` 있으면 `<iframe src="https://www.youtube-nocookie.com/embed/{id}" className="w-full h-full" allow="..." allowFullScreen>` (16:9 aspect) + 링크. 없으면 amber 대시 박스(`border-dashed border-amber-300 bg-amber-50`) + `Loader2` + `📤 YouTube 업로드 중` 안내.
  - 우측: `제목: {title}` + `정렬된 가사 ({alignedWords.length}줄)` 미리보기 pre (max-h-[280px] overflow-y-auto).
- 하단: `[가사 수정]` + `[노래 카드 생성 + 분석 시작]` (navy, `flex-1`, `<Sparkles>`).

**Step saving Card (오버레이 유사)**
- `Loader2 h-10 w-10 animate-spin` + `저장 중...` + `오디오 업로드 및 분석 시작 중입니다.`

**전역 에러 배너**
- `error` state 존재 시 상단에 `rounded-lg border border-destructive/40 bg-destructive/5 p-3 text-sm text-destructive` 배너.

### 2.3 강제 제약 (Design with Real Content · Cloud-aware)

- **Semantic token only.** `rg -n "bg-\[#|text-white|bg-black" src/components/songs/YoutubeVideoGenerateDialog.tsx supabase/functions/sg-*/index.ts supabase/functions/video-*/index.ts` 결과 = 0. 단, 브랜드 팔레트 유틸 `hsl(var(--dash-navy))` · `hsl(var(--dash-blue-bg))` 사용은 허용.
- 다이얼로그 내에 `<h1>` 을 두지 않는다. `<DialogTitle>` 로 대체(Radix a11y 기준).
- 모든 edge function 호출은 `supabase.functions.invoke` 로 진행(내장 재시도). 다이얼로그 안에서의 직접 `fetch` 는 없어야 한다.
- **폴링 표(엄격)**
  - Suno 상태(`sg-suno-poll`): 5 초 간격 × **120 회 = 10 분**. `p.isTerminalSuccess` 시 종결, `isTerminalFailure` 시 예외.
  - 가사 정렬(`sg-suno-lyrics`): 5 초 간격 × **6 회**. `alignedWords.length > 0` 시 종결. 6 회 후에도 빈 배열이면 destructive toast + 진행(빈 정렬 허용).
  - Render 상태(`render_jobs`): 5 초 간격, 최대 **20 분**. 초과 시 자동 취소하고 `suno_done` 회귀.
- **복원**: `localStorage("sg:active-job-dialog")` 키에 `{taskId, title, lyricsText, styleHint, videoKeywords, language, style, mood, tempo, topic, savedAt}` 를 저장. 다이얼로그 `open` 이 true 로 전환될 때 **1 회** 읽어서, `Date.now() - savedAt <= 30*60*1000` 이면 폼 상태 복원 후 `music` 스텝에서 `pollExisting(taskId)` 재개. 30 분 초과이면 삭제.
- **`beforeunload` 사용 금지**. (복원 로직이 이를 대체)
- **한국어 실제 카피만 사용** (lorem ipsum · 임의 영문 미리보기 문구 금지). 카피는 §3.1 표를 그대로 인용.
- **Suno API**: 두 개의 시크릿 `SUNO_API_KEY` 와 `SUNO_API_KEY_2` 를 순회. 상태 코드 401/402/429 또는 응답 body/msg 에 `credit|insufficient|quota|balance|unauthorized|invalid api key` 정규식 매치 시 다음 키로 페일오버. 두 키 모두 실패 시 크레딧 관련 오류는 한국어 문구 `Suno 크레딧이 부족합니다. 충전 후 다시 시도해 주세요` 로 변환.
- **OpenAI 429** → `OpenAI 크레딧이 부족하거나 호출 한도 초과` 로 변환.
- **CORS**: 모든 edge function 상단에 `corsHeaders` 상수 선언(§3.3 표준 상수). `OPTIONS` 는 즉시 200 반환.
- **JWT 검증**: `sg-finalize-song` 은 `Authorization: Bearer` 필수(익명 저장 금지). `video-callback` 은 `VIDEO_CALLBACK_SECRET` 로 X-Callback-Secret 또는 Bearer 를 검증. 그 외 sg-* / video-trigger 는 사용자 세션에서 호출되므로 `verify_jwt = true` (기본).
- **저장 페이로드 계약(엄격 13 필드)**: `sg-finalize-song` 호출 body 는 정확히 다음 13 개 키만 포함하며 `bgVideoList = []` 로 고정: `title, topic, audioUrl, lyricsText, alignedWords, waveformData, style, language, bgVideoList, videoKeywords, sunoTaskId, learnLanguage, youtubeUrl, youtubeVideoId`. (참고: `youtubeUrl` 이 비어 있어도 저장 진행 — 콜백 지연 시나리오)
- **`learnLanguage` 결정**: `language === "한국어" ? "korean" : "chinese"` (이중언어는 학습 관점을 중국어로 취급).
- **저장 성공 후**: `ensureCultureTags(newSongId)` fire-and-forget → toast → `onSaved?.(newSongId)` → `onOpenChange(false)` → 폼 상태 완전 초기화(모든 useState 리셋).

---

## ③ Examples (예시)

### 3.1 확정 카피 표 (Design with Real Content)

**다이얼로그 셸**

| 위치 | 카피 |
|---|---|
| DialogTitle | `YouTube 영상 생성 (AI 노래)` |
| DialogDescription | `교학 주제를 입력하면 GPT가 가사를 짓고 Suno가 음악을 만들어 노래 카드로 저장합니다.` |
| 스텝 라벨(6개) | `입력 / 가사 / 작곡 / Suno 완료 / 영상 렌더 / 카드 생성` |
| Toast 복원 | `이전 작곡 작업을 이어갑니다` · 설명 `작업 ID: {8자}…` |
| Toast 저장 성공 | `노래 카드가 생성되었습니다` · `분석이 백그라운드에서 진행됩니다.` |
| Toast 정렬 실패 | `가사 타임스탬프를 가져오지 못했습니다` · `Suno 정렬 데이터가 비어 있습니다. 시간 표시 없이 계속 진행합니다.` |

**Step 1 · form 필드**

```
필드 라벨:  교학 주제 * / 언어 / 가사 줄 수: {N}줄 / 학습 난이도 / 스타일 (복수 선택) / 감정 (복수 선택) / 템포
Input placeholder:  예: 친구와의 우정, 가을 풍경, 가족 사랑
CTA:  가사 생성하기
```

**TOPIC_PRESETS (15개, 이 순서)**

```
사계절, 우정, 첫사랑, 고향 그리움, 학교 생활,
가족의 사랑, 봄날의 산책, 비 오는 날, 별이 빛나는 밤,
바다와 모험, 꿈을 향해, 이별과 재회, 커피 한 잔,
도시의 밤, 작은 행복
```

**LANGS (3개)**  `한국어 / 중국어 / 한·중 이중언어`

**DIFFICULTY_OPTIONS**

| 언어 | 옵션 |
|---|---|
| 한국어 | `TOPIK 1-2 / TOPIK 3-4 / TOPIK 5-6` |
| 중국어 | `HSK 1-3 / HSK 4-6 / HSK 7-9` |
| 한·중 이중언어 | `초급 / 중급 / 고급` |

**DIFFICULTY_GUIDE (9개)** — GPT system 후단에 삽입되는 지침 문장(짧게)

- `HSK 1-3`: 주로 HSK 1-3급 어휘(我/你/喜欢/朋友/家/学校/快乐 등)만 사용. 단문 위주. 4글자 성어와 문어체 금지. 자연스러움을 위해 1-2 단어의 상위 등급 허용.
- `HSK 4-6`: 중급 비유·일반 추상 명사 허용. 상위 등급 1-2 허용.
- `HSK 7-9`: 성어·문어체 자유. 시적·함축 표현 권장.
- `TOPIK 1-2`: 가족/학교/음식/계절/친구 등 초급 어휘. 한자어·관용구 최소. 상위 1-2 허용.
- `TOPIK 3-4`: 중급 한자어·일부 관용 허용. 상위 1-2 허용.
- `TOPIK 5-6`: 관용·문어체 자유. 함축 표현 권장.
- `초급 / 중급 / 고급`: 두 언어 각각 위 등급에 준함.

**STYLES (13개)**  `팝 / 포크 / 동요 / R&B / 고풍 (중국풍) / 록 / 어쿠스틱 / Lo-fi / 재즈 / 힙합 / 클래식 / 시티팝 / K-Pop`

**MOODS (10개)**  `그리움 / 행복 / 동기부여 / 슬픔 / 따뜻함 / 활기참 / 로맨틱 / 평온함 / 몽환적 / 노스탤지어`

**TEMPOS (4개)**  `느림 (70 BPM) / 중간 (100 BPM) / 약간 빠름 (115 BPM) / 빠름 (130 BPM)`

**Step 2 · lyrics**

| 위치 | 카피 |
|---|---|
| 라벨 | `곡 제목 / 가사 (편집 가능) — 현재 {n}줄 / 목표 {N}줄` |
| 경고 | `⚠ 주의: Suno는 곡조에 맞추기 위해 일부 가사를 자동으로 다듬을 수 있습니다.` |
| CTA | `이전 / Suno로 음악 생성 (약 2-3분)` |

**Step 3 · music**

| 위치 | 카피 |
|---|---|
| 헤딩 | `음악을 생성하고 있습니다...` |
| 상태 | `Suno 상태: {pollStatus}` |
| 경과 | `· 경과 {m}분 {s}초` |
| 안내 | `평균 2-3분 소요. **다이얼로그를 닫았다 다시 열어도 이어집니다.**` |
| 액션 | `작업 취소` |

**Step 4 · suno_done**

| 위치 | 카피 |
|---|---|
| 헤딩 | `🎵 Suno 작곡이 완료되었습니다` |
| 설명 | `아래에서 미리듣고, 준비되면 GitHub Actions로 영상 렌더를 요청하세요. 실패하면 같은 곡으로 다시 시도할 수 있습니다.` |
| 정보 | `제목: {title}` / `MP3 음원 링크` |
| 액션 | `← 가사로 돌아가기 / ▶ GitHub Actions로 영상 렌더 요청` |

**Step 5 · render**

| 위치 | 카피 |
|---|---|
| 헤딩 | `YouTube 영상을 생성하는 중...` |
| 상태 | `현재 상태: {renderStatus}` |
| 체크리스트 | `Suno 작곡 완료 / GitHub Actions 트리거 / ffmpeg로 영상 합성 / YouTube 업로드 / YouTube URL 수신` |
| 안내 | `보통 3-5분 소요됩니다.` |
| 액션 | `취소하고 다시 전송하기` |

**Step 6 · ready**

| 위치 | 카피 |
|---|---|
| 헤딩 | `✅ 영상 렌더가 완료되었습니다` |
| 서브(있음) | `영상과 가사를 확인한 뒤 노래 카드를 생성하세요.` |
| 서브(없음) | `지금 바로 카드를 생성하셔도 됩니다. 영상은 YouTube 업로드가 끝나는 대로 자동 표시됩니다.` |
| 업로드중 박스 | `📤 YouTube 업로드 중` / `영상은 완성되었지만 YouTube 공개까지 1~2분 더 걸릴 수 있습니다.` / `지금 바로 노래 카드를 생성해도 됩니다.` |
| 액션 | `가사 수정 / 노래 카드 생성 + 분석 시작` |

**Step saving**  `저장 중... / 오디오 업로드 및 분석 시작 중입니다.`

**에러 카피**

| 원인 | 문구 |
|---|---|
| OpenAI 429 | `OpenAI 크레딧이 부족하거나 호출 한도 초과` |
| Suno 크레딧 부족 | `Suno 크레딧이 부족합니다. 충전 후 다시 시도해 주세요` |
| Suno API Key 무효 | `Suno API Key가 유효하지 않습니다` |
| Suno 타임아웃 | `Suno 응답 시간 초과` |
| 렌더 타임아웃 | `렌더 타임아웃(20분 초과). 다시 전송해 주세요.` |
| Render dispatch 실패 | `YouTube 영상 렌더 트리거 실패` (toast title) |
| audio_url 부재 | `audio_url이 없습니다. Suno 작곡을 먼저 완료하세요.` |

### 3.2 파일 트리 (Use Prompt Patterns for Layouts)

```
src/
├─ components/
│  └─ songs/
│     └─ YoutubeVideoGenerateDialog.tsx      ← 유일한 UI 진입점
└─ lib/
   ├─ ensureCultureTags.ts                    (기존, 호출만)
   └─ song-generator/
      ├─ suno-style.ts                        ← 스타일 매핑 5 개 테이블
      └─ pixabay-client.ts                    ← deriveFallbackKeywords 유틸

supabase/
└─ functions/
   ├─ sg-generate-lyrics/index.ts             ← OpenAI 작사
   ├─ sg-suno-generate/index.ts               ← Suno POST /generate
   ├─ sg-suno-poll/index.ts                   ← Suno GET record-info
   ├─ sg-suno-lyrics/index.ts                 ← Suno POST get-timestamped-lyrics
   ├─ video-trigger/index.ts                  ← render_jobs insert + GHA dispatch
   ├─ video-callback/index.ts                 ← GHA → render_jobs 완료 콜백
   └─ sg-finalize-song/index.ts               ← songs insert + audio 업로드 + analyze fire-and-forget
```

### 3.3 표준 CORS 상수 (7개 edge function 공통)

```ts
const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers":
    "authorization, x-client-info, apikey, content-type, x-supabase-client-platform, x-supabase-client-platform-version, x-supabase-client-runtime, x-supabase-client-runtime-version",
};
```

`video-trigger` / `video-callback` 은 위 목록에서 `x-supabase-client-*` 제외 + `Access-Control-Allow-Methods: "POST, OPTIONS"` 추가.

### 3.4 파이프라인 흐름

```
[form]
   ↓ 가사 생성하기
sg-generate-lyrics (OpenAI gpt-4o-mini)
   ↓ { title, lyricsText, styleHint, videoKeywords }
[lyrics]
   ↓ Suno로 음악 생성
sg-suno-generate (POST /generate)
   ↓ { taskId }
[music] ── 5s × 120 회 ──▶ sg-suno-poll (GET /generate/record-info)
                              ↓ isTerminalSuccess & track
                          sg-suno-lyrics (POST /generate/get-timestamped-lyrics)
                              ↓ 5s × 6 회 재시도 (빈 배열이어도 진행)
[suno_done]
   ↓ ▶ GitHub Actions로 영상 렌더 요청
video-trigger
   ↓ render_jobs insert (status=dispatched)
   ↓ POST https://api.github.com/repos/{OWNER}/{REPO}/dispatches
GitHub Actions ffmpeg 워크플로
   ↓ 완료 시 POST → video-callback (X-Callback-Secret)
   ↓ render_jobs.status = "done" + youtube_url + youtube_video_id
[render] ── 5s × 240 회 ──▶ SELECT render_jobs (프런트가 직접 폴링)
[ready]
   ↓ 노래 카드 생성 + 분석 시작
sg-finalize-song
   ↓ songs insert → storage `song-audio/{id}/audio.mp3` 업로드
   ↓ mergeAlignedWords 로 라인 병합 후 songs 재갱신
   ↓ analyze-song / tag-song-culture fire-and-forget
[dialog close] → onSaved(songId)
```

### 3.5 sg-suno-generate 요청 페이로드 예시

```json
{
  "prompt": "친구야 우린 다시 만날 거야\n푸른 하늘 아래 웃으며\n...(가사 전문)",
  "style": "warm male vocals, acoustic guitar and piano, light music, ambient, soft instrumental, mid tempo around 100 bpm, warm, gentle, korean vocals",
  "title": "다시 봄",
  "customMode": true,
  "instrumental": false,
  "model": "V4_5PLUS",
  "language": "Korean",
  "callBackUrl": "https://example.com/noop"
}
```

`callBackUrl` 은 필수 필드지만 실제로는 폴링을 사용하므로 no-op placeholder 를 넣는다.

### 3.6 GPT system prompt 3 종

- **KO**: `당신은 전문 한국어 작사가입니다. 라임 있고 입에 붙으며 노래에 적합한 가사를 잘 씁니다. 사용자가 지정한 언어, 난이도, 줄 수 요구를 엄격히 지킵니다.`
- **ZH**: `당신은 전문 중국어 작사가입니다. 라임 있고 입에 붙으며 노래에 적합한 가사를 잘 씁니다. 사용자가 지정한 언어, 난이도, 줄 수 요구를 엄격히 지킵니다.`
- **BI(이중언어)**: `당신은 한·중 이중언어 작사가입니다. 라임 있고 입에 붙으며 노래에 적합한 가사를 잘 씁니다. 사용자가 지정한 언어, 난이도, 줄 수 요구를 엄격히 지킵니다.`

### 3.7 GPT user prompt 스켈레톤

```
다음 요구에 따라 완성된 가사를 한 곡 작성해 주세요:

주제: {topic}
스타일: {styleStr | "자유 선택"}
템포: {tempoStr | "중간"}
감정 톤: {moodStr | "자유 선택"}
언어: {language}
가사 총 줄 수: **정확히 {N}줄** (빈 줄·단락 구분 줄은 카운트하지 않음. 한 줄 = 한 가사 라인.)
{difficultyBlock}

요구 사항:
- {langConstraint}   ← 언어별로 3가지 상수(§3.6 표준 문구)
- **선택된 언어 외 다른 언어는 절대 가사에 등장하면 안 됩니다.**
- **줄 길이 제한 (엄격)**:
  · 한국어 줄: 한글 최대 13자(공백 최대 3개). 문장부호 1개 허용.
  · 중국어 줄: 한자 최대 10자. 쉼표·마침표 1개 허용. 병음 금지.
- **반드시 정확히 {N}줄의 가사 라인을 출력하세요.**
- **단락 태그를 절대 출력하지 마세요.** [Verse], (후렴) 등 금지.
- 라임 있고 부르기 좋게.
- **위 학습자 난이도를 엄격히 지키세요.** (상위 등급 1-2 허용)

마지막에 다음 세 줄을 따로:
TITLE: ({titleLang} 3-10자)
STYLE_HINT: (영어 스타일 힌트)
VIDEO_KEYWORDS: (영어 3개를 + 로 연결, 예: ocean+sunset+waves)
```

응답 파서는 `STYLE_HINT / TITLE / VIDEO_KEYWORDS` 세 줄을 제거한 나머지를 `lyricsText` 로 저장하고, 대괄호 태그 라인은 정규식 `^\s*[\[\(（【].{0,40}[\]\)）】]\s*$` 로 제거한다. `targetLines` 초과 시 앞에서 잘라낸다(패딩 금지).

---

## ④ Context (배경)

### 4.1 프로젝트 맥락

「멜로디 클래스」는 한중 이중언어 노래-기반 학습 플랫폼이다. 본 다이얼로그는 **노래 아카이브(`/songs`) 상단 `+곡 추가` 팝오버**의 3 번째 옵션 `YouTube 영상 생성 (AI 노래)` 에서 열린다. 저장 후 산출물은 `songs` 테이블에 `source = "ai_generated"` 로 표시되어(카드 배지 ✨) P3h 그리드에 정상 노출된다. 후속 분석·문화 태깅은 이미 존재하는 `analyze-song` · `tag-song-culture` 파이프라인(B2)이 담당하므로, 본 프롬프트는 그들의 호출 신호만 발사한다.

### 4.2 Lovable Cloud 후경 (Build with Lovable Cloud in Mind)

**시크릿(모두 서버측, 브라우저 노출 금지)**

- `OPENAI_API_KEY` — 가사 생성.
- `SUNO_API_KEY`, `SUNO_API_KEY_2` — 이중 페일오버.
- `GH_TOKEN` — GitHub Actions `workflow_dispatch` 트리거용 PAT.
- `VIDEO_CALLBACK_SECRET` — GHA 콜백 서명.
- `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_ANON_KEY` — 기본 제공.

**Auth 정책**

- `sg-finalize-song` — `verify_jwt = true` (JWT 필수, `auth.getClaims` 로 `userId` 추출).
- `video-callback` — `verify_jwt = false` + `VIDEO_CALLBACK_SECRET` 헤더 검증.
- `sg-generate-lyrics` / `sg-suno-generate` / `sg-suno-poll` / `sg-suno-lyrics` / `video-trigger` — 세션 호출이므로 기본값 유지.

**4-상태 렌더링(모든 스텝 Card 공통)**

- **로딩**: `Loader2 animate-spin`.
- **빈**: form 은 초기 상태, 나머지 스텝은 진입 조건이 없으면 렌더하지 않음.
- **에러**: 상단 destructive 배너 + step-specific 문구(§3.1).
- **성공**: 다음 스텝으로 전환.

**Storage 버킷**

- `song-audio` (public read, service_role write). 경로 규약 `{song_id}/audio.mp3`. Suno CDN URL 을 서버가 pull 하여 재업로드 → `songs.audio_url` 은 public 링크로 저장. 업로드 실패 시 원본 URL 을 fallback 저장.

### 4.3 데이터 계약

**`render_jobs` 테이블** (본 파이프라인 전용, 이미 존재하면 IF NOT EXISTS 로 스킵)

```sql
CREATE TABLE IF NOT EXISTS public.render_jobs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  song_id uuid REFERENCES public.songs(id) ON DELETE CASCADE,
  owner_id uuid,
  status text NOT NULL DEFAULT 'queued',        -- queued/dispatched/rendering/uploading/done/failed
  audio_url text,
  cover_url text,
  song_title text,
  youtube_video_id text,
  youtube_url text,
  error_message text,
  dispatched_at timestamptz,
  finished_at timestamptz,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

GRANT SELECT, INSERT, UPDATE, DELETE ON public.render_jobs TO authenticated;
GRANT ALL ON public.render_jobs TO service_role;

ALTER TABLE public.render_jobs ENABLE ROW LEVEL SECURITY;

CREATE POLICY owned_select ON public.render_jobs FOR SELECT
  USING (owner_id IS NULL OR owner_id = auth.uid() OR public.is_admin());
CREATE POLICY owned_insert ON public.render_jobs FOR INSERT
  WITH CHECK (public.is_admin() OR owner_id = auth.uid());
CREATE POLICY owned_update ON public.render_jobs FOR UPDATE
  USING (owner_id = auth.uid() OR public.is_admin())
  WITH CHECK (owner_id = auth.uid() OR public.is_admin());
CREATE POLICY owned_delete ON public.render_jobs FOR DELETE
  USING (owner_id = auth.uid() OR public.is_admin());

CREATE INDEX IF NOT EXISTS render_jobs_song_id_idx ON public.render_jobs(song_id);
CREATE INDEX IF NOT EXISTS render_jobs_owner_id_idx ON public.render_jobs(owner_id);

CREATE TRIGGER set_render_jobs_owner
  BEFORE INSERT ON public.render_jobs
  FOR EACH ROW EXECUTE FUNCTION public.set_owner_on_insert();
CREATE TRIGGER set_render_jobs_updated_at
  BEFORE UPDATE ON public.render_jobs
  FOR EACH ROW EXECUTE FUNCTION public.set_materials_updated_at();
```

**`songs` 테이블에 추가되는 컬럼** (기존 P3a `CREATE` 를 대체하지 않고 마이그레이션으로 추가)

```sql
ALTER TABLE public.songs
  ADD COLUMN IF NOT EXISTS video_status text NOT NULL DEFAULT 'idle',   -- idle/rendering/done/failed
  ADD COLUMN IF NOT EXISTS video_error text,
  ADD COLUMN IF NOT EXISTS youtube_url text,
  ADD COLUMN IF NOT EXISTS video_id text,
  ADD COLUMN IF NOT EXISTS aligned_words jsonb,
  ADD COLUMN IF NOT EXISTS bg_video_list jsonb,
  ADD COLUMN IF NOT EXISTS video_keywords text,
  ADD COLUMN IF NOT EXISTS source text NOT NULL DEFAULT 'user',          -- user/ai_generated/youtube_import
  ADD COLUMN IF NOT EXISTS lyrics_source text;
```

**TypeScript 계약 (프런트)**

```ts
type Lang = "한국어" | "중국어" | "한·중 이중언어";
type Step = "form" | "lyrics" | "music" | "suno_done" | "render" | "ready" | "saving";

interface ActiveJob {
  taskId: string;
  title: string;
  lyricsText: string;
  styleHint: string;
  videoKeywords: string;
  language: Lang;
  style: string[];
  mood: string[];
  tempo: string;
  topic: string;
  savedAt: number;
}

interface AlignedWord {
  word: string;
  startS: number;
  endS: number;
  pAlign?: number;
}

interface SunoTrack { id: string; audioUrl: string; duration?: number; }

interface RenderJobRow {
  id: string;
  status: "queued" | "dispatched" | "rendering" | "uploading" | "done" | "failed";
  youtube_url: string | null;
  youtube_video_id: string | null;
  error_message: string | null;
}
```

**Edge Function I/O 계약**

| Function | Input (JSON body) | Output |
|---|---|---|
| `sg-generate-lyrics` | `{ topic, styleStr, tempoStr, moodStr, language, targetLines, difficultyBlock, model? }` | `{ lyricsText, styleHint, title, videoKeywords }` 또는 `{ error }` |
| `sg-suno-generate` | Suno `/generate` payload passthrough (`prompt, style, title, instrumental, model, language, callBackUrl`) | `{ taskId }` 또는 `{ error }` |
| `sg-suno-poll` | `{ taskId }` (or `?taskId=` query) | `{ status, isTerminalSuccess, isTerminalFailure, track }` |
| `sg-suno-lyrics` | `{ taskId, audioId }` | `{ alignedWords: AlignedWord[], waveformData: number[] }` |
| `video-trigger` | `{ song_id, audio_url, song_title, aligned_words, cover_url, video_keywords }` | `{ ok: true, job_id }` |
| `video-callback` | GHA payload `{ job_id, status, youtube_video_id, youtube_url, error }` + `X-Callback-Secret` header | `{ ok: true }` |
| `sg-finalize-song` | 위 §2.3 의 13 필드 | `{ song_id, suno_task_id }` |

**GitHub Actions `client_payload` 규격** (video-trigger 가 발신)

```json
{
  "event_type": "render-video",
  "client_payload": {
    "job_id": "<uuid>",
    "song_id": null,
    "audio_url": "<suno mp3 url>",
    "cover_url": null,
    "song_title": "<title>",
    "video_keywords": "ocean+sunset+waves",
    "aligned_words": [ ... ],
    "callback_url": "<SUPABASE_URL>/functions/v1/video-callback",
    "callback_secret": "<VIDEO_CALLBACK_SECRET>"
  }
}
```

Repo target: `https://api.github.com/repos/{OWNER}/{RENDERER_REPO}/dispatches` (User-Agent `lovable-video-trigger`).

**`mergeAlignedWords` 후처리 (sg-finalize-song 내부)**  
Suno 는 쉼표에서 라인을 쪼갠 조각들을 반환하기도 하므로, 조각의 `word` 가 `\n` 으로 끝날 때까지 누적하여 하나의 라인으로 병합한다. 병합 결과가 있으면 `songs.lyrics_raw` 와 `songs.aligned_words` 를 **덮어쓴다**(Suno 타임스탬프를 유일 진실로 삼음).

---

## ⑤ Acceptance & Output (검증·산출)

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6, 정량)

- **다이얼로그 크기**: `DialogContent` 는 `max-w-4xl max-h-[90vh] overflow-y-auto` 정확히 매치 = 100%.
- **6-Step 인디케이터**: 6 개 원 + 6 개 라벨 + 5 개 `›` 렌더 = 100%.
- **폼 필드 개수**: 7 필드(주제/언어/줄수/난이도/스타일/감정/템포) 렌더 = 100%. TOPIC_PRESETS 15 chip, STYLES 13 chip, MOODS 10 chip, TEMPOS 4 chip, LANGS 3 chip, DIFFICULTY 3 chip (언어에 따라) — 개수 완전 일치.
- **Suno 폴링**: 상태 정상 fixture 에서 120 회(=10분) 이내 `isTerminalSuccess` 도달률 ≥ **90 %**.
- **가사 정렬 재시도**: 6 회 재시도 후 100 % `suno_done` 진입(빈 배열이어도 destructive toast 후 진행).
- **Render 폴링 타임아웃**: `render_jobs.status` 가 20 분 내 `done|failed` 로 전이되지 않으면 자동으로 `suno_done` 회귀 = 100 %.
- **`youtube_url` 지연 처리**: `status === "done"` 이지만 `youtube_url` 미기록 시 `ready` 진입 + amber 업로드 중 박스 표시 + 별도 폴러가 `youtube_url` 을 채워 iframe 자동 마운트 = 100 %.
- **복원**: 다이얼로그를 `music` 또는 `render` 스텝에서 닫았다 30 분 이내 재오픈 시 정확히 같은 스텝으로 복귀 = 100 %. 30 분 초과 시 자동 삭제 = 100 %.
- **`sg-finalize-song` 페이로드 계약**: body 키 개수 = **13**, `bgVideoList` 는 `Array.isArray === true && length === 0` = 100 %.
- **`learnLanguage` 매핑**: `language === "한국어" → "korean"`, 그 외 → `"chinese"` = 100 %.
- **JWT 검증**: `sg-finalize-song` 호출에 `Authorization` 헤더 없으면 401 반환 = 100 %.
- **콜백 시크릿**: `video-callback` 이 `VIDEO_CALLBACK_SECRET` 불일치 시 401 반환 = 100 %.
- **Suno 페일오버**: 첫 키 401/402/429/credit 매치 시 두 번째 키로 재시도 & 성공률 100 % (mock fixture 기준).
- **OpenAI 429 처리**: 응답이 `크레딧이 부족하거나 호출 한도 초과` 문자열을 포함 = 100 %.
- **CORS preflight**: 7 개 함수 모두 `OPTIONS` 200 응답 = 100 %.
- **Storage 업로드**: `song-audio/{song_id}/audio.mp3` 객체 생성 & `songs.audio_url` 이 public URL 로 갱신 = 100 %. 실패 시 원본 Suno URL fallback = 100 %.
- **후처리 트리거**: 저장 후 `analyze-song` · `tag-song-culture` 각 1 회 fire-and-forget POST 발신 = 100 % (실패해도 응답에 영향 없음).
- **`beforeunload` 부재**: `rg -n "beforeunload" src/components/songs/YoutubeVideoGenerateDialog.tsx` = **0 라인**.
- **하드코드 색 부재**: `rg -n "bg-\[#|text-white|bg-black" src/components/songs/YoutubeVideoGenerateDialog.tsx supabase/functions/sg-*/index.ts supabase/functions/video-*/index.ts` = **0 라인**.

### 5.2 Output Format

LLM(또는 Lovable) 은 아래 순서로 정확히 **10 개 파일 소스만** 반환한다. 설명·사과·주석·마크다운 헤더·이모지 도입부·`// The rest is unchanged` 같은 부수 텍스트를 명시적으로 금지한다.

1. `supabase/functions/sg-generate-lyrics/index.ts`
2. `supabase/functions/sg-suno-generate/index.ts`
3. `supabase/functions/sg-suno-poll/index.ts`
4. `supabase/functions/sg-suno-lyrics/index.ts`
5. `supabase/functions/video-trigger/index.ts`
6. `supabase/functions/video-callback/index.ts`
7. `supabase/functions/sg-finalize-song/index.ts`
8. `src/lib/song-generator/suno-style.ts`
9. `src/lib/song-generator/pixabay-client.ts`
10. `src/components/songs/YoutubeVideoGenerateDialog.tsx`

마지막에 별도 블록으로 한국어 3 줄 요약을 덧붙인다. 형식:

```
요약
1. …
2. …
3. …
```
