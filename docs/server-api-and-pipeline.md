# 서버 API 계약 + 데이터 파이프라인 설계 (Server API & Data Pipeline) v1.0

> **문서 지위.** 본 문서는 `technical-architecture.md`(이하 `[TA]`)가 §1(책임 분담)·§6(서버 백엔드)·§7(콘텐츠 파이프라인)·§10(보안)에서 확정한 **설계**를, 실제 백엔드 착수·벤더 RFP가 가능한 **API 계약(M1.4.4)·데이터 파이프라인 계약(M1.4.3)** 수준으로 구체화한다. 규칙·결정론 전제는 `[TA]`·`game-design.md`·`mechanics-spec.md`가 정본이며 본 문서는 이를 **바꾸지 않는다**(계약·요구만 확정).
>
> **벤더 중립 원칙(D-106).** BaaS/MMP 벤더는 아직 미선정이다(`decision-log.md` D-106, roadmap M1.2.6). 따라서 본 문서는 **벤더 중립 계약** — 엔드포인트·스키마·인증·파이프라인 요구 — 만 확정하고, 특정 벤더 제품을 강제하지 않는다. §11이 이 계약을 만족해야 하는 벤더 평가 포인트를 인계한다.
>
> **인용:** `[TA §x]`=technical-architecture, `[MON §x]`=monetization, `[LO §x]`=liveops-analytics, `[CS §x]`=content-schema, `[CONV §x]`=localization/string-key-convention, `[GD §x]`=game-design, `[MS §x]`=mechanics-spec.
>
> **표기 규약.** **계약:** = 벤더 무관 필수 요구(위반 시 벤더 부적격). **설계 결정:** = 규칙이 정하지 않은 기술 선택의 확정(근거 1줄+). 스키마 타입 표기는 §0.3.

---

## 목차

0. 범위·전제·표기
1. API 설계 원칙 (벤더 중립·최소 권위·멱등·버전)
2. 인증·세션 모델
3. 엔드포인트 카탈로그
4. 공통 규약 (에러·멱등·버전·페이지네이션)
5. 보안 경계 (위조·변조 방어)
6. 데이터 파이프라인 개요
7. 콘텐츠 종류별 배달
8. 버전·핀·마이그레이션
9. 배포·롤백·검증
10. 분석 파이프라인 (수집→적재)
11. 벤더 선정 인계 (D-106 평가 포인트)
12. 열린 질문

---

## 0. 범위·전제·표기

### 0.1 다루는 것 / 다루지 않는 것

| 다룬다 (계약) | 다루지 않는다 (타 문서 정본) |
|---|---|
| 서버 엔드포인트·요청/응답 스키마·인증·에러·멱등(§2~§5) | KPI 정의·이벤트 카탈로그 세부(`[LO §1~2]`) |
| 콘텐츠 배달 경로·포맷·검증·버전 핀(§6~§9) | 밸런스 수치(`balance.md`)·업종 팩 내용(`[CS]`)·문자열 값(`[CONV]`) |
| 분석 ingest 계약·스키마 레지스트리·멱등(§10) | 상품·가격·오퍼 카탈로그·재화 규칙(`[MON]`) |
| 벤더 평가 요구(§11) | 벤더 **선정**(팀 몫, D-106) |

### 0.2 신뢰 경계 전제 (`[TA §1.1]`·§10.1 반영 — 본 계약의 축)

- **시뮬레이션 = 클라이언트 권위.** 자금·골드·명성·XP·컨디션·충성도 등 게임 내 자원은 서버가 **검증하지 않는다**(`[TA §1.1]`). 세이브 페이로드는 서버에 **불투명 blob**이며, 서버는 이를 파싱·판정하지 않는다.
- **서버 권위 = 실화폐·계정 신뢰 표면만.** (a) IAP 영수증·엔타이틀먼트 원장, (b) 다이아(결제 재화) 원장(`[MON §2.1]` 서버 권위 소비성), (c) 클라우드 세이브 소유·버전, (d) 리모트 config·A/B 배정, (e) 분석 수집, (f) 푸시 토큰.
- 따라서 본 API는 **얇은 신뢰 계층**이며 게임 서버가 아니다(`[TA §6.2]`). 무거운 서버 시뮬 재실행은 MVP 계약에 없다(분쟁·어뷰징 샘플 검증은 옵션, `[TA §10.1]`).

### 0.3 스키마 타입 표기

`string` · `string(id)`(로케일 불변 ASCII 식별자) · `i32`/`i64`(정수, 화폐는 원 단위 i64 `[TA §2.6]`) · `bool` · `enum{a|b}` · `epoch_ms`(UTC 밀리초) · `bytes`(전송은 base64 또는 multipart) · `uuid` · `semver` · `l10n_key` · `object` · `array[T]` · `map<K,V>`. `?`=선택 필드.

---

## 1. API 설계 원칙

### 1.1 벤더 중립 (D-106 미선정 전제)

- **계약: 클라이언트는 서버 접점을 내부 인터페이스(port) 뒤에 두고 어댑터에서만 실제 SDK/HTTP를 호출한다**(`[TA §8]` 어댑터 격리 재확인). 본 문서의 엔드포인트는 **논리 계약**이며, 벤더가 매니지드 SDK로 제공하든 서버리스 함수로 제공하든 이 요청/응답 형태(필드·의미·멱등·에러)를 만족하면 된다.
- **전송·인코딩(계약 기본선):** HTTPS/TLS만. 요청/응답 JSON(텔레메트리 배치는 JSON 배열 또는 압축, §10). 인증 `Authorization: Bearer <token>`. 시각은 전부 `epoch_ms` UTC.
- 벤더별 실제 경로·직렬화가 달라도, 어댑터가 본 계약 형태로 정규화하면 상위 게임 코드는 벤더를 모른다 → 이관(락인 완화)이 값싸다.

### 1.2 최소 권위 (`[TA §1.1]`·§10)

- **계약: 서버는 게임 자원 상태를 검증·판정하지 않는다.** 세이브 blob은 불투명. 서버가 권위를 갖는 것은 §0.2의 6개 표면뿐. 이 원칙을 어기는 엔드포인트(예: "자금 잔액 검증", "명성 서버 계산")는 **계약에 없으며 신설 금지**(비-P2W·소규모 팀 안티치트 집중 `[TA §10.1]`).
- 예외 경계 단일 규칙: **실화폐가 얽히거나(결제·엔타이틀먼트) 계정 간 신뢰가 필요한 것(세이브 소유·복구)만 서버 권위**(`[TA §1.1]`).

### 1.3 멱등성

- **계약: 부수효과가 있는 모든 POST는 멱등이다.** 클라이언트는 `Idempotency-Key` 헤더(uuid)를 붙이고, 서버는 (키 → 최초 응답)을 보존 창(권장 ≥72h) 동안 저장해 재시도에 **동일 응답**을 반환한다. 네트워크 재시도·오프라인 큐 플러시·캐치업 재계산 이중 발신(§10.4)이 중복 지급/중복 적재를 만들지 않게 하는 근간.
- **업무 멱등 키(전송 키와 별개, 서버 원장 차원):** IAP=플랫폼 `transaction_id`(§3.3), 진행축 텔레메트리=`milestone_id`/`회사×등급`(§10, `[LO §2.3]`), 다이아 소비=클라 생성 `spend_id`.

### 1.4 버전

- **계약: 경로 프리픽스 `/v1/`로 API 메이저 버전을 고정**하고, 하위호환 깨지는 변경은 `/v2/`로만 낸다. 필드 추가는 하위호환(클라는 미지의 필드 무시).
- 클라이언트는 매 요청에 `app_version`·`save_schema_version`을 실어 서버가 최소 버전 게이트(`min_app_version`, §3.4)와 호환 판정을 하게 한다. **시뮬 밸런스/콘텐츠 버전은 세이브에 핀**되며 API 버전과 독립이다(§8).

---

## 2. 인증·세션 모델

### 2.1 토큰 모델

- **설계 결정: 2토큰 모델 — 단명 세션 토큰(access) + 장수 갱신 토큰(refresh).** 근거: 세션 토큰이 짧으면 유출 피해가 제한되고, 갱신 토큰은 기기 보안 저장소(iOS Keychain/Android Keystore, `[TA §3.4]`)에만 보관해 재로그인 마찰을 없앤다.

| 토큰 | 수명(기본선) | 보관 | 담는 것 | 용도 |
|---|---|---|---|---|
| `session_token` | 짧음(예: 1h) | 메모리 | `account_id`·`device_id`·scope·exp | 전 엔드포인트 `Bearer` 인증 |
| `refresh_token` | 김(예: 60d, 회전) | 기기 보안 저장소 | 계정 바인딩 | 세션 토큰 재발급(`/v1/auth/refresh`) |

- **계약: refresh 토큰은 회전(rotating)한다** — 사용 시 새 토큰 발급·구 토큰 폐기(재사용 탐지 시 세션 무효화). 로그아웃·계정 삭제·이상 탐지 시 서버가 폐기 가능.

### 2.2 계정 연동 (게스트 → 소셜)

`[TA §3.5]` 계정 모델을 계약화한다.

1. **게스트 생성:** 최초 실행 시 기기 로컬 익명 계정 발급(PII 0). 게스트도 클라우드 세이브·엔타이틀먼트 원장을 가진다(기기 유실 전까지).
2. **연동(승격):** 게스트 → Sign in with Apple / Google Play Games / 커스텀. 서버가 세이브·엔타이틀먼트를 소셜 계정에 **귀속**(`[TA §3.5]`). 스토어 정책상 Apple/Google 로그인 제공 필수(`[TA §6.1]`).
3. **다기기 로그인:** 소셜 로그인으로 다른 기기에서 서버 스냅샷 pull. "쓰기 토큰"(§3.2 `write_token`)으로 마지막 활성 기기 표시(`[TA §3.5]`).
4. **연동 충돌:** 연동하려는 소셜 계정에 **이미 다른 클라우드 세이브가 존재**하면 자동 병합하지 않는다(단일 세이브 게임, 병합 위험 `[TA §3.5]`). 서버는 양쪽 메타를 반환하고 클라가 사용자 선택 프롬프트(로컬 유지 vs 서버 채택)를 띄운다. → 열린 질문 3.

### 2.3 보안 경계 (요약, 상세 §5)

- **영수증 위조·결제 우회 → 서버 영수증 검증(§3.3)**: 클라 신뢰 0.
- **세이브 변조 → 로컬 HMAC(탐지용) + 유료 재화는 세이브가 아닌 서버 원장 권위**: 위조가 무익(`[TA §3.4]`·§10.1).
- **시계 조작 → 서버 시각 권위 + 반되감기 + 상한**(§5.3, `[TA §4.4]`).

---

## 3. 엔드포인트 카탈로그

각 엔드포인트: 목적 · 메서드/경로 · 인증 · 멱등 · 요청/응답 스키마 · 주요 에러. 공통 에러 모델은 §4.1.

### 3.1 계정/인증

#### `POST /v1/auth/guest` — 게스트 계정 생성
인증: 없음 · 멱등: `Idempotency-Key`(기기 최초 1회)

| 요청 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `device_pseudo_id` | string | ✔ | 비식별 기기 키(PII 아님, `[LO §2.2]`) |
| `platform` | enum{ios\|android} | ✔ | |
| `app_version` | semver | ✔ | |

| 응답 | 타입 | 설명 |
|---|---|---|
| `account_id` | string(id) | 익명 계정 ID |
| `session_token` / `refresh_token` | string | §2.1 |
| `expires_in` | i32 | 세션 토큰 잔여 초 |
| `is_new` | bool | 신규 생성 여부 |

#### `POST /v1/auth/refresh` — 세션 재발급
인증: refresh 토큰 · 멱등: 불필요(자연 멱등)
- 요청 `{ refresh_token }` → 응답 `{ session_token, refresh_token(회전), expires_in }`. 에러: `token_expired`·`token_reused`(→ 세션 전면 무효).

#### `POST /v1/auth/link` — 게스트→소셜 연동
인증: `Bearer`(게스트 세션) · 멱등: `Idempotency-Key`

| 요청 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `provider` | enum{apple\|google\|custom} | ✔ | |
| `provider_token` | string | ✔ | 플랫폼 OAuth/ID 토큰(서버가 검증) |

| 응답 | 타입 | 설명 |
|---|---|---|
| `account_id` | string(id) | 승격 후 계정 |
| `linked_providers` | array[enum] | 연동된 provider 목록 |
| `save_conflict` | bool | 소셜 계정에 기존 세이브 존재(§2.2-4) |
| `conflict_meta` | object? | 충돌 시 양쪽 세이브 메타(§3.2 metadata 형태) |

#### `POST /v1/auth/login` — 소셜 로그인(기존/신기기)
인증: 없음 → provider 토큰 · 응답 `{ account_id, session_token, refresh_token, has_cloud_save }`.

#### `POST /v1/account/delete` — 삭제 요청 (PIPA/GDPR, `[TA §6.3]`)
인증: `Bearer` · 멱등: `Idempotency-Key`
- 응답 `{ deletion_ticket_id, status: enum{queued|completed}, purge_targets: array[enum{cloud_save|entitlement_pii|analytics_pseudo|attribution}] }`. 서버는 세이브·분석 PII·어트리뷰션 식별자를 삭제 경로로 태우고 감사 로그를 남긴다(§5.4).

### 3.2 클라우드 세이브

세이브 페이로드는 **서버에 불투명**(§0.2). 서버는 blob + 메타 + 무결성만 보관하고 게임 상태를 파싱하지 않는다.

#### `GET /v1/save/metadata` — 서버 세이브 메타 조회
인증: `Bearer`

| 응답 | 타입 | 설명 |
|---|---|---|
| `exists` | bool | 서버 세이브 존재 여부 |
| `save_version` | i64 | 단조 증가 카운터(`[TA §3.5]`) |
| `game_day` | i64 | 진행 게임일(충돌 판정용, `[MS]` day) |
| `save_schema_version` | i32 | 저장 구조 버전(`[TA §3.3]`) |
| `balance_content_version` | string | 세이브에 핀된 시뮬 밸런스/카탈로그 버전(§8) |
| `app_build` | string | 진단용 |
| `server_updated_at` | epoch_ms | 서버 수신 시각(권위 시각) |
| `size_bytes` | i64 | |
| `checksum` | string | 페이로드 SHA-256(무결성) |
| `write_token` | string | 마지막 활성 기기 표식(§2.2-3) |

#### `POST /v1/save/upload` — 스냅샷 업로드
인증: `Bearer` · 멱등: `Idempotency-Key`

| 요청 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `base_save_version` | i64 | ✔ | 클라가 마지막으로 본 서버 버전(충돌 감지 기준) |
| `game_day` | i64 | ✔ | |
| `save_schema_version` | i32 | ✔ | |
| `balance_content_version` | string | ✔ | 핀 버전(§8) |
| `app_build` | string | ✔ | |
| `blob` | bytes | ✔ | 불투명 세이브(이진·압축, `[TA §2.4·§3.6]`) |
| `checksum` | string | ✔ | blob SHA-256 |
| `write_token` | string? | | 직전 발급 토큰(다기기 클로버 방지) |

| 응답(성공 200) | 타입 | 설명 |
|---|---|---|
| `accepted` | bool | |
| `new_save_version` | i64 | 증가된 서버 버전 |
| `server_updated_at` | epoch_ms | 권위 시각(시계 재조정용 §5.3) |
| `write_token` | string | 새 활성 기기 토큰 |

- **충돌(409 `version_conflict`):** `base_save_version`이 서버 최신과 다르거나 `write_token`이 만료면 서버가 저장하지 않고 `conflict_meta`(현재 서버 메타)를 반환. **해소 규칙(`[TA §3.5]`): 더 진행된(day/version 큰) 쪽 우선 + 사용자 확인.** 자동 병합 없음.

#### `GET /v1/save/download` — 스냅샷 다운로드
인증: `Bearer` · 응답 `{ save_version, game_day, save_schema_version, balance_content_version, blob, checksum, server_updated_at }`.
- 클라가 `save_schema_version`이 자기 지원 상한보다 크면 **앱 업데이트 유도**(구버전은 상위 스키마 다운마이그레이션 불가, `[TA §3.3]` 단조 체인).

#### `POST /v1/save/resolve-conflict` — 충돌 해소 확정
인증: `Bearer` · 멱등: `Idempotency-Key` · 요청 `{ resolution: enum{keep_local|keep_server}, base_save_version }` → 응답 `{ save_version }`. 사용자 선택을 서버에 확정(로컬 채택 시 강제 업로드 승격).

### 3.3 IAP 영수증 검증·엔타이틀먼트 (`[TA §6.1]`·§10.1, `[MON §8.2]`)

매출 직결 — **서버 권위, 클라 신뢰 0**. 다이아·비소비성·구독 부여는 서버가 확정.

#### `POST /v1/iap/validate` — 영수증 검증·지급
인증: `Bearer` · 멱등: `Idempotency-Key`(권장=`transaction_id`) — 업무 멱등키는 서버 원장의 `transaction_id` 유니크 제약

| 요청 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `platform` | enum{apple\|google} | ✔ | |
| `product_id` | string(id) | ✔ | SKU(`[MON §10]`, 예 `dia_t3`·`sub_supporter_monthly`) |
| `transaction_id` | string | ✔ | 플랫폼 거래 ID(중복 방지 원장 키) |
| `receipt` | bytes | ✔(apple) | StoreKit 영수증/JWS |
| `purchase_token` | string | ✔(google) | Play Billing 토큰 |
| `is_subscription` | bool | ✔ | 구독 여부(복원·갱신 처리 분기) |

| 응답 | 타입 | 설명 |
|---|---|---|
| `txn_status` | enum{granted\|already_granted\|invalid\|refunded\|pending} | already_granted=멱등 재요청(`[LO §2.4.6]` 중복) |
| `granted` | object? | 지급 내역(아래) |
| `entitlement_version` | i64 | 원장 버전(클라 캐시 무효화) |

`granted`: `{ diamonds?: i64, non_consumables?: array[string(id)], subscription?: { sku, expires_at: epoch_ms, auto_renew: bool } }`.
- 에러: `receipt_invalid`(422)·`receipt_duplicate`(중복 거래는 `already_granted`로 성공 반환, 재지급 없음). **흐름(`[TA §6.1]`): 클라 결제 → 영수증 서버 전송 → 스토어 서버 검증 API 대조 → 엔타이틀먼트 원장 기록·중복 방지 → 지급 승인. 클라는 서버 승인 없이 유료 재화 확정 금지.**

#### `GET /v1/entitlements` — 엔타이틀먼트/다이아 원장 조회
인증: `Bearer`

| 응답 | 타입 | 설명 |
|---|---|---|
| `diamond_balance` | i64 | 서버 권위 다이아 잔액(`[MON §2.1]` 소비성) |
| `non_consumables` | array[object] | `{ sku, granted_at }`(코스메틱·무광고, 복원 대상) |
| `subscriptions` | array[object] | `{ sku, status: enum{active\|grace\|expired}, expires_at, auto_renew }` |
| `entitlement_version` | i64 | 원장 버전 |

#### `POST /v1/iap/restore` — 복원 (비소비성·구독, 플랫폼 요구 `[MON §8.2]`)
인증: `Bearer` · 멱등: 자연 멱등 · 요청 `{ platform, receipt|purchase_token }` → 응답 `{ non_consumables[], subscriptions[] }`. 소비성(다이아)은 복원 대상 아님.

#### `POST /v1/diamond/spend` — 다이아 소비 확정 (서버 원장 차감)
인증: `Bearer` · 멱등: `Idempotency-Key`(=`spend_id`)
- 다이아는 서버 권위 원장(`[MON §2.1]`)이므로 소비도 서버 확정. 소비처는 `[MON §2.4]` 4종으로 닫힘.

| 요청 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `spend_id` | uuid | ✔ | 클라 생성 멱등키 |
| `sink` | enum{cosmetic\|skip\|tier_skip\|refresh_pool} | ✔ | S-1~S-4(`[MON §2.4]`) |
| `amount` | i64 | ✔ | 차감 다이아 |
| `sku` | string(id)? | | 코스메틱 등 대상 SKU |

- 응답 `{ new_balance, granted?: { non_consumables?[] }, entitlement_version }`. 에러: `insufficient_balance`(409). **주의:** 차감이 성공하면 게임 내 효과(사이클 진행·풀 갱신)는 **클라 결정론 코어**가 수행하며(서버는 자원 판정 안 함, §0.2·§1.2), 서버는 다이아 원장만 차감한다. 시간 단축권의 결정론 결과 불변은 클라 시드 고정으로 보장(`[MON §8.2]`).

#### 스토어 서버 알림 (S2S webhook — 계약 요구)
- **계약: 환불·구독 갱신/만료는 스토어 서버 통지(Apple App Store Server Notifications / Google RTDN)를 수신해 원장을 갱신**한다(클라 폴링만으로 부족). 클라 대면 엔드포인트 아님(벤더/서버 함수 내부), 그러나 벤더는 이 수신·검증·원장 반영을 제공해야 한다(§11). 환불 시 `iap_refund` 텔레메트리(`[LO §2.4.6]`)·미사용 잔액 처리(`[MON §9.2]`).

### 3.4 리모트 config fetch (`[TA §7.2~§7.4]`, `[LO §6.3]`)

앱 기동 시 config·매니페스트를 fetch → 캐시 → 오프라인 폴백(마지막 정상본, `[TA §7.2]`). 무거운 데이터(밸런스·팩·문자열)는 **매니페스트(포인터+버전+체크섬)**만 여기서 받고 실제 바이트는 CDN에서 받는다(§7).

#### `GET /v1/config` — 리모트 config·매니페스트
인증: `Bearer`

| 요청(쿼리/헤더) | 타입 | 설명 |
|---|---|---|
| `app_version` | semver | 최소버전 게이트 판정 |
| `platform`·`device_tier` | enum | 기기 대역(`[TA §9.1]`) |
| `locale`·`region` | string | 문자열팩·지역 config 선택 |
| `pseudo_player_id` | string | A/B 안정 배정 키(`[TA §7.4]`) |
| `pinned_balance_version` | string? | 진행 중 세이브의 핀 버전(있으면 해당 밸런스 매니페스트 반환, §8) |
| `known_config_version` | string? | 변경 없으면 304 |

| 응답 | 타입 | 설명 |
|---|---|---|
| `config_version` | string | 리모트 config 버전 |
| `min_app_version` | semver | 이하 클라는 강제 업데이트 |
| `server_now` | epoch_ms | 권위 시각(시계 대조 §5.3) |
| `ab_assignments` | map<string,string> | 실험ID→그룹(모든 텔레메트리에 부착, `[LO §2.2]`) |
| `remote_config` | object | 비시뮬 config(아래) |
| `balance_manifest` | object | `{ balance_version, url, checksum, is_pinned }`(§8) |
| `content_manifest` | array[object] | 팩별 `{ pack_id, pack_version, url, checksum }`(`[CS §2]`) |
| `string_manifest` | object | `{ locale, version, url, checksum }`(`[CONV §4]`) |
| `ttl` | i32 | 캐시 유효 초 |

- `remote_config`(시뮬 무관 — 진행 중에도 자유 갱신 `[TA §7.3]`): `offers`·`event_calendar`·`season_schedule`·`login_calendar`·`crm_campaigns`·`feature_flags`·`ad_placements`. 구성·윤리 규칙 정본은 `[MON]`, 스케줄·타깃팅은 `[LO §6.3]`.
- **계약: 시뮬 입력(밸런스·카탈로그·해석 파라미터)과 비시뮬(오퍼·UI·이벤트)을 응답에서 분리**한다. 전자는 매니페스트+버전 핀 경유(§8), 후자는 즉시 적용. 실행 코드는 **원격 배포하지 않는다**(iOS 정책, `[TA §7.2]` — 데이터만).

### 3.5 텔레메트리 수집 (배치) — 계약 요약, 상세 §10

#### `POST /v1/telemetry/batch` — 이벤트 배치 ingest
인증: `Bearer` · 멱등: `Idempotency-Key`(=`batch_id`)

| 요청 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `batch_id` | uuid | ✔ | 배치 멱등키 |
| `sent_at` | epoch_ms | ✔ | 클라 발신 시각 |
| `envelope` | object | ✔ | 배치 공통 봉투(§10.1, `[LO §2.2]`) |
| `events` | array[object] | ✔ | 이벤트(각 `event_id`·`event_name`·`event_ts_client`·`game_day`·`is_offline_catchup`·`params`·`business_key?`) |

- 응답 `{ accepted_count, rejected: array[{ event_id, reason: enum{schema_violation|duplicate|malformed} }] }`. **at-least-once + 멱등 dedup**(§10.3): 서버는 `event_id`(및 업무 멱등키)로 중복 제거하고, 스키마 위반은 **드롭이 아니라 격리·플래그**(§10.2). 상세 §10.

### 3.6 푸시 토큰 등록 (`[TA §6.1]`, 옵트인 `[TA §6.3]`)

#### `POST /v1/push/register` — 토큰 등록/갱신
인증: `Bearer` · 멱등: `Idempotency-Key`

| 요청 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `push_provider` | enum{fcm\|apns} | ✔ | |
| `device_token` | string | ✔ | 푸시 토큰 |
| `consent` | bool | ✔ | 동의(옵트인, 미동의면 발송 안 함) |
| `locale`·`region` | string | | 현지화·조용한 시간대 |
| `quiet_hours` | object? | | `{ start_hour, end_hour, tz }`(`[LO §8.1]` 피로도) |

- 응답 `{ registered, subscription_id }`. `DELETE /v1/push/register`(옵트아웃/토큰 폐기). 세그먼트·캠페인 타깃팅은 서버/`[LO §8]` 소관, 계약은 등록·동의·조용한 시간대만.

---

## 4. 공통 규약

### 4.1 에러 모델

- **계약: 모든 에러는 `{ error: { code, message, retriable: bool, details? } }` + HTTP 상태.** `code`는 아래 닫힌 enum. `message`는 진단용(사용자 노출 문자열은 클라가 `[CONV] error.*` 키로 현지화, 서버 문자열 직노출 금지).

| code | HTTP | retriable | 의미 |
|---|---|---|---|
| `unauthenticated` | 401 | 아니오 | 토큰 없음/무효 |
| `token_expired` | 401 | 예(refresh 후) | 세션 만료 |
| `permission_denied` | 403 | 아니오 | 계정 불일치 |
| `invalid_argument` | 400 | 아니오 | 스키마 위반 |
| `failed_precondition` | 412 | 아니오 | 선행조건 불충족(버전 게이트 등) |
| `version_conflict` | 409 | 예(해소 후) | 세이브/원장 버전 충돌 |
| `receipt_invalid` | 422 | 아니오 | 영수증 검증 실패 |
| `insufficient_balance` | 409 | 아니오 | 다이아 부족 |
| `rate_limited` | 429 | 예(백오프) | 레이트 초과 |
| `unavailable` / `internal` | 503/500 | 예(백오프) | 서버 일시/내부 오류 |

- **계약: 재시도 정책은 지수 백오프 + 지터**, `retriable=false`는 재시도 금지. `rate_limited`는 `Retry-After` 헤더.

### 4.2 멱등·정합

- §1.3 멱등 계약을 전 mutating POST에 적용. 서버는 멱등키 창 내 재요청에 최초 응답 재생. **오프라인 큐:** 클라는 오프라인 중 발생한 mutating 요청(텔레메트리·다이아 소비·세이브)을 로컬 큐에 담고 온라인 시 멱등키와 함께 플러시 → 중복 안전.

### 4.3 페이지네이션·버전 헤더

- 목록형(엔타이틀먼트 등 소량이라 MVP 미필요)에 필요 시 커서 페이지네이션 `{ items[], next_cursor? }`. 리소스 응답은 `entitlement_version`/`save_version`/`config_version` 같은 **단조 버전**을 실어 클라 캐시 무효화·낙관적 동시성을 지원.

---

## 5. 보안 경계 (위조·변조 방어)

`[TA §10.1]` 위협 모델을 API 계약으로 고정. 방어의 무게는 **실손해 표면(결제·클라우드)**에 집중.

### 5.1 영수증 위조·결제 우회 (강)
- `POST /v1/iap/validate`가 **스토어 서버 검증 API 대조 + 엔타이틀먼트 원장 + `transaction_id` 유니크**로 방어(§3.3). 클라 신뢰 0. S2S 알림으로 환불·취소를 원장에 반영. 세이브 위조로는 유료 재화를 얻을 수 없다(원장 권위 분리).

### 5.2 세이브 변조 (경~중)
- 로컬 세이브는 HMAC-SHA256 서명(탐지용, `[TA §3.4]`), 클라우드 세이브 최종 권위는 서버. 서버는 blob 불투명이므로 **자원 수치를 검증하지 않는다**(비-P2W — 자기 세션 조작은 남을 해치지 않음). 서버가 지키는 것은 blob 무결성(`checksum`)·버전·소유뿐.

### 5.3 시계 조작 (중) — `[TA §4.4]`
- **계약: 방치 경과 시간의 권위 시각은 서버.** `/v1/config.server_now` 또는 `/v1/save/upload.server_updated_at`이 권위 시각을 공급. 오프라인은 클라가 단조 시계 + 반되감기 워터마크로 방어하고, 온라인 복귀 시 서버 시각으로 재조정·상한(`DAY_OFFLINE_CAP`) 재적용. 서버 시각 괴리는 텔레메트리(`event_ts_client` vs `event_ts_server`, `[LO §2.2]`)로 어뷰징 탐지.

### 5.4 프라이버시 경계 (`[TA §6.3]`, PIPA/GDPR)
- **계약: 텔레메트리·게임 API는 비식별 `pseudo_player_id`만 사용**(PII·실명·이메일·정밀위치·자유텍스트 금지, `[LO §2.2]`). 어트리뷰션(MMP)만 동의(ATT/GDPR) 하에 기기 식별자를 **별도 파이프**로 처리, 게임 텔레메트리와는 가명 키로만 조인. `POST /v1/account/delete`가 세이브·분석 PII·어트리뷰션 식별자 삭제 경로를 보증(§3.1). 리전 데이터 레지던시는 벤더 평가(§11).

---

## 6. 데이터 파이프라인 개요

### 6.1 원칙 — 데이터/코드 분리 (`[TA §7.1]`)

- **계약: 코드 배포(스토어 재심사) 없이 갱신 가능해야 하는 4종 + 리모트 config를 전부 코어 밖 데이터로 격리**하고, 코어는 이를 파라미터/테이블로 주입받는다(`[TA §7.1]`, `[GD §12.1]`). 실행 코드는 원격 배포하지 않는다(iOS 정책).

### 6.2 배달 대상 5종 (소유·형태·경로)

| 콘텐츠 | 소유 문서 | 포맷 | 배달 경로 | 시뮬 입력? |
|---|---|---|---|---|
| 밸런스 값 | `balance.md` | 버전 태그 테이블 | CDN 아티팩트(매니페스트 §3.4) | **예** → 버전 핀(§8) |
| 업종 카탈로그(팩) | `[CS]`/`[GD §12]` | JSON 팩(`schema_version`·`pack_version`) | 콘텐츠 팩 CDN(addressable) | **예** → 버전 핀(§8) |
| 현지화 문자열 | `[CONV]`/accessibility | 키 기반 문자열 테이블(ICU) | 언어팩 CDN | 아니오(표시 전용) |
| 리모트 config·오퍼 | `[LO]`/`[MON]` | 서버 config(JSON) | `/v1/config`(§3.4) | 아니오 → 즉시 갱신 |
| A/B 설정 | `[LO §5]`/`[TA §7.4]` | 배정 규칙+config | `/v1/config.ab_assignments` | 분리(§8.3) |

- **핵심 분기(계약): 시뮬 입력(밸런스·카탈로그)은 세이브 핀 + 마이그레이션 경계(§8), 비시뮬(문자열·오퍼·config·비시뮬 A/B)은 즉시 갱신.** 이 경계가 결정론(`[MS §1.1]`)과 라이브옵스 민첩성을 동시에 지킨다.

---

## 7. 콘텐츠 종류별 배달

각 종류의 **경로·포맷·검증**을 코드 배포 없이 갱신하는 계약으로 확정한다.

### 7.1 밸런스 값
- **경로:** `/v1/config`의 `balance_manifest` → 버전 URL의 **불변 아티팩트**를 CDN에서 fetch → 검증(체크섬)→ 캐시. 진행 중 세이브는 자기 핀 버전(`pinned_balance_version`)의 매니페스트를 받는다(§8).
- **포맷:** 버전 태그(`balance_version`) 붙은 테이블(전역 상수 + 커브 계수). 정수/고정소수 규약 준수(`[TA §2.6]`).
- **검증:** 스키마 유효성 + 가드레일 사전 검사(§9). 아티팩트는 **버전당 불변**(과거 버전 영구 보존 — 핀·리플레이 재현 전제, §8).

### 7.2 업종 카탈로그 (콘텐츠 팩)
- **경로:** `content_manifest`의 팩별 `{pack_id, pack_version, url, checksum}` → 콘텐츠 팩 CDN(addressable/번들, `[TA §7.2]`) 온디맨드 로드·언로드(`[TA §9.2]`). 세이브에 미포함(팩 ID+버전 참조).
- **포맷:** `[CS §2]` JSON 팩 — `schema_version`(로더 호환)·`pack_version`(핀 대상)·`industry_def`·`dept_types[3]`·`catalog[]`·`interpretation_params`·`economy`. **닫힌 enum + 파라미터 참조만**(임의 코드 없음, `[CS §1-3]`, `[TA §2.5]`).
- **검증:** `[CS §13]` `TODO:balance` 토큰을 로더가 **하드 거부**(미완성 팩 게이트), `catalog_id` 정렬 가능성, 부서 3종·상비 존재 계약(`[CS §1-5·6]`). CI 콘텐츠 린트(§9, `[TA §11]`).

### 7.3 문자열 (현지화)
- **경로:** `string_manifest`의 `{locale, version, url}` → 언어팩 CDN. 세이브에 미포함(l10n 키 참조).
- **포맷:** `<surface>.<context>.<identifier>` 키 규약(`[CONV §1]`) + ICU MessageFormat(`[CONV §2]`). 키는 로케일 불변, 값만 로케일별.
- **검증:** 빌드/배포 게이트 G1~G6(`[CONV §5]`) — 하드코딩 검출·키 규약·미번역 키(ko/en 0 필수)·ICU 문법·길이 예산·글리프 커버리지. 신규 업종 문자열은 팩 확장으로 닫힘(`[CONV §4.2]`).

### 7.4 리모트 config·오퍼
- **경로:** `/v1/config.remote_config`(§3.4) — 서버 config, 시뮬 무관이라 진행 중 즉시 적용(`[TA §7.3]`, `[LO §6.3]`). 오프라인 폴백(마지막 정상본).
- **포맷:** 오퍼 구성·이벤트 캘린더·시즌 스케줄·CRM·피처 플래그·광고 배치. 구성·윤리 정본 `[MON]`, 스케줄·타깃팅 `[LO]`.
- **검증:** config 스키마 검사 + 윤리 가드레일(거짓 희소성·FOMO 금지, `[MON §6.2]`) 배포 전 린트. 실행 코드 없음.

### 7.5 A/B 설정
- **경로:** `/v1/config.ab_assignments`(서버 안정 배정, pseudo-id 기반, `[TA §7.4]`). 배정은 세이브에도 기록(핀과 함께).
- **분리(계약, §8.3):** 비시뮬 A/B(UI·오퍼·가격·튜토리얼)=자유; 시뮬 A/B(밸런스 값)=배정 밸런스 버전을 세이브에 핀, **신규 코호트만**.

---

## 8. 버전·핀·마이그레이션

### 8.1 3종 버전 분리 (`[TA §3.3]`)
- **계약:** 세이브 헤더는 3종 버전을 **독립적으로** 담는다 — (1) `save_schema_version`(저장 구조, 마이그레이션 대상), (2) `balance_content_version`(시뮬 재현 핀), (3) `app_build`(진단). 혼동 금지: 구조 마이그레이션과 밸런스 핀은 별개 축.

### 8.2 밸런스/콘텐츠 버전 핀 (결정론 정합 — 핵심, `[TA §7.3]`)
- **계약: 밸런스·콘텐츠 버전을 세이브에 핀한다. 진행 중 세이브는 생성 시점(또는 명시적 마이그레이션 시점)의 버전으로 계속 시뮬레이션하고, 원격 갱신은 신규 게임 또는 명시적 마이그레이션 경계에만 적용한다.** 근거: 밸런스/콘텐츠 값은 `[MS §7]` 결정론 함수의 입력이므로, 원격으로 몰래 바뀌면 "같은 세이브+같은 로그→같은 상태"(`[MS §1.1]`)가 깨지고 진행 중 회사가 급변한다.
- **아티팩트 계약:** 밸런스/팩 아티팩트는 **버전당 불변·영구 보존**. `/v1/config`은 핀 버전을 존중해 해당 버전 매니페스트를 반환(§7.1). 신규 코호트만 최신 버전을 핀.

### 8.3 신규 코호트 vs 마이그레이션 경계
- **신규 코호트:** 새 게임/새 세이브는 배포된 최신(또는 A/B 배정) 밸런스 버전을 핀. 즉시 반영, 결정론 무해.
- **마이그레이션 경계(진행 중 세이브 변경):** 밸런스/콘텐츠를 진행 중 세이브에 반영해야 하면(심각 exploit·데드락 핫픽스, `[LO §9.1]`) **명시적 마이그레이션 이벤트로 승격** — 플레이어 고지 + 콘텐츠 버전 증가 + 마이그레이션 경계를 리플레이 로그에 기록(`[TA §3.3]`)해 재현성을 구간별 유지. 경미 튜닝은 이 경로를 쓰지 않고 신규 코호트로만. 긴급 예외 절차는 D-111(미결).
- **`save_schema_version` 마이그레이션:** 클라이언트 측 N→N+1 단조 변환 체인(`[TA §3.3]`). 서버는 blob 불투명이라 마이그레이션에 관여하지 않음(구조 승급은 클라가 다운로드 후 수행).

### 8.4 A/B 코호트 격리
- **계약: 시뮬 A/B는 신규 코호트에만.** 그룹 간 밸런스 버전이 다르며 각자 세이브에 핀(진행 중 세이브 중간 변경 금지, `[TA §7.4]`, `[LO §5.2]`). 배정은 pseudo-id 안정(세션 간 동일 그룹). 코호트 태그(`ab_assignments`·`balance_content_version`)는 모든 텔레메트리에 부착돼 `[LO]`가 KPI를 그룹별 격리 판정.

---

## 9. 배포·롤백·검증

### 9.1 스키마 유효성 검사 (배포 게이트)
- **계약: 모든 데이터 아티팩트는 배포 전 스키마 검증을 통과해야 한다.** 밸런스=테이블 스키마+가드레일 사전 검사(`[LO §3.1]` 마진율>0 등 헤드리스 재검증 `[LO §11.2]`); 팩=`[CS]` 스키마+`TODO:balance` 토큰 거부; 문자열=G1~G6(`[CONV §5]`); config=오퍼 스키마+윤리 린트. CI 게이트(`[TA §11]`)에 편입.

### 9.2 스테이징·점진 롤아웃
- **계약: 아티팩트/설정은 스테이징 환경에서 검증 후 프로덕션에 점진 롤아웃**(퍼센트 롤아웃)한다. `remote_config`은 A/B 배정 비율로 점진 노출(`[LO §6.3]`). 밸런스 버전은 신규 코호트 A/B(§8.3)로 먼저 검증(`[LO §5.2·9.1]`).

### 9.3 롤백
- **계약: 롤백은 config 포인터를 직전 불변 버전으로 되돌려 즉시 수행**(아티팩트가 버전당 불변이라 복원이 안전). 리모트 config·오퍼·문자열·비시뮬 A/B는 즉시 롤백 가능. 밸런스는 신규 코호트만 영향이므로 포인터 복원으로 신규 세이브부터 회복(진행 중 세이브는 핀 유지라 애초에 영향 없음). 가드레일 경보(`[LO §4.4]`) 초과 실험은 자동 롤백(`[LO §5.4]`).

### 9.4 무결성
- **계약: 매니페스트가 실린 각 아티팩트에 SHA-256 체크섬을 두고 클라가 다운로드 후 검증**(변조·부분 다운로드 거부). config fetch 성공률·무결성 실패는 라이브 헬스 지표(`[LO §11.1]` config fetch ≥99%). 세이브 blob도 `checksum`(§3.2).

---

## 10. 분석 파이프라인 (수집→적재)

`[LO §2.1]`이 요구한 인프라 계약을 확정한다. 경로: **코어 이벤트 버스(`[TA §5.3]`) → 분석 어댑터 tap(`[TA §8]`) → 배치 업로드(`/v1/telemetry/batch`) → ingest → 웨어하우스 → KPI/대시보드(`[LO §4]`)**.

### 10.1 이벤트 봉투 (배치 공통, `[LO §2.2]`)
- **계약: 배치 봉투에 전 이벤트 공통 파라미터를 1회 싣는다** — `pseudo_player_id`·`session_id`·`app_version`·`save_schema_version`·`balance_content_version`·`platform`·`os_version`·`device_tier`·`locale`·`region`·`ab_assignments`·`install_source`. 개별 이벤트는 `event_id`(uuid)·`event_name`(snake_case)·`event_ts_client`·`event_ts_server`(서버 수신 시각 스탬프)·`game_day`·`is_offline_catchup`·`params`. **PII 배제**(§5.4): 이름 문자열·자유텍스트 금지, 엔티티 ID·등급·버킷 값만(`[LO §2.2]`).

### 10.2 스키마 레지스트리·버전 관리 (신규 요구, `[LO §2.1-1]`)
- **계약: 이벤트/파라미터는 스키마 레지스트리에 등록되고 버전·하위호환 계약 검사를 받는다.** ingest 시 등록 스키마로 검증하되, **위반 이벤트는 드롭이 아니라 격리(quarantine)·플래그**(원본 보존 + `rejected` 반환, §3.5) — 유실 없이 스키마 위반율을 라이브 헬스로 감시(`[LO §11.1]` 스키마 위반≈0). 신규 이벤트/파라미터 추가는 하위호환(필드 추가만, 기존 의미 불변).

### 10.3 멱등 ingest (신규 요구, `[LO §2.1-2]`)
- **계약: at-least-once 전달 + 멱등 키로 중복 제거.** 배치 멱등키(`batch_id`)로 배치 재전송을 흡수하고, **업무 멱등키**로 개별 이벤트를 dedup — 결제 `iap_*`=`transaction_id`, 진행축 `progression_*`·`org_promote`·`progression_complete`=`milestone_id`/`회사×등급`(`[LO §2.3]`). 유실율<1%·중복율 감시(`[LO §11.1]`).

### 10.4 캐치업 이중 발신 대응 (`[TA §4]`·`[LO §2.5]`)
- **문제:** 오프라인 캐치업(`[TA §4.1]`)이 결정론 재계산이므로, 세션 중단 후 재부팅하면 **같은 진행축 이벤트가 다시 발화**할 수 있다(같은 seed+day → 같은 결과).
- **계약(2중 방어):** (1) 캐치업 반복 사이클 이벤트는 개별 발신 금지 — `catchup_summary` 1건으로 집계(`[LO §2.5]`); (2) 유실 불가한 진행축·희소 이벤트(승급·해금·마일스톤·완주·파산·흥행 롤)만 개별 발신하되 `is_offline_catchup=true` + **업무 멱등키**를 부착해 ingest가 재발화분을 dedup(§10.3). 이로써 저사양 부팅 예산(`[TA §9]`)과 퍼널 무결성을 동시 보존.

### 10.5 적재·조인
- 웨어하우스 적재 스키마는 봉투+params 평탄화. 코호트 키(설치일·`balance_content_version`·`ab_assignments`·`install_source`·첫 업종)로 조인(`[LO §4.2]`). 어트리뷰션은 가명 키로만 조인(§5.4). 크래시 리포트에 `master_seed`+`game_day`+세이브 버전 부착(`[TA §8·§11]`, `[LO §2.1-4]`)해 결정론 재현.

---

## 11. 벤더 선정 인계 (D-106 평가 포인트)

본 계약을 만족하는 BaaS/MMP 후보를 평가하기 위한 **요구 체크리스트**다. 선정은 팀 몫(D-106, roadmap M1.2.6). `[TA §6.2]`가 후보 부류(통합형 BaaS: Firebase/Amplify/Supabase, 게임 특화 BaaS: PlayFab/Nakama, 자체 서버)를 제시했고, 본 문서는 그 후보가 **반드시 충족해야 하는 계약**을 제시한다.

| 계약 영역 | 벤더 평가 질문 (충족 여부) |
|---|---|
| **인증(§2·§3.1)** | 게스트→소셜 연동, 2토큰·회전, Sign in with Apple/Google, 연동 충돌 처리 API를 제공/확장 가능한가 |
| **클라우드 세이브(§3.2)** | 불투명 blob 저장 + 단조 `save_version` + 충돌 감지(409)·write_token, 계정 귀속, 수백KB blob·리전 |
| **IAP(§3.3)** | Apple/Google 서버 영수증 검증, 엔타이틀먼트 원장·`transaction_id` 유니크, **S2S 알림(환불·갱신)** 수신, 다이아 서버 원장 |
| **리모트 config(§3.4)** | 버전드 config + pseudo-id 안정 A/B 배정 + 점진 롤아웃, **시뮬/비시뮬 분리·버전 핀 매니페스트**(§8) 표현 가능 |
| **콘텐츠 CDN(§7)** | 버전당 불변 아티팩트(밸런스·팩·문자열) 영구 보존·체크섬·온디맨드 |
| **분석(§10)** | ingest 처리량, **멱등 키 dedup + 스키마 레지스트리/버전**, 웨어하우스 export, PII 통제 |
| **푸시(§3.6)** | FCM/APNs, 옵트인·조용한 시간대·세그먼트 |
| **MMP(어트리뷰션)** | Adjust/AppsFlyer/SKAdNetwork, ATT/GDPR 동의 후 활성, 게임 텔레메트리와 가명 키 조인 |
| **횡단** | KR+글로벌 리전·PIPA/GDPR 데이터 레지던시, 비용 확장성, **락인 완화**(어댑터 이관 비용 §1.1), SLA·삭제 요청(§3.1) |

- **계약 준수 최소선:** 어떤 벤더도 §1.2 최소 권위(게임 자원 비검증·blob 불투명)와 §8 버전 핀(시뮬 입력 분리)을 **깨지 않아야** 한다. 벤더가 세이브 파싱·서버 이코노미를 강제하면 본 아키텍처와 부적합.

---

## 열린 질문

본 문서는 `[TA]` 책임 분담·결정론 전제와 정합하는 **벤더 중립 API·파이프라인 계약**을 확정했다. 아래는 벤더 선정·스파이크·타 파트와 함께 답할 미결 사항이다.

1. **BaaS/MMP 벤더 선정·락인(D-106, `[TA §14-7]`)** — 통합형 vs 게임 특화 BaaS의 KR/글로벌 리전·PIPA·비용·영수증 검증 성숙도 대조. 본 계약을 만족하는 후보 중 어댑터 이관 비용이 가장 낮은 것은? PoC로 실측.
2. **오프라인 다이아 소비 정합(§3.4 `diamond/spend`)** — 다이아 원장이 서버 권위인데 게임은 오프라인 가능. 오프라인 다이아 소비를 (a) 금지(온라인 필수)할지 (b) 낙관적 큐+온라인 재조정할지. (b)면 잔액 불일치 UX·롤백 규칙 필요.
3. **계정 연동 충돌 해소 UX(§2.2-4)** — 소셜 계정에 기존 세이브가 있을 때 병합 금지 원칙(`[TA §3.5]`) 하의 사용자 선택 플로우 상세(`ux-design` 연동). 진행도 비교 표시 기준.
4. **세이브 blob 크기·업로드 전송(§3.2)** — 수백KB blob(`[TA §3.6]`)의 멀티파트 vs base64, 델타 업로드 필요성. 저대역 네트워크에서 업로드 실패·재시도 예산.
5. **밸런스 긴급 마이그레이션 예외 절차(§8.3, D-111)** — 진행 중 세이브 밸런스 불변 원칙이 치명적 exploit/소프트락 핫픽스를 막을 때의 "긴급 마이그레이션" 경계 표준화(`[TA §14-6]`, `[LO §9.1]`).
6. **텔레메트리 스키마 레지스트리 구현 형태(§10.2)** — 자체 레지스트리 vs 벤더 제공(예: 스키마드 이벤트) 중 무엇이 하위호환 계약 검사와 격리·플래그(드롭 금지)를 만족하는지.
7. **API 레이트 리밋·비용(§4.1)** — 배치 텔레메트리·config fetch·세이브 업로드의 요청 빈도(`[TA §3.6]` 디바운스 전제)와 벤더 요금 모델의 정합. 저사양·저대역에서의 백오프 파라미터.
8. **서버 샘플 검증 인프라의 필요·비용(§0.2, `[TA §10.1·14-11]`)** — 분쟁/어뷰징 시 코어를 서버에서 재컴파일해 "seed+스냅샷+로그" 재실행하는 옵션을 계약에 상시 포함할지, 온디맨드로 둘지.
