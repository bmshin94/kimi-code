# Kimi Code CLI 전수조사 & 활용 분석

> 이 문서는 Kimi Code CLI 저장소를 전수조사한 결과와, 설치·확장·수익화 관련 논의를 정리한 기록입니다.
>
> - 원본 저장소: <https://github.com/MoonshotAI/kimi-code>
> - 포크 저장소: <https://github.com/bmshin94/kimi-code>
> - 문서: <https://moonshotai.github.io/kimi-code/en/>
> - 라이선스: MIT (`apps/vscode`는 Apache-2.0, `plugins/official/kimi-webbridge`는 Proprietary)
> - 조사 기준 버전: `@moonshot-ai/kimi-code` v2.0.0

---

## 1. 이게 뭔가

**Moonshot AI가 만든 오픈소스 터미널 AI 코딩 에이전트.** 포지션상 Claude Code의 오픈소스 대응작이며,
소스가 전부 공개되어 있다는 점이 가장 큰 차별점이다.

에이전트는 파일을 읽고 고치고, 셸 명령을 실행하고, 웹을 검색하고, 그 결과를 보고 다음 행동을
스스로 정하는 루프를 돈다. Kimi 모델이 기본이지만 다른 프로바이더로 교체할 수 있다.

---

## 2. 저장소 구조 (전수조사)

TypeScript 모노레포. 소스 파일 약 2,000개 이상.

### apps/ — 사용자 진입점

| 경로 | 역할 | TS 파일 수 |
| --- | --- | --- |
| `apps/kimi-code` | CLI / TUI 본체 (`kimi` 명령어) | 339 |
| `apps/kimi-inspect` | kap-server `/api/v1/debug` RPC 웹 인스펙터 | 66 |
| `apps/vscode` | VS Code 확장 (`moonshot-ai.kimi-code` v0.7.5) | 32 |
| `apps/vis` | 세션 / 리플레이 시각 디버깅 도구 | - |
| `apps/kimi-code/dist-web` | 프리빌드 브라우저 UI 번들 (소스는 별도 code-app 저장소) | - |

### packages/ — 엔진 내부

| 패키지 | 역할 | TS 파일 수 |
| --- | --- | --- |
| `agent-core-v2` | DI × Scope 에이전트 엔진 (핵심) | 1,067 |
| `kap-server` | REST + WebSocket 서버 (`/api/v1`, `/api/v1/ws`) | 217 |
| `minidb` | 내장 JSON 문서 저장소 (스냅샷 + WAL, 전문검색) | 61 |
| `klient` | 계약 기반 클라이언트 SDK (zod 검증) | 51 |
| `node-sdk` | 공개 TypeScript SDK / 하네스 | 46 |
| `pi-tui` | 터미널 UI 렌더링 엔진 | 43 |
| `migration-legacy` | v1 → v2 마이그레이션 | 32 |
| `kosong` | LLM / 프로바이더 추상화 | 26 |
| `oauth` | Kimi OAuth + 자격증명 관리 | 26 |
| `acp-server` | Agent Client Protocol 서버 | 26 |
| `transcript` | 대화 렌더링 데이터 레이어 (브라우저 안전, 엔진 import 없음) | 25 |
| `kaos` | 실행 환경 / 파일·프로세스 추상화 | 12 |
| `telemetry` | 클라이언트 텔레메트리 | 9 |
| `tree-sitter-bash` | 순수 TS bash 파서 (런타임 의존성 0, wasm 없음) | 7 |
| `remote-control` | 원격 제어 터널 클라이언트 | 4 |

### 그 외

- `plugins/` — 플러그인 마켓플레이스 카탈로그 (`marketplace.json`)
  - official: `kimi-datasource`(금융·거시 데이터 MCP), `kimi-webbridge`(실제 브라우저 제어)
  - curated: `superpowers`, `vercel-plugin`, `modern-web-guidance`(Google Chrome 팀), `cloudbase`(Tencent)
- `docs/` — 영어 + 중국어 이중 문서 (guides / configuration / reference / customization)
- `.agents/skills/` — 저장소 개발용 스킬 (`gen-changesets`, `tdd`, `write-tui` 등)

---

## 3. 내장 툴 전체 목록

| 분류 | 툴 | 기본 승인 |
| --- | --- | --- |
| 파일 | `Read` `Write` `Edit` `Grep` `Glob` `ReadMediaFile` | 읽기 자동 / 쓰기 승인 |
| 셸 | `Bash` | 승인 필요 |
| 웹 | `WebSearch` `FetchURL` | 자동 |
| 플랜 | `EnterPlanMode` `ExitPlanMode` | 자동 (플랜 확정은 사용자) |
| 상태 | `TodoList` | 자동 |
| 협업 | `Agent` `AgentSwarm` `AskUserQuestion` `NotifyUser` `Skill` | 대부분 자동 |
| 백그라운드 | `TaskList` `TaskOutput` `TaskStop` `WaitFor` | `TaskStop`만 승인 |
| 스케줄 | `CronCreate` `CronList` `CronDelete` | 생성/삭제 승인 |

### 차별화 기능

- **AgentSwarm** — `prompt_template` + `items[]`로 최대 128개 서브에이전트 병렬 실행.
  초기 5개 즉시 시작 후 700ms마다 1개씩 추가. `KIMI_CODE_AGENT_SWARM_MAX_CONCURRENCY`로 상한 설정.
- **Tower** (실험적) — 멀티 에이전트 오케스트레이션.
  `TowerPlan` / `TowerSpawn` / `TowerMerge` / `TowerTeardown` / `TowerSend` / `TowerInbox` /
  `TowerFinding` / `TowerReview` / `TowerMission` / `TowerStatus`.
  워커 간 메시지 교환, 발견사항 공유, 상호 리뷰, 결과 병합까지 지원.
- **Goal 모드** — 목표를 달성할 때까지 턴을 자동으로 이어감. 데드라인 스케줄러와 목표 큐 보유.
  print 모드 종료코드: 완료 `0`, 블록 `3`, 일시정지 `6`.
- **Cron** — 5필드 cron 표현식으로 미래의 자신에게 프롬프트 주입.
  결정론적 지터(주기의 10% 또는 최대 15분)로 정각 몰림 방지, 놓친 발사는 1회로 합산(`coalescedCount`),
  7일 초과 반복 작업은 `stale="true"`로 1회 발사 후 자동 삭제.
- **비디오 입력** — 화면 녹화·데모 영상을 그대로 입력으로 사용.
- 터미널 내 **Mermaid 다이어그램 렌더링** (v2.0.0 신규).

---

## 4. 확장 메커니즘 4종

| | 정체 | 새 능력 추가 | 설정 위치 |
| --- | --- | --- | --- |
| **Skill** | YAML frontmatter + Markdown 문서 | X (지식/절차만) | 스킬 스캔 디렉터리 |
| **MCP** | 외부 프로세스/서비스 | O | `~/.kimi-code/mcp.json`, `.kimi-code/mcp.json` |
| **Hook** | 로컬 스크립트 | X (감시/차단) | config |
| **Plugin** | 위 3개의 배포 단위 | 묶음에 따라 | `kimi.plugin.json` |

- **Skill**: `type: inline`이면 모델이 `Skill` 툴로 직접 호출 가능. 중첩 최대 3단계.
  디렉터리형(`<name>/SKILL.md`)과 플랫형(`<name>.md`) 모두 지원, 충돌 시 디렉터리형 우선.
- **MCP**: stdio / HTTP / SSE 3종 전송. 프로젝트 레벨이 유저 레벨을 덮어씀.
  `/mcp-config`로 JSON 직접 편집 없이 대화형 설정 가능.
- **Hook**: stdin으로 JSON 이벤트 수신, **exit code 0 = 허용, 2 = 차단**.
  **fail-open** 설계 — 스크립트가 실패해도 작업이 멈추지 않으므로 유일한 보안 장치로 쓰면 안 됨.
- **Plugin**: GitHub URL 4가지 형태 지원(레포 / `tree/<ref>` / `releases/tag/<tag>` / `commit/<sha>`).
  `github.com` 리다이렉트와 `codeload.github.com` 다운로드만 사용하고 `api.github.com`은 호출하지 않음.
  `KIMI_CODE_PLUGIN_MARKETPLACE_URL`로 커스텀 마켓플레이스 지정 가능.

---

## 5. 설치 및 사용법

### 설치

```sh
# macOS / Linux
curl -fsSL https://code.kimi.com/kimi-code/install.sh | bash

# Windows (PowerShell)
irm https://code.kimi.com/kimi-code/install.ps1 | iex
```

Node.js 불필요(단일 바이너리). Windows는 Git for Windows 선행 설치 필요
(커스텀 경로면 `KIMI_SHELL_PATH`에 `bash.exe` 절대경로 지정).

### 기본 사용

```sh
kimi --version
cd your-project
kimi            # TUI 실행 → /login 으로 인증
```

### 주요 CLI 옵션 / 서브커맨드

```
kimi -p "<prompt>"                      한 번 실행 (헤드리스)
kimi -p "..." --output-format stream-json   JSON 스트림 출력
kimi -c / -S [id]                       이어하기 / 세션 선택 재개
kimi -y | --auto                        yolo 모드 / 완전 자동 모드
kimi -m <model>                         모델 지정
kimi --agent <name> | --agent-file <p>  에이전트 프로필 지정
kimi --add-dir <dir>                    워크스페이스 디렉터리 추가
kimi --plan                             플랜 모드로 시작
kimi --skills-dir <dir>                 스킬 디렉터리 지정

kimi web | rc | acp | doctor | upgrade
kimi export | fork | provider | session | install-desktop | migrate
```

### 주요 슬래시 커맨드

```
/login /logout /provider /model /secondary-model /settings /permission /theme /experiments
/new /sessions /fork /compact /undo /reload /init /export-md /title /add-dir /copy /tasks
/plan /yolo /auto /swarm /goal
/plugins /mcp /mcp-config /skills
/web /desktop /remote-control
```

### 권한 모드

| 모드 | 동작 |
| --- | --- |
| 기본(manual) | 위험 작업마다 승인 요청 |
| `--yolo` / `-y` | 일상 편집·명령은 자동, 위험 작업·질문·플랜은 승인 |
| `--auto` | 중단 없이 전부 자동 |

### 소스 빌드

```sh
# Node.js >= 24.15.0, pnpm 10.33.0 (engine-strict=true)
pnpm install
pnpm dev:cli / dev:server / test / typecheck / lint / build
```

---

## 6. 플러그인인가, 스킬인가, MCP인가

**셋 다 아니다. Kimi Code는 그것들을 소비하는 "호스트"다.**

- 플러그인을 설치받는 쪽이고, 자체 마켓플레이스를 운영한다.
- 스킬을 로드해 실행하는 런타임이다.
- MCP 클라이언트다 (stdio / HTTP / SSE).

동시에 다음 역할도 겸한다.

- VS Code 확장 (`apps/vscode`)
- ACP 서버 (`kimi acp` → Zed / JetBrains)
- REST + WebSocket 서버 (`kimi web`)
- npm SDK (`@moonshot-ai/kimi-code-sdk`)

---

## 7. API 토큰 / 인증

세 가지 경로가 있다.

1. **Kimi Code OAuth** — `/login` → 디바이스 코드 플로우. API 키를 직접 다루지 않음.
   리전은 `mainland-cn` / `global`. Datasource, Remote Control은 이 로그인 필요.
2. **Kimi Platform API 키** — `https://api.moonshot.cn/v1`(platform.kimi.com) 또는
   `https://api.moonshot.ai/v1`(platform.kimi.ai). 허용 모델 프리픽스 `kimi-k`.
3. **다른 프로바이더 직접 연결** — `config.toml`에 선언.

지원 프로바이더 타입 6종:

| type | 프로토콜 | 자격증명 키 |
| --- | --- | --- |
| `kimi` | OpenAI 호환 | `KIMI_API_KEY` / `KIMI_BASE_URL` |
| `anthropic` | Anthropic Messages | `ANTHROPIC_API_KEY` / `ANTHROPIC_BASE_URL` |
| `openai` | OpenAI Chat Completions | `OPENAI_API_KEY` / `OPENAI_BASE_URL` |
| `openai_responses` | OpenAI Responses | `OPENAI_API_KEY` / `OPENAI_BASE_URL` |
| `google-genai` | Google GenAI | `GOOGLE_API_KEY` |
| `vertexai` | Google GenAI on Vertex | Google Cloud ADC |

DeepSeek / Qwen 등 OpenAI 호환 서비스는 `reasoning_content`와 `reasoning_effort`를 자동 처리한다.

**자격증명 우선순위**: `api_key` 또는 `api_key_env`(둘 중 하나만) > `[providers.<name>.env]` 하위 테이블 >
모두 없으면 시작 시 에러. 명시 선언한 `api_key_env` 외에는 셸 환경변수로 폴백하지 않는다.
커스텀 레지스트리가 `env` 필드를 선언해도 자동 바인딩하지 않고 힌트만 출력한다
(레지스트리가 엔드포인트도 정하므로, 어떤 비밀을 읽을지는 레지스트리가 결정하면 안 되기 때문).

소프트웨어 자체는 MIT로 무료지만 모델 추론 비용은 별도다.
다만 로컬 모델(Ollama, vLLM 등)을 OpenAI 호환 엔드포인트로 띄워 연결하면 API 비용 없이 운용할 수 있다.

---

## 8. GitHub에서 주목받는 이유 (근거 기반 분석)

> 별 개수는 오프라인 환경이라 직접 확인하지 못했다. 아래는 코드와 문서에서 확인한 사실 기반 추정이다.

1. **"오픈소스 Claude Code" 포지션** — 비공개 경쟁 제품들과 달리 MIT로 전부 공개.
2. **코드 품질** — `agent-core-v2` 1,067 파일의 DI × Scope 설계,
   주석 금지 구역을 `scripts/check-no-comments.mjs`로 lint에서 강제,
   `tree-sitter-bash`를 런타임 의존성 없이 순수 TS로 직접 구현,
   `minidb`의 스냅샷 + WAL + 대용량 전문검색 자체 구현.
3. **설치 난이도가 낮음** — `curl | bash` 한 줄, Node.js 불필요.
4. **문서 품질** — 영/중 이중화, 툴별 파라미터·기본값·엣지케이스·종료코드까지 기술.
   `/openapi.json`, `/asyncapi.json`을 런타임 생성하고 문서와 충돌 시 live spec 우선임을 명시.
5. **고유 기능** — 비디오 입력, Tower 오케스트레이션, Remote Control(QR), AgentSwarm 128 병렬, Goal 모드.
6. **생태계 신뢰 신호** — 큐레이티드 플러그인에 Google Chrome 팀, Vercel, Tencent CloudBase 포함.
7. **활발한 커뮤니티** — CHANGELOG의 PR 번호가 #3800번대, 외부 기여자 다수, 릴리스 주기 빠름.
8. **ACP 표준 채택** — 에디터 종속 없음.

---

## 9. 로컬 에이전트 구축에 대한 활용도

### 활용 경로 4가지

1. **SDK로 임베드** — `@moonshot-ai/kimi-code-sdk` (`kimi-harness.ts`, `session.ts`, `tool.ts`,
   `permission.ts`, `mcp.ts`, `replay.ts`). 에이전트 루프를 새로 짤 필요 없음.
2. **서버로 띄우고 API로 조종** — `kimi web --no-open --port <p>` → `/api/v1` + `/api/v1/ws`. 언어 무관.
3. **파이프로 사용** — `kimi -p "..." --output-format stream-json`.
4. **아키텍처 참고** — 아래 패턴들이 특히 참고할 만함.

### 참고할 만한 설계 패턴

- **4계층 Scope** (`app/scopes.ts`): `App` → `Workspace` → `Session` → `Agent`.
  계층별 서비스 생명주기 분리로 누수 방지.
- **Feature seam** (`src/features/`): 기능 하나가 클래스 하나.
  `contributeService` / `contributeTool`로 서비스와 툴을 "기여"하고 `registerFeature`로 등록.
  기존 코드를 건드리지 않고 기능 추가 가능.
- **실험 플래그**: `KIMI_CODE_EXPERIMENTAL_<NAME>` > `[experimental]` config >
  `KIMI_CODE_EXPERIMENTAL_FLAG` > 코드의 `default`. 출시는 `default`를 뒤집는 것으로 끝.
- **툴 관심사 분리**: `toolPolicy` / `toolApproval` / `toolActivation` / `toolDedupe` /
  `toolExecutor` / `toolRegistry` / `toolSelect` / `toolResultTruncation`를 각각 별도 모듈로 분리.
- **백그라운드 태스크 수명주기**: SIGTERM → 5초 유예 → SIGKILL 2단계 종료,
  포그라운드 타임아웃 시 종료 대신 백그라운드로 승격, stdin은 항상 닫아 대화형 명령에 EOF 전달.

### 주의사항

| 항목 | 내용 |
| --- | --- |
| 의존성 무게 | `pnpm install` 결과물이 큼. 임베드 시 배포 크기 고려 필요 |
| API 안정성 | 서버 REST/WS API는 문서에 **experimental**로 명시됨. 버전 고정 권장 |
| 웹 UI 소스 | `dist-web`은 프리빌드 번들만 포함, 소스는 별도 저장소 |
| Node 버전 | `>= 24.15.0` 필수 (`engine-strict=true`) |
| 학습 곡선 | DI 컨테이너 + Scope + Feature 조합 구조 |
| 텔레메트리 | `packages/telemetry` 존재. 배포 전 수집 항목 확인 필요 |

---

## 10. React / PHP로 만들 수 있는가

### A. 기존 React / PHP 앱에 붙이기 — 가능하고 권장

```
┌──────────────┐      ┌─────────────┐      ┌──────────────┐
│  React SPA   │◄────►│ PHP/Laravel │◄────►│  kimi web    │
│  (UI)        │ REST │ (인증/과금)  │ REST │  (에이전트)   │
└──────────────┘      └─────────────┘  WS  └──────┬───────┘
                                                   │
                                            ┌──────▼──────┐
                                            │  LLM API    │
                                            └─────────────┘
```

**React 쪽 핵심**

- 기본 주소 `http://127.0.0.1:58627` (포트 사용 중이면 최대 100회 다음 포트로 재시도).
- REST 인증: `Authorization: Bearer <token>`.
- WebSocket 인증: 동일 헤더 또는 서브프로토콜 `kimi-code.bearer.<token>`.
- 모든 JSON 응답은 `{ code, msg, data, request_id }` 봉투 형태.
  HTTP 상태는 대부분 200이므로 **`code`를 먼저 검사**해야 한다 (`code: 0`이 성공).
- `OPTIONS` 프리플라이트와 `GET /api/v1/healthz`는 인증 면제.
- 비루프백 바인드에서는 60초 내 인증 10회 실패 시 60초 차단(HTTP 429, 코드 `42901`).
- `/openapi.json`, `/asyncapi.json`으로 타입 자동 생성 가능.
- `packages/transcript`는 브라우저 안전한 순수 TS이므로 대화 렌더링 로직을 프론트에서 재사용 가능.

**PHP 쪽 핵심**

- cURL로 REST 호출, 또는 `kimi -p ... --output-format stream-json`을 큐 워커에서 실행.
- Laravel 관리자 화면 → 큐 잡 → CLI 실행 → 결과 DB 저장 형태가 자연스럽다.

### B. React / PHP로 Kimi Code 자체를 재구현 — 비권장

- React: 브라우저 렌더링 라이브러리이므로 터미널 ANSI 제어와 파일시스템·프로세스 접근에 부적합.
  Electron + React로 감싸면 가능하지만 결국 Node.js 백엔드가 필요해진다.
- PHP: 요청-응답 모델이 기본이라 장시간 스트리밍 + 동시 서브프로세스 + WebSocket에 불리하다.
  ReactPHP/Swoole로 가능하긴 하나, `tree-sitter-bash`·`minidb`·`pi-tui`까지 재구현해야 한다.

**결론**: React는 프론트, PHP는 인증/과금/DB, Kimi Code는 에이전트 엔진으로 역할을 나누는 것이 합리적이다.

---

## 11. 수익화 아이디어

### 티어 1 — 즉시 착수 가능

| # | 아이디어 | 내용 | 가격 예시 |
| --- | --- | --- | --- |
| 1 | 한국어 플러그인 팩 | 국내 코드리뷰/커밋 컨벤션 스킬, 금융권 보안 체크, 국내 서비스 MCP(토스페이먼츠·카카오·네이버클라우드 등) | 무료 / ₩29,000월 / 팀 ₩199,000월 |
| 2 | 교육 콘텐츠 | 이 저장소를 교재로 한 에이전트 아키텍처 강의·전자책·기업 워크샵 | 강의 ₩99,000~249,000, 워크샵 ₩3,000,000~ |
| 3 | 컨설팅 / SI | 사내 전용 에이전트 구축, 로컬 모델 조합, 사내 MCP 개발, Hooks 보안 정책 | 구축 ₩10M~50M, 유지보수 ₩1M~3M/월 |

1번이 가능한 근거: 커스텀 마켓플레이스 URL(`KIMI_CODE_PLUGIN_MARKETPLACE_URL`)과 GitHub URL 설치를 지원하므로
배포 진입장벽이 사실상 없다.

### 티어 2 — 3~6개월

| # | 아이디어 | 내용 | 가격 예시 |
| --- | --- | --- | --- |
| 4 | 산업 특화 포크 | 금융/의료/커머스/게임/공공 버티컬별 리브랜딩 제품 (MIT라 합법) | 시트당 ₩50,000~200,000/월 |
| 5 | 팀 협업 SaaS | 팀 대시보드, 비용/쿼터 관리, 감사 로그, 스킬 공유 허브, SSO, 중앙 키 관리 | Team ₩30,000/인/월, Enterprise ₩100,000/인/월 |
| 6 | MCP 서버 마켓플레이스 | 세무·회계, 부동산, 국내 주식, 법령/판례, 쇼핑몰 통합, 사내 위키 등 | ₩30,000~100,000/월 |

5번이 가능한 근거: `kap-server`가 이미 멀티 워크스페이스 + 세션 인덱스 + `minidb` 검색을 제공한다.
6번이 유리한 이유: MCP는 표준 프로토콜이라 한 번 만들면 다른 호스트에서도 판매 가능하다.
공식 `kimi-datasource`가 이미 동일한 모델(데이터를 MCP로 제공하고 플랜 쿼터에서 차감)을 쓰고 있다.

### 티어 3 — 6개월 이상

| # | 아이디어 | 내용 | 가격 예시 |
| --- | --- | --- | --- |
| 7 | 온프렘 어플라이언스 | 포크 + 로컬 LLM + GPU 서버 + 설치/교육/지원. 완전 폐쇄망 | 초기 ₩50M~500M + 연 유지보수 20% |
| 8 | 워크플로우 자동화 플랫폼 | Tower + Cron + Goal 조합. 야간 취약점 스캔→자동 PR→테스트→알림 | 레포당 ₩100,000/월 또는 프로젝트 ₩5M~30M |
| 9 | 모바일 리모컨 제품 | Remote Control 확장. 빌드 상태 확인, 이동 중 승인, CI 실패 알림·지시 | ₩15,000/인/월 애드온 |

7번이 가능한 근거: `kosong`이 OpenAI 호환 엔드포인트면 무엇이든 연결하므로 vLLM/Ollama/LM Studio로 외부 인터넷 없이 운용 가능하다.

### 권장 로드맵

```
1~3개월   아이디어 1 + 2   → 시장 반응 확인, 브랜딩
3~6개월   아이디어 3 + 6   → 현금 흐름 확보, 실제 고객 니즈 발굴
6~12개월  아이디어 5 또는 4 → 스케일업
```

컨설팅으로 실제 고객 문제를 파악한 뒤 그 내용을 제품 스펙으로 삼는 순서가 실패 위험이 낮다.

### 법적 / 실무 체크리스트

| 항목 | 내용 |
| --- | --- |
| MIT 라이선스 | 상업적 이용 가능. 저작권 고지 + 라이선스 전문 유지 필수 |
| 상표권 | "Kimi", "Moonshot AI" 명칭·로고는 사용 불가. 코드 라이선스와 상표는 별개 |
| 의존성 라이선스 | `apps/vscode`는 Apache-2.0, `kimi-webbridge`는 Proprietary(재배포 불가) |
| 텔레메트리 | `packages/telemetry` 존재. 배포 전 수집 항목 확인 및 고지 |
| API 약관 | Moonshot / Anthropic / OpenAI 각각의 재판매 조항 확인 |
| 개인정보 | 개인정보보호법 — 코드에 개인정보 포함 시 해외 API 전송 이슈 |
| API 안정성 | 서버 REST/WS API는 experimental. 버전 고정 필요 |

---

## 12. 참고 링크

- 원본 저장소: <https://github.com/MoonshotAI/kimi-code>
- 포크 저장소: <https://github.com/bmshin94/kimi-code>
- 공식 문서: <https://moonshotai.github.io/kimi-code/en/>
- 이슈: <https://github.com/MoonshotAI/kimi-code/issues>
- Agent Client Protocol: <https://agentclientprotocol.com/>
- Model Context Protocol: <https://modelcontextprotocol.io/>
- pi-tui (TUI 기반 라이브러리): <https://github.com/earendil-works/pi-mono/tree/main/packages/tui>
- 큐레이티드 플러그인
  - Superpowers: <https://github.com/obra/superpowers>
  - Vercel Plugin: <https://github.com/vercel/vercel-plugin>
  - Modern Web Guidance: <https://github.com/GoogleChrome/modern-web-guidance>
  - Tencent CloudBase: <https://github.com/TencentCloudBase/CloudBase-AI-Toolkit>
