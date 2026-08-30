# P3h · 곡 카드 · 그리드 · 즐겨찾기 재현 프롬프트

> **본 프롬프트는 P3 시리즈(P3a–P3q)의 8/17.**
> **적용 대상**: `src/components/songs/SongCard.tsx`, 카드 그리드 컨테이너, 즐겨찾기 하트 토글, 그리고 `songs`·`song_favorites`·`hidden_public_items` 스키마의 완전한 DDL/GRANT/RLS.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* https://platform.openai.com/docs/guides/prompt-engineering (Retrieved July 12, 2026)
2. **Lovable. (n.d.).** *Prompting best practices.* https://docs.lovable.dev/prompting/prompting-one (Retrieved July 12, 2026) — 5개 실천 원칙 채택 근거.
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity

당신은 이중언어 학습 노래 카드와 그 컬렉션 그리드를 설계하는 시니어 프론트엔드 엔지니어이자 백엔드 RLS 설계자입니다. 대한민국 대학의 K-Chinese/K-Korean 교사·학습자를 대상으로 하며, React 18 + Vite 5 + Tailwind v3 + shadcn/ui + Supabase(JS v2) 스택을 사용합니다.

## ② Instructions

### 2.1 산출물 (Lovable 실천 원칙 "Prompt by Component, Not Page")

- `src/components/songs/SongCard.tsx` — 개별 곡 카드(뷰/편집 두 상태).
- `src/components/songs/EmbedDialog.tsx` — 공유 팝오버에서 여는 임베드 대화상자(본 프롬프트 범위 내에서 시그니처만 규정, 상세 UI 는 별도 P3i 참조).
- `src/components/songs/songTagUtils.ts` — `buildTagsFromWordList(word_list, lang)` 헬퍼(재분석 완료 후 `songs.tags` 를 파생).
- `src/components/songs/archive/SongGrid.tsx` — 반응형 그리드(1→2→3→4→5) + pending 곡 skeleton 카드 + 첫 카드에 `data-tour="song-card"` 앵커.
- Postgres 마이그레이션: `songs`, `song_favorites`, `hidden_public_items` + `set_owner_on_insert` 트리거 + `move_to_trash` RPC + `fork_public_item` RPC + GRANT + RLS.

### 2.2 원자적 UI 규칙 (Lovable 실천 원칙 "Speak Atomic")

**SongCard 뷰 상태** — 컨테이너: `bg-card border border-border rounded-[14px] p-4 flex flex-col gap-2.5 h-full`, 그리고 **언어별 4 px 좌측 색조 스트립**:

- `song.language === "bilingual"` → `border-l-[4px] border-l-purple-500`
- `song.language === "korean"` → `border-l-[4px] border-l-[#2557a7]`
- 그 외(중국어) → `border-l-[4px] border-l-[#d4580a]`

카드 내부는 **정확히 다음 7단**으로 구성되며, **커버 썸네일 `<img>` · 호버 오버레이는 존재하지 않는다**(하드 금지):

**(1) 상단 행** `flex items-center justify-between`
- 좌측 배지 그룹 `flex items-center gap-1.5`:
  - **언어 배지**(pill, 국기 SVG 좌측 배치, 국기 이미지 소스는 `https://cdnjs.cloudflare.com/ajax/libs/flag-icon-css/6.6.6/flags/4x3/{cn|kr}.svg`):
    - bilingual: 자주(`bg-purple-500/15 text-purple-700 border border-purple-500/30`) + `KR|CN` 이중 국기.
    - 중국어: `bg-[#fff0e8] text-[#d4580a]` + CN 국기 + "CN".
    - 한국어: `bg-[#eaf1fb] text-[#2557a7]` + KR 국기 + "KR".
  - **소유권 배지** — `isMine = currentUserId && song.owner_id === currentUserId`. `isMine ? "내 곡" : "공용"`. `isMine === true` → `bg-[hsl(var(--dash-blue-bg))] text-[hsl(var(--dash-blue))]`, 아니면 `bg-muted text-muted-foreground`.
- 우측 아이콘 그룹 `flex gap-1.5 items-center` — 모든 아이콘은 `h-3.5 w-3.5`, 배경 없음, 순수 텍스트 색 전환:
  - `Heart` 즐겨찾기: **`currentUserId && onToggleFavorite` 가 모두 truthy 일 때만 렌더**. `isFavorited` 시 `fill-red-500 text-red-500`, 아니면 `text-muted-foreground/50 hover:text-red-400`. `title={isFavorited ? "즐겨찾기 해제" : "즐겨찾기"}`.
  - `Pencil` 편집 — 클릭 시 뷰→편집 상태 전환.
  - `RefreshCw` 재분석 — `Popover` 트리거. 로딩 중이면 `Loader2 animate-spin` 로 치환, `disabled`.
  - `Trash2` 삭제 — `AlertDialog` 트리거.
- **권한 표기 규칙(중요)**: 편집/재분석/삭제/하트 4 아이콘은 **로그인 여부와 상관없이 모든 사용자에게 렌더**된다. 비소유자가 실제로 클릭할 경우 RLS 가 후단에서 거절하고 카드가 각각의 실패 토스트를 띄운다. "소유자만 보인다" 는 서술은 금지.

**(2) 제목 행** `flex items-center gap-1.5`
- 🎵 이모지(`text-sm shrink-0`) + `displayTitle`(`text-[13.5px] font-bold text-foreground truncate`).
- **displayTitle 파생 규칙**(순서대로):
  1. `!song.title` → `"제목 없음"`.
  2. `song.title.includes("_")` → 그대로 반환.
  3. `isChinese && song.title.split(" / ").length === 2` → `parts[0] + "_" + parts[1]`.
  4. 그 외 → 원본 반환.

**(3) 메타 행** `flex items-center gap-1.5 flex-wrap`
- 아티스트 텍스트: `song.artist` 존재 시 `text-[11px] text-muted-foreground truncate max-w-[120px]`, `song._year` 있으면 뒤에 ` (YYYY)` 부착.
- **난이도 배지** — `hskToLevel(song.hsk_level)` 로 계산, TOPIK 접두는 표시 시 유지. 구현식:
  ```ts
  const isTopik = hsk?.toUpperCase().includes("TOPIK");
  const n = parseInt(hsk.replace(/\D/g, ""));
  const level =
    isTopik ? (n <= 2 ? "초급" : n <= 4 ? "중급" : "고급")
            : (n <= 2 ? "초급" : n <= 4 ? "중급" : n <= 6 ? "고급" : "심화");
  ```
  배지 색조: 초급 `dash-teal`, 중급 `dash-blue`, 고급 `dash-gold`, 심화 `destructive/10 + destructive`. 텍스트: TOPIK 이면 `"TOPIK "` 접두 + 등급명.
- **teaching_point 배지** — `song.teaching_point ∈ {"어휘","문법","문화","발음"}` 존재 시 `bg-muted text-muted-foreground border border-border`.
- **theme(주제) 배지** — `song.theme` 존재 시 `bg-[hsl(var(--dash-pink-bg))] text-[hsl(var(--dash-pink))]`. **이 배지가 없으면 검증 실패**.

**(4) 분리선** `h-px bg-border`

**(5) 단어 태그 행**
- `shownTags = (song.tags || []).slice(0, 3)`, 각 태그 `bg-[hsl(var(--dash-purple-bg))] text-[hsl(var(--dash-purple))]`.
- `song.tags.length > 3` → 말미에 `+N` (`text-[10.5px] text-muted-foreground`).
- 태그 배열이 비었으면 "단어 정보 없음" 무채색 문구.

**(6) AI 추정 경고**(선택, `song.lyrics_source ∈ {"ai_search","none"} || (!song.lyrics_source && song.has_subtitles === false)` 일 때만)
- ✐ 원 배지 + "AI 추정" 텍스트 + hover tooltip: "AI가 추정한 가사입니다. 정확도를 확인해주세요."

**(7) 액션 행** `flex gap-[7px] mt-auto pt-2` — **상시 렌더, hover 오버레이 아님**:
- `[👁 분석 보기]` 버튼 `flex-1` 아웃라인 스타일, 클릭 → `onViewAnalysis(song)`.
- `[🔗 공유]` 버튼 `flex-[1.5]` `bg-[#2e3d6b] text-white`. Popover 안에 두 항목: "🔗 링크 복사"(`navigator.clipboard.writeText(${origin}/shared/${share_token || id})`), "📺 퍼가기 (임베드)"(`EmbedDialog` 열기).

**SongCard 편집 상태** — 뷰 상태를 통째로 대체:
- `Input`: 제목, 아티스트.
- `Select`: `HSK_LEVELS = ["HSK 1"..."HSK 9"]` 9 항목.
- `Select`: `TEACHING_POINTS = ["어휘","문법","문화","발음"]` 4 항목.
- `[저장]` → `supabase.from("songs").update({ title, artist, hsk_level, teaching_point }).eq("id", song.id)` → 성공 시 `onSongUpdated?.()` 호출 및 편집 종료.
- `[취소]` → 로컬 state 폐기 후 뷰 상태 복귀.

**재분석 Popover 로직**
- 두 버튼: "🇨🇳 중국어 배우기", "🇰🇷 한국어 배우기".
- 선택 시 `learn_language ∈ {"chinese","korean"}` 로 `supabase.functions.invoke("analyze-song", body)` 호출.
- `body`: AI 생성 곡(`source === "ai_generated"` 또는 `!youtube_url && audio_url` 존재) → `{ song_id, custom_lyrics: lyrics_raw, language, learn_language, force: true }`. 그 외 → `{ youtube_url, force: true, language, learn_language }`.
- 성공 후 자막이 있으면 `reanalyze-wordlist` 를 호출하여 `word_list` 를 보강하고 `song_analyses.word_list` 를 업데이트, `buildTagsFromWordList(word_list, lang)` 결과가 비어있지 않으면 `songs.tags` 도 함께 업데이트.
- 완료 후 `onSongUpdated?.()`. 실패 시 한국어 토스트.

**삭제 로직 — 하드 삭제 금지**
- `AlertDialog` 확인 시 반드시 `supabase.rpc("move_to_trash", { _table: "songs", _id: song.id })` 호출(소프트 삭제, 7일 후 자동 정리).
- 성공 토스트: "휴지통으로 이동되었습니다 · 7일 후 자동 삭제됩니다."

**SongGrid**
- `grid gap-3.5 grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 2xl:grid-cols-5`.
- 배열: `pendingAnalyses` skeleton(좌측 컬러 스트립 + `Loader2 animate-spin` + "분석 중..." + 언어 배지) → `displaySongs.map(<SongCard>)`.
- 첫 실 카드 노드에 `data-tour="song-card"`.

### 2.3 강제 제약(하드 금지)

- **커버 썸네일 금지**: `SongCard.tsx` 내에 `img.youtube.com/vi/*` 또는 유사 썸네일 `<img>` 태그가 존재해서는 안 된다. 그리드는 텍스트 카드만.
- **호버 오버레이 금지**: `bg-black/40` 계열의 전체 카드 오버레이 요소 없음. `분석 보기` 는 상시 하단 버튼.
- **하드 딜리트 금지**: `.from("songs").delete()` 사용 금지, 반드시 `move_to_trash` RPC.
- **"소유자만 편집/삭제 버튼 표시" 금지**: 모든 로그인 사용자에게 아이콘 렌더, RLS 가 후단에서 거절.
- semantic token 우선. `text-white` 는 공유 버튼(`bg-[#2e3d6b]`)과 언어 배지 내부에서만 예외 허용. 언어 색 두 개(`#2557a7` · `#d4580a`)와 공유 버튼 색(`#2e3d6b`)만 브랜드 예외.
- 카드에서 직접 리스트 fetch 금지 — 데이터는 셸(P3a) 에서 주입.
- 곡 제목 규약: `OriginalTitle_TranslatedTitle` (언더스코어 1개). 마이그레이션 검증 쿼리 통과.
- `song.owner_id IS NULL` 인 행은 "공용"으로 취급(mem://features/public-vs-personal-content 모델과 동일).

## ③ Examples

### 3.1 확정 카피 표 (Lovable 실천 원칙 "Design with Real Content")

| 위치 | 카피 |
|---|---|
| 소유권 배지(내 곡) | `내 곡` |
| 소유권 배지(공용) | `공용` |
| 언어 배지 | `CN` / `KR` / `KR\|CN` |
| 하트 title | `즐겨찾기` / `즐겨찾기 해제` |
| 편집 아이콘 title | `편집` |
| 재분석 아이콘 title | `재분석` |
| 재분석 팝오버 헤더 | `학습 언어 선택` |
| 재분석 팝오버 옵션 | `중국어 배우기` / `한국어 배우기` |
| 삭제 아이콘 title | `삭제` |
| 삭제 AlertDialog 제목 | `이 곡을 삭제하시겠습니까?` |
| 삭제 설명 | `되돌릴 수 없습니다. 휴지통에서 7일간 복구 가능합니다.` |
| 삭제 취소/확정 | `취소` / `삭제` |
| 삭제 성공 토스트 | `휴지통으로 이동되었습니다` · `7일 후 자동 삭제됩니다.` |
| 편집 저장 성공 | `저장 완료` |
| 편집 저장 실패 | `저장 실패` |
| 재분석 성공 | `재분석 완료!` |
| 재분석 자막 없음 | `가사를 찾을 수 없음` |
| 재분석 실패 | `재분석 실패` |
| 공유 링크 복사 성공 | `링크가 복사되었습니다.` |
| 공유 팝오버 항목 | `🔗 링크 복사` / `📺 퍼가기 (임베드)` |
| 하단 CTA 좌 | `👁 분석 보기` |
| 하단 CTA 우 | `🔗 공유` |
| 태그 없음 문구 | `단어 정보 없음` |
| AI 추정 라벨 | `AI 추정` |
| AI 추정 tooltip | `AI가 추정한 가사입니다. 정확도를 확인해주세요.` |
| 폴백 제목 | `제목 없음` |
| 429 토스트 | `요청이 많습니다. 잠시 후 다시 시도해주세요.` |

### 3.2 컴포넌트 트리 (Lovable 실천 원칙 "Use Prompt Patterns for Layouts")

```text
<SongGrid>  grid cols-{1|2|3|4|5}
  ├─ PendingSkeleton × N            (좌측 컬러 스트립 + Loader2 + "분석 중...")
  └─ <SongCard>                     (커버 이미지 없음, hover 오버레이 없음)
        ├─ Top Row
        │   ├─ LanguageBadge (KR|CN 국기, bilingual = 자주 이중)
        │   ├─ OwnershipBadge ["내 곡" | "공용"]
        │   └─ IconGroup [♥ ✏ ⟳ 🗑]  ← 모든 로그인 사용자에게 렌더
        ├─ TitleRow (🎵 + displayTitle)
        ├─ MetaRow  [Artist · [Level] [TeachingPoint] [Theme]]
        ├─ Divider
        ├─ TagRow × ≤3  (+N)
        ├─ (선택) AiEstimatedWarning
        └─ ActionRow  [👁 분석 보기] [🔗 공유 → Popover]
              ├─ 🔗 링크 복사
              └─ 📺 퍼가기 (임베드) → EmbedDialog
```

### 3.3 JSX 스켈레톤(뷰 상태 · 필수 골자)

```tsx
<div className={`bg-card border border-border rounded-[14px] p-4 flex flex-col gap-2.5 h-full ${langBar}`}>
  <div className="flex items-center justify-between">
    <div className="flex items-center gap-1.5">
      <LanguageBadge language={song.language} />
      <span className={`text-[9.5px] font-bold px-1.5 py-0.5 rounded ${isMine ? "bg-[hsl(var(--dash-blue-bg))] text-[hsl(var(--dash-blue))]" : "bg-muted text-muted-foreground"}`}>
        {isMine ? "내 곡" : "공용"}
      </span>
    </div>
    <div className="flex gap-1.5 items-center">
      {currentUserId && onToggleFavorite && <HeartToggle />}
      <button title="편집" onClick={handleEdit}><Pencil className="h-3.5 w-3.5" /></button>
      <Popover /* 재분석 학습 언어 선택 */ />
      <button title="삭제" onClick={() => setDeleteOpen(true)}><Trash2 className="h-3.5 w-3.5" /></button>
    </div>
  </div>
  <div className="flex items-center gap-1.5">
    <span className="text-sm shrink-0">🎵</span>
    <span className="text-[13.5px] font-bold truncate">{displayTitle}</span>
  </div>
  <div className="flex items-center gap-1.5 flex-wrap">
    {song.artist && <ArtistText />}
    <LevelBadge />
    {song.teaching_point && <TeachingPointBadge />}
    {song.theme && <ThemeBadge />}
  </div>
  <div className="h-px bg-border" />
  <TagRow tags={song.tags} />
  {isUncertain && <AiEstimatedWarning />}
  <div className="flex gap-[7px] mt-auto pt-2">
    <button className="flex-1 …" onClick={() => onViewAnalysis(song)}>👁 분석 보기</button>
    <Popover>{/* 링크 복사 · 임베드 */}</Popover>
  </div>
</div>
```

## ④ Context

### 4.1 프로젝트 맥락
카드는 아카이브의 최소 단위 콘텐츠이면서, 강의안·수업홈·검색결과 등 다른 페이지(`SongPickerDialog`, `CommunityTab`, `MyFavoriteSongsModule` 등)에서도 재사용된다. 따라서 카드는 stateless-ish(자체 편집 상태만 소유, 리스트/필터/페이징에 무관심)해야 한다.

### 4.2 Lovable Cloud 후경 (Lovable 실천 원칙 "Build with Lovable Cloud in Mind")

- 로딩/빈/에러: 셸(P3a)이 제어. 카드는 4-상태 중 "성공" 만 담당.
- 카드 액션 실패 시 개별 한국어 토스트, 카드 상태 롤백.
- 하트 토글은 낙관적 반영 후 실패 시 롤백.
- 소유권 판정은 클라이언트 편의 표시이며, 실제 쓰기 권한은 RLS 가 결정.

### 4.3 데이터 계약 (완전 DDL — 순서: CREATE TABLE → GRANT → ENABLE RLS → CREATE POLICY)

```sql
-- songs -----------------------------------------------------------------
create table public.songs (
  id uuid primary key default gen_random_uuid(),
  owner_id uuid references auth.users(id) on delete cascade,       -- null = 공용
  title text not null,                                              -- OriginalTitle_TranslatedTitle
  artist text not null,
  language text not null check (language in ('chinese','korean','bilingual')),
  video_id text,
  youtube_url text,
  audio_url text,
  cover_url text,
  hsk_level text,                                                   -- 'HSK 1'..'HSK 9' or 'TOPIK 1'..'TOPIK 6'
  teaching_point text check (teaching_point in ('어휘','문법','문화','발음')),
  theme text,                                                       -- 12 THEME_OPTIONS
  tags text[] default '{}',
  lyrics_raw text,
  lyrics_source text,                                               -- 'ai_search' | 'none' | 'manual' | ...
  has_subtitles boolean,
  share_token text unique,
  source text,                                                      -- 'ai_generated' 등
  deleted_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
grant select on public.songs to anon, authenticated;
grant insert, update, delete on public.songs to authenticated;
grant all on public.songs to service_role;

-- owner_id 자동 채움
create or replace function public.set_owner_on_insert()
returns trigger language plpgsql security definer set search_path = public as $$
begin
  if auth.uid() is not null and new.owner_id is null then
    new.owner_id := auth.uid();
  end if;
  return new;
end $$;
create trigger songs_set_owner before insert on public.songs
for each row execute function public.set_owner_on_insert();

alter table public.songs enable row level security;
create policy "public or owned readable" on public.songs
  for select using (deleted_at is null and (owner_id is null or owner_id = auth.uid()));
create policy "owner or admin inserts" on public.songs
  for insert with check (owner_id = auth.uid() or owner_id is null);
create policy "owner updates" on public.songs
  for update using (owner_id = auth.uid()) with check (owner_id = auth.uid());
create policy "owner deletes (soft only)" on public.songs
  for delete using (owner_id = auth.uid());

-- song_favorites --------------------------------------------------------
create table public.song_favorites (
  user_id uuid not null references auth.users(id) on delete cascade,
  song_id uuid not null references public.songs(id) on delete cascade,
  created_at timestamptz not null default now(),
  primary key (user_id, song_id)
);
grant select, insert, delete on public.song_favorites to authenticated;
grant all on public.song_favorites to service_role;
alter table public.song_favorites enable row level security;
create policy "own favorites only" on public.song_favorites
  for all using (user_id = auth.uid()) with check (user_id = auth.uid());

-- hidden_public_items ---------------------------------------------------
create table public.hidden_public_items (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  table_name text not null,
  item_id uuid not null,
  created_at timestamptz not null default now(),
  unique (user_id, table_name, item_id)
);
grant select, insert, delete on public.hidden_public_items to authenticated;
grant all on public.hidden_public_items to service_role;
alter table public.hidden_public_items enable row level security;
create policy "own hides only" on public.hidden_public_items
  for all using (user_id = auth.uid()) with check (user_id = auth.uid());

-- move_to_trash RPC (soft delete)
create or replace function public.move_to_trash(_table text, _id uuid)
returns void language plpgsql security definer set search_path = public as $$
begin
  execute format('update public.%I set deleted_at = now() where id = $1 and owner_id = auth.uid()', _table)
  using _id;
end $$;
grant execute on function public.move_to_trash(text, uuid) to authenticated;

-- 규약 검증: 언더스코어 1 초과 곡 = 0
-- select count(*) from public.songs where deleted_at is null and array_length(string_to_array(title,'_'),1) > 2;
```

## ⑤ Acceptance & Output

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6 · 정량 임계값)

- `rg -n "img\\.youtube\\.com/vi|maxresdefault|mqdefault" src/components/songs/SongCard.tsx` 매칭 수 = **0**(커버 썸네일 하드 금지).
- `rg -n "bg-black/40|opacity-0 group-hover" src/components/songs/SongCard.tsx` 매칭 수 = **0**(호버 오버레이 하드 금지).
- `rg -n "from\\(\"songs\"\\)\\.delete\\(\\)|\\.from\\('songs'\\)\\.delete\\(\\)" src/components/songs/SongCard.tsx` 매칭 수 = **0**(하드 삭제 금지).
- `rg -n "내 곡|공용" src/components/songs/SongCard.tsx` 매칭 수 ≥ **2**(소유권 배지 두 카피 모두 존재).
- `rg -n "song\\.theme" src/components/songs/SongCard.tsx` 매칭 수 ≥ **1**(주제 배지 조건 렌더 확인).
- `rg -n "move_to_trash" src/components/songs/SongCard.tsx` 매칭 수 = **1**(소프트 삭제 RPC 호출 존재).
- `rg -n "reanalyze-wordlist" src/components/songs/SongCard.tsx` 매칭 수 = **1**(재분석 후 태그 보강 존재).
- 하단 액션 행 두 버튼(`👁 분석 보기`, `🔗 공유`) 은 hover 여부와 무관하게 **`getBoundingClientRect().height > 0`** 이어야 한다(Playwright 검증).
- 사용자 A 가 개인으로 만든 곡(`owner_id = A`)을 사용자 B 로 로그인해 조회 시 SELECT 반환 = **0** 행.
- 소유권 배지 정확도: `song.owner_id === currentUserId` 인 행에 대해 카드 상단 텍스트가 정확히 "내 곡" 인 비율 = **100 %**, 그 외 = "공용".
- 편집 저장 왕복 p95 ≤ **400 ms**(로컬 Lovable Cloud).
- 하트 토글 낙관적 반영 ≤ **50 ms**, 실패 시 500 ms 내 롤백 + 토스트.
- 그리드 반응형 브레이크포인트: 1(sm 미만) → 2(sm) → 3(lg) → 4(xl) → 5(2xl). 각 지점 스크린샷 대조 통과.
- Lighthouse 접근성 ≥ **95**(모든 인터랙티브에 `title`/`aria-label`), CLS ≤ **0.05**.
- `song.tags.length > 3` 인 경우 정확히 `+N` 접미(예: `+2`) 가 렌더된다.
- 규약 검증 SQL(언더스코어 1 초과 곡) 결과 = **0**.

### 5.2 Output Format
반환 순서(설명·사과·주석·마크다운 헤더 금지):
1. Postgres 마이그레이션 SQL (`songs`, `song_favorites`, `hidden_public_items` + `set_owner_on_insert` 트리거 + `move_to_trash` RPC + GRANT + RLS + 검증 쿼리).
2. `src/components/songs/songTagUtils.ts`
3. `src/components/songs/SongCard.tsx`
4. `src/components/songs/archive/SongGrid.tsx`
5. 한국어 3줄 요약.
