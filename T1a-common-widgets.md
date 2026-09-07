# T1a · 공용(Common) 위젯 11종 완전 스펙

> 본 문서는 T1(교사 대시보드) 및 학생 대시보드에서 **동일하게 재사용**되는 role-agnostic 위젯 11 개의 내부 렌더 로직을 정의한다. `DashboardGrid` · `ModuleRenderer` · `useDashboardLayout` · `_ModuleShell` 껍데기는 T1 문서에서 이미 확정되었다고 가정하며, 본 문서는 각 위젯 컴포넌트 파일(`src/components/dashboard/modules/*.tsx`) 의 완전 재현 프롬프트만 기술한다.

---

## 1. Identity

너는 Lovable 기반 React 18 + TypeScript + Tailwind 프로젝트의 프론트엔드 전문가다. T1 에서 확립한 대시보드 위젯 프레임워크(모듈 인스턴스 · 카탈로그 · 드래그 편집) 위에, **공용(Common) 카테고리 11 위젯의 실제 컴포넌트 11 개**를 파일 단위로 구현한다.

- 모든 위젯의 props: `{ instance: ModuleInstance; onConfigChange: (id, config)=>void; editing: boolean }`.
- 상태 지속은 `instance.config` 얕은 병합. 자유 텍스트는 400ms 디바운스 후 저장(`useEffect + setTimeout`).
- 대부분의 위젯은 `<ModuleShell title icon right bodyClassName>` 껍데기를 사용(`title/right` 미제공 시 `noHeader`).
- 색상·간격·폰트는 Tailwind + shadcn 토큰. `text-slate-500`, `text-indigo-600`, `bg-indigo-50` 등을 그대로 사용해도 되지만, 하드코드 hex 는 SNS 브랜드/특수 팔레트에만 허용한다.
- **role-agnostic**: 어떤 모듈도 `useAuth` 또는 `profile.role` 을 검사하지 않는다. 교사·학생 대시보드 둘 다에서 동일하게 동작해야 한다.

## 2. Instructions

`src/components/dashboard/modules/` 아래에 아래 11 파일을 생성한다. 각 위젯은 독립적으로 렌더 가능해야 하며 외부 API 호출은 `supabase.functions.invoke` 만 사용한다.

### 2.1 `_ModuleShell.tsx`

```tsx
interface Props { title?; right?; children; bodyClassName?: string; noHeader?: boolean; icon?: ReactNode; }
```
- Root `h-full flex flex-col`.
- Header (`!noHeader && (title || right)`) — `flex items-center justify-between px-4 pt-3 pb-2 border-b`. 좌: `icon + title` (`text-[13px] font-bold text-slate-800`), 우: `right` (`text-[11px] text-slate-400`).
- Body: `flex-1 min-h-0 ${bodyClassName || "p-4"}`.

### 2.2 `ClockModule.tsx` — 시계

- 자체 카드(껍데기 없이 gradient 배경 자체 렌더).
- `setInterval(1000)` 로 매 초 setState.
- 표시: `hh:mm` (36–40px 볼드 tabular-nums), `:ss` (20px 반투명), 아래 `date.toLocaleDateString("ko-KR",{ year,month,day,weekday: "long" })`.
- 배경 `bg-gradient-to-br from-slate-900 to-indigo-900 text-white rounded-[16px]`, 내부 `flex-col items-center justify-center p-6`.

### 2.3 `NotesModule.tsx` — 메모

- `<ModuleShell title="메모" icon={<span>📝</span>} bodyClassName="p-3">`.
- shadcn `Textarea` 사용. `value={text}` controlled, 400ms 디바운스로 `onConfigChange({ text })`.
- `className="h-full min-h-[120px] resize-none text-[12.5px]"`, placeholder "여기에 메모하세요…".

### 2.4 `YoutubeEmbedModule.tsx` — 영상 임베드

- 껍데기 `title="영상 임베드" icon="▶️"`.
- `parseId(url)` 정규식 `/(?:youtu\.be\/|v=|embed\/)([\w-]{11})/`.
- Input(h-8) 에 URL 입력, 실시간으로 `config.url` 저장.
- 유효 id 있으면 `aspect-video rounded-md overflow-hidden bg-black` 컨테이너에 `<iframe src="https://www.youtube.com/embed/{id}" allowFullScreen>`. 없으면 안내 문구.

### 2.5 `LinkCardModule.tsx` — 링크 카드

- 껍데기 `title="링크 카드" icon="🔗"`.
- `config.{title,url}` 존재 시 `<a target="_blank" class="block p-3 rounded-lg bg-indigo-50 hover:bg-indigo-100">` — 좌: `text-[13px] font-bold text-indigo-700` 제목, 우: `ExternalLink` 아이콘. 아래 `url` truncate. 하단 「변경」 링크로 리셋.
- 없으면 title / url 두 입력(h-8, 각각 debounced-less 즉시 저장).

### 2.6 `QuoteModule.tsx` — 오늘의 한마디

- 껍데기 `title="오늘의 한마디" icon="💬" right={<RefreshCw>}`.
- 상수 `QUOTES` 6 개(한국어·중국어 혼합):
  - "음악은 또 다른 언어다." / "가르치는 것은 두 번 배우는 것이다." — Joseph Joubert / "教学相长" — 礼记 / "노래는 마음을 여는 가장 짧은 길이다." / "学而时习之，不亦说乎" — 论语 / "좋은 교사는 학생을 미래로 데려간다."
- 초기 index 는 `Math.random() * len`, Refresh 버튼 `(prev+1)%len`.
- `<blockquote class="text-[14px] font-medium text-slate-700">` + 우측 정렬 `text-[11px] text-slate-400` 저자.

### 2.7 `PomodoroModule.tsx` — 포모도로

- 껍데기 `title="포모도로" icon="⏱️"`.
- 상수 `TOTAL = 25 * 60`.
- 상태 `left`, `running`. `useEffect` 로 running 시 `setInterval(1000)` 로 `left--` (0 하한).
- 표시: `mm:ss` 36px 볼드, 아래 진행바 `h-1.5 bg-slate-100` 위에 `bg-gradient-to-r from-indigo-500 to-violet-500` width=`((TOTAL-left)/TOTAL)*100%`.
- 하단 2 버튼: 「시작/정지」 파란 배경 flex-1, 「리셋」 border(left=TOTAL, running=false).

### 2.8 `DdayModule.tsx` — D-day

- 껍데기 `title="D-day" icon="📅"`.
- `config.{name,date}` 저장. 즉시 저장(디바운스 없음, 매 keystroke).
- 남은 일 계산: `Math.ceil((new Date(date) - Date.now()) / 86400000)`.
- 표시 모드: 이름(12px slate-500) → `D-{days}` 또는 `D-Day`/`D+{|days|}` (40px 인디고 볼드) → 날짜(10px slate-400) → 하단 「변경」 링크.
- 선택 모드: 이벤트명 input + `<input type="date">`.

### 2.9 `SongRecommendationModule.tsx` — 오늘의 노래 추천

- 껍데기 `title="오늘의 노래 추천" icon={<amber 6x6 원형에 ✨>} right="매일 업데이트"`.
- 상태: `song`, `loading`, `tip`, `tipLoading`, `excludeIds[]`, `playing`.
- 진입 시 `fetchSong([])`:
  1. `supabase.from("songs").select("*").is("deleted_at", null)` — exclude 목록이 있으면 `.not("id","in", ...)`.
  2. 결과 없으면 exclude 없이 재조회.
  3. 랜덤 1 개 선택 → `setSong`, `excludeIds` 에 push, `fetchTip(s)`.
- `fetchTip(song)`: `supabase.functions.invoke("generate-teaching-tip", { body: { title, artist, hsk_level, language, teaching_point, lyrics_raw } })`. status 402 → "AI 크레딧이 부족합니다.", 429 → 재시도 안내. 실패 시 `teaching_point` 로 fallback.
- 카드 렌더:
  - 상단 16:9 검정 컨테이너. `playing && video_id` → autoplay iframe. 아니면 `img.youtube.com/vi/{id}/hqdefault.jpg` + 재생 아이콘 오버레이. 클릭 시 `setPlaying(true)`.
  - 중간: 제목(14px 볼드) + 아티스트, 우측 HSK→레벨 뱃지(`hskToLevel`: HSK 1-2 초급, 3-4 중급, 5+ 고급).
  - 팁 박스: `bg-slate-50 border-l-2 border-amber-300 min-h-[44px]` 안에 tip(로딩 시 spinner + "교학 팁 생성 중…").
  - 버튼 2 개: 「다른 곡」(border, `fetchSong(excludeIds)`), 「분석 보기」(bg `#233057` 다크 네이비, `/songs?view={id}` 이동).

### 2.10 `StickyNoteModule.tsx` — 포스트잇

- 껍데기 없이 자체 카드 (`h-full min-h-[140px] p-3 flex flex-col gap-2`).
- 색상 4택 (yellow/green/red/purple), 각 `{bg, pin, text}` 팔레트.
- 카드: `rounded-[10px] p-3 shadow-md`, `background: bg`, `transform: rotate(-1deg)`. 상단 중앙 `w-3 h-3 rounded-full` 핀. 내부 `<textarea>` — 배경 투명, text 색 = `p.text`, 400ms 디바운스로 `config.{color,text}` 저장.
- 하단 4 색 스와치(원형, 활성 시 `border-slate-900`).

### 2.11 `DividerModule.tsx` — 구분선

- 껍데기 없이 `px-4 py-3 flex-col gap-2`.
- variant 8 종: `solid | dashed | wave | double | gradient | icon | text | glow`.
- `line` 렌더:
  - solid: `flex-1 h-px bg-slate-200`.
  - dashed: `border-top: 2px dashed #c7caff`.
  - double: `border-top: 3px double #94a3b8`.
  - gradient: `h-[2px]` + `linear-gradient(90deg, transparent, #4f52c8, #7c3aed, #4f52c8, transparent)`.
  - glow: `h-[2px] bg-#4f52c8` + `box-shadow: 0 0 8px rgba(79,82,200,.6), 0 0 16px rgba(79,82,200,.3)`.
  - wave: 12px 높이 SVG data-URI 파도 반복.
- `icon`/`text` variant 는 line + label + line 샌드위치. `text` 는 controlled input(placeholder "섹션 제목").
- `editing === true` 일 때만 하단에 variant 8 개 pill 버튼(활성 = 인디고 배경).
- 400ms 디바운스로 `{ label, variant }` 저장.

### 2.12 `CalendarModule.tsx` (참고)

`calendar` 위젯은 껍데기 대신 `<DashboardCalendar />` 전체를 감싼다. 내부 스펙은 T1 문서 2.11 절에서 이미 완결됐으므로 본 문서에서는 매핑만 재확인.

```tsx
export function CalendarModule(_props: ModuleCtxProps) {
  return <div className="h-full"><DashboardCalendar /></div>;
}
```

## 3. Examples

### 예시 A — 「메모」 위젯이 새로고침 후에도 텍스트 유지
1. 사용자가 카탈로그에서 `notes` 를 드래그해 그리드에 추가 → `instance = { id, type:"notes", size:"M" }`.
2. 텍스트를 타이핑 → 400ms 후 `onConfigChange(id, { text })` 호출 → `useDashboardLayout` 이 `instance.config.text` 병합 후 debounce localStorage 저장.
3. 새로고침 → localStorage 복원 → `NotesModule` 초기 state `instance.config?.text` 로 시작 → 동일 텍스트 노출.

## 4. Context

- 관련 파일:
  - `src/components/dashboard/modules/_ModuleShell.tsx`
  - `src/components/dashboard/modules/{Clock|Notes|YoutubeEmbed|LinkCard|Quote|Pomodoro|Dday|SongRecommendation|StickyNote|Divider|Calendar}Module.tsx`
- 의존:
  - `@/integrations/supabase/client` (SongRecommendation).
  - `@/hooks/use-toast` (SongRecommendation).
  - `@/components/ui/textarea` (Notes).
  - Edge function: `generate-teaching-tip`.
- 라우트: `/dashboard`(교사), `/student-home`(학생) — 두 곳 모두에서 동일 위젯 렌더.
- 카탈로그: `dashboardModuleCatalog.tsx` 에서 각 위젯의 `label / description / icon / category:"common" / defaultSize` 를 관리한다. label/description 은 실제 UI 문자열과 일치해야 한다(교사 가이드 T1 및 학생 가이드 문서 참조).

## 5. Acceptance

- [ ] 위 11 개 파일이 존재하며, `ModuleRenderer` switch 에서 각각 정확히 매핑된다.
- [ ] Clock: 매 초 갱신, 한국어 요일 표시, 다크 네이비 gradient.
- [ ] Notes: 400ms 디바운스 저장, 재진입 시 값 복원.
- [ ] Youtube: youtu.be / v= / embed/ URL 모두 파싱, 유효 id 없으면 안내.
- [ ] LinkCard: 표시 모드에서 새 탭 이동, 「변경」으로 폼 복귀.
- [ ] Quote: 6 개 pool 순환, Refresh 클릭 시 다음 인덱스.
- [ ] Pomodoro: 25:00 카운트다운, 시작/정지/리셋, 진행바 gradient.
- [ ] Dday: 오늘=`D-Day`, 미래=`D-N`, 과거=`D+N`.
- [ ] SongRecommendation: 진입 시 랜덤 곡 + AI 팁, 「다른 곡」 시 excludeIds 로 중복 제거, 「분석 보기」 `/songs?view=` 이동, 402/429 에러 문구 정확.
- [ ] StickyNote: 4 색 팔레트 전환 시 배경/글자색 즉시 반영, 회전 -1deg.
- [ ] Divider: 8 variant 렌더 모두 육안으로 구분, editing=false 이면 pill 숨김.
- [ ] 모든 위젯은 `useAuth`/role 검사를 하지 않으며 교사·학생 대시보드 모두에서 동일 동작.
