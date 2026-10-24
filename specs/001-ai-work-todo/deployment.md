# 배포·도메인·비밀값 설계

**Plan**: [plan.md](./plan.md) | **Research**: [research.md](./research.md) | **작성일**: 2026-10-24 (후속 결정 반영)

캡틴 결정: 앱은 **캡틴 1명만** 쓰고, 주소는 캡틴 소유 도메인 `jinjinstar` 의 **서브도메인**이며, AI는 캡틴의 Anthropic API 키(월 크레딧)를 쓴다.
도메인은 캡틴 소유의 **`jinjinstar.com`** 으로 확정되었다. 서브도메인 라벨 `<sub>` 는 캡틴이 아직 정하지 않았으므로 **자리표시자**로 두고, 임의로 지어내지 않는다.

| 자리표시자 | 뜻 | 정하는 사람 |
|-----------|----|-------------|
| `<sub>` | 서브도메인 라벨 (예: 앱 이름) | 캡틴 |
| `<APP_HOST>` | `<sub>.jinjinstar.com` — 앱의 유일한 정식 주소 | — (`<sub>` 만 캡틴이 정하면 됨) |
| `<pages-project>` | Cloudflare Pages 프로젝트 이름 | 구현 시 정함 |
| `<ref>` | Supabase 프로젝트 참조 ID (`<ref>.supabase.co`) | Supabase 가 발급 |

> **서브도메인은 한 번 정하면 바꾸지 않는 게 좋다.** PWA 설치와 브라우저 저장소(IndexedDB, 로그인 세션)는 **출처(origin = 호스트 이름)** 에 묶여서, 호스트를 바꾸면 홈 화면에 추가한 앱을 다시 설치하고 다시 로그인해야 한다.

## 0. 루트 도메인 `jinjinstar.com` 은 건드리지 않는다

`jinjinstar.com` 은 캡틴의 도메인이며, 이 앱 때문에 **루트(apex)와 기존 서브도메인, 메일 레코드는 바뀌지 않는다.**

- **추가하는 DNS 레코드는 딱 하나**: 선택한 서브도메인 라벨에 대한 `CNAME <sub> → <pages-project>.pages.dev`. 그 외 레코드의 추가·수정·삭제는 없다.
- **루트(`jinjinstar.com`, `www` 등)의 `A`/`AAAA`/`CNAME`/`ALIAS`는 그대로** 둔다. 루트를 Cloudflare 로 옮기지 않으며 **네임서버(NS) 변경도 하지 않는다**(서브도메인만 붙일 때는 필요 없음 — [Cloudflare 문서](https://developers.cloudflare.com/pages/configuration/custom-domains/), 2026-10-24; 루트 도메인을 Pages 에 붙일 때만 Cloudflare 네임서버가 필요).
- **다른 기존 서브도메인의 레코드는 그대로**. `<sub>` 가 이미 존재하는 라벨과 겹치지 않는지 만들기 전에 DNS 목록에서 확인한다(같은 이름에 다른 레코드 유형이 있으면 CNAME 을 만들 수 없거나 기존 서비스를 덮어쓴다).
- **메일 관련 레코드(`MX`, SPF/DKIM/DMARC 용 `TXT` 등)는 그대로**. 이 앱의 DNS 작업에서 메일 레코드를 만지지 않는다. (선택 사항인 사용자 지정 SMTP 를 나중에 쓰게 되면 그때 별도로 결정하며, 기존 메일 레코드를 바꾸지 않고 서브도메인용 레코드를 추가하는 방식을 먼저 검토한다.)
- **`CAA` 레코드**: 이미 있는 경우에만 Cloudflare 인증서 발급 기관을 허용하는 레코드를 **추가**해야 할 수 있다(기존 레코드를 지우지 않음). 없으면 아무 작업도 없다.
- **쿠키/세션 격리**: 앱은 상위 도메인 쿠키(`Domain=.jinjinstar.com`)를 쓰지 않으므로 루트나 다른 서브도메인과 로그인 상태를 공유하지도, 영향을 주지도 않는다.
- 작업 전에 현재 DNS 레코드 목록을 캡틴이 한 번 내보내거나 캡처해 두면 되돌릴 때 안전하다.

## 1. 구성

```text
브라우저 / 홈 화면 앱
   │  https://<APP_HOST>/            (정적 파일, Cloudflare Pages)
   │  https://<ref>.supabase.co/…    (인증·데이터·Edge Function)
   ▼
Supabase ──(서버에서만)──▶ Anthropic API   ← API 키는 Supabase 비밀값에만 존재
```

- 앱 주소(`<APP_HOST>`)는 **정적 프론트엔드만** 제공한다. 서버 코드는 없다.
- 백엔드 주소는 Supabase 기본 주소(`<ref>.supabase.co`)를 그대로 쓴다. Supabase 의 커스텀 도메인 기능은 쓰지 않는다(필요 없고, 이번에 가격을 조사하지 않았다).
- 브라우저는 AI 제공자에게 **직접 연결하지 않는다**. 그래서 CSP `connect-src` 에 AI 제공자 주소가 없다(아래 5절).

## 2. 호스팅 (Cloudflare Pages)와 HTTPS

공식 문서 요약 ([Cloudflare Pages: Custom domains](https://developers.cloudflare.com/pages/configuration/custom-domains/), 2026-10-24 확인):

- 서브도메인은 Pages 프로젝트의 **Custom domains → Set up a domain** 에서 추가하고, DNS 공급자에 `CNAME  <sub>.jinjinstar.com  →  <pages-project>.pages.dev` 를 만든다.
- **DNS 가 Cloudflare 에 있지 않아도 된다**("사이트가 Cloudflare zone 일 필요 없음"). 이 경우 CNAME 을 직접 만들어야 하고, **Pages 에서 도메인을 먼저 추가하지 않고 CNAME 만 만들면 522 오류**가 난다. DNS 가 Cloudflare 에 있으면 CNAME 이 자동 생성된다.
- 서브도메인만 쓰므로 **루트(apex) 도메인의 네임서버를 Cloudflare 로 옮길 필요가 없다.** (루트 도메인을 Pages 에 붙이려면 Cloudflare 네임서버가 필요하지만 우리는 해당 없음.)
- `CAA` 레코드를 쓰고 있다면 Cloudflare 가 인증서를 발급할 수 있게 허용해야 한다(문서에 나온 발급 기관: Let's Encrypt, Google Trust Services 계열, SSL.com — `issue`와 `issuewild` 모두). CAA 가 없으면 아무것도 안 해도 된다.
- 인증서 발급·갱신 절차 자체는 해당 문서에 자세히 나와 있지 않다 → **첫 배포 때 `https://<APP_HOST>` 에 자물쇠가 뜨는지 확인**하고, HTTP 접속이 HTTPS 로 넘어가는지도 확인한다(미확인 항목).
- `*.pages.dev` 주소는 기본으로 계속 열려 있다(문서: "disable 하지 않는 한"). 출처가 둘이면 로그인 세션·설치가 갈라지므로 **`*.pages.dev` 접근은 끄거나 정식 주소로 리다이렉트**한다.
- 미리보기(Preview) 배포 주소는 Supabase 로그인 허용 목록에 넣지 않는다. 미리보기에서는 로그인 없이 화면만 확인하고 기능 검증은 로컬에서 한다.
- 한도(무료): 빌드 월 500회, 파일 20,000개, 파일당 25 MiB ([Pages Limits](https://developers.cloudflare.com/pages/platform/limits/), 2026-10-24). 우리 앱에는 충분.

## 3. 인증 리다이렉트 URL (Supabase Auth)

Supabase 는 **Site URL**(기본 복귀 주소)과 **Redirect URLs 허용 목록**을 쓴다. `redirectTo` 는 목록과 일치해야 하고, 프로덕션에는 와일드카드보다 **정확한 경로**를 쓰라고 권한다 ([Supabase Docs: Redirect URLs](https://supabase.com/docs/guides/auth/redirect-urls), 2026-10-24 확인).

| 설정 | 값 |
|------|----|
| Site URL (운영 프로젝트) | `https://<APP_HOST>/` |
| Redirect URLs (운영) | `https://<APP_HOST>/auth/callback` 한 줄 (정확히 일치) |
| 로컬 개발 | 로컬 Supabase 의 설정 파일(`auth.additional_redirect_urls`)에 `http://localhost:5173/**` — **운영 프로젝트 목록에는 넣지 않음** |

- **가입 닫기**: Supabase Auth 설정에서 새 사용자 가입을 끄고, 캡틴 계정은 대시보드에서 직접 초대/생성한다(1인 전용). 이후 일반 가입 경로는 없다.
- **로그인 방식**: 이메일로 받은 **6자리 코드를 앱에 입력(OTP)** 을 기본으로 제안한다. 매직링크는 휴대폰에서 링크를 누르면 설치된 앱이 아니라 브라우저에서 열려, 앱 쪽 세션이 비어 있을 수 있기 때문이다(iOS 동작은 **미확인**, 실기기 검증 대상).
- **이메일 발송 한도**: Supabase 기본 메일 서비스는 **시간당 2통**, 그리고 **프로젝트 팀 구성원 주소로만** 보낸다 ([Supabase Docs: SMTP](https://supabase.com/docs/guides/auth/auth-smtp), 2026-10-24 확인). 팀 구성원인 캡틴 1명 용도로는 동작하지만 재시도가 몰리면 막힌다. 불편하면 ① 사용자 지정 SMTP(별도 서비스 필요, 비용·DNS 레코드 추가 가능성) 또는 ② Google 로그인(OAuth)으로 전환한다. 처음에는 기본 메일로 시작하고 세션을 길게 유지(자동 갱신)해 로그인 횟수를 줄인다.
- 세션은 `localStorage` 에 저장되므로 **상위 도메인 쿠키(`Domain=.jinjinstar.com`)는 쓰지 않는다.** 서브도메인이 서로의 세션을 보지 못하게 한다.

## 4. PWA 범위와 매니페스트

전용 서브도메인이므로 사이트 전체가 앱이다.

| 항목 | 값 |
|------|----|
| `scope` | `/` |
| `start_url` | `/` (쿼리 없이; 필요하면 `/?source=pwa` 로 설치 실행 구분) |
| `id` | `/` (설치 정체성을 경로로 고정; 호스트가 바뀌면 새 앱이 됨) |
| `display` | `standalone` |
| `lang` / `dir` | `ko` / `ltr` |
| `name` / `short_name` | 화면 디자인 단계에서 확정(제안: 서브도메인과 같은 이름) |
| `theme_color` / `background_color` | 화면 디자인 단계에서 확정 |
| `icons` | 192×192, 512×512, maskable 512×512 (PNG) |
| 서비스 워커 | `/sw.js`, 범위 `/` (루트에서 서비스) |

- 앱 셸만 프리캐시하고 `/auth/callback` 같은 로그인 경로와 Supabase 요청은 캐시하지 않는다(네트워크 우선).
- SPA 라우팅(새로고침 시 `/standups/…` 같은 경로): Pages 가 존재하지 않는 경로를 `index.html` 로 돌려주는지 첫 배포에서 확인한다(미확인).
- 서비스 워커 업데이트: 새 버전이 있으면 "새 버전이 있어요 — 새로고침" 안내(화면 디자인 단계).

## 5. 보안 헤더·CORS (정적 호스팅의 `_headers` 설정 등으로 적용 — 구현 시 검증)

- `Content-Security-Policy`: `default-src 'self'; connect-src 'self' https://<ref>.supabase.co; img-src 'self' data:; frame-ancestors 'none'` 등. **AI 제공자 주소는 넣지 않는다.**
- `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`.
- Edge Function `ai`/`delete-account` 의 CORS 는 `Access-Control-Allow-Origin: https://<APP_HOST>` **한 곳만**(로컬 개발 환경에서만 localhost 추가). 환경 변수 `ALLOWED_ORIGIN`.
- 데이터 API(PostgREST)는 JWT + RLS 로 보호된다. 가입이 닫혀 있고 캡틴 계정 하나뿐이므로 JWT 를 가진 주체는 캡틴뿐이다.

## 6. 비밀값 (Anthropic API 키)

> **키는 서버 비밀값으로만 존재한다.** 저장소, 로그, 브라우저 번들, PR, 채팅 어디에도 두지 않는다.

- **발급**: 캡틴이 Anthropic Console 에서 이 앱 전용 **워크스페이스**를 만들고 그 워크스페이스에서 키를 발급한다(워크스페이스마다 지출 한도를 걸 수 있고, 기본 워크스페이스에는 지출 한도를 못 건다 — [Rate limits 문서](https://platform.claude.com/docs/en/api/rate-limits), 2026-10-24 확인).
- **보관 위치**: **Supabase Edge Function 비밀값**(운영: 대시보드의 Edge Function Secrets 화면 또는 캡틴 터미널에서 `supabase secrets set`; 이름 `ANTHROPIC_API_KEY`). 함수는 `Deno.env.get` 으로 읽고, **새 비밀값은 재배포 없이 바로 반영**된다. 비밀값 생성·삭제에는 프로젝트 Owner/Administrator 권한이 필요하다 ([Supabase Docs: Secrets](https://supabase.com/docs/guides/functions/secrets), 2026-10-24 확인).
- **로컬 개발**: `supabase/functions/.env`(문서가 `.gitignore` 에 넣으라고 경고) — 저장소에는 값 없는 `.env.example` 만 둔다.
- **브라우저로 가는 값**: `VITE_` 로 시작하는 공개 값(`VITE_SUPABASE_URL`, 공개용 anon/publishable 키)뿐. Supabase 의 secret/service 키도 **절대 브라우저로 보내지 않는다**(RLS 를 우회하므로 Edge Function 안에서만).
- **방어선**: ① `.gitignore` 에 `.env*`(단 `.env.example` 제외) ② CI 에서 비밀값 스캔 ③ 빌드 산출물(`dist`)에 키 패턴(`sk-ant-`)이 없는지 확인 ④ 함수 로그에 요청 헤더·본문을 남기지 않음 ⑤ 키가 노출된 의심이 들면 Console 에서 즉시 폐기·재발급 후 Supabase 비밀값 교체.
- **캡틴에게 값을 묻지 않는다**: 구현 단계에서도 키를 채팅/PR/이슈에 붙여 넣게 하지 않는다. 캡틴이 자기 터미널에서 직접 `supabase secrets set` 을 하거나 대시보드 입력란에 넣는다.

## 7. 캡틴이 DNS·계정 쪽에서 해야 할 일 (순서대로)

1. **이름 정하기**: `<sub>` 라벨만 정하면 `<APP_HOST>` = `<sub>.jinjinstar.com` 이 확정된다 (한 번 정하면 변경 비권장).
2. **DNS 호스팅 위치 확인**: `jinjinstar.com` 의 DNS 가 Cloudflare 인지, 등록 기관/다른 서비스인지 확인.
3. **Cloudflare 계정**에서 Pages 프로젝트 생성(저장소 연결). 루트 도메인을 Cloudflare 로 옮길 필요는 없다.
4. Pages 프로젝트 **Custom domains → `<APP_HOST>` 추가** (이 순서가 먼저).
5. DNS 가 Cloudflare 가 아니면 **CNAME 레코드 1개**를 만든다: 이름 `<sub>`(전체 이름 `<APP_HOST>`), 값 `<pages-project>.pages.dev`. DNS 가 Cloudflare 면 자동.
6. `CAA` 레코드가 있으면 Cloudflare 인증서 발급 기관을 허용(없으면 건너뜀).
7. 도메인 상태가 **Active** 가 되고 `https://<APP_HOST>` 가 열리는지 확인.
8. `*.pages.dev` 주소는 끄거나 정식 주소로 보내기 설정.
9. **Supabase**: 프로젝트 생성 → Auth 설정에서 가입 끄기, Site URL 과 Redirect URL 입력(3절), 캡틴 계정 생성.
10. **Anthropic Console**: 크레딧을 청구에 연결(아래 research D5), 이 앱 전용 워크스페이스 생성, 워크스페이스·조직 지출 한도 설정, 키 발급 후 **Supabase 비밀값에 직접 입력**.
11. (선택) 이메일 로그인 한도가 불편하면 사용자 지정 SMTP 또는 Google 로그인을 설정 — 이때 DNS 에 메일 인증 레코드(SPF/DKIM 등)가 추가로 필요할 수 있다.

추가로 필요한 **DNS 레코드는 (5)의 CNAME 1개** 이고, 나머지는 계정/대시보드 설정이다.
