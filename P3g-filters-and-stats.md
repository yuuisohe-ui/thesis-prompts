# P3d · 필터 · 정렬 · 통계 대시보드 재현 프롬프트

> **본 프롬프트는 P3 시리즈(P3a–P3q)의 7/17.**
> **적용 대상**: `Songs.tsx` 내 "전체 곡 목록" 접기 영역의 필터 팝오버 · 언어 탭 · 즐겨찾기 토글 · 정렬 · 활성 필터 태그, 그리고 `src/components/songs/SongStatsDrawer.tsx` 대시보드 전체.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* https://platform.openai.com/docs/guides/prompt-engineering (Retrieved July 12, 2026)
2. **Lovable. (n.d.).** *Prompting best practices.* https://docs.lovable.dev/prompting/prompting-one (Retrieved July 12, 2026) — 본 부록의 5개 실천 원칙 채택 근거.
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity (신원)

당신은 다중 축 필터 UX 와 학습 콘텐츠 통계 대시보드를 함께 설계하는 시니어 프론트엔드 엔지니어이자 데이터 시각화 설계자입니다. shadcn/ui `Popover`, `Dialog`, `Select` 와 `recharts`(`PieChart`, `AreaChart`, `BarChart`) 를 사용합니다.

## ② Instructions

### 2.1 산출물 (Lovable 실천 원칙 "Prompt by Component, Not Page")

- `src/components/songs/archive/FilterBar.tsx` — 언어 탭 + [필터] 팝오버 + ♥즐겨찾기 토글 + 정렬 셀렉트.
- `src/components/songs/archive/ActiveFilterTags.tsx` — 활성 필터 chip 목록 + 개별 ✕ + [전체 초기화].
- `src/components/songs/SongStatsDrawer.tsx` — max-w-5xl · h-[90vh] 대시보드 Dialog.
- `src/hooks/useSongFilters.ts` — 필터 상태·URL 동기화 훅(themeFilter, cultureFilter, levelFilter, tpFilter, ownerFilter, langView, favOnly, sortOption).

### 2.2 원자적 UI 규칙 (Lovable 실천 원칙 "Speak Atomic")

**FilterBar** = `flex items-center justify-between gap-2.5 flex-wrap`:

- **언어 탭**(`langView: "all"|"korean"|"chinese"`) — pill 3개.
- **[필터] 버튼** = `Filter` 아이콘 + `필터` + `{activeCount}` 배지. 클릭 시 `Popover`(300 폭) 열림. 팝오버 내부는 정확히 4 섹션(각 사이에 `h-px bg-border` 구분선):
  1. **난이도** (`all/초급/중급/고급/심화`) — 단일 선택 pill 5개. 각 레벨은 고정 색조(초급:teal, 중급:blue, 고급:gold, 심화:destructive).
  2. **주제** — 12개 THEME_OPTIONS(사랑…기타) 다중 선택 pill, 활성 시 pink 톤.
  3. **교학 포인트** (`all/어휘/문법/문화/발음`) — 단일 선택, 활성 시 purple 톤.
  4. **소유** (`all/mine/public`) — 단일 선택, 활성 시 blue 톤.
  하단: [초기화] 텍스트 · [적용] navy 버튼.
- **♥ 즐겨찾기 토글** — 로그인 필요, 미로그인 시 `로그인 후 이용 가능합니다` 토스트 후 no-op. 활성 시 `border-pink-400 text-pink-600 bg-pink-50`, 우측에 즐겨찾기 개수 배지.
- **정렬 셀렉트** — `<select>` 4옵션: `최신 추가순 / 난이도 낮은순 / 난이도 높은순 / 가나다순`.

**ActiveFilterTags** = 활성 chip 목록(레벨/교학/소유/주제/문화). 주제·문화는 pink 톤, 나머지는 navy 톤. 각 chip 우측 `✕` 클릭 시 개별 해제. 마지막에 `전체 초기화` 텍스트 링크.

**SongStatsDrawer**(shadcn Dialog): 헤더 = `<BarChart3>` 아이콘 + `노래 통계 대시보드` 타이틀. 본문 =
- 상단 **KPI 카드 4개**(총 곡수 · 중국어 · 한국어 · 아티스트 수) — 아이콘 · 값 · 라벨.
- **StatSection** 프리미티브(제목 대문자 tracking-wider + accent 컬러 스트라이프 + children):
  - `언어 비율` (PieChart 2조각, col-span-4).
  - `주제 분포` (12테마 bar/pie 하이브리드, `분석 완료 X / 전체 Y곡` 서브라벨). 클릭 시 `onThemeClick(theme)` 로 상위에 필터 반영 후 닫힘.
  - `문화 분포` (17개 카테고리 bar).
  - `연도별 수록 추이` (AreaChart, col-span-5).
  - `인기 아티스트 TOP 12` (수평 BarChart).
- 데이터 조회: `fetchAllPaged` 헬퍼로 1000-row cap 우회, `fetchWithRetry` 로 감싸며, Dialog 오픈 시 1회만 로드.

### 2.3 강제 제약

- semantic token 우선. 팔레트 상수(`--dash-*`) 만 사용 — 새 hex 색상 도입 금지(예외: recharts 데이터 시리즈용 12색 상수 `BUBBLE_COLORS` 는 허용).
- 필터 상태는 훅에 캡슐화, 컴포넌트는 stateless.
- 팝오버 내부 클릭이 바깥 클릭으로 오인되어 닫히지 않도록 `Popover` 컴포넌트 표준 이벤트만 사용(수동 `document.click` 리스너 금지).
- `SongStatsDrawer` 는 `max-w-5xl w-[92vw] h-[90vh] overflow-y-auto`, 모바일에서 스크롤 성능 유지.
- 대시보드 데이터 최소 활성 조회 횟수: Dialog 열림당 1회(재열림 시에도 캐시 우선).

## ③ Examples

### 3.1 확정 카피 표 (Lovable 실천 원칙 "Design with Real Content")

| 위치 | 카피 |
|---|---|
| 언어 탭 | `전체 / 한국어 / 중국어` |
| 필터 버튼 라벨 | `필터` |
| 필터 배지 | 활성 필터 개수 정수 |
| 팝오버 섹션 라벨 | `난이도` / `주제` / `교학 포인트` / `소유` |
| 소유 옵션 | `전체` / `내 곡` / `공용` |
| 팝오버 초기화 | `초기화` |
| 팝오버 적용 | `적용` |
| 즐겨찾기 라벨 | `즐겨찾기` |
| 즐겨찾기 title | `내 즐겨찾기만 보기` |
| 즐겨찾기 미로그인 토스트 | `로그인 후 이용 가능합니다` |
| 정렬 옵션 | `최신 추가순 / 난이도 낮은순 / 난이도 높은순 / 가나다순` |
| 활성 태그 접두 | `난이도: · 교학: · 주제: · 문화:` / 소유는 `내 곡` 또는 `공용` |
| 전체 초기화 링크 | `전체 초기화` |
| 통계 헤더 | `노래 통계 대시보드` |
| KPI 라벨 | `총 곡수 / 중국어 / 한국어 / 아티스트` |
| StatSection 제목 | `언어 비율 / 주제 분포 / 문화 분포 / 연도별 수록 추이 / 인기 아티스트 TOP 12` |
| 주제 서브라벨 | `분석 완료 {X} / 전체 {Y}곡` |

### 3.2 컴포넌트 트리 (Lovable 실천 원칙 "Use Prompt Patterns for Layouts")

```text
<FilterBar>
  ├─ LanguageTabs [전체 · 한국어 · 중국어]
  ├─ FilterPopover
  │    ├─ Section: 난이도  (chip × 5, single)
  │    ├─ Section: 주제    (chip × 12, multi)
  │    ├─ Section: 교학    (chip × 5, single)
  │    ├─ Section: 소유    (chip × 3, single)
  │    └─ Footer: [초기화]  [적용]
  ├─ FavoritesToggle (♥ 즐겨찾기 · count 배지)
  └─ SortSelect

<ActiveFilterTags>
  ├─ Chip × N  (navy | pink)
  └─ [전체 초기화]

<SongStatsDrawer  max-w-5xl h-[90vh]>
  ├─ Header (BarChart3 · 노래 통계 대시보드)
  ├─ KpiGrid × 4
  └─ StatSection × 5
        ├─ 언어 비율      (PieChart)
        ├─ 주제 분포      (Bar/Pie, 클릭→필터)
        ├─ 문화 분포      (BarChart)
        ├─ 연도별 추이   (AreaChart)
        └─ TOP 12 아티스트 (Horizontal BarChart)
```

## ④ Context

### 4.1 프로젝트 맥락
필터/정렬은 하단 `전체 곡 목록` 접기 영역의 카드 그리드(P3e)에 대한 **서버 사이드** 조건이다. 즐겨찾기 필터는 `song_favorites` 조인 결과로 클라이언트에서 최종 필터링. 통계 드로어는 셸의 `<BarChart3>` chip 및 히어로 우측 통계 chip 어느 쪽에서 열어도 동일 인스턴스가 열린다.

### 4.2 Lovable Cloud 후경 (Lovable 실천 원칙 "Build with Lovable Cloud in Mind")

- 대시보드 로딩 = 각 StatSection 별 `<Skeleton>`. 
- 대시보드 에러 = 상단 배너 + [다시 시도].
- `cultureFilter` 활성 시 목록 카운트는 `song_culture_tags` 조인으로 중복 발생 → distinct song_id 로 dedupe 후 count.
- 429/402 는 `fetchWithRetry` 가 한국어 토스트 자동 노출.

### 4.3 데이터 계약 (필터 파생 쿼리)

```sql
-- 예: langView='chinese', levelFilter='중급', ownerFilter='public', themes=['사랑'], cultures=['감정표현']
select s.*
from public.songs s
join public.song_culture_tags c on c.song_id = s.id
where s.deleted_at is null
  and s.user_id is null
  and s.language <> 'korean'
  and s.hsk_level in ('HSK 3','HSK 4')
  and s.theme in ('사랑')
  and c.category in ('감정표현')
order by s.created_at desc
offset :p*30 limit 30;
```

`song_culture_tags`:
```sql
create table public.song_culture_tags (
  song_id uuid not null references public.songs(id) on delete cascade,
  category text not null,
  primary key (song_id, category)
);
grant select on public.song_culture_tags to anon, authenticated;
grant insert, delete on public.song_culture_tags to authenticated;
grant all on public.song_culture_tags to service_role;
alter table public.song_culture_tags enable row level security;
create policy "read culture tags of readable songs" on public.song_culture_tags
  for select using (
    exists (select 1 from public.songs s where s.id = song_culture_tags.song_id
            and (s.user_id is null or s.user_id = auth.uid()))
  );
create policy "owner writes culture tags" on public.song_culture_tags
  for all using (
    exists (select 1 from public.songs s where s.id = song_culture_tags.song_id and s.user_id = auth.uid())
  );
```

## ⑤ Acceptance & Output

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6)

- 필터 변경 시 URL(`?themes=` · `?cultures=` · `?page=`) 이 즉시 반영, 새로고침 후 복원율 = **100 %**.
- 팝오버 [초기화] 클릭 시 4 섹션 전부 `all`/빈 배열로 리셋(단, 팝오버 내 로컬 상태와 실제 상태 일치).
- 활성 필터 태그 개수 = 실제 활성 필터 수(테스트: 3축 활성 시 chip 3개 + 전체 초기화 링크 1).
- `SongStatsDrawer` 오픈 후 500 ms 내 KPI 4개 렌더(캐시 시 100 ms 이하), 5개 StatSection 모두 렌더 완료 p95 ≤ **2 s**(로컬 Lovable Cloud, 1000곡 fixture).
- 주제 분포 클릭 → 드로어 닫힘 + 상위 `themeFilter` 에 해당 theme 추가.
- `rg -n "document\.addEventListener\(.click." src/components/songs/archive/FilterBar.tsx` = **0**(팝오버 표준 이벤트만 사용).
- 정렬 옵션 4종 각각에 대해 `songs` 배열 정렬 결과가 서버 반환 순서와 일치(스냅샷 테스트).
- `rg -n "bg-\[#|text-white" src/components/songs/archive/FilterBar.tsx src/components/songs/archive/ActiveFilterTags.tsx` = **0**(SongStatsDrawer 는 recharts 색상 상수만 예외).

### 5.2 Output Format
반환 순서:
1. Postgres 마이그레이션 SQL(`song_culture_tags` — 아직 없다면).
2. `src/hooks/useSongFilters.ts`
3. `src/components/songs/archive/FilterBar.tsx`
4. `src/components/songs/archive/ActiveFilterTags.tsx`
5. `src/components/songs/SongStatsDrawer.tsx`
6. 한국어 3줄 요약.
