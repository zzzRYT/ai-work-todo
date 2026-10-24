# 계약: AI Edge Function (`ai`)

**Plan**: [../plan.md](../plan.md) | **Data model**: [../data-model.md](../data-model.md) | **Schemas**: [schemas/](./schemas/)

브라우저가 호출하는 AI 엔드포인트의 형태를 정의합니다. 하나의 Supabase Edge Function(`ai`) 안에서 경로로 나눕니다.
AI는 **제안만** 만듭니다. 이 함수는 `tasks` 를 만들거나 바꾸지 않습니다(FR-012). 반영은 [data-api.md](./data-api.md)의 `apply_*` RPC 가 담당합니다.

## 공통

- **URL**: `POST {SUPABASE_URL}/functions/v1/ai/{organize|refine|plan-today}`
- **인증**: `Authorization: Bearer <사용자 JWT>` 필수(함수가 JWT 검증). 인증 정보로 `user_id` 를 얻으며 본문의 사용자 식별자는 신뢰하지 않는다.
- **헤더**: `Idempotency-Key: <클라이언트 UUID>` 필수. 같은 키의 재요청은 새로 과금하지 않고 기존 `ai_requests` 의 결과를 돌려준다.
- **서버가 정하는 값**: "오늘" 날짜와 요일은 서버 시각 + `profiles.timezone` 으로 계산한다(클라이언트가 보낸 날짜를 쓰지 않는다 → FR-011, SC-004).
- **요청 순서(서버)**: ① JWT 검증 → ② 동의 확인 → ③ 입력 검증(길이) → ④ 일일 요청 상한·**월 예산 상한** 확인 → ⑤ 컨텍스트 조립 → ⑥ 제공자 호출(타임아웃 25초) → ⑦ 응답 검증/정제 → ⑧ `ai_requests` 저장(토큰 수와 추정 비용 포함) → ⑨ 응답.
- **로깅 금지**: 요청 본문, 입력 원문, 프롬프트, 모델 응답, **요청/응답 헤더(특히 제공자 `x-api-key`와 `Authorization`)** 를 함수 로그에 남기지 않는다. 제공자 SDK 의 디버그 로그도 끈다. 로그는 `request_id`, `kind`, 상태, 토큰 수, 지연 시간만.
- **응답 공통 필드**: `request_id`(uuid), `expires_at`(ISO 시각), `usage`(`{ remaining_today: int, month_used_pct: int }` — 이번 달 앱 예산 대비 사용 %; 금액 자체는 설정 화면에서만 조회).

## POST /ai/organize — 러프한 문장 → 할 일 후보 (FR-009~FR-011, FR-015)

**요청**
```json
{
  "text": "내일 오전까지 김 과장한테 보낼 분기 보고서 초안 만들고, 금요일 배포 전에 QA 체크리스트도 봐야 함",
  "default_list": "today"
}
```
- `text`: 필수, 1~4,000자(상한은 환경 변수로 조정).
- `default_list`: 선택. 모델이 목록을 정하지 못했을 때(`unspecified`) 클라이언트가 채울 현재 화면 목록. 모델에는 전달하지 않는다.

**모델에 보내는 것**: 시스템 지시문 + 현재 날짜/요일/시간대 + `text`. **그 외 사용자 데이터는 보내지 않는다.**

**성공 응답 200**
```json
{
  "request_id": "…",
  "expires_at": "2026-10-25T09:00:00Z",
  "usage": { "remaining_today": 27 },
  "candidates": [
    {
      "client_key": "c1",
      "title": "분기 보고서 초안 작성 (김 과장 전달)",
      "description": null,
      "subtasks": [],
      "list": null,
      "due_date": "2026-10-25",
      "due_basis": "explicit",
      "due_quote": "내일 오전까지"
    }
  ],
  "no_task_reason": null
}
```
- 서버가 [organize-output.schema.json](./schemas/organize-output.schema.json) 응답을 받은 뒤 **정제**한다.
  - 빈 문자열 → `null`, `unspecified` → `list: null`.
  - `due_date` 형식 오류, 오늘보다 과거(`explicit` 이고 원문 근거가 있을 때만 허용), 오늘로부터 2년 초과 → `null`.
  - `due_basis = explicit` 인데 `due_quote` 가 **원문에 그대로 들어 있지 않으면** `inferred` 로 강등(근거 없음 표시, FR-015).
  - `due_basis` 가 `inferred` 이면 화면은 "추정"으로 표시해야 한다.
  - 제목은 앞뒤 공백 제거 후 180자로 자른다. 제목이 빈 후보는 버린다. 후보 상한 20개.
- `candidates` 가 비면 `no_task_reason` 에 이유가 담긴다. 이때도 200 이며, **아무것도 만들지 않는다**(User Story 2 시나리오 5).

## POST /ai/refine — 기존 할 일 다듬기/쪼개기 (FR-013)

**요청**
```json
{ "task_id": "…", "instruction": "이거 더 구체적으로 나눠 줘" }
```
- `instruction`: 필수, 1~1,000자. `task_id`: 사용자의 할 일(서버가 RLS 로 조회; 없으면 404).

**모델에 보내는 것**: 지시문 + 그 할 일 **하나**의 제목·설명·기존 하위 할 일 제목 + `instruction`. 라벨/미팅/다른 할 일은 보내지 않는다.

**성공 응답 200**: [refine-output.schema.json](./schemas/refine-output.schema.json) 정제 결과(`title`/`description` 은 바꿀 게 없으면 `null`) + `base_updated_at`(제안 시점의 할 일 `updated_at`; 반영 시 그 사이 수정되었는지 검사).

## POST /ai/plan-today — 오늘 계획 제안 (FR-014)

**요청**
```json
{}
```
본문은 비어 있다. 후보 선별은 서버가 한다(클라이언트 입력으로 컨텍스트를 부풀릴 수 없음).

**서버가 조립하는 후보 풀** (최대 `PLAN_POOL_MAX`, 기본 80개)
1. 밀린 일(`plan_date < 오늘`, 미완료) 2. 오늘 목록 3. 내일 목록 4. `due_date ≤ 오늘+7` 인 백로그 5. 나머지 백로그는 우선순위 → 최근 생성 순으로 채움.
소프트 삭제·완료된 할 일은 제외.

**모델에 보내는 것**
- 현재 날짜/요일/시간대.
- 후보마다 `{ ref, title, list, overdue, due_in_days, priority }`. **`ref` 는 요청마다 새로 매기는 불투명한 값(`t1`, `t2`…)** 이고 실제 UUID 는 보내지 않는다. **설명·하위 할 일·라벨은 보내지 않는다.**
- 오늘 정기 미팅의 `{ name, start_time }` 목록. **미팅 기록 내용·스탠드업 내용은 보내지 않는다**(FR-017).

**성공 응답 200**
```json
{
  "request_id": "…",
  "expires_at": "…",
  "usage": { "remaining_today": 26 },
  "picks": [
    { "task_id": "…", "title": "…", "from_list": "overdue", "reason": "어제 못 끝냄" }
  ],
  "summary": "오후 2시 주간 회의가 있어요…",
  "sent_summary": { "task_count": 42, "meeting_names": ["주간 팀 회의"] }
}
```
- 서버가 [plan-today-output.schema.json](./schemas/plan-today-output.schema.json) 응답에서 **후보 풀에 없는 `ref` 는 버리고**, 중복 `ref` 는 합치고, `ref` 를 실제 `task_id`/현재 제목으로 바꿔 돌려준다. AI가 존재하지 않는 할 일을 만들어 낼 수 없다.
- `picks` 는 최대 15개. `sent_summary` 는 화면이 "이번에 AI에 보낸 내용"을 보여 줄 때 쓴다(개인정보 투명성).

## 오류 응답

모든 오류는 `{ "error": { "code": "...", "message": "한국어 안내", "retryable": bool } }` 형태이며, **AI가 실패해도 클라이언트는 입력 원문을 그대로 유지**하고(FR-016) 직접 추가 경로를 안내한다.

| HTTP | code | 상황 | 클라이언트 동작 |
|------|------|------|-----------------|
| 401 | `unauthenticated` | JWT 없음/만료 | 로그인 유도(오프라인이면 호출 자체를 막음) |
| 403 | `consent_required` | `ai_consent_version` 이 현재 고지 버전보다 낮음/없음 | 전송 내용 고지 대화상자 → 동의 후 재시도 |
| 404 | `task_not_found` | refine 대상 없음 | 목록 새로고침 |
| 413 | `input_too_long` | 입력 상한 초과 | 줄이도록 안내(원문 유지) |
| 429 | `quota_exceeded` | 일일 요청 한도 초과 | 남은 한도 0 표시, 직접 추가 안내. 응답에 `reset_at` |
| 429 | `monthly_budget_exceeded` | 이번 달 앱 예산 상한(`AI_MONTHLY_BUDGET_USD`) 도달 | AI 만 비활성 + "이번 달 AI 예산을 다 썼어요", 직접 추가 안내. 응답에 `reset_at`(다음 달 1일) |
| 503 | `provider_budget_exhausted` | 제공자 쪽 지출 한도·크레딧 소진(제공자가 400 `You have reached your specified … usage limits` 또는 429 `enforced_spend_limit_reached` 반환) | `monthly_budget_exceeded` 와 같은 UI. 재시도 무의미(`retryable: false`) |
| 502 | `provider_error` | 제공자 오류/거절(`refusal`)/스키마 불일치(1회 재시도 후) | "다시 시도" + 직접 추가, `retryable: true` |
| 504 | `provider_timeout` | 25초 초과 | 위와 동일 |
| 503 | `ai_disabled` | 서버 설정에서 AI 꺼짐(킬 스위치) | AI 버튼 비활성 + 안내 |

서버 측 1회 자동 재시도는 **스키마 불일치/일시적 5xx** 에만 하며, `max_tokens` 도달은 한 번 더 큰 한도로 재시도한다. 과금 폭주를 막기 위해 그 이상은 재시도하지 않는다.

## 서버 환경 변수 (이름만; 값은 저장소에 두지 않음)

- **비밀값**(Supabase Edge Function Secrets 에만 저장, 저장소·로그·브라우저에 없음): `ANTHROPIC_API_KEY`.
- **설정값**: `AI_MODEL`(기본 `claude-sonnet-5-5`, `claude-haiku-5-5` 로 교체 가능), `AI_ENABLED`, `AI_DAILY_REQUEST_LIMIT`(기본 40), `AI_MONTHLY_BUDGET_USD`(기본 20), `AI_PRICE_IN_PER_MTOK` / `AI_PRICE_OUT_PER_MTOK`(비용 계산용, 모델 가격표 값; 기본 Sonnet 5.5 기준 2 / 10), `AI_MAX_INPUT_CHARS`(기본 4000), `PLAN_POOL_MAX`(기본 80), `AI_CONSENT_VERSION`(현재 고지 버전), `ALLOWED_ORIGIN`(CORS 허용 출처 = `https://<APP_HOST>` 한 곳).
- `SUPABASE_URL` 등 `SUPABASE_` 접두 변수는 Supabase 가 주입한다. 값은 이 저장소에 두지 않는다 — 이름만 `.env.example` 에 둔다.

## 계약 테스트 항목 (구현 시 자동화)

1. 동의 없는 사용자 → 403, 제공자 호출 0회.
2. 일일 상한 또는 월 예산 상한 초과 → 429, 제공자 호출 0회.
   (추가) 제공자가 지출 한도 오류를 주면 `provider_budget_exhausted` 로 매핑.
3. 가짜 제공자가 스키마에 어긋난 JSON 을 주면 1회 재시도 후 502.
4. `explicit` + 원문에 없는 `due_quote` → `inferred` 로 강등.
5. plan-today 에서 풀에 없는 `ref` 제거, 서버가 UUID 를 모델 요청에 넣지 않음.
6. 요청/응답 본문과 헤더(키 포함)가 로그로 출력되지 않음.
7. 같은 `Idempotency-Key` 재요청 시 제공자 호출 1회.
