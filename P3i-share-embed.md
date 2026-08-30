# P3i · 노래 공유 · 임베드 재현 프롬프트

> **본 프롬프트는 P3 시리즈(P3a–P3q)의 9/17.**
> **적용 대상**: `src/components/songs/EmbedDialog.tsx`, `src/pages/SharedSong.tsx`, `src/pages/EmbedSong.tsx`, `src/hooks/useTrackShareVisit.ts` — 즉 카드의 「🔗 공유」 버튼을 눌러 나타나는 팝오버 메뉴 중 **「📺 퍼가기 (임베드)」를 선택한 뒤에 열리는 다이얼로그**와, 그 다이얼로그가 생성하는 공유/임베드 링크가 실제로 열리는 두 개의 익명 라우트(`/shared/:token`, `/embed/:token`), 그리고 방문 기록 훅을 재현한다.
> **버튼과 팝오버 자체(=엔트리)는 P3h 에서 이미 정의**되었으며, 본 문서는 그 엔트리에 연결되는 **구현부**만 다룬다.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* Retrieved July 12, 2026.
2. **Lovable. (n.d.).** *Prompting best practices.* Retrieved July 12, 2026 — 5개 실천 원칙 채택 근거.
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity (신원)

당신은 익명 사용자에게 학습 콘텐츠 미리보기와 임베드 카드를 안전하게 배포하는 시니어 프론트엔드 엔지니어이다. Lovable Cloud(Supabase) 의 `share_token`(UUID) 컬럼을 통해 링크 소유자만 발급/회수할 수 있고, 링크를 가진 사람은 로그인 없이 열람만 가능한 구조를 유지한다.

## ② Instructions

### 2.1 산출물

- `src/components/songs/EmbedDialog.tsx` — 「📺 퍼가기 (임베드)」 선택 시 열리는 다이얼로그. **탭 없음**, iframe 코드 한 벌 + 복사 버튼 하나.
- `src/pages/SharedSong.tsx` — 라우트 `/shared/:token`, 로그인 없이 진입 가능한 곡 상세(분석 탭 포함).
- `src/pages/EmbedSong.tsx` — 라우트 `/embed/:token`, iframe 삽입용 초경량 카드.
- `src/hooks/useTrackShareVisit.ts` — 방문 기록 훅(로그인 사용자가 방문한 경우에 한해 `share_link_visits` 에 insert 하고 언마운트 시 체류시간 update).

### 2.2 「🔗 공유」 팝오버 두 항목의 실제 동작 (엔트리는 P3h)

| 메뉴 | 동작 |
|---|---|
| 🔗 링크 복사 | 다이얼로그를 **열지 않는다**. `${origin}/shared/${song.share_token \|\| song.id}` 를 `navigator.clipboard.writeText` 로 즉시 복사하고 `링크가 복사되었습니다.` 토스트만 띄운 뒤 팝오버를 닫는다. |
| 📺 퍼가기 (임베드) | 팝오버를 닫고 `EmbedDialog` 를 연다. |

### 2.3 EmbedDialog 원자적 UI (탭 없음, 사이즈 프리셋 없음)

- 컨테이너: `<DialogContent className="max-w-lg">`.
- 헤더:
  - `<DialogTitle>`: `📺 퍼가기 (임베드 코드)`.
  - `<DialogDescription>`: `이 코드를 복사해 다른 사이트나 블로그에 붙여넣으면 이 곡 카드를 그대로 임베드할 수 있습니다.`
- 본문: **단일 `<textarea readOnly rows={5}>`**, `font-mono text-xs`, 포커스 시 `select()`. 값은 아래 iframe 문자열 하나로 고정한다.

  ```html
  <iframe src="{origin}/embed/{token}" width="420" height="520" frameborder="0" allowfullscreen title="{safeTitle} — Sino Song Learn" style="border:0;border-radius:16px;"></iframe>
  ```

  - `safeTitle` = `(song.title || "Song").replace(/"/g, "'")` — 따옴표 이스케이프.
  - **크기 프리셋 chip 없음**. 폭/높이는 420×520 하드코드.
- 푸터(같은 줄):
  - 좌: `Powered by Sino Song Learn` (마이크로 카피, `text-[11px]`).
  - 우: `<Button size="sm">` — 기본 상태 `📋 코드 복사`, 클릭 성공 후 2초간 `✓ 복사됨` 로 스왑(아이콘 `Check`). 배경 `bg-[hsl(var(--dash-navy))] hover:bg-[#2e3d6b] text-white`.
- **공개 스위치 없음.** `share_token` 회수/재발급 UI 는 본 다이얼로그에 두지 않는다(추후 별도 관리 화면).

### 2.4 SharedSong 페이지 (`/shared/:token`)

- 익명 접근 가능. `songs` 를 `share_token = :token` 으로 조회, 실패 시 `공유된 노래를 찾을 수 없습니다.` 로 대체.
- 성공 시 `song_analyses` 에서 최신 1건을 함께 로드.
- 레이아웃: `AppLayout` 미사용. 자체 컨테이너 + 하단 `<Footer />`.
- 상단 헤더: 곡 커버(`video_id` 존재 시 `img.youtube.com/vi/{id}/maxresdefault.jpg`) + 제목/아티스트 + `hsk_level` `Badge`.
- 본문: `Tabs` 로 다음 서브뷰를 그대로 렌더한다(편집·저장·삭제·공유 트리거는 전부 숨김).
  - `VideoLyricsTab` 또는 `AudioLyricsTab`(오디오 소스 유형에 따라 자동 분기), `LyricsTab`, `WordListTab`, `PatternsTab`, `ReadingTab`, `ExploreTab`.
  - 각 하위 탭 컴포넌트에는 `readOnly` 의미가 이미 내장되어 있으므로 추가 prop 없이 재사용한다.
- 로딩 스켈레톤: 중앙 `<Loader2 className="animate-spin" />`.
- 방문 기록: `useTrackShareVisit({ linkType: "song", token, ownerId: song?.user_id, resourceTitle: song?.title, enabled: !!song })`.

### 2.5 EmbedSong 페이지 (`/embed/:token`)

- 사이드바/헤더 없음. `bg-background p-3` 컨테이너 하나에 카드 하나.
- 조회 순서: `share_token = :token` → 실패 시 `id = :token` 폴백(즉 소유자가 링크에서 UUID 대신 곡 id 를 붙여도 열리도록 방어).
- 카드 레이아웃(폭 `max-w-[420px]`):
  1. **커버 링크**: `video_id` 있으면 `img.youtube.com/vi/{id}/hqdefault.jpg`, 클릭 시 `${origin}/shared/${token}` 를 `target="_blank"` 로 새 창에서 연다. hover 시 반투명 오버레이 + 삼각 재생 아이콘.
  2. **본문**: 언어 국기(`cn.svg` / `kr.svg` from `flag-icon-css`) + 제목/아티스트, 그 아래 배지 라인 — `HSK/TOPIK 레벨` · `초급/중급/고급/심화`(레벨 매핑) · `teaching_point` · `theme`. 각 배지 카피/색상은 `hskColor` 및 `dash-*` semantic token 사용.
  3. **태그 라인**: `song_analyses.word_list` 에서 `buildTagsFromWordList(list, "ko"|"zh")` 로 최대 6개.
  4. **푸터 링크**: `${origin}/shared/${token}` 로 이동하는 짙은 남색 바 — 좌측 `⚡ Sino Song Learn`, 우측 `자세히 보기 →`.
- 방문 기록: `useTrackShareVisit({ linkType: "embed", token, ownerId: song?.user_id, resourceTitle: song?.title, enabled: !!song })`.
- Not found: `<Music />` 아이콘 + `노래를 찾을 수 없습니다 / Song not found`.

### 2.6 useTrackShareVisit 훅

```ts
type LinkType = "song" | "course" | "embed";
interface Options {
  linkType: LinkType;
  token: string | undefined | null;
  ownerId?: string | null;
  resourceTitle?: string | null;
  enabled?: boolean;   // default true
}
```

동작:
- **로그인 사용자에 한해** insert (`share_link_visits`).  익명 방문은 저장하지 않는다 — 인증 없이 insert 하려면 `authenticated` role 이 없어 RLS 상 거부되기 때문. 익명 트래킹이 필요해지면 별도 마이그레이션(P6)에서 다룬다.
- 마운트 시 `insert({ link_type, token, owner_id, resource_title, visitor_id: user.id })` → `id` 반환값을 ref 에 저장.
- 언마운트/`pagehide` 시 `Date.now() - startedAt` 을 초 단위로 반올림해 `duration_seconds` 를 update.
- `token` 변경 시 이전 세션은 종료 update 후 새 insert.

### 2.7 강제 제약

- `/shared/:token` · `/embed/:token` 는 `App.tsx` 라우트에서 **`RequireAuth` 를 통과시키지 않는다**.
- `EmbedSong.tsx` 에서 `AppLayout` · `Sidebar` 를 import 하지 않는다.
- 색상은 semantic token(`dash-navy`, `dash-teal`, `dash-blue`, `dash-gold`, `dash-pink`, `dash-purple`) 만 사용. `bg-[#...]`, `text-white`, `bg-black` 등 하드코드 금지(다이얼로그 CTA 는 예외적으로 `bg-[hsl(var(--dash-navy))]` + hover `#2e3d6b` 조합만 허용).
- iframe 코드의 사이즈 값(420×520)은 **하드코드**. 사이즈 프리셋/공개 스위치/탭을 임의로 추가하지 말 것.
- 「🔗 링크 복사」는 절대 다이얼로그를 열지 않는다.

## ③ Examples

### 3.1 카피 표

| 위치 | 카피 |
|---|---|
| 팝오버 항목 1 | `🔗 링크 복사` |
| 링크 복사 토스트 | `링크가 복사되었습니다.` |
| 팝오버 항목 2 | `📺 퍼가기 (임베드)` |
| Dialog 제목 | `📺 퍼가기 (임베드 코드)` |
| Dialog 설명 | `이 코드를 복사해 다른 사이트나 블로그에 붙여넣으면 이 곡 카드를 그대로 임베드할 수 있습니다.` |
| 코드 복사 버튼 (기본) | `📋 코드 복사` |
| 코드 복사 버튼 (성공) | `✓ 복사됨` |
| Dialog 푸터 마이크로 카피 | `Powered by Sino Song Learn` |
| SharedSong 오류 | `공유된 노래를 찾을 수 없습니다.` |
| EmbedSong 오류 | `노래를 찾을 수 없습니다 / Song not found` |
| Embed 푸터 좌 | `⚡ Sino Song Learn` |
| Embed 푸터 우 | `자세히 보기 →` |

### 3.2 엔트리 → 구현부 흐름

```text
SongCard (P3h)
  └─ 🔗 공유 (Popover)
       ├─ 🔗 링크 복사   → navigator.clipboard.writeText(`${origin}/shared/${token}`)
       │                    + toast("링크가 복사되었습니다.")
       └─ 📺 퍼가기 (임베드) → <EmbedDialog>  (본 문서)
                                    └─ readonly <textarea> (iframe 420×520)
                                         │
                                         ▼ 붙여넣기
App.tsx
  /shared/:token   → <SharedSong>   (익명, 분석 탭 6개)
  /embed/:token    → <EmbedSong>    (익명, 미니 카드 + 커버·배지·태그)
```

## ④ Context

### 4.1 프로젝트 맥락
교사가 학생 단체채팅에는 링크 한 줄, 블로그/노션에는 iframe 한 줄로 곡 카드를 그대로 배포할 수 있게 하는 배포 채널. 학생은 회원가입 없이 미리보기만 가능하고, 편집/저장/즐겨찾기 등 상호작용은 전혀 노출되지 않는다.

### 4.2 Lovable Cloud 후경

`songs.share_token uuid unique default gen_random_uuid()` 컬럼이 이미 존재한다고 가정.

- SharedSong/EmbedSong 은 익명 SELECT 를 필요로 하므로 별도 정책 필요:

  ```sql
  create policy "anon reads songs by share_token"
    on public.songs for select
    to anon, authenticated
    using (share_token is not null);
  ```

  (실제로는 `share_token` 을 URL 로 알아야 접근 가능하므로 이 정책만으로 노출 위험이 없다. 소유자가 링크를 회수하려면 `share_token` 을 새 UUID 로 update.)

- 방문 로그 테이블:

  ```sql
  create table public.share_link_visits (
    id uuid primary key default gen_random_uuid(),
    link_type text not null check (link_type in ('song','course','embed')),
    token text not null,
    owner_id uuid references auth.users(id),
    resource_title text,
    visitor_id uuid not null references auth.users(id),
    duration_seconds int,
    visited_at timestamptz not null default now()
  );
  grant insert, update on public.share_link_visits to authenticated;
  grant select on public.share_link_visits to authenticated;
  grant all on public.share_link_visits to service_role;
  alter table public.share_link_visits enable row level security;
  create policy "visitor inserts own row"  on public.share_link_visits
    for insert to authenticated with check (visitor_id = auth.uid());
  create policy "visitor updates own row"  on public.share_link_visits
    for update to authenticated using (visitor_id = auth.uid());
  create policy "owner reads visits"       on public.share_link_visits
    for select to authenticated using (owner_id = auth.uid() or visitor_id = auth.uid());
  ```

### 4.3 데이터 계약

```ts
interface EmbedDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  token: string;              // share_token 없으면 song.id 를 폴백으로 넘긴다
  title?: string | null;
}
```

## ⑤ Acceptance & Output

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6)

- 「🔗 링크 복사」 클릭 시 다이얼로그가 열리지 않고, 클립보드에 `${origin}/shared/{token}` 문자열이 들어가며 토스트가 뜬다.
- 「📺 퍼가기 (임베드)」 클릭 시 `<EmbedDialog>` 하나만 열리고, 내부에는 **탭이 없고**, textarea 안 문자열은 정확히 `width="420" height="520"` 를 포함한다.
- 「📋 코드 복사」 클릭 후 버튼 라벨이 2초간 `✓ 복사됨` 으로 바뀐다.
- 익명 브라우저로 `/shared/:token` 진입 시 200 OK. `RequireAuth` 리다이렉트 없음. 분석 탭 6개가 렌더된다.
- 외부 도메인에서 `<iframe src="/embed/:token">` 삽입 시 스타일 격리, 사이드바/헤더 상속 없음.
- `rg -n "AppLayout|Sidebar" src/pages/EmbedSong.tsx` → **0**.
- `rg -n "TabsList|TabsTrigger" src/components/songs/EmbedDialog.tsx` → **0** (탭 UI 금지 확인).
- `rg -n "bg-\[#(?!2e3d6b)|text-white(?!\">\s*코)" src/components/songs/EmbedDialog.tsx src/pages/SharedSong.tsx src/pages/EmbedSong.tsx` 결과가 허용 목록(=`--dash-navy` CTA hover 만) 이외에 나오지 않는다.
- 로그인 사용자로 SharedSong 또는 EmbedSong 방문 후 `share_link_visits` 에 1 row 가 추가되고, 탭을 닫으면 `duration_seconds` 가 채워진다. 로그아웃 방문에서는 insert 가 발생하지 않는다.

### 5.2 Output Format

1. (선택) `songs.share_token` 백필 · `share_link_visits` 마이그레이션 SQL.
2. `src/hooks/useTrackShareVisit.ts`
3. `src/components/songs/EmbedDialog.tsx`
4. `src/pages/SharedSong.tsx`
5. `src/pages/EmbedSong.tsx`
6. `src/App.tsx` 라우트 추가 diff (`/shared/:token`, `/embed/:token` — 둘 다 `RequireAuth` 밖).
7. 한국어 3줄 요약.
