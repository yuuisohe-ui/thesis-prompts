# T3b — 강의안 AI 생성 파이프라인 (`generate-lesson-plan` 엣지 함수 · 프롬프트 · 재개 · 어시스턴트)

> 4.2.3 教师端 네 번째 재현 프롬프트. **T3a** 에서 만든 `LessonPlanDetail` 셸이 호출하는 **AI 생성 백엔드**를 그대로 재현한다. 파일은 `supabase/functions/generate-lesson-plan/index.ts` 한 개, 그리고 클라이언트 측 얇은 래퍼(`WeekDetailView` 의 `handleGenerateDetail`, `FloatingAIChat`, `ExamReviewFrameworkView` 의 `generateReview` / `generateExamples`, `WeeklyFrameworkView` 의 카드별 재생성 버튼) 를 다룬다.
>
> **범위 밖(다른 문서에서 다룸)**: 주차 상세 화면(모듈 드래그·인쇄·즐겨찾기 등) UI 는 T3c 이후에서 다룬다. **강의안 즐겨찾기 기능은 현재 플랫폼에 존재하지 않으므로 언급하지 않는다.**

---

## 1. Identity — 이 프롬프트로 만드는 것

**이름**: 강의안 AI 파이프라인 (Lesson-Plan AI Pipeline).
**형태**: Supabase Edge Function `generate-lesson-plan` (Deno, `verify_jwt = false` 기본) + 클라이언트 `supabase.functions.invoke("generate-lesson-plan", { body: { action, ... } })`.
**한 줄 정의**: 하나의 `lesson_plans` row 를 대상으로 **개요 → 주차 프레임(오리엔테이션 / 표준 / 시험 복습) → 부분 재생성 → 어시스턴트 대화**까지 모든 GPT 호출을 담당하는 **단일 액션 라우터 함수**.
**모델**: `gpt-4o-mini` 고정, `tools` / `tool_choice` 기반 **구조화 JSON 반환** (자유 텍스트는 어시스턴트 액션에서만 사용).

---

## 2. Instructions — 반드시 지킬 것

### 2.1 함수 골격 (액션 라우터)

- CORS: `Access-Control-Allow-Origin: *`, `Allow-Headers` 에 `authorization, x-client-info, apikey, content-type, x-supabase-client-*`. `OPTIONS` 요청은 즉시 응답.
- 시크릿: `OPENAI_API_KEY`. 없으면 500. Lovable Cloud 프로젝트의 `SUPABASE_URL` / `SUPABASE_SERVICE_ROLE_KEY` 로 서비스 롤 클라이언트 생성.
- 진입점은 `body.action` 문자열에 따른 스위치. 지원 액션(전부 구현 필수):
  - `generate_outline` — 학기 전체 개요(주차 배열 + 곡 배치) 생성 후 `lesson_plans.outline` / `lesson_weeks` 초기화.
  - `generate_week` — 한 주차의 본문 프레임 생성 (오리엔테이션 / 표준 / 복습 자동 분기).
  - `generate_exam_review` — 중간·기말 주차의 복습 프레임 생성.
  - `generate_exam_examples` — 복습 프레임의 예제 문제 세트 추가 생성.
  - `add_vocab` / `add_questions` / `add_discussion` / `add_homework` / `regen_dialog` — 표준 프레임 각 카드 **부분 재생성**(기존 항목을 `*_extra` 로 이어붙임).
  - `ask_assistant` — 우측 하단 플로팅 챗봇의 자유 텍스트 응답.
- 알 수 없는 액션은 `{ error: "Unknown action" }` + 400.
- 모든 정상 응답은 `jsonResp(obj, status=200)` 헬퍼로 `{ ...corsHeaders, "Content-Type": "application/json" }` 를 씌워 반환.

### 2.2 GPT 호출 규약

```ts
function callOpenAI(apiKey, messages, tools?, toolChoice?) {
  return fetch("https://api.openai.com/v1/chat/completions", {
    method: "POST",
    headers: { Authorization: `Bearer ${apiKey}`, "Content-Type": "application/json" },
    body: JSON.stringify({ model: "gpt-4o-mini", messages, ...(tools ? { tools, tool_choice: toolChoice } : {}) }),
  });
}
```

- 구조화 결과가 필요한 모든 액션은 `tools: [{ type: "function", function: { name, description, parameters } }]` 와 `tool_choice: { type: "function", function: { name } }` 를 함께 강제한다.
- `response.ok === false` 처리 규약(그대로 사용자 UI로 노출):
  - `429` → `jsonResp({ error: "요청이 너무 많습니다." }, 429)`.
  - `402` → `jsonResp({ error: "크레딧이 부족합니다." }, 402)`.
  - 기타 → `throw new Error("AI generation failed")` → catch 에서 500.
- `tool_calls[0].function.arguments` 는 반드시 `JSON.parse` 후 스키마 필드 존재 여부만 신뢰. 없으면 `throw new Error("AI did not return structured output")`.

### 2.3 이중 언어 · 난이도 강제 주입 (핵심)

두 개의 헬퍼를 **모든 프롬프트 끝**에 순서대로 붙인다. 순서와 문구는 그대로 유지해야 등급차·언어차가 안정적으로 나온다.

- `getOutputLangDirective(plan.output_lang)` — `"zh"` 면 **간체 중국어 강제 블록**(한자/병음/원문 가사·곡명·가수명만 원어 유지), 그 외(`"ko"` 기본)면 **한국어 강제 블록**. 시스템 메시지가 한국어로 쓰였더라도 사용자 응답은 이 지시를 우선한다는 문구 포함.
- `getLevelGuide(plan.level)` — 라벨(`HSK 1-9` / `TOPIK 1-6`)에서 숫자를 뽑아 **입문·중급·고급** 3 tier 로 매핑하고, 각 tier 마다 다음 6개 축의 구체 지침을 반환:
  1. 어휘 범위(예 `HSK 1~3` / `TOPIK 5~6`)
  2. 문장 길이(6-10 / 10-18 / 복문 OK)
  3. 어조(따뜻함 → 또래 → 지적 파트너)
  4. 회화·예문 주제(일상 → 의견 교환 → 추상)
  5. 학습 기대치(따라 말하기 → 경험 연결 → 분석·비평)
  6. 토론 질문 깊이 · 과제 부담 수준
- 매 프롬프트 시스템 메시지에서 "**아래 난이도 가이드를 절대적으로 준수하세요**" 라고 명시하고, 사용자 메시지에도 `## 난이도 지침` 섹션으로 한 번 더 삽입 → GPT 가 등급을 무시하지 못하도록 이중 앵커.

### 2.4 액션별 프롬프트 & 도구 스키마

#### (a) `generate_outline`

- 입력: `{ lesson_plan_id, num_weeks, class_duration_min, mid_exam_week, final_exam_week, songs_per_week, cn_song_count, kr_song_count, keywords? }`.
- 사전 검증:
  - `finalWeek > numWeeks` → `numWeeks` 로 클램프.
  - `midWeek >= finalWeek` → `midWeek = Math.floor(numWeeks/2)`, `finalWeek = numWeeks`.
  - 오리엔테이션(1주) · 중간 · 기말을 제외한 `regularWeeks * songsPerWeek === cnCount + krCount` 이어야 함. 안 맞으면 400.
- 후보 곡 풀 조회: `songs` 테이블에서 `owner_id = plan.owner_id OR owner_id IS NULL`, `deleted_at IS NULL`. 각 곡을 `hsk_level` 숫자 거리 · `tags`·`theme` 매칭으로 점수화(`score`) 후 상위 N배 슬라이스.
- 사용자 프롬프트: 라인당 `[i] song_id="…" | 제목 — 가수 | 언어태그(중/한) | 등급 | score=… meta` 로 후보를 나열하고, "regular 주는 정확히 `songsPerWeek` 곡, `midWeek/finalWeek` 는 곡 없음, 1주차는 orientation 타입" 을 명시.
- Tool: `return_outline({ weeks: Array<{ week, type:"orientation"|"regular"|"midterm"|"final", title, songs?: Array<{title,artist,reason,song_id,language}> }> })`. `regular` 이 아닌 주는 `songs` 금지.
- 후처리: `cnUsed/krUsed` 를 다시 세고 정확히 매칭되지 않으면 로그로 경고(강제 재실행 없음 — 실운영에서 GPT 가 대부분 맞춤). `lesson_plans.outline` 저장, 기존 `lesson_weeks` 를 upsert(`week_number, title, week_type, content:{songs, description?}, song_ids, is_generated:false`).

#### (b) `generate_week` (라우터)

- 입력: `{ lesson_plan_id, week_number, frame_type?, keywords? }`.
- 주차 조회 후 `effectiveFrame = frame_type ?? (week_type === "regular" ? "standard" : week_type)`.
- 분기:
  - `week_type === "orientation"` → `handleGenerateOrientation` 로 위임(전용 프롬프트).
  - `effectiveFrame === "standard"` → **7-섹션 표준 프레임** 생성(아래 (c)).
  - 그 외(`review`, 시험 주차의 legacy) → `buildReviewPrompt` + `reviewToolSchema` 로 **모듈 배열** 반환 후 `buildReviewModules` 로 정규화 → `content.modules` 에 저장.
- 저장 규약: 항상 `content: { ...prev, framework|modules, songs: validSongs }`, `is_generated: true`, `song_ids = validSongs.filter(s=>s.song_id).map(s=>s.song_id)` 로 UPDATE.

#### (c) 표준 프레임 (regular 주)

- **TOPIK 백필**: `plan.level.toUpperCase().startsWith("TOPIK")` 면 `ensureKoreanAnalyses(sb, songIds)` 호출 — 각 곡의 `song_analyses.word_list` 를 훑어 `topik_level` 필드나 `hsk_level` 라벨이 `TOPIK*` 이 아닌 경우 `analyze-song` 를 `learn_language:"korean", force:true` 로 재호출. **실패해도 fatal 아님** (try/catch + `console.warn`), 이유는 T3a 의 클라이언트 백필과 동일한 안전망을 서버측에도 두기 위함.
- 곡 컨텍스트 조립:
  - `songs` 에서 `id, title, artist, language, hsk_level, tags, lyrics_raw` 조회.
  - `song_analyses` 에서 `lyrics_with_pinyin` 조회 → 상위 40줄을 `중문 (한글)` 형식으로 붙여 **선곡 이유 인용용 전체 가사**를 만든다. 분석이 없으면 `lyrics_raw` 1500자로 폴백. 둘 다 없으면 프롬프트에 "(가사 데이터 없음)" 명시 → GPT 에게 `reason_lyric_quote` 를 **빈 문자열로** 두라고 지시.
  - 짧은 발췌 8줄은 별도로 `핵심 가사 발췌` 라인에 유지(레거시 필드 호환).
- 다음 주 정보: `week_number + 1` 을 `maybeSingle` 로 조회해 `nextWeek.title / songs` 를 프롬프트에 삽입(있을 때만).
- 시스템 메시지: "당신은 중국어 노래 기반 교육 플랫폼의 강의안 콘텐츠 생성 전문가입니다." + 5원칙 + `${levelGuide}`.
- 사용자 메시지 필수 섹션 순서: `## 입력 정보` → `## 난이도 지침` → 노래 블록(위 발췌·전체 가사) → 다음 주 정보 → `## 출력 요구사항` → `${langDirective}`.
- Tool: `return_framework(frameworkToolSchema)` — 반환 스키마 필드(모두 구현):
  - `lesson_intro { topic, goals:[3], duration, level, target_learner }`
  - `culture_background { text, image_keywords:[3], keywords:[{cn,kr}]*5 }`
  - `songs_reasons: [{ reason_long, reason_lyric_quote }]` — **입력 노래 순서 그대로**. 후처리 단계에서 index 로 `validSongs` 와 병합해 `songs` 필드로 밀어넣고 `songs_reasons` 는 삭제.
  - `vocabulary: [{cn, pinyin, kr}]*6`
  - `activity { image_questions:[3], image_questions_keywords:[2~3], dialog:[{speaker, cn, kr}]*4, discussion:[2] }`
  - `homework: [{ type:"기본"|"선택", title, desc }]*2`
  - `next_week { title, songs, preview }`
- 후처리:
  - `framework.vocabulary_extra = []`, `framework.homework_extra = []`, `framework.activity.image_questions_extra = []`, `framework.activity.discussion_extra = []` 로 **초기화**(부분 재생성 액션이 여기에 이어붙임).
  - **Pixabay 자동 삽입**: `collectUsedImageIds(planId, thisWeekId)` 로 같은 강의안 내 다른 주차에서 이미 사용된 이미지 id 를 모아 dedup. `culture_background.image_keywords` 로 1장, `activity.image_questions_keywords` 로 4장 검색해 `image_assets: [{id, url, thumb, ...}]` 에 저장. 실패는 try/catch 로 삼킨다(카드 UI 에서 재검색 가능).

#### (d) `handleGenerateOrientation` (1주차 전용)

- 데모 곡: 2주차의 첫 곡을 `songs`/DB 에서 조회해 `video_id / youtube_url / hsk_level / language` 를 채워 프롬프트에 `demoSong` 로 주입.
- Tool: `return_orientation` → `{ overview, goals, schedule_preview, icebreaking:{questions,examples}, tools_intro, next_week:{title,songs,preview} }`.
- 저장: `content.framework` 에만 넣고 `content.modules` 는 건드리지 않는다.

#### (e) `generate_exam_review` / `generate_exam_examples`

- 대상: `week_type ∈ {midterm, final}`.
- 범위: **midterm** 은 2주 ~ (midterm-1), **final** 은 2주 ~ (final-1) 전체(중간 주차는 `type === "regular"` 필터로 자연 제외).
- 프롬프트 컨텍스트: 각 주차의 `content.framework` (theme / vocab / dialog / homework) 요약과 `song_ids` 를 사용한 곡 목록.
- Tool `return_exam_review` 필수 필드: `overview` (2-3문장), `weekly_review:[{ week, theme_kr, theme_cn, summary, keywords:[{cn,kr}]*4 }]` (모든 대상 주차), `key_songs`, `key_vocabulary`, `key_patterns`, `discussion_prep`. 저장은 `content.framework` 에 병합.
- `generate_exam_examples` 는 위 결과에 `examples:[{prompt, model_answer, points}]` 를 추가하는 후속 액션. 클라이언트(`ExamReviewFrameworkView`) 가 이전 결과가 있으면 이 액션만 재호출.

#### (f) 부분 재생성 액션들

`add_vocab / add_questions / add_discussion / add_homework / regen_dialog` 는 공통 패턴을 따른다:

- 입력 `{ week_id }` 만 받고 서버에서 `lesson_weeks.content.framework` 를 로드.
- 해당 카드의 기존 항목 + `*_extra` 항목을 프롬프트에 붙이고 "**아래 목록과 겹치지 않는 새 항목 N개**" 를 요구.
- `regen_dialog` 만 예외적으로 `activity.dialog` 를 **덮어쓰기**(4줄 재생성), 나머지는 반환값을 `vocabulary_extra` / `activity.image_questions_extra` / `activity.discussion_extra` / `homework_extra` 로 **append**.
- 저장 후 `{ framework: newFw }` 반환. 클라이언트는 이 값으로 로컬 state 를 교체해 즉시 반영.

#### (g) `ask_assistant`

- 입력 `{ week_id, question }`. 서버에서 주차의 `framework.lesson_intro` 로 `topic, level, songLine` 컨텍스트를 만든다.
- 시스템: "당신은 한국 학생을 위한 중국어 노래 교육 어시스턴트입니다. 친절하고 간결하게 한국어로 2-3문장 답변하세요."
- 사용자: 위 컨텍스트 요약 + 질문.
- Tool 미사용 — 자유 텍스트 응답 `{ reply: string }`.

### 2.5 클라이언트 래퍼(그대로 재현)

- **T3a 의 `LessonPlanDetail`** → `handleGenerateWeek(weekNum)` 와 `handleGenerateAll` 이 `action: "generate_week"` 를 호출. 전체 생성 루프는 T3a 규정을 따른다.
- **`WeekDetailView.handleGenerateDetail(params?)`** → 같은 `generate_week` 를 `frame_type` / `keywords` 를 실어 재호출 (재생성 다이얼로그 결과 반영).
- **`WeeklyFrameworkView` 카드 액션 버튼**:
  - `+ 어휘 추가` → `add_vocab`
  - `+ 질문 추가` → `add_questions`
  - `+ 토론 추가` → `add_discussion`
  - `+ 과제 추가` → `add_homework`
  - `회화 다시 만들기` → `regen_dialog`
  - 각 버튼은 `callAction(action, setBusy)` 로 로딩 스피너 + `onUpdate(data.framework)` 로컬 반영. 실패시 `toast({ variant: "destructive" })`.
- **`FloatingAIChat`** (주차 우측 하단 플로팅) → `ask_assistant` 를 호출해 `reply` 를 채팅 스레드에 push. 대화 내역은 로컬 state 만 유지(영속화 없음).
- **`ExamReviewFrameworkView`** → 진입 시 `content.framework` 가 없으면 `generateReview(true)` 자동 호출(silent), 예제 버튼 클릭 시 `generateExamples`. 두 함수 모두 429/402 를 toast 로 노출.

### 2.6 에러 · 사용자 노출 규약

- Toast 문구(모두 한국어, T3a 와 동일 어투 유지):
  - 성공: `"${N}주차 내용이 생성되었습니다!"` / `"전체 생성 완료! (N주차)"` / `"업데이트되었습니다"`.
  - 실패: `"생성 실패"` (description = `error.message`), `variant: "destructive"`.
  - 429 응답 body 의 `error` 문자열을 그대로 description 으로 노출("요청이 너무 많습니다.").
  - 402 는 "크레딧이 부족합니다." 를 그대로 노출하고 상단에 결제 안내(존재하는 경우) 링크 유지.
- **부분 실패 재개**는 서버가 아니라 T3a 의 클라이언트 루프가 담당(주차 단위 재시도). 서버는 개별 요청만 원자적으로 처리한다.

### 2.7 보안 · 배포

- `verify_jwt = false` 기본값을 유지(플랫폼 표준). 함수 내부는 서비스 롤로 `lesson_plans / lesson_weeks / songs / song_analyses` 를 직접 읽고 쓴다.
- `OPENAI_API_KEY` 는 Lovable Cloud 시크릿에서만 읽는다(클라이언트로 절대 노출 금지).
- SQL 은 항상 파라미터 바인딩된 supabase-js 체이닝만 사용. `execute_sql` 류 raw SQL 금지.

---

## 3. Examples — 요청/응답 스냅샷

### 3.1 표준 주차 생성

Request (from `LessonPlanDetail.handleGenerateWeek(3)`):

```json
{ "action": "generate_week", "lesson_plan_id": "…", "week_number": 3 }
```

Response (200):

```json
{
  "content": {
    "songs": [
      { "song_id": "…", "title": "月亮代表我的心", "artist": "邓丽君", "language": "chinese",
        "hsk_level": "HSK3", "reason": "…", "reason_long": "이 노래는 …「你问我爱你有多深」이라는 구절처럼 …",
        "reason_lyric_quote": "你问我爱你有多深" }
    ],
    "framework": {
      "lesson_intro": { "topic": "사랑의 은유", "goals": ["…","…","…"], "duration": "90분", "level": "HSK3", "target_learner": "한국인 대학생" },
      "culture_background": { "text": "…", "image_keywords": ["…"], "keywords": [{"cn":"月亮","kr":"달"}, …], "image_assets": [{"id": 12345, "url": "…", "thumb": "…"}] },
      "vocabulary": [{"cn":"深","pinyin":"shēn","kr":"깊다"}, …],
      "vocabulary_extra": [],
      "activity": {
        "image_questions": ["…","…","…"], "image_questions_keywords": ["月亮 夜空","中秋 团圆"],
        "dialog": [{"speaker":"A","cn":"…","kr":"…"}, …],
        "discussion": ["…","…"], "image_questions_extra": [], "discussion_extra": [],
        "image_assets": [{"id":…,"url":"…","thumb":"…"}, … × 4]
      },
      "homework": [{"type":"기본","title":"…","desc":"…"}, {"type":"선택","title":"…","desc":"…"}],
      "homework_extra": [],
      "next_week": { "title": "이별", "songs": "后来 - 刘若英", "preview": "…" }
    }
  }
}
```

### 3.2 부분 재생성 (어휘 추가)

```json
// req
{ "action": "add_vocab", "week_id": "…" }
// res
{ "framework": { "…": "…", "vocabulary_extra": [{"cn":"温柔","pinyin":"wēnróu","kr":"부드럽다"}, …] } }
```

### 3.3 요금 한도 초과 (429)

```json
{ "error": "요청이 너무 많습니다." }
```

클라이언트: `toast({ title: "생성 실패", description: "요청이 너무 많습니다.", variant: "destructive" })`, 스피너 해제.

### 3.4 어시스턴트

```json
// req
{ "action": "ask_assistant", "week_id": "…", "question": "이 노래 대신 다른 곡을 쓰면 어떤 걸 추천하세요?" }
// res
{ "reply": "「甜蜜蜜」도 같은 시대·가수여서 좋습니다. 학생들이 아는 확률이 높아 도입이 쉬워요." }
```

---

## 4. Context — 배경/전제

- 데이터 모델은 T3a 와 동일: `lesson_plans (outline jsonb, output_lang, level, …)` + `lesson_weeks (week_number, week_type, title, content jsonb, song_ids uuid[], is_generated bool)`. 이 문서는 `content.framework` / `content.modules` 두 형태의 페이로드를 모두 다룬다.
- 곡 데이터는 `songs` + `song_analyses` 를 참조하며, **TOPIK 코스일 경우 `analyze-song` 이 `learn_language:"korean"` 모드로 재분석**되어 `word_list[*].topik_level` 이 채워져 있어야 학습자 등급에 맞는 어휘·문법 카드가 나온다. 서버·클라이언트 양쪽에 백필 로직을 둔다(서버 = 2.4.c, 클라이언트 = T3a `ensureKoreanAnalysisForWeek`).
- 이미지 자동 삽입은 Pixabay 를 사용하며 `image_assets` 는 `PixabayImageGrid` 에서 그대로 렌더된다. 강의안 내 중복은 `collectUsedImageIds` 로 방지한다.
- 어시스턴트 대화는 영속화되지 않으며, 창을 닫으면 사라진다. 필요 시 후속 이터레이션에서 `lesson_ai_chats` 등의 테이블로 확장할 수 있다.

---

## 5. Acceptance — 무엇이 되면 이 프롬프트가 성공인가

1. `POST /functions/v1/generate-lesson-plan { action: "generate_week", lesson_plan_id, week_number }` 가 표준 regular 주차에 대해 위 3.1 스키마와 **동일한 최상위 필드**(`songs / framework.lesson_intro / culture_background / vocabulary / activity / homework / next_week` + `*_extra: []`) 를 200 으로 반환한다.
2. `plan.level` 이 `HSK1` / `HSK6` / `TOPIK2` / `TOPIK6` 각각에 대해 `vocabulary` 어휘 난이도·`activity.dialog` 문장 길이·`activity.discussion` 질문 깊이·`homework.desc` 부담 수준이 § 2.3 tier 표에 맞춰 **눈에 띄게 달라진다** (수동 스팟체크).
3. `plan.output_lang === "zh"` 인 강의안에서 반환된 자연어 필드(설명·목표·질문·과제 desc 등)가 **모두 간체 중국어**이며, `songs[i].title/artist`, `reason_lyric_quote` 는 원어를 유지한다.
4. 오리엔테이션(1주차) 요청은 `framework` 에 `overview / goals / schedule_preview / icebreaking / tools_intro / next_week` 필드가 존재하고 `modules` 는 만들지 않는다.
5. midterm/final 주차에 `generate_exam_review` 를 호출하면 대상 범위(2주 ~ 시험 직전, regular 만)의 **모든** 주차가 `weekly_review` 에 등장한다. 이후 `generate_exam_examples` 를 호출하면 기존 `weekly_review` 를 유지한 채 `examples` 필드만 추가된다.
6. 표준 프레임 생성 직후 `culture_background.image_assets.length === 1`, `activity.image_assets.length === 4`, 두 세트 사이 그리고 같은 강의안의 다른 주차와 **id 중복이 없다**. Pixabay 실패 시에는 필드가 없거나 빈 배열이지만 200 으로 정상 반환된다.
7. `add_vocab / add_questions / add_discussion / add_homework` 는 기존 `vocabulary / activity.image_questions / activity.discussion / homework` 를 **건드리지 않고** 각 `*_extra` 배열을 append 한다. `regen_dialog` 는 `activity.dialog` 4줄을 완전히 교체하고 다른 필드는 유지한다.
8. `ask_assistant` 는 tool_call 없이 순수 문자열 `reply` 를 반환하며, 응답 길이가 대략 2-3문장에 수렴한다.
9. GPT 429/402 오류는 각각 `{ error: "요청이 너무 많습니다." }` (429) / `{ error: "크레딧이 부족합니다." }` (402) 로 그대로 흘러와 클라이언트 toast 에 노출된다. 그 외 오류는 500 + `{ error: message }` 이며 스피너가 반드시 해제된다.
10. `plan.level` 이 `TOPIK*` 인 표준 주차 생성은 실행 로그에 `[ensureKoreanAnalyses]` 흔적을 남기고, 백필이 실패해도 최종 응답은 200 이다(치명 오류 아님).
