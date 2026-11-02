# Recovery Service

## 서비스 책임
NE(Network Element) 교체/복구 코디네이터. 단일 REST endpoint `change-ne`로 NE DB 교체, 설정 파일 복원(+선택적 자동 재시작), 재시작 확인 요청을 받아 직접 처리하지 않고 하위 서비스(nebackuprestore_mgmt, ne_configuration, vRAN, product EMS 서비스)에 위임한다.

## 주요 입력
- `change-ne` POST 요청 — 내부 `type` 필드로 sub-type 분기 (`change-ne-db` / `change-ne-restore` / `check-restart`)
- 요청 출처 2종: uNMS(NOC) 표준 envelope / AR Guide raw JSON (`isUnms` 플래그로 응답 포맷 분기)

## 주요 출력
- 위임 서비스 응답의 재포장(`httpJsonResponseData`), 실패 시 `errorResultUnms()`의 HTTP 418 + error_code JSON

## 시작점과 실행 설정
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Recovery/src/main/java/hfr/plugin/my5g/service/Recovery/ServiceStart_Recovery.java`
- `tServiceName = "recovery"`, `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- 별도 ServerConfig 클래스 없음 — MyBatis 설정은 `CommonMybatisDBConfiguration` 공통 주입, 서버 구성은 `runMicroService` 내부 자동
- MongoDB 미사용, 자체 mapper XML 없음

## 아키타입
**B(코디네이터/디스패처)** — `../feature-extension.md` 참조. 직접 DB CRUD 없이 `m_WorkerRestContentManager.executeRestAPI(...)`로 위임한다.
- ② Worker 등록: `Worker_Recovery`(abstractWorker 상속)가 프레임워크 plugin 로딩으로 WorkflowMap에 등재되고, `RestInvokerFunction_Recovery.initialize()`에서 `m_WorkflowMap.getWorker(Worker_Recovery.class)`로 획득
- ③ Scheduler: 없음
- 공통 레지스트리: `AutoRestContentBook.java`에 `"recovery"` 서비스와 `change-ne`/`received-notification` RestContent가 자동 생성 목록으로 등재 (L101-104)

## REST contract
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Recovery/src/main/java/hfr/plugin/my5g/service/Recovery/function/RestInvokerFunction_Recovery.java`
- `change-ne` (POST, v1) — `getContentList()`에 단독 등록 (L407). 상수는 `MicroServiceCommonCD.RequestTypeStringBook.Change_ne`
- request type은 하나지만 body의 `type` 필드로 3개 sub-type이 중첩 분기된다 (`processMicroServiceRestRequest()` L139~):

| sub-type (`type` 필드) | 처리 | 위임 대상 |
| --- | --- | --- |
| `Change_ne_db` (change-ne-db) | old/new NE 조회 후 `m_WorkerLocalBaseData.changeNe(oldNe, tNe, true)` — 유일한 로컬(공통 Worker) 처리 (L179-196) | 없음 (공통 BaseData Worker) |
| `Change_ne_restore` | 백업파일 복원 위임 → 성공 시 `NeCodeBook.valueOf(model).getAutoRestartConfigRestore()`가 false인 모델만 `Ne_restart` 추가 위임 (L198-295) | `SupportService.nebackuprestore_mgmt` → `SupportService.ne_configuration` |
| `Check_restart` | `NeRebootCheckRequest` 생성 후 재시작 상태 조회 위임 (L296-358) | `EmsServiceCode.vRAN.getServiceName()` (SupportService 아님, L304) |
| 그 외 | fallthrough: NE의 productcode로 product EMS 서비스에 원본 요청 그대로 위임 (L360-397) | `productCD.ProductTypeCode.valueOf(productcode).getServiceName()` |

sub-type 상수(`Change_ne_db` 등)는 `MicroServiceCommonCD.RequestTypeStringBook`이 상속하는 상위 `platformOxCD.RequestTypeStringBook`에 정의되어 있다.

## 주요 Worker/Function 흐름
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Recovery/src/main/java/hfr/plugin/my5g/service/Recovery/worker/Worker_Recovery.java`
- `Worker_Recovery` — 비즈니스 로직 없는 헬퍼 Worker:
  - `getRecoveryRequest()` / `getNeConfigurationRequest()`: 위임용 요청 객체 빌더 (WorkType=Modify 고정)
  - `errorResultUnms(errorCode, data)`: HTTP 418 + `{style:"noc", teapot:{error_code, data}}` 에러 응답 생성
  - `checkAutoRestart(Ne)`: **호출부 없음(죽은 코드)** — 아래 §죽은 코드 참조
- `RestInvokerFunction_Recovery extends AbsRestInvokerFunction_my5GService implements INeComfailStatusProvider`:
  - `getComfailStatus(neId)`: NE 조회 후 `getState()`만으로 Success/Failed 판정 (L117-126)
  - `vaildCheckRequest()`는 무조건 Success 반환 — 요청 유효성 검사 없음

## DB/Mongo/파일 흐름
직접 DB 접근 없음. NE 조회(`m_WorkerLocalBaseData.requestNeInfo()`)와 NE 교체(`changeNe()`)는 부모/공통 Worker(`WorkerLocalBaseDataCommon`) 경유. `RecoveryCD.java`는 WorkTypeBook 상수가 전부 주석 처리된 빈 클래스.

## scheduler 흐름
없음.

## 기능 확장점 (feature seams)
| seam 유형 | 추가 방법 | 등록 위치/근거 파일 |
| --- | --- | --- |
| ① rest-request-type | `getContentList()`에 `new RestContent(this, "새-type", HttpMethodBook.POST, getServiceName(), "v1")` 추가 + `processMicroServiceRestRequest()`에 `aRequestType` 분기 추가 + `MicroServiceCommonCD.RequestTypeStringBook`(또는 상위 platformOxCD)에 상수 선언 | `function/RestInvokerFunction_Recovery.java` (L139, L404-408) |
| ① sub-type (권장 대안) | 신규 복구 동작은 새 request type보다 `change-ne` 내부 `type` 분기 추가가 기존 패턴 — `Change_ne_db`/`Change_ne_restore`/`Check_restart` else-if 체인에 분기 추가. 미매칭 type은 product 서비스 fallthrough로 흐르므로 분기 위치 주의 | `function/RestInvokerFunction_Recovery.java` (L179-358) |
| ② worker-function | `Worker_Recovery`에 public 메서드 추가(빌더/에러응답 패턴 참고), 또는 `abstractWorker` 상속 신규 Worker 생성 후 `WorkflowMap.getWorker()`로 획득 | `worker/Worker_Recovery.java` (L23-90) |
| external-adapter (B형 핵심) | 새 하위 서비스 위임 = `m_WorkerRestContentManager.executeRestAPI(this.getClass(), SupportService.<svc>, requestType, json)` 호출을 분기 블록에 삽입. 서비스명 상수는 `MicroServiceCommonCD.SupportService`에 추가 (EMS 위임은 `EmsServiceCode` 사용) | `function/RestInvokerFunction_Recovery.java` (L214, L249, L304) / `uEMS.plugin.my5g.Common/.../data/MicroServiceCommonCD.java` (L18, L26) |
| data-model | 요청/응답 필드 확장은 `RecoveryRequest`/`RecoveryResponse`(`hfr.plugin.my5g.common.communicator`)에 추가. 위임 대상 서비스(nebackuprestore_mgmt 등)의 파싱과 함께 변경할 것 | `uEMS.plugin.my5g.Common/.../communicator/` |
| ⑤ NE 모델별 자동재시작 | restore 후 자동 재시작 여부는 `NeCodeBook.valueOf(model).getAutoRestartConfigRestore()`로 결정 — 새 NE 모델은 `NeCodeBook` enum의 해당 속성으로 제어(이 서비스 코드 수정 불필요) | `function/RestInvokerFunction_Recovery.java` (L237-239) |

## product 확장점
product별 override 구조 없음 — `RestInvokerFunction_Recovery`는 `AbsRestInvokerFunction_my5GService` 직접 상속이고 product 폴더에 Recovery 구현체가 없다. product 차이는 (1) fallthrough 분기의 productcode 기반 위임, (2) `NeCodeBook.getAutoRestartConfigRestore()` 모델 속성으로만 반영된다.

## 죽은/미구현 코드
- `NeRebootCheckProcess` — 필드/initialize/dispose 전부 주석 처리 (L57, L86-92, L96-102). 재부팅 체크 프로세스 비활성. `Check_restart`의 로컬 처리 경로(L305)도 주석이고 실제로는 vRAN 위임으로 대체됨. 활성화하려면 initialize/dispose 흐름 복원 필요
- `Worker_Recovery.checkAutoRestart(Ne)` — MY5GCUDU/MY5GT1/MY5GT2/MY5GBOX_T2_OD 하드코딩 판별 메서드지만 **호출부가 전혀 없다**. 실제 판별은 `NeCodeBook.getAutoRestartConfigRestore()`. 이 메서드를 수정해도 동작에 영향 없음
- `RecoveryCD.WorkTypeBook` — 상수 전부 주석, 참조하는 곳 없음. 여기에 상수를 추가해도 아무 일도 일어나지 않는다

## mock/fake 지점
1. `m_WorkerRestContentManager.executeRestAPI(...)` — 외부 위임 3종(nebackuprestore_mgmt / ne_configuration / vRAN / product 서비스) mock 대상
2. `m_WorkerLocalBaseData.requestNeInfo()` / `changeNe()` — NE 조회·교체 mock 대상
3. `Worker_Recovery.errorResultUnms()` — 순수 함수, 단위 테스트 가능

## 변경 위험
- `change-ne` 단일 type 안에 3개 sub-type + fallthrough가 중첩 — 신규 sub-type을 else-if 체인 뒤에 넣으면 의도치 않게 product fallthrough를 가로채거나 놓칠 수 있음
- fallthrough 경로에서 AR Guide 요청(isUnms=false)은 `tRecoveryRequest`가 null → `tRecoveryRequest.getRequest_type()` NPE → catch에서 rollback + "Request type not found" 반환 (L360-389). 이 경로 수정 시 null 가드 확인
- `vaildCheckRequest()`가 항상 Success — 잘못된 요청도 분기 내부까지 진입
- restore 성공 후 재시작 위임 사이 `Mthread.sleep(2000)` 고정 지연 — timeout 튜닝 시 주의 (`restart_timeout: 7` 하드코딩 응답 포함, L211)
- `NeRebootCheckProcess` 비활성으로 `getComfailStatus()`가 NE 연결 상태(`getState()`)에만 의존
- 새 NE 모델 추가 시 `NeCodeBook.getAutoRestartConfigRestore()` 속성 누락하면 restore 후 자동 재시작 동작이 의도와 달라짐

## 근거 파일
서비스 루트 = `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Recovery/src/main/java/hfr/plugin/my5g/service/Recovery/`
- `ServiceStart_Recovery.java`
- `function/RestInvokerFunction_Recovery.java`
- `worker/Worker_Recovery.java`
- `common/data/RecoveryCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/function/AbsRestInvokerFunction_my5GService.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/autogenerate/AutoRestContentBook.java`
