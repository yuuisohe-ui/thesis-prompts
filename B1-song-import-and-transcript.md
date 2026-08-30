# B1 · 노래 임포트 · 자막 취득 백엔드 재현 프롬프트

> **본 프롬프트는 B 시리즈(B1–B4)의 1/4 — 백엔드(Supabase Edge Functions) 재현 자료.**
> **적용 대상**: `supabase/functions/youtube-search`, `supabase/functions/fetch-transcript`, `supabase/functions/get-youtube-transcript`, `supabase/functions/normalize-lyrics`, `supabase/functions/cover-image-cache`
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**
> **범위 경계**: 가사 *분석*(`analyze-song` 이후)은 B2, Suno 생성은 B3, 영상 합성 콜백은 B4 에서 다룬다. 본 문서는 "곡을 찾아 들여오고, 텍스트(자막/가사)를 확보해 정규화하는" 단계까지만 책임진다.

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* https://platform.openai.com/docs/guides/prompt-engineering (Retrieved July 12, 2026)
2. **Lovable. (n.d.).** *Prompting best practices.* https://docs.lovable.dev/prompting/prompting-one (Retrieved July 12, 2026)
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.

---

## ① Identity (신원)

당신은 **Deno 런타임 기반 Supabase Edge Functions 를 설계하는 시니어 백엔드 엔지니어**입니다. 담당 도메인은 "외부 API 프록시 + 다단계 폴백(fallback) 파이프라인 + LLM 후처리"이며, 다음 세 가지를 항상 최우선으로 지킵니다.

1. **비밀키는 절대 클라이언트에 노출하지 않는다.** `YOUTUBE_API_KEY` · `SUPADATA_API_KEY` · `YT_DLP_SERVICE_KEY` · `OPENAI_API_KEY` · `SUPABASE_SERVICE_ROLE_KEY` 는 오직 `Deno.env.get()` 으로만 읽는다.
2. **외부 서비스는 반드시 실패한다고 가정한다.** 모든 외부 호출은 타임아웃·상태코드 검사·다음 단계 폴백을 갖는다. 어떤 경우에도 함수가 unhandled exception 으로 죽지 않는다.
3. **부분 성공을 허용한다.** 자막이 없어도 `has_subtitles: false` 로 200 을 반환하고, 교사가 가사를 직접 붙여 넣는 경로로 이어지게 한다.

기술 스택: Deno + `Deno.serve` (일부 레거시는 `std@0.168.0/http/server.ts` 의 `serve`), `@supabase/supabase-js@2` (esm.sh), OpenAI Chat Completions API, YouTube Data API v3, Supadata API, 자체 호스팅 yt-dlp 서비스.

---

## ② Instructions (지시)

### 2.1 산출물 (Prompt by Component, Not Page)

파일 단위로 정확히 5개의 Edge Function 을 생성한다.

```text
supabase/
├── config.toml                                  # verify_jwt 설정 추가
└── functions/
    ├── youtube-search/index.ts                  # ①곡 탐색 프록시
    ├── fetch-transcript/index.ts                # ②자막 1차 소스(Supadata)
    ├── get-youtube-transcript/index.ts          # ③자막 3단 폴백 체인
    ├── normalize-lyrics/index.ts                # ④가사 문장 정규화(LLM)
    └── cover-image-cache/index.ts               # ⑤커버 이미지 영구화(서명 URL)
```

### 2.2 공통 규약 (Speak Atomic)

**(a) CORS 헤더 — 모든 함수 상단에 동일 상수로 선언한다.**

```ts
const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers":
    "authorization, x-client-info, apikey, content-type, x-supabase-client-platform, x-supabase-client-platform-version, x-supabase-client-runtime, x-supabase-client-runtime-version",
};
```

첫 줄에서 `if (req.method === "OPTIONS") return new Response(null, { headers: corsHeaders });` 로 preflight 를 처리한다.

**(b) `verify_jwt` 정책** — `supabase/config.toml` 에 다음을 추가한다. `verify_jwt = false` 인 함수는 **함수 본문에서 직접 Bearer 토큰을 검증**한다(익명 공유 페이지에서도 호출되어야 하므로 게이트웨이 레벨 검증은 끄고, 쿼터 소모형 함수만 코드 레벨에서 막는다).

```toml
[functions.fetch-transcript]
verify_jwt = false

[functions.get-youtube-transcript]
verify_jwt = false

[functions.youtube-search]
verify_jwt = false

[functions.normalize-lyrics]
verify_jwt = false
```

**(c) 쿼터 보호 인증 패턴** — `youtube-search` 와 `fetch-transcript` 는 유료 외부 API 를 소모하므로 아래 블록을 본문 최상단에 둔다.

```ts
const authHeader = req.headers.get("Authorization");
if (!authHeader?.startsWith("Bearer ")) {
  return new Response(JSON.stringify({ error: "Unauthorized" }), {
    status: 401, headers: { ...corsHeaders, "Content-Type": "application/json" },
  });
}
const sb = createClient(Deno.env.get("SUPABASE_URL")!, Deno.env.get("SUPABASE_ANON_KEY")!);
const { data: claims, error: claimsErr } = await sb.auth.getClaims(authHeader.replace("Bearer ", ""));
if (claimsErr || !claims?.claims?.sub) {
  return new Response(JSON.stringify({ error: "Unauthorized" }), {
    status: 401, headers: { ...corsHeaders, "Content-Type": "application/json" },
  });
}
```

**(d) 오류 코드 매핑** — LLM 호출 실패 시 `429`(레이트 리밋)과 `402`(크레딧 소진)는 **그대로 통과시키고**, 그 외 모든 실패는 `500` 으로 정규화한다.

```ts
status: r.status === 429 || r.status === 402 ? r.status : 500
```

**(e) 로그 규약** — 모든 `console.log` 는 `[함수명]` 접두사로 시작하고, 외부 응답 본문은 `.slice(0, 200)` 으로 절단해 기록한다.

---

### 2.3 함수 ① `youtube-search` — 곡 탐색 프록시

- 입력: `{ query: string, maxResults?: number = 15, relevanceLanguage?: string }`. `query` 없으면 throw.
- `maxResults` 는 `Math.min(Number(maxResults) || 15, 25)` 로 상한 25 를 강제한다.
- **2단계 호출**:
  1. `GET https://www.googleapis.com/youtube/v3/search?part=snippet&type=video&q=…` → `videoId[]` 추출. 결과 0건이면 `{ results: [] }` 로 즉시 200 반환.
  2. `GET .../videos?part=snippet,contentDetails&id=<ids>` → `contentDetails.caption` (`"true"`/`"false"`)을 **추가 쿼터 없이** 획득.
- ISO-8601 duration(`PT#H#M#S`)을 정규식 `/PT(?:(\d+)H)?(?:(\d+)M)?(?:(\d+)S)?/` 으로 초 단위 변환 후 `m:ss` 로 포맷.
- 자막이 있는 항목만 대상으로 `GET .../captions?part=snippet&videoId=…` 를 `Promise.all` 병렬 호출해 `captionLanguages: string[]`(중복 제거)을 채운다. 실패는 무시(빈 배열 유지).
- 출력 항목 스키마: `{ videoId, title, channelTitle, thumbnail, publishedAt, durationSeconds, duration, caption, captionLanguages }`.
- **프런트 이중 언어 계약**: 클라이언트(`YouTubeSearchDialog`)는 이 함수를 `query` 와 `` `${query} 가사` `` 두 벌로 `Promise.all` 동시 호출해 중·한 결과를 병합한다. 따라서 본 함수는 언어 분기를 내장하지 않고 **순수 프록시**로 유지한다.

### 2.4 함수 ② `fetch-transcript` — 자막 1차 소스(Supadata)

- 입력: `{ videoId: string, language?: "korean" | "chinese" }`. `videoId` 없으면 400.
- 언어 우선순위 배열(원문 그대로):
  - `korean` → `["ko", "en"]`
  - 그 외 → `["zh", "zh-Hans", "zh-CN", "zh-Hant", "zh-TW", "en"]`
- 각 언어에 대해 `GET https://api.supadata.ai/v1/transcript?url=<youtubeUrl>&lang=<lang>` 를 순차 시도(`x-api-key` 헤더). 응답 `content` 가 `offset` 필드를 가진 객체 배열이면 성공으로 판정하고 즉시 break.
- 단위 변환: `start = offset / 1000`, `dur = duration / 1000` (ms → s).
- 우선순위 전부 실패 시 `lang` 파라미터 없이 **auto 1회 재시도**.
- 출력: `{ entries: {text,start,dur}[], lang: string | null }`. 자막이 없어도 200 + 빈 배열.
- `SUPADATA_API_KEY` 미설정 시 500 + `"SUPADATA_API_KEY not configured"`.

### 2.5 함수 ③ `get-youtube-transcript` — 3단 폴백 체인 (핵심)

- 입력: `{ url: string, learn_language?: "korean" | "chinese", check_only?: boolean, languages?: string[] }`.
- `extractVideoId()` 는 11자 raw ID, `?v=`, `youtu.be/`, `/embed/`, `/shorts/` 5가지 형태를 모두 파싱한다. 실패 시 400.
- 언어 우선순위 상수:
  ```ts
  const SUBTITLE_LANG_PRIORITY_CHINESE = ["zh","zh-Hans","zh-CN","zh-Hant","zh-TW","ko","en"];
  const SUBTITLE_LANG_PRIORITY_KOREAN  = ["ko","en","zh","zh-Hans","zh-CN"];
  ```
  `body.languages` 가 오면 그것을 최우선 사용한다.
- **Strategy 1 — 자체 yt-dlp 서비스**: `POST ${YT_DLP_SERVICE_URL}/transcript` (`Authorization: Bearer ${YT_DLP_SERVICE_KEY}`), body `{ url, languages }`. **`AbortController` + 30 초 타임아웃 필수.** 성공 시 `source` 를 `auto_caption|automatic_captions → "yt_dlp_auto"`, 그 외 `→ "yt_dlp_manual"` 로 정규화.
- **Strategy 2 — watch 페이지 스크레이핑**: `https://www.youtube.com/watch?v=<id>&hl=en` 을 데스크톱 UA 로 받아 `/"captionTracks"\s*:\s*(\[.*?\])/s` 정규식으로 추출(`\u0026`, `\"` 언이스케이프 후 `JSON.parse`). 각 트랙은 `baseUrl` 에 `fmt=srv3` 를 붙여 받고 `<text start dur>` 정규식으로 파싱, HTML 엔티티 5종(`&amp; &lt; &gt; &#39; &quot;`)을 복원한다. 언어 우선순위별로 **manual(`kind !== "asr"`) → asr** 순으로 탐색하고, 그래도 없으면 전체 트랙을 순회한다. `source` 는 `"youtube_caption_manual"` / `"youtube_caption_asr"`.
- **Strategy 3 — Supadata**: 우선순위 배열 + 빈 문자열(auto)을 순회. `source = "supadata"`.
- **`check_only: true`** 이면 존재 여부만 알리고 `entries: []`, `plain_text: ""` 로 응답(대역폭 절약, 목록 화면 배지용).
- 모두 실패해도 **200** + `{ ok: true, has_subtitles: false, source: "none", language: null, entries: [], plain_text: "" }`.
- 최종 예외만 500 + `{ ok: false, error }`.

### 2.6 함수 ④ `normalize-lyrics` — 구어체 가사의 문장 정규화 (LLM)

- 입력: `{ song_id: uuid, lyrics: {chinese,korean,pinyin}[], language?: "zh" | "ko" = "zh" }`. 누락 시 400 (`"song_id and lyrics[] required"`).
- **캐시 우선**: `song_feature_cache` 에서 `(song_id, feature_type = "normalized_lyrics")` 를 `maybeSingle()` 로 조회해 있으면 즉시 반환. 캐시 접근은 `SUPABASE_SERVICE_ROLE_KEY` 클라이언트로 수행한다.
- 입력 가사는 **최대 50 행(`slice(0, 50)`)** 으로 절단하고 `idx` 를 부여한다.
- **LLM 호출 사양** (원문 그대로 재현할 것):
  - 엔드포인트 `https://api.openai.com/v1/chat/completions`
  - `model: "gpt-4o-mini"`, `temperature: 0.3`, `response_format: { type: "json_object" }`
  - system: `"You are a Chinese/Korean linguist. Always return valid JSON only, no prose."`
  - user 프롬프트 원문:
    ```text
    다음은 노래 가사입니다. 각 줄의 {중국어|한국어} 표현을 구어체/축약형에서 문법적으로 완성된 자연스러운 문장으로 변환하세요. 다른 언어의 번역과 병음도 변환된 문장에 맞게 조정하세요.

    원본 가사:
    [0] zh: … | ko: … | py: …
    [1] zh: … | ko: … | py: …

    JSON 형식으로 반환하세요:
    {
      "lines": [
        { "idx": 0, "chinese": "...", "korean": "...", "pinyin": "..." },
        ...
      ]
    }
    ```
    (`{중국어|한국어}` 는 `language === "zh"` 여부로 치환한다.)
- **머지 규칙**: 응답 `lines` 를 `idx` 로 매칭하되, 누락·빈 값이면 **원본 값을 유지**한다(`upd.chinese || orig.chinese`). JSON 파싱 실패 시 `{ lines: [] }` 로 처리 → 결과적으로 원본이 그대로 반환된다(무손실 폴백).
- 성공 결과 `{ lines }` 를 `song_feature_cache` 에 insert 한 뒤 반환한다.

### 2.7 함수 ⑤ `cover-image-cache` — 외부 이미지의 영구화

> 본 함수는 곡 자체가 아니라 **`lesson_plans` · `courses` 의 커버 이미지**를 대상으로 하지만, Pixabay 원본 URL 이 만료되는 문제를 해결하는 "임포트 계열" 유틸이므로 B1 에 포함한다.

- 상수: `BUCKET = "cover-images"`(비공개 버킷), `SIGNED_TTL = 60*60*24*365*10`(10년), `ALLOWED_TABLES = ["lesson_plans","courses"]`.
- 클라이언트는 `SUPABASE_SERVICE_ROLE_KEY` + `{ auth: { persistSession: false, autoRefreshToken: false } }`.
- `cacheOne(table, id, sourceUrl)` 4단계:
  1. 원본 다운로드 — **Pixabay CDN 은 실제 브라우저 UA 를 요구**하므로 `User-Agent`(Chrome 124) · `Accept: image/avif,image/webp,...` · `Referer: https://pixabay.com/` 를 반드시 붙인다.
  2. `content-type` 이 `image/` 로 시작하지 않으면 중단. 확장자는 `png|webp|gif` 를 판별하고 기본은 `jpg`.
  3. `storage.from(BUCKET).upload(`${table}/${id}.${ext}`, buf, { contentType, upsert: true })`.
  4. `createSignedUrl(path, SIGNED_TTL)` → 얻은 URL 을 소스 테이블의 `cover_image_url` 에 기록.
- `mode: "bulk"` 이면 `ALLOWED_TABLES` 전체를 순회하며 `pixabay.com` 을 가리키는 행을 마이그레이션하고, 원본이 죽었으면 `PIXABAY_API_KEY` 로 키워드 재검색(`image_type=photo`, `safesearch=true`, `orientation=horizontal`, `per_page=10`)해 대체 이미지를 확보한다. 집계는 `{ migrated, failed, refreshed }`.
- 어떤 단계가 실패해도 예외를 던지지 않고 `null` 반환 + `console.error` 로 기록한다(일괄 처리 중단 방지).

### 2.8 강제 제약

- `SUPABASE_SERVICE_ROLE_KEY` 는 **`normalize-lyrics`(캐시 쓰기) · `cover-image-cache`(스토리지 쓰기)** 두 함수에서만 사용한다. 나머지 함수는 `SUPABASE_ANON_KEY` 로 토큰 검증만 수행한다.
- 어떤 함수도 사용자 입력 문자열을 SQL 로 연결하지 않는다(모두 PostgREST 빌더 사용).
- 자막 취득 함수는 **절대 4xx/5xx 로 "자막 없음"을 표현하지 않는다.** "없음"은 200 + 플래그로만 표현한다.
- 외부 호출에는 예외 없이 타임아웃 또는 상태코드 가드를 둔다.

---

## ③ Examples (예시)

### 3.1 요청/응답 계약 표

| 함수 | 요청 예시 | 성공 응답 예시 |
| --- | --- | --- |
| `youtube-search` | `{ "query": "周杰伦 稻香", "maxResults": 15 }` | `{ "results": [{ "videoId":"…","title":"稻香","channelTitle":"…","durationSeconds":223,"duration":"3:43","caption":true,"captionLanguages":["zh","en"] }] }` |
| `fetch-transcript` | `{ "videoId": "…", "language": "chinese" }` | `{ "entries": [{ "text":"对这个世界如果你有太多的抱怨","start":18.4,"dur":3.2 }], "lang": "zh" }` |
| `get-youtube-transcript` | `{ "url": "https://youtu.be/…", "learn_language": "korean" }` | `{ "ok":true, "has_subtitles":true, "source":"yt_dlp_manual", "language":"ko", "entries":[…], "plain_text":"…" }` |
| `get-youtube-transcript` (없음) | `{ "url":"…", "check_only":true }` | `{ "ok":true, "has_subtitles":false, "source":"none", "language":null, "entries":[], "plain_text":"" }` |
| `normalize-lyrics` | `{ "song_id":"uuid", "language":"zh", "lyrics":[{ "chinese":"别问 我 是谁","korean":"묻지 마","pinyin":"bié wèn wǒ shì shéi" }] }` | `{ "lines":[{ "chinese":"내가 누구인지 묻지 마세요 → 你别问我是谁","korean":"제가 누구인지 묻지 마세요","pinyin":"nǐ bié wèn wǒ shì shéi" }] }` |
| `cover-image-cache` | `{ "table":"courses", "id":"uuid", "sourceUrl":"https://pixabay.com/…" }` | `{ "url":"https://…/cover-images/courses/uuid.jpg?token=…" }` |

### 3.2 자막 폴백 결정 트리

```text
url ──► extractVideoId ──(실패)──► 400 Invalid YouTube URL
             │
             ▼
   ① yt-dlp 서비스 (30s timeout, Bearer)
        │ ok && has_subtitles ──► 200  source = yt_dlp_manual | yt_dlp_auto
        │ 실패 / 자막없음
        ▼
   ② watch 페이지 captionTracks 스크레이핑
        │ manual(kind≠asr) → asr → 전체 트랙 순회
        │ 성공 ──► 200  source = youtube_caption_manual | youtube_caption_asr
        │ 실패
        ▼
   ③ Supadata (lang 우선순위 → auto)
        │ 성공 ──► 200  source = supadata
        │ 실패
        ▼
   200  has_subtitles = false, source = "none"
             │
             ▼
   (프런트) 교사가 가사를 직접 붙여넣는 4차 폴백
```

### 3.3 환경 변수 매트릭스

| 변수 | 사용 함수 | 없을 때 동작 |
| --- | --- | --- |
| `YOUTUBE_API_KEY` | `youtube-search` | 500 `"YOUTUBE_API_KEY is not configured"` |
| `SUPADATA_API_KEY` | `fetch-transcript`, `get-youtube-transcript`(③) | ②는 500, ③은 해당 단계 skip |
| `YT_DLP_SERVICE_URL` / `YT_DLP_SERVICE_KEY` | `get-youtube-transcript`(①) | ① 단계 skip → ②로 진행 |
| `OPENAI_API_KEY` | `normalize-lyrics` | 500 `"OPENAI_API_KEY missing"` |
| `PIXABAY_API_KEY` | `cover-image-cache`(bulk) | 재검색 없이 마이그레이션만 수행 |
| `SUPABASE_URL` / `SUPABASE_ANON_KEY` | 인증 검증 전 함수 | 401 |
| `SUPABASE_SERVICE_ROLE_KEY` | `normalize-lyrics`, `cover-image-cache` | 캐시/스토리지 기능 비활성(전자는 캐시 없이 동작) |

---

## ④ Context (배경)

### 4.1 프로젝트 맥락

「멜로디 클래스」는 한국 대학의 중국어(및 한국어) 수업을 위한 노래 기반 교수·학습 플랫폼이다. B1 이 담당하는 임포트 단계는 전체 파이프라인의 **입구**이며, 여기서 확보한 `entries[]`(타임스탬프가 붙은 가사)가 이후 모든 기능 — 가사 하이라이트 동기화, 단어장, 문형, 받아쓰기, 발음 평가 — 의 원재료가 된다. 따라서 이 단계의 성공률이 플랫폼 전체의 사용 가능성을 결정한다.

### 4.2 Lovable Cloud 후경

- 클라이언트는 언제나 `supabase.functions.invoke(name, { body })` 로만 호출한다. 외부 API 키는 프런트 번들에 절대 포함되지 않는다.
- 익명 공유 페이지(`/shared/:token`, `/embed/:id`)에서도 자막 조회가 필요하므로 `verify_jwt = false` 를 쓰되, 쿼터 소모형 함수는 §2.2(c) 로 코드 레벨에서 로그인 사용자만 통과시킨다.
- 프런트는 4-상태(로딩 / 빈 / 오류 / 성공)를 모두 렌더한다. 특히 `has_subtitles: false` 는 **오류가 아니라 "빈 상태"** 로 처리하고 "가사 직접 입력" CTA 를 노출한다.

### 4.3 데이터 계약

본 시리즈가 접촉하는 테이블은 `songs`(임포트 결과 기록)와 `song_feature_cache`(정규화 캐시)이다.

```sql
-- 1) 곡 원장
CREATE TABLE public.songs (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id      uuid,                       -- NULL = 공용, 그 외 = 개인 소유
  video_id      text,
  youtube_url   text,
  title         text,
  title_bilingual text,                     -- 'OriginalTitle_TranslatedTitle'
  artist        text,
  language      text,                       -- 'chinese' | 'korean'
  hsk_level     text,
  theme         text,
  year          int,
  tags          text[],
  lyrics_raw    text,
  lyrics_source text,                       -- yt_dlp_manual | youtube_caption_asr | supadata | manual
  has_subtitles boolean DEFAULT false,
  share_token   uuid DEFAULT gen_random_uuid(),
  deleted_at    timestamptz,
  created_at    timestamptz NOT NULL DEFAULT now()
);
GRANT SELECT, INSERT, UPDATE, DELETE ON public.songs TO authenticated;
GRANT SELECT ON public.songs TO anon;       -- 공유/임베드 페이지용
GRANT ALL ON public.songs TO service_role;
ALTER TABLE public.songs ENABLE ROW LEVEL SECURITY;
CREATE POLICY "owned_select" ON public.songs FOR SELECT USING (
  ((deleted_at IS NULL) AND (owner_id IS NULL OR owner_id = auth.uid()))
  OR ((deleted_at IS NOT NULL) AND (owner_id = auth.uid() OR (owner_id IS NULL AND public.is_admin())))
);
CREATE POLICY "owned_insert" ON public.songs FOR INSERT TO authenticated
  WITH CHECK (public.is_admin() OR owner_id = auth.uid());
CREATE POLICY "owned_update" ON public.songs FOR UPDATE TO authenticated
  USING (owner_id = auth.uid() OR public.is_admin())
  WITH CHECK (owner_id = auth.uid() OR public.is_admin());
CREATE POLICY "owned_delete" ON public.songs FOR DELETE TO authenticated
  USING (owner_id = auth.uid() OR public.is_admin());

-- 2) 파생 콘텐츠 캐시(정규화 가사 포함)
CREATE TABLE public.song_feature_cache (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  song_id      uuid NOT NULL REFERENCES public.songs(id) ON DELETE CASCADE,
  feature_type text NOT NULL,               -- 'normalized_lyrics' 등
  content      jsonb NOT NULL,
  owner_id     uuid,
  created_at   timestamptz NOT NULL DEFAULT now()
);
GRANT SELECT, INSERT, UPDATE, DELETE ON public.song_feature_cache TO authenticated;
GRANT SELECT ON public.song_feature_cache TO anon;
GRANT ALL ON public.song_feature_cache TO service_role;
ALTER TABLE public.song_feature_cache ENABLE ROW LEVEL SECURITY;
CREATE POLICY "owned_select" ON public.song_feature_cache FOR SELECT
  USING (owner_id IS NULL OR owner_id = auth.uid());
```

### 4.4 권한 설계 원칙 — **역할 평등 + 소유권 격리**

「노래 아카이브」에 대한 접근 제어는 **교사/학생 역할과 무관**하다. 본 재현물에서도 반드시 다음을 지킨다.

1. **역할 평등(Role parity)**: 교사·학생이 사용할 수 있는 기능 집합은 완전히 동일하다. `has_role()` / `is_admin()` 를 곡 관련 RLS 나 함수 분기 조건으로 쓰지 않는다(`is_admin()` 은 운영자 예외 처리에만 등장).
2. **소유권 격리(Ownership isolation)**: 가시성과 수정 권한은 오직 `owner_id` 로 결정한다. `owner_id IS NULL` 은 공용, `owner_id = auth.uid()` 는 개인 소유이며 **개인 곡은 서로 보이지 않는다.**
3. **수정 시 분기(fork-on-edit)**: 비관리자가 공용 곡을 수정하려 하면 `fork_public_item('songs', id)` 로 개인 사본을 만든 뒤 사본을 수정한다. 원본 공용 곡은 변경되지 않으며, 사본의 변경 내용은 **본인에게만** 보인다.

---

## ⑤ Acceptance & Output (검증·산출)

### 5.1 Acceptance Criteria (정량)

| # | 항목 | 임계값 |
| --- | --- | --- |
| 1 | 자막 보유 영상 100건 표본에 대한 `get-youtube-transcript` 취득 성공률 | ≥ 95% |
| 2 | `get-youtube-transcript` 가 4xx/5xx 를 반환하는 경우 | 잘못된 URL(400)과 unhandled(500) **2가지뿐** — "자막 없음"으로 인한 4xx/5xx 0건 |
| 3 | Strategy 1(yt-dlp) 타임아웃 | 정확히 30,000 ms, `AbortController` 사용 — `grep -c "AbortController" get-youtube-transcript/index.ts` ≥ 1 |
| 4 | `youtube-search` p95 응답 | ≤ 2,000 ms (maxResults = 15 기준) |
| 5 | `youtube-search` 의 `maxResults` 상한 | 입력값과 무관하게 ≤ 25 |
| 6 | `fetch-transcript` · `youtube-search` 무인증 호출 | 100% `401` |
| 7 | `normalize-lyrics` 캐시 적중 시 OpenAI 호출 횟수 | 0회 (동일 `song_id` 2회 호출 시 두 번째는 외부 호출 없음) |
| 8 | `normalize-lyrics` 처리 행 수 | ≤ 50 행 (초과분은 절단) |
| 9 | LLM JSON 파싱 실패 시 손실 | 0 — 원본 `lyrics[]` 와 출력 행 수·순서 동일 |
| 10 | 프런트 번들 내 외부 API 키 노출 | `grep -rE "YOUTUBE_API_KEY|SUPADATA_API_KEY|OPENAI_API_KEY|SERVICE_ROLE" src/` 결과 **0건** |
| 11 | CORS preflight | 5개 함수 모두 `OPTIONS` 에 200/204 + `Access-Control-Allow-Origin: *` |
| 12 | `cover-image-cache` 서명 URL 유효기간 | 315,360,000 초(10년) |
| 13 | 곡 관련 RLS 정책·Edge Function 내 역할 분기 | `has_role(` 사용 0건 (§4.4 역할 평등) |

### 5.2 Output Format

LLM(또는 Lovable)은 아래 순서대로 파일만 출력한다. 설명·사과·주석·마크다운 헤더 등 부수 텍스트를 출력하지 않는다.

1. `supabase/functions/youtube-search/index.ts`
2. `supabase/functions/fetch-transcript/index.ts`
3. `supabase/functions/get-youtube-transcript/index.ts`
4. `supabase/functions/normalize-lyrics/index.ts`
5. `supabase/functions/cover-image-cache/index.ts`
6. `supabase/config.toml` (해당 `[functions.*] verify_jwt = false` 블록만 추가)

---

## 부록 · B 시리즈 전체 구성

| 문서 | 범위 | 주요 함수 |
| --- | --- | --- |
| **B1**(본 문서) | 곡 임포트 · 자막 취득 · 가사 정규화 | `youtube-search` · `fetch-transcript` · `get-youtube-transcript` · `normalize-lyrics` · `cover-image-cache` |
| **B2** | 가사 분석 · 부분 재생성 · 심화 카드 | `analyze-song` · `reanalyze-wordlist` · `regenerate-patterns` · `bulk-reanalyze-batch` · `translate-lyrics-style` · `tag-song-culture` · `generate-artist-*` · `generate-cultural-background` · `analyze-rhetoric` · `generate-emotion-analysis` · `generate-song-feature` · `generate-related-words` · `generate-vocab-examples` · `generate-example-sentence` |
| **B3** | Suno 원곡 생성 전 과정 | `sg-generate-lyrics` · `sg-suno-lyrics` · `sg-suno-generate` · `sg-suno-poll` · `sg-repair-aligned` · `sg-finalize-song` |
| **B4** | 배경 영상 합성 · 비동기 콜백 · 계정 정리 | `sg-pixabay-videos` · `pixabay-search` · `video-trigger` · `video-callback` · `delete-account` |
