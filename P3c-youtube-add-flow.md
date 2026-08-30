# P3c · YouTube 노래 추가 흐름 재현 프롬프트

> **본 프롬프트는 P3 시리즈(P3a–P3q)의 3/17.**
> **적용 대상**: `src/components/songs/YouTubeSearchDialog.tsx`, `src/components/songs/YouTubeConfirmDialog.tsx` — URL/검색 이중 진입, 미리보기, 언어·레벨 확정, `analyze-song` 트리거.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* https://platform.openai.com/docs/guides/prompt-engineering (Retrieved July 12, 2026)
2. **Lovable. (n.d.).** *Prompting best practices.* https://docs.lovable.dev/prompting/prompting-one (Retrieved July 12, 2026) — 5개 실천 원칙 채택 근거.
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity (신원)

당신은 YouTube Data API v3 · Supabase Edge Functions · React Query 를 결합해 "URL/검색 → 언어 확정 → AI 분석 예약" 세 단계 위저드를 만드는 시니어 프론트엔드 엔지니어입니다. shadcn `Dialog`, `Select`, `Input` 과 `fetchWithRetry` 를 사용하며, **기본 진입은 중국어·한국어 이중 언어 병렬 검색(`language="all"`)** 이고, 언어 특정 진입(`"chinese" | "korean"`)은 드롭다운 서브뷰에서만 허용합니다. `"all"` 모드는 `Promise.all([fetchLang("zh"), fetchLang("ko")])` 로 두 언어를 동시에 조회한 뒤 결과를 인터리브(교차) 병합하고, 각 카드에 CN/KR 언어 배지를 표시합니다.

## ② Instructions

### 2.1 산출물 (Lovable 실천 원칙 "Prompt by Component, Not Page")

- `src/components/songs/YouTubeSearchDialog.tsx` — 검색 모달(검색어 · 결과 그리드 · 상세 미리보기).
- `src/components/songs/YouTubeConfirmDialog.tsx` — 확정 모달(비디오 프리뷰 · 언어 · HSK/TOPIK 레벨 · 주제 · 교학 포인트 · 커스텀 가사 텍스트박스).
- `src/hooks/useYoutubeSearch.ts` — `youtube-search` edge function 호출 훅. `language: "chinese" | "korean" | "all"` 지원. `"all"` 인 경우 내부에서 `Promise.all` 로 두 언어를 병렬 조회하고, 결과를 라운드로빈으로 인터리브 병합해 단일 배열로 반환한다(각 아이템에 `detectedLanguage: "zh" | "ko"` 필드 부여). 429 백오프.
- 트리거 계약: `onSongAdded(SongAddedData)`, `onAnalysisStart(pendingId)`, `onAnalysisEnd(pendingId, ok)`.

### 2.2 원자적 UI 규칙 (Lovable 실천 원칙 "Speak Atomic")

**YouTubeSearchDialog** (`max-w-4xl h-[85vh]`):
- 헤더: 아이콘 `<Youtube>` + **language 에 따른 타이틀 분기** — `language === "all"` → `노래 검색`(중·한 이중 언어), `"chinese"` → `중국어 노래 검색`, `"korean"` → `한국어 노래 검색`. `"all"` 이 아닐 때만 language chip(`🇨🇳 중국어` | `🇰🇷 한국어`) 노출.
- 검색 폼 = `flex gap-2`: `<Input>` + [검색] navy 버튼. placeholder 는 language 별 분기 — `"all"`: `중·한 노래·아티스트 검색…`, `"chinese"`: `중국어 곡 · 가수 검색…`, `"korean"`: `한국어 곡 · 가수 검색…`. Enter 로 제출.
- 결과 그리드 = `grid grid-cols-2 md:grid-cols-3 gap-3 overflow-y-auto` — 카드당 hqdefault 썸네일 + 제목 2줄 clamp + 채널명 1줄 clamp + `조회수` · `게시일`. `"all"` 모드에서는 각 카드 좌상단에 언어 배지(`🇨🇳 CN` / `🇰🇷 KR`, `absolute top-2 left-2 bg-background/80 backdrop-blur text-[10px] px-1.5 py-0.5 rounded`) 표시. hover 시 `ring-2 ring-primary/40`, 클릭 시 우측 상세 프리뷰로 이동.
- 상세 프리뷰(우측 슬라이드 인 400px): 큰 썸네일 + 원본 제목 + 채널 + `[확정하기]`(→ `YouTubeConfirmDialog` 로 전이, 이 때 확정 다이얼로그의 초기 `language` 는 카드에서 감지된 `detectedLanguage` 로 프리필) + `[뒤로]`.
- 빈 상태: 🔍 아이콘 + language 별 분기 카피 — `"all"`: `중국어·한국어 검색 결과가 없습니다. 다른 키워드로 다시 시도해보세요.`, 언어 특정: `검색 결과가 없습니다. 다른 키워드로 다시 시도해보세요.`
- 로딩: 스켈레톤 6장. `"all"` 모드에서 한쪽 언어만 늦게 도착해도 UI 는 도착 순서대로 병합 렌더(부분 결과 우선).
- 에러(429): 카피 = `YouTube 요청 한도에 도달했습니다. 30초 후 다시 시도해주세요.`

**YouTubeConfirmDialog** (`max-w-2xl`):
- 상단 프리뷰: 16/9 iframe(`youtube.com/embed/{videoId}`) + 제목 편집 가능 `<Input>` + 아티스트 편집 가능 `<Input>`.
- 폼 4 필드(수직):
  1. **언어**(`Select`, `chinese`/`korean`, 초기값 = 다이얼로그 prop).
  2. **레벨**(`Select`, 언어에 따라 `HSK 1..6` 또는 `TOPIK 1..6`).
  3. **주제**(`Select`, 12 THEME_OPTIONS: 사랑/이별/우정/가족/청춘/추억/희망/자연/일상/사회/기타/미분류).
  4. **교학 포인트**(`Select`, 어휘/문법/문화/발음).
- **가사 소스 라디오**(3옵션): `자동(YouTube 자막·Supadata) / 붙여넣기 / 나중에 붙여넣기`. `붙여넣기` 선택 시 `<Textarea rows=8>` 표시.
- 하단: [취소] 텍스트 · [분석 시작] navy 버튼. 클릭 시:
  1. `songs` 테이블 upsert(초안 row, `analysis_status="pending"`).
  2. `onAnalysisStart(newId)` 호출 → 셸 pendingAnalyses 배열에 등록.
  3. `analyze-song` edge function 백그라운드 트리거(`fetchWithRetry`, `keepalive: true`).
  4. Dialog 닫힘 + 토스트 `"{title}" 분석을 시작했어요. 완료되면 목록이 갱신됩니다.`
  5. 완료 시 realtime 구독 또는 폴링으로 `onAnalysisEnd(id, ok)` 호출.

### 2.3 강제 제약

- semantic token. YouTube 브랜드색은 아이콘에만(`text-red-600` 예외 허용).
- 검색 결과 페이지네이션은 nextPageToken 기반, 최대 2 페이지(50건)까지 로드.
- 동일 `video_id` 중복 감지: 확정 전 `songs` 테이블에서 `video_id` 조회 → 존재 시 토스트 `이미 등록된 곡입니다. 목록에서 확인하세요.` 후 닫힘.
- 분석 트리거는 **비동기 fire-and-forget**: 결과를 기다리지 않고 Dialog 를 즉시 닫는다.

## ③ Examples

### 3.1 확정 카피 표 (Lovable 실천 원칙 "Design with Real Content")

| 위치 | 카피 |
|---|---|
| Search 헤더(all) | `노래 검색` |
| Search 헤더(단일) | `중국어 노래 검색` / `한국어 노래 검색` |
| 언어 chip(단일 모드에서만) | `🇨🇳 중국어` / `🇰🇷 한국어` |
| 카드 언어 배지(all 모드) | `🇨🇳 CN` / `🇰🇷 KR` |
| Search placeholder(all) | `중·한 노래·아티스트 검색…` |
| Search placeholder(단일) | `중국어 곡 · 가수 검색…` / `한국어 곡 · 가수 검색…` |
| Search 버튼 | `검색` |
| 상세 확정 | `확정하기` |
| Confirm 헤더 | `노래 정보 확정` |
| 필드 라벨 | `언어 / 레벨 / 주제 / 교학 포인트` |
| 가사 라디오 | `자동 (YouTube 자막)` / `직접 붙여넣기` / `나중에 추가` |
| 가사 placeholder | `가사를 붙여넣으세요…` |
| 하단 액션 | `취소` / `분석 시작` |
| 시작 토스트 | `"{title}" 분석을 시작했어요. 완료되면 목록이 갱신됩니다.` |
| 중복 토스트 | `이미 등록된 곡입니다. 목록에서 확인하세요.` |
| 429 에러 | `YouTube 요청 한도에 도달했습니다. 30초 후 다시 시도해주세요.` |
| 빈 상태(all) | `중국어·한국어 검색 결과가 없습니다. 다른 키워드로 다시 시도해보세요.` |
| 빈 상태(단일) | `검색 결과가 없습니다. 다른 키워드로 다시 시도해보세요.` |

### 3.2 흐름 다이어그램 (Lovable 실천 원칙 "Use Prompt Patterns for Layouts")

```text
[Topbar/Fab (P3b)]
      │  onOpenYoutubeSearch("chinese")
      ▼
<YouTubeSearchDialog>
   ├─ 검색폼 → useYoutubeSearch(q, lang)
   ├─ 결과 그리드 (썸네일 카드 × ≤50)
   └─ 상세 프리뷰 → 확정하기
                       │
                       ▼
                <YouTubeConfirmDialog>
                   ├─ iframe 프리뷰
                   ├─ 제목/아티스트/언어/레벨/주제/교학 편집
                   ├─ 가사 소스 라디오 3택
                   └─ [분석 시작]
                             │
                             ├─ songs upsert(status=pending)
                             ├─ onAnalysisStart(id)  → 셸 pending 배너
                             ├─ analyze-song (fire-and-forget)
                             └─ Dialog close + toast
                                          │
                                          ▼  (edge function 완료 후)
                                    onAnalysisEnd(id, ok)
                                    → useSongsQuery refetch
```

## ④ Context

### 4.1 프로젝트 맥락
`P3b` 의 검색 인풋 로컬 결과 하단 CTA `YouTube에서 검색`, 상단바/FAB 의 `YouTube에서 추가 >` 서브뷰, 그리고 셸의 URL 인풋(YouTube URL 자동 감지 시 `분석` 버튼) — 세 진입 경로 모두 이 두 다이얼로그로 수렴한다.

### 4.2 Lovable Cloud 후경 (Lovable 실천 원칙 "Build with Lovable Cloud in Mind")

- Edge Function: `youtube-search`(YouTube Data API v3 프록시), `analyze-song`(가사·핀인·단어리스트·문법·문화 태그를 순차 생성 → songs·song_analyses·song_culture_tags 저장, `analysis_status` 를 `pending → processing → done|failed` 로 전이).
- `songs.analysis_status` 컬럼은 P4a 에서 정의. 본 프롬프트는 클라이언트가 `pending` row 를 먼저 만들고 백그라운드에서 상태를 업데이트하는 계약만 참조.
- `fetchWithRetry(retries: 3, baseDelay: 500)` 로 429 자동 백오프.

### 4.3 데이터 계약

```ts
type SongAddedData = {
  id: string;
  video_id: string;
  title: string;
  artist: string;
  language: "chinese" | "korean";
  hsk_level: string;
  theme: string;
  teaching_point: string;
  analysis_status: "pending";
};

interface YouTubeSearchDialogProps {
  open: boolean;
  onOpenChange(v: boolean): void;
  language: "chinese" | "korean" | "all";
  initialQuery?: string;
  onSongAdded(s: SongAddedData): void;
  onAnalysisStart(pendingId: string): void;
  onAnalysisEnd(pendingId: string, ok: boolean): void;
}
```

## ⑤ Acceptance & Output

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6)

- 검색어 입력 후 [검색] 클릭 → 500 ms 내 스켈레톤 6장, 3 s 내 결과 6~50건 렌더(p95, YouTube API 정상 시).
- 결과 카드 클릭 → 우측 상세 슬라이드 인 애니메이션 200 ms 완료.
- 확정 후 다이얼로그 close 시각(t0) 부터 `songs` 목록에 pending row 반영까지 = **≤300 ms**(insert 응답).
- 동일 `video_id` 재추가 시도 시 insert 발생 없이 중복 토스트 노출.
- `analyze-song` 완료 이벤트 수신 후 셸 pending 배너에서 해당 곡 제거 + `useSongsQuery.refetch()` 자동 호출.
- `rg -n "bg-\[#|text-white|bg-black" src/components/songs/YouTubeSearchDialog.tsx src/components/songs/YouTubeConfirmDialog.tsx` = **0**.
- 카피 표(§3.1) 문자열 렌더 매칭률 = **100 %**.
- **이중 언어 병렬 검색**: `language="all"` 로 열었을 때 네트워크 탭에 `youtube-search` 호출이 정확히 2건(`lang=zh`, `lang=ko`) 병렬 발생하고, 두 응답을 라운드로빈 인터리브해 단일 배열로 렌더한다. 결과 카드 중 최소 1건 이상은 `🇨🇳 CN` 배지, 1건 이상은 `🇰🇷 KR` 배지를 표시한다.
- `language="all"` 카드에서 `[확정하기]` 클릭 시 `YouTubeConfirmDialog` 의 `language` 초기값이 `detectedLanguage` 로 프리필된다(사용자는 언제든 변경 가능).

### 5.2 Output Format
반환 순서:
1. `src/hooks/useYoutubeSearch.ts`
2. `src/components/songs/YouTubeSearchDialog.tsx`
3. `src/components/songs/YouTubeConfirmDialog.tsx`
4. 한국어 3줄 요약.
