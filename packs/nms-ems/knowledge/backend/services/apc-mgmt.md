# apc_mgmt Service (skeleton)

## 서비스 책임
APC(Automatic Power Control) 외부 연동 세션 시작·관리용 마이크로서비스 — **현재 skeleton(scaffold) 상태로 핵심 비즈니스 로직 미구현**.
`processMicroServiceRestRequest()`는 null 반환, `Worker_apc_mgmt.process_APC_Mgmt_Start()`는 빈 구현, `apc_mgmtCD.WorkTypeBook` 상수는 전부 주석 처리. 기능 요청이 오면 "수정"이 아니라 "최초 구현"으로 접근한다.

## 시작점과 실행 설정
- `ServiceStart_apc_mgmt.main()` → `runMicroService.run("apc_mgmt", new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- 포트 10463 (`home/config/apc_mgmt.xml`), MyBatis 인프라는 준비되어 있으나 전용 mapper/SQL XML 없음
- `RestInvokerFunction_apc_mgmt.initialized()`가 별도 Thread로 `m_Worker.process_APC_Mgmt_Start("default site", "my5ghfruser", ...)` 호출 — **자격증명 하드코딩**. 구현 시 `apc_mgmt.xml`에서 읽도록 개선할 것.

## 아키타입
미분류(skeleton). `RestInvokerFunction_apc_mgmt`는 `AbsRestInvokerFunction_Service`를 **직접 상속**(`AbsRestInvokerFunction_my5GService` 미사용) — unify/inf/manualWorkerLoader 어느 등록 메커니즘도 없다. 외부 APC 서버 연동이 목적이므로 구현 시 아키타입 E 패턴(`../feature-extension.md`)을 따르는 것이 자연스럽다.

## REST contract
등록된 request type 없음. `getContentList()` 미등록, `processMicroServiceRestRequest()`는 빈 JSONObject 조립 후 실질 처리 없이 종료.

## 주요 Worker/Function 흐름
`Worker_apc_mgmt` 1개뿐. `process_APC_Mgmt_Start()`는 빈 메서드.

## DB/Mongo/파일 흐름 · scheduler
전용 mapper/XML/Mongo/schedule 코드 전부 없음.

## 기능 확장점 (feature seams) — 구현 시 절차
| seam 유형 | 추가 방법 | 등록 위치/근거 파일 |
| --- | --- | --- |
| ① rest-request-type | `apc_mgmtCD.WorkTypeBook`에 상수 정의 → `getContentList()` 오버라이드로 `RestContent` 등록 → `processMicroServiceRestRequest()`에 `aRequestType` 분기 추가 | `function/RestInvokerFunction_apc_mgmt.java` |
| ② worker-function | `Worker_apc_mgmt.process_APC_Mgmt_Start()`에 실제 APC 서버 연결 구현 또는 동일 클래스에 public 메서드 추가, `RestInvokerFunction`의 `m_Worker` 참조로 호출 | `worker/Worker_apc_mgmt.java` |
| data-model | `apc_mgmtCD.WorkTypeBook`의 주석 상수(Add/Modify 등) 활성화 또는 내부 class 추가 | `common/data/apc_mgmtCD.java` |
| external-adapter | `initialized()` 내 Thread에서 `process_APC_Mgmt_Start()` 호출 패턴 기존재 — 새 외부 연결은 Worker에 메서드 추가 후 이 Thread 블록에서 호출 | `function/RestInvokerFunction_apc_mgmt.java` |
| ③ scheduler-job | `schedule/` 패키지 없음. 필요 시 신설 + 스케줄러 추상 클래스 상속 Job 작성 + Worker 초기화 시 등록 | `worker/Worker_apc_mgmt.java` |

## mock/fake 지점
- `Worker_apc_mgmt.process_APC_Mgmt_Start()` — 미구현이라 테스트 시 mock 필수
- `WorkflowMap.getWorker(AbsWorkerRestContentManager.class)` — 공통 Worker mock

## 변경 위험
- `process_APC_Mgmt_Start()`는 서비스 시작 시 Thread로 즉시 실행 — 예외 처리 누락 시 Thread가 조용히 종료됨
- `AbsRestInvokerFunction_Service` 직접 상속 탓에 my5G 공통 Worker(WorkerLocalBaseDataCommon 등)가 WorkflowMap에 없을 수 있음 — 공통 기능 추가 전 상속 계층 확인

## 근거 파일
서비스 루트 = `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.apc_mgmt/src/main/java/hfr/plugin/my5g/service/apc_mgmt/`
- `ServiceStart_apc_mgmt.java`, `function/RestInvokerFunction_apc_mgmt.java`, `worker/Worker_apc_mgmt.java`, `common/data/apc_mgmtCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.apc_mgmt/home/config/apc_mgmt.xml` (포트 10463)
- `<BACKEND_ROOT>/uNMS_Common/uEMS.Common/src/main/java/hfr/common/abc/AbsRestInvokerFunction_Service.java`
