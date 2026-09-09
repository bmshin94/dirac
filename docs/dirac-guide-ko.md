# Dirac 한국어 가이드 & 활용 정리

> 이 문서는 Dirac 저장소를 직접 분석하면서 정리한 한국어 안내서입니다.
> 프로젝트 소개 → 설치 → 사용법 → 활용/수익화 아이디어 → 기술 스택 순으로 구성되어 있습니다.

## 저장소 주소

| 구분 | 주소 |
| --- | --- |
| 이 저장소 (fork) | https://github.com/bmshin94/dirac |
| 원본 저장소 (upstream) | https://github.com/dirac-run/dirac |
| 공식 홈페이지 | https://dirac.run |
| NPM 패키지 | https://www.npmjs.com/package/dirac-cli |
| VS Code 마켓플레이스 | https://marketplace.visualstudio.com/items?itemName=dirac-run.dirac |
| Open VSX | https://open-vsx.org/extension/dirac-run/dirac |
| Discord | https://discord.gg/wcYTx9BGea |
| 포크 원본(Cline) | https://github.com/cline/cline |

---

## 1. Dirac이란?

한 줄 요약: **코드를 대신 읽고, 고치고, 실행하고, 검증까지 하는 오픈소스 AI 코딩 에이전트.**

- 유명 오픈소스 에이전트 **Cline**을 포크해서 "비용 효율"에 초점을 맞춰 개조한 프로젝트
- 개발: Dirac Delta Labs
- 라이선스: **Apache License 2.0**
- 현재 버전: `v0.5.11`
- 사용 가능한 환경: **VS Code / Open VSX 계열 에디터 / 터미널(CLI) / ACP 호환 에디터(JetBrains, Zed)**
- 수십 개 프로바이더, 수백 개 모델 지원 (내가 원하는 모델 + 내 API 키를 그대로 사용)

### 핵심 셀링포인트: 같은 일을 더 싸게

README에 공개된 벤치마크 결과입니다. 실제 오픈소스 저장소(Transformers, VS Code, Django)의
멀티파일 리팩터링 과제 8개를 여러 에이전트에게 동일 조건(`gemini-3-flash-preview`, thinking=high)으로 시킨 결과입니다.

| 에이전트 | 성공 | 평균 비용 |
| --- | :---: | :---: |
| Cline | 5/8 | $0.49 |
| Kilo | 5/8 | $0.73 |
| Ohmypi | 6/8 | $0.51 |
| Opencode | 8/8 | $0.44 |
| Pimono | 6/8 | $0.38 |
| Roo | 6/8 | $0.60 |
| **Dirac** | **8/8** | **$0.18** |

> 성공률은 공동 1위, 비용은 최저. 검증용 diff 원본이 `evals/` 폴더에 전부 들어 있어 직접 확인 가능합니다.

---

## 2. 왜 저렴한가 (효율의 3가지 축)

### 2-1. 해시 앵커 편집 (hash-anchored editing)

일반적인 "문자열 찾아 바꾸기" 방식은 주변 코드가 조금만 밀려도 실패해서 재시도가 발생합니다.
Dirac은 소스의 각 줄에 **안정적인 앵커(해시)** 를 부여하고, 모델이 "몇 번 앵커부터 몇 번 앵커까지"를
지정해 정확히 그 범위만 교체합니다. 편집 실패와 재시도가 줄어들어 토큰 비용이 감소합니다.

### 2-2. AST 기반 코드 탐색 / 조작

파일 전체를 읽지 않고 **문법 트리(AST)** 로 구조만 파악합니다. (책을 다 읽지 않고 목차만 보는 방식)

```text
inspect_ast(operation: "outline", paths: ["utils/db.py"])
inspect_ast(operation: "implementation", paths: ["utils/db.py"], symbols: ["DBManager.init_db"])

edit_ast(operation: "replace", targets: [{ path: "utils/db.py", symbol: "DBManager.init_db", replacement: "..." }])
edit_ast(operation: "rename",  targets: [{ path: "src/", symbol: "old_name", replacement: "new_name" }])
```

심볼 단위로 동작하므로, 수백 개의 참조 이름 변경을 **한 번의 호출**로 처리할 수 있습니다.

> 주의: 구조 분석 결과는 파서와 인덱스 커버리지에 따라 달라집니다. 동적 참조는 별도 검증이 필요합니다.

### 2-3. Utility 모델 분리 (토큰 차익거래)

컨텍스트 압축, 새 작업 인계, 커밋 메시지 생성, 권한 판단 같은 **보조 작업**을 더 저렴한 모델에 위임하고,
실제 구현은 메인 모델이 담당합니다. 공식 사례 연구에서 컨텍스트 압축 비용이 **80% 이상 절감**되었습니다.

---

## 3. 주요 기능 요약

| 기능 | 설명 |
| --- | --- |
| **Goal 모드** | `/goal <목표>` — 몇 시간~며칠 동안 목표를 향해 자율적으로 작업을 생성·조율. 입력이 필요하면 일시정지 |
| **실시간 조종(Steering)** | 작업 중 메시지를 보내면 **취소 없이** 다음 안전한 턴에 반영 |
| **커스텀 도구 생성** | `/new-tool <설명>` — 도구를 생성·컴파일·검증·스모크 테스트까지 자동 수행 |
| **서브에이전트** | 여러 에이전트가 동시에 조사/편집/명령 실행. 소스 신선도를 추적해 오래된 편집은 거부 |
| **병렬 편집** | 읽기·검색·편집·명령을 한 응답에 묶어서 처리 |
| **저-verbosity 응답** | 불필요한 진행 설명을 줄이고 결정/주의사항/검증 결과만 유지 |
| **완료 검증** | 별도 모델 패스로 수용 기준 충족 여부를 점검하고 누락 시 후속 작업 반환 |
| **체크포인트** | 워크스페이스 파일 + 작업 상태 저장 및 복원 |
| **권한 제어** | 도구 경계에서 권한 평가, 자연어 정책 기반 자동 승인, 셸 명령 화이트리스트 |
| **자기 설명** | `/askDirac <질문>` — 설치된 빌드의 소스를 읽고 자기 동작을 설명 |

### 내장 도구 목록

```text
read_file        write_to_file    edit_file       edit_ast
inspect_ast      search_files     list_files      execute_command
browser_action   diagnostics_scan new_task        condense
use_skill        list_skills      use_subagents   upsert_tool
respond
```

---

## 4. 설치

### 사전 요구사항

- **Node.js 22.13 ~ 24.x** (Node 25는 메모리 이슈로 미지원)
- npm

```bash
node -v   # v22.x ~ v24.x 확인
```

### 4-1. CLI (가장 간단)

```bash
npm install -g dirac-cli
dirac auth
```

### 4-2. VS Code / Open VSX

확장 검색창에서 `dirac-run.dirac` 설치 (VS Code 1.101 이상).

### 4-3. ACP 에디터 (JetBrains / Zed)

- **JetBrains 2025.3+**: `Settings → Tools → AI Assistant → Agents` → *Install From ACP Registry…*
- **Zed**: `Agent Settings → External Agents` → *Add Agent → Install from Registry*

수동 설정:

```json
{
  "command": "dirac",
  "args": ["--acp"]
}
```

> 에디터가 `dirac`을 PATH에서 못 찾으면 절대경로를 사용하세요. `--cwd <path>` 로 기본 워크스페이스 지정 가능.

### 4-4. 소스에서 빌드

```bash
npm run install:all
npm run cli:build            # CLI 빌드
node cli/dist/cli.mjs --help
```

VS Code 확장으로 빌드:

```bash
npm run install:all
npm run protos               # 필수! protobuf 코드 생성
npm run compile
npm run lint
```

> `npm run protos`를 건너뛰면 생성 타입이 없어 컴파일이 실패합니다.

---

## 5. 인증 (프로바이더 설정)

### 대화형

```bash
dirac auth
```

### 원샷

```bash
dirac auth --provider anthropic --apikey "$ANTHROPIC_API_KEY" --modelid claude-sonnet-4-6
dirac auth --provider openai --apikey "$OPENAI_API_KEY" --modelid gpt-4o --baseurl https://api.example.com/v1
```

### 표준 환경변수 (자동 인식)

```text
ANTHROPIC_API_KEY   OPENAI_API_KEY   OPENROUTER_API_KEY   GEMINI_API_KEY
GROQ_API_KEY        MISTRAL_API_KEY  XAI_API_KEY          HF_TOKEN
```

### 프로세스 단위 강제 지정 (`DIRAC_*` 가 저장된 설정보다 우선)

```bash
DIRAC_PROVIDER=deepseek \
DIRAC_MODEL=deepseek-chat \
DIRAC_API_KEY="$DEEPSEEK_API_KEY" \
dirac --acp
```

| 변수 | 의미 |
| --- | --- |
| `DIRAC_PROVIDER` | 프로바이더 ID (`anthropic`, `deepseek`, `openrouter` 등) |
| `DIRAC_MODEL` | 해당 프로바이더의 정확한 모델 ID |
| `DIRAC_API_KEY` | API 키 |
| `DIRAC_BASE_URL` | 커스텀 엔드포인트(선택) |

### 클라우드 프로바이더

**AWS Bedrock** — `AWS_ACCESS_KEY_ID` 또는 `AWS_BEDROCK_MODEL` 이 있으면 자동 전환

```bash
AWS_REGION=us-east-1 AWS_BEDROCK_MODEL=us.anthropic.claude-sonnet-4-6 \
  aws-vault exec my-profile -- dirac "작업 내용"
```

- `AWS_REGION` 단독으로는 자동 전환되지 않습니다.
- 최신 Claude 모델은 `us.` / `eu.` / `ap.` 리전 접두사가 필요합니다.
- 모드별 분리: `AWS_BEDROCK_MODEL_ACT`, `AWS_BEDROCK_MODEL_PLAN`

**Google Vertex AI** — `GOOGLE_CLOUD_PROJECT` 또는 `GCP_PROJECT` 가 있으면 자동 전환

```bash
gcloud auth application-default login
export GOOGLE_CLOUD_PROJECT=<project-id>
export GOOGLE_CLOUD_LOCATION=us-central1
```

---

## 6. 사용법

### 6-1. 인터랙티브 모드 (기본)

```bash
dirac                                # 컴포저 열기
dirac "이 코드베이스를 분석해줘"
dirac --plan "구현 방안을 설계해줘"    # 계획만, 코드 변경 없음
```

승인/거부, 후속 메시지, 히스토리, 설정, 모델 선택, 스킬, 파일 멘션, 이미지 붙여넣기, 작업 재개를 지원합니다.

### 6-2. 스탠드얼론 모드 (자동화)

`--yolo` / `--json` 이 있거나, stdin이 파이프이거나, 출력이 리다이렉트되면 자동 전환됩니다.

```bash
dirac --yolo "테스트를 실행하고 실패를 고쳐줘"
git diff | dirac "이 변경사항을 리뷰해줘"
dirac --json "관련 파일을 나열해줘" > events.ndjson
```

- **최종 결과는 stdout**, 진행상황·도구 활동·경고·에러는 **stderr** 로 분리 → stdout 파이프 연결이 안전
- `--yolo` 없이 승인이 필요해지면 오류로 종료됩니다
- 파이프로 들어온 입력은 보존되며 프롬프트 앞에 배치됩니다

### 6-3. 자주 쓰는 명령

```bash
dirac auth                          # 프로바이더/모델 설정
dirac config                        # 현재 설정 확인
dirac history                       # 이전 작업 탐색·재개
dirac --continue                    # 이 워크스페이스의 마지막 작업 재개
dirac --taskId <id> "후속 요청"      # 특정 작업 재개
dirac tools                         # 현재 유효한 도구 목록
dirac update                        # 새 릴리스 확인
dirac --acp                         # ACP 에이전트로 실행
dirac kanban                        # 칸반 연동
dirac --help                        # 전체 옵션(항상 최신)
```

`dirac task` (별칭 `dirac t`) 는 기본 프롬프트 명령과 동일한 작업 옵션을 받습니다.

### 6-4. 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| `--act` / `--plan` | 작업 모드 선택 (동시 사용 불가) |
| `--yolo` | 스탠드얼론 출력 + 자동 승인 |
| `--auto-approve-all` | 인터랙티브 UI 유지 + 자동 승인 |
| `--model <id>` | 모델 재정의 |
| `--provider <id-or-url>` | 프로바이더 재정의 (`--model` 필요) |
| `--enable-tool <ids>` / `--disable-tool <ids>` | 도구 켜기/끄기 (쉼표 구분, 반복 가능) |
| `--only-tools <ids>` | 지정한 도구만 사용 (delta 옵션과 병용 불가) |
| `--images <paths...>` | PNG / JPEG / GIF / WebP 첨부 |
| `--thinking [tokens]` | 확장 사고 (기본 1024) |
| `--reasoning-effort <level>` | `none` / `low` / `medium` / `high` / `xhigh` |
| `--timeout <seconds>` | 스탠드얼론 작업 제한 시간 |
| `--verbose` | 추론·진단 상세 출력 |
| `--json` | 줄단위 JSON 이벤트 |
| `--config <path>` | 다른 Dirac 홈 디렉터리 사용 |

```bash
dirac --only-tools read_file,search_files,respond "워크스페이스를 분석해줘"
dirac --yolo --enable-tool read_file,edit_file --disable-tool use_subagents "이슈를 고쳐줘"
```

> 도구 선택은 해당 실행에만 적용되고 저장되지 않습니다. ACP, 분리 리슨 모드, 칸반, Goal 모드에서는 사용할 수 없습니다.
> `dirac tools` 는 현재 설정을 복사·붙여넣기 가능한 옵션 형태로 출력합니다.

이미지는 대화 중 인라인 멘션도 가능합니다: `@/images/screenshot.png` (워크스페이스 기준), 절대경로 및 `./상대경로`도 허용.

### 6-5. 슬래시 명령

| 명령 | 설명 |
| --- | --- |
| `/help` | 도움말 |
| `/settings` | 저-verbosity, Utility 모델, 승인 동작, 서브에이전트 등 설정 |
| `/models` | 모델 변경 |
| `/config` | 현재 설정 |
| `/history` | 작업 기록 |
| `/clear` | 대화 초기화 |
| `/askDirac <질문>` | 설치된 Dirac 빌드에 대해 질문 |
| `/review`, `/review-branch`, `/review-commit` | 코드 리뷰 |
| `/agent` | 에이전트 전환 |
| `/workspace`, `/repo`, `/path` | 작업 범위 지정 |
| `/quiet` (`/q`) | 조용 모드 |
| `/exit` | 종료 |

추가로 `/goal <목표>` (Goal 모드, **인터랙티브 터미널 전용**) 와 `/new-tool <설명>` (도구 생성) 을 지원합니다.

### 6-6. 설정 위치와 기타 환경변수

```text
~/.dirac/data/                # 전역 설정
~/.dirac/data/workspaces/     # 워크스페이스별 상태
```

`--config <path>` 또는 `DIRAC_DIR` 로 다른 홈 디렉터리를 지정할 수 있습니다.
ACP 사용 시 설정과 실행에 **같은 `--config` 경로**를 전달해야 합니다.

| 변수 | 효과 |
| --- | --- |
| `DIRAC_NO_AUTO_UPDATE=1` | 백그라운드 업데이트 확인 비활성화 |
| `DIRAC_NO_EMOJI=1` | 아이콘을 Unicode/ASCII 대체로 |
| `DIRAC_COLOR_MODE=light\|dark\|auto` | 터미널 색상 모드 강제 |
| `CUSTOM_HEADERS` | OpenAI 호환 커스텀 헤더 (JSON 또는 `key=value`) |
| `DIRAC_COMMAND_PERMISSIONS` | 셸 명령 허용/거부 패턴 |

`NO_COLOR`, `FORCE_COLOR` 도 존중하며, `COLORFGBG` 로 밝은 배경이 감지되면 고대비 라이트 팔레트를 자동 선택합니다.

---

## 7. 처음 시작하는 추천 순서

```bash
# 1) 설치
npm install -g dirac-cli

# 2) 인증 (무료 티어가 있는 Gemini/Groq로 시작해도 좋음)
dirac auth

# 3) 안전하게 읽기 전용으로 첫 테스트
cd ~/my-project
dirac --only-tools read_file,search_files,respond "이 프로젝트 구조를 설명해줘"

# 4) 계획 모드로 설계 검증
dirac --plan "○○ 기능 구현 방안을 설계해줘"

# 5) 익숙해지면 자동 실행
dirac --yolo "실패한 테스트를 고쳐줘"
```

`/settings` 에서 반드시 확인할 두 가지:

1. **Utility 모델** — 보조 작업을 저렴한 모델로 위임 (비용 대폭 절감)
2. **Low-verbosity responses** — 불필요한 서술 축소로 토큰 절약

---

## 8. 문제 해결

| 증상 | 확인 사항 |
| --- | --- |
| `dirac` 명령을 찾을 수 없음 | `npm bin -g` 경로가 `PATH` 에 포함됐는지 확인 |
| 소스 빌드 시 컴파일 실패 | `npm run protos` 를 먼저 실행했는지 확인 |
| Node 메모리 오류 | Node 버전이 22.13~24.x 인지 확인 (25 미지원) |
| 에디터가 에이전트를 못 찾음 | ACP 설정에 실행 파일 **절대경로** 지정 |
| 옵션이 문서와 다름 | `dirac --help` 및 설치된 `dirac(1)` man 페이지가 최신 기준 |

---

## 9. 활용 / 수익화 아이디어

### 9-0. 라이선스 전제

Apache 2.0 이므로 **상업적 이용·수정·재배포가 허용**됩니다. 다만 다음 의무가 있습니다.

- 라이선스 사본 포함
- 저작권 고지 유지 (Dirac Delta Labs + 포크 원본인 Cline)
- 변경한 파일에 변경 사실 명시

> 주의: Apache 2.0 은 **코드**에 대한 허가입니다. **"Dirac" 이름과 로고는 상표**로 별도이므로,
> 제품화하려면 리브랜딩이 필요합니다.

### 9-1. 자본 없이 바로 (권장)

| 아이디어 | 근거 / 활용 포인트 |
| --- | --- |
| **도입 컨설팅 · 세팅 대행** | `evals/` 의 경쟁 도구 비교 diff가 그대로 제안서 근거가 됨. 구독료 → 종량제 전환 제안 |
| **한국어 콘텐츠 · 강의** | `locales/ko/` 에 행동강령·기여가이드만 존재. 한국어 사용 문서가 사실상 공백 |
| **레거시 리팩터링 대행** | 벤치마크 과제 자체가 멀티파일 리팩터링. 이 도구의 최강 영역 |

### 9-2. 제품화 (수 주 ~ 수 개월)

| 아이디어 | 활용 포인트 |
| --- | --- |
| **팀 비용 대시보드 SaaS** | `--json` 의 NDJSON 이벤트를 수집 → 사용자/모델/일자별 비용 리포트 |
| **업종별 커스텀 툴팩** | `upsert_tool`, `ToolDiscoveryService`, `UserToolLoader` 등 확장 지점 존재 |
| **CI/CD 자동 수정 봇** | `--yolo` + `--json` + `dirac kanban` 조합으로 이슈 → 자동 PR |

### 9-3. 엔터프라이즈 (장기·고단가)

**온프레미스 사내 배포판.** 근거가 되는 기능이 이미 있습니다.

- `DIRAC_COMMAND_PERMISSIONS` — 셸 명령 허용/거부 패턴
- `src/core/permissions/` — 권한 정책
- `src/core/hooks/` — 도구 실행 전후 훅 (감사 로그 부착 지점)
- 로컬 모델 어댑터(`ollama`, `lmstudio`) — 코드 외부 유출 없이 운영 가능

금융·공공 등 폐쇄망 시장 대상. 단가는 높지만 영업 사이클이 깁니다.

### 9-4. 리스크

1. 오픈소스 래핑 제품은 해자가 약함 → 차별점은 **도메인 지식과 시장 접근성**
2. 업스트림이 동일 기능을 무료로 출시할 수 있음 → 도구 개조보다 **서비스·콘텐츠**가 안전
3. 상표 문제 (리브랜딩 필수)
4. AI 코딩 시장은 경쟁이 치열하나, **"비용 최적화"** 각도는 상대적으로 덜 붐빔

---

## 10. 기술 스택 및 폴더 구조

TypeScript/TSX 파일 약 **2,012개** 규모의 프로젝트입니다.

| 경로 | 역할 |
| --- | --- |
| `src/core/` | 핵심 로직 — `task/`(작업 실행), `api/`(모델 통신), `prompts/`, `goal/`, `hooks/`, `permissions/`, `slash-commands/`, `text-condensation/`, `utility-model/` |
| `src/core/api/providers/` | 40여 개 프로바이더 어댑터 (anthropic, gemini, openai, bedrock, vertex, openrouter, groq, ollama, lmstudio, deepseek 등) |
| `src/services/` | tree-sitter, symbol-index, source-ast, ripgrep, telemetry, browser 등 |
| `webview-ui/` | VS Code 사이드바 UI — **React + Vite + Tailwind + Radix UI + HeroUI** |
| `cli/` | `dirac` 터미널 패키지 |
| `standalone/`, `proto/` | gRPC / protobuf 기반 에디터 외부 실행 |
| `evals/` | 벤치마크 원본 diff (cline, kilo, ohmypi, opencode, pimono, roo, dirac) |
| `agent-registry/` | ACP 레지스트리 메타데이터 |
| `locales/` | 다국어 문서 (`ko` 포함) |
| `docs/` | 프로바이더별 설정, 로그 보존 정책 |
| `AGENTS.md`, `CLAUDE.md` | 에이전트가 이 저장소에서 따라야 할 지침 |

### 언어 지원 (AST 기능)

`src/services/tree-sitter/queries/` 기준:

```text
c   c-sharp   cpp   go   java   javascript   kotlin
php   python   ruby   rust   swift   typescript   zig
```

**PHP는 `SymbolIndexEligibility.ts` 의 심볼 인덱스 대상에도 포함**되어 있어,
`inspect_ast` / `edit_ast` 를 이용한 구조 분석과 심볼 단위 리네임이 동작합니다.
React(`.tsx`/`.jsx`)는 javascript/typescript 파서로 처리됩니다.

> 다만 구조 분석 결과는 파서/인덱스 커버리지에 의존합니다. Blade 템플릿 같은 혼합 문법은
> 일반 텍스트로 처리될 수 있습니다.

### 다른 언어에서 Dirac 연동하기

Dirac은 CLI 프로그램이므로 **언어에 상관없이 프로세스 실행으로 연동**할 수 있습니다.
아래는 PHP에서 JSON 이벤트를 수집하는 예시입니다. (서버에 Node.js 22~24 필요)

```php
<?php
$cmd = 'dirac --yolo --json ' . escapeshellarg($prompt);

$proc = proc_open($cmd, [
    1 => ['pipe', 'w'],   // stdout: 최종 결과 + JSON 이벤트
    2 => ['pipe', 'w'],   // stderr: 진행상황 / 경고
], $pipes, $workdir);

while (($line = fgets($pipes[1])) !== false) {
    $event = json_decode($line, true);
    // 예: 토큰/비용을 DB에 적재하여 대시보드로 노출
}
proc_close($proc);
```

**코어를 PHP로 재작성하는 것은 권장하지 않습니다.** VS Code 확장 API가 JS/TS 전용이고,
tree-sitter가 WASM/Node 바인딩 기반이며, 장시간 스트리밍 응답이 PHP-FPM 모델과 맞지 않기 때문입니다.
PHP는 **Dirac을 감싸는 바깥 레이어**(수집·집계·관리 UI)로 사용하는 편이 효율적입니다.

---

## 11. 개발 / 기여

```bash
npm run install:all
npm run protos          # 필수: protobuf 코드 생성
npm run compile
npm run lint            # biome
```

기타 유용한 스크립트:

```bash
npm run dev             # protos + watch
npm run cli:build       # CLI 빌드
npm run cli:watch       # CLI watch
npm run check-types     # 타입 검사
npm run format:fix      # 포맷 자동 수정
npm run clean:all       # 빌드 산출물 + 의존성 제거
```

기여 가이드는 [CONTRIBUTING.md](../CONTRIBUTING.md) 를 참고하세요.

---

## 12. 한 줄 정리

> **API 키만 있으면 구독료 없이 쓸 수 있는 오픈소스 AI 코딩 에이전트.
> 해시 앵커 편집 · AST 조작 · Utility 모델 분리로 같은 작업을 더 정확하고 더 저렴하게 처리한다.**

- 저장소: https://github.com/bmshin94/dirac
- 원본: https://github.com/dirac-run/dirac
