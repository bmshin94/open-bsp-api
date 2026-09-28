# OpenBSP API 분석 정리 (한국어)

> 이 문서는 Claude Code 세션에서 진행한 OpenBSP API 저장소 전수조사 및
> 활용/수익화 논의를 정리한 기록입니다.

## 저장소 정보

| 항목                  | 내용                                                                    |
| --------------------- | ----------------------------------------------------------------------- |
| 분석 대상 (내 포크)   | https://github.com/bmshin94/open-bsp-api                                |
| 원본(업스트림) 저장소 | https://github.com/matiasbattocchia/open-bsp-api                        |
| 동반 UI 저장소        | https://github.com/matiasbattocchia/open-bsp-ui                         |
| n8n 커뮤니티 노드     | https://github.com/matiasbattocchia/n8n-nodes-openbsp                   |
| 호스팅 데모           | https://web.openbsp.dev                                                 |
| 원격 MCP 엔드포인트   | https://nheelwshzbgenpavwhcy.supabase.co/functions/v1/mcp               |
| 라이선스              | Unlicense (퍼블릭 도메인)                                               |
| 스택                  | Deno + Postgres + Supabase (Edge Functions / Auth / Storage / Realtime) |
| 규모                  | Edge Function 17개, 마이그레이션 110개, 코드 약 2.3만 줄                |

---

## 1. 이게 뭐하는 물건인가

**OpenBSP API = 오픈소스 WhatsApp / Instagram 비즈니스 메시징 플랫폼.**

BSP(Business Solution Provider)는 Meta가 공인한 "기업용 메신저 대행사"를 뜻한다.
원래 Twilio / 360dialog / Infobip 등이 유료로 제공하던 영역을 통째로
오픈소스(퍼블릭 도메인)로 공개한 프로젝트다.

### 폴더 구조

```
open-bsp-api/
├── supabase/
│   ├── functions/     Deno Edge Function 17개 (서버 로직 전부)
│   ├── schemas/       DB 설계도 (여기를 고치면 마이그레이션 생성)
│   ├── migrations/    적용된 DB 변경 이력 110개 (수정 금지)
│   └── seed.sql
├── plugin/            Claude Code 플러그인 (MCP 서버 + 스킬)
│   ├── server.ts          로컬 stdio MCP 서버
│   ├── auth.ts            구글 OAuth 로그인 / 세션 영속화
│   ├── api-reference.ts   MCP 리소스로 노출되는 API 레퍼런스
│   └── skills/configure/  /openbsp:config 스킬
├── .claude-plugin/    플러그인 마켓플레이스 매니페스트
├── .claude/           프로젝트 훅 및 세팅
├── openapi.json       PostgREST OpenAPI 스펙 (81KB)
├── README.md          42KB 대형 문서
├── AUTH.md            인증 헤더 3종 설명
├── INTEGRATING.md     서드파티 온보딩 / 자격증명 캡처
├── MIGRATING_FROM_TWILIO.md
├── MIGRATING_FROM_WHATSAPP_WEB_JS.md
├── IDEAS.md / TODO.md 미래 계획 (수익화 힌트 포함)
└── app-review/        Meta 앱 심사 제출용 영상 4개
```

### 핵심 아키텍처 — "DB가 곧 메시지 버스"

별도의 백엔드 API 서버가 없다. **테이블 자체가 API(PostgREST)** 이고, **DB
트리거가 이벤트 버스** 역할을 한다.

```
① 고객이 WhatsApp 메시지 전송
      ↓
② Meta 서버 → whatsapp-webhook 함수 호출
      ↓
③ messages 테이블에 INSERT            ← 모든 흐름의 시작점
      ↓
④ [INSERT 트리거] → agent-client 함수 자동 호출
      ↓
⑤ AI가 대화 맥락 구성 → 응답 생성 → messages 테이블에 INSERT
      ↓
⑥ [아웃고잉 트리거] → whatsapp-dispatcher 함수 자동 호출
      ↓
⑦ Meta Cloud API로 발송 → 고객 단말 수신
```

즉 **"메시지 전송 = DB row 하나 INSERT"**. 그래서 Supabase SDK가 있는 언어라면
(JS / Python / Dart / PHP 등) 무엇이든 클라이언트가 될 수 있다.

### 지원 채널

- WhatsApp Cloud API (공식, 메인)
- Instagram DM (공식)
- Slack (사내 협업)
- WhatsApp Web (비공식 whatsmeow 브릿지, 개인 계정용)
- Generic 커넥터 (임의 서비스 직접 연결)

### 주요 데이터 모델

| 테이블                    | 역할                                                   |
| ------------------------- | ------------------------------------------------------ |
| `organizations`           | 테넌트(회사). 멀티테넌시의 축                          |
| `organizations_addresses` | 조직이 연결한 전화번호 / 인스타 계정                   |
| `contacts_addresses`      | 주소록 (연결별 연락처 엔트리)                          |
| `conversations`           | 대화방 (direct / group / channel)                      |
| `messages`                | 메시지 본체 (작성자, 타입, 페이로드, 상태, 타임스탬프) |
| `agents`                  | 사람 상담원 또는 AI 에이전트                           |
| `api_keys`                | 조직 범위 API 키                                       |
| `webhooks`                | 아웃바운드 웹훅 구독                                   |
| `conversations_agents`    | 대화 참여자 및 개인별 상태                             |
| `logs`                    | 애플리케이션 레벨 로그                                 |
| `billing.*`               | 과금 스키마 13개 테이블 (이미 구현되어 있음)           |

### AI 에이전트 런타임

`agent-client` (약 4,640줄)가 내장 에이전트를 구동한다.

- 프로토콜: OpenAI Chat Completions / Open Responses
- 내장 툴: MCP 클라이언트, SQL 클라이언트, HTTP 클라이언트, 계산기
- 미디어 해석: 음성 / 이미지 / 영상 / PDF / CSV (`media-preprocessor`)
- 설계 철학: 무거운 에이전트는 외부 서비스로, 이 저장소는 통신 레이어에 집중

### 언제 쓰는가

1. WhatsApp 고객센터 자동화 (24시간 AI 상담)
2. 주문 / 예약 / 배송 알림 발송 (템플릿 메시지)
3. BSP 사업 자체 — 멀티테넌트라 여러 고객사 수용 가능
4. Twilio 등 유료 BSP 비용 절감 (셀프호스팅)
5. n8n / Claude Code 연동 워크플로우

### 나에게 무슨 도움이 되는가

1. 아키텍처 교과서 — "DB 트리거를 이벤트 버스로" 패턴, 주석 품질이 매우 높음
2. Unlicense(퍼블릭 도메인) — 저작권 표기 없이 상용 재배포/판매 가능
3. 과금 시스템이 이미 구현됨 — Stripe만 붙이면 SaaS 성립
4. MCP 서버 + Claude Code 플러그인 실전 구현체
5. 한국 시장 갭 — 국내 WhatsApp BSP 경쟁자가 거의 없음 (해외 대상 비즈니스 타겟)

---

## 2. 쉬운 설명 버전

### 비유로 이해하기

- **WhatsApp** = 카카오톡의 세계 버전 (전 세계 약 30억 사용자)
- **기업이 함부로 못 보냄** → Meta가 "공식 기업용 문"을 따로 운영
- **BSP** = 그 문을 대신 열어주는 대행사 (Twilio 등, 건당 수수료 과금)
- **OpenBSP** = 그 대행사 시스템을 통째로 무료 공개한 것

| 기존 방식                        | OpenBSP                          |
| -------------------------------- | -------------------------------- |
| 라면을 사 먹는다 (Twilio 구독료) | 라면 공장을 통째로 받는다 (무료) |
| 레시피 비공개                    | 레시피 공개 + 판매 허용          |
| 커스텀 불가                      | AI 자동응답까지 내장             |

### 우체국 비유로 본 동작 흐름

```
고객: "배송 언제 와요?"
   ↓
whatsapp-webhook     = 우편함 (편지 받는 곳)
   ↓
messages 테이블       = 편지 보관함  ← 여기 넣으면 나머지는 자동
   ↓
agent-client         = AI 비서 (읽고 답장 작성)
   ↓
messages 테이블       = 답장도 여기에
   ↓
whatsapp-dispatcher  = 집배원 (배달)
   ↓
고객: "내일 도착합니다" 수신
```

### 멀티테넌트 = 쉐어하우스

한 건물(서버 하나)에 여러 회사가 입주하지만 서로의 방은 절대 볼 수 없다.
Postgres의 RLS(Row Level Security)가 자물쇠 역할을 한다. → 고객사를 100개 받아도
서버는 하나면 된다. 수익화의 핵심 근거.

---

## 3. 질문별 상세 답변

### Q1. 설치 및 사용법

**경로 A — 호스팅 버전 사용 (가장 빠름)**

1. web.openbsp.dev 가입 (구글 로그인)
2. Integrations → WhatsApp 연결
3. Settings → API Keys 발급

메시지 전송:

```bash
curl -X POST 'https://nheelwshzbgenpavwhcy.supabase.co/rest/v1/messages' \
  -H 'apikey: <PUBLISHABLE_KEY>' \
  -H 'api-key: <API_KEY>' \
  -H 'Content-Type: application/json' \
  -d '{
    "organization_id": "<ORG_ID>",
    "organization_address": "<PHONE_NUMBER_ID>",
    "conversation_address": "821012345678",
    "service": "whatsapp",
    "content": { "version":"1", "type":"text", "kind":"text", "text":"안녕!" }
  }'
```

무료 할당량: 월 5,000건 / 스토리지 1GB / AI 크레딧 $1(1회성)

**경로 B — 셀프호스팅 (약 15분, 권장)**

1. 저장소 Fork
2. Supabase 프로젝트 생성
3. Supabase Dashboard → Project Settings → Integrations → GitHub 연동 (Working
   directory `.`, Production branch `main`, Deploy to production ON)
4. Vault 시크릿 2개 등록
   - `edge_functions_url` = `https://{PROJECT_ID}.supabase.co/functions/v1`
   - `edge_functions_token` = `SUPABASE_SERVICE_ROLE_KEY`
5. `main`에 커밋 1회 → 최초 배포 트리거

이후 push만 하면 마이그레이션 + Edge Function이 자동 배포된다.

**경로 C — 로컬 개발 (Docker 필요)**

```bash
npx supabase start
npx supabase migration up
npx supabase functions serve
npx supabase gen types typescript --local > supabase/functions/_shared/db_types.ts
```

CI와 동일한 로컬 검사 (반드시 패키지 디렉터리 안에서 실행):

```bash
deno fmt --check
cd supabase/functions && deno lint && deno check .
cd plugin && deno lint && deno check .
```

**경로 D — Claude Code 플러그인**

```
/plugin marketplace add matiasbattocchia/open-bsp-api
/plugin install openbsp@matiasbattocchia-open-bsp-api
/openbsp:config login
/openbsp:config contacts add 821012345678
```

### Q2. 플러그인인가, 스킬인가, MCP인가

**넷 다 해당되지만, 본체는 백엔드 플랫폼이다.**

```
본체 = 멀티테넌트 메시징 백엔드 (Deno Edge Functions + Postgres)   ← 약 95%
  └ 부가물
      ① Claude Code 플러그인   plugin/.claude-plugin/plugin.json
      ② MCP 서버 2종
           - 로컬 stdio : plugin/server.ts
           - 원격 HTTP  : supabase/functions/mcp/index.ts
      ③ Skill                  plugin/skills/configure/SKILL.md
      ④ Marketplace            .claude-plugin/marketplace.json
```

원격 MCP 서버가 제공하는 툴:

| 툴                                  | 기능                                      |
| ----------------------------------- | ----------------------------------------- |
| `list_accounts`                     | 연결된 WhatsApp 계정 목록                 |
| `list_conversations`                | 최근 활성 대화 목록                       |
| `fetch_conversation`                | 특정 연락처 메시지 + 24시간 윈도우 상태   |
| `search_contacts`                   | 이름 / 번호로 연락처 검색                 |
| `send_message`                      | 텍스트 또는 템플릿 발송 (24h 윈도우 강제) |
| `list_templates` / `fetch_template` | 템플릿 조회                               |

Claude Code 연결:

```bash
claude mcp add --transport http openbsp \
  https://nheelwshzbgenpavwhcy.supabase.co/functions/v1/mcp
```

### Q3. API 토큰을 사용해야 하나

필요하다. 용도별로 구분된다.

| 토큰                                                                   | 발급처                  | 필수 | 용도                     |
| ---------------------------------------------------------------------- | ----------------------- | ---- | ------------------------ |
| `META_APP_ID`                                                          | developers.facebook.com | 필수 | Meta 앱 식별             |
| `META_APP_SECRET`                                                      | 동일                    | 필수 | 웹훅 서명 검증           |
| `META_SYSTEM_USER_ID`                                                  | business.facebook.com   | 필수 | 시스템 유저              |
| `META_SYSTEM_USER_ACCESS_TOKEN`                                        | 동일                    | 필수 | WhatsApp 발송 권한       |
| `WHATSAPP_VERIFY_TOKEN`                                                | 직접 지정               | 필수 | 웹훅 검증                |
| `INSTAGRAM_APP_ID` / `INSTAGRAM_APP_SECRET` / `INSTAGRAM_VERIFY_TOKEN` | Meta                    | 선택 | Instagram 사용 시        |
| `SUPABASE_ANON_KEY`                                                    | Supabase                | 필수 | 공개 키                  |
| `SUPABASE_SERVICE_ROLE_KEY`                                            | Supabase                | 필수 | 서버 전용, 노출 금지     |
| OpenBSP `API_KEY`                                                      | OpenBSP UI              | 필수 | 내 앱 → OpenBSP          |
| LLM Provider Key                                                       | OpenAI 등               | 선택 | 자체 키로 AI 크레딧 우회 |
| `WHATSAPP_WEB_URL` / `WHATSAPP_WEB_TOKEN`                              | 직접 지정               | 선택 | 비공식 브릿지            |

인증 헤더 3종 (AUTH.md):

| 헤더            | 담는 값                         | 소비 주체                                           |
| --------------- | ------------------------------- | --------------------------------------------------- |
| `apikey`        | Supabase anon / publishable key | Kong → PostgREST (Postgres role 결정)               |
| `Authorization` | `Bearer <token>`                | PostgREST(JWT 검증) 또는 Edge Function(그대로 전달) |
| `api-key`       | OpenBSP API 키                  | RLS의 `get_authorized_orgs()`                       |

주의: PostgREST 호출 시 `Authorization: Bearer <OpenBSP키>`를 넣으면 JWT가
아니므로 PGRST301로 거부된다. REST는 `api-key`, Edge Function은 `Authorization`.

예외: Claude Code 플러그인은 구글 OAuth만으로 동작하며 API 키가 필요 없다.
세션은 `~/.claude/channels/openbsp/session.json`에 0600 권한으로 저장되고 자동
갱신된다.

### Q4. 왜 GitHub에서 유명한가

(별 개수는 이 세션에서 업스트림 저장소 조회 권한이 없어 확인하지 못했다. 아래는
구조적 근거에 따른 분석이다.)

1. **유료 SaaS 킬러** — Twilio / 360dialog / Infobip의 핵심 비즈니스를 대체
2. **Unlicense(퍼블릭 도메인)** — 저작권 표기 의무조차 없어 리브랜딩 판매가
   합법. 이 규모의 프로젝트가 퍼블릭 도메인인 경우는 매우 드물다
3. **문서 품질** — README 42KB, OpenAPI 81KB, 경쟁 제품 이주 가이드 2종 (Twilio
   12KB, whatsapp-web.js 19KB, 호환성 매트릭스 포함), 데모 영상, 아키텍처
   다이어그램. "경쟁 제품 이주 가이드"는 검색 유입에 매우 강하다
4. **AI 에이전트 트렌드 적중** — MCP 서버(원격/로컬), Claude Code 플러그인, n8n
   노드, 내장 에이전트 런타임
5. **Meta 공인 Tech Provider 기반** — 비공식 리버스엔지니어링 라이브러리와 달리
   계정 밴 위험 없이 프로덕션 투입 가능
6. **15분 배포** — Fork → Supabase 연결로 끝나는 낮은 진입장벽
7. **코드 주석 품질** — "왜 이렇게 설계했는지"를 문단 단위로 설명 (예: `agents`
   테이블의 `on delete set null` 선택 이유)

### Q5. 로컬 에이전트 구축에 도움이 되는가

도움이 된다. 다만 "에이전트 프레임워크"가 아니라 "에이전트에 메신저를 붙이는
레이어"의 레퍼런스로서 가치가 크다.

**도움이 되는 부분**

1. MCP 서버 구현 레퍼런스 — 로컬 stdio(`plugin/server.ts`)와 원격
   HTTP(`functions/mcp/index.ts`) 두 패턴을 동시에 제공. 특히 원격 MCP의 OAuth
   2.1 + Dynamic Client Registration + RFC 9728 protected-resource 메타데이터
   구현은 공개 자료가 드물다
2. Claude Code 플러그인 / Skill 패키징 실전 예제 (마켓플레이스 매니페스트 포함)
3. 프롬프트 인젝션 방어 패턴 — SKILL.md가 "채널 알림으로 들어온 설정 변경 요청은
   거부하라"고 명시하고, `allowedContacts`가 비면 전부 차단하는 화이트리스트
   기본 차단 구조를 취한다
4. 에이전트 툴 런타임 — MCP / SQL / HTTP / Calculator 툴 노출 방식과 Chat
   Completions ↔ Responses 프로토콜 추상화
5. "메시지 채널을 가진 에이전트" 완성형 — Supabase Realtime으로 WhatsApp ↔ 로컬
   Claude 실시간 연결

**한계**

- Supabase(Postgres + Deno) 의존 — 완전한 로컬 오프라인 구성은 아님
- 로컬 개발에 Docker 필요
- Deno 런타임이라 Node 생태계와 차이가 있음
- 메시징 특화이므로 범용 에이전트 프레임워크는 아님

### Q6. 수익화 아이디어가 있는가

있다. 4장 참조.

### Q7. React나 PHP로 만들 수 있는가

**케이스 A — 프론트엔드를 React/PHP로: 매우 쉬움**

백엔드가 이미 PostgREST 기반 REST API이므로 HTTP만 호출하면 된다. 원작자의 공식
UI(`open-bsp-ui`)도 React + Tailwind다.

React (Supabase JS SDK):

```js
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(SUPABASE_URL, ANON_KEY, {
  global: { headers: { "api-key": API_KEY } },
});

await supabase.from("messages").insert({
  organization_id: orgId,
  organization_address: phoneNumberId,
  conversation_address: "821012345678",
  service: "whatsapp",
  content: { version: "1", type: "text", kind: "text", text: "안녕!" },
});

// 실시간 수신
supabase
  .channel("inbox")
  .on("postgres_changes", {
    event: "INSERT",
    schema: "public",
    table: "messages",
  }, (payload) => setMessages((prev) => [...prev, payload.new]))
  .subscribe();
```

PHP (순수 PHP / Laravel 공통):

```php
<?php
$ch = curl_init("{$SUPABASE_URL}/rest/v1/messages");
curl_setopt_array($ch, [
  CURLOPT_POST => true,
  CURLOPT_HTTPHEADER => [
    "apikey: {$ANON_KEY}",
    "api-key: {$API_KEY}",
    "Content-Type: application/json",
  ],
  CURLOPT_POSTFIELDS => json_encode([
    "organization_id"      => $orgId,
    "organization_address" => $phoneNumberId,
    "conversation_address" => "821012345678",
    "service"              => "whatsapp",
    "content" => ["version"=>"1","type"=>"text","kind"=>"text","text"=>"안녕!"],
  ]),
  CURLOPT_RETURNTRANSFER => true,
]);
$res = curl_exec($ch);
```

PHP는 WebSocket이 약하므로 수신은 Realtime 대신 웹훅 등록 방식이 정석이다.
`POST /rest/v1/webhooks`로 엔드포인트를 등록하면 새 메시지마다 POST가 온다.

```php
$body = json_decode(file_get_contents("php://input"), true);
if ($body["entity"] === "messages" && $body["data"]["sender_address"]) {
    // 고객이 보낸 메시지 → 응답 로직
}
```

**케이스 B — 백엔드 전체를 재구현: 가능하지만 비효율**

|           | React(Next.js) 재작성           | PHP(Laravel) 재작성              |
| --------- | ------------------------------- | -------------------------------- |
| 난이도    | 중상                            | 중                               |
| 예상 기간 | 2~4개월                         | 2~3개월                          |
| 장점      | TS 코드 이식 용이 (Deno → Node) | 국내 호스팅 저렴, 인력 확보 쉬움 |
| 단점      | Realtime / RLS 직접 구현        | TS 로직 전면 재작성              |

직접 구현해야 하는 것: RLS 멀티테넌트 격리, DB 트리거 이벤트 버스(→ Redis/SQS
큐), Realtime(→ Pusher / Socket.io / Laravel Reverb), PostgREST 자동 API (→
컨트롤러 수작업).

**권장 구성**

```
백엔드 = OpenBSP 그대로 (Supabase)
        ↕ REST / Realtime / Webhook
프론트 = React 또는 PHP 자유 선택
```

수익은 프론트 / UX / 도메인 특화에서 발생하므로 백엔드 재작성은 권하지 않는다.

| 목적            | 권장 스택                                  |
| --------------- | ------------------------------------------ |
| 빠른 MVP        | Next.js + Supabase JS SDK                  |
| 국내 SI 납품    | Laravel + 웹훅 + Blade                     |
| 관리자 대시보드 | React + shadcn/ui                          |
| 모바일 앱       | React Native / Flutter (Supabase SDK 지원) |

---

## 4. 수익화 아이디어 상세

전제: Unlicense(퍼블릭 도메인)이므로 포크 → 리브랜딩 → 상용 판매가 합법이다.

### 아이디어 1. 한국형 WhatsApp BSP SaaS

- **컨셉**: 한국 기업이 해외 고객과 WhatsApp으로 소통하게 해주는 국내 SaaS
- **타겟**: 크로스보더 이커머스, 인바운드 여행/의료관광, 수출 제조업, 유학원
- **가격 예시**

  | 플랜       | 월 요금  | 포함                           |
  | ---------- | -------- | ------------------------------ |
  | Starter    | ₩49,000  | 3,000건, 번호 1개, AI 500건    |
  | Growth     | ₩149,000 | 20,000건, 번호 3개, AI 5,000건 |
  | Business   | ₩399,000 | 무제한, 번호 10개, 전담 지원   |
  | Enterprise | 별도     | 온프레미스 + SLA               |

- **근거**: `billing` 스키마 13개 테이블(products / tiers / plans / usage /
  ledger / invoices / payments)이 이미 구현돼 있고, TODO.md에 남은 항목이
  "Stripe checkout 연동" 수준이다
- **리스크**: Meta Tech Provider 심사 필요 (`app-review/`에 심사용 영상 4개
  존재)

### 아이디어 2. 버티컬 SaaS (업종 특화)

- **예약 리마인더 SaaS** — 병원 / 치과 / 한의원 / 미용실 / 학원 / PT샵. 예약 D-1
  자동 알림으로 노쇼 방지. 세일즈 포인트: "노쇼 1건 손실 > 월 구독료"
- **글로벌 레스토랑 예약봇** — 외국인 관광객 다국어 AI 응대 (명동/강남/제주)
- **물류·포워딩 추적봇** — B/L 번호 조회 → `agent-client`의 SQL 툴로 즉시 구현
  가능
- **의료관광 코디네이터** — 중동/동남아 환자 상담 자동화, 객단가 최고

### 아이디어 3. AI 상담봇 구축 대행 (SI)

현금 회수가 가장 빠른 모델.

| 서비스                         | 단가            | 기간    |
| ------------------------------ | --------------- | ------- |
| 기본 구축 (번호 연결 + FAQ 봇) | 300~800만원     | 2주     |
| 고급 구축 (ERP / DB 연동)      | 1,500~3,000만원 | 1~2개월 |
| 월 운영비                      | 30~100만원/월   | 지속    |
| AI 튜닝 / 리포팅               | 50만원/월       | 지속    |

원가는 Supabase 요금 + LLM 토큰비 수준이며, 코드가 이미 완성돼 있어
커스터마이징만 하면 된다. 월 운영비가 MRR로 쌓인다.

### 아이디어 4. Claude Code / n8n 플러그인 생태계

| 상품                                       | 가격   |
| ------------------------------------------ | ------ |
| 한국어 특화 스킬팩 (존댓말 / 응대 톤 튜닝) | $19/월 |
| 팀 협업 확장 (다중 상담원 배정, 라우팅)    | $49/월 |
| 분석 플러그인 (CSAT, 응답시간, 리포트)     | $29/월 |
| n8n 프리미엄 노드                          | $15/월 |

"Claude Code로 WhatsApp 고객센터를 운영한다"는 컨셉 자체가 아직 선점되지 않았다.

### 아이디어 5. 매니지드 호스팅 / 온프레미스

| 상품                                       | 가격          |
| ------------------------------------------ | ------------- |
| 셀프호스팅 대행 설치                       | 200만원 (1회) |
| 매니지드 운영 (모니터링 + 업데이트 + 백업) | 50~150만원/월 |
| 온프레미스 구축 (금융 / 공공 / 병원)       | 3,000만원~    |
| 24/7 SLA 지원                              | 별도          |

데이터 외부 반출이 금지된 조직(병원, 금융, 공공, 대기업 보안망)은 해외 SaaS를 쓸
수 없어 온프레미스가 유일한 선택지다. 객단가가 가장 높다.

### 아이디어 6. 교육 / 콘텐츠

| 상품                                    | 가격        |
| --------------------------------------- | ----------- |
| 온라인 강의 "WhatsApp AI 챗봇 만들기"   | 15만원      |
| 유료 뉴스레터 (해외 메시징 트렌드)      | 1만원/월    |
| 노션 템플릿 + 프롬프트팩                | 5만원       |
| 기업 출강 (AI 고객응대 교육)            | 100만원/일  |
| 유튜브 (Supabase / Deno / MCP 튜토리얼) | 광고 + 리드 |

콘텐츠로 리드를 모아 1~5번 상품으로 전환시키는 퍼널을 설계한다.

### 아이디어 7. 국내 메신저 통합 허브 (하이브리드)

`generic-webhook` / `generic-dispatcher` 커넥터 구조를 활용해 국내 채널을
추가한다.

```
카카오톡 ─┐
네이버톡톡─┤
WhatsApp ─┼─ OpenBSP (Generic Connector) ─→ 통합 인박스 + AI 자동응답
인스타 DM ─┤
텔레그램  ─┤
LINE     ─┘
```

- 수익 모델: 채널 통합 SaaS 월 19~99만원, 채널 추가 건당 과금
- 차별점: 국내 경쟁 서비스는 WhatsApp / Instagram 커버리지가 약하다

### 전략 우선순위

| 순위 | 아이디어                    | 착수 난이도 | 수익 속도 | 성장 천장 |
| ---- | --------------------------- | ----------- | --------- | --------- |
| 1    | SI 구축 대행                | 낮음        | 즉시      | 중        |
| 2    | 버티컬 SaaS (예약 리마인더) | 중간        | 3~6개월   | 높음      |
| 3    | Claude 플러그인 생태계      | 낮음        | 느림      | 중~높음   |
| 4    | 한국형 BSP SaaS             | 높음        | 6~12개월  | 최고      |
| 5    | 매니지드 / 온프레미스       | 중간        | 빠름      | 높음      |
| 6    | 교육 콘텐츠                 | 낮음        | 빠름      | 낮음      |
| 7    | 국내 채널 통합 허브         | 높음        | 느림      | 최고      |

### 로드맵

```
0~1개월    셀프호스팅 배포 → 데모 영상 제작 → 기술 블로그 / 유튜브 1편
1~3개월    SI 대행 1~2건 수주(현금 확보) + 교육 콘텐츠로 리드 수집
3~6개월    버티컬 선택(예: 병원 예약봇) → MVP → 베타 10곳 무료 운영
6~12개월   Stripe 결제 연동 → SaaS 전환 → Meta Tech Provider 신청
12개월~    채널 확장(카카오 / 네이버) → 통합 허브로 진화
```

### 리스크 체크리스트

1. **Meta 정책** — 템플릿 승인, 24시간 서비스 윈도우, 스팸 규제가 엄격하다
2. **국내 WhatsApp 보급률이 낮다** — 반드시 해외 대상 비즈니스를 타겟할 것
3. **개인정보보호법** — 메시지 저장 위치, 수집·이용 동의 절차 확인 필요
4. **TODO.md 미완성 항목** — 웹훅 재시도 / 인보이스 생성 / 결제 연동은 직접 구현
5. **업스트림 추적** — 포크이므로 원본 저장소와 주기적 sync 필요

---

## 부록: 자주 쓰는 명령어

```bash
# CI와 동일한 로컬 검사
deno fmt --check
cd supabase/functions && deno lint && deno check .
cd plugin && deno lint && deno check .

# 로컬 DB
npx supabase start
npx supabase db diff -f <migration_name>     # 스키마 수정 후 마이그레이션 생성
npx supabase migration up --local
npx supabase gen types typescript --local > supabase/functions/_shared/db_types.ts

# Edge Functions 로컬 실행
npx supabase functions serve
```

주의사항

- 적용된 마이그레이션은 절대 수정하지 않는다. 항상 새로 생성한다
- 마이그레이션은 `supabase/schemas/`를 수정한 뒤 `db diff`로 생성한다
- `supabase/functions/_shared/db_types.ts`는 자동 생성 파일이므로 직접 수정 금지
- `deno check`는 반드시 패키지 디렉터리 내부에서 실행한다
