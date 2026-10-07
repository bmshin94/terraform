# Terraform 저장소 전수조사 및 활용 전략 정리

> 이 문서는 `bmshin94/terraform` 저장소를 전수조사하고,
> 설치·활용법, AI 에이전트 연동, 수익화 전략까지 논의한 내용을 정리한 기록입니다.

---

## 📍 저장소 정보

| 항목 | 내용 |
|---|---|
| **GitHub 주소** | https://github.com/bmshin94/terraform |
| **원본(upstream)** | https://github.com/hashicorp/terraform |
| **공식 문서** | https://developer.hashicorp.com/terraform |
| **버전** | `1.18.0-dev` (미출시 개발 버전) |
| **언어 / 런타임** | Go 1.26.8 |
| **전체 파일 수** | 5,535개 (Go 소스 2,014개) |
| **저장소 용량** | 61MB |
| **라이선스** | BUSL-1.1 (Business Source License), 저작권자 **IBM** |
| **분석 브랜치** | `claude/brave-mccarthy-dr9p4i` |
| **분석 일자** | 2026-10-07 |

---

## 1. 이 저장소는 무엇인가

### 결론

**HashiCorp Terraform의 본체 소스코드(Terraform Core)** 를 가져온 사본(fork)입니다.

| | 설명 |
|---|---|
| ❌ 아닌 것 | "Terraform으로 AWS 서버를 만드는 예제 모음" |
| ✅ 맞는 것 | **Terraform 프로그램 그 자체를 만드는 Go 소스코드** |

자동차를 몰기 위한 *운전 교본*이 아니라, 자동차를 **만드는 설계도와 부품 전체**입니다.

> 📌 마지막 커밋이 PR `#39237`, 버전이 `1.18.0-dev` 이므로 **거의 최신 시점의 개발 코드**입니다.
> 라이선스 저작권자가 **IBM**인 점은 HashiCorp 인수 결과가 코드에 반영된 것입니다.

### Terraform이 하는 일 (한 문장)

> 텍스트 파일에 "원하는 상태"를 적어두면, Terraform이 클라우드의 현재 상태와 비교해서
> **차이만큼만** 자동으로 만들고 / 바꾸고 / 지워줍니다.

이것을 **IaC (Infrastructure as Code, 코드형 인프라)** 라고 합니다.

### 손으로 할 때 vs Terraform

**AWS 콘솔에서 수동 작업**
```
VPC → 서브넷 4개 → 라우팅 테이블 → IGW → 보안그룹 → EC2 3대 → RDS → ALB
= 클릭 200번, 3시간
개발/스테이징/운영 3개 환경 → 600번 클릭, 그리고 분명히 어딘가 다르게 설정됨
```

**Terraform**
```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0abcdef"
  instance_type = "t3.medium"
  tags = { Name = "web-server" }
}
```
```bash
terraform init && terraform plan && terraform apply
```
→ 30초. 3개 환경이 **글자 하나까지 동일**하게 복제됩니다.

### 핵심 성질: 멱등성(idempotency)

| 방식 | 3번 실행하면 |
|---|---|
| 일반 스크립트(명령형) "서버를 만들어라" | 서버 3대 |
| Terraform(선언형) "서버가 1대 있어야 한다" | 서버 1대 |

300번 실행해도 결과가 같습니다. 안심하고 반복 실행할 수 있는 이유입니다.

---

## 2. 폴더 구조 전수조사

### 최상위 루트

| 파일 | 역할 |
|---|---|
| `main.go` (14KB) | 프로그램 진입점 |
| `commands.go` (12KB) | 명령어 라우팅 테이블 (`plan` → 어느 코드로) |
| `provider_source.go` | 프로바이더 다운로드 경로 결정 |
| `checkpoint.go` | 새 버전 출시 확인 |
| `telemetry.go` | 사용 통계 (OpenTelemetry) |
| `version/VERSION` | `1.18.0-dev` |
| `BUGPROCESS.md` (15KB) | 버그 트리아지 프로세스 — 대규모 OSS 운영 노하우 |
| `BUILDING.md` | 소스 빌드 공식 절차 |

### `internal/` — 심장부 (65개 패키지)

```
438개 파일  internal/command/    CLI 명령어 구현체 (가장 큼)
249개 파일  internal/terraform/  그래프 엔진 (진짜 두뇌)
183개 파일  internal/stacks/     Stacks (차세대 신기능)
139개 파일  internal/backend/    상태 저장소
109개 파일  internal/configs/    .tf 파일 파서
 72개 파일  internal/lang/       HCL 언어 + 내장 함수 99개
 67개 파일  internal/addrs/      리소스 주소 체계
 62개 파일  internal/states/     상태(state) 관리
 61개 파일  internal/cloud/      HCP Terraform 연동
 45개 파일  internal/rpcapi/     외부 프로그램용 gRPC API ⭐
 43개 파일  internal/plans/      실행 계획 자료구조
 38개 파일  internal/moduletest/ terraform test 프레임워크
 19개 파일  internal/dag/        방향성 비순환 그래프 알고리즘
```

### 조직도로 보는 폴더 구조

```
📋 접수처 (main.go, commands.go)
     "어떤 일로 오셨어요?" → 담당 부서 안내

📖 번역팀 (internal/configs/)
     사람이 쓴 .tf 파일을 컴퓨터 말로 번역

🧮 계산팀 (internal/lang/)
     "${var.name}-server" 수식 계산. 내장 함수 99종 보유
     cidr.go / crypto.go / encoding.go / filesystem.go
     collection.go / sensitive.go / redact.go (비밀값 마스킹)

📊 일정관리팀 (internal/dag/)  ⭐ 숨은 공신
     "DB 먼저, 그다음 웹서버"
     "A와 B는 상관없네 → 동시에 진행!"
     위상 정렬(topological sort)로 순서 결정 + 병렬 실행
     ※ 100대 순차 생성 = 100분 / 병렬 = 1분

🧠 기획실 (internal/terraform/)  ⭐ 진짜 두뇌
     설계도 vs 현실 비교 → 차이(diff) 계산 → 할 일 목록

📒 기록보관소 (internal/states/)
     "지금까지 뭘 만들었더라" 장부 (.tfstate)
     ※ 이게 없으면 apply 두 번에 서버가 6대가 됩니다

🏦 금고 (internal/backend/)
     장부 보관 위치 + 자물쇠(동시 작업 방지)

🌍 해외영업팀 (internal/plugin/, providers/)  ⭐ 핵심 설계
     본사는 AWS를 "전혀" 모름
     외주업체(프로바이더)에게 gRPC로 지시만 함

🔌 대외협력실 (internal/rpcapi/)  ⭐ 돈 되는 부서
     다른 프로그램이 Terraform을 호출하는 창구

🧪 품질관리팀 (internal/moduletest/)
     terraform test — 인프라 코드도 테스트

🚀 신사업본부 (internal/stacks/)
     차세대 제품 (파일 183개 = 대규모 투자 중)
```

### 지원 백엔드 (상태 저장소) — 전수 확인

```
s3 (AWS)  ·  azure / azurerm  ·  gcs (Google)  ·  oci (Oracle)
oss (알리바바)  ·  cos (텐센트)  ·  kubernetes  ·  pg (PostgreSQL)
consul  ·  http  ·  local  ·  inmem  ·  cloud / remote (HCP Terraform)
레거시: artifactory, etcd, etcdv3, swift, manta
```

### 그 외 폴더

| 폴더 | 내용 |
|---|---|
| `docs/` | **내부 개발자용 아키텍처 문서** ⭐ `architecture.md`, `resource-instance-change-lifecycle.md`, `planning-behaviors.md`, `destroying.md` + 다이어그램 PNG 9장 |
| `internal/e2e/`, `testing/` | 종단간 테스트 — `.tf` 1,784개 / `.tfstate` 211개 (공식 예제 보물창고) |
| `.changes/` | changie 기반 버전별 변경 이력 (v1.11 ~ v1.18) |
| `website/` | 공식 문서 소스 |
| `tools/` | protobuf 컴파일러 등 개발 도구 |
| `.github/` | CI 워크플로, CONTRIBUTING 가이드 |

---

## 3. 가장 중요한 설계: 플러그인 아키텍처

### "Terraform 본체는 클라우드를 하나도 모른다"

```
            Terraform 본체
       (순서 계산만 하는 지휘자)
                  │ gRPC
      ┌───────┬───┴───┬────────────┐
    AWS팀   Azure팀  GCP팀   Cloudflare팀
  (플러그인)(플러그인)(플러그인)  (플러그인)
      └───────┴───────┴────────────┘
          실제 API 호출은 이 친구들이 함
```

확인한 증거: `internal/plugin/`, `internal/plugin6/`, `docs/plugin-protocol/tfplugin5.proto`, `tfplugin6.proto`

**왜 중요한가**

1. 새 클라우드가 생겨도 Terraform 본체는 **코드 한 줄 안 고쳐도** 됩니다
2. 현재 **4,000개 이상**의 프로바이더가 존재합니다
3. **누구나 자기 서비스용 프로바이더를 만들 수 있습니다** ← 수익화 포인트
4. 프로바이더는 **별도 프로세스**로 실행됩니다 (안정성·격리)

프로토콜이 `tfplugin5` / `tfplugin6` 두 개인 것은 하위 호환성 때문입니다.

---

## 4. 실행 흐름 (docs/architecture.md 기반)

```
사용자: terraform apply
   │
   ├─ main.go            프로그램 시작
   ├─ commands.go        "apply" → ApplyCommand 매핑
   │
   ├─ internal/command/  인자·플래그·환경변수 파싱
   │                     → backendrun.Operation 객체 생성
   │                       (작업종류 / 워크스페이스 / 변수 / 설정경로 / -target)
   │
   ├─ internal/backend/  선택된 백엔드로 Operation 전달
   │                     ※ 대부분 백엔드는 "저장"만 담당 →
   │                       local.Local 로 감싸서 내 PC에서 실행
   │                       (remote / cloud 백엔드만 원격 실행)
   │
   ├─ statemgr           상태 파일 읽기 + 잠금(lock)
   ├─ internal/configs/  .tf 파일 로드 & 파싱
   │
   ├─ terraform.Context  ← 본격 시작
   │   ├─ internal/dag/       의존성 그래프 구축 + 위상 정렬
   │   ├─ internal/lang/      표현식·함수 평가
   │   ├─ internal/plugin/    프로바이더 프로세스 기동 (gRPC)
   │   │   └─ 실제 AWS/Azure API 호출
   │   └─ internal/plans/     diff 계산 → 실행 계획
   │
   ├─ 병렬 적용 (의존성 없는 리소스 동시 처리, 기본 10개)
   ├─ internal/states/   상태 파일 갱신 + 저장
   └─ command/views/     결과 출력 (사람용 색상 / JSON)
```

### 설계상 흥미로운 사실

> `docs/architecture.md` 확인 결과 — 대부분의 백엔드는 "작업 실행" 기능이 **없습니다.**
> `local` 과 HCP Terraform의 `remote`/`cloud` 만 실행 기능을 가지며,
> 나머지(S3, GCS 등)는 **상태 저장만** 합니다.
> 그래서 S3 백엔드를 써도 실제 실행은 내 PC에서 일어납니다.

---

## 5. 지원 명령어 전체 (`commands.go` 추출)

```
init        프로바이더 다운로드, 백엔드 설정  ← 항상 첫 단계
plan        변경 미리보기 (아무것도 안 건드림)
apply       실제 적용
destroy     전부 삭제
validate    문법 검증
fmt         코드 정렬
show        상태/계획 출력
output      출력값 조회
graph       의존성 그래프를 DOT 형식으로
console     대화형 표현식 실험 (REPL) ⭐ 학습에 최고
import      기존 수동 리소스를 Terraform 관리로 편입
taint / untaint   재생성 표시
refresh     실제 인프라 상태를 상태파일에 동기화
state       list / show / mv / rm / pull / push
            replace-provider / force-unlock / identities
workspace   list / new / select / delete / show (환경 분리)
env         workspace의 구 명칭 (하위호환)
providers   lock / mirror / schema   ← schema 가 AI용으로 중요
test        인프라 코드 테스트 ⭐
query       신규 — 기존 인프라 탐색 (list 블록)
stacks      신규 — 스택 관리
metadata    functions — 내장 함수 메타데이터 JSON ⭐
login / logout    레지스트리 인증
rpcapi      gRPC API 서버 모드 ⭐
get / modules     모듈 관련
```

---

## 6. 개발 중인 신기능 (CHANGELOG 1.18.0-dev)

알파 빌드에서만 활성화되는 실험 기능입니다.

1. **Deferred Actions** (`-allow-deferral`)
   `count` / `for_each` 값이 아직 미확정일 때도 계획 수립 가능 — 다단계 배포의 오랜 숙제 해결
2. **`terraform test cleanup`**
   실패한 테스트가 남긴 상태 파일 자동 정리
3. **`test` 의 `backend` 블록 / `skip_cleanup`**
   테스트 인프라를 테스트 간에 살려두어 시간 절약
4. **`terraform query -policies`**
   발견된 리소스에 정책 검사 적용

---

## 7. 설치 및 사용법

### 7-1. 그냥 Terraform을 쓰고 싶다 (99%의 경우)

> **이 저장소를 빌드할 필요 없습니다.** 공식 바이너리를 받으세요.

```bash
# macOS
brew tap hashicorp/tap && brew install hashicorp/tap/terraform

# Windows
winget install HashiCorp.Terraform

# Ubuntu / Debian
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

terraform version
```

> 💡 **실무 추천: `tfenv`** — Node의 nvm 같은 버전 관리자. 프로젝트마다 버전이 달라 사실상 필수.
> ```bash
> brew install tfenv && tfenv install 1.17.0 && tfenv use 1.17.0
> ```

### 7-2. 첫 실습 — 클라우드 계정 없이 5분

`main.tf`
```hcl
terraform {
  required_providers {
    local = { source = "hashicorp/local", version = "~> 2.4" }
  }
}

variable "greeting" {
  type    = string
  default = "안녕하세요"
}

resource "local_file" "hello" {
  filename = "${path.module}/output.txt"
  content  = "${var.greeting}, Terraform!"
}

output "file_path" {
  value = local_file.hello.filename
}
```

```bash
terraform init      # 프로바이더 다운로드
terraform plan      # 미리보기
terraform apply     # 적용 → yes
cat output.txt      # 결과 확인
terraform destroy   # 정리
```

### 7-3. 필수 명령어 치트시트

```bash
terraform init                 # 항상 첫 단계
terraform init -upgrade        # 프로바이더 버전 올리기
terraform fmt -recursive       # 코드 정렬
terraform validate             # 문법 검사
terraform plan -out=tfplan     # 계획을 파일로 저장
terraform apply tfplan         # 저장된 계획 그대로 적용 (안전)
terraform apply -auto-approve  # 확인 없이 (CI 전용)
terraform destroy              # 전부 삭제
terraform state list           # 관리 중인 리소스 목록
terraform console              # 대화형 실험 ⭐
terraform output               # 출력값
terraform graph | dot -Tpng > g.png   # 의존성 그래프 시각화
```

**⭐ `terraform console` 적극 추천**
```
> upper("hello")
"HELLO"
> cidrsubnet("10.0.0.0/16", 8, 2)
"10.0.2.0/24"
> [for n in ["a","b"] : "${n}-server"]
["a-server", "b-server"]
```

### 7-4. 학습 로드맵

```
1일차  resource / variable / output / terraform console
2일차  provider 설정, AWS 실습 (S3, EC2)
3일차  상태(state) 개념, 원격 백엔드(S3)
4일차  module 만들기 & 재사용
5일차  count / for_each / dynamic
6일차  workspace, 환경 분리 전략
7일차  terraform test, CI/CD 연동
```

### 7-5. 이 저장소를 직접 빌드 (소스 개발용)

```bash
# Go 1.26.8 필요 (.go-version 명시)
go build -o bin/terraform .
./bin/terraform version          # Terraform v1.18.0-dev

# 릴리스처럼 빌드 (-dev 꼬리표 제거)
go build -ldflags "-w -s -X 'github.com/hashicorp/terraform/version.dev=no'" -o bin/terraform .

# ⭐ 실험 기능 켜서 빌드 (Stacks, deferred actions 체험)
go build -ldflags "-X 'main.experimentsAllowed=yes'" -o bin/terraform-exp .

# 검사
go test ./internal/dag/...
make fmtcheck && make vetcheck
```

> ⚠️ 전체 테스트(`go test ./...`)는 수십 분 + 대량 메모리를 씁니다. 부분 테스트 권장.

### 7-6. 소스 탐색 요령 (실제로 제일 유용)

```bash
# 에러 메시지로 원인 찾기
grep -rn "Inconsistent dependency lock file" internal/

# 함수 구현 확인
cat internal/lang/funcs/cidr.go

# 공식 .tf 예제 보물창고
ls internal/terraform/testdata/
ls internal/command/testdata/

# 내부 아키텍처 문서 ⭐
cat docs/architecture.md
cat docs/resource-instance-change-lifecycle.md
cat docs/planning-behaviors.md
cat docs/destroying.md
```

---

## 8. 플러그인? 스킬? MCP?

### 정답: **셋 다 아닙니다.**

**독립 실행형 CLI 프로그램(standalone binary)의 소스코드**입니다.
`git`, `docker`, `kubectl` 과 같은 부류이며 어떤 호스트 프로그램도 필요 없습니다.

| 구분 | 정의 | Terraform은? |
|---|---|---|
| **플러그인** | 호스트 안에서 동작하는 확장 | ❌ **Terraform이 오히려 호스트**이고 프로바이더가 플러그인 |
| **스킬** | AI에게 주는 지침 문서 (SKILL.md) | ❌ 아님 |
| **MCP** | AI가 외부 도구를 쓰는 표준 프로토콜 | ❌ 아님. 단, **감쌀 수는 있음** |

### 하지만 관계는 있습니다

**① Terraform은 "플러그인 호스트" (역방향)**
```
      Terraform (호스트)
            │ gRPC
   ┌────────┼────────┐
  AWS     Azure     내가 만든
 프로바이더 프로바이더  프로바이더  ← 이게 플러그인
```

**② MCP로 감쌀 수 있습니다** ⭐
```
Claude ─MCP─→ [MCP 서버] ─실행─→ terraform 바이너리
                  ├ tool: terraform_plan
                  ├ tool: terraform_validate
                  ├ tool: get_provider_schema
                  └ tool: explain_plan
```
실제로 HashiCorp가 공식 Terraform MCP 서버(`hashicorp/terraform-mcp-server`)를 운영 중입니다.
다만 **"내 환경에 맞춘 MCP 서버"를 만드는 것은 여전히 열린 기회**입니다.

**③ 스킬로 만들 수도 있습니다**
`SKILL.md` 에 사내 Terraform 작성 규칙을 넣어두면 AI가 표준에 맞는 코드를 생성합니다.

### 한 줄 정리

> Terraform은 "플러그인/스킬/MCP"가 **아니라 그것들의 대상**입니다.
> 당신이 Terraform을 위한 플러그인·스킬·MCP를 만드는 쪽입니다.

---

## 9. API 토큰이 필요한가

### 정답: Terraform 자체는 불필요. 단, 거의 항상 **클라우드 자격증명**이 필요합니다.

| 용도 | 토큰 필요? |
|---|---|
| Terraform 설치·실행 | ❌ 불필요 (완전 무료, 가입 불필요) |
| 공개 프로바이더 다운로드 | ❌ 불필요 |
| `local`/`random`/`null` 프로바이더 실습 | ❌ 불필요 (클라우드 없이 연습 가능) |
| **AWS/Azure/GCP 리소스 생성** | ✅ 필요 (토큰이 아니라 자격증명) |
| HCP Terraform (SaaS) | ✅ `terraform login` |
| 프라이빗 모듈 레지스트리 | ✅ 조직 토큰 |
| GitHub/Datadog/Cloudflare 프로바이더 | ✅ 각 서비스 API 토큰 |

### 클라우드별 설정 (안전한 순서대로)

```bash
# AWS — ⭐ 1순위: SSO / IAM Identity Center
aws sso login --profile myprofile
export AWS_PROFILE=myprofile

# 2순위: 명명된 프로파일
aws configure --profile dev && export AWS_PROFILE=dev

# 3순위: 환경변수 (임시 자격증명)
export AWS_ACCESS_KEY_ID="ASIA..." AWS_SECRET_ACCESS_KEY="..." \
       AWS_SESSION_TOKEN="..." AWS_REGION="ap-northeast-2"

# ⭐⭐ CI/CD: OIDC — 키를 아예 저장하지 않음 (최선)

# Azure
az login
export ARM_CLIENT_ID=... ARM_CLIENT_SECRET=... ARM_TENANT_ID=... ARM_SUBSCRIPTION_ID=...

# GCP
gcloud auth application-default login
export GOOGLE_APPLICATION_CREDENTIALS="/path/key.json"

# HCP Terraform
terraform login
```

### 🚨 절대 금지

```hcl
# ❌❌❌ .tf 파일에 키 하드코딩
provider "aws" {
  access_key = "AKIAIOSFODNN7EXAMPLE"   # 💀 Git에 올라가면 끝
  secret_key = "wJalrXUtnFEMI/..."       # 💀 봇이 수 분 내 스캔 → 탈취
}
```
> GitHub에 AWS 키를 올리면 **수 분 내 봇이 발견**해 채굴에 사용합니다. 수천만 원 청구 사례가 흔합니다.

```hcl
# ✅ 올바른 방법 — 아무것도 적지 않습니다
provider "aws" {
  region = "ap-northeast-2"
  # 자격증명은 환경변수/프로파일/OIDC 에서 자동 탐색
}
```

### ⚠️ 가장 중요한 경고: `.tfstate` 파일

> **상태 파일에는 DB 비밀번호 같은 민감값이 평문으로 저장됩니다.**

```json
{
  "resources": [{
    "type": "aws_db_instance",
    "instances": [{
      "attributes": { "password": "SuperSecret123!" }   // 😱 평문
    }]
  }]
}
```

`sensitive = true` 로 표시해도 **화면 출력만 가려질 뿐, 상태 파일에는 평문**입니다.

**필수 방어**

`.gitignore`
```gitignore
*.tfstate
*.tfstate.*
.terraform/
*.tfvars
!example.tfvars
# .terraform.lock.hcl 은 오히려 커밋해야 합니다 (무시하지 마세요)
```

원격 백엔드 + 암호화 (팀 작업 시 필수)
```hcl
terraform {
  backend "s3" {
    bucket         = "my-tfstate-bucket"
    key            = "prod/terraform.tfstate"
    region         = "ap-northeast-2"
    encrypt        = true                 # ⭐ 저장 시 암호화
    kms_key_id     = "arn:aws:kms:..."    # ⭐ 고객 관리 키
    dynamodb_table = "tf-lock"            # ⭐ 동시 작업 방지
  }
}
```
+ S3 버킷은 **퍼블릭 차단 / 버저닝 활성화 / IAM 최소 권한** 필수.

---

## 10. AI 에이전트 구축에 도움이 되는가

### 정답: **매우 그렇습니다. 현재 가장 유망한 영역 중 하나입니다.**

### 이유 1 — Terraform은 이미 "AI 친화적 설계"를 갖췄다

AI 에이전트가 도구를 잘 쓰려면 ① 구조화된 입출력 ② 안전한 미리보기 ③ 기계판독 스키마 가 필요합니다.
**Terraform은 셋 다 가지고 있습니다.**

```bash
terraform plan -json               # 계획을 JSON 스트림으로
terraform show -json tfplan        # 계획 전체 구조 JSON
terraform providers schema -json   # ⭐⭐ 프로바이더 전체 스키마
terraform metadata functions -json # 내장 함수 99개 메타데이터
terraform validate -json           # 검증 결과 JSON
terraform output -json             # 출력값 JSON
```

**`plan` 이라는 안전장치가 구조적으로 존재**
```
AI → terraform plan -json → 결과 분석 → 사람에게 보고
                                          ↓
                                   사람이 승인 → apply
```
이 구조 덕분에 **AI가 인프라를 다뤄도 사고가 나지 않습니다.**

**`providers schema -json` 이 환각을 죽입니다** ⭐⭐⭐
```hcl
# AI가 자주 저지르는 환각
resource "aws_instance" "web" {
  instance_size = "t3.medium"   # ❌ 없는 속성 (instance_type 이 맞음)
  enable_backup = true          # ❌ 완전히 지어낸 속성
}
```
스키마 JSON에는 **모든 리소스의 모든 속성·타입·필수여부·설명**이 들어 있습니다.
이걸 AI 컨텍스트나 검증 도구로 쓰면 환각이 사실상 사라집니다.

### 이유 2 — `rpcapi` : AI 에이전트를 위한 정식 통로

`internal/rpcapi/` (45개 파일)에서 확인한 실제 gRPC 서비스:

```protobuf
service Stacks {
  rpc ValidateStackConfiguration(...)   // 검증
  rpc PlanStackChanges(...)             // 계획
  rpc ApplyStackChanges(...)            // 적용
  rpc InspectExpressionResult(...)      // ⭐ 표현식 값 들여다보기
  rpc ListResourceIdentities(...)
  rpc MigrateTerraformState(...)
}
service Dependencies {
  rpc GetProviderSchema(...)            // ⭐⭐ 스키마 조회
  rpc BuildProviderPluginCache(...)
}
service Packages { ... }
service Setup { rpc Handshake(...) }
```

터미널 출력을 정규식으로 긁는 방식 대신 **구조화된 gRPC 호출**이 가능합니다.

### 만들 수 있는 AI 에이전트

| # | 에이전트 | 설명 |
|---|---|---|
| ① | **인프라 코드 생성** | 스키마 확인 → .tf 생성 → `validate` 로 **자가 수정 루프** → plan 보고 |
| ② | **Plan 리뷰** ⭐ | PR의 plan을 분석해 "DB 삭제됨 / SSH 전체개방 / 월 +$500" 자동 코멘트 |
| ③ | **비용 예측** | `plan -json` + 요금 API → 월 비용 증감 계산 |
| ④ | **드리프트 감지** | 매일 plan → "누가 콘솔에서 손으로 바꿨습니다" 알림 |
| ⑤ | **장애 대응** | 알람 → state 조회 → 수정 .tf 생성 → plan 제시 → 승인 후 apply |
| ⑥ | **레거시 역수입** | 수백 개 수동 리소스를 `import`/`query` 로 자동 코드화 |

### 추천 아키텍처

```
┌──────────────────────────────────────────┐
│  Claude / GPT (판단)                      │
└────────────┬─────────────────────────────┘
             │ MCP 프로토콜
┌────────────▼─────────────────────────────┐
│  MCP 서버 (내가 만드는 부분)               │
│   ├ tf_validate      (안전: 읽기전용)      │
│   ├ tf_plan          (안전: 미리보기)      │
│   ├ tf_show_json     (안전)               │
│   ├ tf_schema        (안전) ⭐            │
│   ├ tf_state_list    (안전)               │
│   └ tf_apply         (⚠️ 사람 승인 필수)   │
└────────────┬─────────────────────────────┘
             │ exec 또는 gRPC(rpcapi)
┌────────────▼─────────────────────────────┐
│  terraform 바이너리                        │
└──────────────────────────────────────────┘
```

### ⚠️ 안전 원칙 (반드시 준수)

| 원칙 | 이유 |
|---|---|
| **`apply` 는 AI 단독 실행 금지** | 사람 승인 게이트 필수 |
| **`destroy` 는 도구에서 제외** | 복구 불가능 |
| **읽기 전용부터 시작** | plan/validate/show 만으로도 가치가 큼 |
| **`-out=tfplan` 사용** | AI가 본 계획 = 실제 적용 계획 보장 |
| **상태 파일을 AI 컨텍스트에 넣지 말 것** | 비밀번호 평문 포함 |
| **최소 권한 IAM 역할** | AI 전용 역할 분리 |
| **샌드박스 계정에서 먼저** | 운영 계정 직결 금지 |

---

## 11. React / PHP 로 만들 수 있는가

### 11-1. "Terraform을 React/PHP로 다시 만들기" → ❌ 불가능하고 무의미

| 항목 | 이유 |
|---|---|
| 규모 | Go 소스 2,014개 파일, 10년간 수천 명이 개발 |
| 단일 바이너리 | Go는 의존성 없는 실행파일 1개. PHP는 런타임, React는 브라우저/Node 필요 |
| gRPC 플러그인 | 프로바이더 4,000개가 Go 기반. 재구현하면 생태계 0에서 시작 |
| 동시성 | 그래프 병렬 실행이 goroutine에 깊이 의존. PHP는 구조적으로 부적합 |
| **치명타** | 만들어도 **AWS 프로바이더를 못 씁니다** → 아무 쓸모가 없습니다 |

### 11-2. "React/PHP로 Terraform 활용 도구 만들기" → ✅ 완벽하게 가능. 이게 정답

> Terraform을 **다시 만들지 말고, 엔진으로 가져다 쓰세요.**
> Terraform = 엔진 / 당신이 만드는 것 = 자동차 외관과 대시보드

```
┌────────────────────────────────┐
│  React 프론트엔드               │
│  - 인프라 시각화 (React Flow)    │
│  - Plan 결과 다이어그램          │
│  - 승인 버튼, 비용 대시보드      │
└──────────┬─────────────────────┘
           │ REST / WebSocket
┌──────────▼─────────────────────┐
│  백엔드 (PHP / Node / Go / Python)│
│  - terraform 명령 실행 & 큐잉    │
│  - JSON 출력 파싱               │
│  - 사용자/권한/이력 관리         │
└──────────┬─────────────────────┘
           │ exec() 또는 gRPC(rpcapi)
┌──────────▼─────────────────────┐
│  terraform 바이너리 (그대로 사용) │
└────────────────────────────────┘
```

**PHP 백엔드 예시 (Laravel)**
```php
<?php
class TerraformService
{
    private string $workDir;

    public function plan(): array
    {
        $this->run(['terraform', 'plan', '-out=tfplan', '-input=false']);
        $json = $this->run(['terraform', 'show', '-json', 'tfplan']);
        $plan = json_decode($json, true);

        $summary   = ['create' => 0, 'update' => 0, 'destroy' => 0];
        $dangerous = [];

        foreach ($plan['resource_changes'] ?? [] as $rc) {
            $actions = $rc['change']['actions'];
            if (in_array('create', $actions)) $summary['create']++;
            if (in_array('update', $actions)) $summary['update']++;
            if (in_array('delete', $actions)) {
                $summary['destroy']++;
                $dangerous[] = $rc['address'];   // ⚠️ 삭제 대상
            }
        }
        return compact('summary', 'dangerous') + ['raw' => $plan];
    }

    private function run(array $cmd): string
    {
        $proc = new Symfony\Component\Process\Process($cmd, $this->workDir, [
            'AWS_PROFILE'      => 'readonly-role',
            'TF_IN_AUTOMATION' => '1',
        ]);
        $proc->setTimeout(900);
        $proc->mustRun();
        return $proc->getOutput();
    }
}
```

**React 프론트엔드 예시**
```jsx
function PlanReview({ plan }) {
  const { summary, dangerous } = plan;
  return (
    <div className="plan-review">
      <div className="stats">
        <Stat color="green"  label="생성" value={summary.create} />
        <Stat color="yellow" label="변경" value={summary.update} />
        <Stat color="red"    label="삭제" value={summary.destroy} />
      </div>

      {dangerous.length > 0 && (
        <Alert severity="error">
          <strong>⚠️ 삭제되는 리소스 {dangerous.length}개</strong>
          <ul>{dangerous.map(a => <li key={a}><code>{a}</code></li>)}</ul>
          <p>데이터가 영구 손실될 수 있습니다.</p>
        </Alert>
      )}

      <DependencyGraph data={plan.raw.configuration} />
      <button disabled={dangerous.length > 0 && !confirmed}>승인 및 적용</button>
    </div>
  );
}
```

**추천 기술 스택**

| 레이어 | 1순위 | 대안 |
|---|---|---|
| 프론트 | React + TypeScript | Vue, Svelte |
| 그래프 시각화 | **React Flow** ⭐ | D3, Cytoscape |
| 코드 에디터 | **Monaco Editor** (HCL 문법강조) | CodeMirror |
| 백엔드 | Go (같은 언어라 유리) | Node, PHP, Python |
| 실행 격리 | **Docker 컨테이너** ⭐ 필수 | Firecracker |
| 작업 큐 | Redis + 워커 | SQS, RabbitMQ |

### 🚨 치명적 주의: 명령 주입(Command Injection)

```php
// ❌❌❌ 절대 금지
$dir = $_POST['directory'];
shell_exec("cd $dir && terraform apply -auto-approve");
// 공격자 입력: "/tmp; rm -rf / #"  → 서버 전체 파괴
```
```php
// ✅ 안전 — 배열 인자 + 화이트리스트 + 컨테이너 격리
$allowed = ['plan', 'validate', 'show'];     // apply 는 별도 승인 경로
if (!in_array($action, $allowed, true)) abort(403);
$process = new Process(['terraform', $action, '-no-color'], $safeDir);
```

**보안 체크리스트**
- ✅ `shell_exec`/`exec` 문자열 결합 금지 → 배열 인자만
- ✅ 사용자 입력을 **절대** 명령어에 직접 넣지 않기
- ✅ Terraform 실행은 **반드시 격리된 Docker 컨테이너**에서
- ✅ `.tfstate` 를 웹 공개 디렉터리에 두지 않기
- ✅ `apply` 는 별도 승인 워크플로로 분리
- ✅ 타임아웃 설정 / 로그 민감값 마스킹

### 결론

> ❌ Terraform 재구현 → 불가능하고 무의미
> ✅ Terraform 위에 올라타는 제품 → 완전히 가능하고, **이게 실제 시장**입니다
>
> Terraform Cloud, Spacelift, env0, Scalr 같은 회사들이 전부 이 방식입니다.
> Terraform을 다시 만든 게 아니라 **감싼** 것입니다.

---

## 12. 유튜브 강의 영상 제작

### 정답: **가능할 뿐 아니라 리스크 대비 수익이 가장 좋은 선택지입니다.**

| 근거 | 설명 |
|---|---|
| **라이선스 문제 없음** | BUSL은 "경쟁 호스팅 서비스"만 제한. 교육·강의·영상은 **완전 자유** |
| **한국어 콘텐츠 희소** | 영어권엔 넘치지만 한국어는 질·양 모두 부족 |
| **시청자 구매력 높음** | DevOps/인프라 = 고연봉 직군 = 유료 전환율 높음 |
| **검색 유입 안정적** | "terraform state 오류" 등 매일 꾸준히 발생 |
| **차별화 무기 보유** ⭐ | **소스코드를 읽을 수 있다** |

### 차별화 포인트

```
❌ 널린 콘텐츠:  "terraform apply 치세요"  (10분, 조회수 500)
✅ 당신의 콘텐츠: "apply 를 치면 내부에서 무슨 일이 일어나는가
                  — 소스코드로 추적하기"  (25분, 차별화 확실)
```

**차별화 소재 창고 (이 저장소 안에 있음)**
- `docs/architecture.md` — HashiCorp 내부 개발자용 공식 문서
- `docs/resource-instance-change-lifecycle.md` + 다이어그램 PNG 9장
- `docs/planning-behaviors.md`, `docs/destroying.md`
- `internal/dag/` — 그래프 알고리즘 실제 구현
- 테스트 `.tf` 1,784개 — 공식 예제
- `.changes/v1.18` — 미출시 신기능

### 시리즈 구성 (3단 로켓)

**🚀 1단: 입문 — 유입용**

| # | 제목 | 길이 |
|---|---|---|
| 1 | Terraform 10분 만에 이해하기 — 클라우드 없이 실습 | 10분 |
| 2 | 설치 + 첫 실행 (Win/Mac/Linux) | 12분 |
| 3 | `plan`과 `apply`의 차이 — 사고를 막는 안전장치 | 10분 |
| 4 | AWS S3 버킷 만들기 (프리티어) | 15분 |
| 5 | **상태 파일의 모든 것 — 지우면 어떻게 되는지 직접 해봄** ⭐ | 18분 |
| 6 | variable / output / locals | 14분 |
| 7 | count vs for_each — 언제 뭘 쓰나 | 15분 |
| 8 | 모듈 만들고 재사용하기 | 20분 |
| 9 | 팀 작업 — S3 백엔드 + 잠금 | 18분 |
| 10 | **실수로 운영 DB 날리는 상황 재현 + 방어법** ⭐⭐ | 20분 |

**🚀 2단: 심화 — 신뢰 구축**

| # | 제목 |
|---|---|
| 11 | **소스로 보는 Terraform 내부구조 — `apply` 1초의 여정** ⭐ |
| 12 | **의존성 그래프(DAG)는 어떻게 순서를 정하는가** ⭐ |
| 13 | 프로바이더는 왜 별도 프로세스인가 — gRPC 플러그인 |
| 14 | 리소스가 "재생성"되는 진짜 이유 (공식 내부문서 해설) |
| 15 | `terraform test` — 인프라도 테스트한다 |
| 16 | `import` / `moved` — 기존 인프라 길들이기 |
| 17 | GitHub Actions CI/CD 완전 자동화 |
| 18 | **Terraform 보안 사고 TOP 5와 방어** ⭐ |
| 19 | Stacks — 차세대 기능 미리보기 (1.18-dev 빌드) |
| 20 | 비용 폭탄 방지 — plan으로 요금 예측 |

**🚀 3단: AI 융합 — 바이럴** ⭐⭐⭐

| # | 제목 |
|---|---|
| 21 | **AI에게 인프라를 맡겨도 될까 — Claude + Terraform 실험** |
| 22 | **AI 환각 잡는 법 — `providers schema -json`** |
| 23 | **Terraform MCP 서버 직접 만들기** |
| 24 | **Plan 리뷰 AI 봇 — PR에 자동 코멘트** |
| 25 | React로 Terraform 웹 대시보드 만들기 |

> 21~24는 "AI + 인프라" 검색량 폭증 구간인데 **한국어 콘텐츠가 거의 없습니다.** 선점 가치 큼.

### 제작 실무 팁

- 화면: 왼쪽 에디터(폰트 16pt+, 모바일 고려) / 오른쪽 터미널 / 필요 시 AWS 콘솔
- ⚠️ **AWS 계정 ID·액세스 키 절대 노출 금지** → 더미 계정, 편집 시 블러
- ✅ 영상 끝에 **`terraform destroy` 꼭 보여주기** (시청자 요금 사고 방지 = 신뢰)
- ✅ 실습 코드 GitHub 공개 → 설명란 링크
- ✅ 챕터 타임스탬프 필수 (검색 유입)
- ✅ **에러를 일부러 내고 고치는 장면** — 반응 가장 좋음

**제목 전략**
```
❌ "Terraform 강의 5강 - 변수"
✅ "운영 DB를 날려봤습니다 (Terraform 사고 재현)"
✅ "AWS 요금 500만원 나온 이유 — Terraform으로 막는 법"
✅ "Terraform 소스코드 까보기: apply 를 누르면 생기는 일"
```

### 현실적 조언

| 항목 | 현실 |
|---|---|
| 초기 조회수 | 1~20화까지 낮습니다. 정상입니다 |
| 손익분기 | 보통 30~50편, 6개월~1년 |
| 포기 지점 | 대부분 10편에서 그만둠 → **완주가 곧 차별화** |
| 상표권 | "Terraform"은 HashiCorp 상표 → **채널명에 넣지 말 것** |
| 사칭 금지 | "공식", "Official" 표현 금지 |

---

## 13. 수익화 전략 상세

### 🚨 13-0. BUSL 라이선스 — 사업 전 필독

```
라이선스: Business Source License 1.1
저작권자: IBM (HashiCorp 인수 결과)
적용 범위: Terraform 1.6.0 이상
```

**허용 조항 요약 (LICENSE 원문)**
> "You may make production use of the Licensed Work, provided Your use does not include
> **offering the Licensed Work to third parties on a hosted or embedded basis
> in order to compete with IBM Corp.'s paid versions**."

**✅ 해도 되는 것**

| 항목 | 가능 |
|---|---|
| 사내 업무용 사용 | ✅ 완전 자유 |
| 고객사 Terraform 컨설팅·구축 | ✅ |
| 유료 강의, 책, 영상 | ✅ 완전 자유 |
| 모듈/템플릿 판매 | ✅ |
| 프로바이더 개발·판매 | ✅ (SDK는 MPL 2.0) |
| Terraform을 "사용하는" SaaS (분석·리뷰·알림) | ✅ |
| 소스 읽기·수정·포크·기여 | ✅ |

**❌ 하면 안 되는 것**

| 항목 | 위험 |
|---|---|
| "웹에서 terraform apply 돌려주는" 유료 호스팅 | ❌ HCP Terraform 경쟁 = 명백한 위반 |
| Terraform 실행 기능을 제품에 내장해 유료 판매 | ❌ "embedded basis" 위반 |
| 리브랜딩 판매 | ❌ 위반 + 상표권 |

**🧭 판별 기준**
> 내 제품이 HCP Terraform(app.terraform.io)의 **대체재인가, 보완재인가?**
> 대체재 → ❌ 위험 / 보완재 → ✅ 안전

**🟢 100% 안전한 우회로: OpenTofu**
Terraform의 오픈소스 포크, **MPL 2.0 (완전 자유)**. 호스팅 서비스를 만들 거라면 OpenTofu 기반으로 하면 라이선스 문제가 사라집니다. 문법도 거의 호환.

> ⚠️ 위 내용은 법률 자문이 아닙니다. 사업화 전 변호사 검토를 받으세요.

---

### 🥇 티어 1 — 자본 0원, 즉시 실행 (리스크 최저)

#### 아이디어 1: 한국어 Terraform 교육 사업 ⭐⭐⭐ 최우선

```
유튜브 무료 (신뢰 구축)
    ↓
유료 강의  ₩50,000 ~ ₩150,000 × 수강생
    ↓
전자책 / PDF  ₩15,000 ~ ₩30,000
    ↓
기업 사내교육 ⭐ 여기가 진짜 돈   1일 ₩200만 ~ ₩500만
    ↓
컨설팅 의뢰 유입   프로젝트당 ₩1,000만 ~
```

| 상품 | 가격 | 차별점 |
|---|---|---|
| 「Terraform 입문 7일」 | ₩49,000 | 클라우드 계정 없이 시작 |
| 「Terraform 내부구조 해부」 | ₩149,000 | **소스 기반 — 국내 유일** |
| 「AI × Terraform 실전」 | ₩199,000 | MCP 서버 제작 포함 |

**예상 수익:** 1년차 월 50~300만원, 기업교육 뚫리면 월 1,000만원 이상 가능

#### 아이디어 2: 모듈 / 템플릿 판매

```
📦 한국형 스타트업 AWS 기본 세트        ₩300,000
   VPC + ALB + ECS Fargate + RDS + CloudFront
   + 서울 리전 최적화 + 비용 알람 + 보안 기본값

📦 개인정보보호법 대응 AWS 구성  ⭐      ₩800,000
   암호화 / 접근제어 / 감사로그 / 백업 정책

📦 ISMS-P 인증 대비 인프라 템플릿 ⭐⭐  ₩2,000,000
   인증 심사 항목에 매핑된 Terraform 코드
```

> ⭐ **핵심 인사이트:** 글로벌 모듈은 많지만 **한국 규제(개인정보보호법, ISMS-P, 전자금융감독규정)에 맞춘 것은 거의 없습니다.** 빈 시장입니다.

판매 채널: Gumroad, 자체 사이트, GitHub Sponsors, 크몽

#### 아이디어 3: 프리랜서 / 컨설팅

| 서비스 | 단가 |
|---|---|
| 인프라 코드화 (IaC 전환) | ₩1,000만 ~ 5,000만 |
| Terraform 코드 감사/리뷰 | ₩300만 ~ 800만 |
| 클라우드 비용 최적화 | 절감액의 20~30% |
| **상태 파일 복구 (긴급)** ⭐ | ₩200만 ~ (고단가) |
| 멀티클라우드 마이그레이션 | ₩3,000만 ~ |

> "터진 상태 파일 복구"는 단가가 매우 높습니다. 급하고, 할 수 있는 사람이 적고, 실패 비용이 크기 때문입니다. `internal/states/` 를 읽어둔 사람이 유리합니다.

---

### 🥈 티어 2 — 개발 필요, 3~12개월 (SaaS)

#### 아이디어 4: Plan 리뷰 SaaS ⭐⭐⭐ 가장 유망

**문제:** `terraform plan` 결과가 수천 줄이라 아무도 안 읽습니다. 그래서 사고가 납니다.

```
GitHub PR 생성 → plan -json 수집 → AI 분석 → PR 자동 코멘트

🔴 위험 2건
  • aws_db_instance.prod 삭제됨 → 데이터 영구 손실
  • 보안그룹 0.0.0.0/0:22 개방 → SSH 전체 공개
🟡 주의 1건
  • t3.micro → m5.4xlarge, 월 +$487 예상
🟢 안전 12건
```

**BUSL 안전한 이유:** `apply` 를 대신 실행하지 않습니다. plan 결과를 **읽고 분석만** 합니다 → 보완재.

```
가격: Free 월 50 plan / Team $49 / Business $199 / Enterprise 별도
스택: GitHub App + Node·Go + Claude API + React
기간: MVP 2~3개월 (1인 가능)
```

#### 아이디어 5: 클라우드 비용 예측기

```
이번 변경의 비용 영향
  현재 $2,340/월 → 변경후 $2,827/월   (+$487, +20.8%)
  • m5.4xlarge × 1   +$560
  • NAT Gateway × 2   +$90
  • (절감) t3.micro 3대 제거  -$23
  💡 Savings Plan 적용 시 $340/월 절감 가능
```

경쟁자 Infracost 존재. 차별화 = **한국 기업용**(원화, 한국 리전, 세금계산서, 한국어 지원).

#### 아이디어 6: Terraform MCP 서버 ⭐⭐ 타이밍 최고

```
안전(읽기전용)          ⚠️ 승인 필요
├ tf_validate          └ tf_apply (사람 승인 게이트)
├ tf_plan
├ tf_schema   ⭐ 환각 방지 핵심
├ tf_state_list
├ tf_cost_estimate
└ tf_security_scan
```

수익: 오픈소스 공개 → 스타 확보 → 엔터프라이즈 버전 유료(SSO, 감사로그, 승인 워크플로, 온프레미스)
또는 포트폴리오로 컨설팅·취업 연결

> **왜 지금인가:** MCP 생태계가 막 형성 중입니다. 선점 효과가 큰 시기이며, 1~2년 뒤면 늦습니다.

#### 아이디어 7: 틈새 프로바이더 개발 ⭐ 숨은 보석

```
아직 비어 있는 영역
  • 국내 클라우드 (네이버클라우드, KT클라우드, NHN클라우드)
  • 국내 SaaS (카카오워크, 토스페이먼츠, 아임포트)
  • 사내 레거시 시스템
  • 특수 장비 (네트워크 장비, 스토리지 어플라이언스)
```

| 수익 모델 | 금액 |
|---|---|
| 기업 의뢰 개발 | ₩2,000만 ~ 5,000만 (1~3개월) |
| 유지보수 계약 | 연 ₩500만 ~ |
| 오픈소스 공개 → 공식 채택 | 스폰서십 / 채용 |

> 국내 클라우드 사업자들은 Terraform 지원이 약합니다. 품질 좋은 프로바이더를 만들어 가져가면 **공식 채택 + 계약**으로 이어질 가능성이 있습니다.

#### 아이디어 8: 보안/컴플라이언스 스캐너 (한국 특화)

```
글로벌 경쟁자: tfsec, Checkov, Terrascan (전부 무료)
  ↓ 같은 걸 또 만들면 ❌

✅ 차별화 — 한국 규제 매핑
   • 개인정보보호법 제29조 → 암호화 규칙 체크
   • ISMS-P 인증 기준 → 자동 점검 리포트
   • 전자금융감독규정 → 망분리 구성 검증
   • 공공 클라우드 보안인증(CSAP) → 체크리스트
```

수익: 인증 심사 컨설팅과 묶어 판매 → **기업당 연 ₩1,000만 ~ 3,000만**
근거: 인증 못 받으면 사업을 못 하는 기업이 있습니다. 가격 저항이 낮습니다.

---

### 🥉 티어 3 — 장기, 팀 필요

#### 아이디어 9: 사내 개발자 플랫폼 (IDP) 구축

```
개발자가 웹에서 폼 작성
  "서비스명: payment-api, 환경: staging, DB: 필요"
      ↓
플랫폼이 승인된 모듈 조합으로 Terraform 코드 생성
      ↓
자동 plan → 팀장 승인 → apply
      ↓
3분 뒤 "완료. 접속 주소: ..."
```

수익: 대기업 대상 구축 프로젝트 **₩1억 ~ 5억**
⚠️ 고객사 사내 전용이면 BUSL 안전. **다수 고객에게 호스팅 제공하면 위반** → OpenTofu 권장.

#### 아이디어 10: 산업별 템플릿 마켓플레이스

```
🏥 의료   의료정보 보안 요건 충족 인프라
🏦 금융   전자금융감독규정 대응 망분리 구성
🛒 커머스 트래픽 폭증 대응 오토스케일링
🎮 게임   글로벌 멀티리전 + 저지연
🏭 제조   IoT 데이터 수집 파이프라인
```

---

### 📊 종합 비교표

| # | 아이디어 | 초기비용 | 기간 | 난이도 | 예상수익 | 리스크 | 추천 |
|---|---|---|---|---|---|---|---|
| 1 | 교육/강의 | 0원 | 즉시 | 중 | 월 50~1000만 | 낮음 | ⭐⭐⭐ |
| 2 | 모듈 판매 | 0원 | 1개월 | 중 | 건당 30~200만 | 낮음 | ⭐⭐ |
| 3 | 컨설팅 | 0원 | 즉시 | 상 | 건당 300~5000만 | 낮음 | ⭐⭐⭐ |
| 4 | Plan리뷰 SaaS | 소 | 3개월 | 상 | MRR $5k~50k | 중 | ⭐⭐⭐ |
| 5 | 비용 예측기 | 소 | 3개월 | 중 | MRR $3k~20k | 중 | ⭐⭐ |
| 6 | MCP 서버 | 0원 | 1개월 | 중 | 간접/채용 | 낮음 | ⭐⭐ |
| 7 | 프로바이더 | 0원 | 2개월 | 상 | 건당 2000~5000만 | 낮음 | ⭐⭐ |
| 8 | 보안스캐너 | 소 | 4개월 | 상 | 연 1000~3000만 | 중 | ⭐⭐ |
| 9 | IDP 구축 | 중 | 12개월 | 최상 | 건당 1~5억 | 높음 | ⭐ |
| 10 | 템플릿 마켓 | 중 | 12개월 | 상 | 변동 | 높음 | ⭐ |

---

### 🎯 추천 3단계 로드맵

```
━━━ 0~3개월: 신뢰 자산 만들기 (수익 기대 X) ━━━
  □ 유튜브 입문 시리즈 10편
  □ MCP 서버 오픈소스 공개 (아이디어 6)
  □ 기술 블로그 — 소스코드 분석 글
  목표: "이 사람 진짜 아네" 인식 + GitHub 스타

━━━ 3~9개월: 현금 흐름 만들기 ━━━
  □ 유료 강의 출시 (아이디어 1)
  □ 모듈/템플릿 판매 (아이디어 2)
  □ 컨설팅 의뢰 수락 (아이디어 3)
  목표: 월 300~500만원 안정화

━━━ 9~24개월: 확장 ━━━
  □ Plan 리뷰 SaaS 개발 (아이디어 4)
  □ 기업 교육 영업
  □ 프로바이더 수주 (아이디어 7)
  목표: 월 1,000만원+ / 자산화
```

**왜 이 순서인가**
> SaaS를 먼저 만들면 **"만들었는데 아무도 모름"** 상태가 됩니다.
> 교육/콘텐츠로 **먼저 신뢰와 청중을 확보**하면, 나중에 SaaS를 내놓을 때
> **첫 고객이 이미 준비되어 있습니다.**

---

## 14. 이 저장소가 나에게 주는 가치 (최종 정리)

### 🅰️ 지금 당장 — 사용자로서

> 비유: 전자제품을 샀는데 **AS 기사용 서비스 매뉴얼**까지 받은 상황

| 상황 | 보통 사람 | 이 저장소를 가진 사람 |
|---|---|---|
| 이상한 에러 발생 | 구글 → 스택오버플로 → 2시간 | `grep -r "에러문구" internal/` → 1분 |
| "왜 리소스가 재생성되지?" | 추측 | `docs/resource-instance-change-lifecycle.md` 공식 답 |
| 예제 코드 필요 | 블로그 복붙 (틀릴 수도) | **공식 .tf 예제 1,784개** |
| 신기능 궁금 | 출시까지 대기 | `.changes/v1.18` 에 다 보임 |
| 라이선스 판단 | 블로그 글 참고 | LICENSE 원문 직접 확인 |

### 🅱️ 앞으로 — 만드는 사람으로서

```
① 프로바이더 개발     → "내 서비스를 Terraform으로 관리" 라는 상품
② rpcapi 기반 도구    → Terraform을 엔진으로 내장한 제품
③ AI 에이전트 연동    → AI가 인프라를 설계/검증
④ 교육 콘텐츠         → 소스를 아는 강사 vs 사용법만 아는 강사
```

### ⚠️ 솔직한 경고

- **Go 2,014개 파일, 61MB.** 전체를 이해하려는 시도는 비효율적입니다. 궁금한 지점만 `grep` 하세요.
- **Terraform을 처음 배우는 용도로는 최악의 자료입니다.** 사용법은 공식 튜토리얼로 배우고, 이 저장소는 **막혔을 때 펼치는 사전**으로 쓰세요.
- **BUSL 라이선스.** 사내·개인 사용은 자유, "Terraform 호스팅 서비스"를 유료로 팔면 위반입니다.

---

## 📚 참고 링크

| 구분 | 주소 |
|---|---|
| **이 저장소** | https://github.com/bmshin94/terraform |
| 원본 저장소 | https://github.com/hashicorp/terraform |
| 공식 문서 | https://developer.hashicorp.com/terraform |
| 공식 튜토리얼 | https://developer.hashicorp.com/terraform/tutorials |
| Terraform Registry | https://registry.terraform.io |
| 플러그인 개발 가이드 | https://developer.hashicorp.com/terraform/plugin |
| 자격증 (Terraform Associate) | https://www.hashicorp.com/certification |
| 커뮤니티 포럼 | https://discuss.hashicorp.com/c/terraform-core |
| OpenTofu (MPL 2.0 포크) | https://opentofu.org |
| 라이선스 원문 | https://github.com/hashicorp/terraform/blob/main/LICENSE |

### 저장소 내 추천 문서 (직접 읽을 가치 있음)

```
docs/architecture.md                          내부 아키텍처 총정리 ⭐
docs/resource-instance-change-lifecycle.md    리소스 생명주기 + 다이어그램 ⭐
docs/planning-behaviors.md                    plan 동작 원리
docs/destroying.md                            삭제 처리 로직
docs/plugin-protocol/README.md                플러그인 프로토콜 사양
docs/debugging.md                             디버깅 방법 (VSCode 설정 포함)
BUILDING.md                                   소스 빌드 공식 절차
BUGPROCESS.md                                 버그 트리아지 프로세스
.changes/v1.18/                               개발 중 신기능 목록
```

---

*분석 일자: 2026-10-07 · 분석 대상 커밋: `26ee0ef` · 브랜치: `claude/brave-mccarthy-dr9p4i`*
