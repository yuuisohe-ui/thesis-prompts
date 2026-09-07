# P3b · 상단바 · 곡 추가 흐름 재현 프롬프트

> **본 프롬프트는 P3 시리즈(P3a–P3q)의 2/17.** 
> **적용 대상**: `src/components/songs/archive/ArchiveTopbar.tsx`, `AddSongFab.tsx`, 그리고 이들과 연결되는 진입 다이얼로그의 트리거 계약(다이얼로그 본체 UI 는 P3c·P3d·P3e 범위).
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages: Identity, Instructions, Examples, Context.* OpenAI Platform Documentation. Retrieved July 12, 2026, from https://platform.openai.com/docs/guides/prompt-engineering
2. **Lovable. (n.d.).** *Prompting best practices.* Lovable Documentation. Retrieved July 12, 2026, from https://docs.lovable.dev/prompting/prompting-one — 본 부록에서 인용된 5개 실천 원칙 채택 근거.
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity (신원)

당신은 스티키 헤더·검색 UX·플로팅 진입 컨트롤을 설계하는 시니어 프론트엔드 엔지니어이자 IA(정보 설계자)입니다. React 18 · Tailwind · shadcn/ui `DropdownMenu` · Lucide 아이콘을 사용합니다.

## ② Instructions (지시)

### 2.1 산출물 (Lovable 실천 원칙 "Prompt by Component, Not Page")

- `src/components/songs/archive/ArchiveTopbar.tsx` — 스티키(top-0, z-40) 헤더 하나. 페이지 유일한 `<h1>` 포함.
- `src/components/songs/archive/AddSongFab.tsx` — 우측 하단 fixed(z-40) 플로팅 액션 버튼.
- 트리거 계약(콜백 prop): `onSubmit`, `onOpenAiGenerate`, `onOpenBulkCard`, `onOpenYoutubeSearch(lang: "chinese" | "korean" | "all")`, `onPickLocal(song)`, `onFocusInput`. **기본 진입(상단바 검색 인풋의 하단 CTA `YouTube에서 검색`, FAB 의 메인 `YouTube에서 추가` 알약)은 `"all"` 로 호출** → 중·한 이중 언어 결과가 동시에 노출된다. 언어별 특정 진입은 드롭다운 서브뷰(§2.2 3.lang)에서만 사용.

### 2.2 원자적 UI 규칙 (Lovable 실천 원칙 "Speak Atomic")

**ArchiveTopbar** = `<header sticky top-0 z-40 bg-white/85 backdrop-blur-md border-b>` 안에 `flex items-center gap-3 h-14` 로 정확히 3개 요소:

1. **H1**: `<h1 class="text-base md:text-[15px] font-bold text-[hsl(var(--dash-navy))] shrink-0 whitespace-nowrap">노래 아카이브</h1>`.
2. **검색 인풋 컨테이너**(`flex-1 min-w-0 relative`, `data-tour="songs-search"`):
   - 좌 아이콘: `<Search h-3.5 w-3.5 text-muted-foreground/60>`.
   - `<input placeholder="노래·아티스트·가사 검색…">`, Enter 로 `onSubmit`, `onFocus` 로 로컬 결과 재표시.
   - `url.trim()` 이 있을 때만 우측에 텍스트 버튼 `분석`(URL 감지 시) 또는 `검색`(검색어) 노출.
   - **로컬 결과 드롭다운**(절대 위치, `top-[calc(100%+4px)]`): 최대 320px 세로 스크롤, 각 항목 = `mqdefault.jpg` 썸네일(56×36) + 제목 1줄 clamp + 아티스트 1줄 clamp + HSK 배지. 하단 고정 `YouTube에서 검색` 버튼.
   - **로컬 검색 로딩**: 결과 없고 `localSearching === true` 일 때 `<Loader2 animate-spin> 곡 검색 중...` 미니 카드.
3. **[+ 곡 추가] 드롭다운**: `bg-[hsl(var(--dash-navy))] text-white` 버튼, 라벨 `곡 추가` + `<ChevronDown>`. shadcn `DropdownMenu` `align="end" w-64`, 상태 머신 `subview: "root" | "lang"`:
   - **root**: `AI로 나만의 곡 만들기` / `YouTube에서 추가` (기본 `"all"` — 즉 중·한 동시 검색) / `언어별 추가 >` (서브뷰 진입) / `<separator>` / `일괄 가져오기`.
   - **lang** (진입 옵션): `< 뒤로` + `어떤 언어의 곡을 추가할까요?` + 2-열 그리드 `🇨🇳 중국어` / `🇰🇷 한국어` — 이 두 항목만 `onOpenYoutubeSearch("chinese" | "korean")` 로 호출.
   - 메뉴 닫히면 150 ms 후 `subview → "root"` 리셋.

`scrolled` 상태(scrollY > 4)에 따라 하단 border 색상 전환(border-transparent ↔ border-border).

**AddSongFab** = `fixed bottom-6 right-6 z-40 flex flex-col items-end gap-2` 컨테이너:

- 메인 FAB: `w-14 h-14 rounded-full bg-[hsl(var(--dash-navy))] text-white shadow-lg`. 열림 시 45° 회전, 아이콘 `<Plus>` ↔ `<X>`.
- 열림 & `subview === "root"`: 위쪽으로 순차(30 ms stagger) 알약 버튼 3개 — `AI로 나만의 곡 만들기` / `YouTube에서 추가` (`"all"` 진입, 이중 언어 검색) / `노래 카드 일괄 생성`. 하단에 텍스트 링크 `언어별 추가 →` 로 lang 서브뷰 진입.
- 열림 & `subview === "lang"`: 240px 카드 팝오버 — `< 뒤로` + 2-열 그리드(🇨🇳 중국어 / 🇰🇷 한국어). 각 카드 클릭 시 `onOpenYoutubeSearch("chinese" | "korean")`.
- 바깥 클릭 시 닫힘(`document.addEventListener("click", ...)` cleanup 필수).

### 2.3 강제 제약

- semantic token 사용, 하드코드 색상 금지(정책 예외: 언어별 색 hint `bg-red-50 text-red-500` / `bg-emerald-50 text-emerald-600` 는 국가 시각 관례로 허용, `text-white` 는 dash-navy 배경 위에서만).
- ArchiveTopbar 는 페이지 유일한 `<h1>` 보유. 다른 곳에서 `<h1>` 추가 금지.
- 200 ms 디바운스 로컬 검색은 셸(P3a)의 훅이 소유. 본 컴포넌트는 결과 배열만 렌더.
- 모든 카피는 §3.1 표에 확정.

## ③ Examples

### 3.1 확정 카피 표 (Lovable 실천 원칙 "Design with Real Content")

| 위치 | 카피 |
|---|---|
| H1 | 노래 아카이브 |
| 검색 placeholder | 노래·아티스트·가사 검색… |
| 검색 버튼(URL) | 분석 |
| 검색 버튼(검색어) | 검색 |
| 로컬 로딩 | 곡 검색 중... |
| 드롭다운 하단 CTA | YouTube에서 검색 |
| FAB aria-label | 노래 추가 |
| DropdownMenu 항목 1 title | AI로 나만의 곡 만들기 |
| 항목 1 sub | 맞춤 학습 노래 생성 |
| 항목 2 title | YouTube에서 추가 |
| 항목 2 sub | 검색 또는 URL로 추가 |
| 항목 3 title | 일괄 가져오기 |
| 항목 3 sub | 노래 카드 일괄 생성 |
| 언어 서브뷰 헤더 | 어떤 언어의 곡을 추가할까요? |
| 언어 옵션 | 🇨🇳 중국어 / 🇰🇷 한국어 |
| 뒤로 링크 | 뒤로 |

### 3.2 파일·컴포넌트 트리 (Lovable 실천 원칙 "Use Prompt Patterns for Layouts")

```text
<ArchiveTopbar>  sticky top-0 z-40 h-14
  ├─ <h1>노래 아카이브</h1>
  ├─ <SearchBox flex-1>
  │     ├─ <input placeholder="노래·아티스트·가사 검색…">
  │     ├─ [분석 | 검색] 텍스트 버튼 (url.trim() && 조건부)
  │     └─ <LocalResultsDropdown>          (absolute)
  │           ├─ <ResultRow> × ≤8
  │           └─ <YouTubeSearchCTA>
  └─ <DropdownMenu align=end w-64>
        ├─ root: [AI 생성] [YouTube 추가 >] [── ] [일괄 가져오기]
        └─ lang: [< 뒤로] · [🇨🇳 중국어] [🇰🇷 한국어]

<AddSongFab>  fixed bottom-6 right-6 z-40
  ├─ Pills: [AI 생성] [YouTube 추가 >] [노래 카드 일괄 생성]
  ├─ LangPopover(240px): [< 뒤로] · [🇨🇳] [🇰🇷]
  └─ FAB (14×14, rotate-45 on open)
```

## ④ Context (배경)

### 4.1 프로젝트 맥락
검색·추가 두 축을 상단(스티키)과 하단(FAB) 두 위치에서 **동일한 콜백 인터페이스**로 노출한다. 데스크톱은 상단바, 모바일·롱스크롤 시나리오는 FAB 가 우선 진입 경로. 다이얼로그 본체(YouTubeSearchDialog / BulkSongCardDialog / YoutubeVideoGenerateDialog / CsvImportDialog)의 UI 는 각각 P3c(YouTube 검색)·P3d(CSV 일괄 등록)·P3e(AI 곡 생성) 프롬프트에서 정의하고, 본 프롬프트는 오직 트리거만 담당한다.

### 4.2 Lovable Cloud 후경 (Lovable 실천 원칙 "Build with Lovable Cloud in Mind")

- 인증 미완료 사용자가 `즐겨찾기` 하트 등 로그인 필요 액션을 시도하면 셸(P3a)이 `로그인 후 이용 가능합니다` 토스트를 노출한다(본 컴포넌트에서는 사전 차단하지 않음 — 열림 자체는 허용).
- 로컬 검색 결과는 셸이 200 ms 디바운스로 채워 넣는 프롭. 로딩/빈은 본 컴포넌트가 렌더한다.

### 4.3 데이터 계약
본 프롬프트는 데이터 스키마를 새로 정의하지 않는다(셸·P3e 참조). 콜백 시그니처만 확정:

```ts
interface ArchiveTopbarProps {
  url: string;
  onUrlChange(v: string): void;
  onSubmit(): void;
  onFocusInput(): void;
  isUrl: boolean;
  localSearching: boolean;
  showLocalResults: boolean;
  localSearchResults: Array<{ id: string; video_id: string; title: string; artist: string; hsk_level?: string }>;
  onPickLocal(song: unknown): void;
  onOpenAiGenerate(): void;
  onOpenBulkCard(): void;
  onOpenYoutubeSearch(lang: "chinese" | "korean" | "all"): void;
}
interface AddSongFabProps {
  onOpenAiGenerate?(): void;
  onOpenBulkCard(): void;
  onOpenYoutubeSearch(lang: "chinese" | "korean" | "all"): void;
}
```

## ⑤ Acceptance & Output

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6)

- Topbar 스티키 top=0, `z-40`, 스크롤 4 px 초과 시 border-bottom 표시(스크린샷 대조).
- H1 렌더 횟수 = **1**(페이지 전체 기준, `rg -n "<h1" src` 결과에서 본 파일 외 0 건 유지).
- DropdownMenu 서브뷰 전환은 클릭 1회로 root ↔ lang 이동. 닫힘 후 150 ms 내 root 로 리셋.
- FAB 바깥 클릭 시 200 ms 내 닫힘, `document` 이벤트 리스너가 언마운트 시 제거되는지(`removeEventListener` 호출) 검증.
- 로컬 결과 드롭다운 최대 높이 320 px, 항목 8개 초과 시 세로 스크롤(오버플로 hidden 없음).
- `rg -n "bg-\[#|bg-black" src/components/songs/archive/ArchiveTopbar.tsx src/components/songs/archive/AddSongFab.tsx` = **0**.
- 카피 표(§3.1) 문자열이 전부 그대로 렌더 — Playwright `getByText` 매칭 성공률 = **100 %**.
- **이중 언어 기본 진입**: 상단바 로컬 결과 드롭다운 `YouTube에서 검색` CTA, FAB 메인 `YouTube에서 추가` 알약, 드롭다운 root 의 `YouTube에서 추가` 항목 모두 `onOpenYoutubeSearch("all")` 로 호출된다(스파이 assertion 3건). `"chinese" | "korean"` 은 오직 lang 서브뷰의 두 카드에서만 호출된다.

### 5.2 Output Format
반환 순서:
1. `src/components/songs/archive/ArchiveTopbar.tsx`
2. `src/components/songs/archive/AddSongFab.tsx`
3. 한국어 3줄 요약.
