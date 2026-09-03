# P3g · 필터 · 정렬 · 통계 대시보드 재현 프롬프트

> **본 프롬프트는 P3 시리즈(P3a–P3q)의 7/17.**
> **적용 대상**: `src/pages/Songs.tsx` 내 "전체 곡 목록"(Collapsible) 영역의 언어 탭 · 필터 팝오버 · 즐겨찾기 토글 · 정렬 셀렉트 · 활성 필터 태그, 그리고 `src/components/songs/SongStatsDrawer.tsx` 통계 대시보드 전체.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**
> **기준 시점**: 현재 배포 코드(`Songs.tsx` 1,416행 · `SongStatsDrawer.tsx` 619행).

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* https://platform.openai.com/docs/guides/prompt-engineering (Retrieved July 12, 2026)
2. **Lovable. (n.d.).** *Prompting best practices.* https://docs.lovable.dev/prompting/prompting-one (Retrieved July 12, 2026) — 본 부록의 5개 실천 원칙 채택 근거.
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity (신원)

당신은 다중 축 필터 UX 와 학습 콘텐츠 통계 대시보드를 함께 설계하는 시니어 프론트엔드 엔지니어이자 데이터 시각화 설계자입니다. 스택은 React 18 + Vite 5 + Tailwind CSS v3 + shadcn/ui + Supabase(JS v2) 이며, 차트는 `recharts`(`PieChart`, `AreaChart`)와 자체 구현(수평 바 · 버블 패킹)을 병용합니다. 대상 사용자는 대한민국 대학의 K-Chinese/K-Korean 교사·학습자입니다.

## ② Instructions

### 2.1 산출물 (Lovable 실천 원칙 "Prompt by Component, Not Page")

- `src/pages/Songs.tsx` — 필터/정렬 상태와 UI 를 **페이지 내부에 인라인으로** 보유한다. 별도의 `FilterBar` / `ActiveFilterTags` / `useSongFilters` 파일을 만들지 않는다(현행 구조 유지).
  - 필터 상태: `langView`(`all|chinese|korean`), `levelFilter`(`all|초급|중급|고급|심화`), `tpFilter`(`all|어휘|문법|문화|발음·반복`), `ownerFilter`(`all|mine|public`), `themeFilter: string[]`, `cultureFilter: string[]`, `favOnly: boolean`, `sortOption`(`newest|lv-asc|lv-desc|alpha`), `page`(pageSize = 30).
  - 쿼리 헬퍼: `applyFilters(query)`, `cultureEmbeds()`, `applyCultureFilters(query)`.
  - 파생값: `activeFilterCount`, `activeFilterTags`(`useMemo`), `removeFilter(key)`, `displaySongs`.
- `src/components/songs/SongStatsDrawer.tsx` — `Dialog` 기반 통계 대시보드. 내부에 `fetchAllPaged` 헬퍼, `THEME_META`(12테마 한자·색·이모지), `BUBBLE_COLORS`(17색), `layoutBubbles()`, `StatSection` 프리미티브를 포함한다.

### 2.2 원자적 UI 규칙 (Lovable 실천 원칙 "Speak Atomic")

**전체 곡 목록 셸** — `Collapsible`(트리거: `전체 곡 목록` + `{globalCount}곡` 배지 + 회전하는 `ChevronDown`). 열림/닫힘 상태는 `localStorage["songs.fullListExpanded"]`("1"/"0")에 저장하고 초기값은 열림이다.

**필터 행** = `flex items-center justify-between gap-2.5 flex-wrap min-w-0`:

- **언어 탭** 3개 — 순서와 라벨은 `전체 / 중국어 / 한국어`, 각 버튼 우측에 현재 필터 기준 개수를 `(n)` 으로 표기하고 재조회 중(`filtering`)에는 `(…)` 로 대체한다. 활성 pill 은 `bg-[hsl(var(--dash-navy))] text-white`.
- **인라인 로딩 칩** — `filtering && !loading` 일 때만 `Loader2` 스피너 + `불러오는 중…`, `role="status" aria-live="polite"`.
- **[필터] 버튼** — `Filter` 아이콘 + `필터` + `activeFilterCount` 배지(0이면 배지 없음). 클릭 시 shadcn `Popover` 가 아니라 **`absolute` 로 띄우는 자체 패널**(`w-[300px]`, `top-[calc(100%+6px)]`)을 토글한다. 바깥 클릭 닫힘은 `filterRef` 와 `document.addEventListener("click", …)` 로 구현하고, 버튼 클릭 핸들러에서 `e.stopPropagation()` 을 호출한다.
- **필터 패널 4 섹션**(각 사이 `h-px bg-border` 구분선, 섹션 라벨은 `10.5px` uppercase tracking-wider):
  1. **난이도** — `전체/초급/중급/고급/심화` 단일 선택 pill 5개, 레벨별 고정 색조.
  2. **주제** — `THEME_OPTIONS` 12개(`사랑 이별 그리움 외로움 우정 가족애 희망 인생 자연 정체성 비판 기타`) 다중 선택 pill, 활성 시 pink 톤.
  3. **교학 포인트** — `전체/어휘/문법/문화/발음·반복` 단일 선택, 활성 시 purple 톤.
  4. **소유** — `전체/내 곡/공용` 단일 선택, 활성 시 blue 톤.
  하단 푸터: 좌측 `초기화`(난이도·교학·소유만 초기화하며 **주제·문화는 유지**), 우측 `적용` navy 버튼(패널을 닫기만 하며 별도 커밋 동작이 없다 — 모든 필터는 클릭 즉시 반영되는 낙관적 상태다).
- **♥ 즐겨찾기 토글** — 미로그인 시 `로그인 후 이용 가능합니다` destructive 토스트 후 no-op. 활성 시 `border-pink-400 text-pink-600 bg-pink-50`, 로그인 상태에서만 `favoriteIds.size` 배지 표시. 즐겨찾기는 **서버 쿼리가 아니라 클라이언트 필터**(`displaySongs = favOnly ? songs.filter(s => favoriteIds.has(s.id)) : songs`)이므로 `activeFilterCount` 에 포함되지 않는다.
- **정렬 셀렉트** — 네이티브 `<select>` 4옵션: `최신 추가순(newest) / 난이도 낮은순(lv-asc) / 난이도 높은순(lv-desc) / 가나다순(alpha)`.

**활성 필터 태그** — `activeFilterTags` 가 1개 이상일 때만 렌더. 라벨 규칙은 `난이도: {v}` · `교학: {v}` · 소유는 `내 곡`/`공용` · `주제: {v}` · `문화: {v}`. 주제·문화 chip 은 `--dash-pink` 톤, 나머지는 navy 톤. 각 chip 우측 `✕` 는 `removeFilter(key)`(`lv|tp|owner|theme:{t}|culture:{t}`). 마지막에 `전체 초기화` 텍스트 버튼(5축 전부 리셋).

**쿼리 규칙(`applyFilters`)**:
- 항상 `.is("deleted_at", null)`.
- `langView === "chinese"` → `.or("language.is.null,language.neq.korean")`(중국어 버킷 = 한국어가 아닌 전부), `"korean"` → `.eq("language","korean")`.
- `levelFilter` → `levelToHskValues` 매핑으로 `.in("hsk_level", [...])`(초급 HSK1–2 / 중급 3–4 / 고급 5–6 / 심화 7–9).
- `tpFilter` → `.eq("teaching_point", v)`, `ownerFilter` → `mine`이면 `.eq("user_id", user.id)`, `public`이면 `.is("user_id", null)`.
- `themeFilter` → **OR** 의미의 `.in("theme", themes)`.
- `cultureFilter` → **AND** 의미. 선택 개수만큼 `ct{i}:song_culture_tags!inner(category,keywords)` 별칭 임베드를 만들고 `.eq("ct{i}.category", cat).not("ct{i}.keywords","eq","{}")` 를 건다. `(song_id, category)` 가 유니크하므로 행 중복이 없어 `count` 가 정확하다.
- 정렬은 `hsk_level` / `title` / `created_at desc` 로 서버 정렬, 페이징은 `.range(page*30, page*30+29)`.

**SongStatsDrawer**(shadcn `Dialog`, `max-w-[1180px] w-[94vw] h-[90vh] p-0 flex flex-col overflow-hidden sm:rounded-2xl`):
- 헤더 — 그라디언트 배경(`--dash-blue-bg` → `--dash-purple-bg`), `BarChart3` 아이콘 + `노래 아카이브 통계` + `실시간 · {total}곡` 배지, 우측 `새로고침` 고스트 버튼(`refreshKey` 증가, 로딩 중 비활성 + 스피너).
- 본문 배경 `--dash-surface2`, `overflow-y-auto`.
- **KPI 카드 4개**(`grid-cols-4`) — `총 수록곡 / 아티스트 / 중국어 / 한국어`. 각 카드는 좌측 4px 컬러 스트라이프 + 라벨 + 3xl 값(`toLocaleString()`) + 아이콘 배지, 로딩 시 `Skeleton`.
- **StatSection** 프리미티브 — 제목(12px bold uppercase tracking-wider) + accent 컬러 스트라이프 + 우측 슬롯(`right`) + children.
- Row 1(`grid-cols-12`): `언어 비율`(col-span-4, recharts 도넛 `PieChart` + 중앙 총 수록곡 + 하단 범례) / `주제 분포`(col-span-8, 자체 수평 바 목록, 우측 슬롯 `분석 완료 {X} / 전체 {Y}곡`, 각 행 클릭 시 `onThemeClick(name)` 후 `onOpenChange(false)`).
- Row 2: `문화 테마 분포`(col-span-7, `layoutBubbles()` 로 계산한 **버블 차트**, 720×360, 우측 슬롯 `{n}개 카테고리`) / `연도별 수록 추이`(col-span-5, recharts `AreaChart` + `linearGradient#yearGradient` + 연도 간격 자동 tick).
- Row 3: `인기 아티스트 TOP 12` — recharts 가 아니라 `grid-cols-3` 카드 12개, 상대 비율 바 폭 + 언어별 색(zh `#d4580a` / ko `#2557a7`).
- 데이터 로딩 — Dialog 가 `open` 될 때(그리고 `refreshKey` 변경 시) `Promise.all` 로 count 3건 + `fetchAllPaged` 4건(artists / years / themes / song_culture_tags)을 실행한다. `fetchAllPaged` 는 1000행 단위로 반복 조회하여 Supabase 기본 1000-row cap 을 우회하고 각 페이지를 `fetchWithRetry` 로 감싼다. 언마운트/재호출 시 `cancelled` 플래그로 늦게 도착한 응답을 무시한다.

### 2.3 강제 제약

- semantic token(`--dash-navy`, `--dash-blue`, `--dash-pink`, `--dash-purple`, `--dash-surface2` …) 우선. 차트 데이터 시리즈 색상 상수(`THEME_META.color`, `BUBBLE_COLORS`, 언어별 `#d4580a`/`#2557a7`)는 예외로 허용한다.
- 모든 목록/통계 조회는 `fetchWithRetry` 로 감싸고 AbortController 시그널을 전달한다.
- 필터가 바뀌면 `page` 를 0으로 리셋한다(`prevFiltersRef` 비교, `favOnly` 는 서버 쿼리에 관여하지 않으므로 제외).
- `themeFilter` / `cultureFilter` 만 URL 쿼리(`?themes=`, `?cultures=`)에 `replace: true` 로 동기화한다. `page`, `langView`, `levelFilter`, `tpFilter`, `ownerFilter`, `sortOption` 은 URL 에 싣지 않는다.
- 언어별 개수(`totalChineseCount` / `totalKoreanCount`)는 언어 축을 제외한 동일 필터로 별도 count 쿼리를 돌려 계산하고, `countsSigRef` 로 필터 시그니처가 바뀔 때만 재계산한다.
- 목록 재조회 중에도 기존 카드를 유지하고(`filtering` 플래그) 전체 화면 스켈레톤은 최초 로딩(`loading`)에서만 사용한다.
- 한국어 카피만 사용한다(lorem ipsum 금지).

## ③ Examples

### 3.1 확정 카피 표 (Lovable 실천 원칙 "Design with Real Content")

| 위치 | 카피 |
|---|---|
| 접기 트리거 | `전체 곡 목록` + `{globalCount}곡` |
| 언어 탭 | `전체 / 중국어 / 한국어` (+ `(n)` · 로딩 시 `(…)`) |
| 인라인 로딩 | `불러오는 중…` |
| 필터 버튼 라벨 | `필터` |
| 필터 배지 | `activeFilterCount` 정수 |
| 패널 섹션 라벨 | `난이도` / `주제` / `교학 포인트` / `소유` |
| 난이도 옵션 | `전체 / 초급 / 중급 / 고급 / 심화` |
| 교학 포인트 옵션 | `전체 / 어휘 / 문법 / 문화 / 발음·반복` |
| 소유 옵션 | `전체` / `내 곡` / `공용` |
| 패널 푸터 | `초기화` · `적용` |
| 즐겨찾기 라벨 | `즐겨찾기` |
| 즐겨찾기 title | `내 즐겨찾기만 보기` |
| 즐겨찾기 미로그인 토스트 | `로그인 후 이용 가능합니다` |
| 정렬 옵션 | `최신 추가순 / 난이도 낮은순 / 난이도 높은순 / 가나다순` |
| 활성 태그 접두 | `난이도: · 교학: · 주제: · 문화:` / 소유는 `내 곡` 또는 `공용` |
| 전체 초기화 링크 | `전체 초기화` |
| 결과 없음(필터) | `조건에 맞는 노래가 없어요` / `필터를 변경하거나 다른 검색어를 입력해보세요.` |
| 결과 없음(빈 DB) | `아직 등록된 노래가 없습니다.` / `위에서 YouTube URL을 입력하거나 검색 버튼을 눌러 노래를 추가해보세요.` |
| 조회 실패 | `연결에 실패했습니다` / `서버 연결이 일시적으로 불안정합니다. 잠시 후 다시 시도해주세요.` / `다시 시도` |
| 통계 헤더 | `노래 아카이브 통계` + `실시간 · {total}곡` |
| 통계 새로고침 | `새로고침` |
| KPI 라벨 | `총 수록곡 / 아티스트 / 중국어 / 한국어` |
| StatSection 제목 | `언어 비율 / 주제 분포 / 문화 테마 분포 / 연도별 수록 추이 / 인기 아티스트 TOP 12` |
| 주제 서브라벨 | `분석 완료 {X} / 전체 {Y}곡` |
| 문화 서브라벨 | `{n}개 카테고리` |
| 주제 행 title | `{테마} ({한자}) — {n}곡 클릭하여 필터링` |
| 도넛 중앙 | `{total}` + `총 수록곡` |
| 연도 툴팁 | `{n}곡` / `{year}년` |

### 3.2 컴포넌트 트리 (Lovable 실천 원칙 "Use Prompt Patterns for Layouts")

```text
<Songs.tsx>
└─ <Collapsible 전체 곡 목록  ({globalCount}곡)>
     ├─ FilterRow
     │    ├─ LanguageTabs [전체(n) · 중국어(n) · 한국어(n)]
     │    ├─ InlineLoadingChip (불러오는 중…)
     │    ├─ FilterButton + AbsolutePanel(w-300)
     │    │      ├─ Section 난이도  (chip × 5, single)
     │    │      ├─ Section 주제    (chip × 12, multi)
     │    │      ├─ Section 교학    (chip × 5, single)
     │    │      ├─ Section 소유    (chip × 3, single)
     │    │      └─ Footer [초기화]        [적용]
     │    ├─ FavoritesToggle (♥ 즐겨찾기 · {favoriteIds.size})
     │    └─ SortSelect (native <select> × 4)
     ├─ ActiveFilterTags  (chip × N + [전체 초기화])
     ├─ SongGrid  (1 / 2 / 3 / 4 / 5 cols · pageSize 30)
     └─ Pagination

<SongStatsDrawer  max-w-[1180px] w-[94vw] h-[90vh]>
  ├─ Header (BarChart3 · 노래 아카이브 통계 · 실시간 배지 · [새로고침])
  ├─ KpiGrid × 4   (총 수록곡 · 아티스트 · 중국어 · 한국어)
  ├─ Row1  grid-cols-12
  │     ├─ StatSection 언어 비율        col-span-4  (recharts PieChart 도넛)
  │     └─ StatSection 주제 분포        col-span-8  (수평 바, 클릭→필터+닫힘)
  ├─ Row2  grid-cols-12
  │     ├─ StatSection 문화 테마 분포   col-span-7  (layoutBubbles 버블)
  │     └─ StatSection 연도별 수록 추이 col-span-5  (recharts AreaChart)
  └─ StatSection 인기 아티스트 TOP 12   (grid-cols-3 랭킹 카드 × 12)
```

## ④ Context

### 4.1 프로젝트 맥락
필터/정렬은 「멜로디 클래스」 노래 아카이브 하단 `전체 곡 목록` 접기 영역의 카드 그리드(P3e)에 대한 **서버 사이드** 조건이다. 상단 히어로·셸프(P3a–P3d)는 이 필터의 영향을 받지 않는다. 즐겨찾기만 `song_favorites` 조회 결과로 클라이언트에서 최종 필터링한다. 통계 드로어는 상단 통계 chip 어느 진입점에서 열어도 동일 인스턴스가 열리며, `주제 분포` 클릭은 드로어를 닫고 상위 `themeFilter` 에 해당 테마를 반영한다.

### 4.2 Lovable Cloud 후경 (Lovable 실천 원칙 "Build with Lovable Cloud in Mind")

- 목록 4-상태: 최초 로딩 = 카드 스켈레톤 3개 / 재조회 = 기존 카드 유지 + `불러오는 중…` 칩 / 에러 = `연결에 실패했습니다` + [다시 시도] / 빈 결과 = 필터 여부에 따라 두 가지 카피.
- 대시보드 로딩 = 각 StatSection 별 `<Skeleton>`(높이 260 / 360). 실패 시 콘솔 경고 후 직전 데이터 유지, 사용자는 `새로고침` 으로 재시도.
- RLS 상 공용 곡(`user_id is null`)은 비로그인 사용자도 조회 가능하며, `내 곡` 필터는 로그인 사용자에게만 유효하다.
- 429/402 는 `fetchWithRetry` 가 한국어 토스트를 자동 노출한다.

### 4.3 데이터 계약

파생 쿼리 예시(`langView='chinese'`, `levelFilter='중급'`, `ownerFilter='public'`, `themes=['사랑']`, `cultures=['감정표현']`):

```sql
select s.*
from public.songs s
join public.song_culture_tags ct0
  on ct0.song_id = s.id and ct0.category = '감정표현' and ct0.keywords <> '{}'
where s.deleted_at is null
  and s.user_id is null
  and (s.language is null or s.language <> 'korean')
  and s.hsk_level in ('HSK 3','HSK 4')
  and s.theme in ('사랑')
order by s.created_at desc
offset :p*30 limit 30;
```

```sql
create table public.song_culture_tags (
  song_id uuid not null references public.songs(id) on delete cascade,
  category text not null,
  keywords text[] not null default '{}',
  primary key (song_id, category)
);
grant select on public.song_culture_tags to anon, authenticated;
grant insert, update, delete on public.song_culture_tags to authenticated;
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

- `themeFilter` / `cultureFilter` 변경 시 URL(`?themes=`, `?cultures=`)이 동일 렌더 사이클에 반영되고, 새로고침 후 두 축의 복원율 = **100 %**. 나머지 축은 URL 에 존재하지 않아야 한다(`rg -n "set\(\"page\"" src/pages/Songs.tsx` = **0**).
- 필터 6축(`langView` 포함) 중 하나라도 변경되면 `page === 0` 으로 리셋(단위 테스트 6건 통과).
- 패널 `초기화` 클릭 후 `levelFilter === tpFilter === ownerFilter === "all"` 이고 `themeFilter` / `cultureFilter` 는 **변경되지 않는다**. `전체 초기화` 클릭 후에는 5축 모두 초기값.
- `activeFilterCount` = `(level≠all) + (tp≠all) + (owner≠all) + themeFilter.length + cultureFilter.length`, 즐겨찾기는 포함하지 않는다(3축 활성 + 주제 2개 → 배지 5, chip 5개 + `전체 초기화` 1).
- 문화 태그 AND 의미 검증: 2개 카테고리 선택 시 반환 행 수 = 두 카테고리를 **모두** 가진 곡 수와 일치하고 중복 행 0건(`count` 와 `data.length` 불일치 0건).
- 필터 재조회 중 기존 카드가 유지되고(`songs.length` 감소 없음) `불러오는 중…` 칩이 표시되며, 언어 탭 개수는 `(…)` 로 대체된다.
- `SongStatsDrawer` 오픈 후 KPI 4개 렌더 ≤ **500 ms**, 5개 StatSection 전부 렌더 완료 p95 ≤ **2 s**(1,300곡 기준). `fetchAllPaged` 는 페이지당 1,000행으로 조회하며 총 조회 횟수 = `ceil(rows/1000)` 와 일치.
- 드로어 열림 1회당 데이터 로드 1회(`open` 또는 `새로고침` 이외의 리렌더에서 추가 요청 0건).
- `주제 분포` 행 클릭 → 드로어 닫힘 + 상위 `themeFilter` 에 해당 테마 추가(E2E 1건).
- `rg -n "bg-\[#|text-white" src/components/songs/SongStatsDrawer.tsx` 결과는 차트 데이터 색상 상수(`THEME_META`, `BUBBLE_COLORS`, 언어 색 2종)와 navy pill 텍스트에 한정되며, 그 외 신규 하드코드 색상 **0건**.
- 정렬 4종 각각에 대해 렌더 순서가 서버 반환 순서와 일치(스냅샷 4건).

### 5.2 Output Format
반환 순서:
1. Postgres 마이그레이션 SQL(`song_culture_tags` — 아직 없다면).
2. `src/pages/Songs.tsx` (필터 상태 · `applyFilters` · 필터 행 · 활성 태그 부분)
3. `src/components/songs/SongStatsDrawer.tsx`
4. 한국어 3줄 요약.

설명·사과·주석·추가 마크다운 헤더 등 부수 텍스트는 반환하지 않는다.
