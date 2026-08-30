# P3c · 히어로 · 가로 스크롤 선반 재현 프롬프트

> **본 프롬프트는 P3 시리즈(P3a–P3q)의 6/17.**
> **적용 대상**: `src/components/songs/archive/` 하위의 히어로·선반 5종 및 미니 카드. 즉 `ArchiveHero.tsx`, `ArtistShelf.tsx`, `EmotionShelf.tsx`, `TeachingShelf.tsx`, `ShelfSection.tsx`, `ShelfArrows.tsx`, `MiniSongCard.tsx`, `artistImageMap.ts`.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* Retrieved July 12, 2026, from https://platform.openai.com/docs/guides/prompt-engineering
2. **Lovable. (n.d.).** *Prompting best practices.* Retrieved July 12, 2026, from https://docs.lovable.dev/prompting/prompting-one — 본 부록의 5개 실천 원칙 채택 근거.
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity (신원)

당신은 콘텐츠 아카이브의 첫인상을 결정하는 히어로/큐레이션 선반을 설계하는 시니어 프론트엔드 엔지니어이자 큐레이션 UX 라이터입니다. Netflix/Spotify 계열 가로 스크롤 셀프 패턴을 참고하되, 한중 이중언어 학습이라는 도메인에 맞춰 감정·문화 축을 앞세운 큐레이션을 구현합니다.

## ② Instructions

### 2.1 산출물 (Lovable 실천 원칙 "Prompt by Component, Not Page")

정확히 8개 파일:

- `src/components/songs/archive/ArchiveHero.tsx` — 5장 자동 슬라이드 히어로 + 통계 chip.
- `src/components/songs/archive/ArtistShelf.tsx` — 원형 아바타 12명 가로 스크롤.
- `src/components/songs/archive/EmotionShelf.tsx` — 12개 감정 그라디언트 칩(160×80).
- `src/components/songs/archive/TeachingShelf.tsx` — 17개 교학추천 카드(200 폭).
- `src/components/songs/archive/ShelfSection.tsx` — 제목/화살표/스냅 스크롤 컨테이너 프리미티브.
- `src/components/songs/archive/ShelfArrows.tsx` — `< >` 화살표 페어(ResizeObserver 로 활성/비활성 판정).
- `src/components/songs/archive/MiniSongCard.tsx` — 148×148 정사각 미니 카드(최근 곡용).
- `src/components/songs/archive/artistImageMap.ts` — 아티스트명 → `/public/artists/*.jpg` 매핑 테이블(상수 export).

### 2.2 원자적 UI 규칙 (Lovable 실천 원칙 "Speak Atomic")

**ArchiveHero** (`h-[200px] sm:h-[240px] md:h-[300px] rounded-2xl overflow-hidden`):
- 슬라이드 트랙: `flex h-full transition-transform duration-[1200ms] ease-[cubic-bezier(0.4,0,0.2,1)]`, `translateX(-idx*100%)`.
- 소스: `songs` 중 `song_analyses.length > 0` 인 항목만 뽑아 **오늘 날짜 시드 셔플** → 상위 5장. 분석된 항목이 0이면 전체에서 5장.
- 배경: `hqdefault.jpg` → 실패 시(`onError`) `GRADIENTS[i%5]` 그라디언트로 폴백.
- 오버레이: `bg-gradient-to-t from-black/85 via-black/40 to-black/10`.
- 콘텐츠(하단 정렬): 언어 뱃지(`KR #2563eb` / `CN #dc2626`) + 주제 pill + HSK pill + 제목(`text-xl md:text-3xl`) + 아티스트 + [▶ 분석 보기] [🔗 공유하기].
- 우측 상단 통계 chip 버튼: `전체 / 🇰🇷 / 🇨🇳` — 클릭 시 `onOpenStats()`.
- 좌우 화살표: `group-hover:opacity-100` 만 노출. 도트: 하단 중앙, 현재 도트 `w-6`, 나머지 `w-1.5`.
- 자동 회전 4500 ms, 사용자 조작 시 `resetTimer()`.

**ArtistShelf**:
- 서버 조회: `songs.select("artist").is("deleted_at", null).not("artist","is",null).limit(1000)` → 집계 top12.
- 원형 아바타 `w-20 h-20 rounded-full`, 이미지 소스 우선순위: `ARTIST_IMAGE_MAP[name]` → `/artists/{name}.jpg` → 이니셜 폴백. 그라디언트 12종 해시 배정.
- 하단 텍스트: `{name}` truncate max-w-80 + `{count}곡`.
- 로딩 시 스켈레톤 8개.

**EmotionShelf**: 상수 12개(사랑/이별/그리움/희망/외로움/인생/우정/가족애/정체성/비판/자연/기타) — 각 항목 `160×80 rounded-[10px]`, `linear-gradient(135deg, from → to)` (§4.3 팔레트), 좌측 이모지 · 우측 라벨+subtitle. 클릭 시 `onPickTheme(theme)` + `onScrollToList()`(1200 ms 이지드 스크롤).

**TeachingShelf**: 상수 17개 카드(감정표현/연애관/인간관계/자연경관/인생철학/가치관/심미적 이미지/동식물/음식과 기물/예술과 문학/지리명소/복식과 건축/명절과 절기/민속과 종교/역사와 시대/가정윤리/사회규범) — 각 카드 `w-[200px]`, 상단 3 px 그라디언트 스트라이프 + 이모지 + 제목 + 2줄 clamp 설명 + 태그 pill 최대 3개. 클릭 시 `onPickCulture(theme)` + `onScrollToList()`.

**ShelfSection**: `h3` 제목(옵션 subtitle) + 우측 `<ShelfArrows>`. 자식 배열을 자동 래핑하여 각 자식에 `cardWidth` 를 부여, `snap-x snap-mandatory` 컨테이너에 배치. 스크롤바 숨김(`[scrollbar-width:none] [&::-webkit-scrollbar]:hidden`).

**ShelfArrows**: `w-7 h-7 rounded-full bg-[#f0f0f5]` 페어. `scrollLeft`, `scrollWidth-clientWidth` 로 좌/우 가용성 판정, `ResizeObserver` 로 리사이즈 반응. 클릭당 `cardWidth * step(=3)` 만큼 smooth 스크롤.

**MiniSongCard**: `w-[148px]`, 상단 `148×148 rounded-[10px]` 썸네일(`maxresdefault.jpg` → 실패 시 해시 그라디언트). 좌상단 KR/CN 배지, 우상단 분석 완료 시 초록 체크(`✓`) 배지. 호버 오버레이 = 검정 반투명 + 중앙 `▶` 화이트 버튼. 하단 제목 1줄 + 아티스트 1줄.

### 2.3 강제 제약

- semantic token 우선. 히어로/미니카드의 언어 뱃지 색만 예외로 `#2563eb` / `#dc2626` 허용(브랜드 색 대비 필요).
- 모든 이미지 `loading="lazy"`, `onError` 폴백 필수 — CLS 방지.
- 히어로 자동 회전 인터벌은 `useEffect` cleanup 에서 반드시 해제.
- `ShelfArrows` 이벤트 리스너·ResizeObserver 도 unmount 시 정리.
- 감정/교학 선반 데이터는 상수 배열로만 유지(DB 조회 금지). 아티스트 선반만 서버 집계.
- 파일 최상단 `use client` 지시자 금지(Vite 환경).

## ③ Examples

### 3.1 확정 카피 표 (Lovable 실천 원칙 "Design with Real Content")

| 위치 | 카피 |
|---|---|
| 히어로 빈 상태 | `분석된 곡이 추가되면 여기에 추천이 표시됩니다.` |
| 히어로 CTA 1 | `분석 보기` (Play 아이콘) |
| 히어로 CTA 2 | `공유하기` (Share2 아이콘) |
| 히어로 chip 라벨 | `전체 / 🇰🇷 / 🇨🇳` |
| 아티스트 셀프 제목 | `인기 아티스트` |
| 최근 곡 셀프 제목 | `최근 추가된 곡` |
| 최근 곡 subtitle | `가장 최근에 아카이브에 추가된 곡들` |
| 감정 셀프 제목 | `주제로 찾기` |
| 교학 셀프 제목 | `교학 추천` |
| 교학 subtitle | `클릭하면 해당 주제 곡 목록으로 이동합니다` |
| ShelfArrows aria-label | `이전` / `다음` |
| 미니카드 폴백 | `제목 없음` / `아티스트 미상` |
| 아티스트 폴백 | 이니셜 1자 |
| 감정 12종 | 사랑 · 이별 · 그리움 · 희망 · 외로움 · 인생 · 우정 · 가족애 · 정체성 · 비판 · 자연 · 기타 |
| 교학 17종 | (§2.2 나열 그대로) |

### 3.2 컴포넌트 트리 (Lovable 실천 원칙 "Use Prompt Patterns for Layouts")

```text
<ArchiveHero>
  ├─ SlideTrack (translateX)
  │    └─ Slide × 5
  │          ├─ BgImage (hqdefault) or GradientFallback
  │          ├─ OverlayGradient
  │          ├─ Badges: [KR|CN] [Theme] [HSK]
  │          ├─ Title / Artist
  │          └─ CTAs: [▶ 분석 보기] [🔗 공유하기]
  ├─ StatsChip (top-right, onOpenStats)
  ├─ Arrows (< >)
  └─ Dots (bottom-center)

<ShelfSection title subtitle cardWidth>
  ├─ Header: <h3> + subtitle + <ShelfArrows>
  └─ Scroller (flex snap-x, hide-scrollbar)
        └─ children[i] wrapped { width:cardWidth }

<ArtistShelf>          → circles × 12
<EmotionShelf>         → gradient chips × 12  (160×80)
<TeachingShelf>        → theme cards × 17     (w-200)
<MiniSongCard>         → 148×148 thumb + title/artist
```

## ④ Context

### 4.1 프로젝트 맥락
아카이브 상단 스크롤 영역은 **탐색 우선(browse-first)** 원칙을 따른다: 학습자가 검색어를 몰라도 시각적 큐레이션을 통해 곡에 도달할 수 있어야 한다. Hero(오늘의 추천 5) → Artist(사람) → Recent(시간) → Emotion(감정) → Teaching(문화) 순으로 축이 넓어지며, 클릭 시 모두 셸의 필터 상태를 갱신해 하단 `전체 곡 목록` 을 필터링한다.

### 4.2 Lovable Cloud 후경 (Lovable 실천 원칙 "Build with Lovable Cloud in Mind")

- ArchiveHero: 데이터는 셸이 주입. 슬라이드 5장 이하일 때 화살표·도트 숨김.
- ArtistShelf: 자체 조회. 4-상태 = 로딩(스켈레톤 8) / 빈(아티스트 0 → null 반환) / 에러(supabase 오류 시 조용히 null) / 성공.
- 이미지 실패 시 그라디언트 폴백 — 네트워크 실패에도 시각적 완결성 유지.

### 4.3 데이터 계약 (참고 팔레트·상수)

`EMOTION_PALETTE` (from → to):
```
사랑     #831843 #be185d      이별     #1c1917 #44403c
그리움   #1e1b4b #3730a3      희망     #713f12 #d97706
외로움   #0c4a6e #0369a1      인생     #14532d #15803d
우정     #134e4a #0f766e      가족애   #431407 #9a3412
정체성   #2e1065 #6d28d9      비판     #7f1d1d #b91c1c
자연     #052e16 #166534      기타     #27272a #52525b
```

`ArtistShelf` 쿼리는 `songs.select("artist").is("deleted_at", null).not("artist","is",null).limit(1000)`. RLS 는 P3e 에서 정의.

## ⑤ Acceptance & Output

### 5.1 Acceptance Criteria

- ArchiveHero 자동 회전 인터벌이 4500 ± 50 ms. 조작 시 리셋 확인.
- Slide 이미지 로드 실패 시 500 ms 내 그라디언트 폴백 렌더(스크린샷 검증).
- ArtistShelf 12장 초과 곡 데이터에서 상위 12명만 노출(테스트 fixture 로 검증).
- EmotionShelf 12개, TeachingShelf 17개 카드 렌더 개수 완전 일치. 카피 스트링 100 % 일치.
- ShelfArrows: 스크롤 위치가 좌/우 끝일 때 해당 화살표 `disabled` 속성 = true. ResizeObserver 언마운트 후 leak 0.
- MiniSongCard 호버 오버레이 opacity transition ≤ 250 ms.
- `rg -n "bg-\[#|text-white" src/components/songs/archive` 실행 시 히어로/미니카드의 언어 뱃지(`#2563eb` / `#dc2626`) 이외의 하드코드 색상 매칭 = **0**.
- Lighthouse Perf ≥ 85, CLS ≤ 0.05(이미지 명시 width/height + lazy).

### 5.2 Output Format
반환 순서:
1. `src/components/songs/archive/ShelfArrows.tsx`
2. `src/components/songs/archive/ShelfSection.tsx`
3. `src/components/songs/archive/ArchiveHero.tsx`
4. `src/components/songs/archive/artistImageMap.ts`
5. `src/components/songs/archive/ArtistShelf.tsx`
6. `src/components/songs/archive/EmotionShelf.tsx`
7. `src/components/songs/archive/TeachingShelf.tsx`
8. `src/components/songs/archive/MiniSongCard.tsx`
9. 한국어 3줄 요약.
