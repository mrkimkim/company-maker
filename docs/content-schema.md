# 콘텐츠 팩 데이터 포맷 스키마 (업종 팩) v1.0

> **문서 지위.** 본 문서는 "업종 = 데이터 팩, 코드 분기 없음" 원칙([GD §12.1·§12.8], [MS §10.6], [TA §2.5])을 **실제 기계판독 포맷으로 확정**한다. 한 업종의 부서 구성·프로젝트 카탈로그·고유 메커닉을 `game-design §8.3`의 기본 필드 + 확장 필드 6종만으로 표현하는 JSON 팩의 스키마다. 본 문서는 **새 규칙·새 필드를 만들지 않는다** — 이미 확정된 필드([MS §10.1·§10.6])의 데이터 표현 규약일 뿐이다.
>
> **정본 관계.** 스키마 구조·필드 의미 = 본 문서. 필드 목록·타입의 상위 정본 = `mechanics-spec.md §10`. 게임 규칙 = `game-design.md`. 수치 = `balance.md`. 문자열 키 = `docs/localization/string-key-convention.md` + `glossary.md`. 충돌 시 상위 정본이 우선한다.
>
> **참조 인스턴스.** 본 스키마를 따르는 완전 인스턴스: `data/content/trading.json`(무역 업종). 예시는 모두 이 파일에서 인용한다.
>
> **인용:** `[GD §x]`=game-design.md, `[MS §x]`=mechanics-spec.md, `[BAL §x]`=balance.md, `[TA §x]`=technical-architecture.md.

---

## 1. 설계 원칙 (팩이 반드시 지키는 것)

1. **데이터만, 코드 없음.** 팩은 값·enum·파라미터만 담는다. 진행·정산 로직은 확장 필드 6종의 **해석 함수**([MS §7.4]: μ, 꼬리, δ, 해금 술어)만 평가하며 업종별 `if` 분기를 두지 않는다([GD §12.1-1], [TA §2.5]).
2. **고유 메커닉은 확장 필드 6종의 조합으로만.** 업종당 정확히 1개. 불가능하면 그것은 팩이 아니라 `[GD §8.3]` 개정 안건이다([GD §12.8-3]).
3. **닫힌 enum + 파라미터 참조.** "보상 계수 소스·해금 조건·완료 부가 효과"는 자유 문자열 코드가 아니라 **닫힌 enum + 파라미터 테이블 참조**로 직렬화한다. 코어가 임의 코드를 로드·실행하지 않는다(스토어 정책·보안 [TA §2.5·§7.2]).
4. **문자열은 전부 로케일 키.** 팩의 어떤 표시 문자열도 값으로 한국어/영어를 담지 않는다. 전부 `l10n` 키(§11)이며 실제 텍스트는 문자열 테이블(`data/localization/`)이 소유한다.
5. **모든 부서는 개전 즉시 유효.** 각 부서는 해금 없는 기초 카테고리 ≥ 1을 보유하고, 상비 의뢰가 모든 보드에 존재한다([GD §12.1-4]).
6. **결정론 정합.** 팩은 **버전 태그**(`pack_version`)를 가지며 세이브에 핀되어 결정론을 보존한다([TA §7.3]). 카탈로그는 `catalog_id`로 정렬 가능해야 한다([TA §2.5], [MS §1.2]).

---

## 2. 팩 최상위 구조

업종 1종 = JSON 오브젝트 1개 = 파일 1개(`data/content/<industry>.json`).

| 키 | 타입 | 필수 | 의미 |
|---|---|---|---|
| `schema_version` | string(semver) | ✔ | 본 스키마 문서의 버전. 로더 호환성 판정 |
| `pack_id` | string(id) | ✔ | 업종 팩 식별자. `industry_def.id`와 동일 |
| `pack_version` | string(semver) | ✔ | 콘텐츠 버전 태그. 세이브 핀 대상([TA §7.3]) |
| `refs` | object | – | 출처 문서·절 인용(사람용 메타, 로직 무관) |
| `todo_token` | string | – | 미확정 수치 표기 토큰 선언(§13). 기본 `"TODO:balance"` |
| `industry_def` | object | ✔ | 업종 메타(§3) |
| `stat_weights` | object | ✔ | `A_team` 집계 가중 `w_main`/`w_sub`([BAL §2.4] 인용, §4) |
| `dept_types` | array[3] | ✔ | 부서 종류 3종 정의(§4) |
| `interpretation_params` | object | ✔ | 확장 필드 해석 함수 파라미터(§8) |
| `staple_request` | object | ✔ | 상비 의뢰 정의(§7) |
| `catalog` | array | ✔ | 프로젝트 템플릿 배열(§5·§6) |
| `economy` | object | ✔ | 경제 데이터 `c_seed`/`f_found`/`f_found_g`(§9) |

- **필드 순서는 로직에 무관**하다(순회는 `catalog_id` 등 정준 키로만, [MS §1.2]). 위 순서는 가독성 권장안이다.

---

## 3. `industry_def` — 업종 메타

| 키 | 타입 | 의미 · 제약 |
|---|---|---|
| `id` | string(id) | 업종 ID. `pack_id`와 동일. 슬러그 소문자 |
| `slug` | string | 카탈로그 키·경로용 슬러그. 무역=`trade`, 게임개발=`gamedev`, 화장품=`cosmetics`([glossary §6]) |
| `name_key` | l10n key | 업종 표시명. `catalog.<slug>.industry_def.name`([conv §4.1]) |
| `flavor_key` | l10n key | 업종 한줄 소개(transcreation) |
| `positioning` | object | `[GD §12.5]` 특성 행. 아래 |

**`positioning`**([GD §12.5]·[TA §2.5]): `duration`(기간) / `feedback_loop`(피드백 루프) / `variance`(수익 분산) / `upfront`(선투입) / `key_decision`(핵심 판단) enum 코드 + `key_decision_key` l10n. 예(무역): `duration="shortest"`, `variance="low_to_mid"`, `upfront="small_to_mid"`, `key_decision="launch_timing"`. UI·튜토리얼·자율 친화도 판단의 데이터 근거이며 로직 게이트는 아니다.

---

## 4. `dept_types` — 부서 정의와 주/부 스탯 매핑

배열 길이 정확히 3([GD §7.2] 업종당 3종). 공통 영업부 포함 권장([GD §12.1], [GD §12.8-1]).

| 키 | 타입 | 의미 · 제약 |
|---|---|---|
| `id` | string(id) | 부서 종류 ID. `<slug>.<dept_slug>` 형식(§10) |
| `slug` | string | 부서 슬러그. `sales`/`sourcing`/`logistics`/`dev`/`liveops`/`rnd`/`production`([glossary §6]) |
| `name_key` | l10n key | 부서 표시명. **termbase `dept.*` 키 재사용, 신설 금지**([glossary §6], §11) |
| `common` | bool | 공통 부서(영업부) 여부. 학습 전이 표식 |
| `main_stat` | enum | 주 직무 스탯. `production`/`planning`/`sales`(제작/기획/영업, [glossary §4]) |
| `sub_stat` | enum \| null | 부 직무 스탯. 없으면 `null`(이때 `w_sub`는 0 취급, [MS §7.1]) |

- **`stat_weights`**(팩 최상위): `w_main`·`w_sub`. 팀 역량 집계 `A_team = Σ(w_main·s_main + w_sub·s_sub)`([MS §7.1]). 값은 [BAL §2.4] 인용(`w_main=1.0`, `w_sub=0.4`). 밸런스 소유값을 팩이 인스턴스화한 것이며, 드리프트 방지를 위해 balance 개정 시 동기화한다.
- **주/부 스탯 매핑은 팩(콘텐츠) 소유 데이터**다([GD §6.2] "부서→주/부 스탯 매핑은 업종 콘텐츠 데이터"). `game-design`은 "주 1종 + 선택적 부 1종" 구조만 확정하고 무역 부서별 구체 스탯은 지정하지 않으므로, 아래 매핑은 **팩의 콘텐츠 설계 결정**이다(§13에 근거 열거).

| 부서 | main | sub | 근거 |
|---|---|---|---|
| 영업부(`trade.sales`) | 영업 | 기획 | 중개=영업 주도 매칭, 기획이 딜 구조 보조 |
| 소싱부(`trade.sourcing`) | 기획 | 영업 | 시세·타이밍 분석(기획) 주도, 협상(영업) 보조 |
| 물류부(`trade.logistics`) | 제작 | 기획 | 운송·보관=실행(제작), 라우팅 설계(기획) 보조 |

---

## 5. `catalog[]` — 프로젝트 템플릿 (기본 필드)

각 원소 = 프로젝트 템플릿 1개(정적 데이터, [MS §8.2·§10.1]). 인스턴스(런타임 상태)는 프로젝트 시스템이 생성하며 팩에 없다.

**기본 필드**([MS §10.1], [GD §8.3]):

| 키 | 타입 | 의미 · 제약 |
|---|---|---|
| `catalog_id` | string(id) | 템플릿 식별자. 정렬 타이브레이커·해금 참조([MS §1.2]). §10 규약 |
| `dept_type` | string(ref) | 게시 보드 결정. `dept_types[].id` 참조. (업종은 `pack_id`로 자명 → 템플릿에 `industry` 중복 미기재) |
| `tier` | enum{`T1`..`T5`} | 부서 역량 게이트([GD §8.5]). 회사 등급과 동일 5단계 |
| `difficulty` | enum{`low`,`mid`,`high`} | 같은 티어 내 강도(하/중/상). 컨디션 소모 `c_base(d)` 조회 키([MS §7.2]). 본 팩은 명명 프로젝트를 `mid` 기준으로 인스턴스화(§12) |
| `work` | i64 | 총 작업량 `W(t,d)`. 누적 진척 ≥ W = 완료 |
| `std_days` | i32 | 표준 기간 `D_std`. 진척 정규화·표시·마감 `L=D_std×λ_due` 기준([MS §7.2]) |
| `base_reward` | i64 | 기본 보수 `M(t,d)`. 시세 프로젝트에서는 `R_base`(판매 기준액) |
| `rep_reward` | i64 | 명성 `Rep(t)`. **티어 전용**(난이도 무관, [GD §8.3]) |
| `xp_pool` | i64 | XP 풀 `X(t,d)` |
| `posting_ttl` | i32 \| null \| token | 게시 기한(만료일수). `null`=만료 없음(상비 의뢰 전용, [GD §8.4]). 미확정은 토큰(§13) |
| `is_staple` | bool | 상비 의뢰 여부. 명명 프로젝트는 `false`(상비는 §7 별도) |
| `ext` | object | 확장 필드 6종(§6) |

- `L`(마감 기한)·`c_base(d)`·`base` 진척률 등 **파생값은 팩에 저장하지 않는다** — 전역 상수(`λ_due`, [BAL §1])·balance 테이블과 `std_days`/`difficulty`로 런타임 산출한다([MS §7.2]).

---

## 6. 확장 필드 6종 (`ext`) — 업종 메커닉의 유일한 표현 수단

`[GD §8.3]`·`[MS §10.1]` 확장 필드 6종의 데이터 표현. 전부 **닫힌 목록**이며 신설은 `[GD §8.3]` 개정 선행.

| `ext` 키 | 타입 | enum 값 → 게임 의미 | 기본값 | 파라미터 참조 |
|---|---|---|---|---|
| `c0` | i64 \| token | 착수 비용 `C₀`(경제 O-5 선불, 실패·포기 비환불). 시세 프로젝트는 `C₀×μ(g,착수일)` 실차감 | `0` | — |
| `commodity_group` | string(ref) \| null | 시세 품목군 `g`. `coefficient_source="market_price"`일 때만 유효, 그 외 `null` | `null` | `interpretation_params.market_price.commodity_groups[].id` |
| `coefficient_source` | enum | `fixed`=고정(×1) / `market_price`=시세(μ(g,완료일)) / `performance_grade`=성과등급(q(Q)) / `line_decay`=라인감쇠(δ(n)) | `fixed` | 시세→`market_price`, 성과→`performance_grade`, 감쇠→`line_decay` 테이블(§8) |
| `reward_schedule` | enum | `lump`=일시 / `lump_plus_tail`=일시+꼬리(경과에 단조 감소) | `lump` | 꼬리→`tail_schedule` 테이블(§8) |
| `unlock` | enum | `none`=없음 / `completion_history`=완료 이력 술어(+성과 문턱 옵션) / `dynamic_state`=동적 상태 술어(활성 꼬리 보유 자사 출시작 존재) | `none` | `unlock_graph`(§8). 완료이력=비가역, 동적상태=재잠김 가능([GD §8.3]) |
| `completion_effect` | enum | `none`=없음 / `unlock`=해금(target) / `tail_refresh`=꼬리 갱신(target 규칙 [GD §8.3]) | `none` | `unlock_graph` / 대상 참조 |
| `repeatable` | enum | `one_shot`=1회성 / `repeat`=반복(누적 완료 카운터 `n`) | `repeat` 또는 `one_shot`(카테고리별) | `line_decay`의 `δ(n)` 입력이 `n` |

- **enum ↔ 한국어 정본 매핑**(문서 가독용): `fixed`=고정, `market_price`=시세, `performance_grade`=성과등급, `line_decay`=라인감쇠, `lump`=일시, `lump_plus_tail`=일시+꼬리, `none`=없음, `completion_history`=완료이력술어, `dynamic_state`=동적상태술어, `unlock`=해금, `tail_refresh`=꼬리갱신, `one_shot`=1회성, `repeat`=반복.
- **해금 조건에 성과 문턱을 부가**할 때(완료이력 술어): `unlock_graph`에 `{predicate:"completion_history", target:<catalog_id>, min_perf_grade:<enum>}`로 파라미터화(무역 팩은 미사용).
- **완료 부가 효과의 target·선택 규칙**([GD §8.3] "꼬리 갱신 대상 선택")은 `unlock_graph`가 파라미터로 담고, 직접/자율 대상 지정 규칙은 코어([MS §4.5·§9.4])가 고정한다 — 팩은 규칙을 재정의하지 않는다.
- **무역 팩의 사용 현황:** `c0`·`commodity_group`·`coefficient_source`·`repeatable`만 사용. `reward_schedule`은 전부 `lump`, `unlock`·`completion_effect`는 전부 `none`. 무역 고유 메커닉(시세 마진)은 `coefficient_source=market_price` + `c0>0`(소싱부) 조합 하나로 표현된다.

---

## 7. `staple_request` — 상비 의뢰 표현

상비 의뢰는 **부서당 항상 존재하는 템플릿**([MS §9.3 DD-8])이며, 만료 없이 **역량 게이트를 면제**받는다([GD §8.4]). 값이 업종 불문 범용([BAL §4.3])이므로 팩에 **단일 정의**를 두고 `applies_to`가 가리키는 모든 부서 보드가 각기 1건씩 예약 슬롯으로 인스턴스화한다([MS §10.5]).

| 키 | 타입 | 값(무역) · 의미 |
|---|---|---|
| `catalog_id` | string(id) | `trade.staple`. 부서당 고정 최소 예약 ID([MS §1.2] "상비=부서당 고정 최소값") |
| `name_key` | l10n key | `mechanic.standing_job`(termbase 재사용, [glossary §4]) |
| `flavor_key` | l10n key | 상비 의뢰 플레이버 |
| `applies_to` | array[ref] | 이 상비를 예약하는 부서 종류 목록(전 부서) |
| `tier`/`difficulty` | enum | `T1`/`low`([GD §8.4]) |
| `work`/`std_days` | i64/i32 | `390`/`5`([BAL §4.3]) |
| `base_reward`/`rep_reward`/`xp_pool` | i64 | `60000`/`8`/`60`([BAL §4.3], 최저 보상 — 위기 생명줄·자율 폴백) |
| `posting_ttl` | null | `null`=만료 없음([GD §8.4]) |
| `is_staple` | bool | `true`(게이트·필터 면제 표식) |
| `ext` | object | 전 기본값: `c0=0`, `coefficient_source=fixed`, `reward_schedule=lump`, `unlock=none`, `completion_effect=none`, `repeatable=repeat` |

- 상비 값은 balance 절대값이므로 **무역 산업 배율(§12)을 적용하지 않는다**.

---

## 8. `interpretation_params` — 해석 함수 파라미터

확장 필드 enum이 가리키는 파라미터 테이블. 업종이 쓰지 않는 계열은 `null`(코어가 해당 해석 함수를 호출하지 않음).

| 키 | 타입 | 무역 | 의미 |
|---|---|---|---|
| `market_price` | object \| null | 사용 | 시세 μ 파라미터(아래) |
| `tail_schedule` | object \| null | `null` | 꼬리 `D_tail(Q)`/`ω`(게임개발 전용) |
| `line_decay` | object \| null | `null` | 라인 감쇠 `δ_min`/`ζ`(화장품 전용) |
| `performance_grade` | object \| null | `null` | 성과 등급 절단점 `d_c*`·`σ_hit`(게임개발 전용) |
| `unlock_graph` | object \| null | `null` | 해금 그래프(완료이력·동적술어 술어 테이블) |

**`market_price`**([MS §7.4], [GD §12.2]): `μ(g,day) = 1 + amp(g)·Σ_j sin(2π·day/period_j + phase_j)`. RNG 아님, 일자의 결정적 함수, 현재값만 공개.

| `commodity_groups[]` 키 | 타입 | 의미 · 제약 |
|---|---|---|
| `id` | string(id) | 품목군 g 식별자. `ext.commodity_group`가 참조 |
| `name_key` | l10n key | 품목군 표시명 |
| `amp` | fp(실수, ×FP) | 진동 진폭. **저티어 품목군일수록 좁다**([GD §12.2]) |
| `periods` | array[i32] | 사인 항 주기(일). 항 개수 = `phases` 길이 |
| `phases` | array[fp] | 사인 항 위상. `periods`와 1:1 |

무역 3종([BAL §5.2]): `consumer_goods`(생활소비재, amp 0.08, {17,41}) / `raw_materials`(원자재, amp 0.15, {23,53}) / `electronics`(전자기기, amp 0.25, {13,29,61}). **`phases`는 balance 미정 → 잠정 0**(§13).

---

## 9. `economy` — 경제 데이터

`[MS §10.6]`·`[TA §2.5]` 경제 스칼라. 가드레일 `[GD §5.8]` 충족은 balance 검증 소관.

| 키 | 타입 | 값(무역) | 의미 |
|---|---|---|---|
| `c_seed` | i64(₩) | `2000000` | 설립 초기 자본 `C_seed`(시스템 지급, [GD §5.3 I-2]) |
| `f_found` | i64(₩) | `3500000` | 자금 지불형 설립비 `F_found`(스폰서 차감 싱크). 가드레일 `F_found>C_seed`([GD §5.8-5]) |
| `f_found_g` | i64(골드) | `350` | 골드 지불형 설립비 `F_found_G` = `F_found/ρ_G`(ρ_G=10,000₩/골드, [BAL §6.2]) |

값 출처 [BAL §6.2]. `f_found_g`는 `ρ_G` 환산 결과이며 `ρ_G` 자체는 balance 소유 상수(팩에 미저장).

---

## 10. ID 네이밍 규약

로케일 불변·기계 정렬용 식별자. 전부 소문자, `.` 구분, ASCII.

| 대상 | 형식 | 예 |
|---|---|---|
| 업종 `pack_id`/`industry_def.id` | `<slug>` | `trade` |
| 부서 `dept_types[].id` | `<slug>.<dept_slug>` | `trade.sourcing` |
| 프로젝트 `catalog_id` | `<slug>.<dept_slug>[.<category_slug>].t<N>` | `trade.sourcing.t5`, `gamedev.dev.self.t1` |
| 상비 의뢰 `catalog_id` | `<slug>.staple` | `trade.staple` |
| 품목군 `id` | `<group_slug>` | `electronics` |

- **카테고리 세그먼트 규칙:** 부서에 카테고리(프로젝트 계열)가 **1개뿐이면 생략**하고(무역: `trade.sourcing.t5`), **2개 이상이면 카테고리 슬러그를 반드시 포함**한다(게임개발 개발부 A/B: `gamedev.dev.self.t1`·`gamedev.dev.outsource.t1`, 화장품 R&D·생산부: `cosmetics.rnd.line_dev.t1`·`cosmetics.production.own_line.t1`). `catalog_id`는 오름차순 정렬 키일 뿐이므로 세그먼트 수는 로더에 영향이 없다(불투명 정렬 문자열, [MS §1.2]). 로케일 키(`catalog.*.name|flavor`)도 동일 구조를 미러링한다([conv §4.1]).

- **`catalog_id`는 정렬 키**([TA §2.5] "catalog_id 정렬", [MS §1.2] 타이브레이커). 로더는 `catalog_id` 오름차순 뷰를 유지한다. 파일 내 배열 순서(본 인스턴스는 부서별 그룹핑)는 권장 가독 순서일 뿐 비권위적이다.
- **런타임 엔티티 ID(u64 카운터, [TA §2.1])와 별개다.** `catalog_id`는 정적 콘텐츠 식별자, 엔티티 ID는 인스턴스 생성 시 부여되는 결정론 카운터다.
- 부서 슬러그는 termbase([glossary §6]) 고정 어휘를 재사용한다(신설 금지).

---

## 11. 로케일 키 규약 (문자열 분리)

팩은 **로직 값과 표시 문자열을 분리**한다([conv §4], [GD §12.8]). 표시 문자열은 값이 아니라 키만 담고, 실제 텍스트는 문자열 테이블이 로케일별로 소유한다. 업종 추가 시 현지화가 데이터 팩 확장으로 닫힌다.

| 대상 | 키 형식([conv §4.1]) | 예 |
|---|---|---|
| 업종명 | `catalog.<slug>.industry_def.name` | `catalog.trade.industry_def.name` |
| 업종 플레이버 | `catalog.<slug>.industry_def.flavor` | — |
| 부서명 | **termbase `dept.*` 재사용** | `dept.sales` / `dept.sourcing` / `dept.logistics` |
| 프로젝트 기능명(직역) | `catalog.<slug>.<dept>.t<N>.name` | `catalog.trade.sourcing.t5.name` |
| 프로젝트 플레이버(≤24자, transcreation) | `catalog.<slug>.<dept>.t<N>.flavor` | `catalog.trade.sourcing.t5.flavor` |
| 품목군명 | `catalog.<slug>.commodity.<id>.name` | `catalog.trade.commodity.electronics.name` |
| 상비 의뢰명 | termbase `mechanic.standing_job` 재사용 | `mechanic.standing_job` |

- **부서명은 termbase `dept.*` 키를 재사용**한다([glossary §6] "부서명 규칙 용어 — 신설 금지"). `[conv §4.1]`의 `catalog.<industry>.<dept>.dept_name` 형식은 termbase에 없는 신규 부서용 폴백이며, 무역 3부서는 모두 termbase에 존재하므로 `dept.sales`/`dept.sourcing`/`dept.logistics`를 직접 참조한다(화면 간 완전 일치·CI G3 검출 정합).
- 키에 한국어·언어 코드·동적 값(금액·수치)을 넣지 않는다([conv §1]). 키는 값이 바뀌어도 안정.
- 재화·숫자·날짜 표기는 문자열 테이블/렌더 계층이 처리([conv §3]) — 팩은 관여하지 않는다.

---

## 12. 값 파생 규칙 (balance 인용)

명명 프로젝트의 기본 필드는 [BAL §4.1] 기준 테이블(난이도 중)에 [BAL §5.1] 무역 산업 배율을 곱해 인스턴스화한다. 정수화는 **round-half-up**([MS §2.1]).

- **적용 배율(무역, [BAL §5.1]):** `std_days ×0.60`, `base_reward ×0.90`. `work`·`xp_pool`·`rep_reward`는 산업 배율 없음. `rep_reward`는 티어 전용(난이도 무관).
- **본 팩 난이도:** 명명 프로젝트 전부 `mid`(중, 난이도 배율 ×1.0). `low`/`high` 변형은 [BAL §4.1] 난이도 배율표로 파생 가능하며 스키마 변경 없이 템플릿 추가만으로 확장된다(현 카탈로그 규약은 티어당 1건, [conv 열린질문 3]).

**영업부·물류부(고정 계수), 그리고 소싱부의 `work`/`std_days`/`rep`/`xp`:**

| tier | work | std_days | base_reward | rep | xp |
|---|---|---|---|---|---|
| T1 | 600 | 4 | 162,000 | 20 | 120 |
| T2 | 1,500 | 7 | 378,000 | 55 | 300 |
| T3 | 3,200 | 12 | 855,000 | 140 | 720 |
| T4 | 6,000 | 19 | 1,890,000 | 340 | 1,600 |
| T5 | 10,000 | 29 | 4,050,000 | 800 | 3,400 |

- `std_days`: 6·12·20·32·48 ×0.60 = 3.6·7.2·12·19.2·28.8 → 4·7·12·19·29(round-half-up).
- `base_reward`: 180k·420k·950k·2.1M·4.5M ×0.90.

**소싱부(시세, 트레이딩 딜, [BAL §5.2]):** `base_reward`는 판매 기준액 `R_base`, `c0`는 매입 원가. 가드레일 `R_base>C₀`(μ=1, [GD §5.8-7]). balance는 **T1만 명시**(`R_base=200,000`, `C₀=120,000`, 중립 마진 +80,000). T2~T5의 `base_reward`·`c0`는 미확정 → 토큰(§13). `work`/`std_days`/`rep`/`xp`는 위 표준 트레이드 커브를 공유한다.

품목군 매핑([GD §12.2]): T1·T2 = `consumer_goods`, T3·T5 = `raw_materials`, T4 = `electronics`.

---

## 13. 미확정 값 (TODO) 목록

`balance.md`(1차안)에 값이 없어 `todo_token`(`"TODO:balance"`)으로 표기한 항목. 로더는 팩 검증 시 토큰을 **거부**(미완성 팩 게이트)해야 한다.

| # | 위치 | 항목 | 상태 |
|---|---|---|---|
| 1 | `catalog[].posting_ttl` (명명 15건 전부) | 게시 기한(만료일수) | balance 미정. 보드 유지보수 전역값으로 추정되나 [BAL §1] 전역 상수·[BAL §4]에 부재 |
| 2 | `trade.sourcing.t2/t3/t4/t5 .base_reward` (4건) | 트레이딩 딜 `R_base` | balance는 T1만 명시([BAL §5.2]) |
| 3 | `trade.sourcing.t2/t3/t4/t5 .ext.c0` (4건) | 트레이딩 딜 매입 원가 `C₀` | balance는 T1만 명시([BAL §5.2]) |
| 4 | `interpretation_params.market_price.commodity_groups[].phases` (총 7개) | 사인 항 위상 `phase_{g,j}` | balance 미정([BAL §5.2]는 amp·period만). μ 계산 위해 **잠정 0**으로 채움(토큰 아님 — 수치 필요). [BAL 열린질문]·[MS 열린질문 8]과 연동 |

**비-balance 콘텐츠 결정(참고, TODO 아님):**
- 부서 주/부 스탯 매핑(§4)은 팩 소유 콘텐츠 결정이며 game-design이 무역에 대해 미지정. 위 매핑은 본 팩의 설계안으로, 플레이테스트 튜닝 대상이다.
- 명명 프로젝트 난이도 = `mid` 고정은 현 카탈로그 키 규약(티어당 1건)의 반영이며, 난이도 변형 도입 여부는 [conv 열린질문 3].

---

## 열린 질문

1. **`posting_ttl`의 소유·입도** — 게시 기한이 업종 콘텐츠 데이터인가, 아니면 부서/프로젝트 시스템의 전역/티어 기본값인가. 후자라면 balance 전역 상수로 승격해 팩에서 제거하고, 팩은 필요 시 오버라이드만 담는 편이 데이터 중복을 줄인다.
2. **트레이딩 딜 `R_base`/`C₀` 커브** — balance가 T1만 정의했다. T2~T5를 표준 트레이드 M 커브에 얹을지(고정 배수), 티어별 위험(기간·품목군 amp)에 연동한 별도 커브로 둘지. 후자는 "고위험 고수익" 정체성([GD §12.2] T5)을 수치로 표현하지만 튜닝면이 늘어난다.
3. **주/부 스탯 매핑의 정본 위치** — 부서→스탯 매핑을 팩(콘텐츠)이 소유하되, game-design이 무역에 대해 미지정이라 팩이 사실상 규칙을 신설한 모양새다. game-design §12 카탈로그에 매핑 행을 명시해 정본을 상류로 올릴지 검토.
4. **난이도 변형의 데이터 표현** — 티어당 1 템플릿(난이도 mid) vs 티어×난이도 곱집합(하/중/상 3배)의 스키마·문자열 물량 트레이드오프([conv 열린질문 3]). 곱집합 시 `catalog_id`·플레이버 키 물량과 board refill 풀([MS §3.3])의 균등 선택 분포 영향.
5. **`phases` 확정 절차** — 사인 항 위상이 "몰라도 평균 흑자, 활용하면 상방"([GD §12.2], [MS 열린질문 8])을 만족하도록 amp·period와 결합 설계되어야 한다. 위상을 balance가 소유할지, 팩이 품목군 정체성으로 소유할지 경계 확정 필요.
6. **팩 검증 게이트의 토큰 정책** — `TODO:balance` 토큰을 로더가 하드 거부할지(프로덕션 빌드), 개발 빌드에서 placeholder 값으로 통과시킬지. CI 콘텐츠 린트([TA §11])와 정합할 기준.
7. **상비 의뢰의 팩 소유 여부** — 값이 업종 불문 범용([BAL §4.3])이므로 팩마다 재정의하는 현 구조가 중복이다. 코어/balance가 단일 상비 템플릿을 소유하고 팩은 `applies_to`만 선언하는 편이 DRY하나, "모든 부서 보드에 존재" 보증을 어디서 강제할지 결정 필요.
