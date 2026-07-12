# AGENTS.md — 프로젝트 핸드오프

다음 세션(사람 또는 에이전트)이 맥락을 빠르게 잡기 위한 요약. **지금까지 한 것**과 **앞으로의 방향**, 그리고 **지켜야 할 제약**을 담는다. 상세는 `docs/`를 본다.

---

## 1. 프로젝트 한눈에
- **무엇:** "회사 경영 시뮬레이션" 게임 — 직원 채용→프로젝트→수익→급여→강화, 여러 회사 동시 운영, 부서별 자율 주행/직접 하달, 청산→골드 회수.
- **플랫폼/모델:** 모바일 우선 · F2P 지향 · 미드코어 (설계 전제 P1~P4, `docs/README.md`).
- **개발 규모: 1인 개발.** ← 매우 중요. 팀/외주/예산/법무/마케팅/벤더/"승인" 같은 조직 오버헤드를 만들지 말 것.
- **현재 단계: 설계·기획·데이터 완료, 코드 미착수.** 다음 = 실제 빌드 시작(엔진 결정부터).

## 2. 지금까지 한 것 (히스토리)
1. **게임 시스템 정본** `docs/game-design.md` — 규칙 확정. 반복 개정: 슬롯 10개·동일 업종 중복 허용·청산 반환(골드)·업종 확장 계약.
2. **제작 문서 세트** — mechanics-spec, balance, ux-design, art-direction, design-tokens, audio-design, narrative-tone, technical-architecture, tech-spikes-and-decisions, content-schema, m2-prototype-spec.
3. **데이터** — 3업종 콘텐츠 팩(`data/content/{trading,gamedev,cosmetics}.json`), 현지화 골격(`docs/localization/`, `data/localization/`).
4. **교차 문서 정합성 검증·수정** 완료(독립 감사 2종 + 통합).
5. **계획·결정** — `docs/roadmap-milestones.md`(MVP 로드맵), `docs/decision-log.md`.
6. **MVP 스코핑** — 비-MVP 문서(수익화·마케팅·라이브옵스·서버·접근성·프로덕션-QA-리스크)를 `docs/post-mvp/`로 이연.
7. **GitHub 이슈 보드** 구축 → MVP·1인개발 기준으로 통폐합(현재 21개 열림).
8. `main`이 기본 트렁크(최신). 파일 작업 전부 반영됨.

## 3. 핵심 제약 (다음 에이전트가 반드시 지킬 것)
- **1인 개발 — 조직 오버헤드 금지.** 팀 롤·외주 발주·예산·법무·마케팅·벤더·"승인/캐스팅" 티켓/작업 만들지 말 것. 과한 세분화 금지.
- **MVP 범위 고수.** MVP = **M1 기반 → M2 프로토타입 → M3 버티컬 슬라이스 → M4.1 3업종** = "플레이·테스트 가능한 빌드". 수익화·풀 분석·풀 현지화·알파·베타·런치·운영은 **post-MVP**(`docs/post-mvp/`). 사용자 요청 없이 되살리지 말 것.
- **정본 위계:** 규칙 = `game-design.md` / 구현 세부 = `mechanics-spec.md` / 수치 = `balance.md`. 규칙을 바꾸려면 `game-design.md` 개정을 먼저.
- **결정론이 아키텍처의 축.** 오프라인 방치 진행이 바이트 단위로 재현돼야 함(`technical-architecture.md`, `mechanics-spec.md §1·§3`). 부동소수 금지, 좌표 키잉 PRNG, 고정소수.
- **주요 확정 결정(일부):** 시간 = 이산 사이클 + 실시간 자동 진행(`1게임일=12분`, 오프라인 상한 48) · 비-P2W · 자원 6종(자금·골드·명성·경험치·컨디션·충성도). 전체는 `docs/decision-log.md`.

## 4. 문서 지도
| 경로 | 내용 |
|---|---|
| `docs/README.md` | 활성 문서 인덱스 + 설계 전제 |
| `docs/game-design.md` | **규칙 정본** |
| `docs/mechanics-spec.md` | 구현 스펙(수식·상태머신·결정론·RNG·엣지케이스) |
| `docs/balance.md` | 밸런스 수치(1차안) |
| `docs/content-schema.md` + `data/content/*.json` | 콘텐츠 팩 포맷 + 3업종 데이터 |
| `docs/technical-architecture.md`, `tech-spikes-and-decisions.md` | 아키텍처·엔진/동결 결정 지원 |
| `docs/ux-design.md`, `art-direction.md`, `design-tokens.md`, `audio-design.md`, `narrative-tone.md` | 표현(UX·아트·색·오디오·톤) |
| `docs/m2-prototype-spec.md` | **M2 빌드 스펙(31티켓·크리티컬 패스·테스트)** |
| `docs/roadmap-milestones.md`, `decision-log.md` | MVP 로드맵 · 결정 추적 |
| `docs/localization/` | 문자열 키 규약·용어집 |
| `docs/post-mvp/` | 이연 문서(수익화·마케팅·라이브옵스·서버·접근성·프로덕션QA리스크) — MVP 아님 |

## 5. GitHub 이슈 보드 (21개 열림, 1인개발용)
- **원칙:** 가까운 일(M1) = 구체 티켓, 나중 일(M2~M4) = Epic 덩어리(상세는 스펙 문서에). 세부는 문서에 다 있음.
- **M1** #1 — 실행 티켓 6: `#8` 엔진=C#, `#9` PRNG, `#10` 고정소수, `#11` 세이브포맷, `#52` L1 코어·CI, `#87` 리포·CI
- **M2** #7 — Epic `#131`(시뮬 코어)·`#132`(무역 슬라이스)·`#133`(최소 UI)·`#134`(검증); 31티켓 상세는 `m2-prototype-spec.md`
- **M3** #12 — Epic `#13`(시스템 통합)·`#14`(아트/오디오)·`#15`(온보딩)·`#16`(밸런스)
- **M4.1** #32 — Epic `#34`(게임개발·화장품 추가)
- 닫힌 이슈(post-MVP·완료·통폐합)는 `is:closed`로 참조 가능. (기본 브랜치 변경·이슈는 브랜치와 무관한 리포 메타데이터)

## 6. 앞으로의 방향 (순서)
1. **`#8` 엔진·코어 언어 = C# 확정** ← 첫 액션. 권장 Unity(C#), 대안 Godot 4(C#). L1 코어는 엔진 비의존 순수 C#. ("언어=C#"만 정하면 아래 2를 병렬로 열 수 있음.)
2. **동결 3종 스파이크** — `#9` 해시 PRNG(권장 SplitMix64) / `#10` 고정소수 `FP=10⁶`+i128 / `#11` 세이브 포맷(권장 FlatBuffers). 교체 시 세이브 리플레이 비호환 → **런치 전 동결**. 근거·수용기준: `tech-spikes-and-decisions.md`.
3. **`#52` 결정론 코어(L1) 골격 + CI**(리플레이·회사 순서 무관성 테스트) · **`#87` 리포·CI** 셋업.
4. **M2 프로토타입** — `m2-prototype-spec.md`대로: 시뮬 코어 → 무역 수직 슬라이스 → 최소 UI(그레이박스) → 검증·재미 판정(Go/No-Go). 무역 팩(`data/content/trading.json`)은 로더 준비 완료.
5. **M3 버티컬 슬라이스** — 자율/멀티/청산/진행성장 전 시스템 + 1차 아트/오디오 + 온보딩 + 밸런스 튜닝(헤드리스).
6. **M4.1 3업종** — 게임개발·화장품 통합(데이터 준비됨).

## 7. 미결 (사람 결정 — 대부분 후속)
- **MVP 크리티컬:** 엔진 결정(#8), 동결 3종(#9~11) — 권장안은 `tech-spikes-and-decisions.md`에 있음, "검증하고 동결"만 하면 됨.
- **post-MVP(지금 불필요):** 확률 공시 법무, BaaS/MMP 벤더, 개인정보/약관, 브랜드명·상표 — `decision-log.md` B절. MVP 단계에선 건드리지 말 것.

## 8. 작업 관례
- **브랜치:** `main`이 기본 트렁크·최신. 커밋 후 push.
- **아직 게임 코드 없음** — 전부 설계·문서·데이터. 다음 단계가 실제 코딩의 시작(엔진 결정 → M1.2 → M2).
- 새 작업 시 이 파일(§2·§6)과 `decision-log.md`를 먼저 갱신·참조.
