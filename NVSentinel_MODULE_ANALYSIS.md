# NVSentinel 모듈 구조 분석

## 개요
NVSentinel은 NVIDIA GPU 클러스터를 위한 헬스 모니터링 및 장애 관리 시스템입니다. 이 문서는 프로젝트의 전체 모듈 구조와 각 컴포넌트의 역할, 상호작용을 엄밀하게 분석한 내용입니다.

---

## 1. Health Monitors (장애 감지 계층)

### GPU Health Monitor
- **위치**: `modules/health-monitors/gpu-health-monitor/`
- **언어**: Python
- **파일 수**: 32 files
- **역할**: DCGM(Data Center GPU Manager) 기반 GPU 상태 모니터링
- **주요 기능**:
  - GPU 에러(XID, SXID) 감지
  - GPU 성능 메트릭 수집
  - 실시간 헬스 체크

### Syslog Health Monitor
- **위치**: `modules/health-monitors/syslog-health-monitor/`
- **언어**: Go
- **파일 수**: 54 files
- **역할**: 시스템 로그에서 XID/SXID 에러 감지
- **주요 기능**:
  - 커널 로그 파싱
  - NVIDIA 드라이버 에러 식별
  - 실시간 syslog 스트리밍 처리

### NIC Health Monitor
- **위치**: `modules/health-monitors/nic-health-monitor/`
- **언어**: Go
- **파일 수**: 50 files
- **역할**: 네트워크 카드 상태 모니터링
- **주요 기능**:
  - NIC 에러 감지
  - 네트워크 연결성 확인
  - RDMA 상태 모니터링

### CSP Health Monitor
- **위치**: `modules/health-monitors/csp-health-monitor/`
- **언어**: Go
- **파일 수**: 20 files
- **역할**: 클라우드 서비스 프로바이더 유지보수 이벤트 감지
- **주요 기능**:
  - AWS/GCP/Azure 유지보수 알림 수신
  - 예정된 다운타임 감지
  - 클라우드별 이벤트 정규화

---

## 2. Platform Connectors (이벤트 통합 계층)

- **위치**: `modules/platform-connectors/`
- **파일 수**: 48 files
- **역할**: 모든 헬스 모니터 이벤트 수신 및 파이프라인 처리
- **주요 기능**:
  - 이벤트 중복 제거 (Deduplication)
  - 메타데이터 enrichment
  - 데이터스토어 저장
  - 이벤트 정규화 및 표준화

---

## 3. Data Models & Store Client (데이터 계층)

### Data Models
- **위치**: `modules/data-models/`
- **파일 수**: 12 files
- **역할**: 공통 데이터 모델 및 프로토콜 버퍼 정의
- **주요 기능**:
  - 헬스 이벤트 스키마 정의
  - 노드 상태 모델
  - 프로토콜 버퍼 직렬화/역직렬화

### Store Client
- **위치**: `modules/store-client/`
- **파일 수**: 47 files
- **역할**: MongoDB/PostgreSQL 추상화 및 변경 스트림
- **주요 기능**:
  - 데이터스토어 인터페이스 추상화
  - ChangeStream 기반 실시간 이벤트 구독
  - 트랜잭션 관리
  - 재시도 로직 및 오류 처리

---

## 4. Fault Quarantine (격리 계층)

- **위치**: `modules/fault-quarantine/`
- **파일 수**: 26 files
- **역할**: 장애 규칙 평가 후 노드 격리 및 애너테이션 관리
- **주요 기능**:
  - 장애 심각도 평가
  - 노드 격리 결정
  - Kubernetes 노드 애너테이션
  - 격리 상태 관리

---

## 5. Fault Remediation (복구 계층)

- **위치**: `modules/fault-remediation/`
- **파일 수**: 25 files
- **역할**: 격리된 노드에 대한 자동 복구 작업
- **주요 기능**:
  - GPU 리셋
  - 노드 재부팅
  - 복구 전략 실행
  - 복구 결과 모니터링

---

## 6. Node Drainer (드레인 계층)

- **위치**: `modules/node-drainer/`
- **파일 수**: 25 files
- **역할**: 장애 노드 워크로드 안전한 대피
- **주요 기능**:
  - Pod 안전 이동
  - 드레인 전략 (cordon, drain)
  - 워크로드 상태 모니터링
  - 드레인 완료 확인

---

## 7. Labeler (라벨링 계층)

- **위치**: `modules/labeler/`
- **파일 수**: 12 files
- **역할**: 노드 GPU 개수, 상태 등 라벨 자동 부여
- **주요 기능**:
  - 노드 메타데이터 기반 라벨 생성
  - GPU 토폴로지 라벨링
  - 실시간 라벨 동기화

---

## 8. Metadata Collector (메타데이터 수집)

- **위치**: `modules/metadata-collector/`
- **파일 수**: 20 files
- **역할**: GPU, NIC 토폴로지 메타데이터 수집
- **주요 기능**:
  - 하드웨어 토폴로지 매핑
  - NVLink 연결 정보 수집
  - 네트워크 토폴로지 발견

---

## 9. Janitor (수명 주기 관리)

- **위치**: `modules/janitor/`
- **파일 수**: 49 files
- **역할**: CRD 수명 주기 관리, 외부 복구 연동, 분산 락 처리
- **주요 기능**:
  - GPUReset 컨트롤러
  - RebootNode 컨트롤러
  - TerminateNode 컨트롤러
  - 분산 락 관리 (Distributed Lock)
  - 외부 시스템 연동

---

## 10. Event Exporter (이벤트 내보내기)

- **위치**: `modules/event-exporter/`
- **파일 수**: 18 files
- **역할**: 헬스 이벤트 외부 시스템으로 내보내기
- **주요 기능**:
  - 이벤트 포맷 변환
  - 외부 엔드포인트 전송 (Webhook, Kafka 등)
  - 전송 보증 (At-least-once delivery)

---

## 11. Log Collector (로그 수집)

- **위치**: `modules/log-collector/`
- **파일 수**: 13 files
- **역할**: 장애 시 진단 로그 수집
- **주요 기능**:
  - 자동 로그 스냅샷
  - 진단 데이터 패키징
  - 스토리지 업로드

---

## 12. Preflight (사전 점검)

- **위치**: `modules/preflight/`
- **파일 수**: 24 files
- **역할**: GPU 작업 전 사전 건강도 점검
- **주요 기능**:
  - 작업 전 GPU 상태 확인
  - 네트워크 연결성 테스트
  - 사전 장애 예방

---

## 13. Preflight Checks (점검 도구들)

### DCGM Diag
- **위치**: `modules/preflight-checks/dcgm-diag/`
- **파일 수**: 23 files
- **역할**: DCGM 진단 도구

### NCCL AllReduce
- **위치**: `modules/preflight-checks/nccl-allreduce/`
- **파일 수**: 24 files
- **역할**: NCCL AllReduce 통신 테스트

### NCCL Loopback
- **위치**: `modules/preflight-checks/nccl-loopback/`
- **파일 수**: 9 files
- **역할**: NCCL 루프백 테스트

---

## 14. Janitor Provider (클라우드 연동)

- **위치**: `modules/janitor-provider/`
- **파일 수**: 15 files
- **역할**: 클라우드별 노드 관리
- **지원 클라우드**:
  - AWS
  - GCP
  - Azure
  - OCI (Oracle Cloud Infrastructure)
  - Nebius

---

## 15. Plugins (확장 플러그인)

- **위치**: `plugins/`
- **역할**: 확장 가능한 플러그인 아키텍처
- **주요 플러그인**:
  - Slurm 기반 드레인
  - 테스트용 Slurm 모방

---

## 16. Commons (공통 유틸리티)

- **위치**: `commons/`
- **파일 수**: 35 files
- **역할**: 공통 라이브러리
- **주요 기능**:
  - 로깅 (Logging)
  - 메트릭 (Metrics)
  - 트레이싱 (Tracing)
  - 감사 로깅 (Audit Logging)
  - 유틸리티 함수

---

## 17. API (gRPC API)

- **위치**: `api/`
- **파일 수**: 8 files
- **역할**: gRPC API 정의
- **주요 API**:
  - GPU 디바이스 API
  - CSP 프로바이더 API

---

## 18. GPU Reset

- **위치**: `gpu-reset/`
- **파일 수**: 4 files
- **역할**: GPU 리셋 쉘 스크립트

---

## 전체 아키텍처 흐름

```
[Health Monitors]
       ↓
[Platform Connectors] → [Data Models] → [Store Client] → [Datastore (MongoDB/PostgreSQL)]
       ↓                                              ↓
[Event Exporter]                              [ChangeStream Subscription]
                                                       ↓
                                            [Fault Quarantine]
                                                       ↓
                    ┌──────────────────────┬───────────┴───────────┬──────────────────────┐
                    ↓                      ↓                       ↓                      ↓
              [Node Drainer]      [Fault Remediation]        [Labeler]            [Metadata Collector]
                    ↓                      ↓
              [Janitor Controllers] → [Janitor Provider] → [Cloud APIs]
                    ↓
              [Log Collector]
```

---

## 코드 수정 시 고려사항

### 1. 데이터 일관성
- 스키마 호환성 유지 (프로토콜 버퍼 버전 관리)
- 마이그레이션 전략 수립
- 하위 호환성 보장

### 2. 분산 시스템 특성
- 분산 락 메커니즘 준수
- 멱등성 (Idempotency) 보장
- 재시도 로직 구현

### 3. 상태 관리
- 상태 전이 머신 (State Machine) 명확히 유지
- 상태 불일치 방지
- 복구 가능성 확보

### 4. 관측 가능성 (Observability)
- 메트릭 체계 유지 (Prometheus 호환)
- 감사 로깅 (Audit Logging) 필수 포함
- 분산 트레이싱 (OpenTelemetry) 연동

### 5. 확장성
- 플러그인 아키텍처 준수
- 인터페이스 기반 설계
- 의존성 주입 (Dependency Injection) 활용

### 6. 테스트
- 단위 테스트 커버리지 유지
- 통합 테스트 시나리오 포함
- E2E 테스트 검증

---

## 주요 기술 스택

- **언어**: Go, Python, Shell
- **데이터스토어**: MongoDB, PostgreSQL
- **메시징**: Kubernetes Events, ChangeStreams
- **모니터링**: Prometheus, OpenTelemetry
- **배포**: Helm, Kubernetes
- **GPU 통합**: NVIDIA DCGM, GPU Operator

---

## 문서 정보

- **생성일**: 2025
- **분석 대상**: NVSentinel v1.x
- **문서 버전**: 1.0

---

> **참고**: 본 문서는 NVSentinel 프로젝트의 모듈 구조를 엄밀하게 분석한 것으로, 코드 수정 시 반드시 참조해야 합니다.
