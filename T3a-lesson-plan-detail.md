# T3a — 강의안 상세 골격 `LessonPlanDetail` (헤더 · 주차 리스트 · 개별/전체 생성 · 즐겨찾기 · TOPIK 백필)

> 4.2.3 教师端 세 번째 재현 프롬프트. `MyLessonsSection` 에서 강의안을 클릭하면 열리는 **상세 페이지 셸**을 그대로 재현한다. 파일 하나 `src/components/lessons/LessonPlanDetail.tsx` 로 완결되며, AI 생성 파이프라인의 프롬프트·엣지 함수·재개 로직은 **T3b** 에서 별도로 다룬다.

---

## 1. Identity — 이 프롬프트로 만드는 것

**이름**: 강의안 상세 (Lesson Plan Detail).
**진입점**: 
- `Workspace.tsx` 의 `selectedPlan` 이 채워지면 페이지를 대체해 렌더 (`<LessonPlanDetail plan onBack />`).
- 즐겨찾기 뷰 `BookmarkedWeeksView` 에서도 `showBookmark={true}` 로 재사용.
**한 줄 정의**: 하나의 `lesson_plans` row 를 열어 **N 개 `lesson_weeks` 를 세로 리스트**로 보여주고, **개별/전체 AI 생성 · 재생성 · 즐겨찾기 · 주차 상세 진입**을 한 화면에서 처리하는 셸.
**뒤로**: `onBack()` — 워크스페이스 또는 즐겨찾기 뷰로 복귀.
**앞으로**: 주차 카드 클릭 → 이 컴포넌트 안에서 `selectedWeek` 를 세팅하고 `WeekDetailView` 로 전환 (라우팅 없음, 로컬 state 스왑).

---

## 2. Instructions — 반드시 지킬 것

### 2.1 데이터 계약

- Props:
  - `plan: any` — `lesson_plans` 한 row. **최소 필드**: `id, title, level, courses?.name`. 필요 시 `plan.title` 를 컴포넌트에서 **직접 mutate** 한다 (제목 저장 후 로컬 참조 유지용).
  - `onBack: () => void`.
  - `showBookmark?: boolean` — 기본 `false`. 즐겨찾기 뷰에서만 `true`.
- 상태:
  - `weeks: LessonWeek[]` — `lesson_weeks` 를 `week_number asc` 로 조회.
  - `loading` (초기 fetch), `generatingWeek: number | null`, `selectedWeek`, `editingTitle`, `planTitle`, `bookmarkedIds: Set<string>`, `bulkProgress: {current, total} | null`, `cancelBulkRef: useRef(false)`.
- `LessonWeek` 타입: `{id, lesson_plan_id, week_number, title, week_type, content, song_ids, is_generated}`.
- 초기 로드: `fetchWeeks()` 는 `lesson_weeks` 를 `lesson_plan_id = plan.id` 로 `.select("*").order("week_number", { ascending: true })`, 성공 시 `setWeeks`, 실패해도 `setLoading(false)`.

### 2.2 상단 헤더 바

- `flex items-center gap-3` 한 줄에 좌→우:
  1. `ArrowLeft` 아이콘 버튼 (`variant="ghost" size="icon"`) → `onBack()`.
  2. **제목 블록** (`flex-1`):
     - 기본: `h1 text-2xl font-bold` 로 `planTitle` + 옆에 `Edit2` 아이콘 버튼 (7×7).
     - 편집 모드: `Input text-xl font-bold` + `Save` 아이콘 버튼 (`size="sm"`). 저장 시 `lesson_plans.update({title: planTitle}).eq('id', plan.id)` → `plan.title = planTitle` (참조 갱신) → `toast("제목 저장 완료")`.
     - 제목 아래 회색 서브라인: `{plan.level} · {plan.courses?.name || ""}`.
  3. **전체 생성 / 중단 버튼** (`bulkProgress` 유무로 스왑):
     - 진행 중: `variant="destructive" size="sm"` "`Square` 중단 (current+1/total)" → `cancelBulkRef.current = true`.
     - 대기 중: `variant="default" size="sm"` "`Sparkles` 전체 생성" → `handleGenerateAll()`. `disabled` 조건: `generatingWeek !== null || weeks.every(w => w.is_generated)`.
  4. `showBookmark === true` 일 때만: `variant="outline" size="sm"` "`Bookmark` 전체 즐겨찾기" → `handleBookmarkAll()`.

### 2.3 주차 리스트 (본문)

- `loading` → 중앙 `Loader2 h-8 w-8 animate-spin`.
- 아니면 `space-y-2` 세로 리스트, 각 요소는 `Card hover:shadow-md transition-shadow cursor-pointer group`.
- `CardContent p-4 flex items-center gap-4`:
  1. **주차 번호 원형 배지**: `h-10 w-10 rounded-full bg-primary/10 text-primary font-bold text-sm shrink-0`, 텍스트 = `w.week_number`.
  2. **본문** (`flex-1 min-w-0`):
     - 첫 줄: `weekTypeIcon(w.week_type)` + `p font-medium truncate` = `w.title`.
     - 곡이 있을 때 두 번째 줄: `text-xs text-muted-foreground` "🎵 " + `w.content.songs.map(s => s.title).join(" / ")` (한 줄 truncate).
  3. **우측 도구 클러스터** (`flex items-center gap-2 shrink-0`):
     - `Badge variant="outline"` + `weekTypeBadge(week_type)` 색상 + `weekTypeLabel[week_type] || week_type`.
     - `is_generated` 이면 "`Sparkles` 다시 생성" 버튼 (`onClick=stopPropagation → handleGenerateWeek`).
     - `showBookmark && is_generated` 이면 즐겨찾기 토글 아이콘: `bookmarkedIds.has(w.id)` 면 `BookmarkCheck text-primary`, 아니면 `Bookmark text-muted-foreground hover:text-primary`.
     - 상태별 우측 마감 요소:
       - `generatingWeek === w.week_number` → `Loader2 animate-spin`.
       - `is_generated` → `ChevronRight` (그룹 hover 시 foreground).
       - 미생성 → `variant="outline" size="sm"` "`Sparkles` 생성" 버튼 (`stopPropagation → handleGenerateWeek`).

### 2.4 카드 클릭 라우팅

- 카드 자체 `onClick` 분기:
  - `!w.is_generated` → `handleGenerateWeek(w.week_number)` 만 실행하고 상세로 넘어가지 않음 (사용자가 먼저 콘텐츠를 만들도록 강제).
  - 생성된 주차 → **`await ensureKoreanAnalysisForWeek(w)`** 후 `setSelectedWeek(w)`.
- `selectedWeek` 가 있으면 컴포넌트는 리스트를 감추고 `<WeekDetailView week={selectedWeek} planTitle={planTitle} allWeeks={weeks} onBack={() => { setSelectedWeek(null); fetchWeeks(); }} />` 하나만 리턴한다. `WeekDetailView` 자체의 재현은 **T3b** 이후 별도 문서.

### 2.5 개별 생성 `handleGenerateWeek`

- `setGeneratingWeek(weekNum)` → `supabase.functions.invoke("generate-lesson-plan", { body: { action: "generate_week", lesson_plan_id: plan.id, week_number: weekNum } })`.
- `error` 또는 `data?.error` 있으면 throw. 성공 시:
  - toast `"{weekNum}주차 내용이 생성되었습니다!"`.
  - `await fetchWeeks()` 로 리스트 재조회.
  - 응답의 `data.content` 를 기반으로 임시 `LessonWeek` 를 조립해 `ensureKoreanAnalysisForWeek(justGen, { silent: true })` 를 **비동기 백그라운드** 로 태운다 (실패 무시).
- `finally` 에서 `setGeneratingWeek(null)`. 실패 시 `toast destructive`.

### 2.6 전체 생성 `handleGenerateAll` — 순차 실행 + 취소 가능

- `pending = weeks.filter(!is_generated).map(week_number).sort(asc)`. 0 이면 안내 toast 후 반환.
- `cancelBulkRef.current = false` → `setBulkProgress({current:0, total:pending.length})`.
- `for` 루프로 순차 처리 (**병렬 금지** — 429/402 방지):
  - 매 반복 시작에서 `if (cancelBulkRef.current) break;`.
  - `setGeneratingWeek(weekNum)` + `setBulkProgress({current:i, total})`.
  - `generate_week` 호출 → 성공 시 `success++`, `await fetchWeeks()` 로 리스트 즉시 반영. 실패 시 `failed.push(weekNum)` 하고 계속 진행.
- 종료:
  - 취소됨: `"생성 중단됨 ({success}/{total} 완료)"`.
  - 모두 성공: `"전체 생성 완료! ({success}주차)"`.
  - 일부 실패: `destructive` toast, `"{success}주차 완료, {failed.length}주차 실패"` + description `"실패: {n1, n2, …}주차. 개별 생성 버튼으로 다시 시도하세요."`.
- 종료 시 `setGeneratingWeek(null)` + `setBulkProgress(null)`.

### 2.7 TOPIK 백필 `ensureKoreanAnalysisForWeek(week, { silent? })`

- **목적**: TOPIK 강의안에서 각 곡이 `song_analyses` 에 한국어(=`learn_language: "korean"`) 모드 결과를 갖게 보장.
- 가드: `plan.level.toUpperCase().startsWith("TOPIK")` 이 아니면 즉시 return. `week.song_ids` 가 비어도 return.
- 데이터 수집:
  - `songs: id, youtube_url, language, hsk_level` (`in ids`).
  - `song_analyses: song_id, word_list` (`in ids`).
- **재분석 대상**: 다음을 모두 만족하는 song
  - `youtube_url` 존재.
  - 해당 분석이 없거나, 분석의 `word_list` 안에 `topik_level != null` 인 항목이 하나도 없거나, `songs.hsk_level` 문자열이 `"TOPIK"` 로 시작하지 않음.
- 대상이 0 이면 return. `silent !== true` 이면 toast: 제목 `"곡 분석 정렬 중…"`, 설명 `"{N}곡을 한국어 배우기 모드로 재분석합니다."`.
- 각 대상에 대해 순차로 `functions.invoke("analyze-song", { body: { youtube_url, force:true, language, learn_language:"korean" } })`, 실패 시 `console.warn` 만 남기고 계속. (사용자 흐름 차단 금지.)

### 2.8 즐겨찾기 (`showBookmark === true` 일 때만)

- 단일 `handleBookmarkWeek(week, e)`:
  - `stopPropagation`. 이미 세트에 있으면 안내 toast 후 반환.
  - `bookmarked_weeks.insert({ source_week_id: week.id, week_data: week.content || {}, title: week.title, week_type: week.week_type })`.
  - 성공 시 `setBookmarkedIds(new Set([...prev, week.id]))` + toast `"즐겨찾기에 추가되었습니다!"`. 실패 → toast destructive.
- 전체 `handleBookmarkAll`:
  - 대상 = `weeks.filter(is_generated && !bookmarkedIds.has(id))`. 0 이면 안내 toast.
  - `insert` 를 배열 한 번으로 (`inserts = mapped rows`). 성공 시 `Set` 에 모두 추가 + toast `"{N}개 주차가 즐겨찾기에 추가되었습니다!"`.

### 2.9 주차 타입 시각 토큰

- `weekTypeIcon(type)`:
  - `orientation` → `BookOpen`, `midterm` / `final` → `FileText`, 기본 → `Music`. 크기 `h-4 w-4`.
- `weekTypeBadge(type)` (Tailwind 문자열):
  - `orientation`: `bg-blue-500/10 text-blue-600 border-blue-500/20`.
  - `midterm`: `bg-amber-500/10 text-amber-600 border-amber-500/20`.
  - `final`: `bg-red-500/10 text-red-600 border-red-500/20`.
  - 기본(`regular`): `bg-emerald-500/10 text-emerald-600 border-emerald-500/20`.
- `weekTypeLabel`: `{orientation:"오리엔테이션", regular:"정규 수업", midterm:"중간고사", final:"기말고사"}`.

### 2.10 안전 / UX 규칙

- **개별 생성 중에도 전체 생성 버튼은 disabled**. 반대는 성립 (전체 도중에도 카드가 순차적으로 스피너로 표시).
- 카드 안의 모든 버튼은 `onClick=(e)=>{ e.stopPropagation(); … }` — 카드 자체의 클릭(주차 상세 진입)과 충돌 금지.
- 헤더 제목 편집 중에는 `Enter` 없이 오직 저장 버튼으로만 확정 (실수 저장 방지).
- `WeekDetailView` 에서 복귀 시 반드시 `fetchWeeks()` 를 재실행해 최신화.
- TOPIK 백필은 **best-effort · 무시 가능**. 실패해도 상세 진입은 계속 진행 (사용자를 절대 막지 않는다).

---

## 3. Examples — 골격 스니펫

### 3.1 컴포넌트 시그니처와 초기 fetch

```tsx
// src/components/lessons/LessonPlanDetail.tsx
export function LessonPlanDetail({ plan, onBack, showBookmark = false }: Props) {
  const [weeks, setWeeks] = useState<LessonWeek[]>([]);
  const [loading, setLoading] = useState(true);
  const [generatingWeek, setGeneratingWeek] = useState<number | null>(null);
  const [selectedWeek, setSelectedWeek] = useState<LessonWeek | null>(null);
  const [editingTitle, setEditingTitle] = useState(false);
  const [planTitle, setPlanTitle] = useState(plan.title);
  const [bookmarkedIds, setBookmarkedIds] = useState<Set<string>>(new Set());
  const [bulkProgress, setBulkProgress] = useState<{current:number;total:number}|null>(null);
  const cancelBulkRef = useRef(false);
  const { toast } = useToast();

  useEffect(() => { fetchWeeks(); }, []);
  // …
}
```

### 3.2 전체 생성 루프 뼈대

```tsx
const handleGenerateAll = async () => {
  const pending = weeks.filter(w => !w.is_generated).map(w => w.week_number).sort((a,b)=>a-b);
  if (!pending.length) return void toast({ title: "모두 이미 생성되었습니다." });
  cancelBulkRef.current = false;
  setBulkProgress({ current: 0, total: pending.length });
  let success = 0; const failed: number[] = [];
  for (let i = 0; i < pending.length; i++) {
    if (cancelBulkRef.current) break;
    setGeneratingWeek(pending[i]);
    setBulkProgress({ current: i, total: pending.length });
    try {
      const { data, error } = await supabase.functions.invoke("generate-lesson-plan", {
        body: { action: "generate_week", lesson_plan_id: plan.id, week_number: pending[i] },
      });
      if (error || data?.error) throw new Error(error?.message || data.error);
      success++;
      await fetchWeeks();
    } catch (e) { failed.push(pending[i]); }
  }
  setGeneratingWeek(null); setBulkProgress(null);
  // … 결과별 toast
};
```

### 3.3 주차 카드 라우팅

```tsx
<Card onClick={async () => {
  if (!w.is_generated) return handleGenerateWeek(w.week_number);
  await ensureKoreanAnalysisForWeek(w);
  setSelectedWeek(w);
}}>
  {/* 번호 배지 · 아이콘+제목+곡 · badge · (다시 생성) · (즐겨찾기) · ChevronRight/Loader/생성 */}
</Card>
```

---

## 4. Context — 이 문서가 기대는 것

- `Workspace.tsx` 가 `selectedPlan` state 를 유지하고 상세를 페이지 대체로 렌더 (T2 §2.1).
- `WeekDetailView` 컴포넌트는 이 문서가 아니라 후속 문서에서 정의. 여기서는 인터페이스 (`week, planTitle, allWeeks, onBack`) 만 계약으로 잡는다.
- Supabase edge function `generate-lesson-plan` 의 `action` 스펙 (`generate_outline` / `generate_week`) 은 **T3b** 에서 정의. 이 문서는 호출 지점과 결과 반영만 다룬다.
- Supabase edge function `analyze-song` 은 곡 아카이브 파이프라인 (P3 시리즈) 문서에 이미 정의.
- 데이터 리트라이/타임아웃은 프로젝트 공용 규칙 (`fetchWithRetry`) 을 따라도 되지만, `functions.invoke` 는 엣지 함수 내부에 자체 재개 로직이 있으므로 여기서는 감싸지 않는다.
- Toast · Dialog · Badge · Card 는 shadcn/ui 표준 컴포넌트를 그대로 사용.

---

## 5. Acceptance — 재현 검수 기준

1. 강의안을 열면 헤더의 제목·레벨·반 이름이 즉시 보인다. 제목 오른쪽 연필 아이콘으로 편집·저장 시 리스트 리로드 없이 헤더가 갱신되고 DB `lesson_plans.title` 이 반영된다.
2. 아직 미생성 주차 카드를 클릭하면 상세로 이동하지 않고 그 자리에서 스피너가 돌면서 생성이 시작된다. 완료 후 카드가 "생성됨" 상태로 바뀐다.
3. 생성된 주차 카드를 클릭하면 `WeekDetailView` 로 진입하고, 돌아오면 리스트가 자동 재조회된다.
4. **전체 생성** 버튼을 누르면 순차적으로 각 미생성 주차가 스피너 → 완료로 진행되며 헤더 버튼이 "중단 (i+1/n)" 로 바뀐다. 도중에 "중단" 을 누르면 진행 중인 요청 완료 후 루프가 즉시 종료되고 완료/전체 개수 toast 가 뜬다.
5. 일부 실패해도 나머지는 계속 진행되며, 종료 시 실패한 주차 번호 목록이 destructive toast description 에 그대로 노출된다.
6. TOPIK 강의안에서 곡의 분석이 HSK 모드로만 있던 경우, 주차 상세 진입 직전에 "곡 분석 정렬 중…" toast 가 뜨고 백엔드가 한국어 모드로 재분석을 태운다. 완료를 기다리지 않고 상세로 진입한다.
7. `showBookmark={true}` 로 열면 헤더에 "전체 즐겨찾기" 버튼이 노출되고, 각 생성된 주차 카드 우측에 즐겨찾기 아이콘이 추가된다. 눌러 추가/이미 추가됨 상태가 즉시 시각적으로 구분된다.
8. AI 생성 파이프라인(엣지 함수·프롬프트·재개·429/402 처리)의 상세 계약은 **T3b** 문서에서 검수한다. 이 문서 범위 안에서는 "호출 지점 · 성공/실패 UX · 순차·취소" 만을 판정 대상으로 삼는다.
