# 회사 경영 시뮬레이션 — 제작 문서 세트

이 디렉터리는 게임 "회사 경영 시뮬레이션"의 제작 문서 모음이다. 각 문서는 하나의 제작 파트를 상세하게 정의하며, 모든 문서는 시스템 정본인 [`game-design.md`](./game-design.md)를 기반으로 한다.

---

## 0. 문서 맵

| # | 문서 | 담당 파트 | 상태 |
|---|---|---|---|
| — | [game-design.md](./game-design.md) | **게임 시스템 정본** (규칙·엔티티·자원·의사결정) | ✅ |
| 1 | [narrative-tone.md](./narrative-tone.md) | 내러티브 · 톤 · 세계관 · 컨셉 | ✅ |
| 2 | [art-direction.md](./art-direction.md) | 그래픽 테마 · 아트 디렉션 · 에셋 파이프라인 | ✅ |
| 3 | [ux-design.md](./ux-design.md) | UX · 화면 구성 · 정보구조 · 플로우 | ✅ |
| 4 | [audio-design.md](./audio-design.md) | 사운드 · 음악 · SFX · 오디오 시스템 | ✅ |
| 5 | [mechanics-spec.md](./mechanics-spec.md) | 게임 내부 세부 디테일 (구현 스펙: 수식·상태·엣지케이스) | ✅ |
| 6 | [balance.md](./balance.md) | 게임 밸런스 정의 (변수 초기값·커브·튜닝 타깃) | ✅ |
| 7 | [monetization.md](./monetization.md) | 수익화 (재화·상품·가격·상점·라이브 오퍼) | ✅ |
| 8 | [marketing.md](./marketing.md) | 마케팅 (포지셔닝·UA·ASO·커뮤니티·런치) | ✅ |
| 9 | [technical-architecture.md](./technical-architecture.md) | 기술 아키텍처 (데이터모델·세이브·시뮬레이션·플랫폼) | ✅ |
| 10 | [liveops-analytics.md](./liveops-analytics.md) | 라이브옵스 · 분석/지표(KPI) · 텔레메트리 · 콘텐츠 캘린더 | ✅ |
| 11 | [accessibility-localization.md](./accessibility-localization.md) | 접근성 · 현지화(i18n/l10n) | ✅ |
| 12 | [production-qa-risk.md](./production-qa-risk.md) | 개발 로드맵 · QA/테스트 계획 · 리스크 레지스터 | ✅ |

> **범위 경계:** `game-design.md`는 "게임 규칙이 무엇인가"만 다룬다(수치·구현·표현 배제). 본 세트의 나머지 문서가 그 배제 영역 — 표현(아트/사운드/UX), 구현(기술/밸런스 수치), 사업(수익화/마케팅), 운영(라이브옵스/QA/리스크) — 을 채운다. 규칙 자체를 바꾸는 결정은 반드시 `game-design.md` 개정으로 선행한다.

### 제작·계획 산출물 (파트 문서에서 파생)

| 문서/데이터 | 내용 | 파생 |
|---|---|---|
| [roadmap-milestones.md](./roadmap-milestones.md) | 사업 마일스톤 마스터 플랜(M1~M8, Phase·SubTask) | production-qa-risk |
| [decision-log.md](./decision-log.md) | 확정/미결/플레이테스트 결정 통합 추적 | 전체 |
| [tech-spikes-and-decisions.md](./tech-spikes-and-decisions.md) | 엔진·동결 3종 스파이크 결정 지원(D-102~105) | technical-architecture |
| [m2-prototype-spec.md](./m2-prototype-spec.md) | M2 프로토타입 착수 스펙(31티켓·테스트) | mechanics-spec |
| [server-api-and-pipeline.md](./server-api-and-pipeline.md) | 서버 API 계약 + 데이터 파이프라인 | technical-architecture |
| [content-schema.md](./content-schema.md) | 콘텐츠 팩 데이터 포맷 스키마 | mechanics-spec |
| [design-tokens.md](./design-tokens.md) | 색 토큰 v0(HEX·대비 검증) | art-direction |
| [localization/](./localization/) | 문자열 키 규약·용어집 | accessibility-localization |
| `data/content/{trading,gamedev,cosmetics}.json` | 3업종 콘텐츠 팩 인스턴스 | content-schema |
| `data/localization/strings.sample.json` | 스타터 ko/en 문자열 | localization |

---

## 1. 설계 전제 (Design Premises)

아래는 표현·사업 문서 전반을 좌우하는 **최상위 전제**다. 이 전제가 바뀌면 관련 문서를 동반 개정해야 하므로, 각 문서는 자신이 의존하는 전제를 명시한다. 모든 전제는 **변경 가능**이며, 시스템 디자인의 신호로부터 도출한 기본값이다.

### P1. 타깃 플랫폼 — 모바일 우선 (iOS / Android)
- **세로(portrait) 우선** UI, 한 손 조작, 짧은 세션(1~5분) + 자율 주행 방치 플레이.
- 크로스플랫폼(PC/웹) 확장은 아키텍처 상 열어두되(§technical-architecture) **런치 범위는 모바일**.
- 근거: 자율 주행·다수 회사 방치 운영·사이클 기반 시간 모델이 모바일 세션 구조와 정합. 직원 등급 채용(가챠 인접)·프리미엄 재화(다이아)가 F2P 모바일 관례와 일치.

### P2. 비즈니스 모델 — F2P + IAP
- 기본 무료, 인앱결제로 수익. **프리미엄 재화 = 다이아**(`game-design.md`에서 post-MVP로 명명된 것을 본 세트에서 정식화).
- **비(非) 페이 투 윈 원칙**: 결제는 시간 단축·편의·확장·코스메틱에 한정. 결제로만 얻는 전투력/승리 독점 없음. (상세 §monetization)
- 광고는 보조(리워드 광고 옵트인)로 검토, 필수 아님.

### P3. 톤·포지셔닝 — 미드코어, 밝고 경쾌한 스타일라이즈드
- 접근성 있는 진입 + 관리 깊이의 균형. 진지한 하드코어 수치 시뮬도, 극단적 캐주얼도 아님.
- 아트: 밝고 깔끔한 2D 스타일라이즈드(플랫/세미플랫), 친근한 캐릭터. (상세 §art-direction, §narrative-tone)
- 유머러스하되 냉소적이지 않은 경영 드라마 톤. (상세 §narrative-tone)

### P4. 지역·언어
- **1차 출시 언어: 한국어·영어**. 아키텍처·UX는 다국어(일본어·중국어 간체 우선 확장) 대응 구조로 설계. (상세 §accessibility-localization)

### P5. 개발 규모 전제
- 소규모 팀(인디~부티크) 기준의 MVP → 소프트런치 → 글로벌 런치 파이프라인을 가정. (상세 §production-qa-risk)

---

## 2. 공유 규약 (Conventions)

모든 문서가 공유하는 규약. 문서 간 모순을 막기 위한 단일 기준이다.

### 2.1 용어·엔티티
- 엔티티·자원·규칙 용어는 **`game-design.md` §4 용어 사전을 정본**으로 한다. 새 용어를 만들지 말고 재사용한다.
- 자원 6종: **자금(₩, 회사별) · 골드(경영자) · 명성(회사별) · 경험치(직원) · 컨디션(직원) · 충성도(직원)**. 표현·상점·지표 문서는 이 6종 외의 게임플레이 자원을 신설하지 않는다(수익화의 다이아는 자원이 아닌 **실화폐 결제 재화**로, §monetization이 단독 정의).

### 2.2 변수 표기
- 밸런스 수치의 **정본은 `balance.md`**다. 다른 문서가 수치를 언급할 때는 값이 아니라 변수 기호(`α_auto`, `C_seed` 등)로 참조하고, 구체 값이 필요하면 `balance.md`를 인용한다.
- 구현 세부(수식·상태 전이·엣지케이스)의 **정본은 `mechanics-spec.md`**다.

### 2.3 소유권 (문서 간 중복 방지)
- **화면·플로우**: `ux-design.md`. 다른 문서는 화면을 참조만 한다.
- **비주얼 스타일·에셋**: `art-direction.md`.
- **오디오**: `audio-design.md`.
- **상품·가격·상점·오퍼**: `monetization.md`.
- **KPI·텔레메트리·이벤트 캘린더**: `liveops-analytics.md`.
- **데이터 모델·세이브·시뮬레이션 루프·플랫폼 SDK**: `technical-architecture.md`.
- **일정·QA·리스크**: `production-qa-risk.md`.

### 2.4 재화 표기 규약
- 게임 내 **자금**은 전용 아이콘을 1차로 노출하고 `₩` 텍스트 기호는 폴백으로만 쓴다. **골드·다이아**는 각각 전용 아이콘. **스토어 실화폐 가격**은 스토어 제공 현지 통화로 표기한다. 정본은 `accessibility-localization.md §8.6`(UX·아트·수익화가 준수). 게임 재화 `₩`와 실화폐 `₩`(원화 가격)를 절대 혼동 표기하지 않는다.

### 2.5 개발 착수 기준 (Definition of Ready)
각 문서는 "이 문서만 보고 해당 파트 개발/제작에 착수할 수 있는가"를 자기 검증한다. 부족분은 각 문서 말미 "열린 질문"에 남긴다.

### 2.6 문서 상태·개정 관리
- **정본 개정 우선 원칙:** 게임 규칙을 바꾸는 결정은 반드시 `game-design.md` 개정으로 선행한다. 하위 문서가 규칙 변경을 필요로 하면 `[game-design 개정 필요]` 태그로 표기하고, 승인 전까지 해당 규칙을 임의로 신설하지 않는다.
- **미결 개정 안건·교차 이슈의 단일 집결지는 `production-qa-risk.md §A5`(규칙 개정 안건)와 §C9(리스크 레지스터)**다. 여러 문서에 걸친 미결 결정(예: 사이클↔실시간 매핑, 확률 공시 법무, 리아 캐스팅, 색 토큰, 런치 전 동결 3종)은 이곳에서 추적한다.
- 문서 상태는 §0 문서 맵의 ✅로 표기한다. 초안 완성 ≠ 결정 확정 — 각 문서의 "열린 질문"과 `production-qa-risk.md`의 준비도 체크리스트(§C10)가 실제 착수 가능 여부를 정의한다.

---

## 3. 읽는 순서 (권장)

1. `game-design.md` (규칙 이해) → 2. `narrative-tone.md` (톤 프레임) → 3. `mechanics-spec.md` + `balance.md` (구현·수치) → 4. `ux-design.md` + `art-direction.md` + `audio-design.md` (표현) → 5. `monetization.md` + `marketing.md` (사업) → 6. `technical-architecture.md` + `liveops-analytics.md` (구축·운영) → 7. `accessibility-localization.md` + `production-qa-risk.md` (품질·일정).

> **개발 착수 전 필독:** 문서 세트 전반의 미결 핵심 결정과 착수 준비도는 `production-qa-risk.md`(§A5·§C9·§C10)에 종합돼 있다. 실제 프로덕션 킥오프는 이 문서의 "지금 결정해야 할 항목"부터 확인할 것.
