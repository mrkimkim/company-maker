# 현지화 용어집 (Glossary / Termbase) — ko/en 골격 (M1 SubTask 1.4.2)

문서 지위: 본 문서는 `accessibility-localization.md` §9.4 결정("game-design §4 용어 사전을 정본으로 하는 현지화 용어집 구축, 핵심 용어는 로케일별 확정 번역을 고정")의 **스타터 정본**이다. 시스템 중심 게임에서 같은 개념이 화면마다 다르게 번역되면 학습 전이가 깨지므로(README §2.1 "새 용어를 만들지 말고 재사용"), 본 용어집은 번역 벤더·LQA의 단일 참조가 된다.

- **정본:** 한국어(KO) 표기는 `game-design.md` §4 용어 사전을 그대로 따른다(신조어 금지). 영어(EN)는 **제안**이며 (가안) 표시 항목은 LQA에서 확정.
- **키후보:** `string-key-convention.md` §1 규약의 termbase stem. 실제 UI 키는 표면을 앞에 붙여 파생(예: 재무 부실 라벨 = `status.finance.distressed`, 툴팁 = `tooltip.finance.distressed`).
- **일관성 강제:** 부서명·자원명·상태명은 특히 화면 간 완전 일치. termbase로 잠가 문자열 간 불일치를 CI가 자동 검출(`string-key-convention.md` §5 G3).
- **확장 로케일:** ja/zh-Hans 확정 번역은 확장 A/B(§accessibility §12 로드맵)에서 로케일 톤 시트와 함께 고정한다. 본 골격은 ko/en 1차.

---

## 1. 자원 · 재화 (6종 + 결제 재화)

| 키후보 | 한국어(정본) | 영어(제안) | 정의 | 주의 |
|---|---|---|---|---|
| `resource.funds` | 자금(₩) | Funds | 회사별 독립 원장의 운영 화폐. 잔액 하한 0, 음수(부채) 없음 | 기호는 전 로케일 공통 아이콘 1차·`₩`는 폴백(§8.6). 회사 간 이전 금지 |
| `resource.gold` | 골드 | Gold | 경영자 귀속 재화. 획득=청산 반환, 소비=설립비 | 자금과 직접 교환 불가. 전용 아이콘 |
| `resource.reputation` | 명성 | Reputation | 회사별·문턱형 값. 지원자 풀 등급 분포에 단조 증가 | 회사 단위(공용 아님) |
| `resource.xp` | 경험치(XP) | XP | 직원별 성장치. 프로젝트 참여로만 획득 | `α_auto` 페널티 면제(XP는 감쇠 없음) |
| `resource.condition` | 컨디션 | Condition | 직원 소모·회복 상태치. 효율 계수 e의 근거 | 스탯 아님(상태치). 과로=저하 |
| `resource.loyalty` | 충성도 | Loyalty | 직원 이탈 임계 상태치 | 임계 하회 시 "이탈 위험" 배지 |
| `resource.diamond` | 다이아 | Diamond (가안) | 실화폐 결제 재화 | post-MVP·정본 `monetization.md`. 전용 아이콘 |

---

## 2. 회사 · 조직 · 엔티티

| 키후보 | 한국어(정본) | 영어(제안) | 정의 | 주의 |
|---|---|---|---|---|
| `entity.cycle` | 사이클(일) | Cycle (Day) | 게임 시간 1일. 모든 진행 판정의 최소 단위 | "D-일"은 로케일 단위(§8.6) |
| `entity.month_close` | 월말 정산 | Monthly Settlement | 매월 30일차 이후 고정 5단계 시퀀스 | 실제 캘린더 아님(게임 달력) |
| `entity.industry` | 업종 | Industry | 회사 사업 분야. 부서·카탈로그 결정 | MVP 3종. 데이터 팩(§4 규약) |
| `company.grade` | 회사 등급 | Company Rank | 5단계 성장 단계 | 순서 보존 필수(아래 §3) |
| `entity.department` | 부서 | Department | 팀들을 묶는 관리 계층. 의뢰 보드·자율 단위 | 부서명 신설 금지(§7) |
| `entity.dept_capability` | 부서 역량 | Dept Capability | 소속 활성 팀 역량 합산. 선택 가능 티어 결정 | — |
| `entity.team` | 팀 | Team | 프로젝트 수행 단위. 팀장 0..1 + 팀원 0..k | — |
| `status.team.active` | 활성 팀 | Active Team | 팀장이 있는 팀 | — |
| `status.team.adrift` | 표류 팀 | Adrift Team | 팀장 공석 팀(역량 합산 제외, 진척 0) | "표류 · 팀장 공석" 4중 부호 라벨 |
| `entity.bench_pool` | 미배치 풀 | Bench Pool | 팀 미배속 직원의 회사 단위 풀 | 회사별 귀속 |
| `stat.leadership` | 팀장 리더십 | Leadership | 팀장 보직일 때만 발현. 팀원 상한 k 근거 | 보직 스탯(상시 아님) |
| `entity.team_capability` | 팀 역량 | Team Capability | 구성원 직무 스탯 집계 × 팀장 보정 m | 컨디션 미반영 정적값 |
| `entity.efficiency_e` | 효율 계수 e | Efficiency (e) | 컨디션에 단조 증가(상한 1)하는 직원별 계수 | 진척 산출에만 적용 |

---

## 3. 등급 · 라벨 순서 (순서 보존 필수)

| 키후보 | 한국어(정본) | 영어(제안) | 정의 | 주의 |
|---|---|---|---|---|
| `company.grade.startup` … `.group` | 스타트업 / 소기업 / 중기업 / 대기업 / 그룹 | Startup / Small / Mid / Large / Group | 회사 5단계 등급(팀·총원·부서 상한·유지비 결정) | 5단계 **순서 보존**. 로케일별 자연 표현 |
| `grade.employee.c` … `.s` | C / B / A / S | C / B / A / S | 잠재력 대역의 공개 라벨(4단계) | **전 로케일 라틴 문자 유지**(§9.4 결정). 순서 C<B<A<S. 배지 문자 채널 |
| `status.finance.healthy` | 건전 | Healthy | 재무 상태 — 미지급/미납 0, 직전 월말 전액 지급 | 위기 4중 부호 라벨(§2.1) |
| `status.finance.caution` | 주의 | Caution | 매일 갱신 경고 오버레이(정식 상태 아님) | "다음 달 빠듯" 예측. 규칙적 제약 없음 |
| `status.finance.distressed` | 부실 | Distressed | 미지급/미납 발생·지속 상태 | 자진 강등 허용·복귀 체크리스트 표시 |
| `status.finance.bankrupt` | 파산 | Bankrupt | 부실 연속 지속 → 몰수형 소멸(반환 0) | 청산과 구분. 조롱 금지(narrative §8) |
| `perf.flop` … `perf.hit` | 실패작 / 평작 / 성공작 / 히트작 | Flop / Modest / Success / Hit (가안) | 성과 등급 Q 4단계(게임개발 출시형) | **실패작 비하 톤 금지**(narrative §6.3). 히트작=최대 축하 |
| `quality.poor` … `quality.excellent` | 미흡 / 표준 / 우수 | Below / Standard / Excellent | 품질 등급 3종(품질 점수 0~1 구간 라벨) | 성과 등급과 별개 축 |

---

## 4. 프로젝트 · 업무

| 키후보 | 한국어(정본) | 영어(제안) | 정의 | 주의 |
|---|---|---|---|---|
| `stat.role` | 직무 스탯 | Role Stats | 제작 / 기획 / 영업 3종 | 업종 불문 공통 3종 |
| `stat.role.production` | 제작 | Production | "만드는 사람" 직무 스탯 | 라벨 ≤6자 |
| `stat.role.planning` | 기획 | Planning | "설계하는 사람" 직무 스탯 | 라벨 ≤6자 |
| `stat.role.sales` | 영업 | Sales | "연결하는 사람" 직무 스탯 | 라벨 ≤6자 |
| `stat.potential` | 잠재력 | Potential | 성장 한계(불변). 등급의 근거 | "인간 가치" 아님(툴팁 재프레이밍) |
| `entity.tier` | 프로젝트 티어 | Project Tier | 요구 역량 등급 T1~T5 | 부서 역량 ≥ 문턱일 때 선택 가능 |
| `entity.difficulty` | 난이도 | Difficulty | 같은 티어 내 강도 3단계(하/중/상) | 작업량·컨디션 소모·보상 가중 |
| `entity.difficulty.low` … | 하 / 중 / 상 | Low / Mid / High | 난이도 3단계 라벨 | 순서 보존 |
| `entity.board` | 의뢰 보드 | Job Board | 부서별 프로젝트 게시판(게시 기한 후 만료) | — |
| `mechanic.standing_job` | 상비 의뢰 | Standing Job (가안) | 만료 없이 역량 게이트 면제되는 T1·난이도 하 의뢰(모든 보드 상시 1건) | "언제든 믿는 안전한 일감" 톤(narrative §5.5). 위기 생명줄 |
| `entity.startup_cost` | 착수 비용 C₀ | Upfront Cost (C₀) | 프로젝트 착수 시 선불 차감(기본 0) | 실패·포기 시 비환불 |
| `entity.reward_schedule` | 보상 스케줄 | Reward Schedule | 완료 시 확정 지급 계획(일시 또는 일시+꼬리) | — |
| `entity.tail_revenue` | 꼬리 수익 | Tail Revenue | 완료 후 일정 기간 매일 지급되는 분할 보상 | 게임개발 한정 |
| `entity.quality_score` | 품질 점수 | Quality Score | 완료 시 산출 연속 점수(0~1) | 등급은 §3 quality.* |
| `entity.perf_grade` | 성과 등급 Q | Performance Grade (Q) | 품질 점수 + 흥행 롤로 정해지는 4등급 | 게임개발 출시형 한정 |
| `entity.price_mu` | 시세 계수 μ | Market Rate (μ) | 무역 품목군별 매입·판매가 배율(일자의 결정적 함수) | 가격 변동 아님(보상 계수) |
| `entity.line_decay` | 라인 감쇠 δ | Line Decay (δ) | 화장품 자사 라인 반복 생산 보상 감쇠(하한 δ_min>0) | — |

---

## 5. 자율 주행 · 정책 (신뢰·위임의 언어)

| 키후보 | 한국어(정본) | 영어(제안) | 정의 | 주의 |
|---|---|---|---|---|
| `mechanic.direct` | 직접 하달 | Direct (Manual) | 플레이어가 프로젝트·수행 팀을 지정. 보상 효율 100% | 자율 부서에도 개별 착수 가능 |
| `mechanic.auto` | 자율 주행 | Auto (Delegated) | 부서가 정책에 따라 자체 선택·수행 | "믿고 맡긴 유능한 팀"이지 AI 로봇 아님(narrative §2.2·§0.3). "위임"의 신뢰 뉘앙스 유지 |
| `mechanic.alpha_auto` | α_auto | α_auto (Auto Penalty) | 자율 착수 프로젝트의 보상 페널티(0<α<1) | 자금·명성 적용, **XP 면제**. 툴팁 재프레이밍 |
| `mechanic.policy` | 정책 P1/P2/P3 | Policy P1/P2/P3 | 자율 부서 파라미터: 위험 성향 / 우선 목표 / 컨디션 보호선 | — |

---

## 6. 부서명 (규칙 용어 — 신설 금지, narrative §5.3)

| 키후보 | 한국어(정본) | 영어(제안) | 정의 | 주의 |
|---|---|---|---|---|
| `dept.sales` | 영업부 | Sales | 공통 부서(세 업종). 톤 일관 유지(학습 전이) | 업종 추가 시 재사용 |
| `dept.sourcing` | 소싱부 | Sourcing | 무역 | — |
| `dept.logistics` | 물류부 | Logistics | 무역 | — |
| `dept.dev` | 개발부 | Dev | 게임개발 | — |
| `dept.liveops` | 라이브운영부 | LiveOps | 게임개발 | — |
| `dept.rnd` | R&D부 | R&D | 화장품 | 라틴 약어 유지 |
| `dept.production` | 생산부 | Production | 화장품 | 직무 스탯 "제작"과 라벨 혼동 주의(EN 동음) |

- **업종 slug(카탈로그 키):** 무역=`trade` · 게임개발=`gamedev` · 화장품=`cosmetics`(§4 카탈로그 규약).

---

## 7. 멀티 컴퍼니 · 재무 상태 · 위기

| 키후보 | 한국어(정본) | 영어(제안) | 정의 | 주의 |
|---|---|---|---|---|
| `entity.company_slot` | 회사 슬롯 | Company Slot | 회사를 담는 자리(최대 10). 잠김/빈/점유 | 업종 제한 없음(중복 허용) |
| `mechanic.liquidation` | 청산 | Liquidation | 자발적 회사 소멸. 산정식으로 골드 반환 | **경영 판단**(비극 아님). 담담한 톤 |
| `entity.liq_value` | 청산 가치 G_liq | Liquidation Value (G_liq) | 청산 시 반환 골드액 | 파산(강제 소멸)은 반환 0 |
| `mechanic.bankruptcy` | 파산 | Bankruptcy | 강제 몰수형 소멸(반환 0) | 청산과 구분. 게임오버 아님(재설립) |
| `entity.sponsor` | 스폰서 | Sponsor | 추가 회사 설립비를 내는 기존 회사 | — |
| `entity.found_fee` | 설립비 F_found | Founding Fee | 추가 회사 설립비(자금형/골드형 택일) | 소멸형 싱크(이전 아님) |
| `entity.seed` | 시드 C_seed | Seed Capital (C_seed) | 신규 회사가 시스템에게 받는 초기 자본(업종별) | 시스템 지급(회사 간 이전 아님) |
| `mechanic.founding_pick` | 창립 선발 | Founding Pick | 설립 시 1회 창립 풀에서 계약금 없이 최대 N_found명 채용 | — |
| `entity.unpaid_balance` | 미지급 잔액 | Unpaid Balance | 월말 미지급 급여·유지비의 이월 청구액 | **부채 아님**(지연 청구). 이자 없음 |
| `status.finance` | 재무 상태 | Financial State | 회사 단위 상태 머신(건전/부실/파산) + 주의 오버레이 | 라벨은 §3 |
| `mechanic.self_demote` | 자진 강등 | Voluntary Downgrade | 부실 한정, 등급 1단계 낮춰 유지비·상한 축소(플레이어 선택) | 위기 복귀 4수단 중 하나 |
| `entity.unlock_spine` | 해금 스파인 | Unlock Spine | 자율 주행 → 슬롯 2..10 → 완주 진행 해금 순서 | — |
| `entity.clear` | 완주 | Game Clear | 전 업종(3종) 각각 그룹(5등급) 도달 이력 확보 | 소프트 엔딩(플레이 계속). "완주 소요 일수"=스코어 |

---

## 8. 호칭 · 화자 (리아 보이스)

| 키후보 | 한국어(정본) | 영어(제안) | 정의 | 주의 |
|---|---|---|---|---|
| `title.player.rep` | 대표님 | Chief (가안) | 창업기~확장기 플레이어 호칭 | 존댓말의 "정중하되 친근" 결 유지(§9.2) |
| `title.player.chairman` | 회장님 | Chairman (가안) | 그룹 회사 보유 후 승격 호칭 | 등급 연동 승격이 정체성 진화 표현 |
| `character.ria` | 경영 보좌관 리아 | Chief of Staff Ria (가안) | 게임 전체 텍스트의 단일 화자 | **"비서" 아닌 "보좌관"**(narrative §3.1). 전 로케일 단일 화자 유지 |

---

## 9. 로케일별 이름 풀 정책 (음역 금지 · 시드 고정)

`accessibility-localization.md` §9.3 / narrative §5.1을 용어집 부속 정책으로 명시한다. 이름 풀은 §5(문자열 외부화)의 **예외**로, 로케일별 이름 데이터셋으로 별도 관리한다(용어집 termbase 대상 아님).

- **음역 금지:** 직원·회사 이름은 활성 로케일의 **자연스러운 이름 풀**에서 생성한다. 한국어 플레이어는 "김서준", 영어 플레이어는 "Ethan Park"을 본다. 음역("Kim Seojun"을 영어판에)은 어느 쪽에도 자연스럽지 않으므로 금지.
- **로케일별 독립 shipping:** 각 로케일이 성/이름 풀을 독립적으로 제공한다.
- **시드 고정 · 엔티티 항상성:** 직원 엔티티는 결정론적 시드(mechanics-spec §3, art §4.4 조합형 아바타 시드)로 생성되며, 이름도 **"생성 시점 로케일의 풀 + 시드"로 1회 확정해 상태에 굳힌다**. 이후 로케일을 바꿔도 이미 생성된 직원의 이름·얼굴은 유지(같은 직원 = 같은 이름/얼굴).
- **무국적·다양성:** 성별·연령 인상이 고르게 분포하도록 풀 구성, 특정 배경 쏠림 금지(narrative §3.3 금지선). 로케일당 다양성 검수 프로세스(§10.2)를 거친다.
- **성능 비유추:** 등급·직무는 이름에 영향을 주지 않는다(이름으로 성능 유추 금지 — 등급 배지가 정보 채널).
- **폰트 정합:** 이름 풀에 쓰이는 글리프 범위는 CJK 폰트 서브셋에 포함(§8.3) — 두부(□) 방지.
- **회사명 추천 생성기:** 회사명은 플레이어가 짓되 업종별 톤의 자동 추천(랜덤 생성기)을 제공하며, 추천 패턴·풀도 **로케일별 생성 규칙으로 shipping**(한길상사 / Meridian Trading 등). 실존 기업명·상표 회피(narrative §7.3).
- **풀 규모(A1 미결):** 로케일당 성/이름 개수·중복 체감 방지선은 밸런싱·플레이테스트로 확정(narrative 열린질문 5 / accessibility 열린질문 6 단일 트랙). 최소 규모는 최대 10사×수십 명 동시 보유에서 이름 중복이 몰입을 해치지 않는 수준.

**이름 풀 예시(narrative §5.1, 시드 데이터 — 정본 아님, 규모는 미결):**
- `KO 성:` 김·이·박·최·정·한·서·윤·임·오·강·조·신·문·배
- `KO 이름:` 서준·하은·도윤·지아·민재·수빈·예린·준호·채원·유진·태경·소율·현우·나연
- `EN First:` Ethan·Mia·Liam·Ava·Noah·Sofia·Jay·Hana·Leo·Zoe·Ravi·Chloe·Omar·Yuki
- `EN Last:` Park·Cho·Reyes·Kim·Novak·Adeyemi·Tanaka·Silva·Lund·Okafor·Costa·Bauer

---

## 확정 대기(LQA 사인오프 대상)

(가안) 표기는 인게임 LQA에서 고정한다: `resource.diamond`(Diamond) · `mechanic.standing_job`(Standing Job) · `perf.*`(Flop/Modest/Success/Hit) · `title.player.*`(Chief/Chairman) · `character.ria`(Chief of Staff Ria). ja/zh-Hans 확정 번역은 확장 A/B에서 로케일 톤 시트와 함께 등재(§accessibility §12). 등급 라벨 C/B/A/S는 전 로케일 라틴 유지로 확정(§9.4).
