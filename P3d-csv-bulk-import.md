# P3d · CSV 노래 카드 일괄 가져오기 재현 프롬프트

> **본 프롬프트는 P3 시리즈(P3a–P3q)의 4/17.**
> **적용 대상**: `src/components/songs/BulkSongCardDialog.tsx`, `CsvImportDialog.tsx`, `CsvImportPanel.tsx`, `BulkRepairPanel.tsx` — CSV 템플릿 다운로드 · 업로드 · 파싱 · 중복 감지 · 진행률 UI · 재분석 파이프라인.
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* Retrieved July 12, 2026.
2. **Lovable. (n.d.).** *Prompting best practices.* Retrieved July 12, 2026 — 5개 실천 원칙 채택 근거.
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity (신원)

당신은 CSV 파서 · 언어 자동 감지 · 배치 upsert · 진행률 스트리밍 UI 를 다루는 시니어 프론트엔드 엔지니어입니다. `papaparse`, shadcn `Dialog`, `Progress`, 그리고 Supabase Edge Function `bulk-reanalyze-batch` 를 사용합니다.

## ② Instructions

### 2.1 산출물

- `src/components/songs/BulkSongCardDialog.tsx` — 3-스텝 위저드(템플릿 → 업로드 → 진행률).
- `src/components/songs/CsvImportDialog.tsx` — 단순 CSV 임포트 다이얼로그(재사용 가능).
- `src/components/songs/CsvImportPanel.tsx` — 파일 드롭존 + 미리보기 테이블 + 매핑.
- `src/components/songs/BulkRepairPanel.tsx` — 실패 곡 재시도 UI.
- `src/lib/csvSongs.ts` — 파서·별칭·언어 감지 유틸.

### 2.2 원자적 UI 규칙

**STEPS = `["템플릿", "업로드", "진행"]`** — `StepIndicator` 는 3개 원형 번호 + 현재 스텝만 navy fill.

**Step 1 · 템플릿**
- 카피: `CSV 템플릿을 다운로드하여 형식에 맞게 작성해주세요.`
- `TEMPLATE_HEADERS = ["title", "artist", "youtube_url", "language", "hsk_level", "theme", "teaching_point", "lyrics"]`
- 예시 행 3개(`TEMPLATE_ROWS`) 포함(중/한 각 1, 미지정 1).
- [템플릿 다운로드] navy 버튼(BOM 포함 UTF-8, MIME `text/csv`), [다음] 버튼.

**Step 2 · 업로드**
- Drop-zone(`border-2 border-dashed`): 파일 드래그 or 클릭 시 파일 선택. 카피 = `CSV 파일을 여기에 놓거나 클릭하여 선택`.
- 업로드 즉시 papaparse 로 파싱 → `CsvImportPanel` 하위에 미리보기 테이블(최대 10행) + 매핑 결과(감지된 컬럼 별칭 표시). 컬럼 별칭 사전:
  - title ← `title|제목|곡명|노래제목`
  - artist ← `artist|가수|아티스트|singer`
  - youtube_url ← `youtube_url|url|링크|유튜브`
  - language ← `language|언어|lang`
  - hsk_level ← `hsk_level|레벨|level|hsk|topik`
  - theme ← `theme|주제`
  - teaching_point ← `teaching_point|교학포인트|point`
  - lyrics ← `lyrics|가사`
- 언어 자동 감지: `language` 비어 있으면 `title+artist+lyrics` 문자열의 CJK Unified Ideograph ratio > 0.3 → `chinese`, 아니면 Hangul 존재 시 `korean`, 둘 다 없으면 `chinese` 기본.
- 중복 감지 배너: 이미 등록된 `video_id` 개수 표시(`skipCount`), `{skip}곡은 이미 등록되어 있어 건너뜁니다.`
- 하단: [뒤로] · [{N}곡 가져오기] navy(N = 유효 행 수, 중복 제외).

**Step 3 · 진행률**
- `<Progress value={percent}>` + 현재/총 카운트(`{done}/{total}`) + 현재 곡 제목 표시.
- 배치 사이즈 = 5, 병렬 실행. 각 곡별 `analyze-song` fire-and-forget. 실패 목록은 하단 `BulkRepairPanel` 에 축적.
- 완료 시 카피 = `가져오기 완료: 성공 {ok}곡 · 실패 {fail}곡`. 실패가 있으면 `[다시 시도]` 버튼 노출.
- 하단 [닫기] 버튼: 진행 중에는 확인 다이얼로그(`진행 중인 작업이 있습니다. 정말 닫으시겠습니까?`).

**CsvImportDialog** = 위 Step2 + Step3 만 사용하는 라이트 버전(사이드바 다른 페이지에서 재사용).

### 2.3 강제 제약

- 최대 파일 크기 = 2 MB, 최대 행 수 = 500. 초과 시 에러 토스트.
- BOM 감지 후 제거. 개행 CRLF/LF 모두 허용. 인용부 이스케이프 papaparse 기본값 사용.
- 필수 컬럼 = `title`, `artist`, `youtube_url` 중 최소 2개 이상 존재. 아니면 Step 2 진행 차단.
- semantic token, 하드코드 색상 금지.
- 모든 upsert 는 `owner_id = auth.uid()` 를 강제.

## ③ Examples

### 3.1 카피 표

| 위치 | 카피 |
|---|---|
| Dialog 헤더 | `노래 카드 일괄 생성` |
| Step 라벨 | `템플릿 / 업로드 / 진행` |
| 템플릿 안내 | `CSV 템플릿을 다운로드하여 형식에 맞게 작성해주세요.` |
| 템플릿 버튼 | `템플릿 다운로드` |
| 다음/뒤로 | `다음` / `뒤로` |
| Drop-zone | `CSV 파일을 여기에 놓거나 클릭하여 선택` |
| 중복 배너 | `{skip}곡은 이미 등록되어 있어 건너뜁니다.` |
| 가져오기 버튼 | `{N}곡 가져오기` |
| 진행 라벨 | `가져오는 중… ({done}/{total})` |
| 완료 | `가져오기 완료: 성공 {ok}곡 · 실패 {fail}곡` |
| 실패 재시도 | `다시 시도` |
| 필드 오류 | `필수 컬럼(title, artist, youtube_url) 중 최소 2개 이상이 필요합니다.` |
| 크기 오류 | `파일 크기는 2MB 이하여야 합니다.` |
| 행수 오류 | `한 번에 최대 500행까지 가져올 수 있습니다.` |
| 닫기 확인 | `진행 중인 작업이 있습니다. 정말 닫으시겠습니까?` |

### 3.2 컴포넌트 트리

```text
<BulkSongCardDialog max-w-3xl h-[80vh]>
  ├─ Header · StepIndicator [1][2][3]
  ├─ Step1 · Template
  │    └─ [템플릿 다운로드] [다음]
  ├─ Step2 · Upload
  │    ├─ DropZone
  │    ├─ <CsvImportPanel> (preview × 10 + 컬럼 매핑 배지)
  │    ├─ DuplicateBanner
  │    └─ [뒤로] [{N}곡 가져오기]
  └─ Step3 · Progress
       ├─ <Progress>
       ├─ 현재 곡 라벨
       ├─ <BulkRepairPanel> (실패 목록 + 재시도)
       └─ [닫기]
```

## ④ Context

### 4.1 프로젝트 맥락
교사가 학기 초 다수의 곡을 한 번에 등록하거나, 공개 아카이브에서 CSV 로 내보낸 곡 목록을 자기 아카이브로 옮길 때 사용. 진행 중에도 Dialog 를 닫아도 백그라운드에서 분석이 계속되도록 fire-and-forget.

### 4.2 Lovable Cloud 후경

- Edge Function `bulk-reanalyze-batch`(B2) 로 배치 재분석. 클라이언트는 5-병렬 스로틀만 담당.
- 성공 시 `songs` insert + `analyze-song` 트리거. 실패 시 클라이언트 배열에 축적 → 재시도 시 `bulk-reanalyze-batch` 호출.

### 4.3 데이터 계약

```ts
interface ParsedRow {
  rowIndex: number;
  title?: string; artist?: string; youtube_url?: string;
  language?: "chinese" | "korean";
  hsk_level?: string; theme?: string; teaching_point?: string;
  lyrics?: string;
  video_id?: string;      // extracted
  isDuplicate: boolean;
  errors: string[];
}
```

## ⑤ Acceptance & Output

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6)

- 500행 CSV 파싱 p95 ≤ **500 ms**(로컬).
- 5-병렬 배치로 500곡 트리거 소요 시간 ≤ **90 s**(edge function 응답 정상 시).
- Step2 → Step3 전환 시점부터 첫 곡 삽입까지 ≤ **1 s**.
- 중복 감지: 기존 `video_id` 100 % 스킵(테스트: pre-seed 10곡 + CSV 30곡 중 10 중복 → insert 20).
- 파일 > 2 MB 업로드 시 파싱 시작 전 에러 토스트, `songs` insert 0회.
- 컬럼 별칭 사전 8개 항목 모두 매칭(각 별칭당 fixture 1행 → 유효 행 판정).
- `rg -n "bg-\[#|text-white|bg-black" src/components/songs/BulkSongCardDialog.tsx src/components/songs/CsvImportDialog.tsx src/components/songs/CsvImportPanel.tsx src/components/songs/BulkRepairPanel.tsx` = **0**.

### 5.2 Output Format
1. `src/lib/csvSongs.ts`
2. `src/components/songs/CsvImportPanel.tsx`
3. `src/components/songs/BulkRepairPanel.tsx`
4. `src/components/songs/CsvImportDialog.tsx`
5. `src/components/songs/BulkSongCardDialog.tsx`
6. 한국어 3줄 요약.
