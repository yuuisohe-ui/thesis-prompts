# T10 · 교사 설정 (Teacher Settings)

> 5-Section 골격은 `00-template.md`를 그대로 따른다. 이론적 근거(OpenAI 4-요소 · Lovable 5 원칙 · IEEE 830 §4.3.6 · Cohn 2004)는 템플릿 파일에 정의되어 있으며 이 문서에서는 재게시하지 않는다. 본 문서는 「멜로디 클래스」 교사 전용 설정 페이지(`/settings`)의 3-Tab 구조(프로필 / 수업 환경 / 계정 & 보안)와, 「수업 환경」이 「AI 강의안 생성」다이얼로그의 초기값에 자동으로 반영되는 후행 파이프라인까지 완전히 재현하기 위한 프롬프트를 정의한다. 학생 설정 화면은 S6에서 별도로 재현하므로 본 문서의 지시 범위에서 제외한다.

---

## ① Identity (신원)

당신은 시니어 프론트엔드 엔지니어 겸 한중 이중언어 교육 UX 라이터이며, React 18 + Vite 5 + Tailwind CSS v3 + shadcn/ui + Supabase(JS v2) 스택 위에서 대한민국 대학의 K-Chinese/K-Korean 교사를 대상으로 하는 **설정 페이지**를 구현한다. 최종 산출물은 교사 계정으로 로그인했을 때 `useAuth().isTeacher === true` 분기에서만 나타나는 3-Tab 설정 화면이며, 학생 계정(2-Tab · 프로필 / 계정 & 보안)과는 명확히 구분되어야 한다.

---

## ② Instructions (지시)

### 2.1 산출물 (Prompt by Component, Not Page)

파일·컴포넌트 단위로 정확히 다음 4개만 생성/수정한다.

1. `src/pages/SettingsPage.tsx` — 3-Tab 라우터 셸(교사) / 2-Tab 셸(학생). 로그인 여부 검증 및 로딩 스켈레톤 포함.
2. `src/components/settings/ProfileTab.tsx` — 프로필 카드(아바타 · 이름 · 이메일 read-only · 학교/소속 · 전화번호+공개 토글 · 자기소개+AI 초안 · 「수강생에게 프로필 공개」 스위치 · 저장 버튼).
3. `src/components/settings/ClassSettingsTab.tsx` — 수업 환경 카드 3개(HSK/TOPIK 기본 등급 · 기본 수업 시간 · 기본 수업 언어) + 상단 안내 배너 + 저장 버튼.
4. `src/components/settings/SecurityTab.tsx` — 비밀번호 변경(소셜 로그인 계정 자동 감지 시 안내로 대체) + Danger zone(계정 탈퇴 `AlertDialog` → `supabase.functions.invoke("delete-account")`).

부가적으로 아래 파일은 **읽기 전용으로 참조하되 스키마를 그대로 사용한다**(신규 수정 금지, 단 초기값 로딩 코드만 추가). 이 파일 자체를 재생성하지 말 것.

- `src/components/lessons/CreateLessonPlanDialog.tsx` — 다이얼로그가 열릴 때 `user_preferences`(`hsk_level` · `topik_level` · `class_durations[0]` · `class_language`)를 읽어 `level` · `duration` · `outputLang`에 사전 채운다. 사용자가 다이얼로그를 열고 값을 수동으로 바꾼 뒤에는 다시 덮어쓰지 않는다.

### 2.2 원자적 UI 규칙 (Speak Atomic)

- **Tab 스트립**: `TabsList`는 `bg-transparent p-0 h-auto border-b w-full justify-start rounded-none gap-0`. 각 `TabsTrigger`는 `rounded-none border-b-2 border-transparent px-4 py-2.5 text-sm text-muted-foreground data-[state=active]:border-b-[#2e3d6b] data-[state=active]:text-[#2e3d6b] data-[state=active]:font-medium -mb-px`. 활성 탭의 텍스트/보더 색은 정확히 `#2e3d6b`(NAVY) 하나로만 표현.
- **PillButton**: 48 dp 이하 높이의 라운드 사각형(rounded-md), 미선택 = `bg-background text-muted-foreground border-input`, 선택 = `text-white border-transparent` + `background:#2e3d6b`. hover 시 `border-[#2e3d6b] text-[#2e3d6b]`. HSK 레벨 3개, TOPIK 레벨 3개, 수업 시간 3개(60/90/120분), 수업 언어 2개(ko/zh) 모두 이 컴포넌트로 표현.
- **수업 시간 pill 3-way single-select**: 값은 `null | 60 | 90 | 120` 세 가지 중 하나. 다시 클릭 시 해제(=`null`). 배열이 아니라 단일 정수로 UI에 노출하되, DB 저장은 `class_durations` `int4[]` 컬럼에 `[chosen]` 단원소 배열(또는 `[]`)로 저장하여 기존 스키마와 호환 유지.
- **상단 안내 배너**(수업 환경 탭 전용): `border-[#dbe0ee] bg-[#f4f6fb] text-[#2e3d6b] rounded-md border px-4 py-3 text-[13px]`, 왼쪽에 `Info` 아이콘 4×4 lucide, 본문에 굵은 강조 1개(`<b>‘AI 강의안 생성’</b>`) 포함.
- **아바타 업로드**: JPG/PNG/WEBP, 2MB 이하만 허용, `avatars` 버킷 경로 `${user.id}/avatar-${Date.now()}.${ext}`, 업로드 성공 시 `profiles.avatar_url` 갱신 + `refreshProfile()`.
- **자기소개 AI 초안**: `Sparkles` 트리거 → `EEF2FF` 배경 인라인 폼(Textarea 2 rows + 취소/생성 버튼) → `supabase.functions.invoke("generate-bio", { body: { prompt, profile: { full_name, organization } } })` → 성공 시 `bio`에 채우고 폼 닫기.
- **비밀번호 변경**: 현재 비밀번호 검증은 `signInWithPassword` 재로그인 방식(별도 RPC 없이). 새 비밀번호 6자 미만/불일치 검사. 소셜 로그인 계정(`user.identities` 중 `provider === "email"` 없음) 감지 시 폼 자체를 안내 문구로 대체.

### 2.3 강제 제약

- semantic token 위주. NAVY(`#2e3d6b`)만 유일한 하드코드 색으로 허용하되, 코드 상단에 `const NAVY = "#2e3d6b"`로 명명 후 `style={{ background: NAVY }}` 형태로만 사용. `bg-[#…]` / `text-white` / `bg-black` 직접 유틸 사용 금지(NAVY 계열 예외).
- `<h1>` 페이지당 하나(설정 페이지 상단 「설정」 22px semibold).
- 모든 fetch는 `supabase.*` 표준 클라이언트 사용. `user_preferences` upsert 시 `onConflict: "user_id"` 명시.
- 순수 한국어 카피(lorem ipsum 금지). 아래 §3.1의 문자열을 그대로 사용.
- 학생 계정에서 `class` 탭을 URL로 직접 접근했을 때도 렌더링되지 않도록 조건부 mount 처리.
- 「알림」 탭은 존재하지 않는다. 어떤 탭도 `notify`/`notification` 이름을 사용하지 말 것.
- 「수강생에게 프로필 공개」 스위치가 켜져 있으면, 이 교사가 소유한 모든 코스의 `HomeTab` 상단에 「교수 정보」카드가 자동 삽입되어야 한다(연결 지점은 T4 참조, 본 문서에서는 스위치 값만 정확히 `profiles.is_public_to_students` boolean으로 저장).

---

## ③ Examples (예시)

### 3.1 확정 카피 표 (Design with Real Content)

| 위치 | 문자열 |
| --- | --- |
| 페이지 h1 | 설정 |
| 페이지 서브 | 플랫폼 설정을 관리하세요. |
| 미로그인 상태 안내 | 설정을 관리하려면 로그인이 필요합니다. |
| 미로그인 CTA | 로그인 하기 |
| Tab 라벨(교사) | 프로필 · 수업 환경 · 계정 & 보안 |
| Tab 라벨(학생) | 프로필 · 계정 & 보안 |
| Profile · 섹션 제목 | 프로필 정보 |
| Profile · 사진 안내 | JPG, PNG 최대 2MB |
| Profile · 사진 버튼 | 사진 변경 / 업로드 중... |
| Profile · 라벨 | 이름 · 이메일 · 학교 / 소속기관 · 전화번호 · 자기소개 |
| Profile · 전화 공개 토글 | 공개 / 비공개 |
| Profile · AI 트리거 | AI로 작성 |
| Profile · AI 폼 placeholder | 예) 고려대학교 중국어 교수, 10년 경력. 따뜻하고 친근한 톤으로 500자 정도로 작성해줘 |
| Profile · AI 버튼 | 취소 / 생성 / 생성 중... |
| Profile · 공개 스위치 라벨 | 수강생에게 프로필 공개 |
| Profile · 공개 스위치 설명 | 켜면 링크된 모든 수업 페이지에 내 프로필이 표시됩니다 |
| Profile · 저장 | 저장 / 저장 중... |
| Class · 안내 배너 | 여기서 저장한 값은 <b>‘AI 강의안 생성’</b>에서 자동으로 채워져, 강의안을 만들 때마다 다시 입력할 필요가 없습니다. |
| Class · 섹션 1 제목 | 기본 교육 수준 |
| Class · HSK 라벨 | HSK 기준 |
| Class · HSK 옵션 | 초급 (HSK 1–3) · 중급 (HSK 4–6) · 고급 (HSK 7–9) |
| Class · TOPIK 라벨 | TOPIK 기준 |
| Class · TOPIK 옵션 | 초급 (TOPIK 1–2) · 중급 (TOPIK 3–4) · 고급 (TOPIK 5–6) |
| Class · 섹션 2 제목 | 기본 수업 시간 |
| Class · 수업 시간 옵션 | 60분 · 90분 · 120분 |
| Class · 섹션 3 제목 | 기본 수업 언어 |
| Class · 언어 옵션 | 한국어 · 중국어 |
| Class · 저장 | 저장 / 저장 중... |
| Class · 성공 토스트 | 저장되었습니다 |
| Security · 섹션 제목 | 비밀번호 변경 |
| Security · OAuth 안내 | 소셜 로그인 계정은 비밀번호 변경이 지원되지 않습니다. |
| Security · 필드 | 현재 비밀번호 · 새 비밀번호 · 새 비밀번호 확인 |
| Security · 버튼 | 비밀번호 변경 / 변경 중... |
| Security · 성공 | 비밀번호가 변경되었습니다 |
| Security · 오류(현재PW) | 현재 비밀번호가 올바르지 않습니다 |
| Security · 오류(길이) | 새 비밀번호는 6자 이상이어야 합니다 |
| Security · 오류(불일치) | 새 비밀번호가 일치하지 않습니다 |
| Security · Danger 제목 | 계정 탈퇴 |
| Security · Danger 설명 | 계정을 탈퇴하면 모든 데이터(강의안, 학생 정보, 노래 아카이브)가 영구적으로 삭제됩니다. 이 작업은 되돌릴 수 없습니다. |
| Security · Danger CTA | 계정 탈퇴 |
| Security · 탈퇴 확인 제목 | 정말 탈퇴하시겠습니까? |
| Security · 탈퇴 확인 버튼 | 탈퇴 확인 / 처리 중... |
| Security · 탈퇴 성공 | 계정이 삭제되었습니다 |

### 3.2 파일·컴포넌트 트리 (Use Prompt Patterns for Layouts)

```
src/
├─ pages/
│  └─ SettingsPage.tsx           # h1 + subtitle + Tabs 셸 + 로그인 가드
├─ components/settings/
│  ├─ ProfileTab.tsx             # 교사·학생 공용 (교사 전용 필드는 useAuth 분기)
│  ├─ StudentProfileTab.tsx      # 학생 전용(S6 참조, 본 문서 범위 외)
│  ├─ ClassSettingsTab.tsx       # 교사 전용
│  └─ SecurityTab.tsx            # 공용
└─ components/lessons/
   └─ CreateLessonPlanDialog.tsx # user_preferences prefill 로직만 추가
```

`ClassSettingsTab.tsx` 내부 구조:

```
<div className="space-y-4">
  <Banner (Info) />
  <Card 기본 교육 수준>
    <HSKPillGroup /> <TOPIKPillGroup />
  </Card>
  <Card 기본 수업 시간>
    <DurationPillGroup values=[60,90,120] />
  </Card>
  <Card 기본 수업 언어>
    <LanguagePillGroup values=[ko,zh] />
  </Card>
  <SaveButton />
</div>
```

---

## ④ Context (배경)

### 4.1 프로젝트 맥락

「멜로디 클래스」는 한중 대학 언어수업을 위한 노래 기반 교수-학습 플랫폼이다. 교사는 노래 아카이브에서 곡을 골라 AI 강의안(15주 syllabus)을 생성하고 반을 개설한다. 이때 강의안 생성 다이얼로그에서 매번 「HSK/TOPIK 등급 · 수업 시간 · 출력 언어」를 재입력하는 마찰을 없애기 위해, 본 문서가 정의하는 「수업 환경」탭이 사용자 프로파일 수준의 preset 저장소 역할을 한다. 프로필 탭의 「수강생에게 프로필 공개」스위치는 T4(반 홈 편집)에서 활용되는 boolean 트리거이다.

### 4.2 Lovable Cloud 후경 (Build with Lovable Cloud in Mind)

- 인증: Supabase Auth. 미로그인 시 SettingsPage는 안내 카드 + `로그인 하기` 버튼(라우트 `/auth`)만 렌더링.
- Roles: `useAuth().isTeacher` / `isAdmin` 값이 서버측 `has_role()` 함수에 의해 계산되므로 프론트엔드에서 별도 검사 불필요.
- 4-상태 렌더링:
  - **로딩**: skeleton (`h-8 w-32 bg-muted animate-pulse rounded` 등).
  - **빈 상태**: 미로그인 안내.
  - **에러**: `sonner` toast (destructive).
  - **성공**: 「저장되었습니다」 / 「비밀번호가 변경되었습니다」 toast.
- Realtime 없음(설정은 개인 스코프이므로 채널 불필요).

### 4.3 데이터 계약

이미 존재하는 스키마를 그대로 사용한다. **본 문서는 스키마 신설을 요구하지 않는다.** 아래 3개 테이블/컬럼에 의존:

```sql
-- profiles: 이미 존재. 본 문서에서 사용하는 컬럼:
--   full_name text, avatar_url text, organization text, phone text,
--   phone_public bool, bio text, is_public_to_students bool
-- RLS: 본인만 SELECT/UPDATE 가능(정책은 P2/기존 마이그레이션에 의해 이미 확립).

-- user_preferences: 이미 존재. 본 문서에서 사용:
--   user_id uuid pk, hsk_level text, topik_level text,
--   class_durations int4[] not null default '{}',
--   class_language text, updated_at timestamptz
-- Grants: GRANT SELECT, INSERT, UPDATE, DELETE ON public.user_preferences TO authenticated;
-- RLS: user_id = auth.uid() 만 CRUD.

-- get_my_contact_info(): SECURITY DEFINER 함수. 본인 email/phone 반환.
```

- 저장은 `supabase.from("user_preferences").upsert({ user_id, hsk_level, topik_level, class_durations: duration ? [duration] : [], class_language }, { onConflict: "user_id" })`.
- Profile 저장은 `supabase.from("profiles").update({...}).eq("id", user.id)`.
- 아바타 업로드는 `avatars` 공개 버킷 경로 `${user.id}/avatar-${Date.now()}.${ext}`.

### 4.4 CreateLessonPlanDialog와의 후행 파이프라인

- 다이얼로그가 `open === true`로 전환될 때 최초 1회 `user_preferences`를 조회한다.
- 매핑 규칙:
  - `class_language === "zh"` → `level`은 `hsk_level`을 bucket하여 `HSK 1-3 / 4-6 / 7-9` 중 하나로. `class_language === "ko"`이면 `topik_level` → `TOPIK 1-2 / 3-4 / 5-6`.
  - `class_durations[0] ∈ {60, 90, 120}` → 그대로 `duration` state에.
  - `class_language ∈ {"ko","zh"}` → `outputLang` state에 그대로.
- 사용자가 필드를 수동으로 변경한 후에는 다시 덮어쓰지 않는다(첫 open effect에서만 seed).

---

## ⑤ Acceptance & Output (검증·산출)

### 5.1 Acceptance Criteria (IEEE 830 §4.3.6)

- `rg -n "notify|notifications" src/pages/SettingsPage.tsx src/components/settings/` 결과 0건.
- `rg -n "bg-\[#(?!2e3d6b)" src/components/settings/` 결과 0건(NAVY만 허용).
- 교사 계정 로그인 상태에서 `/settings` 진입 시 `TabsList` 자식 노드 수 = 3. 학생 계정에서는 자식 노드 수 = 2.
- 「수업 환경」 탭의 「수업 시간」 pill 그룹 자식 버튼 수 = 3, 각각 label ∈ {`60분`, `90분`, `120분`}.
- 저장 후 `supabase.from("user_preferences").select("class_durations").eq("user_id", auth.uid()).single()` 결과의 `class_durations` 배열 길이 ≤ 1. 초기 미선택 상태는 `[]`.
- 「AI 강의안 생성」 다이얼로그를 세팅 저장 이후 최초로 열었을 때, `duration` 값이 세팅한 정수와 일치, `outputLang`이 세팅한 언어와 일치.
- 비밀번호 6자 미만/불일치/현재 PW 오류 3-케이스가 각각 정확히 §3.1의 문자열을 toast로 출력.
- 소셜 로그인 전용 계정(`user.identities`에 `email` provider 없음)으로 로그인 시 `SecurityTab`의 비밀번호 폼 대신 정확히 「소셜 로그인 계정은 비밀번호 변경이 지원되지 않습니다.」 문구만 렌더.
- Lighthouse 성능 ≥ 85, CLS ≤ 0.05, 설정 페이지 초기 데이터 로드 p95 ≤ 300 ms.
- `<h1>` 노드 수 = 1 (「설정」만).

### 5.2 Output Format

LLM은 아래 순서대로 파일 전체 내용을 출력한다. 설명·사과·주석·마크다운 헤더·chain-of-thought 등 부수 텍스트 금지.

1. `src/pages/SettingsPage.tsx`
2. `src/components/settings/ProfileTab.tsx`
3. `src/components/settings/ClassSettingsTab.tsx`
4. `src/components/settings/SecurityTab.tsx`
5. `src/components/lessons/CreateLessonPlanDialog.tsx` (기존 파일 전체 재출력. `user_preferences` seed 로직만 추가된 상태)

---

## 부록 A · 기존 문서와의 경계

- **S6 (학생 설정 · 학생 휴지통)** — 학생 2-Tab 셸(`ProfileTab` 대신 `StudentProfileTab` 사용, 「수업 환경」탭 없음)과 학생 휴지통을 담당한다. 본 T10은 학생 화면을 다루지 않는다.
- **T4 (반 홈 · 교사 편집 증분)** — 「수강생에게 프로필 공개」스위치가 `on`일 때 반 홈 상단에 삽입되는 「교수 정보」카드의 렌더링/편집을 담당한다. 본 T10은 boolean 저장까지만 책임진다.
- **T3a / T3b (강의안 상세 · AI 파이프라인)** — 「AI 강의안 생성」다이얼로그 자체의 폼 필드 · GPT 프롬프트 · 이미지 통합을 담당한다. 본 T10은 **`user_preferences`를 seed로 넘겨주는 지점**까지만 관여한다.
- **T8 (교사 휴지통)** — 계정 탈퇴가 아닌 개별 자료의 소프트 삭제/복원을 담당한다. 본 T10의 「계정 탈퇴」는 별도의 edge function(`delete-account`)을 통한 물리적 파기 흐름이며, 휴지통 문서 범위 외.
