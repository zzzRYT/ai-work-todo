# Phase 1 데이터 모델

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Research**: [research.md](./research.md)

저장소는 Postgres(Supabase)입니다. 아래는 **논리 모델**이며 실제 마이그레이션 SQL은 구현 단계(tasks)에서 만듭니다.
모든 테이블은 `user_id`를 갖고 **행 수준 보안(RLS)** 으로 `user_id = auth.uid()` 인 행만 읽고 쓸 수 있습니다(FR-027).
공통 열: `id uuid`(클라이언트가 생성 가능), `created_at`, `updated_at`(서버 트리거가 갱신, 충돌 시 마지막 수정 우선).

## 관계 개요

```text
auth.users 1─1 profiles
auth.users 1─* labels ─* task_labels *─ tasks 1─* subtasks
                                          │  ▲
                       (연결, 삭제 시 끊김) │  │ task_id
standups 1─* tasks (origin_standup_id)    │  │
recurring_meetings 1─* meeting_sessions 1─* action_items ┘
auth.users 1─* ai_requests ── (apply 시) ── tasks.ai_request_id
```

## 엔티티

### profiles (스펙: 사용자)

| 필드 | 타입 | 설명 / 규칙 |
|------|------|-------------|
| user_id | uuid PK, FK→auth.users, cascade | 계정 삭제 시 함께 삭제(FR-030) |
| timezone | text, 기본 `Asia/Seoul` | IANA 시간대 이름. "오늘/내일" 기준(FR-029). 유효성은 앱에서 `Intl` 로 검사 |
| ai_consent_version | int null | 사용자가 동의한 AI 데이터 전송 고지 버전. null = 미동의(AI 차단) |
| ai_consent_at | timestamptz null | 동의 시각 |

### tasks (스펙: 할 일)

| 필드 | 타입 | 설명 / 규칙 |
|------|------|-------------|
| title | text NOT NULL | 1~180자, 앞뒤 공백 제거(FR-003 필수) |
| description | text null | 최대 2,000자 |
| plan_date | date null | **목록을 결정하는 유일한 값**. null=백로그, 오늘 날짜=오늘, 오늘+1=내일 (research D10) |
| due_date | date null | 마감일(선택). 목록과 별개 |
| priority | smallint null | 1(높음)~4(낮음), null=없음 |
| position | double precision | 같은 목록 안의 순서(FR-005). 사이 값 삽입, 간격이 1e-6 미만이면 재정렬 |
| completed_at | timestamptz null | null=미완료. 완료 날짜별 보기(FR-006)에 사용 |
| deleted_at | timestamptz null | 소프트 삭제(되돌리기 FR-008). 30일 후 영구 삭제 |
| created_via | enum | `manual` \| `ai` \| `standup` \| `meeting` |
| origin_standup_id | uuid null, FK→standups, **on delete set null** | 스탠드업에서 만든 할 일의 출처 |
| origin_excerpt | text null | 출처 문장 원문(출처 기록이 지워져도 남는 참고용) |
| ai_request_id | uuid null, FK→ai_requests, on delete set null | 어떤 AI 제안에서 만들어졌는지 |

- 미팅 회차의 액션 아이템에서 만든 할 일은 `action_items.task_id` 로 연결한다(역방향 조회: 할 일 → 액션 아이템 → 회차).
- **파생(저장하지 않음)** — 사용자 시간대의 오늘 = `T`:
  - `list`: `plan_date IS NULL`→백로그, `= T+1`→내일, `<= T`→오늘
  - `is_overdue`(밀린 일): 미완료 ∧ `plan_date < T`
  - `due_soon`: 미완료 ∧ `due_date <= T+2`(AI 오늘 계획 후보 선별에 사용)
- 인덱스: `(user_id, completed_at)`, `(user_id, plan_date) WHERE completed_at IS NULL AND deleted_at IS NULL`, `(user_id, due_date)`.

**상태 전이 (할 일)**

```text
[백로그 plan_date=null] ⇄ [내일 T+1] ⇄ [오늘 T]      (이동 = plan_date 변경)
각 상태 ──완료──▶ completed_at 설정 (완료 목록, plan_date 유지) ──완료 취소──▶ 이전 목록
각 상태 ──삭제──▶ deleted_at 설정 ──되돌리기──▶ 복원 / 30일 경과 ──▶ 영구 삭제
날짜 변경: 저장 값은 그대로, 파생 list 만 변함 (내일→오늘, 오늘 미완료→오늘의 "밀린 일")
```

### subtasks (스펙: 하위 할 일 / 체크리스트)

| 필드 | 타입 | 설명 |
|------|------|------|
| task_id | uuid FK→tasks, cascade | |
| title | text NOT NULL | 1~180자 |
| done | boolean | |
| position | double precision | |

### labels, task_labels (스펙: 라벨/프로젝트)

스펙의 "라벨/프로젝트"는 **하나의 개념(라벨)** 으로 통합한다(첫 버전은 프로젝트 계층 없음).

| labels 필드 | 타입 | 설명 |
|------|------|------|
| name | text NOT NULL | 사용자 안에서 유일(대소문자 무시) |
| color | text null | 선택 |

`task_labels(task_id, label_id)` 복합 PK, 양쪽 cascade 삭제.

### standups (스펙: 스탠드업 기록)

| 필드 | 타입 | 설명 / 규칙 |
|------|------|-------------|
| standup_date | date NOT NULL | `UNIQUE (user_id, standup_date)` — 날짜당 하나 |
| yesterday | text | 어제 한 일 |
| today | text | 오늘 할 일 |
| blockers | text | 막힌 점 |
| notes | text | 자유 메모 |

- 사용자가 직접 작성한다. AI가 쓰거나 요약하지 않는다(FR-017).
- 작성 화면의 "참고용" 데이터(어제 완료한 할 일, 오늘 목록)는 **tasks 에서 조회**하는 것이지 복사 저장이 아니다(FR-019).
- 삭제하면 `tasks.origin_standup_id` 는 null 이 되고 할 일은 남는다(Edge case).

### recurring_meetings (스펙: 정기 미팅)

| 필드 | 타입 | 설명 / 규칙 |
|------|------|-------------|
| name | text NOT NULL | |
| weekday | smallint NOT NULL | 0(일)~6(토), 첫 버전은 매주 특정 요일만 |
| start_time | time null | 선택. AI 오늘 계획이 "오늘 미팅 일정"으로 참고(FR-014) |
| started_on | date NOT NULL | 이 날짜부터 회차가 생김 |
| ended_on | date null | 종료(FR-020 "종료"). 이후 회차 없음, 과거 기록은 유지 |

### meeting_sessions (스펙: 미팅 회차 기록)

| 필드 | 타입 | 설명 / 규칙 |
|------|------|-------------|
| meeting_id | uuid FK→recurring_meetings, cascade | |
| session_date | date NOT NULL | `UNIQUE (meeting_id, session_date)` |
| status | enum | `open` \| `skipped`. 건너뜀 표시(공휴일/취소, FR-020) |
| content | text | 회의 내용(사용자가 직접 작성) |

- **회차 행은 필요할 때 생성**한다. 요일이 맞는 날짜는 화면에서 "기록할 수 있는 회차"로 **계산해서** 보여 주고(User Story 5 시나리오 1), 사용자가 처음 쓰거나 건너뜀을 누를 때 행을 만든다. 배치 작업 불필요.
- 지난 회차 = 같은 `meeting_id`에서 `session_date`가 더 이른 가장 가까운 행(FR-022).

### action_items (회차의 액션 아이템)

| 필드 | 타입 | 설명 / 규칙 |
|------|------|-------------|
| session_id | uuid FK→meeting_sessions, cascade | |
| text | text NOT NULL | 사용자가 적은 문장(AI가 추출하지 않음, FR-017) |
| task_id | uuid null, FK→tasks, **on delete set null** | 할 일로 만들면 연결. 어느 쪽을 지워도 다른 쪽은 남는다 |
| done | boolean | 할 일에 연결되지 않은 항목의 자체 완료 표시 |
| position | double precision | |

- **진행 상황(파생)**: `task_id` 가 있으면 `tasks.completed_at IS NOT NULL`, 없으면 `done`. "지난 회차의 미완료 액션 아이템"(FR-022)은 이 파생값이 미완료인 것.
- 할 일을 지우면(소프트 삭제 포함) 연결은 끊긴 것으로 표시하고 액션 아이템 텍스트는 남는다.

### ai_requests (스펙: AI 요청과 제안)

| 필드 | 타입 | 설명 / 규칙 |
|------|------|-------------|
| kind | enum | `organize` \| `refine` \| `plan_today` |
| status | enum | `proposed` → `confirmed` \| `discarded` \| `expired`, 또는 `failed` |
| input_text | text null | 사용자가 입력한 원문(organize/refine 지시문). 확정/폐기 후 **30일 지나면 null 로 비움** |
| context_json | jsonb null | 전송한 컨텍스트 요약(ref 목록 등). 같은 보존 정책 |
| proposal_json | jsonb null | 서버가 검증한 제안(후보 목록/계획안). 같은 보존 정책 |
| result_json | jsonb null | 사용자가 확정한 최종 결과 요약 |
| model | text | 사용한 모델 이름 |
| tokens_in, tokens_out | int null | 사용량 |
| error_code | text null | 실패 사유 |
| idempotency_key | text | `UNIQUE (user_id, idempotency_key)` — 더블 클릭/재시도에 중복 과금 방지 |
| expires_at | timestamptz | 제안 만료(생성 후 24시간). 만료된 제안은 확정 불가 |

- 확정 전까지 tasks 에는 아무 영향이 없다(FR-012). 확정은 RPC 한 번에 원자적으로 수행([contracts/data-api.md](./contracts/data-api.md)).
- 일일 쿼터 계산은 `created_at` 으로 이 테이블을 센다. `input_text` 등은 지우더라도 **행(사용량)** 은 유지한다.
- 인덱스: `(user_id, created_at)`.

## 유효성·제약 요약

- 제목/이름 류: 앞뒤 공백 제거 후 1자 이상, 길이 상한 DB `CHECK` 로 강제.
- `plan_date`, `due_date` 는 날짜만(시각 없음). 시간대 변환은 클라이언트의 "오늘" 계산에서만 발생.
- 외래키 삭제 규칙은 스펙 Edge case("연결만 끊어짐")를 따른다: 연결 열은 `ON DELETE SET NULL`, 소유 관계(할 일→하위 할 일, 회차→액션 아이템)는 `CASCADE`.
- 계정 삭제(FR-030): `auth.users` 삭제 → 모든 테이블 cascade.

## 내보내기 형식 (FR-030)

`export_my_data()` 가 아래 구조의 JSON 하나를 반환하고 클라이언트가 파일로 내려받는다.
`{ "version": 1, "exported_at", "profile", "labels", "tasks"(+subtasks, label ids), "standups", "recurring_meetings", "meeting_sessions"(+action_items) }`.
`ai_requests` 는 사용량 기록이므로 원문·제안을 제외한 요약만 포함한다.

## 오프라인 캐시에 올라가는 범위

열린 할 일 전체(+하위 할 일, 라벨) · 최근 30일 완료 · 최근 90일 스탠드업과 회차 · 정기 미팅 정의 · 프로필. `ai_requests` 는 캐시하지 않는다.
