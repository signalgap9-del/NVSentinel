# NVSentinel 기여자 가이드: 환경 설정부터 첫 PR 까지

> **초기 기여자를 위한 완전 가이드**  
> 이 문서는 NVSentinel 프로젝트에 기여하려는 개발자를 위해 환경 설정, 테스트 실행, 코드 분석, PR 제출까지의 전 과정을 상세히 설명합니다.

---

## 📋 목차

1. [시작하기 전에](#-시작하기-전에)
2. [시스템 요구사항](#-시스템-요구사항)
3. [개발 환경 설정](#-개발-환경-설정)
4. [테스트 실행 방법](#-테스트-실행-방법)
5. [코드 분석 및 기여 영역 찾기](#-코드-분석-및-기여-영역-찾기)
6. [PR 작성 및 제출](#-pr-작성-및-제출)
7. [자주 묻는 질문](#-자주-묻는-질문)

---

## 🚀 시작하기 전에

### 필수 지식
- **Go 1.25+** 기본 문법 이해
- **Kubernetes** 기본 개념 (Pod, Deployment, Service 등)
- **Git** 기본 워크플로우 (fork, branch, commit, PR)
- **Docker** 컨테이너 기본 이해

### NVSentinel 이란?
NVSentinel 은 GPU 지원 Kubernetes 클러스터를 위한 **지능형 장애 감지 및 자동 복구 시스템**입니다:
- **감지**: GPU, 시스템 로그, 클라우드 이벤트에서 장애 실시간 감지
- **격리**: 장애 노드를 자동으로 격리 (cordon)
- **대피**: 워크로드를 안전한 노드로 이동 (drain)
- **복구**: GPU 리셋, 노드 재부팅 등 자동 복구 작업 수행
- **검증**: 사전 점검을 통해 정상 여부 확인 후 클러스터에 복귀

### 현재 분석된 주요 취약점 (우리가 식별한 것)
1. **경쟁 조건**: 분산 락 만료와 작업 완료 사이의 타이밍 문제
2. **리소스 누수**: DCGM 세션, 로그 버퍼, 네트워크 연결 정리 미흡
3. **오류 처리 불완전**: GPU 리셋 실패 시 대체 경로 부족
4. **보안 취약**: 민감 정보 암호화 미비, API 키 노출 가능성
5. **관측성 격차**: 메트릭 누락, 감사 로그 부재

> ✅ **초기 기여자 추천 영역**: Health Event 유틸리티 및 오류 처리 로직 리팩토링  
> 이 영역은 핵심 데이터 흐름을 이해하면서도 시스템 장애 위험이 낮아 안전한 진입점입니다.

---

## 💻 시스템 요구사항

### 최소 사양
| 구성 요소 | 최소 요구사항 | 권장 사양 |
|-----------|---------------|-----------|
| **CPU** | 4 코어 | 8 코어 이상 |
| **RAM** | 8 GB | 16 GB 이상 |
| **저장공간** | 20 GB 여유 공간 | 50 GB SSD 이상 |
| **OS** | Linux (Ubuntu 20.04+), macOS 12+ | Linux (Ubuntu 22.04 LTS) |
| **Docker** | Docker Desktop 또는 Docker Engine 20.10+ | 최신 안정 버전 |

### 아키텍처 지원
- **x86_64 (amd64)**: 완전 지원
- **ARM64 (aarch64)**: 완전 지원 (Apple Silicon, ARM 서버)

### 필수 소프트웨어
```bash
# 버전 확인 명령어
go version                    # Go 1.25+ 필요
docker --version              # Docker 20.10+ 필요
kubectl version --client      # kubectl 설치 필요
helm version                  # Helm 3.0+ 필요
make --version                # GNU Make 필요
yq --version                  # YAML 프로세서 필요
```

### 선택적 소프트웨어 (로컬 Kubernetes 개발용)
- **Tilt**: 핫 리로딩 개발 환경
- **ctlptl**: Kubernetes 클러스터 관리
- **Kind**: 로컬 Kubernetes 클러스터
- **MongoDB Compass**: 데이터베이스 GUI

---

## 🛠️ 개발 환경 설정

### 1 단계: 저장소 클론

```bash
# GitHub 에서 포크한 후 클론
git clone https://github.com/YOUR_USERNAME/NVSentinel.git
cd NVSentinel

# upstream 원격 추가 (선택 사항이지만 권장)
git remote add upstream https://github.com/NVIDIA/NVSentinel.git
git fetch upstream
```

### 2 단계: 자동 환경 설정 (권장)

```bash
# 모든 의존성 자동 설치 (yq, Go, Python, lint 도구 등)
make dev-env-setup AUTO_MODE=true

# 디버그 모드 (설치 실패 시 상세 로그 확인)
DEBUG=true make dev-env-setup AUTO_MODE=true
```

이 스크립트는 다음을 수행합니다:
- OS 와 아키텍처 자동 감지
- `.versions.yaml` 에서 버전 정보 읽기
- yq 설치 (버전 관리에 필요)
- Go 1.26.3 설치
- Python 3.13 및 Poetry 설치
- golangci-lint, gotestsum, gocover-cobertura 설치
- Protocol Buffers 컴파일러 및 gRPC 도구 설치
- shellcheck, addlicense, black 설치

### 3 단계: 수동 환경 설정 (고급 사용자)

```bash
# Go 도구 직접 설치
make install-lint-tools

# 버전 확인
make show-versions
```

### 4 단계: 로컬 Kubernetes 환경 (선택 사항)

```bash
# Tilt 기반 개발 환경 시작 (종료형 Kind 클러스터 + 로컬 레지스트리)
make dev-env

# 또는 수동으로 단계별 실행
make cluster-create    # Kind 클러스터 생성
make tilt-up           # Tilt 시작 (http://localhost:10350 에서 UI 접근)

# 상태 확인
make cluster-status

# 종료
make dev-env-clean     # Tilt 중지 및 클러스터 삭제
```

### 5 단계: 환경 변수 설정

```bash
# GOPATH 및 Go 캐시 설정
export GOPATH=$(go env GOPATH)
export GO_CACHE_DIR=$(go env GOCACHE)

# 컨테이너 레지스트리 (포크한 경우)
export CONTAINER_REGISTRY="ghcr.io"
export CONTAINER_ORG="your-github-username"
```

---

## 🧪 테스트 실행 방법

### 테스트 계층 구조

NVSentinel 은 세 가지 테스트 계층을 가집니다:

1. **단위 테스트**: 개별 모듈 내부 함수 테스트
2. **모듈 테스트**: 전체 모듈 기능 테스트
3. **E2E 테스트**: 전체 시스템 통합 테스트 (Kubernetes 클러스터 필요)

### 1. 단위 테스트 실행 (가장 빠름, GPU 불필요)

```bash
# 특정 Go 모듈의 단위 테스트만 실행
cd platform-connectors
make test

# 또는 상위 디렉토리에서
make platform-connectors-test

# 테스트 결과 상세 출력
make -C platform-connectors test GOTESTSUM_FORMAT=standard-verbose

# 커버리지 리포트 생성
make -C platform-connectors test-cover

# HTML 로 커버리지 확인
go tool cover -html=platform-connectors/coverage.txt
```

### 2. 모듈 린트 및 테스트 (권장)

```bash
# 린트 + 테스트 한 번에 실행
make platform-connectors-lint-test

# 여러 모듈 동시 테스트
make commons-lint-test
make platform-connectors-lint-test

# 모든 Go 모듈 테스트 (시간 소요: 30 분+)
make lint-test-all
```

### 3. E2E 테스트 실행 (전체 시스템, Kubernetes 클러스터 필요)

```bash
# E2E 테스트 실행 (Tilt 환경에서)
make e2e-test

# 또는 tests 디렉토리에서 직접
cd tests
make test

# PostgreSQL 사용 시 (기본값은 MongoDB)
cd tests
USE_POSTGRESQL=1 make test

# 특정 테스트만 실행
cd tests
go test -run TestPlatformConnectorDedup -tags=mongodb ./...
```

### 4. Helm 차트 테스트

```bash
# Helm 유닛 테스트
make helm-test

# Helm 차트 린트
make -C distros/kubernetes lint
```

### 5. Python 모듈 테스트 (GPU Health Monitor)

```bash
cd health-monitors/gpu-health-monitor

# Poetry 환경 설정
poetry install

# 테스트 실행
poetry run pytest

# 린트 실행
poetry run black --check .
```

### 6. 빠른 피드백 루프 (Tilt 사용)

```bash
# Tilt 시작 후 코드 변경 시 자동 재빌드 및 테스트
make tilt-up

# 브라우저에서 http://localhost:10350 접속
# 리소스 상태, 빌드 로그, 테스트 결과 실시간 확인 가능
```

### 테스트 실패 시 디버깅

```bash
# 상세 로그 출력
gotestsum --format testname -- -v ./...

# 특정 테스트만 반복 실행
go test -run TestSpecificFunction -count=3 ./...

# 경쟁 조건 탐지
go test -race ./...

# 메모리 프로파일링
go test -memprofile=mem.out ./...
go tool pprof mem.out
```

---

## 🔍 코드 분석 및 기여 영역 찾기

### 현재 식별된 기여 기회 (우리가 분석한 것)

#### 🎯 추천: Health Event 유틸리티 리팩토링

**위치**: `commons/pkg/eventutil/`, `commons/pkg/errutil/`

**문제**:
- 여러 모듈에 중복된 이벤트 처리 로직 존재
- 일시적 오류 감지 로직이 표준화되지 않음
- 재시도 로직이 일관되지 않음

**기여 내용**:
```go
// 기존: 각 모듈마다 중복 구현
// 개선: errutil 에 통합된 일시적 오류 감지 및 재시도 유틸리티

// commons/pkg/errutil/errors.go 에 추가 제안
func IsTemporaryError(err error) bool {
    // 네트워크 타임아웃, 리소스 부족 등 일시적 오류 판별
}

func WithRetry(maxAttempts int, delay time.Duration, fn func() error) error {
    // 지수 백오프 포함 재시도 로직
}
```

**테스트 전략**:
1. 기존单元测试 수정 없이 통과 확인
2. 새로운 유틸리티 함수에 대한 단위 테스트 작성
3. device platform connector 에서 새 유틸리티 사용하도록 리팩토링
4. 성능 저하 없는지 벤치마크 테스트

#### 2. DCGM 세션 관리 개선

**위치**: `health-monitors/gpu-health-monitor/`

**문제**:
- 예외 상황에서 DCGM 세션 정리 안 될 수 있음
- defer 문 누락 가능성

**기여 내용**:
```python
# 기존
def monitor_gpu():
    session = dcgm.create_session()
    # ... 예외 발생 시 session cleanup 누락 가능

# 개선
def monitor_gpu():
    session = None
    try:
        session = dcgm.create_session()
        # ... 모니터링 로직
    finally:
        if session:
            dcgm.destroy_session(session)
```

#### 3. 다단계 복구 전략 구현

**위치**: `fault-remediation/`, `janitor/`

**문제**:
- GPU 리셋 실패 시 바로 노드 재부팅으로 직행
- 중간 단계 (드라이버 리로드 등) 없음

**기여 내용**:
```yaml
# Helm values.yaml 에 추가
faultRemediation:
  strategy:
    steps:
      - name: gpu_reset
        timeout: 5m
        retries: 2
      - name: driver_reload    # 새로 추가
        timeout: 3m
        retries: 1
      - name: node_reboot
        timeout: 10m
        retries: 1
```

#### 4. 민감 데이터 암호화

**위치**: `store-client/`, `commons/pkg/config/`

**문제**:
- MongoDB 연결 문자열에 비밀번호 평문 노출
- API 키 환경 변수로 직접 전달

**기여 내용**:
```go
// Kubernetes Secret 참조 기능 추가
type StoreConfig struct {
    PasswordSecretRef struct {
        Name      string `json:"name"`
        Namespace string `json:"namespace"`
        Key       string `json:"key"`
    } `json:"passwordSecretRef,omitempty"`
}
```

### 코드 분석 도구 활용

```bash
# 정적 분석
make platform-connectors-lint

# 복잡도 분석
gocyclo -over 15 platform-connectors/pkg/

# 의존성 그래프 시각화
go mod graph | dot -Tpng > deps.png

# 테스트 커버리지 확인
make -C platform-connectors test-cover
go tool cover -html=platform-connectors/coverage.txt
```

---

## 📝 PR 작성 및 제출

### 1. 이슈 확인 및 브랜치 생성

```bash
# 관련 이슈 확인 (GitHub Issues)
# https://github.com/NVIDIA/NVSentinel/issues

# 브랜치 생성 (명명 규칙: feat/, fix/, docs/, refactor/)
git checkout -b refactor/commons-event-util

# upstream 과 동기화
git fetch upstream
git rebase upstream/main
```

### 2. 코드 변경 및 테스트

```bash
# 코드 수정 후 반드시 테스트
make commons-lint-test

# 변경 사항 확인
git diff

# 커밋 (서명 포함)
git commit -S -m "refactor: unify health event utilities in commons

- Consolidate duplicate event processing logic
- Add IsTemporaryError() for retry decisions
- Implement WithRetry() with exponential backoff
- Update device connector to use new utilities

Signed-off-by: Your Name <your.email@example.com>"
```

### 3. PR 작성 체크리스트

- [ ] **Self-review 완료**: `git diff upstream/main` 직접 검토
- [ ] **테스트 통과**: `make lint-test-all` 또는 관련 모듈 테스트
- [ ] **문서 업데이트**: 코드 주석, README, godoc 등
- [ ] **커밋 서명**: DCO 서명 포함 (`Signed-off-by`)
- [ ] **브랜치 최신화**: `git rebase upstream/main`
- [ ] **PR 설명 작성**: 변경 사유, 테스트 방법, 관련 이슈 링크

### 4. PR 템플릿 예시

```markdown
## Summary
- Consolidated duplicate health event utilities in commons/pkg/eventutil
- Added unified temporary error detection in commons/pkg/errutil
- Refactored device platform connector to use new utilities

## Type of Change
- [x] 🔧 Refactoring
- [x] ✨ New feature (utility functions)
- [ ] 🐛 Bug fix
- [ ] 📚 Documentation

## Component(s) Affected
- [x] Core Services (commons)
- [ ] Health Monitors
- [ ] Fault Management
- [ ] Janitor

## Testing
- [x] Tests pass locally: `make commons-lint-test`
- [x] No breaking changes
- [ ] Manual testing completed (if applicable)

## Checklist
- [x] Self-review completed
- [x] Documentation updated (godoc comments)
- [x] Ready for review

## Related Issues
Fixes #(issue number if applicable)
```

### 5. PR 제출 후

1. **CI 체크 기다리기**: GitHub Actions 자동 실행
2. **리뷰 응답**: CodeRabbit 및 인간 리뷰어 코멘트에 답변
3. **수정 요청 시**: 같은 브랜치에 추가 커밋 푸시
4. **Merge 대기**: 승인 후 maintainers 가 병합

---

## ❓ 자주 묻는 질문

### Q: GPU 가 없어도 기여할 수 있나요?
**A**: 네! 대부분의 모듈 (platform-connectors, commons, fault-quarantine 등) 은 GPU 없이도 테스트 가능합니다. GPU 가 필요한 모듈은 CI 에서 테스트됩니다.

### Q: 로컬에서 E2E 테스트를 꼭 실행해야 하나요?
**A**: 초기 기여자는 단위 테스트와 모듈 테스트만 통과해도 충분합니다. E2E 테스트는 CI 에서 자동 실행됩니다.

### Q: PR 이 거절될까 봐 두려워요.
**A**: 작은 변경부터 시작하세요. 문서 수정, 오타 수정, 테스트 추가 등으로 시작하여 점진적으로 복잡한 기여로 나아가는 것을 권장합니다.

### Q: 어떤 이슈부터 처리해야 할까요?
**A**: `good first issue` 또는 `help wanted` 라벨이 붙은 이슈부터 시작하세요. 우리가 분석한 Health Event 유틸리티 리팩토링도 좋은 시작점입니다.

### Q: 개발 환경 설정이 계속 실패해요.
**A**: 
```bash
# 디버그 모드로 상세 로그 확인
DEBUG=true make dev-env-setup AUTO_MODE=true

# 수동으로 단계별 설치
make install-lint-tools
poetry install  # Python 모듈용

# 그래도 안 되면 GitHub Issues 에 환경 정보와 함께 질문
```

### Q: 테스트가 너무 느려요.
**A**: 
- 특정 모듈만 테스트: `make platform-connectors-test`
- 캐시 활용: 두 번째 실행부터는 빠릅니다
- Tilt 사용: 코드 변경 시 자동 재빌드

---

## 📚 추가 리소스

### 공식 문서
- [README.md](../README.md): 프로젝트 개요
- [CONTRIBUTING.md](../CONTRIBUTING.md): 기여 가이드라인
- [DEVELOPMENT.md](../DEVELOPMENT.md): 개발 가이드
- [NVSentinel_MODULE_ANALYSIS.md](./NVSentinel_MODULE_ANALYSIS.md): 모듈 구조 분석

### 학습 자료
- [Go 공식 문서](https://go.dev/doc/)
- [Kubernetes 문서](https://kubernetes.io/docs/home/)
- [DCGM 문서](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/index.html)

### 커뮤니티
- [GitHub Discussions](https://github.com/NVIDIA/NVSentinel/discussions)
- [GitHub Issues](https://github.com/NVIDIA/NVSentinel/issues)

---

## 🎯 다음 단계

1. ✅ 이 문서 읽음
2. ⬜ 개발 환경 설정 (`make dev-env-setup`)
3. ⬜ 단위 테스트 실행 (`make commons-lint-test`)
4. ⬜ 코드 분석 (`commons/pkg/eventutil/` 살펴보기)
5. ⬜ 작은 리팩토링 시작
6. ⬜ PR 작성 및 제출

**기여하시겠습니까?** 지금 바로 시작하세요!  
어떤 단계에서 도움이 필요하시면 언제든지 물어보세요.

---

*문서 버전: 1.0*  
*최종 업데이트: 2025 년*  
*NVSentinel 프로젝트 커뮤니티*
