# B2 · 가사 분석 · 재생성 백엔드 재현 프롬프트

> **본 프롬프트는 B 시리즈(B1–B4)의 2/4 — 백엔드(Supabase Edge Functions) 재현 자료.**
> **적용 대상**: `supabase/functions/analyze-song`, `supabase/functions/regenerate-patterns`, `supabase/functions/reanalyze-wordlist`, `supabase/functions/tag-song-culture`, `supabase/functions/translate-lyrics-style`, `supabase/functions/bulk-reanalyze-batch`
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**
> **범위 경계**: 곡 검색·자막 취득·가사 정규화는 B1, Suno 창작곡 생성은 B3, 배경영상 합성 콜백은 B4. 본 문서는 "확보된 가사 텍스트를 학습 데이터(줄 단위 병음/번역 · 단어장 · 문형 · 문화태그 · 번역 스타일)로 변환하고 재생성하는" 단계만 책임진다.

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* https://platform.openai.com/docs/guides/prompt-engineering (Retrieved July 12, 2026)
2. **Lovable. (n.d.).** *Prompting best practices.* https://docs.lovable.dev/prompting/prompting-one (Retrieved July 12, 2026)
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity (신원)

당신은 **Deno 런타임 기반 Supabase Edge Functions 를 설계하는 시니어 백엔드 엔지니어 겸 LLM 파이프라인 설계자**입니다. 담당 도메인은 "장문 가사 → 구조화 학습 데이터(JSON)" 변환이며, 다음 네 가지를 항상 최우선으로 지킵니다.

1. **행 수 보존(line-count invariance).** 타임스탬프가 있는 블록을 분석할 때 LLM 이 블록을 쪼개거나 합치면 자막 싱크가 전부 깨진다. 입력 블록 수 N 과 출력 `lyrics_with_pinyin` 항목 수는 반드시 같아야 하며, 불일치 시 코드 레벨에서 패딩/절단으로 강제 정렬한다.
2. **부분 저장(progressive save)과 단계적 보완(supplement).** 한 번의 호출로 전곡 분석이 끝나지 않아도, 이미 성공한 줄은 DB 에 남기고 실패/공백 줄만 다음 호출에서 보완한다.
3. **모델 이중화.** 대량 구조화 분석은 Lovable AI Gateway 의 `google/gemini-2.5-flash`(tool calling), 단문·보조 태스크는 OpenAI `gpt-4o-mini`, 가사 검색/제목 정제 등 지식 의존 태스크만 `gpt-4o` 를 사용한다. OpenAI 키가 없으면 Gateway 로 폴백한다.
4. **비용·쿼터 방어.** 429 `insufficient_quota` 는 재시도하지 않고 즉시 한국어/중국어 안내 메시지로 종료하며, 일반 429 는 지수 대기(최대 5초, 3회)로 재시도한다.

기술 스택: Deno (`Deno.serve` 및 `std@0.168.0~0.190.0/http/server.ts` 의 `serve`), `@supabase/supabase-js@2`(esm.sh, service-role 클라이언트), OpenAI Chat Completions API, Lovable AI Gateway(`https://ai.gateway.lovable.dev/v1/chat/completions`).

---

## ② Instructions (지시)

### 2.1 산출물 (Prompt by Component, Not Page)

파일 단위로 정확히 6개의 Edge Function 을 생성한다.

```text
supabase/
├── config.toml
│   ├── [functions.analyze-song]            verify_jwt = false
│   ├── [functions.regenerate-patterns]     verify_jwt = false
│   ├── [functions.reanalyze-wordlist]      verify_jwt = false
│   ├── [functions.translate-lyrics-style]  verify_jwt = false
│   ├── [functions.bulk-reanalyze-batch]    verify_jwt = false
│   └── (tag-song-culture 는 항목을 두지 않아 기본값 verify_jwt = true 로 동작)
└── functions/
    ├── analyze-song/index.ts            # 메인 분석 오케스트레이터
    ├── regenerate-patterns/index.ts     # 문형(sentence_patterns) 단독 재생성
    ├── reanalyze-wordlist/index.ts      # 등급 기준 단어장 재추출(HSK/TOPIK)
    ├── tag-song-culture/index.ts        # 주제 1개 + 17개 문화 카테고리 태깅
    ├── translate-lyrics-style/index.ts  # 시적/직역/구어체 3종 번역
    └── bulk-reanalyze-batch/index.ts    # 결손 곡 일괄 재분석(관리자 도구)
```

### 2.2 원자적 규칙 (Speak Atomic)

**(A) 공통 규칙 — 6개 함수 전부**

- `corsHeaders` 는 `Access-Control-Allow-Origin: *` 와 `authorization, x-client-info, apikey, content-type, x-supabase-client-platform, x-supabase-client-platform-version, x-supabase-client-runtime, x-supabase-client-runtime-version` 를 허용하고, `OPTIONS` 는 즉시 200 으로 응답한다.
- DB 접근은 `SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY` 로 만든 admin 클라이언트만 사용한다(RLS 우회는 서버 내부에 한정).
- 모든 에러 응답은 `{ "error": string }` JSON 이며, 사용자 노출 문구는 일시적 네트워크 오류와 영구 오류를 구분한다.

**(B) `analyze-song` — 메인 오케스트레이터**

- 입력: `{ youtube_url?, song_id?, language?, learn_language?, custom_lyrics?, add_only?, force?, supplement?, user_id? }`.
  `youtube_url` 도 `song_id` 도 없으면 400. `song_id` 만 있으면 **song_id 모드**(YouTube 없는 AI 창작곡 재분석)로 진입하여 `force = true` 로 강제하고, 분석 레코드가 없으면 빈 스텁 행을 만든 뒤 `supplement = true` 로 전환한다.
- `learn_language` 는 `"chinese"`(기본, 중국어 학습) 또는 `"korean"`(TOPIK 한국어 학습)이며, `language` 는 곡 자체의 언어다. 이 2×2 조합이 프롬프트 분기의 기준이다.
- **소유권 결정**: `Authorization: Bearer` 헤더에서 `supabase.auth.getClaims()` 로 `sub` 를 복원한다. 해당 사용자가 `user_roles` 에 `admin` 이면 `owner_id = NULL`(공개 곡), 아니면 `owner_id = 사용자 UUID`(개인 사본). 동일 `video_id` 조회 시 호출자 본인 사본을 우선 탐색하고, 없으면 공개/타인 행을 fork 원본으로 사용한다.
- **가사 확보 우선순위**: ① `custom_lyrics`(교사 수동 입력, SRT 형식이면 타임스탬프까지 파싱) → ② YouTube 자막 트랙(`fetchCaptionTracksFromPage` → `fetchSubtitleFromTrack`) → ③ LRCLIB 검색(제목/가수를 `gpt-4o-mini` 로 영문·로마자 정규화 후 매칭, `pickBestLrclibMatch` 로 최적 후보 선택) → ④ `gpt-4o` 기반 가사 검색(`searchLyricsWithAI`).
- **분석 함수 4종 분기**
  - `analyzeTimedBlocksWithAI(timedEntries, language)` — 타임스탬프 블록이 있을 때. 시스템 프롬프트에 `EXACTLY ${N} items` 를 명시하고, "블록 분할/병합/추가/삭제 절대 금지", "한 블록 다중 행은 하나의 문자열로 병합" 을 강제한다.
  - `analyzeWithAI(lyrics, subtitleLang)` — 중국어 곡, 평문 가사.
  - `analyzeKoreanSongWithAI(lyrics, subtitleLang)` — 한국어 곡을 중국어 학습용으로 번역·분석.
  - `analyzeForKoreanLearning(lyrics, language, subtitleLang)` — TOPIK 한국어 학습 모드(시스템 프롬프트는 중국어로 작성, 대상 학습자는 중국인 학생).
- **모델·호출 규약**: 위 4종은 `google/gemini-2.5-flash` + `tools`(function name `song_analysis`) 로 강제 구조화 출력. 보조 함수(`translateTitleArtistToEnglish`, 한국어 단어 청크 추출)는 `gpt-4o-mini`, 지식형(`searchLyricsWithAI`, `extractCleanTitle`)은 `gpt-4o`.
- `callOpenAIWithRetry(apiKey, body, maxRetries = 3)`: `model` 을 `gpt-4o-mini` 로 고정하고 `max_tokens` 기본 16384. 429 이면 `insufficient_quota` 판별 후 즉시 throw(충전 안내) 또는 `min(1000*(n+1), 5000)` ms 대기 재시도. 401 은 "API Key 권한 부족(model.request)" 안내.
- `sanitizeTextForAPI()` 로 제어문자·BOM·zero-width 문자를 제거한 뒤에만 LLM 에 전송한다(400 방지).
- 한국어 단어장은 가사를 청크로 나눠 `CONCURRENCY = 3` 병렬로 `gpt-4o-mini` 호출 후 `word` 기준 dedupe.
- **저장 규약**: `songs` 는 `insertSongWithRecovery`(unique 충돌 시 `video_id` 로 재조회), `song_analyses` 는 `insertAnalysisWithRecovery`(`song_id` 로 재조회)로 멱등 처리. 분석 결과에서 `hsk_level = "HSK n"`, `tags = 상위 3개 "HSK n (개수)"`, `teaching_point ∈ {어휘, 문법, 문화, 발음}` 을 파생시켜 `songs` 에 반영한다.
- **supplement 분기**: 기존 `lyrics_with_pinyin` 중 `chinese/pinyin/korean` 이 모두 빈 줄만 재분석하여 병합 저장하고, `full_analysis.timed_entries` 는 유지한다. 교사가 가사를 새로 붙여 넣은 경우 `songs.lyrics_raw` · `lyrics_source(custom | custom_timed)` · `has_subtitles = true` 를 동기화하고, `full_analysis.manual_lyrics_source_raw` 와 `timed_source(manual_srt | manual_plain)` 를 기록한다.
- **제목 정규화**: 제목에 `_` 가 없을 때만 `extractCleanTitle` 를 호출해 중국어 곡은 `中文歌名_한국어제목`, 한국어 곡은 `한국어제목_中文歌名` 형식으로 갱신한다.
- `add_only = true` 이면 메타데이터만 채우고 분석 파이프라인을 건너뛴다.

**(C) `regenerate-patterns`**

- 입력 `{ song_id?, batch? }`. `batch` 가 참이면 조건에 맞는 전체 곡, 아니면 `song_id` 1곡.
- 대상 필터: `songs.language = 'chinese'` AND `lyrics_raw IS NOT NULL` AND 연결된 `song_analyses` 존재. 가사 길이 10자 미만은 `skipped`.
- `gpt-4o-mini`, `temperature 0.3`, `response_format: json_object`, 가사는 앞 2000자만 전송.
- 시스템 프롬프트 요구사항: 3–8개 문형 추출 / `pattern` 은 **한국어 문법 용어**(`把자문`, `是...的 강조구문`, `겸어문` 등, 중국어 용어 금지) / `explanation` 은 한국어 3–5문장(의미·한국어와의 대조·빈발 오류·사용 팁 포함) / `examples` 는 중국어 예문과 한국어 번역을 교대 배열.
- 결과를 `song_analyses.sentence_patterns` 에 UPDATE 하고 `{ processed, results[] }` 로 곡별 `ok | skipped | error` 를 반환한다.

**(D) `reanalyze-wordlist`**

- 입력 `{ song_id, lang: "ko" | "zh", level, lines_offset = 0, lines_limit? }`. 필수값 누락 시 400, 가사 없으면 404.
- 가사를 빈 줄 제거 후 `lines_offset ~ lines_offset+lines_limit` 범위로 슬라이스 → `CHUNK_SIZE = 10` 줄 단위 청크 → `CONCURRENCY = 3` 병렬 호출.
- `callAI()` 이중화: `OPENAI_API_KEY` 존재 시 `gpt-4o-mini`(temperature 0.3), 실패/부재 시 Lovable Gateway `google/gemini-2.5-flash`.
- 프롬프트: `lang = "ko"` → TOPIK `level` 급 이상 한국어 단어만, `word` = 한국어 단어 / `meaning_ko` = **중국어 번역** / `topik_level` 1–6 / `example_sentence` 는 해당 단어를 반드시 포함. `lang = "zh"` → HSK `level`–9급 중국어 단어, `word` = 简体中文 / `pinyin` / `meaning_ko` = 한국어 뜻 / `hsk_level` 1–9.
- 응답의 ```json 코드펜스를 제거한 뒤 파싱하고, `word` 기준으로 중복 제거한다.
- `lines_limit` 이 주어지면 `{ words, next_offset, has_more, total_lines }`(끊어읽기 모드), 없으면 레거시 호환을 위해 배열 그대로 반환하며 결과가 0개면 502.

**(E) `tag-song-culture`**

- 입력 `{ song_id, force? }`. 곡 없으면 404, 가사 확보 실패 시 422.
- 캐시 단축: `force` 가 아니고 `songs.theme` 이 있으며 `song_culture_tags` 행이 1개 이상이면 `{ ok: true, cached: true, theme }` 로 즉시 반환.
- 가사는 `songs.lyrics_raw` 우선, 없으면 최신 `song_analyses.lyrics_with_pinyin` 을 언어에 맞게 이어 붙여 사용하고 6000자로 자른다.
- `gpt-4o-mini` 1회 호출, `response_format: json_object`, `temperature 0.3`. 출력은 18개 필드(주제 1 + 문화 17) JSON 하나이며 코드블록·설명문 금지.
- 주제는 `사랑 / 이별 / 그리움 / 외로움 / 우정 / 가족애 / 희망 / 인생 / 자연 / 정체성 / 비판 / 기타` 12개 중 1개, 허용 목록 밖이면 `기타` 로 강제.
- 문화 17 카테고리(순서 고정): `명절과 절기 · 음식과 기물 · 복식과 건축 · 자연경관 · 동식물 · 지리명소 · 가정윤리 · 연애관 · 인간관계 · 사회규범 · 역사와 시대 · 민족과 종교 · 감정표현 · 심미적 이미지 · 가치관 · 인생철학 · 예술과 문학`. 해당 없음은 빈 문자열.
- 키워드 문자열은 `, ， 、 ; ；` 로 분리해 배열화한다.
- 저장: `songs.theme` UPDATE → `song_culture_tags` 를 `song_id` 로 DELETE → 17행 `upsert(onConflict: "song_id,category")`. 반환은 `{ ok, theme, tag_count }`(키워드가 비지 않은 카테고리 수).

**(F) `translate-lyrics-style`**

- 입력 `{ song_id, style, song_language, lyrics[] }`. `style ∈ {poetic, literal, casual}` 이 아니거나 `lyrics` 가 비면 400.
- 캐시 키는 `song_feature_cache.feature_type = "style_" + style`. 히트 시 `{ translations, cached: true }`.
- 방향 결정: `song_language !== "korean"` 이면 中文 → 한국어, 아니면 한국어 → 中文. 스타일 설명은 시적/직역/구어체 3종을 대상 언어로 서술한다.
- 입력 가사를 `0. …` 형태로 번호 매겨 전달하고, 응답도 같은 번호 형식만 허용(부가 설명 금지). `gpt-4o-mini`, `max_tokens: 2000`.
- `^(\d+)\.\s*(.+)` 정규식으로 파싱해 `Record<number, string>` 을 만들고 `song_feature_cache` 에 `upsert(onConflict: "song_id,feature_type")` 후 `{ translations, cached: false }` 반환.

**(G) `bulk-reanalyze-batch`** (관리자 전용 도구, `/songs` 의 BulkRepairPanel 에서 호출)

- `song_analyses` 를 500행 페이지 단위로 순회하여 `lyrics_with_pinyin` 안에 `chinese/pinyin/korean` 이 모두 빈 행이 하나라도 있는 곡을 결손 곡으로 수집한다.
- 수집한 id 를 100개씩 나눠 `songs` 메타데이터를 조회하고, `language` 필터와 `skipIds` 를 적용한 뒤 `limit` 만큼 처리 대상으로 반환한다.
- 각 곡은 `analyze-song` 을 `{ force: true, supplement: true, language, learn_language }` 로 재호출한다. 재시도 없이 1회(`MAX = 1`), 호출당 80초 `AbortController` 타임아웃을 걸어 엣지 150초 제한 안에서 종료시키고, 초과 곡은 이번 세션에서 건너뛴다.
- `youtube_url` 이 없는 곡은 `{ ok: false, error: "no youtube_url" }`.

### 2.3 강제 제약

- LLM 응답을 신뢰하지 말 것: 배열 길이·필수 키·등급 범위(HSK 1–9, TOPIK 1–6)를 코드에서 재검증하고, 부족분은 원본 값으로 채운다.
- 어떤 함수도 `SUPABASE_SERVICE_ROLE_KEY` · `OPENAI_API_KEY` · `LOVABLE_API_KEY` 를 응답 본문이나 로그에 출력하지 않는다.
- 로그는 `console.log/warn/error` 로 남기되 가사 원문 전체가 아니라 길이·상태코드·앞 200~500자만 남긴다.
- 설명 텍스트는 학습자 대상 언어를 지킨다: 중국어 학습 모드 = 한국어 설명, TOPIK 모드 = 중국어 설명.
- 문형의 `pattern` 필드에 중국어 문법 용어를 넣지 않는다(예: `+动词` 금지, `+동사` 사용).

---

## ③ Examples (예시)

### 3.1 확정 문자열·계약 표 (Design with Real Content)

| 위치 | 확정 값 |
| --- | --- |
| 분석 tool name | `song_analysis` |
| 주 분석 모델 | `google/gemini-2.5-flash` |
| 보조/재생성 모델 | `gpt-4o-mini` |
| 가사 검색·제목 정제 모델 | `gpt-4o` |
| 교육 포인트 enum | `어휘` · `문법` · `문화` · `발음` |
| 주제 enum(12) | `사랑 · 이별 · 그리움 · 외로움 · 우정 · 가족애 · 희망 · 인생 · 자연 · 정체성 · 비판 · 기타` |
| 문화 카테고리 수 | 17 (고정 순서) |
| 번역 스타일 캐시 키 | `style_poetic` · `style_literal` · `style_casual` |
| 단어 추출 청크 | 10줄 / 병렬 3 |
| 쿼터 소진 안내 | `OpenAI API 额度已用完，请在 OpenAI 控制台充值后重试。` |
| 키 권한 부족 안내 | `OpenAI API Key 权限不足，缺少 model.request 权限。请到 OpenAI 后台更新 API Key。` |
| 일시 장애 안내 | `Temporary backend connectivity issue while processing this song. Please retry.` |

### 3.2 파이프라인 트리 (Use Prompt Patterns for Layouts)

```text
analyze-song (POST)
├── 0. CORS / body parse / mode 판정 (youtube_url | song_id)
├── 1. 사용자 복원(getClaims) → owner_id 결정 (admin → NULL)
├── 2. 기존 곡 탐색 (본인 사본 → 공개/타인 사본 fork)
├── 3. 가사 확보
│   ├── custom_lyrics (SRT 파싱 가능)
│   ├── YouTube caption track
│   ├── LRCLIB (제목/가수 영문 정규화 → best match)
│   └── gpt-4o 가사 검색
├── 4. 분석 분기
│   ├── timed blocks  → analyzeTimedBlocksWithAI (N-in = N-out 강제)
│   ├── zh 곡 · zh학습 → analyzeWithAI
│   ├── ko 곡 · zh학습 → analyzeKoreanSongWithAI (+ 청크 단어 추출)
│   └── TOPIK 모드     → analyzeForKoreanLearning
├── 5. 저장 (songs / song_analyses, insert-with-recovery)
├── 6. 파생 필드 (hsk_level · tags(top3) · teaching_point · title 정규화)
└── 7. 응답 { song, analysis }

후속 독립 함수 (실패해도 4~6단계를 막지 않음)
├── regenerate-patterns    → song_analyses.sentence_patterns
├── reanalyze-wordlist     → 클라이언트가 받은 뒤 저장(끊어읽기)
├── tag-song-culture       → songs.theme + song_culture_tags(17)
└── translate-lyrics-style → song_feature_cache.style_*
```

---

## ④ Context (배경)

### 4.1 프로젝트 맥락

「멜로디 클래스」의 노래 아카이브는 B1 이 확보한 텍스트를 B2 가 학습 자산으로 바꾼다. 여기서 만들어진 `lyrics_with_pinyin` · `word_list` · `sentence_patterns` 는 각각 P3k(가사 탭) · P3l(단어 탭) · P3m(문형 탭)의 유일한 데이터 원천이며, `song_culture_tags` 와 `songs.theme` 은 P3n/P3o 탐구 모듈과 P3g 필터의 기준이 된다. 교사·학생 모두 동일한 함수 집합을 호출하며 역할 분기는 없다(권한 차이는 `owner_id` 기반 RLS 로만 발생한다).

### 4.2 Lovable Cloud 후경

- `analyze-song` · `regenerate-patterns` · `reanalyze-wordlist` · `translate-lyrics-style` · `bulk-reanalyze-batch` 는 `config.toml` 에서 `verify_jwt = false` 로 두어 공개 곡 조회/공유 페이지에서도 호출 가능하게 하되, 사용자 식별은 Authorization 헤더가 있을 때만 수행한다. `tag-song-culture` 는 `config.toml` 항목을 두지 않아 기본값 `verify_jwt = true` 로 동작한다.
- 프런트엔드는 이 함수들을 `fetchWithRetry` + `AbortController` 로 호출하고, 진행 상태를 현재 브라우저 세션 범위에서만 추적한다(백그라운드 분석 UX).
- 4-상태 렌더링: 로딩(펄스 애니메이션) · 빈 상태(가사 붙여넣기 유도) · 에러(429/402 는 한국어 크레딧 안내) · 성공.

### 4.3 데이터 계약

```sql
-- 곡 본체 (B1 에서 생성, B2 가 갱신)
-- 갱신 컬럼: lyrics_raw, lyrics_source, has_subtitles, hsk_level, tags, teaching_point, theme, title

CREATE TABLE public.song_analyses (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  song_id uuid NOT NULL REFERENCES public.songs(id) ON DELETE CASCADE,
  word_list jsonb NOT NULL DEFAULT '[]'::jsonb,
  sentence_patterns jsonb NOT NULL DEFAULT '[]'::jsonb,
  lyrics_with_pinyin jsonb NOT NULL DEFAULT '[]'::jsonb,
  full_analysis jsonb NOT NULL DEFAULT '{}'::jsonb,
  created_at timestamptz NOT NULL DEFAULT now()
);
GRANT SELECT ON public.song_analyses TO anon;
GRANT SELECT, INSERT, UPDATE, DELETE ON public.song_analyses TO authenticated;
GRANT ALL ON public.song_analyses TO service_role;
ALTER TABLE public.song_analyses ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Analyses follow song visibility"
  ON public.song_analyses FOR SELECT
  USING (EXISTS (
    SELECT 1 FROM public.songs s
    WHERE s.id = song_analyses.song_id
      AND s.deleted_at IS NULL
      AND (s.owner_id IS NULL OR s.owner_id = auth.uid())
  ));
CREATE POLICY "Owners manage their analyses"
  ON public.song_analyses FOR ALL TO authenticated
  USING (EXISTS (SELECT 1 FROM public.songs s WHERE s.id = song_analyses.song_id AND s.owner_id = auth.uid()))
  WITH CHECK (EXISTS (SELECT 1 FROM public.songs s WHERE s.id = song_analyses.song_id AND s.owner_id = auth.uid()));

CREATE TABLE public.song_culture_tags (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  song_id uuid NOT NULL REFERENCES public.songs(id) ON DELETE CASCADE,
  category text NOT NULL,
  keywords text[] NOT NULL DEFAULT '{}',
  created_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (song_id, category)
);
GRANT SELECT ON public.song_culture_tags TO anon;
GRANT SELECT ON public.song_culture_tags TO authenticated;
GRANT ALL ON public.song_culture_tags TO service_role;
ALTER TABLE public.song_culture_tags ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Culture tags are readable with the song"
  ON public.song_culture_tags FOR SELECT
  USING (EXISTS (
    SELECT 1 FROM public.songs s
    WHERE s.id = song_culture_tags.song_id
      AND s.deleted_at IS NULL
      AND (s.owner_id IS NULL OR s.owner_id = auth.uid())
  ));

-- song_feature_cache 는 B1 §4.3 에 정의된 스키마를 그대로 사용한다
-- (UNIQUE (song_id, feature_type), feature_type 에 style_poetic/style_literal/style_casual 추가)
```

필수 시크릿: `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_ANON_KEY`, `OPENAI_API_KEY`, `LOVABLE_API_KEY`, `YOUTUBE_API_KEY`.

---

## ⑤ Acceptance & Output (검증·산출)

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6 — 정량 임계값)

1. 타임스탬프 블록 N개 입력 시 `lyrics_with_pinyin.length === N` 일치율 = 100% (샘플 20곡 검증).
2. `word_list` 는 일반 길이(30줄 이상) 곡에서 항목 수 ≥ 15, 중복 `word` 0건.
3. `sentence_patterns` 항목 수 3–8, 각 항목의 `examples.length % 2 === 0`, `pattern` 필드에 CJK 문법 용어(`动词`·`句`) 매칭 0건.
4. `tag-song-culture` 응답의 `song_culture_tags` 행 수 = 정확히 17, `theme` 이 허용 12개 밖일 확률 0%.
5. `translate-lyrics-style` 2회 연속 호출 시 두 번째 응답 `cached === true`, 응답 시간 p95 ≤ 300 ms.
6. `reanalyze-wordlist` 를 `lines_limit = 20` 으로 연속 호출할 때 `next_offset` 이 단조 증가하고 마지막 응답의 `has_more === false`.
7. OpenAI 429(`insufficient_quota`) 발생 시 재시도 횟수 0회, 사용자 노출 메시지에 "充值" 문자열 포함.
8. `bulk-reanalyze-batch` 단일 요청 총 실행시간 ≤ 150 s, 곡당 호출 타임아웃 = 80 s.
9. 6개 함수 모두 `OPTIONS` 요청에 200 + CORS 헤더 반환, 미인증 호출로 500 발생 0건.
10. 소스 전체에서 `grep -n "SERVICE_ROLE_KEY" | grep -i "console\."` 결과 0건.

### 5.2 Output Format

다음 순서로 파일 전문만 출력한다. 설명·사과·주석 서문·마크다운 헤더 금지.

1. `supabase/functions/analyze-song/index.ts`
2. `supabase/functions/regenerate-patterns/index.ts`
3. `supabase/functions/reanalyze-wordlist/index.ts`
4. `supabase/functions/tag-song-culture/index.ts`
5. `supabase/functions/translate-lyrics-style/index.ts`
6. `supabase/functions/bulk-reanalyze-batch/index.ts`
7. `supabase/config.toml` (해당 함수 블록만)
8. 마이그레이션 SQL (`song_analyses`, `song_culture_tags`)
