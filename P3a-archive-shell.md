# P3a · 노래 아카이브 셸 (`/songs` 페이지 골격) 재현 프롬프트

> **본 프롬프트는 P3 시리즈(P3a–P3q)의 1/17 이며, 완전한 아카이브 재현은 P3a → P3q 를 순서대로 실행할 때 성립한다.**
> **적용 대상**: `src/pages/Songs.tsx` — 라우팅·상태 오케스트레이션·데이터 조회·URL 동기화·페이지네이션·빈/에러/로딩 상태 및 하위 컴포넌트(P3b~P3q) 조립.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages: Identity, Instructions, Examples, Context.* OpenAI Platform Documentation. Retrieved July 12, 2026, from https://platform.openai.com/docs/guides/prompt-engineering
2. **Lovable. (n.d.).** *Prompting best practices.* Lovable Documentation. Retrieved July 12, 2026, from https://docs.lovable.dev/prompting/prompting-one — 본 부록에서 인용된 5개 실천 원칙(Prompt by Component, Not Page · Speak Atomic · Design with Real Content · Use Prompt Patterns for Layouts · Build with Lovable Cloud in Mind) 채택 근거.
3. **IEEE. (1998).** *IEEE Recommended Practice for Software Requirements Specifications* (IEEE Std 830-1998), §4.3.6 "Verifiable", p. 7. IEEE.
4. **Cohn, M. (2004).** *User Stories Applied: For Agile Software Development*, Chapter 6, pp. 67–74. Addison-Wesley.

---

## ① Identity (신원)

당신은 대한민국 대학의 K-Chinese/K-Korean 노래 학습 콘텐츠 아카이브를 설계·구축하는 시니어 프론트엔드 엔지니어입니다. React 18 + Vite 5 + Tailwind CSS v3 + shadcn/ui + Supabase JS v2 + `fetchWithRetry(AbortController + Exponential Backoff)` 조합을 사용하며, 본 절에서는 페이지 파일 하나가 **상단바(P3b) · YouTube추가(P3c) · CSV일괄(P3d) · AI생성(P3e) · 히어로/선반(P3f) · 필터·통계(P3g) · 카드 그리드(P3h) · 공유/임베드(P3i) · 분석다이얼로그(P3j) · 가사 탭(P3k)** 네 조각을 조립하고, 데이터 조회·페이징·URL 상태·다이얼로그 라우팅을 총괄하도록 만듭니다.

## ② Instructions (지시)

### 2.1 산출물 (Lovable 실천 원칙 "Prompt by Component, Not Page")
아래 파일만 생성/수정한다. 하위 컴포넌트는 임포트만 하고 본문은 각 하위 프롬프트(P3b–P3q)에서 정의한다.

- `src/pages/Songs.tsx` — 셸 컴포넌트.
- `src/hooks/useSongsQuery.ts` — `fetchWithRetry` + `AbortController` 기반 100/page(운영값 30/page) 페이지네이션 훅.
- `src/hooks/useSongFavorites.ts` — 즐겨찾기 ID Set 조회/토글 훅(구현은 P3h 와 공유).

임포트(구현은 각 하위 프롬프트):
```
import { ArchiveTopbar } from "@/components/songs/archive/ArchiveTopbar";        // P3b
import { AddSongFab } from "@/components/songs/archive/AddSongFab";              // P3b
import { ArchiveHero } from "@/components/songs/archive/ArchiveHero";            // P3f
import { ArtistShelf } from "@/components/songs/archive/ArtistShelf";            // P3f
import { EmotionShelf } from "@/components/songs/archive/EmotionShelf";          // P3f
import { TeachingShelf } from "@/components/songs/archive/TeachingShelf";        // P3f
import { ShelfSection } from "@/components/songs/archive/ShelfSection";          // P3f
import { MiniSongCard } from "@/components/songs/archive/MiniSongCard";          // P3f
import { SongStatsDrawer } from "@/components/songs/SongStatsDrawer";            // P3g
import { SongCard } from "@/components/songs/SongCard";                          // P3h
```

### 2.2 원자적 UI 규칙 (Lovable 실천 원칙 "Speak Atomic")

셸 페이지는 세로 방향으로 정확히 다음 블록만 이 순서로 렌더한다(`space-y-4`, `overflow-x-hidden`):

1. `<ArchiveTopbar />` — 스티키 상단(검색·언어 인디케이터·곡 추가 드롭다운) [P3b].
2. `분석 중 N곡` 배너(선택) — `pendingAnalyses.length > 0` 일 때만 `border-primary/30 bg-primary/5 text-primary` 로 렌더, `Loader2 animate-spin`.
3. `<ArchiveHero />` — 5장 자동 슬라이드 히어로(4500 ms 인터벌, 좌우 화살표, 도트, 통계 chip) [P3f].
4. `<ArtistShelf />` — 인기 아티스트 원형 12개 가로 스크롤 [P3f].
5. `<ShelfSection title="최근 추가된 곡" cardWidth={148}>` × `<MiniSongCard>` — `recentSongs` 상위 12곡 [P3f].
6. `<EmotionShelf />` — 12개 감정 그라디언트 칩 [P3f].
7. `<TeachingShelf />` — 17개 교학추천 카드 [P3f].
8. `<Collapsible open={fullListOpen}>` "전체 곡 목록 · {globalCount}곡" 접기/펼치기 트리거 → 내부에 **필터 행(P3d)** + **활성 필터 태그 행(P3d)** + **정렬 셀렉트(P3d)** + **카드 그리드(P3e)** + **페이지네이션(본 셸)**.
9. `<SongAnalysisDialog />`, `<YouTubeSearchDialog />`, `<YouTubeConfirmDialog />`, `<CsvImportDialog />`, `<BulkSongCardDialog />`, `<YoutubeVideoGenerateDialog />`, `<SongStatsDrawer />` — 오프스크린 다이얼로그·드로어(제어 상태만 셸이 소유).
10. `<AddSongFab />` — 우측 하단 플로팅(fixed) [P3b].

### 2.3 강제 제약 (Lovable 실천 원칙 "Design with Real Content")

- semantic token(`hsl(var(--dash-navy))` 등)만 사용, `bg-[#…]` / `text-white` / `bg-black` 하드코드 금지. 히어로 내부의 오버레이 그라디언트는 P3c 범위이므로 본 셸 파일에는 등장 금지.
- `<h1>` 은 페이지당 정확히 1개(`ArchiveTopbar` 내부에 위치). 셸에서는 추가 `<h1>` 금지.
- 모든 Supabase 조회는 `fetchWithRetry(() => query, { retries: 3, baseDelay: 400 })` 로 감싸며, 필터/페이지/언어 변경 시 이전 요청은 **반드시** `AbortController.abort()` 로 취소.
- 페이지 사이즈: `pageSize = 30`. Supabase `.range(from, to)`, `from = page * 30`, `to = from + 29`.
- URL 파라미터 동기화: `?themes=사랑,이별&cultures=감정표현&page=2` — `themeFilter` / `cultureFilter` / `page` 변경 시 `setSearchParams` 로 URL 반영, 새로고침 후 복원.
- 로컬 로컬 검색(`localSearchResults`)은 200 ms 디바운스로 `title ilike %q%` OR `artist ilike %q%` 조회.
- 순수 한국어 카피, lorem ipsum 금지. 모든 문자열은 §3.1 표에 확정.

## ③ Examples (예시)

### 3.1 확정 카피 표

| 위치 | 카피 |
|---|---|
| 진행 배너 | `분석 중 {N}곡` |
| Collapsible 트리거 좌측 | `전체 곡 목록` |
| Collapsible 배지 | `{globalCount}곡` |
| 로딩 스켈레톤 개수 | 3장(그리드 컬럼 기준) |
| 빈 상태 아이콘 | 🎵 (opacity 40%) |
| 빈 상태(전체) | `아직 등록된 노래가 없습니다.` / `위에서 YouTube URL을 입력하거나 검색 버튼을 눌러 노래를 추가해보세요.` |
| 빈 상태(필터) | `조건에 맞는 노래가 없어요` / `필터를 변경하거나 다른 검색어를 입력해보세요.` |
| 에러 상태 | ⚠️ / `연결에 실패했습니다` / `서버 연결이 일시적으로 불안정합니다. 잠시 후 다시 시도해주세요.` / [다시 시도] |
| 페이지네이션 입력 placeholder | 현재 페이지 번호 |
| 페이지네이션 라벨 | `페이지로 이동` |
| 429 토스트 | `요청이 많습니다. 잠시 후 다시 시도해주세요.` |

### 3.2 컴포넌트 트리 (Lovable 실천 원칙 "Use Prompt Patterns for Layouts")

```text
<Songs>  (space-y-4, overflow-x-hidden)
  ├─ <ArchiveTopbar />                             ── P3b
  ├─ [분석 중 N곡 배너]                            ── 본 셸
  ├─ <ArchiveHero />                               ── P3f
  ├─ <ArtistShelf />                               ── P3f
  ├─ <ShelfSection title="최근 추가된 곡">         ── P3f
  │     └─ <MiniSongCard> × ≤12
  ├─ <EmotionShelf />                              ── P3f
  ├─ <TeachingShelf />                             ── P3f
  ├─ <Collapsible fullListOpen>
  │     ├─ CollapsibleTrigger: "전체 곡 목록 · N곡"
  │     └─ CollapsibleContent
  │           ├─ FilterRow (LangTabs · [필터]팝오버 · ♥즐겨찾기) ── P3g
  │           ├─ SortSelect                        ── P3g
  │           ├─ ActiveFilterTags                  ── P3g
  │           ├─ SongGrid (SongCard × ≤30 + pending skeletons) ── P3h
  │           └─ Pagination (< 1 … N > + [페이지로 이동])   ── 본 셸
  ├─ <SongAnalysisDialog />                        ── P4(예정)
  ├─ <YouTubeSearchDialog />                       ── P3b 참조
  ├─ <YouTubeConfirmDialog />                      ── P3b 참조
  ├─ <CsvImportDialog />                           ── T2 참조
  ├─ <BulkSongCardDialog />                        ── T2 참조
  ├─ <YoutubeVideoGenerateDialog />                ── P7(AI 생성)
  ├─ <SongStatsDrawer />                           ── P3g
  └─ <AddSongFab />                                ── P3b
```

## ④ Context (배경)

### 4.1 프로젝트 맥락
「멜로디 클래스(멜로디 클래스)」의 `/songs` 는 로그인·비로그인 모두 접근 가능한 공용 아카이브 페이지이며, 사용자·교사·학생·관리자 전원의 진입점이다. 셸은 **오케스트레이터** 역할만 한다: 하위 컴포넌트에 데이터/콜백을 주입하고, 다이얼로그 열림 상태를 소유한다.

### 4.2 Lovable Cloud 후경 (Lovable 실천 원칙 "Build with Lovable Cloud in Mind")

4-상태 렌더 계약:
- **로딩**: `loading === true` → 그리드 자리에 shadcn `<Skeleton>` 3장(카드 컬럼 골격).
- **에러**: `fetchError && songs.length === 0` → 중앙 정렬 에러 블록 + [다시 시도].
- **빈**: `displaySongs.length === 0 && pendingAnalyses.length === 0` → 중앙 정렬 빈 블록(전체 vs 필터 결과 문구 분기).
- **성공**: 그리드 + 페이지네이션.

429/402 응답은 `fetchWithRetry` 가 한국어 토스트로 자동 노출(§3.1 카피 표).

### 4.3 데이터 계약 (셸이 소비하는 스키마 요약; 전체 DDL 은 P3e·P3d 참조)

`songs`(id, user_id, title, artist, language, video_id, hsk_level, theme, teaching_point, deleted_at, created_at) · `song_favorites`(user_id, song_id) · `song_culture_tags`(song_id, category) — RLS · GRANT 는 P3e 에서 정의.

셸의 조회 요약:
```sql
-- 목록 (예)
select *, song_analyses(id), song_culture_tags!inner(category)
from public.songs
where deleted_at is null
  and (owner filter · theme filter · culture filter · hsk range · language filter)
order by created_at desc
limit 30 offset :page*30;

-- 카운트: 언어별/전역 3회 병렬 카운트(head:true, count:'exact')
-- 최근 12곡: order by created_at desc limit 12 (필터 무시)
-- 로컬 검색: title ilike '%q%' or artist ilike '%q%' limit 8, 디바운스 200 ms
```

## ⑤ Acceptance & Output (검증·산출)

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6 · 정량 임계값)

- `rg -n "bg-\[#|text-white|bg-black" src/pages/Songs.tsx` 결과 = **0 건**.
- `rg -n "<h1" src/pages/Songs.tsx` = **0 건**(H1 은 P3b `ArchiveTopbar` 안에서만 1회 렌더).
- 30곡·1페이지 초기 로드 p95 ≤ **300 ms**(로컬 Lovable Cloud 기준). 필터 변경 시 이전 요청이 `AbortError` 로 취소되는지 콘솔 로그로 확인 가능.
- 새로고침 후 URL 파라미터로부터 `themeFilter` · `cultureFilter` · `page` 복원율 = **100 %**.
- 필터·페이지 변경 시 스크롤이 최상단으로 스무스 이동(behavior: "smooth"), 페이지 입력창 Enter 로 임의 페이지 이동 가능.
- 사용자 A 가 생성한 개인 곡(`user_id = A`)이 사용자 B 로그인 시 목록·카운트·최근 곡·검색 결과 어디에도 노출되지 않음(반환 행 수 = 0).
- 4-상태(loading/empty/error/success) 각각의 스크린샷을 재현할 수 있어야 함(Playwright 검증 대상).
- Lighthouse: 성능 ≥ **85**, 접근성 ≥ **95**, CLS ≤ **0.05**.

### 5.2 Output Format
반환 순서(부수 텍스트·마크다운 헤더·사과 금지):
1. `src/hooks/useSongsQuery.ts`
2. `src/hooks/useSongFavorites.ts`
3. `src/pages/Songs.tsx`
4. 한국어 3줄 요약(1줄: 사용된 5원칙 이름, 2줄: 페이지네이션/URL 동기화 요지, 3줄: 4-상태 렌더 요지).
