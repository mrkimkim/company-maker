# 회사 경영 시뮬레이션 — 게임 내부 세부 디테일 (구현 스펙) v1.0

> **문서 지위.** 본 문서는 `game-design.md`(시스템 정본)가 "규칙이 무엇인가"로 확정한 것을 **결정론적·구현 가능한 세부**로 정밀화한다. `README.md` 규약 2.2에 따라 본 문서는 **수식의 형태·계산 순서·상태 전이·엣지케이스·RNG 규격**의 정본이며, **구체 밸런스 값은 `balance.md`가, 데이터 저장·플랫폼은 `technical-architecture.md`가** 소유한다.
>
> **불가침 원칙.** 본 문서는 game-design의 규칙을 **바꾸지 않는다**. 이미 확정된 결정은 §번호로 인용해 재확정하고, 그 규칙을 실행 가능하게 만들기 위한 새 확정은 **설계 결정:** 으로 표기한다(game-design 표기 규약 §1.4 준수). 밸런스 수치는 값 대신 기호(`α_auto`, `C_seed`, …)로 남긴다.
>
> **인용 표기.** `[GD §x.y]` = game-design.md 해당 절. 본 문서 내 상호참조는 `→ §n`.

---

## 목차

0. 문서 지위와 범위
1. 결정론 기반 (Determinism Foundations)
2. 계산 정밀도·단위
3. RNG 규격
4. 일 처리 5단계 의사코드
5. 월말 정산 5단계 의사코드
6. 자율 주행 선택 알고리즘 (완전 결정론 명세)
7. 핵심 수식 함수 라이브러리
8. 상태 머신 상세
9. 엣지케이스 처리 통합
10. 데이터 정의 (필드 스키마)
11. balance.md · technical-architecture.md 인계 인터페이스
12. 열린 질문

---

## 1. 결정론 기반 (Determinism Foundations)

`game-design`의 결정론 요구(자율 주행 §9.1-3 "동일 상태+동일 정책=동일 선택")를 **시뮬레이션 코어 전체**로 확장한다. 결정론은 세이브/로드 재현·리플레이·서버 검증·버그 재현의 토대다.

### 1.1 결정론 불변식 (Core Invariant)

> **[불변식 D] 동일한 `(게임 상태 스냅샷, 정렬된 입력 로그)` → 동일한 후속 상태.**

이를 위해 코어가 만족해야 할 4개 하위 규칙:

1. **정준 순회 순서(§1.2)** — 엔티티 반복은 항상 고정된 정렬 키를 따른다. 해시맵 반복 순서·삽입 순서 의존 금지.
2. **결정론 ID(§1.3)** — 모든 엔티티는 결정론적으로 할당된 단조 정수 ID를 가지며 타이브레이커의 최종 기준이 된다.
3. **단일 명령 큐(§1.4)** — 플레이어 입력은 제출 순서로 직렬화되어 재검증·적용된다.
4. **무부동소수점 결정론(§1.5, §2)** — 게임플레이에 영향을 주는 모든 산술은 고정소수점/정수로 수행한다.

### 1.2 정준 순회 순서 (Canonical Iteration Order)

어떤 단계든 엔티티 집합을 순회할 때 아래 키의 **오름차순**을 사용한다. 이벤트 발행 순서·리포트 순서도 동일하다.

| 집합 | 정렬 키 |
|---|---|
| 회사 | 슬롯 인덱스 `slot_idx` (0..9) |
| 부서(회사 내) | `dept_id` (개설 순 = ID 순) |
| 팀(부서 내) | `team_id` |
| 직원(회사 내) | `employee_id` |
| 게시물(보드 내) | `posting_id` (상비 의뢰는 부서당 고정 예약 ID, 항상 최소값) |
| 프로젝트 인스턴스 | `project_id` |
| 지원자(풀 내) | `applicant_index` (0..N−1, 생성 순) |

- **설계 결정 (DD-19): 회사 간 처리는 결과에 무관하다(회사별 독립 원장·이전 금지 [GD §5.9]). 그럼에도 정준 순서를 slot_idx 오름차순으로 고정**하여 이벤트/리포트/로그의 바이트 단위 재현성을 보장한다. 회사 간 자원 경합이 없으므로 이 순서는 밸런스에 영향을 주지 않는다.

### 1.3 결정론 ID 할당

- 전역 단조 카운터 `next_entity_id : u64`(세이브 상태의 일부). 엔티티 생성 시 현재값을 부여하고 1 증가.
- 생성은 항상 결정론적 단계 순서(§4, §5) 안에서 일어나므로 ID 열은 재현적이다.
- ID는 **불변·재사용 금지**. 소멸해도 회수하지 않는다(RNG 키 안정성 §3의 전제).
- **함의:** RNG 롤이 entity_id로 키잉되므로(§3), ID의 결정론성이 곧 롤의 결정론성이다. ID 할당은 순회 순서(§1.2)와 명령 순서(§1.4)에만 의존해야 하며, 벽시계·해시주소·비결정 컨테이너에 의존해서는 안 된다.

### 1.4 단일 명령 큐 모델 (Command Queue)

**설계 결정 (DD-1): 사이클 사이 뷰 구간에서 플레이어가 낸 모든 명령은 제출 순서(FIFO)로 하나의 큐에 적재되어, 다음 사이클 일 처리 1단계에서 순서대로 재검증·적용된다.**

- game-design이 "예약(→ 다음 사이클 1단계 확정)"이라 부른 것(승급 승인 [GD §7.1], 취소 [GD §8.9], 설립/청산 등)은 **"클릭 즉시가 아니라 다음 1단계에 효과가 적용된다"**는 뜻이다. 별도의 예약 우선순위 클래스를 두지 않는다 — **제출 순서가 유일한 우선순위**다. 이로써 [GD §2.2](처리 종류 나열)·[GD §7.1](승급 예약)·[GD §7.3]("명령 순서대로")·[GD §8.9](취소 예약)가 단일 모델로 정합된다.
- **재검증(reject-on-invalid):** 각 명령은 적용 시점의 현재 상태로 조건을 다시 판정한다. 위반 시 그 명령만 기각(로그 발행), 나머지는 계속. 선불 원칙 [GD §5.4.1]과 정합 — 잔액 경합은 "앞선 명령이 먼저 차감"으로 결정론적으로 해소된다.
- **§2.2 처리 종류 표는 시간 순서가 아니라 "1단계에 실리는 처리의 목록"이다.** 실제 시간 순서는 §4.2가 확정한다: [플레이어 명령 스트림(제출 순)] → [시스템 재조정(자동 승계 등)] → [불변식 확정].
- **월말 step 4 적용 명령의 예외:** 자진 강등 신청 [GD §5.5-4]은 일 처리 1단계가 아니라 **월말 정산 4단계**에 적용된다. 이는 회사에 걸리는 `pending_self_demotion` 플래그로 표현되고 해당 단계에서 소비된다(§5). 승급 승인은 통상 명령이므로 다음 1단계에 처리된다.

**명령 종류(닫힌 목록, 1단계 적용 대상):** 채용 확정 / 배치·이동 / 팀장 임명·교체·해임 / 팀 신설·해체 / 훈련 / 직접 하달 착수 / 취소 / 자율 모드·정책 전환·일괄 토글 / 승급 승인 / 부서 개설 / 회사 설립 / 회사 청산 / 해고. (월말 step4 적용: 자진 강등 신청.)

### 1.5 무부동소수점 결정론

- **설계 결정 (DD-3): 게임플레이 결과에 영향을 주는 모든 산술(진척, 효율 계수, 품질, 시세, 보상, 확률)은 고정소수점 정수 또는 정수로 계산한다.** IEEE-754 부동소수점의 플랫폼 간 반올림·초월함수 편차가 [불변식 D]를 깨기 때문이다.
- 표현·규약은 §2.3. 표시(UI) 전용 파생값(총자산 등 [GD §5.2])은 부동소수점 허용(규칙에 재입력 금지).

---

## 2. 계산 정밀도·단위

### 2.1 자원 타입·범위·반올림

| 자원 | 타입 | 범위 | 반올림/증감 규칙 |
|---|---|---|---|
| 자금(₩) | `i64` (정수 ₩) | `[0, +∞)` — 하한 0, 음수 없음 [GD §5.2] | 원장 경계에서만 정수화(§2.2). 부분 ₩ 없음 |
| 골드 | `i64` | `[0, +∞)` | 정수. 청산 반환액 `G_liq` 산정 후 정수화(round-half-up) |
| 명성 | `i64` | `[0, +∞)` — 단조 비감소 [GD §2.5] | 획득분 정수화 후 가산. 소비·감소 경로 없음 |
| XP | `i64` (직원별 누적) | `[0, +∞)` | 정수. 레벨업 시 소비(자동 변환) |
| 컨디션 | `i32` 고정소수 | `[0, COND_MAX]` (권장 `COND_MAX = 1000` = 100.0%) | 일일 정수 증감, clamp |
| 충성도 | `i32` 고정소수 | `[0, LOY_MAX]` (권장 `LOY_MAX = 1000`) | 정수 증감, clamp |
| 레벨 | `i32` | `[1, level_cap(잠재력)]` | 정수 |

- **round-half-up = round half away from zero.** 모든 화폐성 최종 트랜잭션의 유일한 반올림 규칙. (게임플레이 값은 모두 ≥0이므로 실질적으로 "0.5 올림".)
- **단일 반올림 경계:** 보상액처럼 여러 계수를 곱하는 식(§7.4의 `M×변동률×m_Q×계수소스×α_auto`)은 **고정소수점으로 전 과정을 곱한 뒤 원장 기입 직전 1회만** 정수화한다. 중간 반올림 누적 금지.
- **꼬리 분할(§7)·XP 지분 분배**는 정수 합이 총액과 정확히 일치해야 하므로 **최대 잉여법(largest-remainder)**으로 정수화한다(§7.4, §4.5). 이 규칙이 "합 = 총액" 불변식을 보장한다.

### 2.2 시간 단위·인덱싱

- 최소 단위 = **일(Day)** [GD §2.1]. 전역 카운터 `day : i64`, 1-기반.
- `month_index(day) = ceil(day / 30)`; `day_of_month(day) = ((day − 1) mod 30) + 1`.
- **월말 = `day_of_month == 30`.** 해당 일의 일 처리 5단계 완료 **직후** 월말 정산 5단계 실행 [GD §2.3].
- **월(Month) = 30일 고정** [GD §2.1]. 주 단위 없음.
- 사이클(=일)은 전 회사 동시 적용, 일시정지·배속 없음 [GD §2.1, §10.1].

### 2.3 고정소수점·확률 표현

- **고정소수 스칼라(효율 `e`, 시세 `μ`, 품질 `q_raw`, 계수 `m`, `δ` 등):** 스케일 `FP = 1_000_000` (Q값 = 실수 × 10⁶, `i64` 보관). 곱셈은 `(a*b)/FP`, 나눗셈은 `(a*FP)/b`. 모든 팀이 동일 스케일·동일 연산 순서를 쓴다.
- **확률:** `p_scaled ∈ [0, PROB_ONE]`, `PROB_ONE = 1_000_000`. 베르누이 판정 = `uniform32(roll) * PROB_ONE >> 32 < p_scaled` 형태의 정수 비교(§3.3).
- **clamp/보간:** 모든 `f(x)` 계단·구간 함수는 정수 경계로 정의(부동소수 경계 비교 금지).

---

## 3. RNG 규격

### 3.1 카운터 기반 무상태 PRNG (아키텍처)

**설계 결정 (DD-2): 순차 상태형(stateful) RNG가 아니라 좌표 키잉 카운터 기반(stateless, counter-based) 해시 PRNG를 사용한다.**

근거: 최대 10사 × 다부서 × 예외 기반 처리(§9.6)에서 롤 횟수·순서가 플레이어 행동·버전에 따라 달라진다. 순차 스트림 하나면 "이번 달 채용 수가 흥행 롤 시퀀스를 밀어버리는" 교차 오염이 생겨 재현성이 깨진다. 좌표 키잉은 **각 롤을 다른 롤과 독립**으로 만들어 이 문제를 원천 차단한다.

```
roll(stream_id, c0, c1, c2, c3):
    # h = 결정론적 64비트 정수 해시 (예: SplitMix64 / PCG / xxHash — 구현 고정)
    h = hash64(master_seed, stream_id, c0, c1, c2, c3)
    return h            # 이후 §3.3 매핑으로 분포화
uniform01_fp(...) = (roll(...) mod FP)                     # [0, FP) 고정소수 균등
uniform_range(a, b, ...) = a + (roll(...) mod (b − a + 1)) # [a, b] 정수 균등
```

- `master_seed : u64` — 새 게임 시작 시 1회 생성, **세이브 전 수명 불변**.
- 해시 알고리즘·바이트 순서·믹싱 상수는 **technical-architecture가 단일 고정**한다(교체 시 리플레이 비호환).

### 3.2 스트림 목록 & 키 좌표 (닫힌 목록)

| # | `stream_id` | 용도 | 키 좌표 `(c0,c1,c2,c3)` | 발생 단계 |
|---|---|---|---|---|
| 1 | `APPLICANT` | 지원자 풀 생성(등급·초기 스탯·리더십·잠재력) | (company_id, month_or_founding_seq, applicant_index, field_id) | 월말 5 / 설립(1단계) |
| 2 | `HIT` | 게임개발 흥행 롤(성과 등급) | (project_id, 0,0,0) | 완료(4단계) |
| 3 | `ATTRITION` | 자발적 퇴사 베르누이 | (employee_id, day, 0,0) | 상태 갱신(5단계) |
| 4 | `PROGRESS_EPS` | 일일 진척 소변동 `ε` | (project_id, day, 0,0) | 진행(3단계) |
| 5 | `POSTING_VAR` | 게시 시 보수 ±변동률 롤 | (posting_id, 0,0,0) | 보드 보충(5단계) |
| 6 | `BOARD_REFILL` | 보충 게시물 티어·템플릿 선택 | (dept_id, day, slot_index, 0) | 보드 보충(5단계) |

- **결정적이라 스트림 없음:** 시세 `μ(g, day)`(일자 결정 함수 [GD §12.2, §7])·라인 감쇠 `δ(n)`(누적 카운터의 결정 함수). 이들은 RNG가 아니라 순수 함수(§7.4)다 — 태스크 요구대로 μ는 RNG 목록에서 제외.
- `field_id`(스트림1): {grade=0, stat_제작=1, stat_기획=2, stat_영업=3, leadership=4, potential=5, 창립_leader_guarantee=6}.

### 3.3 각 롤의 분포

| 롤 | 분포/매핑 |
|---|---|
| **흥행 롤(성과 등급)** | `u = uniform01_fp(HIT, project_id)`; `y = clamp(q_raw + σ_hit·(u − FP/2)/FP, 0, FP)`; `Q = bucket(y; d_c1<d_c2<d_c3)` → {실패작/평작/성공작/히트작}. `q_raw`가 평균을, `σ_hit`가 분산을 준다(§7.4). **게임 내 유일한 대형 결과 RNG** [GD §12.3]. |
| **자발적 퇴사** | `충성도 ≥ Loy_min`이면 확률 0(판정 skip). 아니면 `p = P_quit(loy)`(§7.4), 베르누이(`uniform01_fp(ATTRITION, id, day) < p`). |
| **일일 진척 `ε`** | `ε = ε_max·(2·uniform01_fp − FP)/FP`, 즉 `ε ∈ [−ε_max, +ε_max]`. **설계 결정 (DD-5): `ε_max < FP`(=1.0)로 강제해 `(1+ε) > 0`** — 진척이 음수·0이 되지 않는다(질감용, 성패 불역전 [GD §8.6]). |
| **지원자 풀** | 등급: reputation 파라미터 categorical(§7.4 `GradeDist`) inverse-CDF. 초기 스탯/리더십: 등급 대역 내 균등 또는 절단정규(고정소수 근사). 잠재력: 등급 대역 균등. 창립 풀: `field_id=6` 좌표로 리더십 ≥ `L_min` 보유자 1인 **강제 주입** 후 나머지 생성 [GD §6.4]. |
| **게시 변동률** | `v_post = FP + v_amp·(2·uniform01_fp − FP)/FP ∈ [1−v_amp, 1+v_amp]`. (중앙 편향이 필요하면 삼각분포 옵션 — balance 선택.) |
| **보드 보충** | 도달 티어 `T*` 이하 위주 + 잠김 미리보기 1티어(`T*+1`, 열람 전용)의 **가중 categorical**로 티어 결정 → 해당 (부서 종류, 티어) 템플릿 풀에서 균등 선택. 가중치는 balance. |

### 3.4 세이브/로드 & 재현성

- **세이브에 보관하는 RNG 상태 = `master_seed`(불변) + `day`(이미 게임 상태).** 그 외 RNG 상태 블롭 없음 — 모든 롤은 `(seed, stream, 좌표)`의 순수 함수이므로.
- **재현성 정리:** 좌표(entity_id, day, index)가 §1.3에 의해 결정론적이므로, 같은 세이브를 로드해 같은 입력 로그를 재생하면 모든 롤이 동일 → [불변식 D] 성립.
- **로드 중복 롤 방지:** 카운터 기반이라 "이미 소비된 롤"이 없다. 단, **이미 결과가 확정돼 상태에 반영된 롤(예: 확정된 성과 등급 `Q`, 생성된 지원자 카드)은 상태에 저장**하고 재롤하지 않는다(롤은 값 산출 시 1회, 이후 상태가 진실의 원천). 저장하지 않는 순간형 롤(ε, 퇴사 베르누이)만 필요 시 재계산한다.

---

## 4. 일 처리 5단계 의사코드

전역 규약: 각 단계는 **[읽음]/[씀]/[발행]**을 명시. 순회는 §1.2. 모든 화폐 차감은 경제 트랜잭션 [GD §2.4-3].

### 4.1 전역 루프

```
advance_one_cycle():
    day += 1
    step1_apply_commands()        # 지시 반영
    step2_autonomy_dispatch()     # 자율 주행 판단
    step3_project_progress()      # 프로젝트 진행
    step4_completion_payout()     # 완료 판정 및 수익 지급
    step5_state_update()          # 상태 갱신
    if day_of_month(day) == 30:
        for company in companies (slot asc):   # 월말 정산(§5), 회사별 독립
            month_end_settle(company)
```

### 4.2 단계 1 — 지시 반영 [GD §2.2-1]

```
step1_apply_commands():
    # (A) 플레이어 명령 스트림 — 제출 순서(§1.4), 각 명령 재검증·reject-on-invalid
    for cmd in command_queue (submission order):
        if validate(cmd, current_state):
            apply(cmd)            # 아래 apply 규칙
        else:
            emit EVT_CMD_REJECTED(cmd, reason)
    command_queue.clear()

    # (B) 시스템 재조정 — 플레이어 스트림 이후, 최종 조직 상태를 읽어 실행
    for dept in autonomous_depts (canonical):
        for team in dept.teams (team_id asc):
            if team.is_drifting and team.drift_seen_prev_cycle:
                auto_succeed_leader(team)   # §9.4 유일 조직 변경 예외, §7.3

    # (C) 불변식 확정 — 위반 시 코어 버그(로그·assert). k축소 자동이동은 apply 내부에서 이미 수행됨
    assert_org_invariants()      # [GD §7.5]
```

**apply 규칙(명령별 핵심):**

| 명령 | [읽음] | [씀] | [발행] | 재검증 조건 |
|---|---|---|---|---|
| 채용 확정 | 잔액, 총원, `E_max(g)` | −계약금(O-1), 직원 생성(미배치) | EVT_HIRED, M1 | `잔액≥계약금 ∧ 총원<E_max` [GD §6.4] |
| 훈련 | 잔액, 대상 스탯, 상한 | −훈련비(O-2), 스탯 상승 | EVT_TRAINED, M6 | `잔액≥훈련비 ∧ 스탯<상한` [GD §6.5] |
| 배치·이동 | 팀 구성, `k=f(리더십)` | 소속 갱신, k축소 자동이동(§9) | EVT_ROSTER | 불변식 2·6·7 [GD §7.5] |
| 팀장 임명/교체/해임 | 리더십, `L_min`, 진행 프로젝트 | 보직·충성도(승격↑/강등↓), `P_swap` 타이머 | EVT_LEADER, M2 | `리더십≥L_min` [GD §6.7] |
| 팀 신설/해체 | 팀 수, `T_max(g)`, 진행 프로젝트 | 팀 생성/소멸(구성원→미배치) | EVT_TEAM | `팀수<T_max`; 해체는 진행 프로젝트 없음 [GD §7.3] |
| 직접 하달 착수 | 게시물, 팀(활성), `C₀` | −`C₀`(시세면 `×μ(g,day)`), 프로젝트 진행 전이(직접 태그) | EVT_LAUNCH_DIRECT, M3 | 선택가능식(§8.5)·`잔액≥C₀`·팀 유휴·활성 |
| 취소 | 진행 프로젝트 | 인스턴스 소멸(중도 포기, `C₀` 비환불) | EVT_CANCEL | 대상 진행 중 [GD §8.9] |
| 자율 모드·정책 전환 | 해금 플래그 | 부서 모드/정책 | EVT_MODE | 해금 플래그 true(자율 전환 시) [GD §9.2] |
| 승급 승인 | 명성, 잔액, 가동률 | −`C_up`, 상한 갱신(원자) | EVT_PROMOTED | 3조건 전부 재통과, 실패 시 기각·가능표시 보존(§9) [GD §7.1] |
| 부서 개설 | 개설 수, `D_max(g)`, 잔액 | −`D_open`(첫 부서 면제), 부서+빈 팀1 생성 | EVT_DEPT | `개설수<D_max ∧ 종류 미보유 ∧ 잔액≥D_open` [GD §7.2] |
| 회사 설립 | 빈 슬롯, 스폰서 잔액/골드 | 설립비 소멸, 신규 회사(§10.4 초기상태) | EVT_FOUND, M10/M13 | §8 설립 상태머신 조건 |
| 회사 청산 | 청산 스냅샷 | 정리 절차(§9), `G_liq` 골드 지급 | EVT_LIQUIDATE | 항상 허용(마지막 1사 포함) [GD §10.5] |
| 해고 | 직원 | 즉시 소멸(개인 미지급 잔액 소멸) | EVT_FIRED | 항상 허용(무비용) [GD §6.9] |

- **결정론:** 스트림 순서(§1.4) + 각 apply의 순회(§1.2) + entity ID 할당(§1.3)만으로 완전 결정. RNG는 설립 시 창립 풀 생성(스트림1)에만 개입하며 좌표 키잉이라 순서 무관.

### 4.3 단계 2 — 자율 주행 판단 [GD §2.2-2]

```
step2_autonomy_dispatch():
    for dept in departments (canonical) where dept.mode == AUTO and unlock_flag:
        autonomy_select_and_launch(dept)   # §6 완전 명세, 자금 무지출
```

- [읽음] 유휴 팀(팀장 보유 ∧ 담당 프로젝트 없음 — 1단계 직접 하달로 점유된 팀은 제외 [GD §9.2]), 팀 역량·평균 컨디션, 게이트 통과 게시물. [씀] 자율 태그 착수(프로젝트 진행 전이). [발행] EVT_LAUNCH_AUTO. **자금 무지출** [GD §9.5] — `C₀>0` 후보 제외.

### 4.4 단계 3 — 프로젝트 진행 [GD §2.2-3]

```
step3_project_progress():
    for proj in in_progress_projects (project_id asc):
        proj.elapsed_days += 1
        team = proj.team
        if team.is_drifting:                          # 팀장 공석
            dP = 0                                     # [GD §7.3] 진척 0, 기한은 계속 소모
        else:
            e_team = mean(e(cond(m)) for m in team.members ∪ {team.leader})
            eps = eps_roll(PROGRESS_EPS, proj.id, day) # §3.3
            dP  = base(Cap_team, t, d) * e_team/FP * (FP + eps)/FP   # §7.4
            # 컨디션 소모 산출(적용은 5단계)
            overload = overload_coeff(Cap_team, t)     # §7.4, ≥1
            for m in performing_members(team):
                proj.pending_drain[m] += c_base(d) * overload / FP
                proj.cond_sum += cond(m); proj.cond_samples += 1   # 평균 컨디션 러닝 집계(q_raw용)
        proj.progress += dP
        for m in committed_members(team):              # 표류일 포함(§9 DD-9)
            proj.participation_days[m] += 1
            if m == team.leader: proj.leader_days[m] += 1
```

- [읽음] 팀 역량·컨디션·`e`·난이도·티어. [씀] 진척·경과일·컨디션 소모 예약·참여일수·컨디션 러닝합. [발행] 없음(질감 이벤트는 UI 폴링).
- `α_auto`는 **진척에 미적용**(보상 전용) [GD §8.6, §9.2].

### 4.5 단계 4 — 완료 판정 및 수익 지급 [GD §2.2-4]

```
step4_completion_payout():
    # (A) 신규 완료/실패 판정 — 완료 우선(DD-6)
    for proj in in_progress_projects (project_id asc):
        if proj.progress >= W(t,d):
            complete(proj)                     # 아래
        elif proj.elapsed_days > L(t,d):       # L = D_std × λ_due
            fail_deadline(proj)                # 자금·명성 0, 축소 XP, C₀ 비환불 [GD §8.8]
    # (B) 활성 꼬리 스케줄 일일 지급 [GD §5.3 I-1]
    for tail in active_tails (release_id asc):
        pay = tail.daily_amount[tail.age]      # 경과일 단조감소
        ledger_credit(tail.company, pay * (α_auto if tail.auto_tag else FP)/FP)
        tail.age += 1
        if tail.age >= tail.D_tail: close(tail)

complete(proj):
    q_raw, q_grade = quality(proj)             # §7.4, 세 인자
    if proj.is_gamedev_release:
        Q = hit_roll_grade(proj, q_raw)        # §3.3, HIT 스트림, 상태 저장
    coef = coef_source_eval(proj, day)         # 고정=FP / 시세=μ(g,day) / 성과=q(Q) / 감쇠=δ(n)
    cash = M(t,d) * v_post/FP * m_Q(q_grade)/FP * coef/FP * (α_auto if auto else FP)/FP
    # 스케줄 확정
    if has_tail: split cash into 일시분 + 꼬리(§7.4 largest-remainder), 경제에 등록(태그·α_auto 전파)
    ledger_credit(company, 일시분_round)        # round-half-up 1회
    rep = Rep(t) * r_Q(q_grade)/FP * (α_auto if auto else FP)/FP
    reputation_ledger_add(company, round(rep)) # [GD §7.6] 단조 비감소
    distribute_xp(proj, pool = X(t,d), factor = FP)   # 참여일수·팀장가중, α_auto 미적용
    if proj.completion_effect == UNLOCK: mark_unlocked(target)      # 비가역
    if proj.completion_effect == TAIL_REFRESH: tail_refresh(target) # 잔여기간 리셋(§9)
    if proj.repeatable: line_counter[proj.template] += 1            # δ(n)용 / 갱신횟수
    team.release_project(); set proj.state = 완료
    emit EVT_COMPLETE(proj, tier, mode, Q?)

fail_deadline(proj):
    xp = X(t,d) * p(progress/W(t,d))/FP        # p<1, 진척도 단조증가 [GD §8.8]
    distribute_xp(proj, pool = xp, factor = FP)
    team.release_project(); set proj.state = 기한실패
    emit EVT_FAIL(proj, mode)
```

- **설계 결정 (DD-6): 같은 4단계에서 `progress≥W`와 `elapsed>L`이 동시 성립하면 완료가 우선**한다(진척이 목표에 도달했으므로). 완료를 먼저 검사한다.
- **설계 결정 (DD-7): 같은 사이클 착수-완료 허용.** 1/2단계 착수 → 3단계 첫 진척 → 4단계에서 `ΔP≥W`면 그날 완료. game-design이 금지하지 않으며 최소 기간 하한을 두지 않는다.
- **XP 지분 분배(distribute_xp):** 각 참여자 몫 = `pool × (참여일수 + 팀장가중일수) / Σ(전원 가중일수)`, **최대 잉여법**으로 정수화(Σ=pool 보장). 팀장 가중 = leader_days × (`w_leaderXP − 1`) 추가분. 중도 합류·이탈자는 자기 참여일수만큼 [GD §8.7]. `α_auto` 미적용 [GD §9.2].
- [발행] EVT_COMPLETE/EVT_FAIL → 직원·자율 주행·진행/성장·경제(매출 집계용).

### 4.6 단계 5 — 상태 갱신 [GD §2.2-5]

```
step5_state_update():
    for company (slot asc):
      for emp in company.employees (employee_id asc):
        # (1) XP → 레벨업(루프)
        while emp.xp >= XP_need(emp.level+1) and emp.level < level_cap(emp.potential):
            emp.xp -= XP_need(emp.level+1); emp.level += 1
            allocate_level_stats(emp)          # 사용 기반 자동 분배(§7.4)
            emp.pay = Pay(emp.grade, emp.level)  # 급여 자동 인상
        # (2) 컨디션: 수행자 소모 / 비수행자 회복 [GD §2.5, §6.6]
        if emp performed a project this cycle:
            emp.cond = clamp(emp.cond − performed_drain[emp], 0, COND_MAX)
            if emp.cond < C_min: emp.overwork_days += 1; emp.loyalty −= overwork_loss(emp.overwork_days)  # 체증
        else:
            emp.cond = clamp(emp.cond + recover_rate(emp.placement), 0, COND_MAX)  # 미배치/유휴/표류/휴식 동일 규칙
        # (3) 퇴사 판정
        if emp.loyalty < Loy_min:
            if attrition_roll(emp): quit(emp)   # §3.3; 개인 미지급 소멸, 팀 공석 등 이벤트
      # (4) 의뢰 보드 유지보수 [GD §8.4]
      for dept in company.depts (dept_id asc):
        expire_postings(dept)                   # 게시 기한 경과분 소멸(상비 의뢰 면제)
        refill_board(dept)                      # 빈 슬롯 보충: 티어=BOARD_REFILL, 변동률=POSTING_VAR
      # (5) 재무 '주의' 오버레이 예측 갱신 [GD §5.6]
      company.caution = (balance < expected_next_monthend_fixed_cost(company))
    # (6) 마일스톤·알림 발신(§8 상태머신 트리거 판정) — 정준 순서로
    emit milestones, exception_alerts(자율 실패/팀장 부재/장기 저가동)
```

- [읽음] XP·컨디션·충성도·배치·급여 결과·게시판. [씀] 레벨·스탯·급여·컨디션·충성도·퇴사·보드·주의 플래그. [발행] EVT_LEVELUP, EVT_QUIT, EVT_ALERT_*, 마일스톤.
- **결정론:** ε·퇴사 롤이 좌표 키잉(§3)이라 순회 순서가 값에 무관. 이벤트 발행만 정준 순서.

---

## 5. 월말 정산 5단계 의사코드 [GD §2.3, §5.5]

30일차 일 처리 5단계 **직후**, 회사별 독립 실행. 순서는 [GD §2.3] 불변.

```
month_end_settle(company):
    # 1. 매출 확정 — 당월 I-1 지급분(일시+꼬리) 집계(재지급 아님, 리포트용)
    report.revenue = sum(month_credit_log(company))            # [읽음] 당월 원장 크레딧 로그

    # 2. 급여 지급 [GD §5.5-2]
    claims = [(emp, arrears[emp] + Pay(emp.grade, emp.level) + (팀장수당 if leader)) for emp in company.employees]
    sort claims by (청구액 asc, employee_id asc)                # 동률 시 근속 오래된(=id 작은) 순
    for (emp, claim) in claims:
        if balance >= claim:
            ledger_debit(company, claim); arrears[emp] = 0; emit EVT_PAID(emp)
        else:
            arrears[emp] = claim; emit EVT_UNPAID(emp, claim)   # 직원 단위 전액 원칙: 부분 지급 없음, 이월
    # (직원 시스템: 정시 지급→충성도 회복, 미지급→하락(월 누적 체증) — 아래 5단계에서 반영)

    # 3. 유지비 차감 [GD §5.5-3]
    due = unpaid_upkeep + U(company.grade)
    paid = min(balance, due); ledger_debit(company, paid)
    unpaid_upkeep = due − paid                                  # 부족분 미납 이월

    # 4. 등급 심사 [GD §5.5-4, §7.1]
    if company.pending_self_demotion and company.finance == 부실:  # 자진 강등(부실 한정)
        apply_self_demotion(company)                            # 유지비·상한↓, 명성 보존, 초과 예외(§8)
    company.promotion_eligible = (명성 ≥ R_up(g+1)) and (활성팀수 ≥ θ_util·개설부서수·T_max(g))
        # 자금 조건은 승인 후 다음 1단계 원자 재검증(§4.2); 여기선 표시만
    # (멀티: 슬롯 해금 판정이 이 등급 결과를 읽음 — §8)

    # 5. 월간 결산 리포트 + 상태 전이 확정 [GD §5.5-5, §5.6]
    #   충성도 급여 반영
    for emp: if EVT_PAID(emp): emp.loyalty += pay_recover
             else: emp.loyalty −= pay_loss(emp.unpaid_months++)  # 체증
    #   재무 상태 전이(§8 재무 상태머신)
    transition_finance_state(company)   # 미지급/미납 0이며 전액지급→건전(+부실카운터 0); 발생→부실; 부실 M_bankrupt개월→파산
    if company.finance == 파산: emit EVT_BANKRUPT(company)      # → 멀티 몰수형 소멸(§9)
    #   지원자 풀 전면 갱신(직원 시스템, 미채용자 소멸)
    regenerate_applicant_pool(company, month_index(day))       # 스트림 APPLICANT
    #   자율 부서 월간 요약(페널티 소실분 등, §9.6) 포함
    build_report(company)
```

- **[읽음]** 원장·arrears·미납·명성·가동률·재무 상태. **[씀]** 잔액·arrears·미납·충성도·재무 상태·승급 가능 표시·지원자 풀·리포트. **[발행]** EVT_PAID/UNPAID, EVT_BANKRUPT, 재무 전이 이벤트, 리포트.
- **결정론:** 회사별 독립(§1.2 DD-19). 지급 순서는 `(청구액, employee_id)` 전순서라 완전 결정. 지원자 풀 좌표 = (company_id, month) → 처리 순서 무관.

---

## 6. 자율 주행 선택 알고리즘 (완전 결정론 명세) [GD §9.3]

부서 단위, 일 처리 2단계, **무난수·완전 결정론**. 모든 정렬은 §1.2 ID로 종단 타이브레이크된다.

```
autonomy_select_and_launch(dept):
    # 입력 스냅샷(불변): 유휴 팀, 팀별 역량·평균 컨디션, 게이트 통과 게시물
    idle = [t in dept.teams if t.leader != null and t.project == null]  # 직접 점유 팀 자연 제외
    # 1. 유휴 팀 정렬: 팀 역량 내림차순, 동률 시 team_id 오름차순
    sort idle by (Cap_team desc, team_id asc)

    board = candidate_postings(dept)   # 선택가능식(§8.5) 통과분 + 상비 의뢰(게이트 면제)
    for team in idle:
        # 2a. 휴식 판정(P3)
        if team_avg_condition(team) < C_guard(dept.policy.P3):
            emit(rest); continue        # 그 사이클 미착수(회복은 §4.6 규칙으로 자동)
        # 2b. 후보 필터
        s = safety_factor(dept.policy.P1)                     # 안정>균형>도전, s≥1
        cand = [ p in board if p.C0 == 0
                              and est_days(team, p) * s <= L(p) ]  # 예상 소요일 안전 계수
        cand += 상비의뢰(dept)          # 필터·게이트 면제, 항상 포함(폴백 보증)
        # 2c. 점수화: P2 지정 자원의 명목 보상 ÷ 예상 소요일
        for p in cand: p.score = nominal_reward(p, dept.policy.P2) * FP / est_days(team, p)
                       # α_auto는 전 후보 동일 곱이므로 순위 무관 → 미적용
        # 2d. 배정: 점수 내림차순 → 티어 오름차순 → 기간(D_std) 오름차순 → 카탈로그 ID 오름차순
        sort cand by (score desc, tier asc, D_std asc, catalog_id asc)
        best = cand[0]
        launch(team, best, tag = AUTO)
        if best != 상비의뢰: board.remove(best)   # 상비 의뢰는 제거하지 않음(§9 DD-8, 다중 인스턴스)
    # 3. 부서 간 경합 없음(보드는 부서 소유) — 부서 처리 순서 무관, 정준 순서로 순회
```

- **설계 결정 (DD-15): `est_days(team, p) = ceil( D_std(t,d) · Cap_ref(t) · FP / (Cap_team · e_team_snapshot) )`** — `ε=0`, 현재 컨디션 스냅샷의 `e_team` 사용(§7.4). 결정론적 정수. 이 값은 필터(2b)·점수화(2c) 양쪽의 유일한 기간 추정치다.
- **상비 의뢰 폴백:** 게이트·필터·`C₀`=0·최저 난이도 [GD §8.4, §9.3-e] → 후보가 절대 공집합이 되지 않음(구조적 보증).
- **타이브레이커 완전성:** `(score, tier, D_std, catalog_id)`로도 동률이면 `posting_id` 오름차순을 최종 종단 키로 추가(상비 의뢰는 부서 고정 예약 ID). → 임의의 입력에서도 유일 결과.
- **비행동(명시적):** 진행 중 중단·교체 없음, 채용·해고·훈련·편성·개설·승급·자기 정책/모드 변경 없음 [GD §9.3, §9.4]. 유일 조직 변경 예외 = 표류 팀장 자동 승계(1단계 §4.2-B).

---

## 7. 핵심 수식 함수 라이브러리

game-design이 "단조 증가/감소"로만 정의한 관계에 **구현할 함수 형태를 확정**한다. 계수(`k_base`, `w_*`, `κ_*`, `φ_*`, `ζ`, `σ_hit` 등)의 값은 **balance.md 소관**이며 여기선 **구조**만 확정한다. 모든 스칼라는 고정소수(§2.3).

### 7.1 역량·팀

| 기호 | 시그니처 | 확정 형태 | 단조성 | 정의역/치역 |
|---|---|---|---|---|
| `k = f(L)` | 리더십→팀원 상한 | **계단 함수.** 경계 배열 `L_min=b_0<b_1<…<b_K`; `f(L) = k_base + max{i : b_i ≤ L}`. `f(L_min)=k_base≥1` [GD §6.7] | 비감소 | `L≥L_min`; `k∈[k_base,k_base+K]` |
| `m = m(L)` | 리더십→팀장 보정 | 비감소, `m≥1`. 권장 `m(L)=FP + κ_m·max(0, L−L_min)` (선형 상한 clamp) | 증가 | `m≥FP`(=1). `P_swap`일간 `m=FP` [GD §7.3] |
| `A_team` | 직무 스탯 집계 | `Σ_{p∈멤버∪팀장} (w_main(dept)·s_main(p) + w_sub(dept)·s_sub(p))`. 팀장 직무 스탯 포함 [GD §7.3]. `w_sub=0` 허용 | 합산 증가 | ≥0 |
| `Cap_team` | 팀 역량 | `A_team × m(L_leader) / FP`. 컨디션 미반영(정적) [GD §7.2] | 증가 | ≥0 |
| `Cap_dept` | 부서 역량 | `Σ_{활성 팀} Cap_team` (표류 팀 제외) [GD §7.2] | — | ≥0 |

### 7.2 진척·효율·컨디션

| 기호 | 형태 | 비고 |
|---|---|---|
| `e(C)` | 효율 계수 | **구간 선형+하한.** `e(C)=clamp( e_floor + (FP−e_floor)·(C−C_lo)/(C_hi−C_lo), e_floor, FP )`. 단조 증가, 상한 `FP`(=1) [GD §6.6] |
| `ΔP` | 일일 진척 | `base(Cap_team,t,d)·e_team/FP·(FP+ε)/FP` [GD §8.6] |
| `base` | 진척 기준율 | `(W(t,d)/D_std(t,d))·(Cap_team/Cap_ref(t))`. 기준역량 팀(`Cap_team=Cap_ref, e=1, ε=0`)은 정확히 `D_std`일에 완료 → `L=D_std·λ_due`, `est_days`와 일관 |
| `Cap_ref(t)` | 티어 기준 역량 | balance 테이블, `t`에 증가. "티어 기준 역량 팀"의 정의값 |
| `overload(Cap_team,t)` | 과부하 계수 | `max(FP, (Cap_ref(t)·FP/Cap_team)^q_ov)`. `Cap_team≥Cap_ref`면 1, 미달 시 >1 [GD §8.6] |
| 컨디션 소모 | 일일 | `c_base(d)·overload/FP` [GD §8.6] |
| `est_days` | 예상 소요일 | §6 DD-15. `ceil(D_std·Cap_ref·FP/(Cap_team·e_team))`, `ε=0` |

### 7.3 품질·보상

| 기호 | 형태 |
|---|---|
| `q_raw` | **세 인자 볼록 결합.** `q_raw = clamp( w1·x1 + w2·x2 + w3·x3, 0, FP )`, `Σw=FP`. 각 `x∈[0,FP]` 단조 증가(설계 결정 DD-16): `x1=`기한 여유율`=clamp((L−used_days)·FP/L,0,FP)`; `x2=`역량 초과율`=clamp((Cap_team·FP/Cap_ref(t) − FP)/z_sat, 0, FP)`; `x3=`평균 컨디션`=clamp((avgCond−C_q0)·FP/(C_q1−C_q0),0,FP)`. `avgCond=proj.cond_sum/proj.cond_samples`(러닝 집계 §4.4) [GD §8.7] |
| 품질 등급 | 절단점 `q_c1<q_c2`: `q_raw<q_c1`→미흡, `<q_c2`→표준, else 우수. `m_Q`(자금)·`r_Q`(명성) 등급별 배율, 등급에 증가 |
| 성과 등급 `Q` | (게임개발 출시형) DD-17. `y=clamp(q_raw + σ_hit·(u−FP/2)/FP·2, 0, FP)`, `u=uniform01_fp(HIT,id)`; 절단점 `d_c1<d_c2<d_c3`으로 4버킷 → 실패작/평작/성공작/히트작. `q_raw`↑ → 상위 등급 확률 단조↑. `σ_hit`=분산 폭(열린 질문). `q(Q)` 배율 등급에 증가 |
| 자금 보상 | `M(t,d)·v_post/FP·m_Q/FP·coef/FP·(α_auto if auto)/FP`, 원장 기입 직전 round-half-up 1회 [GD §8.7] |
| `coef` | 계수 소스 평가값 | 고정=`FP` / 시세=`μ(g,완료일)` / 성과=`q(Q)` / 감쇠=`δ(n)` [GD §8.7] |
| 명성 보상 | `Rep(t)·r_Q/FP·(α_auto if auto)/FP`, round-half-up [GD §7.6] |
| 실패 XP | `X(t,d)·p(진척률)/FP`, `p(x)=p_cap·x^q_p` (또는 선형), `p<1` [GD §8.8] |
| 꼬리 스케줄 | `D_tail(Q)`일, `daily(i)=T_total(Q)·ω_i`, `ω_i` 정규화 감쇠 가중(경과에 단조 감소, `Σω=FP`). **정수화는 최대 잉여법(Σ daily = T_total 보장)**. `D_tail`·`T_total` 은 `Q`에 증가 [GD §12.3] |

### 7.4 경제·성장·업종 함수

| 기호 | 형태 |
|---|---|
| `Pay(G,Lv)` | `P_base(G)·(FP + κ_pay·(Lv−1))/FP`. 등급·레벨에 증가. **스탯 무관** [GD §6.8]. 팀장 수당 = `Pay·수당률/FP` |
| `U(g)` | 등급별 테이블(5값), 증가 [GD §7.1] |
| `C_up(g→g+1)` | 등급 전이별 테이블, 증가. 승급 마진으로 유한 개월 회수(§가드레일 5.8-3) [GD §7.1] |
| 계약금 O-1 | `ContractBase(G) + κ_ct·Σ(초기 직무 스탯)`. 가드레일: ≥ 해당 직원 기본급 `n_guard`개월분 [GD §5.8-8] |
| 훈련비 O-2 | 현재치 볼록 증가(체증). 권장 `τ0(stat)·φ_train^s` (`φ_train>1`) 또는 `τ0·(s/S_ref)^q_train` (`q_train>1`) [GD §6.5] |
| `XP_need(Lv)` | 볼록 증가. 권장 `X0·ψ^(Lv−1)` (`ψ>1`) [GD §6.2] |
| 레벨 스탯 분배 | **사용 기반 자동.** 직전 레벨 구간의 (주 스탯 사용, 부 스탯 사용, 팀장 재직 시 리더십) 가중 비중으로 `S_lv` 포인트를 **최대 잉여법** 배분. 상한(잠재력 파생) 초과분은 우선순위 `주>부>리더십`으로 이월 [GD §6.2] |
| 성장 한계 | `level_cap = f_levelcap(잠재력)`(증가); `stat_cap(s) = min(f_statcap(level), potential_abs(s))` — 이중 구조 [GD §6.2] |
| `δ(n)` | 라인 감쇠 | `δ_min + (FP−δ_min)·ζ^n`, `ζ∈(0,FP)`, `n`=누적 완료. 하한 `δ_min>0` [GD §12.4] |
| `μ(g,day)` | 시세 계수 | **결정 함수.** `μ = FP + amp(g)·Σ_j sin_fp(2π·day/period_{g,j} + phase_{g,j})` (경계 진동, `amp` 저티어 품목군일수록 좁음). RNG 아님, 공개 [GD §12.2] |
| `GradeDist(rep)` | 지원자 등급 분포 | categorical over {C,B,A,S}, `P(≥A)`·`P(=S)` 명성에 단조 증가. inverse-CDF 추출 [GD §6.3] |
| `P_quit(loy)` | 퇴사 확률 | `loy≥Loy_min`→0; else `p_max·clamp((Loy_min−loy)/(Loy_min−Loy0),0,FP)^q_quit`. 부족분에 증가 [GD §6.9] |
| `G_liq` | 청산 반환 | `γ_cash·max(0, 잔액 − 미지급·미납 − C_seed(업종)) + γ_up·Σ(C_up 지출 누계)`. 모든 `γ∈(0,1)` → 항상 순손실 [GD §10.5] (형태는 GD 확정, 재확정만) |

- **양의 마진·자율 생존·승급 회수·설립 순소비** 등 관계식 가드레일 [GD §5.8-1..9]은 balance가 위 계수를 채울 때 만족해야 할 **제약**이며, 본 문서는 함수 형태가 그 제약을 표현 가능함을 보장한다(예: `α_auto·자율수입 > 급여+U`가 성립하도록 `α_auto` 대역 선택 여지 존재).

---

## 8. 상태 머신 상세

전이 시점 표기: `[S1]`=일 처리 1단계, `[S4]`=4단계, `[ME4]`=월말 4단계 등.

### 8.1 프로젝트 인스턴스

```
        (게시)                (착수: 직접[S1]/자율[S2])           (progress≥W)[S4]
템플릿 ─────────▶ 대기(Posted) ───────────────────────▶ 진행(InProgress) ───────────▶ 완료(Completed)
                    │  │                                    │  ├─(elapsed>L)[S4]────▶ 기한실패(FailedDeadline)
              (게시기한 경과)[S5]                            │  ├─(취소 확정)[S1]─────▶ 중도포기(Abandoned)
                    ▼  (상비 의뢰는 만료 없음)                 │  └─(회사 소멸)[S1]─────▶ 강제실패(정산 생략)
                 만료(Expired)                              (팀장 공석 시 진척0, 기한은 계속 소모)
```

| 전이 | 트리거 | 시점 | 조건 |
|---|---|---|---|
| 대기→진행 | 착수 명령/자율 배정 | S1(직접)/S2(자율) | 선택가능식(§8.5 GD) ∧ 팀 활성·유휴 ∧ `잔액≥C₀`(직접만) |
| 대기→만료 | 게시 기한 경과 | S5 | 비상비 의뢰만 |
| 진행→완료 | `누적 진척≥W` | S4 | 완료 우선(DD-6) |
| 진행→기한실패 | `경과일>L` ∧ 미완료 | S4 | 완료 미성립 시 |
| 진행→중도포기 | 취소 확정 | S1 | 대상 진행 중 |
| 진행→강제실패 | 회사 청산/파산 | S1 | 보상·XP 정산 생략 [GD §10.5] |

### 8.2 재무 상태 (회사) [GD §5.6]

```
   ┌──────── 건전(Healthy) ◀───(이월포함 전액지급 월말; 부실카운터=0)───┐
   │             │                                                    │
   │      (미지급/미납 발생)[ME5]                                       │
   │             ▼                                                    │
   └──────▶  부실(Distressed) ──(부실 연속 M_bankrupt개월, M≥2)[ME5]──▶ 파산(Bankrupt) → 몰수형 소멸(§8.6)
                 (자진 강등 허용, 충성도 하락, 복귀 체크리스트)

   [오버레이] 주의(Caution): 정식 상태 아님. 매일 S5 예측 갱신. 잔액 < 다음 월말 고정지출(이월+급여총+U). 규칙적 제약 없음
```

- 신규 회사 = **건전** 초기값 [GD §10.4]. 전이 확정은 월말 5단계 내부 [GD §5.6]. 주의는 어느 상태에도 중첩.

### 8.3 팀 [GD §7.3]

```
(신설[S1], 팀장 없음) ──▶ 표류(Drifting) ──(팀장 임명/자동승계)[S1]──▶ 활성(Active) ──(팀장 퇴사/이동)──▶ 표류
                                                                        │
                                                              (해체[S1], 진행 프로젝트 없을 때만) ─▶ 소멸(구성원→미배치)
```

| 상태 | 정의 | 효과 |
|---|---|---|
| 활성 | 팀장 존재 | 부서 역량 합산, 착수 가능 |
| 표류 | 팀장 공석(신설 포함) | 역량 합산 제외, 진행 프로젝트 진척 0(기한 소모), 신규 팀원 배치 금지, 불변식6 유예(§7.5) |

- **자동 승계(자율 부서만):** 표류 다음 사이클 S1-B에서 팀 내부 `L_min` 자격 최고 리더십자 임명(`P_swap` 적용). 자격자 없으면 표류 유지+알림 [GD §7.3, §9.4].

### 8.4 회사 슬롯 (경영자) [GD §10.3]

```
잠김(Locked) ──(해금조건, 월말 등급 심사 결과 읽어 자동판정)[ME]──▶ 빈 슬롯(Empty) ──(설립)[S1]──▶ 점유(Occupied) ──(소멸)[S1]──▶ 빈 슬롯
```

- 슬롯 1 = 게임 시작. 슬롯 n(2..10) = 중기업(3등급) 이상 n−1곳(개수 문턱 일반식) [GD §10.3]. **해금은 비가역·플레이어 귀속**(달성 회사 강등·소멸해도 회수 없음).

### 8.5 승급 예약 (회사) [GD §7.1]

```
없음 ──(월말 심사: 명성·가동률 충족)[ME4]──▶ 가능(Eligible) ──(플레이어 승인)──▶ 예약(Reserved)
                                              ▲                                    │
                                              │                        (다음 S1 원자 재검증)
                                              │                          ┌──────────┴──────────┐
                                              └──(재검증 실패, 가능 보존)─ 기각              통과 ─▶ 적용(−C_up, 상한 갱신)
```

- **설계 결정 (DD-12): 예약이 다음 S1 원자 재검증에서 실패(예: 잔액 부족)하면 예약만 소멸하고 "가능" 자격은 보존**된다(조건이 유지되는 한 [GD §7.1] "만료 없음"). 플레이어는 재승인할 수 있다.

### 8.6 회사 생애주기 (멀티) [GD §10.4, §10.5]

```
(설립: 첫회사/회사0=무료, 추가=스폰서자금 또는 골드)[S1]
     └─▶ 초기상태(스타트업·C_seed·명성0·건전·부서1선택·창립선발) ─── 운영 ───┐
                                                                            │
        ┌───────────────────────────────────────────────────────────────────┤
   (청산: 플레이어, G_liq 반환)         (파산: 경제 판정, 반환 0/몰수)          │
        └────────────┬───────────────────────────────────────┬──────────────┘
                     ▼           동일 정리 절차(§9)             ▼
              슬롯 반환 · 직원/꼬리 소멸 · 해금·마일스톤·골드 보존
```

### 8.7 자율 부서 모드 & 직원 수명주기 (요약)

- **부서 모드:** 수동 ⇄ 자율(전환 S1, 비용·쿨다운 없음; 자율 전환은 해금 플래그 필요) [GD §9.2]. 착수 태그는 **착수 시 인스턴스에 고정**(주체 기준), 이후 전환에 불변.
- **직원:** 지원자 → (채용 S1) → 미배치 → 배치(팀원/팀장) → (해고 S1 / 퇴사 S5) → 소멸. 회사 소멸 시 전원 소멸.

---

## 9. 엣지케이스 처리 통합

game-design 전역에 흩어진 엣지케이스를 한 곳에 구현 규칙으로 모으고, 규칙이 없던 지점은 **설계 결정:** 으로 확정한다.

### 9.1 조직·팀

1. **팀장 공석 중 진척 0, 기한 계속 소모** [GD §7.3, §8.5]. 별도 유예 없음. 기한 초과 시 기한 실패.
2. **k 축소 자동 이동** [GD §7.3]: 팀장 교체·강등·자동 승계로 `k`가 줄면 초과 인원을 S1에 미배치 풀로 이동. 플레이어 미지정 시 **직무 스탯 합 낮은 순**. **설계 결정 (DD-10): 동률 시 `employee_id` 큰(=근속 짧은/신규) 순으로 방출**한다 — 투자된 고참을 보존.
3. **자동 승계 대상 선택** [GD §7.3]: 팀 내부 `L_min` 자격자 중 최고 리더십. **설계 결정 (DD-11): 동률 시 `employee_id` 오름차순(=근속 오래된 우선) 임명.**
4. **다중 동일-사이클 구조 명령** [GD §7.3, §1.4]: 제출 순서로 순차 적용, 각 직후 불변식 검증. **설계 결정 (DD-13): 진행 프로젝트가 있는 팀에 대한 팀장 교체가 사이클 내 여러 번이면 `P_swap` 타이머는 마지막 교체 시점에서 재시작**하고, `k` 축소는 최종 팀장 기준으로 1회 정산.
5. **진행 중 팀 해체 불가** [GD §7.3]; **진행 중 타 팀 이관 불가** [GD §8.9]. 조정은 팀원 교체로만(역량 변화는 다음 진척부터).
6. **표류 팀 신규 팀원 배치 금지**(기존 잔류) [GD §7.3]. 불변식6은 표류 중 유예, 새 팀장 임명 시 즉시 회복 [GD §7.5].
7. **자진 강등 후 상한 초과**: 인원·팀·부서 해체하지 않되 신규 유입(채용·팀 신설·증원·부서 개설) 금지, 초과 부서도 정상 운영(재승급으로만 해소) [GD §7.1, §7.5].

### 9.2 경제·급여·정산

8. **급여일 자금 부족**: 청구액 = 이월 미지급 + 당월 급여. 잔액 부족 시 **청구액 낮은 직원부터, 동률 근속 오래된(=id 작은) 순** 전액 지급, 미지급분 이월(부분 지급 없음) [GD §5.5-2].
9. **유지비 부족**: 잔액 전액 차감 후 부족분 미납 이월 [GD §5.5-3].
10. **퇴사·해고자 개인 미지급 잔액 소멸**(청구 주체 소멸) [GD §5.2]. 이를 노린 해고는 계약금 가드레일로 항상 손해 [GD §5.8-8].
11. **잔액 = 청구액**: 선불 판정은 `잔액 ≥ 청구액`(inclusive) → 잔액 0까지 허용 [GD §5.2].
12. **실패 무과금·`C₀` 비환불** [GD §5.3, §8.8]: 프로젝트 실패 시 금전 유출입 없음, 위약금 없음, 착수 비용만 손실.
13. **시세 품목 착수/완료 비대칭**: 매입가 `C₀·μ(g,착수일)` 선불(S1), 판매가 `R_base·μ(g,완료일)` [GD §12.2]. 각각 round-half-up.
14. **승급 재검증 실패 시 가능 표시 보존** (DD-12, §8.5).
15. **설계 결정 (DD-18): 충성도 반영 시점** — 과로 하락은 **매일 S5**(수행 중 `컨디션<C_min` 일수 체증), 급여 결과(정시↑/미지급↓)는 **월말 S5**(payroll 결과 확정 후). 팀장 승격↑·강등↓은 **S1**(즉시).

### 9.3 프로젝트·진척

16. **완료 우선 판정** (DD-6, §4.5). **같은 사이클 착수-완료 허용** (DD-7).
17. **참여일수 정의** — **설계 결정 (DD-9): 프로젝트가 InProgress인 각 사이클에 팀에 소속된 구성원은 참여일수 1 누적(표류일 포함, 진척 0이어도 시간 구속)**. 팀장은 leader_days 별도 누적(가중). 중도 합류·이탈자는 자기 누적분만 XP 분배 [GD §8.7].
18. **XP 다중 레벨업**: 한 S5에서 XP가 여러 레벨분이면 `xp < XP_need(next) 또는 level=level_cap`까지 루프. 각 레벨업마다 스탯 자동 분배 [GD §6.2].
19. **ε 하한 보장** (DD-5): 진척 음수·0 불가.
20. **부서 역량이 티어 문턱 아래로 하락해도 진행 유지**(게이트는 착수 1회) [GD §8.5].
21. **자율 착수 vs 직접 하달 경합**: 직접(S1)이 팀 점유 → 자율(S2)에서 자연 제외 [GD §9.2, §4.3].
22. **설계 결정 (DD-8): 상비 의뢰 템플릿 의미론** — 상비 의뢰는 단일 게시물이 아니라 **부서당 항상 존재하는 템플릿**으로, 착수 시 인스턴스가 생성되며 **후보 목록에서 제거되지 않는다**. 따라서 같은 사이클에 여러 유휴 팀이 각자 상비 의뢰 인스턴스를 착수할 수 있고, 플레이어도 여러 팀에 상비 의뢰를 직접 하달할 수 있다. 반면 일반 게시물은 단일 인스턴스라 착수 시 보드에서 제거된다(중복 착수 시 두 번째 명령 기각). 이로써 [GD §8.4·§9.3]의 "후보 공집합 없음" 보증과 다팀 폴백이 정합된다.
23. **동일 게시물 이중 착수 명령**(일반 게시물): 제출 순서상 첫 명령이 착수(보드 제거), 두 번째는 재검증 실패로 기각 [GD §1.4].

### 9.4 해금·꼬리·업종 메커닉

24. **동적 해금 술어 재잠김**: "활성 꼬리 보유 자사 출시작 존재"가 전 꼬리 소진으로 거짓이 되면 라이브운영 B 재잠김 [GD §8.3]. 착수 시점마다 선택가능식에서 평가.
25. **꼬리 갱신 대상 선택**: 직접 = 착수 시 플레이어 1개 지정(활성 대상 없으면 착수 거부), 자율 = 잔여 기간 최단, 동률 시 완료 오래된 순 [GD §8.3]. 대상은 착수 시 고정, 수행 중 소진돼도 완료 시 갱신 유효(갱신 횟수 상한은 정상 소모).
26. **회사 소멸 시 활성 꼬리 소멸**: 경제 지급 엔진이 해당 회사 꼬리 스케줄 잔여분 전량 드롭 [GD §5.3, §10.5].
27. **δ(n)·해금 카운터는 완료 시 갱신**(§4.5). 라인 감쇠는 하한 `δ_min>0` 수렴 [GD §12.4].

### 9.5 생애주기·정리 절차

28. **청산/파산 정리 절차(단일)** [GD §10.5] — 다음 S1 일괄:
```
liquidate_or_bankrupt(company, mode):
    1. 진행 중 프로젝트 전부 강제 실패 — 보상·XP 정산 생략
    2. 소속 직원 전원 소멸(계약 해지, 개인 미지급 잔액 소멸), 지원자 풀 반환 없음
    3. if mode==청산: G_liq(§7.4) 골드 지급; 잔여 자금·명성·활성 꼬리 전액 소멸
       if mode==파산: 몰수(반환 0), 동일하게 잔여 전액 소멸
    4. 슬롯 반환(점유→빈). 해금·마일스톤·골드는 보존
```
29. **회사 0 재설립 = 첫 회사 규칙**(설립비 면제·시드·창립 선발) [GD §10.4] — 데드락 부재 보증.
30. **마지막 1사 청산 허용** [GD §10.5]. **파산 직전 청산은 의도된 플레이**(몰수 대비 반환 확보).
31. **[GD §5.8-9] 왕복 손실 구조적 보장**: 시드·채무 공제 + `γ<1` + 명성 리셋 → 설립·청산 반복 항상 순손실. 별도 쿨다운 불요.

### 9.6 신규 발굴 엣지케이스 (설계 결정)

32. **설계 결정 (DD-a): 30일차 완료·꼬리 지급과 월말 매출 집계 순서** — 30일차 일 처리 4단계의 완료/꼬리 크레딧이 원장에 먼저 반영되고, 그 **직후** 월말 1단계 매출 확정이 당월 크레딧 로그를 집계하므로 30일차 지급분이 자연 포함된다 [GD §5.5-1]. 별도 처리 불요.
33. **설계 결정 (DD-b): 30일차 퇴사와 급여 상호작용** — 30일차 일 처리 5단계 퇴사가 월말 2단계 급여보다 먼저 실행되므로, 그날 퇴사한 직원은 당월 급여 청구에서 제외되고 개인 미지급 잔액도 소멸한다(§5.2와 정합). 의도된 순서.
34. **설계 결정 (DD-c): 완료와 동시 팀 해방 후 같은 사이클 재착수 불가** — 완료(S4)로 팀이 해방돼도 자율 재선택은 이미 지난 S2에서 끝났고, 직접 재착수는 다음 사이클 S1이다. 한 사이클에 한 팀이 두 프로젝트를 수행할 수 없다(팀당 동시 1개 [GD §8.5]).
35. **설계 결정 (DD-d): 부동소수 미사용 강제**로 인한 나눗셈 0 방어 — `Cap_team=0`(팀장 단독이나 스탯 0 등 극단)일 때 `est_days`·`base`의 분모 0을 막기 위해 `Cap_team = max(1, Cap_team)`으로 floor. 팀장 직무 스탯 포함(§7.1)이라 활성 팀은 통상 >0이나 방어적 하한을 둔다.
36. **설계 결정 (DD-e): 지원자 풀 크기 변동 시 좌표 안정성** — 풀 재생성은 (company_id, month) 좌표라 크기 `N_pool`이 밸런스로 바뀌어도 같은 달·회사면 앞쪽 인덱스 지원자는 동일 생성(applicant_index 좌표). 리플레이 안정.
37. **설계 결정 (DD-f): 회사 처리 순서 무관성 검증 훅** — 회사 간 자원 경합이 없음을 CI에서 보장하기 위해, 임의 슬롯 순열로 월말을 처리해도 상태 해시가 동일해야 한다는 프로퍼티 테스트를 technical에 인계(§11).

---

## 10. 데이터 정의 (필드 스키마)

타입 수준 정의(저장 표현·인덱싱은 technical 소관). `ref<T>`=엔티티 ID 참조, `fp`=고정소수(§2.3), `enum{…}`.

### 10.1 프로젝트

**템플릿(업종 콘텐츠 공급, 정적) — 기본 필드 [GD §8.3]:**

| 필드 | 타입 | 비고 |
|---|---|---|
| catalog_id | id | 정렬 타이브레이커·해금 참조 |
| industry / dept_type | enum | 게시 보드 결정 |
| tier `t` | enum{T1..T5} | 부서 역량 게이트 |
| difficulty `d` | enum{하,중,상} | 강도 |
| work `W(t,d)` | i64 | 총 작업량 |
| std_days `D_std` | i32 | 표준 기간(표시·`base`·`L` 기준) |
| base_reward `M(t,d)` | i64 | 기본 보수 |
| rep_reward `Rep(t)` | i64 | 명성 |
| xp_pool `X(t,d)` | i64 | XP 풀 |
| posting_ttl `게시 기한` | i32 | 만료일수(상비 의뢰=∞) |
| is_staple `상비 의뢰` | bool | 게이트·필터 면제 |

**확장 필드 6종 [GD §8.3]:**

| 필드 | 타입 | 기본값 |
|---|---|---|
| `C₀` | i64 (+시세 품목군 `g`) | 0 |
| 보상 계수 소스 | enum{고정, 시세(g), 성과등급, 라인감쇠} | 고정 |
| 보상 스케줄 | enum{일시, 일시+꼬리(파라미터)} | 일시 |
| 해금 조건 | enum{없음, 완료이력술어(+성과등급 문턱?), 동적상태술어} | 없음 |
| 완료 부가 효과 | enum{없음, 해금(target), 꼬리 갱신(target 규칙)} | 없음 |
| 반복 가능 | enum{1회성, 반복} | 1회성 |

**인스턴스(런타임) [GD §8.3]:**

| 필드 | 타입 |
|---|---|
| project_id / template_ref | id / ref |
| state | enum{대기,진행,완료,기한실패,중도포기,강제실패} |
| progress / elapsed_days | i64 / i32 |
| team_ref | ref<Team> |
| launch_tag | enum{직접,자율} (착수 시 고정) |
| C₀_paid_at_launch | i64 (시세 반영 실차감액) |
| posted_variance `v_post` | fp (게시 시 롤 고정) |
| participation_days / leader_days | map<ref<Emp>, i32> |
| cond_sum / cond_samples | i64 / i32 (q_raw 평균 컨디션) |
| tail_target_ref? | ref (꼬리 갱신 대상, 착수 시 고정) |
| resolved_Q? | enum{실패작..히트작} (완료 시 저장, 재롤 금지) |

### 10.2 직원·지원자

**직원 [GD §6]:**

| 필드 | 타입 |
|---|---|
| employee_id / company_ref | id / ref |
| grade | enum{C,B,A,S} (잠재력 대역 공개 라벨) |
| potential | fp (불변) |
| stat_제작/기획/영업 | i32 |
| leadership | i32 |
| level / xp | i32 / i64 |
| stat_usage_accum | map (레벨 분배용 사용 비중) |
| condition / loyalty | i32 / i32 |
| overwork_days / unpaid_months | i32 / i32 |
| placement | enum{미배치, 팀원(team_ref), 팀장(team_ref)} |
| base_pay(캐시) | i64 (=Pay(grade,level)) |
| hired_day | i32 (근속·타이브레이크 참고, 실 타이브레이크는 id) |
| arrears | i64 (개인 미지급 잔액) |

**지원자/풀 [GD §6.4]:** applicant_index, grade, 초기 스탯 3종, leadership, potential, 계약금, 기본급 (전부 공개). 풀 메타: company_ref, 생성 month/founding_seq, `N_pool`, is_founding(리더십≥`L_min` 1인 보장).

### 10.3 회사·부서·팀·경영자

**회사 [GD §7, §5, §10.4]:** company_id, slot_idx, industry, grade{1..5}, balance(i64), reputation(i64), finance_state{건전,부실,파산}, caution(bool), distress_months(i32), unpaid_upkeep(i64), promotion_eligible(bool), pending_promotion(bool), pending_self_demotion(bool), C_up_spent_total(i64, `G_liq`용), founding_seed(C_seed 캐시), 부서 목록, 미배치 풀(ref 집합), month_credit_log.

**부서 [GD §7.2, §9]:** dept_id, dept_type, mode{수동,자율}, policy{P1,P2,P3}, teams[], board(§10.5), idle_streak(팀별 저가동 카운터), 기본 정책은 회사 보유(상속용).

**팀 [GD §7.3]:** team_id, dept_ref, leader_ref?, member_refs[], project_ref?, is_drifting(bool), drift_seen_prev_cycle(bool), p_swap_until(i32, `m=1` 만료일).

**경영자(플레이어) [GD §10.2]:** gold(i64), companies[](0..10), slots[10]{잠김,빈,점유}, slot_unlock_flags[10](비가역), autonomy_unlocked(bool), milestone_flags[M1..M15], 완주 이력(업종별), liquidation/bankruptcy 통계, master_seed, day.

### 10.4 경제 원장·꼬리·미지급 [GD §5]

- **원장 트랜잭션:** company_ref, day, kind(유입 2종/유출 8종 태그), amount(i64), memo. 잔액 하한 0.
- **꼬리 스케줄(경제 소유):** release_id, company_ref, launch_tag, auto_tag(α_auto 적용 여부), D_tail, age, daily_amount[](사전 정수화, Σ=T_total), remaining_days(동적 술어 조회용).
- **미지급:** 직원별 arrears(§10.2) + 회사 unpaid_upkeep(§10.3).

### 10.5 보드·게시물

- **보드:** dept_ref, slots[`S_board`] (상비 의뢰 1 예약 슬롯 포함), posting refs.
- **게시물:** posting_id, template_ref, tier(잠김 미리보기는 열람 전용 플래그), posted_day, expire_day(상비=∞), v_post(고정), 선택 가능 여부(파생).

### 10.6 업종 콘텐츠 카탈로그 스키마 [GD §12]

업종 = **데이터 팩**(코드 분기 없음 [GD §12.1]). 한 업종 팩이 제공:

| 스키마 요소 | 내용 |
|---|---|
| industry_def | id, 표시명, 포지셔닝(기간/분산/선투입/핵심판단 — §12.5 행) |
| dept_types[3] | dept_type id, 주 직무 스탯, 부 직무 스탯?(가중 `w_main/w_sub`), 공통 영업부 권장 |
| catalog[] | 프로젝트 템플릿 배열(기본+확장 6종, §10.1) — 부서마다 해금 없는 기초 카테고리 ≥1 |
| 해석 함수 파라미터 | 시세 `μ`(품목군 3, `amp/period/phase`), 꼬리 스케줄(`D_tail(Q)/ω`), 감쇠 `δ`(`δ_min/ζ`), 해금 그래프(완료이력·동적술어), 성과 등급 매핑 절단점 `d_c*`·`σ_hit` |
| 경제 데이터 | `C_seed`, `F_found`, `F_found_G` (가드레일 §5.8 충족) |

- **업종 추가 = 팩 추가.** 시스템(§5~§11) 규칙 불변 [GD §12.8]. 고유 메커닉은 확장 6종 조합으로만 표현.

---

## 11. balance.md · technical-architecture.md 인계 인터페이스

- **함수 형태(§7)는 확정, 계수는 balance 채움.** balance는 §7 각 행의 기호(`k_base`, `w_main/w_sub`, `κ_pay`, `φ_train`, `ψ`, `ζ`, `δ_min`, `σ_hit`, `α_auto`, `θ_util`, `M_bankrupt`, `γ_cash/γ_up`, `ε_max`, `C_guard`, `s(성향)`, 절단점 `q_c*`/`d_c*`, `Cap_ref(t)`, `C_seed/F_found`)만 정의하고 형태를 바꾸지 않는다.
- **가드레일(§7.4 말미, [GD §5.8])**은 balance가 만족해야 할 제약. 함수 형태가 제약 충족 여지를 보장.
- **technical이 반드시 고정할 것:** ① 해시 알고리즘·믹싱 상수(§3.1) ② 고정소수 스케일 `FP`/`PROB_ONE`(§2.3) ③ round-half-up·최대 잉여법 구현(§2.1) ④ 정준 순회 순서·엔티티 ID 카운터의 결정론 저장(§1.2·1.3) ⑤ 명령 큐 직렬화·리플레이 로그(§1.4) ⑥ RNG 상태=`master_seed`+`day`만 세이브(§3.4) ⑦ 회사 순서 무관성 프로퍼티 테스트(DD-f).

---

## 12. 열린 질문

본 문서는 game-design의 규칙을 결정론 구현 세부로 확정했다. 아래는 밸런스/기술 확정이 필요한 **구현 수준** 질문(게임 규칙 질문은 [GD §15] 참조).

1. **해시 PRNG 선택** — SplitMix64/PCG/xxHash 중 성능·플랫폼 재현성 균형. 32/64비트 출력 폭과 분포 매핑 편향 허용치.
2. **고정소수 스케일** — `FP=10⁶`이 `μ`·`q_raw`·`e`·복리형 곱(`ζ^n`, `φ^s`)의 누적 반올림 오차를 게임 기간 내 무해하게 유지하는지. 필요 시 Q32.32 상향.
3. **`base`/`est_days`의 `Cap_ref(t)` 이원 사용** — 진척 정규화와 품질 `x2`·과부하가 모두 `Cap_ref`에 의존하므로 이 한 커브의 튜닝이 3개 서브시스템에 동시 파급. 분리 필요성 검토([GD §15-7]과 연동).
4. **꼬리 정수화 잔차 배분** — 최대 잉여법이 초반 큰 일일 지급에 잉여를 몰아줄 때 체감 왜곡. 잔차를 균등 분산할지 앞당길지.
5. **`avgCond` 러닝 집계의 표류일 처리** — 진척 0인 표류일을 평균 컨디션 표본에 넣을지(현재 §4.4는 수행일만 표본화). 품질에 미치는 방향 확인.
6. **레벨 스탯 자동 분배의 상한 이월 우선순위** — `주>부>리더십`이 팀장 육성 의도와 충돌하지 않는지(팀장인데 리더십이 마지막 우선순위).
7. **명령 큐 리플레이 로그 크기** — 장기 세션에서 입력 로그 누적 vs 주기적 상태 스냅샷 체크포인트 전략(technical).
8. **`μ(g,day)` 사인 합의 주기 설계** — "몰라도 평균 흑자, 활용하면 상방"([GD §12.2])을 만족하는 진폭·주기 조합과 미래 예고 차단(공개는 현재값만).
9. **성과 등급 `y` 절단점과 `σ_hit`의 결합** — `q_raw`가 절단점 근처일 때 흥행 롤의 등급 역전 빈도가 "분산 최대" 정체성과 좌절감([GD §15-3]) 사이 어디에 놓이는지.
10. **동일 업종 복수 회사의 시세 롤 독립성** — `μ`는 전역 결정 함수라 공유되지만 `POSTING_VAR`·`HIT`는 인스턴스 키잉이라 독립. 이 조합이 무역 병렬 수확([GD §15-13])을 충분히 억제하는지 시뮬레이션.
