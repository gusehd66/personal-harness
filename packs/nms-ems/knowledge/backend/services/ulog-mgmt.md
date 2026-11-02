# ulog_mgmt Service

## 서비스 책임
운영 로그(Operation Log)를 MongoDB에 수집·조회·삭제하는 support 마이크로서비스. 로그 기록(Add), 필터 목록 조회(GetFilterList), 검색(Search), 건수 조회(TotalCount), 보관 기간 초과 데이터 Shrink를 담당한다.

## 주요 입력
- REST 요청: 로그 등록/검색/필터/건수 (`operation-log`), Shrink 요청 (`operationlog-shrink`)
- 타 서비스가 보내는 운영 로그 payload (`operation_Log` / `RestComunication` / `Model4OperationLog` logSourceType)

## 주요 출력
- MongoDB `operation_data` DB의 `operation_log` 컬렉션 document (insert / queryList / deleteMany)
- 검색·필터·건수 REST 응답

## 시작점과 실행 설정
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.ulog_mgmt/src/main/java/hfr/plugin/my5g/service/ulog_mgmt/ServiceStart_ulog_mgmt.java`
- `.../ulog_mgmt/ulogServerConfig.java`

확인 기준:
- `tServiceName = "ulog_mgmt"`, `setUseMongoDB4MicroService(true)`, `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- `ulogServerConfig extends ServerConfigOnlyDB`: `getServiceName()=SupportService.ulog_mgmt`, `useMongoDB()=true`
- MongoDB: host `mongo_db.{LOCAL_DOMAIN}`(운영) / `127.0.0.1`(디버그), port 27017, DB=`operation_data`, `setInitClassList(operation_log.class)`로 컬렉션 초기화

## 아키타입
**C(.inf 플러그인 로더)** — `../feature-extension.md` 참조.
- ②③ 등록 위치 = `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.ulog_mgmt/home/jar/ulog_mgmt.inf`에 FQCN 한 줄 추가. 이 한 줄을 빠뜨리면 클래스를 만들어도 로딩되지 않는다.
- 현재 .inf 등록: `ulogServerConfig`, `RestInvokerFunction_ulog_mgmt`, `Worker_ulog_mgmt` + my5G 공통 Workers(`WorkerLocalBaseData_my5G`, `WorkerCommonNotificationReceiver_my5G`, `WorkerCommonNotifier`, `WorkerCommonRequestHandler_my5g` 등)
- **`ScheduleShrinkOperationLogData`는 .inf에 미등록 = 비활성** (아래 "죽은/미구현 코드")

## REST contract
근거: `.../ulog_mgmt/function/RestInvokerFunction_ulog_mgmt.java` (getContentList / processMicroServiceRestRequest)

request type (`MicroServiceCommonCD.RequestTypeStringBook` 상수):
- `operation-log` (`operation_log`) — 로그 등록·검색·필터·건수. 내부에서 logSourceType(`operation_Log`/`RestComunication`/`Model4OperationLog`) switch 분기
- `operationlog-shrink` (`OperationLog_Shrink`) — 보관 기간 초과 데이터 삭제
- `alarm-definition` (`Alarm_Definition`), `service-status` (`Service_Status`) — 공통 처리 분기(super 공통 contract)

⚠ `RestComunication` case에 **break 누락(fall-through)** — RestComunication 요청이 Model4OperationLog 처리 경로로 흘러간다. 수정 요청 시 이 버그 존재를 전제로 확인할 것.

## 주요 Worker/Function 흐름
근거: `.../ulog_mgmt/worker/Worker_ulog_mgmt.java`, `.../ulog_mgmt/process/Operation_Log_Process.java`
- `RestInvokerFunction_ulog_mgmt.processMicroServiceRestRequest()` → logSourceType switch → `Worker_ulog_mgmt.processRestRequest*()` 호출
- `Worker_ulog_mgmt`(abstractWorker 직접 상속, 추상 계층 없음)가 `m_OperationProcess`(`Operation_Log_Process`)에 위임
- `Operation_Log_Process`가 MongodbDriver → `m_MongoDBClient`를 획득해 insert / queryList / queryListLimit / queryListCount / deleteMany 직접 호출
- workType은 `ulog_mgmtCD.WorkTypeBook` 기준(Add/Search/GetFilterList/TotalCount 등)
- `processRestRequestModel4OperationLog()`·`processRestRequestRestComunicationLog()`의 Search/GetFilterList/TotalCount 처리는 **주석 처리 상태** — 해당 workType은 빈 응답 반환

## DB/Mongo 흐름
- MongoDB 전용(Mybatis mapper 미사용). DB=`operation_data`, 컬렉션=`operation_log`
- `operation_log`(AbsMongoDBData) 인덱스(@MongoDataFieldOption): `module`, `function+operation`(복합), `ne_id`, `operation_date`(desc) — `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/mongodb/operation_log/operation_log.java`
- RDB fallback 경로(`driver4Mybatis`)는 골격만 존재(`Operation_Log_Process`의 else 블록, mjjoe::2024-06-20 주석)

## scheduler 흐름
- `ScheduleShrinkOperationLogData`(`.../ulog_mgmt/schedule/`) — `WorkerShrinkDataStorage.runShrink_OperationLog()`로 보관 기간 초과 로그 삭제. **구현됐으나 ulog_mgmt.inf 미등록으로 현재 돌지 않는다.**
- 활성화: `ulog_mgmt.inf`에 `hfr.plugin.my5g.service.ulog_mgmt.schedule.ScheduleShrinkOperationLogData`와 `hfr.plugin.my5g.common.worker.WorkerShrinkDataStorage` 각 한 줄 추가

## 기능 확장점 (feature seams)
| seam 유형 | 추가 방법 | 등록 위치/근거 파일 |
| --- | --- | --- |
| ① rest-request-type | `getContentList()`에 `new RestContent(this, RequestTypeStringBook.{새타입}, HttpMethodBook.POST, getServiceName(), "v1")` 추가 + `processMicroServiceRestRequest()` if-else 분기 추가. 상수는 `MicroServiceCommonCD.RequestTypeStringBook`에 선언 | `function/RestInvokerFunction_ulog_mgmt.java` |
| ② worker-function (아키타입 C) | 새 `E_LogSourceType` 값 추가 → `Worker_ulog_mgmt`에 `processRestRequest*()` 메서드 추가 → switch case 연결. 신규 Worker 클래스는 `ulog_mgmt.inf`에 FQCN 한 줄 추가해야 WorkflowMap 등록 | `worker/Worker_ulog_mgmt.java`, `home/jar/ulog_mgmt.inf` |
| ③ scheduler-job (아키타입 C) | `ScheduleAdapterRat` 상속 클래스 작성 → `ulog_mgmt.inf`에 FQCN 추가. 기존 `ScheduleShrinkOperationLogData` 활성화도 동일(위 scheduler 절) | `home/jar/ulog_mgmt.inf`, `schedule/ScheduleShrinkOperationLogData.java` |
| data-model (Mongo) | `operation_log` 필드 추가, 또는 새 AbsMongoDBData 상속 모델 작성 후 `ulogServerConfig.getMongoDBConfig()`의 `setInitClassList()`에 추가 → 컬렉션 자동 초기화 | `ulogServerConfig.java` |
| ④ db-mapper (RDB fallback) | `Operation_Log_Process.process_Insert_Operation_Log()` 및 `query*` 메서드의 else 블록에 Mybatis mapper 호출 구현 + mapper xml을 `CommonMybatisDBConfiguration`에 등록 | `process/Operation_Log_Process.java` |
| 변환 로직 완성 | `ConvertToLog_OperationLog()` 구현(현재 null 반환) + `process_Insert_Operation_Log()`의 Model4OperationLog case에서 `m_MongoDBClient.insert(tOperationLog)` 주석 해제 | `process/Operation_Log_Process.java` |

## product 확장점
없음. AbsFunction*/Worker* 추상 계층 없이 `Worker_ulog_mgmt` 단일 구현이며 서비스 로직에 product 분기가 없다. (.inf의 my5G 공통 Workers는 프레임워크 공통 기능일 뿐 product override 지점이 아니다.)

## 죽은/미구현 코드
- `ScheduleShrinkOperationLogData` — 구현됐으나 `.inf` 미등록 = 비활성
- `ConvertToLog_OperationLog()` — TODO 상태, null 반환. Model4OperationLog 타입 로그가 저장되지 않음
- `processRestRequestRestComunicationLog()` — Worker에 존재하나 switch fall-through로 실제 호출되지 않음
- RestComunication / Model4OperationLog 경로의 Search·GetFilterList·TotalCount — 주석 처리

## mock/fake 지점
1. `Operation_Log_Process.m_MongoDBClient` — insert/query*/deleteMany 전체 Mongo 접근 단일 지점
2. `Worker_ulog_mgmt.m_OperationProcess` — Worker 레이어 단위 테스트 시 Mock
3. `WorkerShrinkDataStorage.runShrink_OperationLog()` — 스케줄러 단위 테스트 시 Mock

## 변경 위험
- Shrink 스케줄러 비활성 → 운영 환경에서 `operation_log` 컬렉션 용량 무한 증가. 활성화 시에는 반대로 삭제 기준 설정 오류에 따른 대량 삭제 위험을 확인할 것
- RestComunication fall-through — 이 분기를 "수정"하면 현재 fall-through에 의존하던 동작이 바뀔 수 있으므로 호출 서비스 확인 필요
- workType(Search/GetFilterList/TotalCount) 주석 해제 시 UI 계약과 응답 형식 재확인 필요
- `.inf` 등록 누락 — 새 Worker/Schedule 클래스 추가 후 가장 흔한 실수

## 근거 파일
서비스 루트 = `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.ulog_mgmt/`
- `src/main/java/hfr/plugin/my5g/service/ulog_mgmt/ServiceStart_ulog_mgmt.java`, `ulogServerConfig.java`
- `function/RestInvokerFunction_ulog_mgmt.java`
- `worker/Worker_ulog_mgmt.java`, `process/Operation_Log_Process.java`
- `schedule/ScheduleShrinkOperationLogData.java`
- `home/jar/ulog_mgmt.inf`
- `common/data/ulog_mgmtCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/mongodb/operation_log/operation_log.java`
