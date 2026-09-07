# thesis-prompts

박사논문 제4장 「한·중 노래 기반 AI 언어교육 플랫폼 설계 및 구축」에서 사용된 Lovable 프롬프트의 최종본을 모아 놓은 저장소입니다.

## 이 저장소에 대하여

여기 수록된 프롬프트는 개발 과정에서 실제로 입력했던 이력 그대로가 아니라, **플랫폼 복현을 위해 논문이 제시하는 각 모듈의 최종 버전**입니다. 논문 4.2절에서 밝힌 대로, 임의의 연구자가 논문 본문의 기능 설명과 이 저장소의 프롬프트만으로 이론적으로 동일한 시스템을 독립적으로 재구축할 수 있도록 하는 것이 목적입니다.

파일은 실제 개발 저장소(`sino-song-learn`)로부터 GitHub Actions를 통해 자동 동기화되며, 최신 커밋이 그때그때의 최종본을 반영합니다.

## 프롬프트 표준 템플릿

`00-template.md`를 제외한 모든 파일은 다음 5단 구조를 따릅니다.

| 단계 | 내용 |
|---|---|
| ① Identity | 역할, 기술 스택, 언어·문화 정위를 한 문단으로 확정 |
| ② Instructions | 산출물 단위(컴포넌트 단위) 서술, 원자적 UI 서술 규칙, 강제 제약(semantic token, `fetchWithRetry` 등) |
| ③ Examples | 확정 카피 표, 파일/컴포넌트 트리 |
| ④ Context | 프로젝트 맥락, Lovable Cloud 후경(Auth/RLS/GRANT/4상태 렌더링), 데이터 계약(`CREATE TABLE`/`GRANT`/RLS/`CREATE POLICY`) |
| ⑤ Acceptance & Output | IEEE 830 §4.3.6 방식의 정량 임계값 기준, 출력 형식·파일 순서 지정 |

①·②·④는 OpenAI 개발자 메시지 4요소와 Lovable 공식 「Prompting Best Practices」의 5원칙(Prompt by Component / Speak Atomic / Design with Real Content / Use Prompt Patterns for Layouts / Build with Lovable Cloud in Mind)에 근거하며, ⑤는 이 두 문서만으로는 "완성 판정 기준"이 없다는 한계를 보완하기 위해 IEEE(1998)와 Cohn(2004)의 인수기준 논의에 근거해 추가한 항목입니다. 이론적 논증의 전체 서술은 논문 4.2절 도입부를 참고하십시오.

## 파일 ↔ 논문 목차 대응표

> 파일명의 접두 번호(P1, T4, S2 등)는 저장소 내부 관리 번호이며, **논문 절 번호와 반드시 일치하지 않습니다.** 아래 표를 기준으로 찾으시기 바랍니다.

### 4.2절 도입부

| 파일 | 대응 절 |
|---|---|
| `00-template.md` | 4.2 플랫폼 기능의 공통 설계 기반 (도입부 — 프롬프트 표준 템플릿) |

### 4.2.1 ~ 4.2.3 공통 관문 및 기반 인프라

| 파일 | 대응 절 | 내용 |
|---|---|---|
| `P1-home.md` | 4.2.1 랜딩 홈페이지 | 비로그인 방문자용 공개 홈 |
| `P2-auth.md` | 4.2.2 로그인·회원가입 및 신규 사용자 온보딩 흐름 | 인증, 온보딩, 역할(교사/학생) 부여 |
| `P6-infra.md` | 4.2.3 공용 하부 인프라 | 레이아웃, 사이드바, 라우팅, 디자인 토큰, `fetchWithRetry` |

### 4.2.4 노래 아카이브(프론트엔드)

| 파일 | 대응 절 | 내용 |
|---|---|---|
| `P3a-archive-shell.md` | 4.2.4.1 페이지 레이아웃 | 아카이브 페이지 골격 |
| `P3b-topbar-and-add.md` | 4.2.4.2 상단 툴바 및 노래 등록 경로 | 상단바, 곡 추가 진입점 |
| `P3c-youtube-add-flow.md` | 4.2.4.3 YouTube 연동 등록 흐름 | 검색·확인 다이얼로그 |
| `P3d-csv-bulk-import.md` | 4.2.4.4 CSV 일괄 등록 | 대량 가져오기 |
| `P3e-ai-song-generate.md` | 4.2.4.5 AI 곡 생성 기능 | Suno+Pixabay 프론트엔드 종단 흐름 |
| `P3f-hero-and-shelves.md` | 4.2.4.6 홈 추천 및 콘텐츠 발견 진열대 | 히어로, 가로 선반 5종, MiniSongCard |
| `P3g-filters-and-stats.md` | 4.2.4.7 필터링·정렬 및 통계 패널 | |
| `P3h-song-card-grid.md` | 4.2.4.8 노래 카드 및 목록 뷰 | 카드, 그리드, 즐겨찾기 |
| `P3i-share-embed.md` | 4.2.4.9 공유 및 임베드 기능 | |
| `P3j-analysis-dialog-shell.md` | 4.2.4.10 노래 분석 모달 프레임 및 진입 경로 | 4개 하위 기능(가사/어휘/문법/탐구) 진입 |

### 4.2.5 가사·어휘·문법 학습 기능

| 파일 | 대응 절 |
|---|---|
| `P3k-lyrics-tab.md` | 4.2.5.1 가사 학습 탭 |
| `P3l-word-list-tab.md` | 4.2.5.2 어휘 학습 탭 (하위 기능 12종 포함) |
| `P3m-patterns-tab.md` | 4.2.5.3 문법 학습 탭 |

### 4.2.6 학습 활동 및 형성평가 기능

| 파일 | 대응 절 | 비고 |
|---|---|---|
| `P3n-explore-shell-and-info.md` | 4.2.6.1 탐구 모듈 내비게이션 프레임 (+ 4.2.6.2 일부) | 내비게이션 셸 + 곡 정보 4카드(아티스트 스토리/제작 배경/발매 당시 반응/시대 배경, 완결) + 가사 심화 4카드의 카드 UI(상세 페이지 제외) |
| `P3o-explore-lyric.md` | 4.2.6.2 탐구: 정보형 콘텐츠 | 가사 심화 4카드의 상세 페이지(문화 비교/비유·상징/시대 언어 특징/감정 분석) |
| `P3p1-quiz.md` | 4.2.6.3 탐구: 연습—퀴즈 | |
| `P3p2-dictation.md` | 4.2.6.4 탐구: 연습—받아쓰기 | |
| `P3p3-pronunciation.md` | 4.2.6.5 탐구: 연습—발음 연습 | 프론트엔드만 |
| `P3p4-writing.md` | 4.2.6.6 탐구: 연습—AI 글쓰기 챌린지 | |
| `P3q-explore-class-tools.md` | 4.2.6.7 탐구: 수업 도구 | |

### 4.2.7 노래 아카이브(백엔드 처리 파이프라인)

| 파일 | 내용 |
|---|---|
| `B1-song-import-and-transcript.md` | 가져오기 및 자막(가사) 처리 |
| `B2-song-analysis-and-regeneration.md` | 분석 및 재생성 |
| `B3-suno-song-generation.md` | Suno 곡 생성 (4.2.4.5의 백엔드 짝) |

### 4.2.8 반 상세페이지

| 파일 | 대응 절 | 내용 |
|---|---|---|
| `P5a-home-tab-and-edit-mode.md` | 4.2.8.1 페이지 레이아웃 및 편집 모드 프레임 | 라우팅 진입, Hero, 5개 탭 전환바, 편집 스위치, 우측 툴바, 입반자료 작성, 반 메타정보 편집 |
| `P5b-home-blocks.md` | 4.2.8.2 홈 블록 목록 및 편집기 | 8종 블록 유형, 렌더링, "강의안 자동 채우기" |
| `P5c-calendar-and-materials.md` | 4.2.8.3 캘린더 및 수업자료 탭 | |
| `P5d-notifications-and-community.md` | 4.2.8.4 알림 및 반 커뮤니티 탭 | |

### 4.3 교사용 기능 분화

| 파일 | 대응 절 | 내용 |
|---|---|---|
| `T1-teacher-dashboard.md` | 4.3.1.1 대시보드 프레임 및 교사 전용 위젯 | |
| `T1a-common-widgets.md` | 4.3.1.2 공통 위젯 | 11종 (image 모듈 삭제 후) |
| `T1b-deco-and-fun-widgets.md` | 4.3.1.3 장식 및 오락 위젯 | 장식 7 + 재미 6 |
| `T2-workspace.md` | 4.3.2 교사 워크스페이스 | 4개 섹션, 페이지네이션, Fork |
| `T3a-lesson-plan-detail.md` | 4.3.3 강의안 상세: 주차 레이아웃·편집 툴바 | |
| `T3b-lesson-plan-ai-pipeline.md` | 4.3.4 강의안 상세: AI 생성·생성 이력 | |
| `T5-course-basics-and-membership.md` | 4.3.5 반 생성과 학생 입반 | |
| `T4-course-home-teacher-edits.md` | 4.3.6 반 홈 교사 편집 증분 | |
| `T6-course-workspaces-teacher-edits.md` | 4.3.7 캘린더·공지·자료·우리 반 교사 작성층 | |
| `T7-workspace-student-management.md` | 4.3.8 워크스페이스 학생 관리 섹션 상세 | |
| `T8-teacher-trash.md` | 4.3.9 교사 휴지통 | |
| `T10-teacher-settings.md` | 4.3.10 교사 측 설정 | |
| `T9-teacher-guide.md` | 4.3.11 교사 사용 가이드 | |

### 4.4 학생용 기능 분화

| 파일 | 대응 절 | 내용 |
|---|---|---|
| `S2-student-auth-invite-onboarding.md` | 4.4.1 학생 인증 및 최초 진입 | |
| `S1-student-home.md` | 4.4.2 학생 홈 | 기존 구상의 S3 내용이 이 파일에 통합됨 (S3 파일 자체는 존재하지 않음) |
| `S4-student-course-detail.md` | 4.4.3 반 상세페이지 학생 뷰어층 | |
| `S6-student-settings-and-trash.md` | 4.4.4 학생 설정·휴지통 | |
| `S7-student-guide.md` | 4.4.5 학생 사용 가이드 | |
