# 계약: 데이터 접근과 서버 측 동작

**Plan**: [../plan.md](../plan.md) | **Data model**: [../data-model.md](../data-model.md) | **AI**: [ai-api.md](./ai-api.md)

앱이 서버와 주고받는 인터페이스를 정의합니다. 일반 CRUD 는 Supabase 의 자동 REST API(PostgREST)를 **RLS 아래에서 브라우저가 직접** 호출하고,
"여러 행을 한 번에 일관되게 바꿔야 하는 동작"만 Postgres 함수(RPC)로 만듭니다. 서버 코드(Edge Function)는 [AI](./ai-api.md)와 계정 삭제뿐입니다.

## 1. 테이블 접근 규칙 (RLS)

| 테이블 | SELECT | INSERT | UPDATE | DELETE |
|--------|--------|--------|--------|--------|
| profiles | 본인 | 가입 시 트리거가 생성 | 본인(`timezone`, `ai_consent_*`) | 불가(계정 삭제로만) |
| tasks, subtasks, labels, task_labels | 본인 | 본인 (`user_id = auth.uid()` 강제) | 본인 | tasks 는 **소프트 삭제**(UPDATE `deleted_at`); 30일 후 서버 정리 작업이 영구 삭제 |
| standups, recurring_meetings, meeting_sessions, action_items | 본인 | 본인 | 본인 | 본인 (연결은 FK 규칙대로 끊김) |
| ai_requests | 본인(읽기 전용) | **불가** (Edge Function 서비스 역할만) | **불가** (RPC 만 상태 변경) | 불가 |

- 모든 정책은 `user_id = auth.uid()` 하나로 표현한다. 다른 사용자의 행 id 를 알아도 읽거나 쓸 수 없다(FR-027, pgTAP 로 검증).
- 클라이언트는 `id`(UUID)를 직접 만들어 INSERT 한다. 재시도 시 `upsert` 로 중복 생성 방지.
- `updated_at` 은 트리거가 설정한다(클라이언트 값 무시). 충돌은 마지막 수정이 이긴다.
- 연결 만들기(할 일로 만들기)는 아래 RPC 를 쓴다. 목록 화면의 읽기는 아래 "조회 계약"을 따른다.

## 2. 조회 계약 (클라이언트가 캐시에 올리는 쿼리)

| 이름 | 내용 | 비고 |
|------|------|------|
| `tasks.open` | `completed_at IS NULL AND deleted_at IS NULL` 전부 + `subtasks` + `task_labels` | 오늘/내일/백로그/밀린 일은 **클라이언트가 `plan_date` 로 계산**(research D10) |
| `tasks.done.recent` | 최근 30일 완료 | 완료 날짜별 보기(FR-006) |
| `tasks.done.range(from,to)` | 날짜 범위 완료 | 온라인에서만(오래된 기록) |
| `standups.recent` / `standups.byDate(d)` | 최근 90일 / 특정 날짜 | FR-018, FR-023 |
| `meetings.all` + `sessions.recent` + `action_items` | 정기 미팅 정의 + 최근 90일 회차 | FR-020~FR-023 |
| `profile` | 시간대·동의 상태 | |

작성 화면의 참고 데이터(스탠드업: 어제 완료한 할 일 = `completed_at` 이 어제인 할 일; 오늘 목록)는 위 캐시에서 계산한다(FR-019).

## 3. RPC (Postgres 함수, `SECURITY INVOKER` → RLS 적용)

모든 `apply_*` 는 **원자적**이며 `ai_requests.status = 'proposed'` 이고 `expires_at` 이 지나지 않은 본인 요청에만 동작한다.
이미 `confirmed` 인 요청에 다시 호출하면 **같은 결과를 반환하고 아무것도 더 만들지 않는다**(멱등, 더블 클릭 방지).

### `apply_ai_organize(request_id uuid, items jsonb)`
- `items`: 사용자가 **화면에서 고치고 확정한** 후보 배열 `[{ client_key, title, description, subtasks[], list, due_date, label_ids[] }]`. 빼 버린 후보는 포함하지 않는다.
- 동작: 각 항목을 검증(제목 1~180자 등) → `tasks`(+`subtasks`, `task_labels`) 삽입, `created_via='ai'`, `ai_request_id` 설정, `list` → `plan_date` 변환(서버가 사용자 시간대의 오늘 기준으로 계산) → 요청을 `confirmed` 로 → 생성된 id 목록 반환.
- 서버는 **모델의 제안을 신뢰하지 않고** 클라이언트가 보낸 확정본을 다시 검증한다.

### `apply_ai_refine(request_id uuid, apply jsonb)`
- `apply`: `{ title?, description?, subtasks?[] }` 중 사용자가 체크한 부분만.
- 동작: 대상 할 일의 `updated_at` 이 제안 시점(`base_updated_at`)과 다르면 `409 stale_task`(그 사이 다른 기기에서 수정됨; 화면이 다시 제안 비교를 보여 줌). 아니면 갱신하고 하위 할 일을 추가한다.

### `apply_ai_plan_today(request_id uuid, task_ids uuid[])`
- `task_ids`: 사용자가 선택한 항목만(FR-014 시나리오 2). 제안에 있던 id 의 부분집합이어야 한다.
- 동작: 선택된 할 일의 `plan_date` 를 오늘로 바꾼다(`position` 은 제안 순서). **선택하지 않은 할 일은 변경하지 않는다.**

### `discard_ai_request(request_id uuid)`
- 사용자가 승인하지 않고 닫을 때. 상태를 `discarded` 로. 어떤 할 일도 바뀌지 않는다(User Story 3 시나리오 3).

### `create_task_from_standup(standup_id uuid, text text, list text default 'today')`
- 스탠드업 기록의 문장을 할 일로 만든다(FR-021). `created_via='standup'`, `origin_standup_id`, `origin_excerpt = text`.

### `create_task_from_action_item(action_item_id uuid, list text default 'today')`
- 액션 아이템 텍스트로 할 일을 만들고 `action_items.task_id` 연결(FR-021). 이미 연결돼 있으면 기존 할 일을 반환.

### `export_my_data() → jsonb`
- 본인의 모든 데이터를 [data-model.md](../data-model.md#내보내기-형식-fr-030) 형식으로 반환(FR-030).

## 4. Edge Function: `delete-account`

- `POST /functions/v1/delete-account`, JWT 필수, 본문 `{ "confirm_email": "<본인 이메일>" }` — 불일치 시 400.
- 서비스 역할로 `auth.users` 를 삭제 → 모든 사용자 데이터가 cascade 로 삭제된다(FR-030). 응답 204. 이후 클라이언트는 로컬 캐시를 지우고 로그아웃한다.

## 5. 서버 정리 작업 (예약 실행, pg_cron 등)

| 작업 | 주기 | 내용 |
|------|------|------|
| 소프트 삭제 영구화 | 매일 | `deleted_at` 이 30일 지난 행 삭제 |
| AI 원문 비우기 | 매일 | 확정/폐기/만료된 지 30일 지난 `ai_requests` 의 `input_text`, `context_json`, `proposal_json` 을 null 로 (사용량 열은 유지) |
| 제안 만료 | 매일 | `expires_at` 지난 `proposed` → `expired` |

## 6. 클라이언트 오류 처리 규약

- 오프라인(`navigator.onLine === false` 또는 네트워크 오류): 모든 변경/AI 호출을 **보내지 않고** 차단 안내를 보인다(FR-028). 작성 중이던 폼 값은 메모리에 유지하고 "저장되지 않음"을 표시, 연결 복구 후 사용자가 다시 저장.
- `409`/충돌: 마지막 수정 우선이므로 일반 UPDATE 는 충돌 오류가 없다. `apply_ai_refine` 만 위의 `stale_task` 를 사용한다.
- 되돌리기(FR-008): 삭제·완료 직후 10초 동안 토스트로 되돌리기 제공. 삭제는 소프트 삭제라 10초 뒤에도 30일 안에는 복구 가능(UI 노출 여부는 화면 설계에서 결정).
