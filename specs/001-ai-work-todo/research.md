# Phase 0 리서치: 기술 선택 제안

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **작성일**: 2026-10-24

이 문서의 모든 선택은 **제안**입니다. 항목마다 추천안, 검토한 대안, 이유를 적었으니 캡틴이 뒤집을 수 있어요.
서비스 한도·가격 같은 주장에는 출처와 확인일(2026-10-24)을 붙였고, 확인하지 못한 것은 "미확인"이라고 적었습니다.

판단 기준(헌법 II): 개인 사이드 프로젝트 → 작게 시작, 비용 최소, 검증된 도구.

## 선택 한눈에 보기

| # | 주제 | 추천 | 주요 대안 |
|---|------|------|-----------|
| D1 | 웹/모바일 제공 방식 | 반응형 **PWA** (홈 화면 추가) | Capacitor 래핑, React Native/네이티브 |
| D2 | 프론트엔드 | **React + TypeScript + Vite** SPA | Next.js, SvelteKit, Vue |
| D3 | 백엔드/저장소 | **Supabase** (Postgres + Auth + RLS + Edge Functions) | Firebase, Cloudflare Workers+D1, 자체 호스팅 |
| D4 | AI 호출 방식 | **서버(Edge Function)에서만** AI 호출, 키는 서버 비밀값 | 브라우저 직접 호출, BYO 키 |
| D5 | AI 제공자/모델 | **Anthropic Claude Haiku 5.5** (교체 가능한 어댑터) | Sonnet 5.5, Gemini, OpenAI |
| D6 | 인증 | **Supabase Auth**, 이메일 매직링크 + (선택) Google, **가입 닫기** | 비밀번호, 직접 구현 |
| D7 | 배포 | 정적 SPA는 **Cloudflare Pages**, 백엔드는 Supabase 클라우드 | Vercel Hobby, Netlify, GitHub Pages |
| D8 | 오프라인(읽기 전용) | 서비스 워커(앱 셸) + **TanStack Query 캐시를 IndexedDB에 저장** | 오프라인 우선(local-first) 동기화 엔진 |
| D9 | 동기화/충돌 | 새로고침·포커스 시 재조회 + 마지막 수정 우선(LWW), 클라이언트 UUID, 소프트 삭제 | 실시간 구독, CRDT |
| D10 | "오늘/내일" 계산 | 저장은 `plan_date` 하나, 목록은 **읽을 때 계산** | 자정 배치 작업으로 이동 |
| D11 | 검색 | 첫 버전은 **클라이언트 검색**(캐시된 데이터) | Postgres `pg_trgm`, 외부 검색 서비스 |
| D12 | 테스트 | Vitest + Testing Library, Playwright(모바일 뷰포트), Supabase 로컬 + pgTAP(RLS) | Jest, Cypress |

---

## D1. 웹/모바일 제공 방식

- **추천**: 하나의 반응형 웹앱을 **PWA**(웹 앱 매니페스트 + 서비스 워커)로 만들어 PC 브라우저와 휴대폰 브라우저에서 쓰고, 휴대폰은 홈 화면에 추가해 앱처럼 연다.
- **대안**
  - Capacitor/Ionic으로 감싼 스토어 앱: 푸시·네이티브 기능이 필요하면 유리하지만 스펙의 Assumptions가 "앱 스토어 배포는 범위 밖", 알림도 범위 밖이라 지금은 이득이 없다.
  - React Native/Flutter: 코드베이스가 2개 이상으로 늘고 스토어 심사·계정 비용이 든다.
- **이유**: 스펙 FR-024/FR-025와 Assumptions가 정확히 PWA 형태. 코드 한 벌, 배포 한 번.
- **근거/주의**: iOS Safari는 iOS 11.3(2018)부터 서비스 워커와 웹 매니페스트를 지원한다
  ([love2dev, 2018](https://love2dev.com/blog/apple-ships-service-workers), [MagicBell PWA iOS 가이드](https://www.magicbell.com/blog/pwa-ios-limitations-safari-support-complete-guide), 2026-10-24 검색 확인).
  다만 iOS에서 홈 화면 앱과 Safari 탭의 저장소·로그인 세션이 분리되는지는 **미확인** → 구현 초기에 실기기로 확인할 항목(quickstart 시나리오 7).

## D2. 프론트엔드

- **추천**: **React + TypeScript + Vite** 단일 페이지 앱(SPA), 서버 상태는 **TanStack Query**, PWA는 **vite-plugin-pwa**(Workbox).
- **대안**
  - Next.js: 서버 렌더링이 필요 없고(로그인 뒤 개인 앱, SEO 불필요), 별도 서버 런타임을 얹게 된다.
  - SvelteKit/Vue: 가볍고 좋지만 React가 PWA·쿼리 캐시 영속화·UI 라이브러리 자료가 가장 풍부하다.
  - 사전 프로토타입(바닐라 `index.html` + localStorage)을 확장: 로그인·동기화·AI 확장 시 상태 관리 한계. (프로토타입은 UX 참고용으로만 읽었고 가져오지 않았다. 참고한 것: 오늘/내일/백로그 3탭, 빠른 추가 입력, 되돌리기 토스트, 밀린 일 표시.)
- **이유**: 보편적이라 도움 받기 쉽고, 오프라인 캐시 영속화(D8)에 필요한 TanStack Query가 React 어댑터를 공식 제공한다.
- **열린 항목**: 스타일링(Tailwind vs CSS Modules)과 UI 컴포넌트 라이브러리는 **화면 디자인 단계에서 결정**한다.

## D3. 백엔드/저장소

- **추천**: **Supabase 클라우드 무료 플랜** — Postgres(관계형 데이터), Auth, 행 수준 보안(RLS), Edge Functions(AI 프록시), 자동 생성 REST API(PostgREST).
- **이유**
  - 데이터가 관계형(할 일↔미팅 기록↔액션 아이템 연결, 삭제 시 연결 끊기)이라 SQL의 외래키/제약이 잘 맞는다.
  - 본인 데이터만 보기(FR-027)를 **RLS 정책으로 DB 레벨에서** 강제할 수 있어 서버 코드가 거의 필요 없다. 일반 CRUD는 브라우저가 RLS 아래에서 직접 수행하고, 서버 코드는 AI·계정 삭제 두 가지뿐이다.
  - 한 곳에서 인증+DB+서버 함수를 해결 → 운영 대상이 하나.
- **무료 플랜 수치** ([Supabase Pricing](https://supabase.com/pricing), 2026-10-24 확인): DB 500 MB/프로젝트, 월 활성 사용자 50,000, 송신 5 GB, Edge Function 호출 월 500,000회 포함, **1주 비활동 시 프로젝트 일시정지**, **무료 플랜은 일일 백업 없음**(Pro부터 7일).
  Edge Function 한도 ([Supabase Docs: Limits](https://supabase.com/docs/guides/functions/limits), 2026-10-24 확인): 요청당 CPU 2초(비동기 I/O 제외), 메모리 256 MB, 응답 대기 150초(초과 시 504), 무료 플랜 워커 벽시계 150초.
- **우리 사용량 대비**: 1인 사용, 할 일 수천 건이어도 수 MB. AI 호출은 하루 수십 회 이하 → 한도에 한참 못 미친다.
- **리스크와 완화**
  1. **1주 비활동 일시정지**: 평일에 쓰면 문제없지만 휴가 후 첫 접속이 실패할 수 있다. 완화: 설정 화면 안내 + 복구 방법 문서화. 계속 거슬리면 Pro(월 $25, 같은 Pricing 페이지) 전환 또는 대안 D3-C.
  2. **무료 플랜 백업 없음**: 완화: 앱의 JSON 내보내기(FR-030)를 정기 백업 수단으로 안내(예: 월 1회).
  3. **벤더 종속**: Postgres 표준 SQL 마이그레이션으로 관리 → 다른 Postgres 호스트로 이전 가능. 인증·RLS 정책이 Supabase 고유 부분이다.
- **대안**
  - **Firebase**(Auth + Firestore + Cloud Functions): 오프라인 쓰기 동기화가 강점이나 우리는 오프라인 편집이 필요 없고(FR-028), 문서형 DB는 연결·삭제 시 연결 끊기 같은 관계 규칙이 번거롭다. 이 문서에서는 Firebase 요금·한도를 조사하지 않았다(**미확인**) — 기술 적합성 때문에 후순위.
  - **Cloudflare Workers + D1(SQLite) + 자체 인증**: 비용이 가장 낮고 호스팅과 한 벤더지만 인증·세션·접근 제어를 직접 짜야 해서 코드와 보안 부담이 커진다. Supabase 일시정지가 큰 불편이 되면 2순위 후보.
  - **자체 호스팅(Node + SQLite/Postgres on VPS)**: 월 고정비와 운영(패치, 백업, TLS)이 생긴다.

## D4. AI는 어디서 호출하나 (서버에 둘 것)

- **추천**: 브라우저 → **Supabase Edge Function `ai`** → AI 제공자. API 키는 Edge Function **비밀값**으로만 보관.
- **서버에 남기는 것(브라우저에 두지 않음)**: API 키, 시스템 프롬프트, 모델 이름, 응답 스키마, 호출 한도(쿼터) 검사, 동의 확인, 사용량 기록, 응답 검증.
- **대안**
  - 브라우저에서 AI 직접 호출: 키가 노출돼 누구나 가져다 쓸 수 있다 → 불가.
  - 사용자가 자기 API 키를 입력(BYO key): 개인용이라 가능하지만 키를 브라우저 저장소에 두게 되고 기기마다 입력해야 한다. 서버 키 + 쿼터가 더 단순하다. 멀티 유저로 확장할 때 재검토.
- **이유**: 키 보호, 일관된 쿼터/비용 통제, 응답 검증을 한 곳에서 한다(AI 설계는 [plan.md](./plan.md#ai-동작-설계) 참고).

## D5. AI 제공자와 모델

- **추천**: **Anthropic Claude Haiku 5.5** (`claude-haiku-5-5`)를 기본으로, 모델 이름은 환경 변수로 바꿀 수 있게 한다. 품질이 SC-003(90%)에 못 미치면 **Sonnet 5.5**(`claude-sonnet-5-5`)로 올린다.
- **가격** ([Anthropic Pricing](https://platform.claude.com/docs/en/about-claude/pricing), 2026-10-24 확인): Haiku 5.5는 프롬프트 10만 토큰 이하 **입력 $0.10 / 출력 $0.50 (백만 토큰당)**, 10만 토큰 초과 시 $0.50 / $2.50. Sonnet 5.5는 $2 / $10. 프롬프트 캐시 읽기 Haiku 5.5 $0.01/MTok.
  우리 요청은 10만 토큰을 한참 밑돈다.
- **비용 추정(가정 포함)**
  - 정리 요청: 입력 약 1.5k 토큰(시스템 프롬프트+원문), 출력 약 0.6k → 약 **$0.0005/회**.
  - 오늘 계획: 후보 최대 80개 × 약 60 토큰 + 미팅/지시문 → 입력 약 7k, 출력 약 1k → 약 **$0.0012/회**.
  - 하루 상한 30회를 전부 오늘 계획으로 써도 하루 약 $0.04, **월 최대 약 $1.1**. 한국어 토큰이 영어보다 많이 나오고, 한 기사는 새 토크나이저가 이전 모델 대비 같은 입력에서 약 30% 더 많은 토큰을 낸다고 보도했으므로 ([TechEnclave, 2026-10](https://techenclave.com/news/items/claude-haiku-5-5-launches-with-0-10-input-pricing-for-shorter-prompts), 2차 출처) 여유를 두어도 **월 $2 이내**로 본다. 실제 토큰 수는 구현 후 `ai_requests`에 기록해 확인한다.
- **구조화 출력**: Messages API가 `output_config.format`에 `type: "json_schema"`로 스키마를 받아 유효한 JSON을 돌려준다. `claude-haiku-5-5` 지원 ([Structured outputs 문서](https://platform.claude.com/docs/en/build-with-claude/structured-outputs), 2026-10-24 확인). 제약: 재귀 스키마, 숫자/문자열 길이 제약, `additionalProperties: false` 외 값은 미지원 → [contracts/schemas](./contracts/schemas/)는 이 부분집합만 쓴다. `stop_reason`이 `refusal`/`max_tokens`면 스키마와 어긋날 수 있어 서버에서 따로 처리한다.
- **데이터 보관/학습** (개인정보 판단 근거):
  - Anthropic 상업용(API) 기본 정책: 입력·출력을 백엔드에서 **수신/생성 후 30일 이내 자동 삭제**, 예외(Files API 파일, 별도 계약, 정책 위반 집행, 법적 요구) ([Anthropic Privacy Center](https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data), 2026-10-24 확인).
  - API 문서는 "보관된 데이터는 명시적 허락 없이 모델 학습에 쓰이지 않는다"고 적고 있다 ([API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention), 2026-10-24 확인; 해당 페이지의 문구이며, 약관 전체를 법률 검토한 것은 아님).
  - Zero Data Retention(ZDR)은 별도 신청·검토가 필요해 개인 프로젝트에는 기대하지 않는다.
- **대안**
  - **Gemini 무료 티어**: 비용 0이 매력적이지만, 약관상 **무료(Unpaid) 서비스는 제출한 내용과 응답이 Google 제품·머신러닝 개선에 쓰이고 사람이 읽을 수 있으며 "민감/기밀/개인 정보를 제출하지 말라"고 경고**한다. 반면 유료(Paid)는 프롬프트를 제품 개선에 쓰지 않는다 ([Gemini API Additional Terms](https://ai.google.dev/gemini-api/terms), 2026-10-24 확인). 회사 업무 정보를 보내는 앱이므로 **무료 티어는 부적합**. 유료 티어는 선택지로 남는다.
  - OpenAI 등 다른 제공자: 이번에 가격·약관을 조사하지 않았다(**미확인**). 어댑터 한 파일만 바꾸면 되도록 제공자 호출을 분리해 둔다.
  - 로컬/온디바이스 모델: 휴대폰 지원·품질 문제로 첫 버전 범위 밖.

## D6. 인증

- **추천**: Supabase Auth, 첫 버전은 **이메일 매직링크(OTP)**; Google 로그인은 선택 옵션. **공개 가입은 닫고** 소유자 계정만 만든다(Supabase의 가입 허용 설정을 끔).
- **이유**: 스펙 Assumptions "표준 인증". 비밀번호를 직접 저장·재설정할 필요가 없다. 가입을 닫는 이유는 **AI 호출 비용 노출 방지** — 서버 키로 AI를 쓰므로 아무나 가입하면 내 비용이 나간다.
- **대안**: 이메일+비밀번호(재설정/유출 대응 부담), 직접 구현(보안 부담).
- **캡틴 확인 필요**: FR-026/FR-027이 "소유자 1명만 쓰는 앱"인지 "각자 자기 데이터만 보는 서비스"인지 문구만으로는 모호하다. 데이터 모델은 사용자별 RLS라 둘 다 지원하지만, 가입 정책(닫음/초대제/공개)과 사용자별 AI 쿼터는 달라진다. 추천은 **가입 닫음**.
- **오프라인 주의**: 오프라인에서는 토큰 갱신이 안 되므로, 만료된 세션 때문에 읽기 전용 화면이 로그아웃으로 튕기지 않게 해야 한다(quickstart 시나리오 7). 로그아웃 시에는 기기에 캐시된 데이터를 지운다(공용 PC 보호).

## D7. 배포

- **추천**: 정적 SPA를 **Cloudflare Pages**에 배포(Git 연동 자동 빌드), 백엔드는 Supabase 클라우드. 환경별 분리는 Supabase 프로젝트 1개(운영) + 로컬 Supabase(개발)로 시작.
- **한도** ([Cloudflare Pages Limits](https://developers.cloudflare.com/pages/platform/limits/), 2026-10-24 확인): 무료 플랜 빌드 월 500회, 사이트당 파일 20,000개, 파일 하나 25 MiB. 해당 페이지에는 대역폭·정적 요청 한도 언급이 없다(제3자 글은 "정적 자산 무제한"이라 하나 공식 확인은 못 함).
- **대안**: Vercel Hobby(개인 용도 무료지만 약관/한도는 이번에 **미확인**), Netlify, GitHub Pages(빌드 파이프라인이 단순하지만 SPA 라우팅/헤더 설정이 제한적). 정적 파일만 올리므로 어느 쪽이든 이전 비용이 낮다.
- **CI**: GitHub Actions(타입체크·린트·단위·RLS 테스트)는 PR마다. 배포는 Pages의 Git 연동에 맡긴다.

## D8. 오프라인(읽기 전용) 전략

- **추천**
  - **앱 셸**: vite-plugin-pwa(Workbox)로 JS/CSS/아이콘을 프리캐시 → 오프라인에서도 앱이 열린다.
  - **데이터**: TanStack Query 캐시를 `persistQueryClient`로 **IndexedDB에 영속화**하고, 오프라인이면 캐시를 읽기 전용으로 보여 준다. 마지막으로 불러온 시각(`dataUpdatedAt`)을 배너에 표시한다.
  - **쓰기 차단**: 오프라인(`navigator.onLine` + 요청 실패)에서는 모든 변경·AI 버튼을 비활성화하고 안내를 보인다(FR-028).
- **근거** ([TanStack Query persistQueryClient 문서](https://tanstack.com/query/latest/docs/framework/react/plugins/persistQueryClient), 2026-10-24 확인): 영속화 어댑터(`idb-keyval` 기반 IndexedDB 예시 포함), `maxAge` 기본 24시간(초과하면 폐기), `gcTime`을 `maxAge` 이상으로 두어야 하며 `Infinity`로 GC를 끌 수 있다. → `gcTime: Infinity`와 긴 `maxAge`(예: 90일)를 쓰되, 장기 오프라인 동작은 구현 단계에서 검증한다.
- **대안**: local-first 동기화 엔진(PowerSync, ElectricSQL 등)은 오프라인 **편집**용인데 스펙이 오프라인 편집을 배제했으므로 과하다. 직접 만든 IndexedDB 스냅샷은 가능하지만 TanStack Query 캐시와 이중 관리가 된다.
- **데이터 범위**: 열려 있는 할 일 전부 + 최근 30일 완료 + 스탠드업/미팅 최근 90일을 미리 불러와 캐시한다. 그보다 오래된 기록은 온라인에서 조회할 때만 가져온다(스펙 갭 참고).
- **프라이버시**: IndexedDB는 기기에 평문으로 남는다. 로그아웃 시 삭제하고, 설정에 "이 기기 캐시 지우기"를 둔다.

## D9. 동기화와 충돌

- **추천**: 앱 열기·탭 포커스 복귀·네트워크 복구 때 재조회(staleTime 짧게), 변경은 낙관적 업데이트 후 서버 확정. 충돌은 **마지막 수정 우선(LWW)**: 모든 행에 `updated_at`(서버 트리거가 갱신). ID는 **클라이언트가 만든 UUID**라 재시도해도 중복 생성되지 않고(멱등 upsert), 삭제는 **소프트 삭제(`deleted_at`)** 라 되돌리기(FR-008)가 가능하며 동시 수정이 행을 되살리거나 사라지게 만들지 않는다.
- **SC-007(5초 이내 반영)**: 새로 열거나 새로고침 시 재조회로 충족. 열려 있는 다른 기기에 즉시 반영하는 Supabase Realtime 구독은 스펙 요구가 아니므로 **첫 버전 제외**(나중에 추가 가능).
- **대안**: CRDT/실시간 협업 - 1인 사용이라 불필요.

## D10. "오늘 / 내일 / 백로그" 표현

- **추천**: 할 일에 `plan_date`(date, null 허용) 하나만 저장.
  - 백로그 = `plan_date IS NULL`
  - 내일 = `plan_date = 오늘+1`
  - 오늘 = `plan_date = 오늘`, **밀린 일** = `plan_date < 오늘` 이면서 미완료(오늘 목록에 "밀린 일"로 구분 표시)
  - "오늘"의 기준은 `profiles.timezone`의 자정(FR-029). 목록 구성은 **읽을 때 클라이언트가 계산**한다.
- **이유**: 자정 배치 작업 없이 FR-004(내일→오늘 자동 이동, 밀린 일 유지)가 된다. 앱을 며칠 안 열어도, 오프라인 캐시를 열어도 항상 맞고, 시간대를 바꾸면 즉시 다시 계산된다. 이동은 `plan_date`만 바꾸면 된다(오늘=오늘 날짜, 내일=오늘+1, 백로그=null).
- **대안**: 상태 열(`bucket` enum) + 자정에 DB 작업으로 이동 → 작업 스케줄링, 시간대별 처리, 놓친 날 보정이 필요해 복잡. 프로토타입이 `bucket`+`dayKey`+앱 시작 시 이동을 쓰지만 쓰기가 생기고 다중 기기에서 충돌할 수 있다.
- 자정에 앱이 켜져 있으면: 클라이언트가 다음 자정에 타이머 + `visibilitychange`에서 "오늘"을 다시 계산한다.

## D11. 검색

- **추천**: 첫 버전은 **클라이언트 검색**(이미 캐시된 열린 할 일·최근 기록을 제목/설명/라벨로 필터). 오프라인에서도 동작하고 서버 인덱스가 필요 없다.
- **근거**: 1인 사용이라 열린 할 일은 수백~수천 건 규모로 브라우저 메모리로 충분(FR-007의 "백로그 수백 개").
- **확장 경로**: 오래된 완료 기록·미팅 기록 검색은 Postgres `pg_trgm`(한국어는 기본 전문 검색 설정이 형태소 분석을 못 하므로 트라이그램이 현실적)로 서버 검색을 추가한다. 이 선택은 규모가 커질 때까지 미룬다.

## D12. 테스트

- 단위/컴포넌트: **Vitest + Testing Library**. 날짜 계산(`plan_date` 파생, 시간대 경계)은 순수 함수로 분리해 집중 테스트.
- 종단: **Playwright**(데스크톱 + 작은 휴대폰 뷰포트로 가로 스크롤 없음 검증 — SC-008).
- DB/RLS: **Supabase CLI 로컬 스택 + pgTAP**으로 "다른 사용자 데이터 접근 불가", 연결 삭제 시 끊김, 소프트 삭제를 검증.
- AI: 제공자를 가짜로 바꾼 **계약 테스트**(스키마 검증, 근거 없는 날짜 강등, 후보 풀 밖 참조 제거, 쿼터/동의 거절) + 선택적 **실제 모델 평가 스크립트**(한국어 표본 문장으로 SC-003/SC-004 측정, 수동 실행, 키는 로컬 환경 변수).

---

## 미해결/미확인 요약

| 항목 | 상태 |
|------|------|
| iOS 홈 화면 앱과 Safari의 세션/저장소 분리 | 미확인 → 실기기 검증 |
| Firebase, OpenAI, Vercel 요금·약관 | 조사 안 함(추천안에서 제외한 이유는 위에 명시) |
| Cloudflare Pages 정적 대역폭 한도 | 공식 문서에 언급 없음(미확인) |
| Haiku 5.5 한국어 품질(SC-003/004) | 구현 후 표본 평가로 확인, 미달 시 Sonnet 5.5 |
| TanStack Query 장기(수 주) 오프라인 캐시 보존 | 구현 단계에서 검증 |
