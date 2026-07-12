# 문자열 키 규약 · ICU · 표기 규칙 (M1 SubTask 1.4.2)

문서 지위: 본 문서는 `accessibility-localization.md` §8(국제화 설계)의 **구현 규격**이다. §8.1(외부화·하드코딩 금지·키 규약), §8.2(ICU MessageFormat), §8.6(재화·숫자·날짜 표기 정책), §8.4(길이 예산)을 코드/데이터 팀이 착수 가능한 규칙으로 확정한다. 상위 계약과 충돌하면 `accessibility-localization.md`가 우선한다. 용어의 정본은 `game-design.md` §4이며(신조어 금지), 문자열의 톤 정본은 `narrative-tone.md`(리아 단일 화자)이다.

정본 파일:
- 용어집(termbase): `docs/localization/glossary.md`
- 스타터 문자열 테이블: `data/localization/strings.sample.json`

---

## 1. 키 네이밍 규약 — `<표면>.<맥락>.<식별자>`

`accessibility-localization.md` §8.1 계약(`notif.crisis.unpaid`, `onboarding.m1.hire`, `catalog.trade.sourcing.t5`)을 3세그먼트 규약으로 확정한다.

```
<surface>.<context>.<identifier>[.<sub>]
   │          │            │          └ 선택: 상태/변형(예: .empty, .t5, .m7)
   │          │            └ 식별자: 무엇에 대한 문자열인가
   │          └ 맥락: 어느 흐름/화면/상황인가
   └ 표면: 어떤 종류의 문자열인가(고정 어휘)
```

- **키는 로케일 불변, 값만 로케일별**(§8.1). 키에 한국어·언어 코드·동적 값(회사명·금액·직원명)을 넣지 않는다.
- **표기:** 전부 소문자 `snake_case`, 세그먼트 구분은 `.`(점). 공백·대문자·비ASCII 금지 → 빌드 게이트가 강제(§5).
- **키는 안정적이다:** 값이 바뀌어도 키는 유지, 의미가 바뀌면 새 키. 키 삭제는 미사용 확인 후.

### 1.1 표면(surface) 고정 어휘 — 화면 맵 S0~S21과 정렬

| surface | 용도 | 대표 ux 화면 | transcreation |
|---|---|---|---|
| `ui` | 버튼·라벨·탭·스탯명·헤더(≤6자 라벨 포함) | 전 화면 | 기능=직역 |
| `notif` | 알림·큐 항목(위기/개입/정보 3티어) | S0·S18 | 직역 |
| `onboarding` | M1~M6 코치마크 | S5·S4·S6·S7·S11·S9 | 직역(+작은 축하는 톤 적응) |
| `milestone` | M1~M15 마일스톤 축하 | S12 | 톤 적응(transcreation) |
| `event` | 시스템 사건 반응 플레이버(완료/성과/실패/퇴사) | S7·S17 | 톤 적응 |
| `dialog` | 리아 대사(리포트 소감·위기 안내·상황 코멘트) | S11·S16 등 | 톤 적응 |
| `report` | 결산 리포트 항목 라벨(수입/지출/순이익 등) | S11·S13 | 직역 |
| `tooltip` | 용어·게이트 재프레이밍 툴팁 | 전 화면 | 직역(정확) |
| `status` | 상태 라벨(재무·팀·직원 상태) | S1·S2·S8 | 직역 |
| `resource` | 자원·재화 라벨(자금·골드·명성·XP·컨디션·충성도·다이아) | 전 화면 | 직역 |
| `error` | 명령 거부·엣지 사유(§7.3) | 전 화면 | 직역(모호성 0) |
| `settings` | 설정 항목명(접근성/오디오/언어) | S20 | 직역 |
| `catalog` | 업종 데이터 팩 표시 문자열(§4) | S6·S14 | 기능명=직역, 플레이버=적응 |
| `common` | 표면 공통(확인/취소/닫기/예약됨 등) | 전 화면 | 직역 |
| `glossary` | 용어집 termbase 표제 stem(§glossary.md) | 참조용 | — |

표면 어휘를 신설하려면 본 절 개정이 선행한다(어휘 폭주 방지).

### 1.2 맥락(context)·식별자 규칙

- **맥락:** 흐름·화면·상황을 나타낸다 — `crisis`/`intervene`/`info`(알림 티어), `m1`~`m15`(온보딩·마일스톤), `finance`/`team`/`employee`(status), `report`(dialog) 등.
- **식별자:** 그 맥락 안의 구체 문자열 — 동사·명사 단수형. 라벨은 화면 액션과 1:1(`ui.recruit.confirm` = 채용 확정 버튼).
- **한 키 = 한 상황.** 같은 문자열이 두 상황에 쓰이면 키를 분리한다(§8.2 문맥 오역 방지, `accessibility` §10.2). "확정"이 채용 확정인지 승급 승인인지는 키로 구분(`ui.recruit.confirm` vs `ui.promotion.approve`).

**예시(스타터 테이블과 정합):**
```
onboarding.m1.hire          notif.crisis.unpaid          status.finance.distressed
ui.recruit.confirm          notif.intervene.captain_empty resource.funds.label
ui.roster.assign            notif.info.idle              dialog.report.good
ui.board.dispatch           common.action.confirm        error.funds.insufficient
ui.finance.payroll          common.action.cancel         catalog.trade.sourcing.t5
```

---

## 2. ICU MessageFormat 사용법

`accessibility-localization.md` §8.2 결정("문자열 포맷 = ICU MessageFormat 계열")을 문법 규칙으로 확정한다. **문장은 변수 슬롯을 포함한 완전한 템플릿**으로 두고, 문자열 이어붙이기(concatenation)로 조립하지 않는다 — 어순(한/일 SOV, 영/중 SVO)이 언어별 템플릿에서 흡수된다.

### 2.1 변수 삽입 (interpolation)
- 회사명·직원명·금액·일수·회사 태그는 `{변수}` 슬롯. 문장을 변수로 쪼개 잇지 않는다.
- 표준 변수명(전 로케일 공통): `{company}` `{employee}` `{amount}` `{days}` `{months}` `{count}` `{tier}` `{grade}` `{dept}`.
- 예: `"{company}의 팀장 자리가 비었어요."` / `"{company} has no team lead."`

### 2.2 복수형 (plural)
- CJK(ko/ja/zh)는 복수 굴절이 없어 `other`만, 영어는 `one`/`other`. `#`는 해당 수치.
```
ko: "{days, plural, other {#일째 저가동이에요.}}"
en: "{days, plural, one {Idle for # day.} other {Idle for # days.}}"
```
- 숫자만 다르고 문형이 같아도 plural 블록으로 감싸 언어별 복수 규칙을 열어둔다.

### 2.3 선택 (select) — 성별·등급·변형 분기
- 성별: narrative가 직원 성별을 성능·서사에 결부하지 않으므로 **문법 성별 분기는 기본 회피**하고, 3인칭이 필요한 EN 문장은 성 중립(they / 이름 재사용)으로 둔다(§8.2). 성별 select는 불가피한 로케일에서만.
- 변형 분기(성과 등급 등)는 `select`로:
```
ko: "{q, select, hit {히트작이에요!! 꼬리 수익이 한동안 들어와요.} success {성공작이에요.} modest {평작이에요.} other {이번엔 조용히 묻혔어요. 다음에 살리죠.}}"
```

### 2.4 한국어 조사 처리 (§8.2)
- 변수 뒤 조사(은/는·이/가·을/를)는 받침 유무로 달라진다. 원칙:
  1. **조사 비의존 문형 우선** — `"{company} — 팀장 공석"`처럼 조사를 피하는 어미/구두점 구조.
  2. 불가피하면 **조사 자동 선택**(받침 판정 헬퍼) 대상으로 키 메타에 `josa: true` 표시. 동적 이름(회사·직원)이 조사 앞에 오는 문장은 자동화 후보로 플래그.
- 이 규칙은 조사 자동화가 없는 로케일(en/ja/zh)에는 영향 없다.

### 2.5 중첩·숫자 포맷
- plural 안에 변수, select 안에 plural 중첩 허용하되 2단계까지(가독·LQA 비용). 숫자는 `{amount, number}`로 로케일 포맷(§3)에 위임.

---

## 3. 재화 · 숫자 · 날짜 표기 규칙 (§8.6 정본)

### 3.1 재화(자금·골드·다이아)
- **설계 결정 상속(§8.6): 게임 재화 기호는 전 로케일 공통, 아이콘이 1차 채널·텍스트 기호는 폴백.** 자금은 전용 아이콘 1차 노출, `₩` 텍스트는 폴백으로만(무국적 세계관·CJK 통화 혼동 방지·색맹 안전, art §3.6). 골드·다이아는 각각 전용 아이콘.
- 재화 **수량(숫자)**은 로케일 숫자 규칙·축약을 따르되 **기호/아이콘은 공통**. 문자열에는 아이콘 자리표시자를 두고 수치만 삽입: `"{icon_funds}{amount}"` 형태(아이콘은 렌더 계층이 주입, 키 값에 하드 글리프 금지).
- 스토어 실화폐(IAP)는 스토어 제공 현지 통화 문자열을 그대로 노출(정본 `monetization.md`) — 게임 재화와 표기상 명확히 구분.

### 3.2 숫자
- 천단위 구분자·소수점·자릿수 그룹은 로케일 규칙(`{n, number}`).
- 대형 수 축약 단위는 **문자열 외부화 대상**: `ko` 만/억 · `ja` 万/億 · `zh-Hans` 万/亿 · `en` K/M/B. 축약 임계·단위 문자열은 로케일 리소스로 관리(§8.1).
- 증감은 부호(+/−)+방향 아이콘(↑/↓)+색으로, **색 단독 금지**(accessibility §2.1).

### 3.3 날짜·시간
- 게임 내 시간은 게임 달력(일 Day / 월 = 30일, game-design §2.1)이며 실제 캘린더가 아니다. "D-일"·"N개월"은 로케일 단위(일/day/日)로 현지화, ICU plural로 처리(§2.2).
- 실제 날짜(세이브 시각·오퍼 기한 등, monetization 소관)만 로케일 날짜·시간 포맷을 따른다.

---

## 4. 콘텐츠 팩(업종 카탈로그) 문자열 키 규약

`accessibility-localization.md` §8.1·§10.4, game-design §12.8(업종 = 코드 분기 없는 데이터 팩)에 대응한다. 카탈로그 데이터는 **로직 값(티어·보상 파라미터)과 표시 문자열(로케일별)을 분리**해 담는다. 업종 추가 시 신규 문자열 현지화가 데이터 팩 확장으로 닫힌다.

### 4.1 카탈로그 키 형태
```
catalog.<industry>.<dept>[.<category>].<tier>[.<slot>]
        │          │        │           │       └ name | flavor | (선택) desc
        │          │        │           └ t1~t5
        │          │        └ (부서에 카테고리 2개 이상일 때만) self|outsource|service|own|line_dev|oem|own_line ...
        │          └ sourcing | logistics | dev | liveops | rnd | production | sales
        └ trade | gamedev | cosmetics   (업종 slug)
```
- **카테고리 세그먼트 규칙(content-schema §10과 동일):** 부서 카테고리가 1개면 생략(`catalog.trade.sourcing.t5`), 2개 이상이면 포함(`catalog.gamedev.dev.self.t1`, `catalog.cosmetics.rnd.line_dev.t1`). `catalog_id`와 로케일 키가 동일 구조를 미러링한다.
- **업종 표시명:** `catalog.<industry>.industry_def.name`
- **부서 표시명:** `catalog.<industry>.<dept>.dept_name` (공통 영업부는 용어집 기존 값 `dept.sales` 재사용, 신설 금지 — narrative §5.3)
- **기능명(직역 원칙):** `catalog.trade.sourcing.t5.name` → game-design §12 카탈로그 기능명 정본
- **플레이버(≤24자·1행, transcreation):** `catalog.trade.sourcing.t5.flavor`

예: `catalog.trade.sourcing.t5.name` = "원자재 장기 계약 딜" / `.flavor` = "길게 묶는 만큼 크게 먹거나, 크게 데인다."

### 4.2 업종 추가 현지화 산출물(§10.4 체크리스트)
신규 업종 1종 = 데이터 팩 추가로, 현지화 관점에서 아래로 닫힌다:
1. `industry_def.name` 전 로케일  2. 고유 부서 2종 `dept_name`(공통 영업부 재사용)  3. `catalog[].name` 전 항목(직역)  4. `.flavor` 부서·티어별(≤24자, transcreation)  5. 용어집 증분(신 메커닉 명칭 termbase 등재)  6. 로케일별 회사명 추천 패턴 갱신(필요 시)  7. 문화 리스크 리뷰(§9.5). 코어 문자열·파이프라인 구조 변경을 요구하지 않는다.

---

## 5. 하드코딩 금지 · 빌드 게이트

`accessibility-localization.md` §8.1 결정("하드코딩된 사용자 문자열은 빌드 게이트에서 차단")을 게이트 규칙으로 확정한다.

- **외부화 대상:** UI 라벨·리아 대사·알림·툴팁·카탈로그 표시명·오류/거부 사유·숫자 단위(만/억)·날짜 포맷 문자열.
- **비대상(외부화 불필요):** 로케일별 이름 풀(§glossary 이름 풀 정책, 별도 shipping)·재화 아이콘 글리프(§3.1).
- **빌드 게이트(CI):**
  - **G1 하드코딩 검출:** 코드·씬의 사용자 노출 리터럴 문자열(비ASCII·UI 바인딩) 발견 시 빌드 실패. 문자열은 키 참조로만.
  - **G2 키 규약 검증:** 키가 `snake_case`·소문자·표면 어휘(§1.1)를 위반하면 실패.
  - **G3 미번역 키:** 로케일 값 누락·폴백(en) 노출 키를 플래그(런치 로케일 ko/en는 0 필수).
  - **G4 ICU 문법·변수 정합:** 변수 슬롯이 로케일 간 불일치하거나 ICU 파싱 실패 시 실패.
  - **G5 길이 예산 초과:** §6 예산 초과·1행 상한 위반·렌더 잘림(…) 검출(로케일별, accessibility §10.3).
  - **G6 글리프 커버리지:** 두부(□) 글리프 0(§8.3).
- 게이트는 규칙이 아니라 파이프라인 강제 사항이다 — `technical-architecture.md`(리소스 포맷·로더)와 정합.

---

## 6. 길이 예산 표기 (§8.4 단일 기준)

KO 기준 상한 + 현지화 +40% 흡수(ux §9 / narrative §7.1과 단일 기준). 키 메타 `budget` 필드에 표면 예산을 명시한다.

| budget 값 | KO 상한 | 현지화 여유 | 표면 |
|---|---|---|---|
| `notif` | ≤40자·1행 | +40% 흡수 | 알림·큐 항목(회사 태그+상황+행동 힌트) |
| `label` | ≤6자 | 아이콘 병기 | 버튼·라벨·스탯명(최소 폰트 이하 축소 금지) |
| `flavor` | ≤24자·1행 | +40% | 프로젝트 플레이버(기능명 별도) |
| `tooltip` | 1~2문장 | 리플로우 | 용어·규칙 설명 |
| `onboarding` | 1~2문장 | 리플로우 | 한 걸음 한 개념 |
| `report` | 1문장 | 리플로우 | 리포트 소감 |
| `milestone` | 1~2문장 | 리플로우 | 전환점(M7·M9·M12·M15) 완화 |
| `ending` | 제한 없음 | 리플로우 | 완주 엔딩(유일 예외) |

- `label`(≤6자)의 CJK↔라틴 비대칭: 6자 KO 라벨이 EN에서 길어지면 아이콘 병기·약어·툴팁 보완(잘림보다 아이콘+짧은 텍스트 우선).
- 폰트 스케일 확대(accessibility §2.3)와 현지화 팽창은 **동일 세로 확장 여백을 공유** — 한 번의 레이아웃 강건성으로 두 요구 만족.

---

## 7. 키별 메타데이터 (TMS/번역 키트)

`accessibility-localization.md` §10.1 — 문맥 없는 짧은 문자열의 오역을 막기 위해 각 키에 메타를 부여한다. 스타터 테이블(`strings.sample.json`)의 필드와 정합:

| 필드 | 내용 |
|---|---|
| `surface` | ux 화면 코드(S0~S21) 또는 표면 |
| `speaker` | 화자(`ria` / `system` / `none`) — 리아 톤 여부 |
| `budget` | §6 예산 키 |
| `vars` | 변수 슬롯 정의(있을 때) |
| `transcreation` | `true`=플레이버 적응 / `false`=기능 직역 |
| `note` | 컨텍스트/뉘앙스 노트(transcreation 원문 의도·금지선) |

용어집(§glossary.md)을 termbase로 연결해 용어 일관성을 CI에서 자동 검사(G3·§5).

---

## 열린 질문

1. **조사 자동화 범위** — 받침 판정 헬퍼를 코어에 둘지, 조사 비의존 문형으로 원문을 통제할지의 비율. 동적 이름(회사·직원)이 조사 앞에 오는 문장의 실제 빈도 측정 필요(§2.4).
2. **아이콘 자리표시자 규격** — 재화 아이콘 주입 방식(`{icon_funds}` 토큰 vs 렌더 계층 프리픽스)을 `technical-architecture.md` 리소스 로더와 확정(§3.1).
3. **키 네임스페이스 세분도** — 카탈로그 `slot`(name/flavor/desc) 외에 난이도(하/중/상)별 플레이버가 필요한지, 티어×난이도 곱집합의 문자열 물량 상한(§4·balance 연동).
4. **transcreation 물량·반복 체감** — 플레이버(성격 디스크립터·프로젝트 플레이버) 적응 번역 물량과 로케일 반복 피로, "조용한 달" 코멘트 생략 규칙의 현지화 정합(narrative 열린질문 4·9).
5. **길이 예산 자동 검출 임계** — G5의 렌더 오버플로우 판정을 최장 로케일×최대 폰트 크기 곱집합에서 어떻게 캘리브레이션할지(accessibility §10.3·열린질문 3·12).
6. **다이아·상비 의뢰·성과 등급 라벨 확정 번역** — 용어집 (가안) 표기(Diamond / Standing Job / Flop·Modest·Success·Hit)의 LQA 사인오프(§glossary.md).
7. **ja/zh-Hans 확장 시 키 불변 검증** — 표면 어휘·변수 슬롯이 확장 로케일 추가로 변경되지 않음을 회귀로 보장(§12 로드맵·데이터 팩 철학).
