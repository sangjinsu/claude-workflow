# claude-workflow

Define and run YAML workflows inside Claude Code. Zero dependencies.

배포, 인시던트 대응, 인프라 점검 등 반복되는 작업 흐름을 YAML로 정의하면
Claude Code가 각 단계를 순서대로 실행합니다.

## 특징

- **외부 의존성 없음** - Python, Node, 바이너리 설치 불필요
- **Claude가 직접 실행** - YAML을 읽고 변수를 치환하고 Bash로 실행
- **간단한 YAML 스키마** - 5분이면 워크플로우를 정의할 수 있음
- **depends_on으로 실행 순서 제어** - DAG 기반 의존성 관리

## 설치

### Claude Code 마켓플레이스 (권장)

```
/plugin install sangjinsu/claude-workflow
```

### 수동 설치

```bash
git clone https://github.com/sangjinsu/claude-workflow.git ~/.claude/plugins/claude-workflow
```

## 사용법

### 워크플로우 생성

```
/workflow-builder create deploy
```

템플릿을 선택하고 `.workflows/` 디렉토리에 워크플로우 파일을 생성합니다.

### 워크플로우 실행

```
/workflow-builder run deploy
```

`.workflows/deploy.yaml`을 읽고 각 step을 순서대로 실행합니다.

### 기타 명령

```
/workflow-builder list              # 워크플로우 목록
/workflow-builder show deploy       # 상세 보기
/workflow-builder validate deploy   # 유효성 검증
```

## 워크플로우 YAML 예시

```yaml
workflow:
  name: "배포 사전 점검"
  version: "1.0.0"

config:
  base:
    SERVICE_NAME: "my-service"
    NAMESPACE: "default"

steps:
  - id: check-pods
    name: "Pod 상태 확인"
    type: command
    commands:
      - "kubectl get pods -n ${NAMESPACE} -l app=${SERVICE_NAME}"
    on_failure: abort

  - id: health-check
    name: "헬스체크"
    type: command
    commands:
      - "curl -sf http://localhost:8080/health"
    depends_on: [check-pods]
    on_failure: continue
```

## 워크플로우 저장 위치

| 위치 | 경로 | 용도 |
|------|------|------|
| 프로젝트 | `.workflows/` | 프로젝트별 워크플로우 (Git 추적) |
| 글로벌 | `~/.workflows/` | 모든 프로젝트에서 사용 |

## 라이선스

MIT
