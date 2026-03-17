# NemoClaw 소스코드 상세 분석 리포트

> 분석 대상: NemoClaw v0.1.0 (Apache-2.0)
> 리포지토리: https://github.com/NVIDIA/NemoClaw
> 분석일: 2026-03-17

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [디렉토리 구조와 아키텍처](#2-디렉토리-구조와-아키텍처)
3. [핵심 모듈 분석](#3-핵심-모듈-분석)
4. [주요 클래스와 함수](#4-주요-클래스와-함수)
5. [의존성 및 기술 스택](#5-의존성-및-기술-스택)
6. [컴포넌트 간 상호작용 방식](#6-컴포넌트-간-상호작용-방식)
7. [주요 알고리즘 및 기법](#7-주요-알고리즘-및-기법)
8. [네트워크 정책, 샌드박스, 인퍼런스 프로파일 구현](#8-네트워크-정책-샌드박스-인퍼런스-프로파일-구현)
9. [OpenClaw 플러그인 통합 방식](#9-openclaw-플러그인-통합-방식)
10. [데이터 흐름 및 상태 관리](#10-데이터-흐름-및-상태-관리)
11. [보안 설계](#11-보안-설계)
12. [설치 및 배포 파이프라인](#12-설치-및-배포-파이프라인)

---

## 1. 프로젝트 개요

NemoClaw는 **NVIDIA가 개발한 OpenClaw 플러그인**으로, OpenClaw(AI 코딩 에이전트)를 **OpenShell 샌드박스** 안에서 격리 실행하면서 NVIDIA NIM(NVIDIA Inference Microservices) 기반 추론을 연결하는 도구이다.

### 핵심 가치
- **보안 격리**: OpenClaw를 Landlock + seccomp + netns 샌드박스 안에서 실행하여, 에이전트의 파일시스템/네트워크 접근을 세밀하게 제어
- **NVIDIA 추론 통합**: NVIDIA Build API, NCP(NVIDIA Cloud Partner), 로컬 NIM, vLLM, Ollama 등 다양한 추론 엔드포인트를 단일 인터페이스로 관리
- **마이그레이션**: 기존 호스트에 설치된 OpenClaw를 손실 없이 샌드박스로 이전하고, 필요시 원복 가능

### 아키텍처 철학: "Thin Plugin, Versioned Blueprint"
- **Thin Plugin** (TypeScript): CLI 등록, 설정 관리, 사용자 인터랙션 담당 (~3KB 컴파일됨)
- **Versioned Blueprint** (Python + YAML): 실제 오케스트레이션 로직을 독립 아티팩트로 관리하여 플러그인 업데이트 없이 블루프린트만 교체 가능

---

## 2. 디렉토리 구조와 아키텍처

```
NemoClaw/
├── bin/                          # 호스트 CLI 진입점 (JavaScript)
│   ├── nemoclaw.js               # 메인 CLI 디스패처 (367줄)
│   └── lib/                      # CLI 헬퍼 라이브러리
│       ├── runner.js             # 명령 실행 유틸리티
│       ├── registry.js           # 샌드박스 레지스트리 (JSON 기반)
│       ├── onboard.js            # 7단계 대화형 온보딩 위저드
│       ├── policies.js           # 네트워크 정책 프리셋 관리
│       ├── nim.js                # NIM 컨테이너 라이프사이클
│       ├── credentials.js        # 자격증명 저장/조회
│       └── nim-images.json       # NIM 모델 메타데이터
│
├── nemoclaw/                     # TypeScript OpenClaw 플러그인
│   ├── src/
│   │   ├── index.ts              # 플러그인 진입점 + SDK 타입 정의
│   │   ├── cli.ts                # Commander.js 서브커맨드 등록
│   │   ├── commands/             # CLI 명령 구현
│   │   │   ├── launch.ts         # 신규 설치 (plan → apply)
│   │   │   ├── migrate.ts        # 호스트→샌드박스 마이그레이션
│   │   │   ├── eject.ts          # 샌드박스→호스트 원복
│   │   │   ├── onboard.ts        # 추론 엔드포인트 설정
│   │   │   ├── status.ts         # 상태 표시
│   │   │   ├── logs.ts           # 로그 스트리밍
│   │   │   ├── connect.ts        # 샌드박스 셸 접속
│   │   │   ├── slash.ts          # 채팅 /nemoclaw 핸들러
│   │   │   └── migration-state.ts # 마이그레이션 상태 관리 (680줄)
│   │   ├── blueprint/            # 블루프린트 인프라
│   │   │   ├── resolve.ts        # 버전 해석 + 캐시
│   │   │   ├── verify.ts         # SHA-256 다이제스트 검증
│   │   │   ├── fetch.ts          # 레지스트리 다운로드 (스텁)
│   │   │   ├── exec.ts           # Python 러너 실행
│   │   │   └── state.ts          # 상태 영속화
│   │   └── onboard/              # 온보딩 헬퍼
│   │       ├── config.ts         # 온보드 설정 영속화
│   │       ├── prompt.ts         # readline 기반 프롬프트
│   │       └── validate.ts       # API 키 검증
│   ├── dist/                     # 컴파일된 JS + 선언 파일
│   ├── package.json              # 플러그인 npm 패키지
│   ├── tsconfig.json             # TypeScript 설정
│   └── openclaw.plugin.json      # OpenClaw 플러그인 매니페스트
│
├── nemoclaw-blueprint/           # 버전 관리되는 블루프린트 아티팩트
│   ├── blueprint.yaml            # 오케스트레이션 매니페스트
│   ├── policies/
│   │   ├── openclaw-sandbox.yaml # 기본 네트워크 정책
│   │   └── presets/              # 9개 서비스별 프리셋
│   ├── orchestrator/
│   │   ├── runner.py             # 블루프린트 plan/apply/rollback 실행기
│   │   └── __init__.py
│   └── migrations/
│       └── snapshot.py           # 상태 마이그레이션 로직
│
├── scripts/                      # 설치/배포 스크립트
│   ├── install.sh                # curl-pipe-bash 설치기
│   ├── onboard.js                # 레거시 온보드 위저드
│   ├── brev-setup.sh             # Brev VM 배포
│   ├── start-services.sh         # 서비스 오케스트레이션
│   ├── fix-coredns.sh            # Colima DNS 패치
│   └── ...
│
├── test/                         # 테스트 스위트
├── Dockerfile                    # 샌드박스 Docker 이미지
├── package.json                  # 루트 패키지
└── README.md
```

### 아키텍처 다이어그램 (레이어 구조)

```
┌─────────────────────────────────────────────────────────┐
│                    사용자 인터페이스                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ nemoclaw CLI │  │ openclaw CLI │  │ /nemoclaw    │   │
│  │ (bin/)       │  │ (nemoclaw/)  │  │ (slash cmd)  │   │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘   │
├─────────┼──────────────────┼─────────────────┼───────────┤
│         │        플러그인 코어 (TypeScript)     │          │
│         │  ┌──────────────────────────────────┐│          │
│         │  │ commands/ │ blueprint/ │ onboard/ ││          │
│         │  └──────────┬───────────────────────┘│          │
├─────────┼─────────────┼────────────────────────┼──────────┤
│         │    블루프린트 실행기 (Python)           │          │
│         │  ┌──────────────────────────────────┐│          │
│         │  │  orchestrator/runner.py           ││          │
│         │  │  (plan → apply → rollback)        ││          │
│         │  └──────────┬───────────────────────┘│          │
├─────────┼─────────────┼────────────────────────┼──────────┤
│         │        OpenShell 인프라                │          │
│  ┌──────┴──────┐  ┌───┴────┐  ┌───────┐  ┌────┴────┐    │
│  │  Sandbox    │  │Provider│  │Policy  │  │Inference│    │
│  │  (Docker)   │  │(NIM)   │  │(YAML)  │  │(Route)  │    │
│  └─────────────┘  └────────┘  └────────┘  └─────────┘    │
└─────────────────────────────────────────────────────────┘
```

---

## 3. 핵심 모듈 분석

### 3.1 `bin/nemoclaw.js` — 호스트 CLI 디스패처

**역할**: NemoClaw의 주 진입점. npm 글로벌 설치 시 `nemoclaw` 명령으로 실행된다.

**명령 구조**:
- **글로벌 명령**: `onboard`, `list`, `deploy`, `setup`, `start`, `stop`, `status`
- **샌드박스 스코프 명령**: `nemoclaw <sandbox-name> <action>` 패턴
  - `connect`, `status`, `logs`, `policy-add`, `policy-list`, `destroy`

**핵심 설계**:
```
argv[2] ─── GLOBAL_COMMANDS에 있으면 ──→ 글로벌 핸들러 (onboard, deploy 등)
         │
         └── 없으면 ──→ registry.getSandbox(cmd)로 조회
                         │
                         ├── 존재하면 ──→ 샌드박스 스코프 액션 (connect, status 등)
                         └── 없으면 ──→ 에러 + 등록된 샌드박스 이름 제안
```

**`deploy` 함수 (51~133줄)**: Brev VM에 원격 배포하는 전체 파이프라인:
1. Brev 인스턴스 생성 (GPU 지정)
2. SSH 연결 대기 (최대 60회 × 3초)
3. rsync로 소스 동기화
4. 환경변수 파일 전송 (mode 600)
5. brev-setup.sh 실행
6. 서비스 시작 (Telegram 토큰 존재 시)
7. 샌드박스 접속

### 3.2 `bin/lib/onboard.js` — 7단계 온보딩 위저드

**역할**: 완전한 제로-투-러닝 온보딩을 7단계로 안내한다.

| 단계 | 함수 | 설명 |
|------|------|------|
| 1 | `preflight()` | Docker, openshell CLI, GPU 감지 |
| 2 | `startGateway()` | OpenShell 게이트웨이 시작 + CoreDNS 패치 |
| 3 | `createSandbox()` | Dockerfile 빌드 → 샌드박스 생성 → 포트 포워딩 |
| 4 | `setupNim()` | NIM/vLLM/Ollama 중 추론 방식 선택 |
| 5 | `setupInference()` | openshell provider create + inference set |
| 6 | `setupOpenclaw()` | 샌드박스 내 OpenClaw 설정 |
| 7 | `setupPolicies()` | 네트워크 정책 프리셋 적용 (자동감지 포함) |

**GPU 자동감지 흐름** (`nim.js:detectGpu()`):
```
nvidia-smi ──→ NVIDIA GPU (VRAM 쿼리)
         │
         └── GB10 감지 ──→ DGX Spark (통합 메모리, free -m로 대체)
                    │
                    └── system_profiler ──→ Apple Silicon (nimCapable=false)
                                      │
                                      └── null (GPU 없음)
```

### 3.3 `nemoclaw/src/` — TypeScript 플러그인 코어

#### 3.3.1 `index.ts` — 플러그인 등록

**역할**: OpenClaw 플러그인 SDK와 연동하는 진입점.

**핵심 구현**:
- **타입 스텁 정의**: `OpenClawPluginApi`, `PluginCommandContext`, `PluginCommandResult` 등 11개 인터페이스를 로컬에 정의 (빌드 시 SDK 패키지 접근 불가하므로)
- **`register()` 함수** (179~259줄): 플러그인 등록의 핵심
  1. `/nemoclaw` 슬래시 커맨드 등록 (채팅 인터페이스)
  2. `openclaw nemoclaw` CLI 서브커맨드 등록 (Commander.js)
  3. `nvidia-nim` 프로바이더 등록 (Nemotron 모델 4개 + Bearer 인증)
  4. 초기화 배너 출력

**등록되는 Nemotron 모델**:
| 모델 ID | 컨텍스트 윈도우 | 최대 출력 |
|---------|----------------|----------|
| nvidia/nemotron-3-super-120b-a12b | 131,072 | 8,192 |
| nvidia/llama-3.1-nemotron-ultra-253b-v1 | 131,072 | 4,096 |
| nvidia/llama-3.3-nemotron-super-49b-v1.5 | 131,072 | 4,096 |
| nvidia/nemotron-3-nano-30b-a3b | 131,072 | 4,096 |

#### 3.3.2 `commands/migrate.ts` — 호스트→샌드박스 마이그레이션

**역할**: 가장 복잡한 명령. 기존 호스트 OpenClaw 설치를 샌드박스로 안전하게 이전한다.

**마이그레이션 파이프라인** (297줄):

```
detectHostOpenClaw()
  │
  ▼
resolveBlueprint() → verifyBlueprintDigest()
  │
  ▼
execBlueprint("plan") → execBlueprint("apply")
  │
  ▼
createSnapshotBundle()
  │
  ▼
buildMigrationArchives()        # tar 아카이브 생성 (심볼릭 링크 보존)
  │
  ▼
syncSnapshotBundleIntoSandbox() # openshell sandbox cp + tar 추출
  │
  ▼
verifySandboxMigration()        # 샌드박스 내부에서 Node.js 스크립트로 검증
  │
  ▼
saveState()                     # 롤백 스냅샷 경로 저장
```

**아카이브 동기화 함수** (`syncArchive`, 204~219줄):
1. `openshell sandbox cp`로 tar 파일을 샌드박스로 복사
2. 샌드박스 내에서 `mkdir -p` + `tar -xf`로 추출
3. 경로별 독립 아카이브 (state.tar + 각 external root별 .tar)

**검증 함수** (`verifySandboxMigration`, 221~266줄):
- 샌드박스 내에서 Node.js 스크립트를 실행하여:
  - 마이그레이션된 상태 디렉토리 존재 확인
  - config 파일의 경로 바인딩이 올바르게 재작성되었는지 확인
  - 심볼릭 링크가 보존되었는지 확인

#### 3.3.3 `commands/migration-state.ts` — 마이그레이션 상태 관리

**역할**: 프로젝트에서 가장 긴 파일 (680줄). 호스트 OpenClaw 상태 감지, 스냅샷 생성/복원의 모든 로직을 담당.

**`detectHostOpenClaw()` (370~466줄)**:
- `OPENCLAW_HOME`, `OPENCLAW_STATE_DIR`, `OPENCLAW_CONFIG_PATH` 환경변수 우선 참조
- `~/.openclaw` 디렉토리 존재 확인
- JSON5 형식 config 파싱 (유연한 문법 지원)
- 외부 루트 수집: workspace, agentDir, skillsExtraDir
- extensions, skills, hooks 디렉토리 감지

**`collectExternalRoots()` (223~368줄)**:
- OpenClaw config에서 `agents.defaults.workspace`, `agents.list[*].workspace`, `agents.list[*].agentDir`, `skills.load.extraDirs` 경로를 파싱
- `registerRoot()`로 중복 제거하며 등록 (같은 경로는 바인딩만 추가)
- 상태 디렉토리 내부 경로는 제외 (`isWithinRoot`)
- 필수/선택 루트 구분 → 필수 루트 누락 시 에러, 선택 루트 누락 시 경고
- 심볼릭 링크 수집 및 보존

**`createSnapshotBundle()` (552~618줄)**:
- 타임스탬프 기반 디렉토리 생성 (`~/.nemoclaw/snapshots/<timestamp>/`)
- `cpSync`으로 상태 디렉토리 복사
- 외부 config 별도 보존
- 각 외부 루트 복사 + 심볼릭 링크 수집
- `prepareSandboxState()`로 config 경로를 샌드박스 경로로 재작성

**`setConfigValue()` (489~531줄)**: 점-표기법 경로로 중첩 객체의 값을 설정하는 범용 함수.
- `agents.list[0].workspace` 같은 복합 경로 지원
- 배열 인덱스와 객체 키를 자동 구분하여 중간 구조 생성

**`restoreSnapshotToHost()` (647~679줄)**:
- 기존 상태 디렉토리를 `.nemoclaw-archived-<timestamp>`로 이름 변경
- 스냅샷에서 복원
- 외부 config 복원

#### 3.3.4 `commands/onboard.ts` — 추론 엔드포인트 설정

**역할**: 추론 엔드포인트를 대화형/비대화형으로 설정하는 430줄 명령.

**지원 엔드포인트 6종**:

| 타입 | 프로바이더 이름 | 프로파일 | 인증 환경변수 |
|------|---------------|---------|-------------|
| build | nvidia-nim | default | NVIDIA_API_KEY |
| ncp | nvidia-ncp | ncp | NVIDIA_API_KEY |
| nim-local | nim-local | nim-local | NIM_API_KEY |
| vllm | vllm-local | vllm | OPENAI_API_KEY |
| ollama | ollama-local | ollama | OPENAI_API_KEY |
| custom | nvidia-ncp | ncp | NVIDIA_API_KEY |

**Ollama 자동감지** (112~116줄):
```typescript
function detectOllama(): { installed: boolean; running: boolean } {
  const installed = testCommand("command -v ollama >/dev/null 2>&1");
  const running = testCommand("curl -sf http://localhost:11434/api/tags >/dev/null 2>&1");
  return { installed, running };
}
```
- Ollama가 실행 중이면 자동으로 `ollama` 엔드포인트 타입 선택

**게이트웨이 프록시 URL** (`HOST_GATEWAY_URL = "http://host.openshell.internal"`):
- vLLM, Ollama 등 로컬 서비스는 `host.openshell.internal` 게이트웨이를 통해 샌드박스에서 접근
- 샌드박스 네트워크 격리를 유지하면서 호스트 로컬 서비스에 접근 가능

**비대화형 모드**: `--api-key`, `--endpoint`, `--model` 플래그를 모두 제공하면 프롬프트 없이 실행

#### 3.3.5 `commands/launch.ts` — 신규 설치

**역할**: OpenClaw를 OpenShell 샌드박스 안에 새로 부트스트랩한다 (143줄).

**흐름**:
1. 기존 호스트 OpenClaw 감지 → 있으면 `migrate` 권장, 없으면 `openshell sandbox create` 직접 사용 권장
2. `--force` 플래그로 강제 진행 시:
   - 블루프린트 해석 → 다이제스트 검증 → 호환성 확인
   - `plan` → `apply` 실행
   - 상태 저장

**버전 호환성 확인** (`verify.ts:checkCompatibility()`):
```typescript
function satisfiesMinVersion(actual: string, minimum: string): boolean {
  const aParts = actual.split(".").map(Number);
  const mParts = minimum.split(".").map(Number);
  for (let i = 0; i < Math.max(aParts.length, mParts.length); i++) {
    const a = aParts[i] ?? 0;
    const m = mParts[i] ?? 0;
    if (a > m) return true;
    if (a < m) return false;
  }
  return true;
}
```

#### 3.3.6 `commands/eject.ts` — 샌드박스 원복

**역할**: 샌드박스 배포를 롤백하고 호스트 OpenClaw 설치를 복원한다 (95줄).

**3단계 원복 프로세스**:
1. **블루프린트 롤백**: `execBlueprint("rollback")` — 샌드박스 중지/제거
2. **호스트 복원**: `restoreSnapshotToHost()` — 스냅샷에서 `~/.openclaw` 복원
3. **상태 초기화**: `clearState()` — NemoClaw 상태 파일 리셋

### 3.4 `nemoclaw/src/blueprint/` — 블루프린트 인프라

#### 3.4.1 `resolve.ts` — 블루프린트 버전 해석

**캐시 경로**: `~/.nemoclaw/blueprints/<version>/`

**해석 전략**:
```
version === "latest" ──→ 항상 레지스트리에서 fetch
version !== "latest" ──→ 로컬 캐시 확인
                          │
                          ├── 캐시 히트 ──→ 캐시된 매니페스트 반환
                          └── 캐시 미스 ──→ 레지스트리에서 fetch
```

**매니페스트 파싱** (`parseManifestHeader`): 정규식 기반 간이 YAML 파서
```typescript
const get = (key: string): string => {
  const match = raw.match(new RegExp(`^${key}:\\s*(.+)$`, "m"));
  return match?.[1]?.trim() ?? "";
};
```

#### 3.4.2 `verify.ts` — 무결성 검증

**SHA-256 디렉토리 다이제스트** (`computeDirectoryDigest`):
1. 블루프린트 디렉토리 내 모든 파일을 재귀 수집
2. 파일명 알파벳순 정렬
3. 각 파일의 상대 경로 + 내용을 순차적으로 SHA-256 해시에 업데이트
4. 최종 hex 다이제스트 반환

**검증 논리**: 매니페스트의 `digest` 필드가 비어있으면 검증을 건너뜀 (개발 모드). 값이 있으면 계산된 다이제스트와 비교.

#### 3.4.3 `exec.ts` — 블루프린트 실행기

**역할**: Python 블루프린트 러너를 서브프로세스로 실행한다.

**실행 프로토콜**:
```
TypeScript ──spawn──→ python3 runner.py <action> --profile <profile> [옵션들]
                        │
                        ├── stdout: "PROGRESS:<0-100>:<label>" (진행률)
                        ├── stdout: "RUN_ID:<id>" (실행 ID)
                        ├── stderr → logger.warn()
                        └── exit code: 0=성공, 非0=실패
```

**환경변수 전달**:
- `NEMOCLAW_BLUEPRINT_PATH`: 블루프린트 디렉토리 경로
- `NEMOCLAW_ACTION`: 실행 액션 (plan/apply/status/rollback)

### 3.5 `nemoclaw-blueprint/` — 블루프린트 아티팩트

#### 3.5.1 `blueprint.yaml` — 오케스트레이션 매니페스트

**구조**:
```yaml
version: "0.1.0"
min_openshell_version: "0.1.0"
min_openclaw_version: "2026.3.0"
profiles: [default, ncp, nim-local, vllm]

components:
  sandbox:          # Docker 이미지 + 포트 포워딩
  inference:        # 프로파일별 추론 설정 (6종)
  policy:           # 기본 정책 + NIM 서비스 추가
```

**추론 프로파일 4종**:

| 프로파일 | provider_type | 엔드포인트 | 인증 |
|---------|--------------|----------|------|
| default | nvidia | integrate.api.nvidia.com/v1 | NVIDIA_API_KEY |
| ncp | nvidia | (동적) | NVIDIA_API_KEY |
| nim-local | openai | nim-service.local:8000/v1 | NIM_API_KEY |
| vllm | openai | localhost:8000/v1 | OPENAI_API_KEY (기본: "dummy") |

#### 3.5.2 `orchestrator/runner.py` — Python 실행기

**역할**: 블루프린트의 plan/apply/status/rollback 4가지 액션을 실행한다 (347줄).

**`action_plan()`**:
1. 블루프린트 YAML 로드
2. 프로파일 존재 확인
3. openshell CLI 사용 가능 확인
4. 플랜 JSON 생성 (sandbox, inference, policy_additions)

**`action_apply()`**:
1. `openshell sandbox create --from <image> --name <name> --forward <port>` 실행
2. `openshell provider create --name <provider> --type openai --credential ... --config ...` 실행
3. `openshell inference set --provider <provider> --model <model>` 실행
4. 실행 상태를 `~/.nemoclaw/state/runs/<run-id>/plan.json`에 저장

**`action_rollback()`**:
1. 실행 ID로 플랜 로드
2. `openshell sandbox stop` → `openshell sandbox remove`
3. `rolled_back` 마커 파일 기록

**보안 주의점**: `subprocess.run()`에 `shell=True`를 사용하지 않음 — 명령 주입 방지.

### 3.6 `bin/lib/` — CLI 헬퍼 라이브러리

#### 3.6.1 `registry.js` — 샌드박스 레지스트리

**저장 위치**: `~/.nemoclaw/sandboxes.json` (mode 600)

**데이터 구조**:
```json
{
  "sandboxes": {
    "my-assistant": {
      "name": "my-assistant",
      "createdAt": "2026-03-17T...",
      "model": "nvidia/nemotron-3-super-120b-a12b",
      "nimContainer": "nemoclaw-nim-my-assistant",
      "provider": "nvidia-nim",
      "gpuEnabled": true,
      "policies": ["pypi", "npm", "telegram"]
    }
  },
  "defaultSandbox": "my-assistant"
}
```

**API**: `registerSandbox`, `updateSandbox`, `removeSandbox`, `listSandboxes`, `setDefault`

#### 3.6.2 `nim.js` — NIM 컨테이너 관리

**GPU 감지 알고리즘** (`detectGpu()`):
1. `nvidia-smi --query-gpu=memory.total` → NVIDIA GPU (VRAM 직접 쿼리)
2. `nvidia-smi --query-gpu=name` → GB10 감지 (DGX Spark, 통합 메모리)
3. `system_profiler SPDisplaysDataType` → Apple Silicon (nimCapable=false로 표시)

**NIM 컨테이너 라이프사이클**:
- `pullNimImage(model)`: `nim-images.json`에서 이미지 조회 → `docker pull`
- `startNimContainer(sandbox, model)`: `docker run -d --gpus all -p 8000:8000 --shm-size 16g`
- `waitForNimHealth()`: 5초 간격으로 `curl http://localhost:8000/v1/models` 폴링 (최대 300초)
- `stopNimContainer()`: `docker stop` + `docker rm`
- 컨테이너 이름 규칙: `nemoclaw-nim-<sandbox-name>`

#### 3.6.3 `policies.js` — 정책 프리셋 관리

**프리셋 적용 알고리즘** (`applyPreset()`, 72~166줄):
1. 프리셋 YAML에서 `network_policies:` 섹션 추출
2. 현재 정책을 `openshell policy get --full`로 조회
3. **3분기 병합 전략**:
   - 기존 정책에 `network_policies:` 있음 → 해당 섹션 끝에 삽입
   - 기존 정책에 `network_policies:` 없음 → 새 섹션으로 추가
   - 기존 정책 없음 → 전체 새로 생성 (`version: 1` 추가)
4. 임시 파일로 저장 → `openshell policy set --policy <file> --wait <sandbox>` 적용
5. 레지스트리에 적용된 프리셋 이름 기록

#### 3.6.4 `credentials.js` — 자격증명 관리

**저장**: `~/.nemoclaw/credentials.json` (디렉토리 mode 700, 파일 mode 600)
**조회 우선순위**: 환경변수 → credentials.json
**지원 키**: `NVIDIA_API_KEY`, `GITHUB_TOKEN`, `TELEGRAM_BOT_TOKEN` 등

#### 3.6.5 `runner.js` — 명령 실행 유틸리티

**Colima Docker 소켓 자동감지**:
```javascript
const candidates = [
  path.join(home, ".colima/default/docker.sock"),      // 레거시
  path.join(home, ".config/colima/default/docker.sock"), // XDG
];
```
- `DOCKER_HOST` 미설정 시 자동으로 Colima 소켓 감지 → macOS 호환성 보장

---

## 4. 주요 클래스와 함수

### TypeScript 인터페이스

| 인터페이스 | 파일 | 용도 |
|-----------|------|------|
| `OpenClawPluginApi` | index.ts:120 | OpenClaw 플러그인 SDK API 정의 |
| `NemoClawConfig` | index.ts:139 | 플러그인 설정 (blueprint, sandbox, provider) |
| `HostOpenClawState` | migration-state.ts:42 | 호스트 OpenClaw 상태 스냅샷 |
| `SnapshotBundle` | migration-state.ts:69 | 마이그레이션 스냅샷 번들 |
| `SnapshotManifest` | migration-state.ts:58 | 스냅샷 메타데이터 |
| `MigrationExternalRoot` | migration-state.ts:31 | 외부 루트 디렉토리 정보 |
| `BlueprintManifest` | resolve.ts:9 | 블루프린트 매니페스트 |
| `BlueprintRunResult` | exec.ts:22 | 블루프린트 실행 결과 |
| `NemoClawState` | state.ts:9 | 영속 상태 (runId, action, snapshot) |
| `NemoClawOnboardConfig` | onboard/config.ts | 온보드 설정 |
| `ProviderPlugin` | index.ts:99 | 모델 프로바이더 등록 형식 |
| `ModelProviderEntry` | index.ts:85 | 모델 카탈로그 항목 |

### 핵심 함수

| 함수 | 파일 | 기능 |
|------|------|------|
| `register()` | index.ts:179 | 플러그인 등록 진입점 |
| `cliMigrate()` | migrate.ts:32 | 호스트→샌드박스 마이그레이션 |
| `cliLaunch()` | launch.ts:19 | 신규 샌드박스 부트스트랩 |
| `cliEject()` | eject.ts:20 | 원복 |
| `cliOnboard()` | onboard.ts:145 | 추론 엔드포인트 설정 |
| `cliStatus()` | status.ts:17 | 상태 조회 |
| `detectHostOpenClaw()` | migration-state.ts:370 | 호스트 OpenClaw 감지 |
| `createSnapshotBundle()` | migration-state.ts:552 | 스냅샷 생성 |
| `restoreSnapshotToHost()` | migration-state.ts:647 | 스냅샷 복원 |
| `collectExternalRoots()` | migration-state.ts:223 | 외부 루트 수집 |
| `resolveBlueprint()` | resolve.ts:62 | 블루프린트 버전 해석 |
| `verifyBlueprintDigest()` | verify.ts:16 | SHA-256 무결성 검증 |
| `execBlueprint()` | exec.ts:34 | Python 러너 실행 |
| `handleSlashCommand()` | slash.ts:17 | 채팅 슬래시 명령 |
| `validateApiKey()` | validate.ts:10 | API 키 유효성 검증 |
| `onboard()` | bin/lib/onboard.js:466 | 7단계 온보딩 위저드 |
| `applyPreset()` | bin/lib/policies.js:72 | 네트워크 정책 프리셋 적용 |
| `detectGpu()` | bin/lib/nim.js:26 | GPU 감지 (NVIDIA/Apple) |
| `action_plan()` | runner.py:80 | 배포 계획 생성 |
| `action_apply()` | runner.py:138 | 배포 적용 |
| `action_rollback()` | runner.py:273 | 롤백 실행 |

---

## 5. 의존성 및 기술 스택

### 런타임 요구사항

| 요구사항 | 최소 버전 | 권장 버전 | 용도 |
|---------|----------|---------|------|
| Node.js | 20.0.0 | 22 | CLI + 플러그인 런타임 |
| npm | 10.0.0 | - | 패키지 설치 |
| Docker | - | 최신 | 샌드박스 컨테이너 |
| OpenShell CLI | 0.1.0 | - | 샌드박스/정책/추론 관리 |
| Python | 3.11+ | - | 블루프린트 러너 |
| OpenClaw | 2026.3.0 | 2026.3.11 | AI 코딩 에이전트 |

### npm 의존성

**루트 패키지**:
- `openclaw@2026.3.11` — AI 코딩 에이전트 (유일한 런타임 의존성)

**플러그인 패키지** (nemoclaw/):
- `commander` — CLI 인수 파싱
- `yaml` — YAML 파싱
- `json5` — JSON5 config 파싱 (주석, 후행 쉼표 허용)
- `tar` — tar 아카이브 생성 (심볼릭 링크 보존)

**개발 의존성**:
- `typescript` — 소스 컴파일
- `eslint`, `prettier` — 코드 린팅/포매팅
- `@types/node` — Node.js 타입 정의

### Python 의존성
- `pyyaml` — YAML 파싱 (pip install)

### 지원 플랫폼

| 플랫폼 | 아키텍처 | 특이사항 |
|-------|---------|---------|
| Linux (Ubuntu 22.04+) | x86_64, aarch64 | 기본 지원 |
| macOS | x86_64 (Intel), aarch64 (Apple Silicon) | Colima 또는 Docker Desktop |
| DGX Spark | aarch64 (GB10) | `setup-spark.sh`로 cgroup v2 + Docker 설정 |

---

## 6. 컴포넌트 간 상호작용 방식

### 6.1 명령 실행 흐름

```
사용자 ─── nemoclaw onboard ──→ bin/nemoclaw.js
                                   │
                                   ▼
                              bin/lib/onboard.js
                                   │
    ┌──────────────────────────────┼──────────────────────────────┐
    │                              │                              │
    ▼                              ▼                              ▼
bin/lib/nim.js              bin/lib/policies.js           bin/lib/registry.js
(GPU감지, NIM관리)           (정책 프리셋 적용)             (샌드박스 등록)
    │                              │                              │
    └──────────────────────────────┼──────────────────────────────┘
                                   │
                                   ▼
                            openshell CLI
                    (sandbox, provider, inference, policy)
```

### 6.2 플러그인 CLI 흐름

```
사용자 ─── openclaw nemoclaw migrate ──→ OpenClaw 호스트
                                            │
                                            ▼
                                     nemoclaw/src/cli.ts
                                            │
                                            ▼
                                  commands/migrate.ts
                                            │
                     ┌──────────────────────┼──────────────────────┐
                     │                      │                      │
                     ▼                      ▼                      ▼
            migration-state.ts       blueprint/resolve.ts    blueprint/exec.ts
            (상태 감지/스냅샷)        (버전 해석/검증)         (Python 러너 실행)
                     │                      │                      │
                     │                      │                      ▼
                     │                      │               runner.py
                     │                      │               (plan → apply)
                     │                      │                      │
                     └──────────────────────┼──────────────────────┘
                                            │
                                            ▼
                                     openshell CLI
                              (sandbox cp, sandbox connect)
```

### 6.3 이중 CLI 구조

NemoClaw는 **두 가지 독립적인 CLI**를 제공한다:

| CLI | 진입점 | 컨텍스트 | 대상 사용자 |
|-----|-------|---------|-----------|
| `nemoclaw` | bin/nemoclaw.js | 호스트 (npm global) | 초기 설정, 일상 관리 |
| `openclaw nemoclaw` | nemoclaw/src/cli.ts | OpenClaw 플러그인 내부 | 마이그레이션, 고급 관리 |

**분리 이유**: 호스트 CLI는 OpenClaw 없이도 독립 실행 가능 (설치/온보딩). 플러그인 CLI는 OpenClaw 생태계 안에서 통합 실행.

### 6.4 TypeScript ↔ Python 상호작용

```
exec.ts ──spawn──→ python3 runner.py <action>
  │                        │
  │  env:                  │  stdout:
  │  NEMOCLAW_BLUEPRINT_PATH  │  PROGRESS:50:Creating sandbox
  │  NEMOCLAW_ACTION       │  RUN_ID:nc-20260317-143022-a1b2c3d4
  │                        │
  │  args:                 │  exit code:
  │  --profile default     │  0 (성공) / 非0 (실패)
  │  --json                │
  │  --plan <run-id>       │
  └────────────────────────┘
```

---

## 7. 주요 알고리즘 및 기법

### 7.1 디렉토리 다이제스트 알고리즘

```typescript
// verify.ts:71-79
function computeDirectoryDigest(dirPath: string): string {
  const hash = createHash("sha256");
  const files = collectFiles(dirPath).sort(); // 알파벳순 정렬 → 결정적
  for (const file of files) {
    hash.update(file);                        // 상대 경로 포함
    hash.update(readFileSync(join(dirPath, file)));  // 파일 내용
  }
  return hash.digest("hex");
}
```

**특징**:
- 파일명 정렬로 결정적(deterministic) 해시 보장
- 상대 경로를 해시에 포함하여 파일 이동 감지
- 디렉토리 구조 변경도 감지 가능

### 7.2 심볼릭 링크 보존 마이그레이션

```typescript
// migration-state.ts:155-176
function collectSymlinkPaths(rootPath: string): string[] {
  const symlinks: string[] = [];
  function walk(currentPath: string, relativePath: string): void {
    const stat = lstatSync(currentPath);  // lstat: 심볼릭 링크 자체 정보
    if (stat.isSymbolicLink()) {
      symlinks.push(relativePath || ".");
      return;  // 심볼릭 링크 하위는 탐색하지 않음
    }
    if (!stat.isDirectory()) return;
    for (const entry of readdirSync(currentPath)) {
      walk(path.join(currentPath, entry), ...);
    }
  }
  walk(rootPath, "");
  return symlinks.sort();
}
```

**tar 아카이브 옵션**:
```typescript
await createTar({
  cwd: sourceDir,
  file: archivePath,
  portable: true,   // 이식성 보장
  follow: false,     // 심볼릭 링크를 따라가지 않음 (보존)
  noMtime: true,     // 타임스탬프 무시 (재현 가능한 아카이브)
}, ["."]);
```

### 7.3 Config 경로 재작성 알고리즘

마이그레이션 시 호스트 경로를 샌드박스 경로로 변환:

```
호스트: /home/user/.openclaw/workspace
  ↓ (setConfigValue)
샌드박스: /sandbox/.nemoclaw/migration/workspaces/workspaces-default-workspace
```

`setConfigValue()`는 점-표기법 경로 (`agents.list[0].workspace`)를 파싱하여:
1. 각 세그먼트를 순회하며 중첩 객체/배열을 생성
2. 마지막 세그먼트에 새 값 할당
3. 배열 인덱스(`[N]`)와 객체 키를 자동 구분

### 7.4 네트워크 정책 병합 알고리즘

```javascript
// policies.js: 행 기반 YAML 병합
const lines = currentPolicy.split("\n");
for (const line of lines) {
  const isTopLevel = /^\S.*:/.test(line);

  if (line.trim().startsWith("network_policies:")) {
    inNetworkPolicies = true;  // 섹션 시작
  }

  if (inNetworkPolicies && isTopLevel && !inserted) {
    result.push(presetEntries);  // 다음 최상위 키 전에 삽입
    inserted = true;
  }
}
// network_policies가 마지막 섹션이면 끝에 추가
```

### 7.5 SemVer 호환성 비교

```typescript
// verify.ts:59-69 — 좌→우 세그먼트 비교
function satisfiesMinVersion(actual: string, minimum: string): boolean {
  for (let i = 0; i < Math.max(aParts.length, mParts.length); i++) {
    const a = aParts[i] ?? 0;  // 누락 세그먼트는 0으로 처리
    const m = mParts[i] ?? 0;
    if (a > m) return true;    // 조기 반환 (상위)
    if (a < m) return false;   // 조기 반환 (하위)
  }
  return true;  // 동일 → 만족
}
```

---

## 8. 네트워크 정책, 샌드박스, 인퍼런스 프로파일 구현

### 8.1 네트워크 정책 시스템

#### 기본 정책 (`openclaw-sandbox.yaml`)

**설계 원칙**: Deny-by-default. 핵심 기능에 필요한 최소한의 접근만 허용.

**파일시스템 정책**:
```yaml
filesystem_policy:
  read_only: [/usr, /lib, /proc, /dev/urandom, /app, /etc, /var/log]
  read_write: [/sandbox, /tmp, /dev/null]

landlock:
  compatibility: best_effort  # 커널 미지원 시 graceful degradation

process:
  run_as_user: sandbox
  run_as_group: sandbox
```

**네트워크 정책 계층**:

| 정책 이름 | 허용 호스트 | 바이너리 제한 | 용도 |
|----------|-----------|-------------|------|
| claude_code | api.anthropic.com, statsig.anthropic.com, sentry.io | /usr/local/bin/claude | Claude API |
| nvidia | integrate.api.nvidia.com, inference-api.nvidia.com | claude, openclaw | NVIDIA 추론 |
| github | github.com, api.github.com | gh, git | GitHub 접근 |
| clawhub | clawhub.com | openclaw만 (GET+POST) | 플러그인 디스커버리 |
| openclaw_api | openclaw.ai | openclaw만 (GET+POST) | 인증 |
| openclaw_docs | docs.openclaw.ai | openclaw만 (GET만) | 문서 |
| npm_registry | registry.npmjs.org | openclaw, npm | 패키지 설치 |
| telegram | api.telegram.org | (무제한) | 봇 알림 |

**핵심 보안 메커니즘**:
- **바이너리 제한**: 각 정책에 `binaries` 필드로 해당 엔드포인트에 접근 가능한 실행파일 제한
- **TLS 종료**: `tls: terminate`로 OpenShell이 TLS 인증서 검증을 대행
- **메서드 제한**: 일부 엔드포인트는 GET만 허용 (docs), 일부는 GET+POST (인증 흐름)
- **경로 패턴**: `/bot*/**` 같은 패턴으로 Telegram Bot API만 허용

#### 프리셋 정책 (9종)

모두 `nemoclaw-blueprint/policies/presets/` 디렉토리에 있으며, 동일한 구조:

```yaml
preset:
  name: <서비스명>
  description: "<설명>"

network_policies:
  <정책명>:
    name: <정책명>
    endpoints:
      - host: <호스트>
        port: 443
        protocol: rest
        enforcement: enforce
        tls: terminate
        rules:
          - allow: { method: <메서드>, path: "<경로패턴>" }
```

| 프리셋 | 허용 호스트 | 설명 |
|-------|-----------|------|
| telegram | api.telegram.org | Telegram Bot API (GET/POST /bot*/**) |
| slack | slack.com, api.slack.com, files.slack.com | Slack API |
| discord | discord.com, discordapp.com, gateway.discord.gg | Discord API + WebSocket |
| npm | registry.npmjs.org | npm 패키지 레지스트리 |
| pypi | pypi.org, files.pythonhosted.org | Python 패키지 |
| docker | registry-1.docker.io, auth.docker.io, ghcr.io | Docker 레지스트리 |
| jira | *.atlassian.net | Jira 이슈 트래커 |
| huggingface | huggingface.co, cdn-lfs.huggingface.co | HuggingFace Hub |
| outlook | outlook.office365.com, graph.microsoft.com, login.microsoftonline.com | Outlook/Exchange |

### 8.2 샌드박스 구현

#### Dockerfile 분석

```dockerfile
FROM node:22-slim

# 시스템 패키지: Python3, curl, git, iproute2 (네트워크 격리)
RUN apt-get install -y python3 python3-pip python3-venv curl git ca-certificates iproute2

# 샌드박스 사용자 (OpenShell 관례)
RUN groupadd -r sandbox && useradd -r -g sandbox -d /sandbox -s /bin/bash sandbox

# OpenClaw CLI 글로벌 설치
RUN npm install -g openclaw@2026.3.11

# 블루프린트 로컬 캐시에 배치
RUN mkdir -p /sandbox/.nemoclaw/blueprints/0.1.0 \
    && cp -r /opt/nemoclaw-blueprint/* /sandbox/.nemoclaw/blueprints/0.1.0/

# OpenClaw 기본 설정: NVIDIA 프로바이더, inference.local 프록시
RUN python3 -c "..." # openclaw.json 생성

# NemoClaw 플러그인 설치
RUN openclaw plugins install /opt/nemoclaw

USER sandbox
WORKDIR /sandbox
```

**핵심 설계 요소**:
- `sandbox` 사용자로 비특권 실행
- `/sandbox` 디렉토리에 격리된 홈
- `inference.local` 프록시를 통한 추론 라우팅 (API 키 노출 방지)
- `chmod 700 /sandbox/.openclaw`으로 설정 보호

#### 샌드박스 생성 흐름 (onboard.js)

```
빌드 컨텍스트 준비 (임시 디렉토리)
  ↓
Dockerfile + nemoclaw/ + nemoclaw-blueprint/ + scripts/ 복사
  ↓
openshell sandbox create --from <Dockerfile> --name <name> --policy <base-policy> [--gpu]
  ↓
openshell forward start --background 18789 <name>  # 대시보드 포트 포워딩
  ↓
registry.registerSandbox()  # 로컬 레지스트리에 등록
  ↓
빌드 컨텍스트 정리
```

### 8.3 인퍼런스 프로파일 시스템

#### 엔드포인트 타입별 설정 상세

**NVIDIA Build (기본)**:
```
엔드포인트: https://integrate.api.nvidia.com/v1
인증: NVIDIA_API_KEY (Bearer)
프로바이더: nvidia-nim (openai 호환)
모델: nvidia/nemotron-3-super-120b-a12b (기본)
```

**NCP (NVIDIA Cloud Partner)**:
```
엔드포인트: (동적 — 파트너별 URL)
인증: NVIDIA_API_KEY (Bearer)
프로바이더: nvidia-ncp
프로파일: ncp
특징: dynamic_endpoint: true
```

**로컬 NIM**:
```
엔드포인트: http://nim-service.local:8000/v1
인증: NIM_API_KEY
프로바이더: nim-local (openai 호환)
Docker: --gpus all --shm-size 16g
헬스체크: curl http://localhost:8000/v1/models
```

**vLLM**:
```
엔드포인트: http://host.openshell.internal:8000/v1
인증: OPENAI_API_KEY (기본: "dummy")
프로바이더: vllm-local
특징: host.openshell.internal 게이트웨이 프록시 사용
```

**Ollama**:
```
엔드포인트: http://host.openshell.internal:11434/v1
인증: OPENAI_API_KEY (기본: "ollama")
프로바이더: ollama-local
자동감지: curl -sf http://localhost:11434/api/tags
```

#### 프로바이더 등록 흐름

```
cliOnboard() ──→ execOpenShell(["provider", "create",
                    "--name", providerName,
                    "--type", "openai",
                    "--credential", `${credentialEnv}=${apiKey}`,
                    "--config", `OPENAI_BASE_URL=${endpointUrl}`])
                 │
                 ├── 성공 ──→ execOpenShell(["inference", "set",
                 │              "--provider", providerName,
                 │              "--model", model])
                 │
                 └── AlreadyExists ──→ execOpenShell(["provider", "update", ...])
```

---

## 9. OpenClaw 플러그인 통합 방식

### 9.1 플러그인 SDK 인터페이스

NemoClaw는 OpenClaw 플러그인 SDK (`openclaw/plugin-sdk`)의 인터페이스를 로컬에 스텁으로 정의한다. SDK는 OpenClaw 호스트 프로세스 내에서만 사용 가능하므로 빌드 시 직접 임포트할 수 없기 때문이다.

```typescript
export interface OpenClawPluginApi {
  id: string;
  name: string;
  version?: string;
  config: OpenClawConfig;
  pluginConfig?: Record<string, unknown>;
  logger: PluginLogger;
  registerCommand: (command: PluginCommandDefinition) => void;  // 슬래시 커맨드
  registerCli: (registrar: PluginCliRegistrar) => void;          // CLI 서브커맨드
  registerProvider: (provider: ProviderPlugin) => void;          // 모델 프로바이더
  registerService: (service: PluginService) => void;             // 백그라운드 서비스
  resolvePath: (input: string) => string;                        // 경로 해석
  on: (hookName: string, handler: (...args) => void) => void;   // 이벤트 훅
}
```

### 9.2 플러그인 매니페스트 (`openclaw.plugin.json`)

```json
{
  "id": "nemoclaw",
  "name": "NemoClaw",
  "version": "0.1.0",
  "configSchema": {
    "properties": {
      "blueprintVersion": { "default": "latest" },
      "blueprintRegistry": { "default": "ghcr.io/nvidia/nemoclaw-blueprint" },
      "sandboxName": { "default": "openclaw" },
      "inferenceProvider": { "default": "nvidia" }
    }
  }
}
```

### 9.3 등록 시퀀스

`register()` 함수가 호출되면:

1. **슬래시 커맨드 등록**: `/nemoclaw` → `handleSlashCommand()`
   - 채팅 인터페이스에서 `status`, `eject`, `onboard` 서브커맨드 제공
   - Markdown 형식으로 응답 반환

2. **CLI 서브커맨드 등록**: `registerCli()` → `registerCliCommands()`
   - Commander.js로 7개 서브커맨드 등록 (status, migrate, launch, connect, logs, eject, onboard)
   - 각 커맨드에 옵션 플래그 정의 (--json, --dry-run, --force 등)

3. **모델 프로바이더 등록**: `registerProvider()`
   - `nvidia-nim` 프로바이더 등록
   - aliases: ["nvidia", "nim"]
   - Nemotron 모델 4종 카탈로그
   - Bearer 인증 방식

4. **배너 출력**: 엔드포인트/모델 정보가 포함된 ASCII 박스

### 9.4 설정 주입 방식

```
openclaw.plugin.json
      │
      ▼ (OpenClaw 호스트가 파싱)
api.pluginConfig
      │
      ▼ (getPluginConfig()가 타입 안전하게 추출)
NemoClawConfig {
  blueprintVersion: "latest" | "<semver>",
  blueprintRegistry: "<OCI URL>",
  sandboxName: "<name>",
  inferenceProvider: "nvidia" | "<custom>"
}
```

---

## 10. 데이터 흐름 및 상태 관리

### 10.1 상태 파일 구조

```
~/.nemoclaw/
├── state/
│   └── nemoclaw.json          # 플러그인 상태 (lastRunId, lastAction, snapshot)
│   └── runs/
│       └── nc-<timestamp>-<uuid>/
│           └── plan.json      # 실행 계획 (Python 러너가 생성)
├── config.json                # 온보드 설정 (endpointType, model, profile)
├── credentials.json           # 자격증명 (mode 600)
├── sandboxes.json             # 샌드박스 레지스트리
├── blueprints/
│   └── 0.1.0/                 # 캐시된 블루프린트
│       ├── blueprint.yaml
│       ├── orchestrator/
│       └── policies/
├── snapshots/
│   └── <timestamp>/           # 마이그레이션 스냅샷 (영구)
│       ├── snapshot.json      # 매니페스트
│       ├── openclaw/          # 호스트 상태 복사본
│       ├── config/            # 외부 config (해당 시)
│       ├── external/          # 외부 루트 복사본
│       └── sandbox-bundle/    # 샌드박스용 준비된 번들
│           ├── openclaw/      # 경로 재작성된 상태
│           └── archives/      # tar 아카이브들
└── staging/
    └── <timestamp>/           # 임시 스냅샷 (--skip-backup 시)
```

### 10.2 상태 전이 다이어그램

```
(초기) ──── nemoclaw onboard ────→ config.json 생성
                                      │
         ┌────────────────────────────┘
         │
         ▼
  openclaw nemoclaw launch ────→ state.json { lastAction: "launch" }
         │
         ▼
  openclaw nemoclaw migrate ───→ state.json { lastAction: "migrate",
         │                                    migrationSnapshot: "<path>" }
         ▼
  openclaw nemoclaw eject ─────→ state.json 초기화
                                  호스트 ~/.openclaw 복원
```

---

## 11. 보안 설계

### 11.1 다중 계층 격리

| 계층 | 메커니즘 | 구현 위치 |
|------|---------|---------|
| 파일시스템 | Landlock LSM | openclaw-sandbox.yaml (`landlock.compatibility: best_effort`) |
| 시스템콜 | seccomp | OpenShell 샌드박스 런타임 |
| 네트워크 | netns + 정책 프록시 | openclaw-sandbox.yaml (`network_policies`) |
| 프로세스 | 비특권 사용자 | Dockerfile (`USER sandbox`) |
| 자격증명 | 파일 권한 | credentials.js (mode 600, 디렉토리 mode 700) |
| TLS | 종료 프록시 | `tls: terminate` 정책 |
| 바이너리 제한 | 정책별 실행파일 화이트리스트 | `binaries` 필드 |

### 11.2 자격증명 보호

- `~/.nemoclaw/credentials.json`: 파일 mode 0o600, 디렉토리 mode 0o700
- Brev 배포 시 `.env` 파일: mode 0o600으로 생성, 전송 후 즉시 삭제
- API 키 마스킹: `nvapi-****XXXX` 형태로 로그 출력
- 샌드박스 내 `apiKey: 'openshell-managed'` — 실제 키는 OpenShell 프로바이더가 관리

### 11.3 명령 주입 방지

- Python 러너: `subprocess.run(args, shell=False)` — argv 리스트 방식
- TypeScript: `execFileSync("openshell", [...args])` — 셸 미사용
- 셸 인용: `shellQuote()` 함수로 싱글쿼트 이스케이프
- 샌드박스 내 스크립트: `sh -lc` + 인용된 인수

---

## 12. 설치 및 배포 파이프라인

### 12.1 curl-pipe-bash 설치 (`scripts/install.sh`)

```
curl -fsSL .../install.sh | bash
      │
      ▼
OS/아키텍처 감지 (Darwin/Linux × x86_64/aarch64)
      │
      ▼
Node.js 버전 관리자 감지 (asdf > nvm > fnm > brew > nodesource)
      │
      ▼
Node.js 22 설치 (필요 시)
      │
      ▼
런타임 검증 (Node ≥ 20, npm ≥ 10)
      │
      ▼
Docker 설치/확인 (macOS: Colima + Docker CLI, Linux: docker.io)
      │
      ▼
OpenShell CLI 설치 (GitHub Release에서 바이너리 다운로드)
      │
      ▼
npm install -g nemoclaw
      │
      ▼
PATH 확인 (asdf reshim 등)
      │
      ▼
"nemoclaw onboard" 안내
```

### 12.2 Dockerfile 빌드 파이프라인

```
node:22-slim
  ↓
시스템 패키지 (python3, curl, git, iproute2)
  ↓
sandbox 사용자/그룹 생성
  ↓
npm install -g openclaw@2026.3.11
  ↓
pip install pyyaml
  ↓
nemoclaw dist/ + blueprint/ 복사
  ↓
npm install --omit=dev (런타임 의존성만)
  ↓
블루프린트 로컬 캐시 배치
  ↓
openclaw.json 기본 설정 생성 (Python 인라인)
  ↓
openclaw plugins install /opt/nemoclaw
  ↓
USER sandbox, WORKDIR /sandbox
```

### 12.3 테스트 스위트

- `test/cli.test.js` — CLI 명령 디스패치 테스트
- `test/registry.test.js` — 샌드박스 레지스트리 CRUD 테스트
- `test/policies.test.js` — 정책 프리셋 로드/병합 테스트
- `test/nim.test.js` — GPU 감지/NIM 컨테이너 관리 테스트
- `test/install-preflight.test.js` — 설치 사전 검사 테스트
- 실행: `npm test` (Node.js 내장 테스트 러너)

---

## 부록: 파일별 줄 수 및 복잡도

| 파일 | 줄 수 | 복잡도 | 핵심 역할 |
|------|-------|--------|---------|
| commands/migration-state.ts | 680 | 높음 | 마이그레이션 상태 감지/스냅샷/복원 |
| bin/lib/onboard.js | 481 | 높음 | 7단계 온보딩 위저드 |
| commands/onboard.ts | 430 | 중간 | 추론 엔드포인트 설정 |
| bin/nemoclaw.js | 367 | 중간 | 호스트 CLI 디스패처 |
| orchestrator/runner.py | 347 | 중간 | 블루프린트 plan/apply/rollback |
| commands/migrate.ts | 297 | 높음 | 마이그레이션 파이프라인 |
| bin/lib/nim.js | 207 | 중간 | NIM 컨테이너 관리 |
| bin/lib/policies.js | 180 | 중간 | 정책 프리셋 병합/적용 |
| index.ts | 260 | 낮음 | 플러그인 등록 + SDK 타입 |
| commands/slash.ts | 148 | 낮음 | 채팅 슬래시 명령 |
| commands/launch.ts | 143 | 중간 | 신규 설치 |
| commands/status.ts | 143 | 낮음 | 상태 조회 |
| cli.ts | 137 | 낮음 | Commander.js 등록 |
| bin/lib/credentials.js | 134 | 낮음 | 자격증명 관리 |
| bin/lib/registry.js | 104 | 낮음 | 샌드박스 레지스트리 |
| blueprint/exec.ts | 99 | 중간 | Python 러너 실행 |
| blueprint/verify.ts | 96 | 중간 | SHA-256 검증 |
| commands/eject.ts | 95 | 중간 | 원복 |
| blueprint/resolve.ts | 83 | 중간 | 블루프린트 버전 해석 |
| commands/logs.ts | 73 | 낮음 | 로그 스트리밍 |
| onboard/prompt.ts | 73 | 낮음 | readline 프롬프트 |
| blueprint/state.ts | 70 | 낮음 | 상태 영속화 |
| blueprint/fetch.ts | 62 | 낮음 | 레지스트리 다운로드 (스텁) |
| onboard/validate.ts | 59 | 낮음 | API 키 검증 |
| onboard/config.ts | 55 | 낮음 | 온보드 설정 영속화 |
| bin/lib/runner.js | 55 | 낮음 | 명령 실행 유틸리티 |
| commands/connect.ts | 40 | 낮음 | 샌드박스 셸 접속 |

---

> 이 분석 리포트는 NemoClaw v0.1.0의 모든 소스 파일을 대상으로 작성되었습니다.
