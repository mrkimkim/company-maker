# 회사 경영 시뮬레이션 — 문서 (MVP 범위)

이 디렉터리는 **MVP 제작에 필요한 문서**만 담는다. 프로젝트 관리를 단순하게 유지하기 위해, 수익화·마케팅·라이브옵스·서버·풀 접근성/현지화·런치/리스크 문서는 [`post-mvp/`](./post-mvp/)로 이연했다(삭제 아님, git 이력·복원 가능).

## MVP 범위
**M1 기반 → M2 코어 루프 프로토타입 → M3 전 시스템+1차 아트/오디오 → M4.1 3업종 콘텐츠**, 즉 "플레이·테스트 가능한 빌드"까지. 그 이후(수익화·분석·풀 현지화·알파·베타·런치·운영)는 post-MVP.

---

## 활성 문서 (MVP)

| 문서 | 내용 |
|---|---|
| [game-design.md](./game-design.md) | **게임 시스템 정본** — 규칙·엔티티·자원·의사결정 |
| [mechanics-spec.md](./mechanics-spec.md) | 구현 스펙 — 수식 형태·상태 머신·결정론·RNG·엣지케이스 |
| [balance.md](./balance.md) | 밸런스 수치 (1차안) |
| [content-schema.md](./content-schema.md) | 콘텐츠 팩 데이터 포맷 |
| [ux-design.md](./ux-design.md) | UX·화면 구성·플로우 |
| [art-direction.md](./art-direction.md) | 아트 디렉션·에셋 |
| [design-tokens.md](./design-tokens.md) | 색 토큰 v0 (HEX·대비 검증) |
| [audio-design.md](./audio-design.md) | 사운드·음악·오디오 시스템 |
| [narrative-tone.md](./narrative-tone.md) | 내러티브·톤·세계관 |
| [technical-architecture.md](./technical-architecture.md) | 기술 아키텍처·결정론 코어·세이브 |
| [tech-spikes-and-decisions.md](./tech-spikes-and-decisions.md) | 엔진·동결 3종 스파이크 결정 지원 |
| [m2-prototype-spec.md](./m2-prototype-spec.md) | M2 프로토타입 착수 스펙(티켓·테스트) |
| [roadmap-milestones.md](./roadmap-milestones.md) | MVP 로드맵 (M1~M4.1) |
| [decision-log.md](./decision-log.md) | 결정 로그 |
| [localization/](./localization/) | 문자열 키 규약·용어집 |
| `data/content/*.json` | 3업종 콘텐츠 팩 |
| `data/localization/strings.sample.json` | 스타터 ko/en 문자열 |

> `game-design.md`는 "규칙이 무엇인가"만 다룬다(수치·구현·표현 배제). 나머지 활성 문서가 그 구현·표현을 채운다. 규칙 변경은 `game-design.md` 개정으로 선행한다.

**이연(post-MVP):** monetization · marketing · liveops-analytics · server-api-and-pipeline · accessibility-localization · production-qa-risk → [`post-mvp/`](./post-mvp/). 해당 단계(수익화·런치·운영)에 진입할 때 활성 문서로 승격한다.

---

## 설계 전제 (변경 가능)
- **P1 플랫폼:** 모바일 우선(iOS/Android), 세로·짧은 세션.
- **P2 비즈니스:** F2P+IAP(수익화 상세는 post-MVP). 비 페이투윈.
- **P3 톤:** 미드코어, 밝고 경쾌한 스타일라이즈드 2D, 보좌관 '리아' 단일 화자.
- **P4 언어:** MVP는 한국어 중심(풀 다국어는 post-MVP).

## 프로젝트 관리
- **MVP 백로그·진행:** GitHub Issues (마일스톤 M1~M4.1 · Epic · Ticket). post-MVP 이슈는 닫힘.
- **미결 결정:** [decision-log.md](./decision-log.md).
