# 00-index · 논문 목차 ↔ 프롬프트 문서 대응표

> 본 문서는 학위논문 제4장의 절 번호와 본 부록의 프롬프트 재현 파일을 1:1로 연결한다.
> 저장소 현황(총 51개 파일): 템플릿 1 · README 1 · 공용 P 계열 27 · 교사 T 계열 13 · 학생 S 계열 5 · 백엔드 B 계열 4.
> 파일 번호(P·T·S·B)는 개발 과정에서 부여한 식별자이며 연속하지 않을 수 있다.

## 4.2 플랫폼 기능의 공통 설계 기반

| 절 | 파일 | 담당 범위 |
|---|---|---|
| 4.2 도입부 | `00-template.md` | 5-Section 프롬프트 표준 템플릿 및 이론적 근거 |
| 4.2.1 랜딩 홈페이지 | `P1-home.md` | 비로그인 공개 홈 |
| 4.2.2 로그인·회원가입 및 온보딩 | `P2-auth.md` | 인증, 온보딩, 역할 부여 |
| 4.2.3 공용 하부 인프라 | `P6-infra.md` | 레이아웃·사이드바·라우팅·디자인 토큰·`fetchWithRetry` |

### 4.2.4 노래 아카이브(프론트엔드)

| 절 | 파일 |
|---|---|
| 4.2.4.1 페이지 레이아웃 | `P3a-archive-shell.md` |
| 4.2.4.2 상단 툴바 및 노래 등록 경로 | `P3b-topbar-and-add.md` |
| 4.2.4.3 YouTube 연동 등록 흐름 | `P3c-youtube-add-flow.md` |
| 4.2.4.4 CSV 일괄 등록 | `P3d-csv-bulk-import.md` |
| 4.2.4.5 AI 곡 생성 기능 | `P3e-ai-song-generate.md` |
| 4.2.4.6 홈 추천 및 진열대 | `P3f-hero-and-shelves.md` |
| 4.2.4.7 필터링·정렬 및 통계 패널 | `P3g-filters-and-stats.md` |
| 4.2.4.8 노래 카드 및 목록 뷰 | `P3h-song-card-grid.md` |
| 4.2.4.9 공유 및 임베드 | `P3i-share-embed.md` |
| 4.2.4.10 노래 분석 모달 프레임 | `P3j-analysis-dialog-shell.md` |

### 4.2.5 가사·어휘·문법 학습 기능

| 절 | 파일 |
|---|---|
| 4.2.5.1 가사 학습 탭 | `P3k-lyrics-tab.md` |
| 4.2.5.2 어휘 학습 탭 | `P3l-word-list-tab.md` |
| 4.2.5.3 문법 학습 탭 | `P3m-patterns-tab.md` |

### 4.2.6 학습 활동 및 형성 평가 기능

| 절 | 파일 | 비고 |
|---|---|---|
| 4.2.6.1 탐구 모듈 내비게이션 프레임 | `P3n-explore-shell-and-info.md` | 셸 + `곡 정보` 4모듈 + `가사 심화` 4카드의 카드 UI |
| 4.2.6.2 탐구: 정보형 콘텐츠 | `P3o-explore-lyric.md` | 가사 심화 4모듈의 **상세 페이지** |
| 4.2.6.3 탐구: 퀴즈 | `P3p1-quiz.md` | |
| 4.2.6.4 탐구: 받아쓰기 | `P3p2-dictation.md` | |
| 4.2.6.5 탐구: 발음 연습 | `P3p3-pronunciation.md` | 프론트엔드 + `speech-evaluate` 계약 포함 |
| 4.2.6.6 탐구: AI 글쓰기 챌린지 | `P3p4-writing.md` | |
| 4.2.6.7 탐구: 수업 도구 | `P3q-explore-class-tools.md` | |

**정보형 콘텐츠 8종의 파일별 분할 (경계 확정)**

| 파일 | 카드/모듈 |
|---|---|
| `P3n` — 곡 정보 4모듈 | 아티스트 스토리 · 제작 배경 · 발매 당시 반응 · 시대 배경 |
| `P3n` — 가사 심화 4카드(카드 UI만) | 문화 비교 · 가사 속 비유/상징 · 시대 언어 특징 · 감정 분석 |
| `P3o` — 가사 심화 4모듈(상세 화면) | `CultureComparePage` · `MetaphorSymbolPage` · `EraLanguagePage` · `EmotionAnalysisPage` |

즉 "카드 = P3n, 상세 화면 = P3o" 가 경계선이며, 4.2.6.1 본문에는 8개 카드의 진입 UI를, 4.2.6.2 본문에는 가사 심화 4개 상세 화면을 서술한다.

### 4.2.7 노래 아카이브(백엔드 처리 파이프라인)

| 파일 | 내용 |
|---|---|
| `B1-song-import-and-transcript.md` | 곡 검색·가져오기·자막(가사) 확보 및 정규화 |
| `B2-song-analysis-and-regeneration.md` | 분석·재생성(`analyze-song`, `bulk-reanalyze-batch` 등) |
| `B3-suno-song-generation.md` | Suno 창작곡 생성 (4.2.4.5의 백엔드 짝) |
| `B4-video-render-and-account-cleanup.md` | 배경영상 소재 조달(`sg-pixabay-videos`·`pixabay-search`) · 렌더링 위임과 콜백(`video-trigger`·`video-callback`) · 계정 정리(`delete-account`) |

### 4.2.8 반 상세페이지

| 절 | 파일 |
|---|---|
| 4.2.8.1 페이지 레이아웃 및 편집 모드 프레임 | `P5a-home-tab-and-edit-mode.md` |
| 4.2.8.2 홈 블록 목록 및 편집기 | `P5b-home-blocks.md` |
| 4.2.8.3 캘린더 및 수업자료 탭 | `P5c-calendar-and-materials.md` |
| 4.2.8.4 알림 및 반 커뮤니티 탭 | `P5d-notifications-and-community.md` |

## 4.3 교사용 기능 분화

| 절 | 파일 | 비고 |
|---|---|---|
| 4.3.1.1 대시보드 프레임 및 교사 전용 위젯 | `T1-teacher-dashboard.md` | |
| 4.3.1.2 공통 위젯 | `T1a-common-widgets.md` | 11종 |
| 4.3.1.3 장식 및 오락 위젯 | `T1b-deco-and-fun-widgets.md` | 장식 7 + 재미 6 |
| 4.3.2 교사 워크스페이스 | `T2-workspace.md` | |
| 4.3.3 강의안 상세: 주차 레이아웃·편집 툴바 | `T3a-lesson-plan-detail.md` | |
| 4.3.4 강의안 상세: AI 생성·생성 이력 | `T3b-lesson-plan-ai-pipeline.md` | |
| 4.3.5 반 생성과 학생 입반 | `T5-course-basics-and-membership.md` | |
| 4.3.6 반 홈 교사 편집 증분 | `T4-course-home-teacher-edits.md` | |
| 4.3.7 캘린더·공지·자료·우리 반 교사 작성층 | `T6-course-workspaces-teacher-edits.md` | |
| 4.3.8 워크스페이스 학생 관리 섹션 | `T7-workspace-student-management.md` | |
| 4.3.9 교사 휴지통 | `T8-teacher-trash.md` | |
| 4.3.10 교사 측 설정 | `T10-teacher-settings.md` | |
| 4.3.11 교사 사용 가이드 | `T9-teacher-guide.md` | |

## 4.4 학생용 기능 분화

| 절 | 파일 | 비고 |
|---|---|---|
| 4.4.1 학생 인증 및 최초 진입 | `S2-student-auth-invite-onboarding.md` | |
| 4.4.2 학생 홈 | `S1-student-home.md` | |
| 4.4.3 반 상세페이지 학생 뷰어층 | `S4-student-course-detail.md` | |
| 4.4.4 학생 설정·휴지통 | `S6-student-settings-and-trash.md` | |
| 4.4.5 학생 사용 가이드 | `S7-student-guide.md` | |

## 4.5 교사·학생 데이터 연동 메커니즘 (기존 파일 참조)

4.5는 별도 프롬프트 파일 없이, 아래 기존 파일의 해당 절이 구현을 담당한다.

| 절 | 참조 파일과 주요 절 |
|---|---|
| 4.5.1 입반: 학생 조작에서 교사 가시화까지 | `S2-student-auth-invite-onboarding.md`(초대 링크 진입·온보딩·반 참여), `T5-course-basics-and-membership.md`(JoinCourseDialog), `S1-student-home.md`(내 수업), `T7-workspace-student-management.md`(반별 보기·명단 등록) |
| 4.5.2 교육 콘텐츠 발행 | `T6-course-workspaces-teacher-edits.md`(캘린더·공지·자료), `T4-course-home-teacher-edits.md`(홈 블록 발행), `P5c-calendar-and-materials.md`, `P5b-home-blocks.md`, `P5d-notifications-and-community.md`, `S4-student-course-detail.md`(학생 수신측) |
| 4.5.3 학습 피드백 회류 | `S1-student-home.md`(선생님께 메시지·공지 읽음), `P5d-notifications-and-community.md`(학생 게시물·답글 스레드·결석 신청), `S4-student-course-detail.md`(학생 게시물 리스트·답글), `T6-course-workspaces-teacher-edits.md`(교사 측 알림) |
