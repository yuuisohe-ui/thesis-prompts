# B3 · Suno 창작곡 생성 전 과정 백엔드 재현 프롬프트

> **본 프롬프트는 B 시리즈(B1–B3)의 3/3 — 백엔드(Supabase Edge Functions) 재현 자료.**
> **적용 대상**: `supabase/functions/sg-generate-lyrics`, `sg-suno-generate`, `sg-suno-poll`, `sg-suno-lyrics`, `sg-pixabay-videos`, `sg-finalize-song`, `sg-repair-aligned`
> **보조 모듈**: `src/lib/song-generator/suno-style.ts`(한국어 UI 라벨 → Suno 영문 태그 매핑), `src/lib/song-generator/suno-client.ts`(브라우저/Deno 공용 타입·단발 호출 래퍼)
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**
> **범위 경계**: YouTube 검색·자막 취득·가사 정규화는 B1, 확보된 가사의 학습 데이터 변환은 B2, 배경영상 합성 콜백(`video-trigger`/`video-callback`)은 미작성 범위. 본 문서는 "주제 입력 → 가사 생성 → Suno 작곡 → 상태 폴링 → 단어 단위 타임스탬프 → 배경영상 후보 → 아카이브 저장" 사슬만 책임진다.

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* https://platform.openai.com/docs/guides/prompt-engineering (Retrieved July 12, 2026)
2. **Lovable. (n.d.).** *Prompting best practices.* https://docs.lovable.dev/prompting/prompting-one (Retrieved July 12, 2026)
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.
5. **Fielding, R. T. (2000).** *Architectural Styles and the Design of Network-based Software Architectures*, Ch. 5(무상태 제약) — 장시간 작업을 서버 루프가 아닌 클라이언트 폴링으로 분해하는 근거.

---

## ① Identity (신원)

당신은 **Supabase Edge Functions(Deno) 기반 AI 작곡 파이프라인 엔지니어**다. 담당 범위는 다음 3계층이다.

1. **작사 계층** — OpenAI Chat Completions(`gpt-4o-mini`)로 주제·스타일·템포·감정·언어·난이도·줄 수 제약을 만족하는 가사 본문과 메타 3줄(TITLE / STYLE_HINT / VIDEO_KEYWORDS)을 산출한다.
2. **작곡 계층** — Suno API(`https://api.sunoapi.org/api/v1`)에 대해 **한 함수 = 한 HTTP 호출** 원칙을 지킨다. 생성(`/generate`), 상태 조회(`/generate/record-info`), 타임스탬프 가사(`/generate/get-timestamped-lyrics`)를 각각 독립 함수로 분리하고, **Edge Function 내부에서 sleep-loop 폴링을 절대 하지 않는다**(무상태·짧은 실행시간 제약).
3. **영속 계층** — 완성된 오디오를 Suno CDN에서 내려받아 `song-audio` 버킷으로 재업로드하고, Suno 원시 `alignedWords`를 **줄 단위로 병합**한 뒤 `songs` 행을 만들고, 분석·문화태그 파이프라인(B1/B2)을 fire-and-forget으로 기동한다.

**전제**: 이미 존재하는 Lovable Cloud 프로젝트에 위 7개 함수를 추가한다. 프런트엔드 화면(`/song-generator` 5-step 위저드)은 별도 문서에서 다루며, 본 문서는 그 화면이 호출하는 계약(요청/응답 스키마)까지만 규정한다.

---

## ② Instructions (지시)

### 2.1 산출물 (Prompt by Component, Not Page)

| # | 파일 | 책임 | 외부 의존 | JWT |
|---|---|---|---|---|
| 1 | `sg-generate-lyrics/index.ts` | 가사 + TITLE/STYLE_HINT/VIDEO_KEYWORDS 생성·파싱 | `OPENAI_API_KEY` | verify_jwt = false |
| 2 | `sg-suno-generate/index.ts` | Suno 작곡 태스크 생성 → `taskId` 반환 | `SUNO_API_KEY`, `SUNO_API_KEY_2` | verify_jwt = false |
| 3 | `sg-suno-poll/index.ts` | 단발 상태 조회 → 정규화된 status/track | 동상 | verify_jwt = false |
| 4 | `sg-suno-lyrics/index.ts` | 단어 단위 정렬(alignedWords) + waveform | 동상 | verify_jwt = false |
| 5 | `sg-pixabay-videos/index.ts` | 배경영상 후보 URL 목록(최대 20) | `PIXABAY_API_KEY` | verify_jwt = false |
| 6 | `sg-finalize-song/index.ts` | 오디오 영속화 + `songs` insert + 후속 분석 기동 | Service Role, Storage | **JWT 필수(코드 내 검증)** |
| 7 | `sg-repair-aligned/index.ts` | 과거 저장분의 원시 정렬 데이터 1회성 재병합·재분석 | Service Role | **JWT 필수(코드 내 검증)** |

### 2.2 원자적 규칙 (Speak Atomic)

**[A. 공통 인프라]**

- A1. 모든 함수는 `https://deno.land/std@0.177.0/http/server.ts` 의 `serve()` 로 시작하고, 최상단에 동일한 `corsHeaders` 상수를 둔다. `Access-Control-Allow-Origin: *`, 허용 헤더에 `authorization, x-client-info, apikey, content-type` 및 `x-supabase-client-platform/-version`, `x-supabase-client-runtime/-version` 6종을 모두 포함한다(Supabase JS v2가 자동 부착하므로 누락 시 프리플라이트 실패).
- A2. `OPTIONS` 요청은 본문 없이 `corsHeaders` 만 반환한다.
- A3. 모든 성공·실패 응답에 `corsHeaders` 를 병합한다. 실패는 `{ error: string }` 형태로 통일한다.
- A4. **크레딧 소진 예외**: 사용자가 재시도해도 무의미한 "잔액 부족" 계열은 HTTP 500이 아니라 **200 + `{ error: "…크레딧이 부족합니다…" }`** 로 내려 프런트에서 토스트로 안내되게 한다(`sg-suno-generate` 한정).

**[B. 이중 API 키 페일오버 — Suno 3개 함수 공통 `sunoCall()`]**

- B1. `SUNO_API_KEY`, `SUNO_API_KEY_2` 를 순서대로 배열에 담는다. 둘 다 없으면 `SUNO_API_KEY missing` 예외.
- B2. 키를 순회하며 호출한다. `res.ok` 면 즉시 `{ text, status }` 반환하고 `[suno] key#N ok <path>` 로그를 남긴다.
- B3. 실패 시 `[suno] key#N fail <status> <path> <msg>` 경고를 남기고, **크레딧성 오류일 때만** 다음 키로 넘어간다. 그 외 오류(4xx 파라미터 오류 등)는 즉시 중단한다 — 키를 바꿔도 결과가 같기 때문.
- B4. 크레딧성 판정: `status ∈ {401, 402, 429}` 이거나 본문/`msg` 소문자에 `credit|insufficient|quota|balance|unauthorized|invalid api key` 정규식이 매치될 때.
- B5. 최종 실패 메시지는 한국어로 변환한다. 429/402/잔액문자열 → `"Suno 크레딧이 부족합니다. 충전 후 다시 시도해 주세요"`, 401 → `"Suno API Key가 유효하지 않습니다"`, 그 외 → `"Suno <status>: <본문 200자>"`.

**[C. `sg-generate-lyrics` — 작사]**

- C1. 입력: `{ topic, styleStr, tempoStr, moodStr, language, lengthStr, targetLines, difficultyBlock, model? }`. `topic` 없으면 `topic required` 예외.
- C2. 모델 기본값 `gpt-4o-mini`(요청으로 오버라이드 가능). 엔드포인트 `https://api.openai.com/v1/chat/completions`.
- C3. **system 프롬프트를 언어별로 분기**한다.
  - `한국어` → `"당신은 전문 한국어 작사가입니다. 라임 있고 입에 붙으며 노래에 적합한 가사를 잘 씁니다. 사용자가 지정한 언어, 난이도, 줄 수 요구를 엄격히 지킵니다."`
  - `중국어` → 동일 문장에서 "전문 중국어 작사가"
  - 그 외(이중언어) → "한·중 이중언어 작사가"
- C4. **언어 순수성 제약**을 user 프롬프트에 삽입한다.
  - 한국어: 전체 100% 한글, 한자·병음·가나 금지.
  - 중국어: 전체 100% 간체자, 한글·가나·영어 금지.
  - 이중언어: **홀수 단락 100% 중국어 간체, 짝수 단락 100% 한글**(단락 엄격 교차).
- C5. **줄 길이 상한**(학습 난이도 통제의 핵심): 한국어 줄 = 한글 최대 13자(공백 3개 이내, 끝 문장부호 1개 허용), 중국어 줄 = 한자 최대 10자(쉼표·마침표 1개 허용, 영단어·병음 금지). 이중언어는 각 언어 줄에 그대로 적용.
- C6. **줄 수는 정확히 `targetLines`**(기본 16). 빈 줄·단락 구분 줄은 카운트하지 않는다.
- C7. **단락 태그 출력 금지** — `[Verse]`, `[Chorus]`, `(후렴)`, `【…】` 등.
- C8. 난이도 블록은 프런트가 만들어 보낸 문장을 그대로 삽입한다: `"학습자 수준: <등급>. 이 등급의 어휘·문법을 위주로 작성하되, 자연스러움을 위해 가끔 더 높은 등급의 단어가 1-2개 등장하는 것은 허용합니다."`(HSK 1–6 / TOPIK 1–6 문자열). 등급 미선택 시 빈 문자열.
- C9. 응답 말미에 반드시 3줄을 별도로 요구한다.
  - `TITLE:` — 간결한 곡명 3–10자. 언어는 `중국어` 선택 시 중국어, 그 외 한국어.
  - `STYLE_HINT:` — 영어 스타일 힌트(예: `warm female vocals, acoustic guitar`).
  - `VIDEO_KEYWORDS:` — 영어 키워드 3개를 `+` 로 연결(예: `ocean+sunset+waves`).
- C10. **파싱**: 각 줄을 `^\s*(TITLE|STYLE_HINT|VIDEO_KEYWORDS)\s*[:：]` 로 매치해 분리하고 나머지를 가사로 모은다. TITLE은 앞뒤 `《》"'「」『』` 를 제거한다. VIDEO_KEYWORDS는 `[^a-zA-Z0-9+\s-]` 제거 → 공백을 `+` 로 → 중복 `+` 축약 → 양끝 `+` 제거.
- C11. **태그 잔재 제거**(`stripTags`): 한 줄 전체가 `[...]`/`(...)`/`（…）`/`【…】`(내부 40자 이내)인 줄은 삭제하고, 줄 안의 `[…]` 조각도 제거한다. 3줄 이상 연속 개행은 2줄로 축약하고 trim.
- C12. **줄 수 강제(자르기만, 채우지 않음)**: 파싱 후 비어있지 않은 줄 수가 `targetLines` 를 초과하면 앞에서부터 잘라낸다. 부족해도 임의 생성하지 않는다(사용자가 편집 단계에서 보완).
- C13. OpenAI 429 → `"OpenAI 크레딧이 부족하거나 호출 한도 초과"`, 그 외 비정상 → `"OpenAI <status>: <본문 200자>"`.
- C14. 응답: `{ lyricsText, styleHint, title, videoKeywords }`.

**[D. `sg-suno-generate` — 작곡 태스크 생성]**

- D1. 요청 본문을 그대로 Suno `/generate` 로 전달하되 서버에서 3개 필드를 강제·보정한다: `callBackUrl ||= "https://example.com/noop"`(Suno 필수 필드지만 콜백은 사용하지 않음 — 폴링 방식), `customMode = true`, `model ||= "V4_5PLUS"`.
- D2. `taskId` 추출은 5중 폴백: `data.taskId → data.task_id → data.id → taskId → task_id → id`.
- D3. `taskId` 가 비었고 `msg` 에 `credit|insufficient|top up|quota|balance` 가 매치되면 **200 + 한국어 충전 안내**(`"Suno 크레딧이 부족합니다. sunoapi.org에서 충전 후 다시 시도해 주세요."`)를 반환한다. 그 외에는 `Suno did not return taskId. msg=…` 예외.
- D4. 성공 응답: `{ taskId }`.
- D5. catch 단계에서도 메시지에 `credit|insufficient|top up|크레딧` 이 있으면 status 200으로 내린다.

**[E. `sg-suno-poll` — 단발 상태 조회]**

- E1. `taskId` 는 쿼리스트링 우선, 없으면 JSON 본문에서 읽는다(GET/POST 양쪽 허용). 없으면 `taskId required`.
- E2. `GET /generate/record-info?taskId=…` 1회 호출. **함수 내부 반복·대기 금지.**
- E3. 트랙 추출 3중 폴백: `data.response.sunoData[0] → data.response.data[0] → data.sunoData[0]`.
- E4. 오디오 URL 4중 폴백: `audioUrl → audio_url → streamAudioUrl → stream_audio_url`.
- E5. `isTerminalSuccess = (status === "SUCCESS" && audioUrl 존재)`. `isTerminalFailure = status ∈ {FAILED, ERROR, CREATE_TASK_FAILED}`. 중간 상태(`PENDING`, `TEXT_SUCCESS`, `FIRST_SUCCESS`)는 둘 다 false.
- E6. 응답: `{ status, isTerminalSuccess, isTerminalFailure, track: { id, audioUrl, duration } | null }`.

**[F. `sg-suno-lyrics` — 단어 단위 정렬]**

- F1. 입력 `{ taskId, audioId }`(둘 다 필수). `POST /generate/get-timestamped-lyrics`.
- F2. 응답의 `alignedWords` 를 **camelCase로 정규화**한다: `startS ?? start_s`, `endS ?? end_s`, `pAlign ?? p_align ?? palign`.
- F3. 응답: `{ alignedWords: [{ word, startS, endS, pAlign }], waveformData: number[] }`.

**[G. `sg-pixabay-videos` — 배경영상 후보]**

- G1. 입력은 GET 쿼리(`q`, `minDuration`, `limit`) 또는 POST 본문 모두 허용. 기본값 `minDuration = 4`, `limit = 20`.
- G2. `q` 의 `+` 를 공백으로 바꾸고 다중 공백을 축약한다. 비어 있으면 `"nature sky sunlight"`.
- G3. Pixabay `GET https://pixabay.com/api/videos/` 파라미터: `video_type=film`, `safesearch=true`, `per_page=20`.
- G4. `duration >= minDuration` 이고 `videos.medium.url` 이 있는 항목만 채택.
- G5. **결과가 3개 미만이면 폴백 질의** `"nature sky sunlight people city journey"` 를 추가 수행해 합집합을 만든다(빈 배경 방지).
- G6. 중복 제거 → **Fisher–Yates 셔플** → `limit` 개로 절단. 응답 `{ bgVideoList: string[] }`. 실패 시에도 `{ error, bgVideoList: [] }` 로 배열 계약을 유지한다.

**[H. `sg-finalize-song` — 아카이브 영속화]**

- H1. **인증 필수**: `Authorization: Bearer …` 헤더가 없으면 401. anon 키 클라이언트로 `auth.getClaims(token)` 를 호출해 `claims.sub` 를 얻고, 없으면 401.
- H2. 데이터 조작은 Service Role 클라이언트(`admin`)로 수행한다.
- H3. 필수 입력 검증: `title`, `audioUrl`, `lyricsText` 중 하나라도 없으면 예외.
- H4. **UI 언어 → DB 언어 매핑**: `한국어→korean`, `중국어→chinese`, `한·중 이중언어→bilingual`, 그 외에는 `learnLanguage` 로 추정(`korean` 아니면 `chinese`).
- H5. **소유권 규칙**: `user_roles` 에 해당 사용자의 `admin` 행이 있으면 `owner_id = NULL`(공용 자료), 아니면 `owner_id = userId`. `user_id` 에는 항상 실제 생성자를 기록한다.
- H6. `songs` insert 페이로드 고정값: `artist = "AI 생성"`, `lyrics_source = "ai_generated"`, `source = "ai_generated"`, `has_subtitles = true`. 가변값: `language`, `lyrics_raw`, `aligned_words`, `bg_video_list`, `video_keywords`, `hsk_level = hskLevel || topikLevel || null`, `youtube_url`, `video_id`, `video_status = youtubeUrl ? "done" : "idle"`.
- H7. **오디오 영속화**: `sourceAudioUrl` 을 fetch → `Uint8Array` → `song-audio` 버킷 `"<songId>/audio.mp3"` 에 `contentType: audio/mpeg`, `upsert: true` 로 업로드 → `getPublicUrl` 결과를 `audio_url` 에 저장. **실패해도 중단하지 않고** 원본 Suno URL을 `audio_url` 로 저장한 뒤 경고 로그만 남긴다(Suno CDN 만료 위험을 감수하되 사용자 작업은 보존).
- H8. **정렬 데이터 줄 단위 병합(`mergeAlignedWords`)**: Suno 원시 배열은 쉼표 지점에서 한 줄이 여러 조각으로 쪼개진다. 조각을 누적하다가 **텍스트가 `\n` 으로 끝날 때 하나의 줄로 확정**한다. 줄의 `startS` 는 첫 조각 시작, `endS` 는 마지막 조각 끝, `pAlign` 은 마지막 관측값. 남은 버퍼는 마지막 줄로 push. 시간값은 `startS → start_s → start` 순 폴백, 숫자가 아니면 0.
- H9. 병합 결과가 1개 이상이면 `aligned_words` 를 병합본으로 **덮어쓰고**, 병합본에서 파생한 텍스트(각 줄 끝 개행 제거 → 빈 줄 제외 → `\n` 결합)를 `lyrics_raw` 로 저장한다. → **"실제로 불린 가사"가 단일 진실원천**이 되어 카드 오픈 시 줄 수와 타임스탬프 수가 1:1로 일치한다(수동 "수정" 불필요).
- H10. **후속 파이프라인 fire-and-forget**(await 금지, 실패는 warn 로그만):
  - `analyze-song` — `{ song_id, custom_lyrics: 병합가사 || 원본, language: dbLang, learn_language, force: true, user_id }`. **YouTube 수입곡과 동일한 분석 경로를 재사용**한다.
  - `tag-song-culture` — `{ song_id, force: true }`.
  - 두 호출 모두 Service Role 키를 Bearer로 사용한다.
- H11. 응답: `{ song_id, suno_task_id }`.

**[I. `sg-repair-aligned` — 과거 데이터 1회성 보정]**

- I1. 인증·클라이언트 구성은 H1–H2와 동일. 입력 `{ song_id }`.
- I2. `songs` 에서 `id, owner_id, language, aligned_words, source` 를 읽는다.
- I3. **이미 병합됨 판정**: 모든 조각의 텍스트가 `\n` 으로 끝나면 병합 불필요로 간주한다.
- I4. 미병합이면 H8과 동일 알고리즘으로 병합 후 `aligned_words` 를 갱신한다.
- I5. 기존 `song_analyses` 행을 **삭제**한 뒤 재분석 함수를 호출해 깨끗하게 교체한다(부분 갱신 금지 — 줄 수 불일치 잔재 방지).
- I6. 응답 `{ ok: true, line_count }`.

### 2.3 강제 제약

- **폴링은 클라이언트 책임**: Edge Function 안에서 `sleep + loop` 로 Suno 완료를 기다리지 않는다. 프런트가 5초 간격 × 최대 120회(≈10분) 로 `sg-suno-poll` 을 호출한다. `pErr`(네트워크성 오류)는 무시하고 다음 회차로 넘어가되, 응답 본문의 `error` 는 즉시 중단 사유로 처리한다.
- **작업 복구**: 프런트는 `taskId` 와 폼 상태를 `localStorage["sg:active-job"]` 에 저장하고, **30분 이내**면 새로고침 후에도 폴링을 이어받는다. 성공·실패 시 반드시 키를 삭제한다.
- **비밀키 노출 금지**: `SUNO_API_KEY*`, `OPENAI_API_KEY`, `PIXABAY_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY` 는 Edge Function 안에서만 읽는다. 프런트는 언제나 `supabase.functions.invoke()` 만 사용한다.
- **`song-audio` 버킷은 public**, `cover-images` 는 private. 오디오 경로는 `"<songId>/audio.mp3"` 로 고정해 곡 삭제 시 prefix 단위 정리가 가능하게 한다.
- **모델 고정**: 작사는 `gpt-4o-mini`, 작곡은 Suno `V4_5PLUS`. 변경 시 줄 길이·언어 순수성 제약을 재검증해야 한다.

---

## ③ Examples (예시)

### 3.1 확정 문자열·계약 표 (Design with Real Content)

| 위치 | 확정 문자열 / 값 |
|---|---|
| Suno Base | `https://api.sunoapi.org/api/v1` |
| 기본 모델 | `V4_5PLUS` (허용: `V3_5 · V4 · V4_5 · V4_5PLUS · V4_5ALL · V5 · V5_5`) |
| 콜백 자리표시자 | `https://example.com/noop` |
| 크레딧 오류 | `Suno 크레딧이 부족합니다. 충전 후 다시 시도해 주세요` |
| 키 오류 | `Suno API Key가 유효하지 않습니다` |
| OpenAI 한도 | `OpenAI 크레딧이 부족하거나 호출 한도 초과` |
| 아티스트 고정값 | `AI 생성` |
| 가사 출처 | `lyrics_source = "ai_generated"`, `source = "ai_generated"` |
| Pixabay 기본 질의 | `nature sky sunlight` / 폴백 `nature sky sunlight people city journey` |
| 스타일 폴백 | `k-pop ballad` |
| 작업 복구 키 | `sg:active-job` (TTL 30분) |
| 폴링 주기·횟수 | 5,000 ms × 120회 |

**한국어 UI 라벨 → Suno 영문 태그 매핑(`suno-style.ts`)**

| 축 | 예시 매핑 |
|---|---|
| 장르(20종) | `포크→folk acoustic, fingerstyle guitar` · `고풍 (중국풍)→chinese traditional, guzheng, erhu, bamboo flute` · `Lo-fi→lofi hip hop, chill, vinyl crackle, mellow` · `시티팝→city pop, 80s japanese pop, glossy synths` · `국풍 힙합→chinese style hip hop, guzheng with trap beat` |
| 템포(6종) | `매우 느림 (60 BPM)→very slow tempo around 60 bpm` … `매우 빠름 (150+ BPM)→very fast tempo above 150 bpm` |
| 감정(14종) | `그리움→nostalgic, longing` · `몽환적→dreamy, shoegaze, reverb heavy` · `웅장함→epic, cinematic` |
| 언어 | `한국어→korean vocals` · `중국어→mandarin chinese vocals` · `한·중 이중언어→bilingual mandarin and korean vocals` |
| 네거티브(3종) | `일렉트로닉 사운드→edm, synth heavy` · `랩→rap` · `헤비메탈→heavy metal, screaming` |
| Suno `language` 코드 | `Korean` · `Chinese` · `Chinese,Korean` |

`buildSunoStyle()` 결합 순서: `styleHint → 장르 → 템포 → 감정 → 언어` 를 `, ` 로 연결하고, 모두 비면 `k-pop ballad`. 사용자가 `기타` 를 고르면 해당 축의 자유 입력 문자열을 그대로 삽입한다.

### 3.2 파이프라인 트리 (Use Prompt Patterns for Layouts)

```
[주제·스타일·템포·감정·언어·난이도·줄 수 입력]
        │
        ▼
  sg-generate-lyrics ──(gpt-4o-mini)──► { lyricsText, title, styleHint, videoKeywords }
        │                                         │
        │  (사용자가 가사 직접 편집 가능)          │
        ▼                                         │
  buildSunoStyle() ── 한국어 라벨 → 영문 태그 문자열
        │
        ▼
  sg-suno-generate ──► { taskId }        ── localStorage["sg:active-job"] 저장
        │
        ▼   (클라이언트 폴링 5s × 120)
  sg-suno-poll ──► PENDING → TEXT_SUCCESS → FIRST_SUCCESS → SUCCESS
        │                                            │
        │                                    { track: { id, audioUrl, duration } }
        ├──────────────► sg-suno-lyrics  ──► alignedWords[] (원시, 조각 분할)
        └──────────────► sg-pixabay-videos ─► bgVideoList[] (셔플·중복 제거)
                              (두 호출은 Promise.all 병렬)
        │
        ▼   [미리듣기: 오디오 + 배경영상 교체 가능]
  sg-finalize-song
        ├─ songs INSERT (owner_id: admin이면 NULL)
        ├─ Suno 오디오 → song-audio 버킷 재업로드 (실패 시 원본 URL 폴백)
        ├─ mergeAlignedWords() → 줄 단위 병합 → aligned_words / lyrics_raw 덮어쓰기
        ├─ analyze-song      (fire-and-forget, B2 파이프라인)
        └─ tag-song-culture  (fire-and-forget, B2 파이프라인)
        │
        ▼
      /songs 로 이동 · 백그라운드 분석 진행 안내 토스트

[유지보수 경로] sg-repair-aligned : 과거 원시 정렬 데이터 → 재병합 → song_analyses 삭제 → 재분석
```

---

## ④ Context (배경)

### 4.1 프로젝트 맥락

「멜로디 클래스」는 노래 기반 중국어/한국어 교육 플랫폼이다. YouTube 수입곡만으로는 (1) 특정 문법·어휘 등급을 겨냥한 곡을 찾기 어렵고, (2) 저작권 범위가 제한되며, (3) 자막 타임스탬프 품질이 들쭉날쭉하다. **AI 창작곡 생성기**는 이 세 문제를 동시에 해결한다 — 교사가 HSK/TOPIK 등급과 줄 수를 지정하면, 저작권이 자유롭고 단어 단위 타임스탬프가 정확한 교재용 곡이 만들어진다.

### 4.2 Lovable Cloud 후경

- 시크릿: `OPENAI_API_KEY`, `SUNO_API_KEY`, `SUNO_API_KEY_2`, `PIXABAY_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_URL`, `SUPABASE_ANON_KEY`.
- 스토리지: `song-audio`(public), `cover-images`(private).
- `supabase/config.toml` — B3의 7개 함수 중 `sg-*` 는 기본 배포 정책을 따르되, 인증이 필요한 `sg-finalize-song`·`sg-repair-aligned` 는 **코드 내부에서 JWT를 직접 검증**한다(서명키 체계 전제).

### 4.3 데이터 계약

**`songs`(본 파이프라인이 쓰는 컬럼)**

| 컬럼 | 타입 | AI 창작곡에서의 값 |
|---|---|---|
| `title` | text | 모델이 만든 TITLE(사용자 편집 가능) |
| `artist` | text | 상수 `AI 생성` |
| `language` | text | `korean` / `chinese` / `bilingual` |
| `lyrics_raw` | text | **Suno 타임스탬프 병합본**(있으면), 없으면 편집된 가사 |
| `lyrics_source` | text | `ai_generated` |
| `source` | text | `ai_generated` |
| `has_subtitles` | boolean | `true` |
| `aligned_words` | jsonb | `[{ word, startS, endS, pAlign }]` — **줄 단위 병합본** |
| `audio_url` | text | `song-audio` 공개 URL(폴백: Suno CDN URL) |
| `bg_video_list` | jsonb | Pixabay medium mp4 URL 배열 |
| `video_keywords` | text | `ocean+sunset+waves` 형태 |
| `hsk_level` | text | `hskLevel || topikLevel || null` |
| `video_status` | text | `youtube_url` 있으면 `done`, 없으면 `idle` |
| `user_id` | uuid | 실제 생성자 |
| `owner_id` | uuid \| null | 관리자 생성 시 `NULL`(공용), 그 외 생성자 |

**RLS 전제**: `songs` 는 `owner_id IS NULL`(공용) 또는 `owner_id = auth.uid()` 인 행만 조회 가능하며, 공용 곡을 개인이 수정하면 `fork_public_item()` 으로 개인 사본이 생성된다(B1·B2와 동일 규약). 본 파이프라인의 insert는 Service Role로 수행되므로 RLS를 우회하지만, **소유권 규칙(H5)을 코드에서 반드시 재현**해야 정책과 일관성이 유지된다.

**`alignedWords` 병합 전/후 예시**

```
[원시] {"word":"朋友的路途"}, {"word":"多漫长\n"}, {"word":"携手并肩共欣赏\n"}
[병합] {"word":"朋友的路途多漫长\n","startS":12.31,"endS":15.04}
       {"word":"携手并肩共欣赏\n","startS":15.20,"endS":18.02}
```

---

## ⑤ Acceptance & Output (검증·산출)

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6 — 정량 임계값)

1. **가사 줄 수**: `targetLines = 16` 요청 시 반환 가사의 비어있지 않은 줄 수가 **16을 초과하지 않는다**(초과분 절단 확인).
2. **언어 순수성**: 한국어 곡 결과에 한자·병음이 0건, 중국어 곡 결과에 한글이 0건이어야 한다(샘플 10곡 기준 위반 ≤ 1곡).
3. **메타 3줄 파싱**: `TITLE`/`STYLE_HINT`/`VIDEO_KEYWORDS` 3개 필드가 모두 비어있지 않고, 가사 본문에 이 3줄이 **잔류하지 않는다**.
4. **태그 제거**: 반환 가사에 `[Verse]`·`(후렴)` 류 단독 라인이 0건.
5. **키 페일오버**: 1번 키를 무효화한 상태에서 `sg-suno-generate` 호출 시 2번 키로 성공하고, 로그에 `key#1 fail` → `key#2 ok` 가 순서대로 남는다.
6. **무상태 폴링**: `sg-suno-poll` 단일 호출 응답 시간이 **3초 이내**이며, 함수 내부에 대기 루프가 없다(코드 검사).
7. **완주율**: 정상 크레딧 상태에서 생성 시작 후 **10분(5s×120회) 이내**에 `isTerminalSuccess = true` 도달.
8. **배경영상**: `sg-pixabay-videos` 응답의 `bgVideoList` 길이가 **3 이상**이며 모든 항목이 `https://` 로 시작하는 mp4 URL.
9. **오디오 영속화**: 저장 완료 후 `songs.audio_url` 이 `song-audio` 버킷 공개 URL이며, 새 탭에서 200으로 재생 가능하다. 업로드 실패 시에도 행은 생성되고 `audio_url` 이 비어있지 않다.
10. **줄/타임스탬프 1:1**: 저장 직후 곡 카드를 열면 `lyrics_raw` 줄 수 = `aligned_words` 길이(±0). 수동 "수정" 없이 하이라이트가 정확히 동기화된다.
11. **후속 분석**: 저장 60초 이내에 `song_analyses` 행과 `song_culture_tags` 행이 각각 1건 이상 생성된다(네트워크 정상 시).
12. **권한**: JWT 없이 `sg-finalize-song` 호출 시 401이며 어떤 행도 생성되지 않는다.
13. **크레딧 UX**: 잔액 0 상태에서 생성 시도 시 HTTP 200 + 한국어 충전 안내 문구가 프런트 토스트로 노출되고, 화면은 가사 단계로 되돌아간다.
14. **작업 복구**: 폴링 도중 새로고침해도 30분 이내면 동일 `taskId` 폴링이 자동 재개되고, 완료 시 `sg:active-job` 키가 삭제된다.
15. **CORS**: 6종 Supabase 클라이언트 헤더를 포함한 프리플라이트가 7개 함수 모두에서 204/200으로 통과한다.

### 5.2 Output Format

- `supabase/functions/sg-generate-lyrics/index.ts`
- `supabase/functions/sg-suno-generate/index.ts`
- `supabase/functions/sg-suno-poll/index.ts`
- `supabase/functions/sg-suno-lyrics/index.ts`
- `supabase/functions/sg-pixabay-videos/index.ts`
- `supabase/functions/sg-finalize-song/index.ts`
- `supabase/functions/sg-repair-aligned/index.ts`
- `src/lib/song-generator/suno-style.ts` (라벨→태그 매핑 · `buildSunoStyle` · `buildNegativeTags` · `buildSunoLanguage`)
- `src/lib/song-generator/suno-client.ts` (브라우저/Deno 공용 타입 및 단발 호출 래퍼)

모든 함수는 TypeScript/Deno 표준 API만 사용하며(Node·Cloudflare 전용 API 금지), 외부 응답 스키마 변형에 대비해 위에 명시한 다중 폴백 경로를 그대로 구현한다.
