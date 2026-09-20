# B4 · 배경영상·비동기 렌더 콜백·계정 정리 백엔드 재현 프롬프트

> **본 프롬프트는 B 시리즈의 보완편(4/4) — 백엔드(Supabase Edge Functions) 재현 자료.**
> **적용 대상**: `supabase/functions/sg-pixabay-videos`, `pixabay-search`, `video-trigger`, `video-callback`, `delete-account`
> **관련 데이터**: `render_jobs` 테이블, `songs.video_status` · `songs.video_error` · `songs.youtube_url` · `songs.video_id`, `auth.users`(및 ON DELETE CASCADE 로 연결된 `profiles` · `user_preferences`)
> **본 프롬프트는 `00-template.md` 의 5-Section 골격을 그대로 따른다.**
> **범위 경계**: YouTube 검색·자막 취득·가사 정규화는 B1, 가사의 학습 데이터 변환은 B2, 주제 입력→작사→Suno 작곡→폴링→저장 사슬은 B3 가 책임진다. 본 문서는 논문 4.2.7 후반부 — **① 영상 소재 조달(Pixabay 정지화상·동영상), ② 외부 렌더 저장소로의 비동기 위임과 콜백 수신, ③ 계정 물리 파기** 세 갈래만 책임진다. 화면 측 계약(`/song-generator` 위저드의 `render` 스텝, `YoutubeVideoGenerateDialog`, 설정의 Danger zone)은 각각 P3e · T10 · S6 에서 다루며, 본 문서는 그 화면이 호출하는 요청/응답 계약까지만 규정한다.

---

## 이론적 근거

1. **OpenAI. (n.d.).** *Prompt engineering — developer messages.* https://platform.openai.com/docs/guides/prompt-engineering (Retrieved July 12, 2026)
2. **Lovable. (n.d.).** *Prompting best practices.* https://docs.lovable.dev/prompting/prompting-one (Retrieved July 12, 2026)
3. **IEEE. (1998).** IEEE Std 830-1998, §4.3.6, p. 7.
4. **Cohn, M. (2004).** *User Stories Applied*, Ch. 6, pp. 67–74.
5. **Fielding, R. T. (2000).** *Architectural Styles and the Design of Network-based Software Architectures*, Ch. 5(무상태 제약) — 수 분 단위의 영상 렌더링을 Edge Function 내부 루프가 아니라 외부 작업자 + 콜백으로 분해하는 근거.
6. **Hohpe, G., & Woolf, B. (2003).** *Enterprise Integration Patterns*, "Request-Reply" · "Correlation Identifier", pp. 154–163 — `job_id` 를 상관 식별자로 삼아 요청과 콜백을 짝짓는 설계의 근거.

---

## ① Identity (신원)

당신은 **Supabase Edge Functions(Deno) 기반 미디어 파이프라인·계정 수명주기 엔지니어**다. 담당 범위는 다음 3계층이다.

1. **소재 조달 계층** — Pixabay 공개 API 를 프런트에 직접 노출하지 않고 Edge Function 으로 감싼다. 정지화상(`pixabay-search`)과 동영상(`sg-pixabay-videos`)을 각각 독립 함수로 두고, 어떤 실패 경로에서도 **호출자가 받는 배열의 형태를 무너뜨리지 않는다**.
2. **비동기 위임 계층** — 가사 영상 합성은 Edge Function 의 실행시간·메모리 한도를 넘어서는 작업이므로 **외부 렌더 저장소(GitHub Actions)** 에 위임한다. `video-trigger` 는 작업 행을 만들고 디스패치만 한 뒤 즉시 응답하며, `video-callback` 은 공유 비밀 헤더로 신원을 확인한 뒤 결과를 반영한다. 두 함수 사이를 잇는 유일한 끈은 `render_jobs.id`(상관 식별자)다.
3. **계정 정리 계층** — 사용자가 스스로 계정을 파기할 수 있게 한다. 삭제는 서버 권한(Service Role)으로만 수행하되, **먼저 호출자의 세션을 검증해 "자기 자신만" 삭제**할 수 있게 한다. 연쇄 삭제는 애플리케이션 코드가 아니라 DB 의 `ON DELETE CASCADE` 에 맡긴다.

**전제**: 이미 존재하는 Lovable Cloud 프로젝트(B1–B3 파이프라인 가동 중)에 위 5개 함수를 추가한다. 외부 렌더 저장소 자체의 워크플로 정의는 **본 저장소 범위 밖**이며, 본 문서는 그 저장소와 주고받는 페이로드 규격만 확정한다.

---

## ② Instructions (지시)

### 2.1 산출물 (Prompt by Component, Not Page)

| # | 파일 | 책임 | 외부 의존 | JWT |
|---|---|---|---|---|
| 1 | `sg-pixabay-videos/index.ts` | 배경 동영상 후보 URL 목록(최대 20) | `PIXABAY_API_KEY` | `config.toml` 미등록 = 기본값(검증) |
| 2 | `pixabay-search/index.ts` | 커버·모듈 배경용 정지화상 후보(최대 8) | `PIXABAY_API_KEY` | `config.toml` 미등록 = 기본값(검증) |
| 3 | `video-trigger/index.ts` | `render_jobs` 행 생성 + 외부 저장소 디스패치 | `GH_TOKEN`, Service Role | `verify_jwt = false`(헤더가 있으면 소유자 귀속) |
| 4 | `video-callback/index.ts` | 공유 비밀 검증 + `render_jobs`·`songs` 상태 확정 | `VIDEO_CALLBACK_SECRET`, Service Role | `verify_jwt = false` + 비밀 헤더 검증 |
| 5 | `delete-account/index.ts` | 호출자 본인 계정 물리 파기 | Service Role | **JWT 필수(코드 내 검증)** |

### 2.2 원자적 규칙 (Speak Atomic)

**[A. 공통 인프라]**

- A1. 함수 진입점은 두 계열을 혼용하지 않는다. `sg-pixabay-videos` 는 `https://deno.land/std@0.177.0/http/server.ts`, `pixabay-search` 는 `https://deno.land/std@0.168.0/http/server.ts` 의 `serve()` 를 사용하고, `video-trigger` · `video-callback` · `delete-account` 는 `Deno.serve()` 를 직접 사용한다.
- A2. `corsHeaders` 는 **두 가지 프로필**을 둔다.
  - **P-프로필**(Pixabay 2종): `Access-Control-Allow-Origin: *`, 허용 헤더 `authorization, x-client-info, apikey, content-type` + `x-supabase-client-platform/-version`, `x-supabase-client-runtime/-version` 6종.
  - **V-프로필**(`video-trigger`·`video-callback`): 허용 헤더 `authorization, content-type, apikey, x-client-info` 4종 + `Access-Control-Allow-Methods: "POST, OPTIONS"`.
  - `delete-account` 는 `authorization, x-client-info, apikey, content-type` 4종만 둔다(메서드 헤더 없음).
- A3. `OPTIONS` 요청은 본문 없이(또는 `"ok"` 문자열로) `corsHeaders` 만 반환한다.
- A4. 모든 성공·실패 응답에 `corsHeaders` 를 병합하고, 실패 본문은 `{ error: string }` 으로 통일한다.
- A5. **비밀값은 오직 `Deno.env.get()` 로만 읽는다.** 응답 본문·로그·에러 메시지에 키 값, 토큰, 콜백 비밀, 외부 저장소 접근 자격을 절대 포함하지 않는다. 실패 로그에는 HTTP 상태 코드와 외부 응답 본문 앞부분만 남긴다.

**[B. `sg-pixabay-videos` — 배경 동영상 후보]**

> B3 §2.2 [G] 와 **동일한 함수**를 다른 관점(소재 조달 계층)에서 재기술한 것이다. 두 문서의 규칙은 항목 단위로 일치해야 하며, 충돌 시 코드가 기준이다.

- B1. 입력은 GET 쿼리(`q`, `minDuration`, `limit`) 또는 POST 본문(`{ q, minDuration, limit }`) 모두 허용. 기본값 `minDuration = 4`, `limit = 20`.
- B2. `q` 의 `+` 를 공백으로 치환하고 다중 공백을 축약한다. 비면 `"nature sky sunlight"`.
- B3. 호출: `GET https://pixabay.com/api/videos/` · `video_type=film` · `safesearch=true` · `per_page=20`.
- B4. `duration >= minDuration` 이고 `videos.medium.url` 이 존재하는 항목만 채택하고, 그 URL 문자열만 수집한다.
- B5. 결과가 **3개 미만이면 폴백 질의** `"nature sky sunlight people city journey"` 를 1회 추가 수행해 합집합을 만든다(빈 배경 방지).
- B6. 중복 제거 → **Fisher–Yates 셔플** → `limit` 개 절단. 응답 `{ bgVideoList: string[] }`.
- B7. 예외 발생 시에도 `{ error, bgVideoList: [] }` 를 HTTP 500 으로 반환해 **배열 계약을 유지**한다(프런트가 `.map()` 으로 바로 터지지 않게).
- B8. `PIXABAY_API_KEY` 부재는 예외로 던져 위 B7 경로로 흘려보낸다.

**[C. `pixabay-search` — 정지화상 후보]**

- C1. 입력은 POST 본문 `{ keywords: string[], lang?, perKeyword?, totalLimit?, excludeIds?: number[] }`.
- C2. **키워드 정규화**: 문자열만 남기고 trim → 빈 문자열과 60자 초과 제거 → `Set` 중복 제거 → **최대 6개**로 절단. 결과가 0개면 즉시 `{ assets: [] }` + HTTP 200.
- C3. **수치 클램프**: `perKeyword` 는 1–5, `totalLimit` 은 1–8 로 강제한다. 프런트가 더 큰 값을 보내도 서버가 잘라낸다.
- C4. `lang` 기본값 `"zh"`. Pixabay 가 지원하는 코드 중 `zh · ko · en · ja` 일 때만 `lang` 파라미터를 붙이고, 그 외에는 생략한다.
- C5. 호출: `GET https://pixabay.com/api/` · `image_type=photo` · `safesearch=true` · `per_page=20` · `orientation=horizontal`.
- C6. **인메모리 캐시**: 키 `"<lang>::<소문자·trim 키워드>"`, TTL **5분**. 웜 인스턴스 내에서만 유효한 최선노력(best-effort) 캐시이며, 캐시 미스여도 동작은 동일하다.
- C7. 응답 항목 매핑: `{ id, url: webformatURL, thumb: previewURL, pageURL, user, keyword }`. `keyword` 를 실어 보내 프런트가 "어느 키워드에서 온 이미지인지" 표시할 수 있게 한다.
- C8. **키워드별 풀 확보**: 키워드마다 `max(perKeyword * 4, 4)` 개를 미리 받아두고, `excludeIds` 에 포함된 `id` 는 제외한다(사용자가 "다른 이미지" 를 누를 때 같은 결과가 반복되지 않게).
- C9. **라운드로빈 수집**: 키워드 풀들을 번갈아 1개씩 꺼내 `totalLimit` 까지 채운다. 이미 수집한 `id` 는 건너뛴다. 어느 풀에서도 더 꺼낼 게 없으면 즉시 종료하고, 안전 카운터 200회 상한을 둔다. → 키워드 1개가 결과를 독식하지 않는다.
- C10. 외부 호출은 **키워드 순차 실행**(Pixabay 레이트 리밋 배려). 개별 키워드 실패는 `console.error` 후 빈 배열로 취급하고 전체를 실패시키지 않는다.
- C11. `PIXABAY_API_KEY` 부재 시 HTTP 500 + `{ error: "PIXABAY_API_KEY is not configured" }`.
- C12. 응답은 항상 `{ assets: ImageAsset[] }`.

**[D. `video-trigger` — 외부 렌더 저장소 위임]**

- D1. 입력 `{ song_id?, audio_url, song_title?, aligned_words?, cover_url?, video_keywords? }`. `audio_url` 이 없으면 HTTP 400 + `{ error: "audio_url required" }`.
- D2. **소유자 귀속(선택적 인증)**: `Authorization: Bearer …` 헤더가 있으면 anon 키 클라이언트로 `auth.getClaims(token)` 를 호출해 `claims.sub` 를 `owner_id` 로 쓴다. 헤더가 없거나 검증이 실패하면 `owner_id = null` 로 두고 작업은 계속한다(익명 호출을 막지는 않되 소유권도 주지 않는다).
- D3. 이후 모든 DB 조작은 Service Role 클라이언트로 수행한다.
- D4. `render_jobs` 에 1행 삽입: `{ song_id, owner_id, status: "dispatched", audio_url, cover_url, song_title, dispatched_at: now }`. 삽입 실패는 즉시 예외(작업 추적 불가능한 디스패치를 만들지 않는다).
- D5. `song_id` 가 있으면 `songs` 를 `{ video_status: "rendering", video_error: null }` 로 갱신한다. → 목록 화면이 진행 중 배지를 즉시 표시한다.
- D6. **디스패치**: `POST https://api.github.com/repos/{OWNER}/{RENDERER_REPO}/dispatches`(외부 렌더 저장소 `melody-video-renderer`). 헤더 `Authorization: Bearer <GH_TOKEN>`, `Accept: application/vnd.github+json`, `Content-Type: application/json`, `User-Agent: lovable-video-trigger`. `GH_TOKEN` 부재는 예외.
- D7. **`client_payload` 규격**(외부 저장소와의 계약 — 필드명 변경 금지):
  `{ job_id, song_id, audio_url, cover_url, song_title, video_keywords, aligned_words, callback_url, callback_secret }`.
  `callback_url` 은 `"<SUPABASE_URL>/functions/v1/video-callback"` 로 서버가 조립하고, `aligned_words` 는 없으면 `[]`.
- D8. **디스패치 실패 보상**: GitHub 응답이 `ok` 가 아니면 `render_jobs` 를 `{ status: "failed", error_message: "GitHub dispatch failed <status>: <본문>", finished_at: now }` 로, `songs` 를 `{ video_status: "failed", video_error: "GitHub dispatch failed: <status>" }` 로 되돌린 뒤 예외를 던진다. **"rendering 인데 아무도 렌더하지 않는" 유령 상태를 남기지 않는다.**
- D9. 성공 응답 `{ ok: true, job_id }`. 함수는 렌더 완료를 기다리지 않는다(위임 즉시 종료).

**[E. `video-callback` — 결과 수신과 상태 확정]**

- E1. **인증은 공유 비밀 헤더로만 한다.** `x-callback-secret` 헤더, 또는 `Authorization: Bearer <값>` 의 값이 `VIDEO_CALLBACK_SECRET` 과 **완전 일치**할 때만 통과한다. 환경변수가 비어 있거나 값이 다르면 HTTP 401 + `{ error: "Unauthorized" }` 이며 **어떤 행도 변경하지 않는다**. 사용자 JWT 는 사용하지 않는다(호출자가 외부 CI 이므로 세션이 없다).
- E2. 입력 `{ job_id, status, youtube_video_id?, youtube_url?, error? }`. `job_id` 또는 `status` 가 없으면 HTTP 400.
- E3. `render_jobs` 에서 `job_id` 로 `song_id` 를 조회한다. 없으면 예외(`job not found`) — **미지의 `job_id` 로 임의의 곡을 조작할 수 없다**.
- E4. `status === "done"` 이면: `render_jobs` ← `{ status: "done", finished_at: now, youtube_video_id, youtube_url }`, `song_id` 가 있으면 `songs` ← `{ video_status: "done", youtube_url, video_id: youtube_video_id }`.
- E5. 그 외 모든 `status` 는 실패로 간주한다: `render_jobs` ← `{ status: "failed", error_message: error ?? "unknown error", finished_at: now }`, `songs` ← `{ video_status: "failed", video_error: error ?? "unknown error" }`.
- E6. **멱등성**: 같은 `job_id` 로 동일 페이로드가 재전송돼도 결과는 같아야 한다(모든 쓰기가 덮어쓰기이며 누적 증감 연산을 쓰지 않는다). 외부 CI 의 재시도를 허용하기 위한 조건이다.
- E7. 성공 응답 `{ ok: true }`. 프런트에는 아무것도 푸시하지 않는다 — 진행 상황은 프런트가 `render_jobs` 를 직접 폴링해 확인한다(P3e: 5초 간격, 최대 20분, 초과 시 자동 취소 후 이전 단계 회귀).

**[F. `delete-account` — 계정 물리 파기]**

- F1. `Authorization` 헤더가 없으면 HTTP 401 + `{ error: "Unauthorized" }`.
- F2. anon 키 + 호출자 헤더로 만든 **사용자 클라이언트**로 `auth.getUser()` 를 호출해 신원을 확정한다. 실패하면 HTTP 401 + `{ error: "Invalid session" }`.
- F3. 확정된 `user.id` 만 삭제 대상으로 삼는다. **요청 본문에서 대상 사용자 id 를 받지 않는다**(타인 계정 삭제 경로를 구조적으로 차단).
- F4. 삭제는 Service Role 클라이언트의 `auth.admin.deleteUser(uid)` 로 수행한다. 실패 시 서버 로그를 남기고 HTTP 500 + `{ error: <메시지> }`.
- F5. **연쇄 삭제는 애플리케이션이 하지 않는다.** `profiles`, `user_preferences` 등 `auth.users(id)` 를 참조하는 테이블은 `ON DELETE CASCADE` 로 자동 정리된다. 함수 안에서 개별 `DELETE` 를 나열하지 않는다(누락·순서 오류 방지).
- F6. 성공 응답 `{ ok: true }`. 프런트는 이 응답을 받은 뒤 `signOut()` → `/` 이동을 수행한다(T10 · S6).

### 2.3 강제 제약

- **긴 작업은 함수 안에서 기다리지 않는다.** `video-trigger` 에 `sleep`·폴링 루프를 두지 않는다. 대기는 외부 작업자가, 관측은 프런트 폴링이 담당한다.
- **상관 식별자는 `render_jobs.id` 하나뿐이다.** 외부 저장소는 `song_id` 만으로 상태를 갱신할 수 없고 반드시 `job_id` 를 돌려줘야 한다.
- **상태는 두 곳에 이중 기록한다.** 상세 이력은 `render_jobs`, 목록 화면용 요약은 `songs.video_status`. 한쪽만 갱신하는 경로를 만들지 않는다(성공·실패·디스패치 실패 3경로 모두 쌍으로 갱신).
- **`video_status` 어휘는 4개로 고정**: `idle · rendering · done · failed`. `render_jobs.status` 는 `queued · dispatched · rendering · uploading · done · failed`. 두 어휘를 뒤섞지 않는다.
- **비밀값 문서화 금지**: 본 프롬프트와 코드 주석 어디에도 실제 토큰·콜백 비밀·외부 서비스의 완전한 내부 주소를 적지 않는다. 환경변수 **이름만** 기술한다.
- **Pixabay 응답을 프런트가 직접 받지 않는다.** 두 함수 모두 API 키를 서버에 가둔 채 정규화된 최소 필드만 반환한다.
- **실패해도 형태를 지킨다.** `sg-pixabay-videos` 는 `bgVideoList: []`, `pixabay-search` 는 `assets: []` — 호출자가 분기 없이 순회할 수 있어야 한다.

---

## ③ Examples (예시)

### 3.1 확정 문자열·계약 표 (Design with Real Content)

| 위치 | 확정 문자열 / 값 |
|---|---|
| Pixabay 이미지 API | `https://pixabay.com/api/` |
| Pixabay 영상 API | `https://pixabay.com/api/videos/` |
| 영상 기본 질의 | `nature sky sunlight` |
| 영상 폴백 질의 | `nature sky sunlight people city journey` |
| 이미지 캐시 TTL | 5분(300,000 ms) |
| 이미지 키워드 상한 | 6개 · 키워드 길이 ≤ 60자 |
| 이미지 수치 클램프 | `perKeyword` 1–5 · `totalLimit` 1–8 |
| 영상 기본값 | `minDuration = 4` · `limit = 20` |
| 디스패치 대상 | 외부 렌더 저장소 `melody-video-renderer` (GitHub `repository_dispatch`) |
| 디스패치 이벤트 타입 | `render-video` |
| 디스패치 User-Agent | `lovable-video-trigger` |
| 콜백 경로 | `<SUPABASE_URL>/functions/v1/video-callback` |
| 콜백 비밀 헤더 | `x-callback-secret` (또는 `Authorization: Bearer`) |
| 콜백 거부 응답 | HTTP 401 · `{ "error": "Unauthorized" }` |
| 디스패치 실패 메시지 | `GitHub dispatch failed <status>: <본문>` |
| 콜백 미상 실패 메시지 | `unknown error` |
| `songs.video_status` 어휘 | `idle · rendering · done · failed` |
| `render_jobs.status` 어휘 | `queued · dispatched · rendering · uploading · done · failed` |
| 탈퇴 실패 응답 | `{ "error": "Unauthorized" }` / `{ "error": "Invalid session" }` |
| 탈퇴 성공 응답 | `{ "ok": true }` |

**함수별 요청/응답 계약**

| 함수 | 요청 | 응답 |
|---|---|---|
| `sg-pixabay-videos` | GET `?q=&minDuration=&limit=` 또는 POST `{ q, minDuration, limit }` | `{ bgVideoList: string[] }` (실패 시 `{ error, bgVideoList: [] }`) |
| `pixabay-search` | POST `{ keywords, lang, perKeyword, totalLimit, excludeIds }` | `{ assets: { id, url, thumb, pageURL, user, keyword }[] }` |
| `video-trigger` | POST `{ song_id, audio_url, song_title, aligned_words, cover_url, video_keywords }` | `{ ok: true, job_id }` |
| `video-callback` | POST `{ job_id, status, youtube_video_id, youtube_url, error }` + 비밀 헤더 | `{ ok: true }` |
| `delete-account` | POST(본문 없음) + `Authorization: Bearer <세션 JWT>` | `{ ok: true }` |

**외부 저장소로 보내는 `client_payload` 구조**(값은 예시가 아니라 필드 형태만 확정)

```json
{
  "job_id": "<render_jobs.id>",
  "song_id": "<songs.id | null>",
  "audio_url": "<공개 오디오 URL>",
  "cover_url": "<커버 URL | null>",
  "song_title": "<곡명 | null>",
  "video_keywords": "<ocean+sunset+waves | null>",
  "aligned_words": [],
  "callback_url": "<SUPABASE_URL>/functions/v1/video-callback",
  "callback_secret": "<환경변수에서 주입 — 문서에 값 기재 금지>"
}
```

### 3.2 파이프라인 트리 (Use Prompt Patterns for Layouts)

```
[소재 조달]
  sg-pixabay-videos ──► Pixabay /videos  ──► 필터(duration·medium.url)
                          └ 3개 미만이면 폴백 질의 1회 ──► 합집합
                                        └ 중복 제거 → 셔플 → limit 절단
                                                       └► { bgVideoList[] }

  pixabay-search ──► 키워드 ≤6 정규화 ──► (5분 캐시 조회)
                        └ 키워드별 풀 = max(perKeyword×4, 4), excludeIds 제외
                              └ 라운드로빈 수집 → totalLimit
                                       └► { assets[{id,url,thumb,pageURL,user,keyword}] }

[비동기 렌더 위임]
  프런트 ──► video-trigger
               ├─ (Authorization 있으면) getClaims → owner_id
               ├─ render_jobs INSERT (status = dispatched, dispatched_at)
               ├─ songs UPDATE (video_status = rendering, video_error = null)
               └─ POST repository_dispatch → 외부 렌더 저장소
                       │            (event_type = render-video, client_payload)
                       │   실패 시 ─► render_jobs = failed + songs = failed  ◄─ 보상
                       ▼
              [외부 작업자: 영상 합성 · 업로드]  ※ 본 저장소 범위 밖
                       │
                       ▼  POST + x-callback-secret
                  video-callback
                       ├─ 비밀 불일치 ─► 401, 무변경
                       ├─ job_id 조회 실패 ─► 예외
                       ├─ done  ─► render_jobs(done, youtube_*) + songs(done, youtube_url, video_id)
                       └─ 그 외 ─► render_jobs(failed, error_message) + songs(failed, video_error)

  프런트 ── 5s 간격 SELECT render_jobs (최대 20분) ──► 배지·스텝 전환

[계정 정리]
  설정 Danger zone ──► delete-account
                          ├─ Authorization 없음 ─► 401 Unauthorized
                          ├─ getUser() 실패 ─► 401 Invalid session
                          └─ admin.deleteUser(자기 uid)
                                   └ auth.users 삭제 → profiles · user_preferences 자동 CASCADE
                          ◄─ { ok: true } ─► signOut() → "/"
```

---

## ④ Context (배경)

### 4.1 프로젝트 맥락

「멜로디 클래스」의 AI 창작곡은 B3 단계에서 **오디오와 단어 단위 타임스탬프**까지 확보된다. 그러나 교실에서 실제로 재생되는 형태는 "가사가 흐르는 영상"이다. 영상 합성은 수 분에 걸친 CPU·대역폭 집약 작업이라 Edge Function 의 실행 한도 안에서 끝낼 수 없다. 그래서 본 모듈은 **합성 자체를 하지 않고, 합성을 시킬 뿐**이다 — 작업을 기록하고, 외부 저장소를 깨우고, 결과를 받아 상태를 확정한다. Pixabay 두 함수는 그 영상과 카드·강의안 배경에 쓰일 소재를 저작권 안전하게 조달하는 공용 창구이며, `delete-account` 는 개인정보 처리 관점에서 수집한 계정을 사용자 의사로 파기하는 출구다. 즉 본 문서는 논문 4.2.7 후반 — **"본문 생성 이후의 주변 설비"** 세 가지를 다룬다.

### 4.2 Lovable Cloud 후경

- **시크릿**: `PIXABAY_API_KEY`, `GH_TOKEN`, `VIDEO_CALLBACK_SECRET`, `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`. 값은 어디에도 기록하지 않는다.
- **`supabase/config.toml`**: `video-trigger`, `video-callback` 두 항목만 `verify_jwt = false` 로 등록한다. `sg-pixabay-videos` · `pixabay-search` · `delete-account` 는 등록하지 않아 기본값(JWT 검증)을 따르며, `delete-account` 는 그와 별개로 코드 안에서 세션을 한 번 더 검증한다.
- **인증 모델 3종**: ① 플랫폼 JWT(`delete-account`, Pixabay 2종) ② 선택적 JWT(`video-trigger` — 있으면 소유자 귀속) ③ 공유 비밀 헤더(`video-callback` — 기계 대 기계).
- **4-상태 렌더링**(호출 화면 공통, P3e·T4·P5a 기준): **로딩** = `Loader2 animate-spin`(영상 렌더는 진행 배지 + 경과 시간), **빈** = `키워드로 검색해보세요.` 류 안내, **에러** = destructive 배너 + 한국어 문구 + 재시도 버튼, **성공** = 후보 그리드 렌더 또는 다음 스텝 전환.
- **실시간 갱신 없음**: 콜백은 프런트로 푸시하지 않는다. 진행 표시는 전적으로 `render_jobs` 폴링에 의존하며, 20분 초과 시 프런트가 폴링을 중단하고 이전 단계로 회귀시킨다.

### 4.3 데이터 계약

> `render_jobs` 정본 정의와 `songs` 컬럼 추가분은 **P3e §4.3 과 동일**하다. 아래는 본 문서 범위에서 반드시 성립해야 하는 부분만 재게시한 것이며, 두 문서가 어긋나면 P3e 를 정본으로 본다.

```sql
CREATE TABLE IF NOT EXISTS public.render_jobs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  song_id uuid REFERENCES public.songs(id) ON DELETE CASCADE,
  owner_id uuid,
  status text NOT NULL DEFAULT 'queued',   -- queued/dispatched/rendering/uploading/done/failed
  audio_url text,
  cover_url text,
  song_title text,
  youtube_video_id text,
  youtube_url text,
  error_message text,
  dispatched_at timestamptz,
  finished_at timestamptz,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

GRANT SELECT, INSERT, UPDATE, DELETE ON public.render_jobs TO authenticated;
GRANT ALL ON public.render_jobs TO service_role;

ALTER TABLE public.render_jobs ENABLE ROW LEVEL SECURITY;

CREATE POLICY owned_select ON public.render_jobs FOR SELECT
  USING (owner_id IS NULL OR owner_id = auth.uid() OR public.is_admin());
CREATE POLICY owned_insert ON public.render_jobs FOR INSERT
  WITH CHECK (public.is_admin() OR owner_id = auth.uid());
CREATE POLICY owned_update ON public.render_jobs FOR UPDATE
  USING (owner_id = auth.uid() OR public.is_admin())
  WITH CHECK (owner_id = auth.uid() OR public.is_admin());
CREATE POLICY owned_delete ON public.render_jobs FOR DELETE
  USING (owner_id = auth.uid() OR public.is_admin());

CREATE INDEX IF NOT EXISTS render_jobs_song_id_idx ON public.render_jobs(song_id);
CREATE INDEX IF NOT EXISTS render_jobs_owner_id_idx ON public.render_jobs(owner_id);
```

```sql
ALTER TABLE public.songs
  ADD COLUMN IF NOT EXISTS video_status text NOT NULL DEFAULT 'idle',  -- idle/rendering/done/failed
  ADD COLUMN IF NOT EXISTS video_error text,
  ADD COLUMN IF NOT EXISTS youtube_url text,
  ADD COLUMN IF NOT EXISTS video_id text,
  ADD COLUMN IF NOT EXISTS bg_video_list jsonb,
  ADD COLUMN IF NOT EXISTS video_keywords text;
```

**RLS 와 Service Role 의 역할 분담**: `video-callback` 은 사용자 세션이 없으므로 위 정책 아래에서는 어떤 행도 갱신할 수 없다. 따라서 콜백·트리거 두 함수는 Service Role 로 RLS 를 우회하되, **그 우회를 정당화하는 유일한 근거가 E1 의 공유 비밀 검증과 E3 의 `job_id` 존재 확인**이다. 이 두 검증이 빠지면 함수 전체가 임의 곡 조작 창구가 된다.

**계정 파기의 데이터 계약**: `auth` 스키마는 직접 조작하지 않는다. `profiles`, `user_preferences` 등 사용자 종속 테이블은 `user_id uuid REFERENCES auth.users(id) ON DELETE CASCADE` 로 선언돼 있어야 하며, 이 선언이 곧 파기 범위의 명세다. Pixabay 두 함수는 DB 를 읽거나 쓰지 않는다(무상태 프록시).

---

## ⑤ Acceptance & Output (검증·산출)

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6 — 정량 임계값)

1. **영상 후보 최소 개수**: 임의 키워드 10건 중 **9건 이상**에서 `sg-pixabay-videos` 의 `bgVideoList` 길이가 3 이상이며, 모든 항목이 `https://` 로 시작한다.
2. **영상 실패 계약**: `PIXABAY_API_KEY` 를 제거한 상태의 응답이 HTTP 500이면서 본문에 `bgVideoList: []` 배열이 존재한다(= 프런트 순회 오류 0건).
3. **이미지 클램프**: `perKeyword = 99`, `totalLimit = 99`, 키워드 20개로 호출해도 `assets` 길이 ≤ 8 이고 외부 API 호출 횟수 ≤ 6 이다.
4. **이미지 중복 제거**: 1차 응답의 `id` 전부를 `excludeIds` 로 넣어 재호출하면, 반환된 `assets` 중 1차와 겹치는 `id` 가 **0건**이다.
5. **키워드 분산**: 서로 다른 키워드 4개 · `totalLimit = 8` 호출 시 결과에 **3개 이상**의 서로 다른 `keyword` 값이 포함된다.
6. **캐시**: 동일 키워드·언어로 5분 이내 재호출 시 외부 API 호출이 0회 발생한다(웜 인스턴스 기준, 로그로 확인).
7. **트리거 입력 검증**: `audio_url` 누락 호출이 HTTP 400이며 `render_jobs` 행이 **0건** 생성된다.
8. **디스패치 성공 기록**: 정상 호출 후 `render_jobs` 에 `status = "dispatched"` 행이 1건, 동일 `song_id` 의 `songs.video_status = "rendering"` 이 **동시에** 성립한다(양쪽 불일치 0건).
9. **디스패치 실패 보상**: `GH_TOKEN` 을 무효화한 상태에서 호출하면 60초 이내에 `render_jobs.status = "failed"` 이고 `songs.video_status = "failed"` 이며, `rendering` 상태로 남는 행이 **0건**이다.
10. **콜백 인증**: 비밀 헤더 없이 또는 틀린 값으로 `video-callback` 을 호출하면 **100%** HTTP 401이고, 대상 `render_jobs`·`songs` 행의 `updated_at` 이 변하지 않는다.
11. **콜백 상관 검증**: 존재하지 않는 `job_id` 로 정상 비밀과 함께 호출해도 어떤 `songs` 행도 변경되지 않는다(변경 행 0건).
12. **콜백 멱등성**: 동일 `done` 페이로드를 3회 연속 전송한 뒤 `render_jobs.status`, `songs.video_status`, `songs.youtube_url` 이 1회 전송 시와 **완전히 동일**하다.
13. **엔드투엔드 지연**: 외부 렌더 완료 콜백 수신 시각과 프런트 배지가 `done` 으로 바뀌는 시각의 차이가 **폴링 주기 + 1초(≤ 6초)** 이내.
14. **탈퇴 권한**: JWT 없이 `delete-account` 호출 시 HTTP 401이며 `auth.users` 행 수가 변하지 않는다. 타인 id 를 본문에 실어 보내도 삭제되는 계정은 **호출자 본인 1건뿐**이다.
15. **탈퇴 연쇄**: 탈퇴 성공 후 해당 `user_id` 를 참조하는 `profiles` · `user_preferences` 행이 **0건**으로 남는다(수동 삭제 코드 없이).
16. **비밀 누출**: 5개 함수의 모든 성공·실패 응답 본문과 로그를 `grep` 했을 때 `PIXABAY_API_KEY` · `GH_TOKEN` · `VIDEO_CALLBACK_SECRET` · Service Role 키의 값이 **0건** 노출된다.
17. **CORS**: 각 함수의 프로필에 맞는 프리플라이트가 모두 204/200으로 통과하며, `video-trigger`·`video-callback` 응답에 `Access-Control-Allow-Methods: POST, OPTIONS` 가 포함된다.

### 5.2 Output Format

1. `supabase/functions/sg-pixabay-videos/index.ts`
2. `supabase/functions/pixabay-search/index.ts`
3. `supabase/functions/video-trigger/index.ts`
4. `supabase/functions/video-callback/index.ts`
5. `supabase/functions/delete-account/index.ts`
6. `supabase/config.toml` 의 `[functions.video-trigger]` · `[functions.video-callback]` 항목(각 `verify_jwt = false`)
7. `render_jobs` 및 `songs` 컬럼 추가 마이그레이션 SQL(§4.3, 이미 적용돼 있으면 생략)

모든 함수는 TypeScript/Deno 표준 API만 사용한다(Node·Cloudflare 전용 API 금지). 설명·사과·주석성 산문·마크다운 헤더 등 부수 텍스트는 출력하지 않는다.
