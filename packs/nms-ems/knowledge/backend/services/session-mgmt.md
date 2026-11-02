# Session_mgmt Service

## 서비스 책임
`my5g.Service.Session_mgmt`는 마이크로서비스 생태계의 게이트키퍼다. 사용자/서비스/EMS/NMS 간 인증(Authentication), 세션·토큰 관리, WebSocket 연결 라우팅, Notification 전달을 담당한다.

- 주요 입력: 서비스 인증 요청(`Service_Authentication`), EMS/NMS 등록·인증 요청, 사용자 인증 검증(`Check_authentication`, `Check_Keycloak_authentication`), 타 서비스가 보내는 `Notification_Request*`
- 주요 출력: 인증 토큰(서비스/EMS/NMS별), WebSocket으로의 알림 push(`sendNotifyToClient`), 세션/서비스 목록 응답, basedata 서비스로의 등록 데이터 위임 호출

## 시작점과 실행 설정
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Session_mgmt/src/main/java/hfr/plugin/my5g/service/Session_mgmt/ServiceStart_Session_mgmt.java`

확인 기준:
- `tServiceName = "session_mgmt"`
- `tserverRunningConfig.setSessionManagerClass(WorkerRemoteSessionManager.class)` — 세션 매니저를 프레임워크에 직접 주입 (line 39)
- `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- Worker/Schedule/Function 로딩은 `home/jar/session_mgmt.inf`의 FQCN 목록으로 선언

## 아키타입
**C (.inf 플러그인 로더)** — `../feature-extension.md` 참조.

②(Worker)③(Scheduler) 등록 위치 = `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Session_mgmt/home/jar/session_mgmt.inf`. 클래스를 만들어도 이 파일에 FQCN 한 줄을 추가하지 않으면 로딩되지 않는다.

`.inf`에 선언된 서비스 자체 클래스:
- `function.RestInvokerFunction_Session_mgmt`
- `worker.WorkerRemoteSessionManager`, `worker.WorkerProcessRequestRest`
- `worker.WorkerKeepAliveChecker`, `worker.Worker_Session_mgmt`, `schedule.ScheduleKeepAlive` — **3개 모두 src 트리에 소스 없음** (아래 죽은/미구현 코드 참조)
- 공통: `WorkerLocalRestSessionManager`, `WorkerRestContentManager`, `WorkerCommonNotifier`, `WorkerLocalBaseData_my5G` 등

## REST contract
근거: `RestInvokerFunction_Session_mgmt.getContentList()` (line 117~162). 상수는 `MicroServiceCommonCD.RequestTypeStringBook`.

| 그룹 | request type |
| --- | --- |
| 연결/서비스 인증 | `Connect`, `Service_Authentication`, `Service_query_servicelist`, `Update_Service_Authentication` |
| 사용자 | `request_UserLogin_Data`, `request_User_AuthorizedGroup`, `Check_authentication`, `Check_Keycloak_authentication` |
| NMS | `NMS_Authentication`, `NMS_exchange_contents`, `NMS_registration` |
| EMS | `Ems_registration`, `Ems_query_registration`, `Ems_Authentication`, `Ems_exchange_contents`, `Ems_Authentication_Sync`, `Ems_query_AccessToken` |
| 알림 | `Notification_Request`, `Notification_Request_Site`, `Notification_Request_Site_All` |
| 운영 | `Keep_alive`(GET), `sendRequestWebSocket`, `request_License`, `request_service_update`, `request_not_service_update`, `System_config`(requestType), `Log_level`(GET) |

내부 전용 API는 `ScopeTypeBook.Inner`로 선언되어 있다. 신규 타입 추가 시 기존 항목의 Scope/HttpMethod 패턴을 따른다.

## 주요 Worker/Function 흐름
근거 디렉터리: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Session_mgmt/src/main/java/hfr/plugin/my5g/service/Session_mgmt/`

- `RestInvokerFunction_Session_mgmt` — 진입점. `processMicroServiceRestRequest()`/`processRestRequest()`의 if-else 분기로 request type을 worker에 매핑. `initialize()`에서 `aWorkFlowMap.getWorker(...)`로 worker 참조 획득 (line 179~196)
- `WorkerRemoteSessionManager` (약 2,400줄, 핵심 싱글턴) — 사용자 세션/서비스 세션/EMS 세션 3종을 HashMap으로 관리. `processService_Authentication()` 토큰 발급, `localAuthentication()`(basedata REST 호출), WebSocket 세션 관리(`AbsWorkerWebSocketSessionManager` 상속), 알림 push와 `checkNotifySend()` 필터 (line 476, 500)
- `WorkerNOCSessionManager` — NMS 등록/인증 흐름. `processRequest_Nms_*` 메서드가 `m_WorkerLocalBaseData.processBaseDataGetRequest()`로 basedata 서비스에 위임
- `WorkerProcessRequestRest` — EMS 등록 처리(`processEms_Registration()`). `m_WorkerRestContentManager.executeRestAPI()`로 basedata에 위임
- `UserAuthorityStateManager` / `UserAuthorityState` / `TargetGroupState` / `TargetGroupRoleState` — 사용자·그룹 권한 상태 캐시. 알림 전송 시 `E_NotifySendCheckType` 기반 권한 필터링에 사용

### Notification 전달 흐름
`Notification_Request*` 수신 → `WorkerRemoteSessionManager`가 접속 세션 순회 → `checkNotifySend(InnerUserData, AbsNotifySendCheckData)`로 권한 필터 → 통과 세션에만 `sendNotifyToClient` WebSocket push.

수신 알림(타 서비스 발행 구독)은 `getReceive_NotificationTypes()`(line 915) 목록 + `received_NotificationData()`(line 924) 분기로 처리한다.

## DB/Mongo/파일 흐름
- DB 주입은 `CommonMybatisDBConfiguration`(MySQL/PostgreSQL 공용)이지만 **서비스 자체 mapper/xml 없음**
- 세션·인증 데이터는 메모리(HashMap) + 파일 기반 Properties `cn_data.pro`로 영속화. `WorkerRemoteSessionManager.loadConfig_DB()`(line 303)가 POD 재시작 시 `DataPath.getNonVolatile_Path` 하위 `cn_data.pro`에서 서비스 인증 정보를 복구
- EMS/NMS 등록 데이터의 실제 저장은 REST로 basedata 서비스에 위임 (`executeRestAPI`, `processBaseDataGetRequest`)
- Mongo 사용 없음

## scheduler 흐름
`.inf`에 `schedule.ScheduleKeepAlive`가 선언되어 있으나 **src 트리에 소스가 없다**(JAR/빌드 산물 의존 추정). Keep-alive 확인은 `Keep_alive` GET request type으로도 노출된다. 스케줄 로직 수정 요청을 받으면 먼저 실제 클래스 소스 위치(빌드 산물 포함)를 확인해야 한다.

## 기능 확장점 (feature seams)

| seam 유형 | 추가 방법 | 등록 위치 / 근거 파일 |
| --- | --- | --- |
| ① REST request type | `getContentList()`에 `new RestContent(this, <type>, HttpMethodBook.POST, getServiceName(), "v1", ScopeTypeBook.Inner)` 한 줄 + `processMicroServiceRestRequest()`/`processRestRequest()` else-if 분기 + `MicroServiceCommonCD.RequestTypeStringBook`에 상수 | `function/RestInvokerFunction_Session_mgmt.java` (line 117~162, 432~885) |
| ② Worker | `worker` 패키지에 `abstractWorker` 상속 클래스 작성 → `.inf`에 FQCN 한 줄 추가 → `RestInvokerFunction_Session_mgmt.initialize()`에서 `aWorkFlowMap.getWorker(NewWorker.class)` 참조 | `home/jar/session_mgmt.inf`; `RestInvokerFunction_Session_mgmt.java` (line 179~196) |
| ③ Scheduler | `schedule` 패키지에 `ScheduleAdapterRat` 상속 클래스(`isAutoRun()=true`, `isNextRunAfterComplete()=true` 패턴) → `.inf`에 FQCN 등록 | `home/jar/session_mgmt.inf` |
| NMS/EMS 인증 로직 | `WorkerNOCSessionManager`에 `processRequest_*` 메서드 추가 → `processRestRequest()`의 해당 type 분기에서 호출 | `worker/WorkerNOCSessionManager.java` (line 70~170) |
| WebSocket 세션/알림 필터 | 세션 관리 확장은 `WorkerRemoteSessionManager`에 메서드 추가. 알림 필터 확장은 `checkNotifySend()` + `UserAuthorityStateManager`에 새 `E_NotifySendCheckType` 분기 | `worker/WorkerRemoteSessionManager.java` (line 500~522); `worker/UserAuthorityStateManager.java` |
| 수신 알림 타입 | `getReceive_NotificationTypes()`에 타입 문자열 추가 + `received_NotificationData()`에 처리 분기 추가 | `RestInvokerFunction_Session_mgmt.java` (line 914~967) |
| 데이터 모델 | 서비스 전용은 `common/data/`에 추가. 공통 모델(Ems_registration, User, Role 등)은 `uEMS.plugin.my5g.Common`의 unify_base/unify_system 하위 — 공통 모듈 변경이 곧 이 서비스에 반영됨 | `common/data/` (Data_EmsRegistration 등) |

## product 확장점
product별 override 구조 없음. `RestInvokerFunction_Session_mgmt`는 `AbsRestInvokerFunction_my5GService` 단일 상속, `WorkerRemoteSessionManager`는 `AbsWorkerWebSocketSessionManager` 상속. my5G/openhaul/sixgen-ems product 폴더에 동명 서비스 없음.

단, `WorkerRemoteSessionManager`는 여러 product start(NOC, my5G, uCM, NDAS, LicensePortal)에서 `setSessionManagerClass`로 공용 참조된다 — 이 클래스 수정은 전 product에 영향.

## 죽은/미구현 코드
- `.inf` 선언 3개 클래스 소스 부재: `schedule.ScheduleKeepAlive`, `worker.WorkerKeepAliveChecker`, `worker.Worker_Session_mgmt` — src 트리에 없음. JAR/빌드 산물 의존 가능성. 관련 수정 전 실제 소스 위치 확인 필수
- `worker/` 디렉터리 안에 `session_mgmt.inf` 사본이 하나 더 있다 — 실제 로딩 대상은 `home/jar/session_mgmt.inf`이므로 혼동 주의

## mock/fake 지점
- `WorkerRemoteSessionManager.processService_Authentication()` — 토큰 HashMap 상태 mocking
- `WorkerRemoteSessionManager.localAuthentication()` — basedata REST 호출 mock
- `WorkerNOCSessionManager.processRequest_Nms_Authentication()` — `processBaseDataGetRequest()` mock
- `WorkerProcessRequestRest.processEms_Registration()` — `executeRestAPI()` mock (basedata 의존)
- `RestInvokerFunction_Session_mgmt.checkLicense()` — `ClusterLicenseToolProcess` 외부 의존, `isDebugMode=true`로 우회 가능

## 변경 위험
- `WorkerRemoteSessionManager`는 서비스 간 인증 토큰 HashMap을 메모리에서 직접 관리하는 핵심 싱글턴 — 수정 시 전체 마이크로서비스 인증 흐름에 영향
- EMS 세션 map(`m_EmsAccessToken4IDMap`, `m_EmsAccessToken4ServiceNameMap`)은 ReentrantLock 보호 — 다수 EMS 동시 접속 시 lock 경합 가능
- `Check_authentication`은 body/header 이중 지원(fallback) — 신규 인증 방식 추가 시 fallback 우선순위 충돌 주의
- `cn_data.pro` 복구 로직(`loadConfig_DB`)은 POD 재시작 시 동작 — `DataPath.getNonVolatile_Path` 설정 오류 시 서비스 인증 전체 손실
- `.inf` 소스 부재 3개 클래스 관련 변경은 빌드 구성 확인 전까지 보류

## 근거 파일
`<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Session_mgmt/` 기준:
- `src/main/java/hfr/plugin/my5g/service/Session_mgmt/ServiceStart_Session_mgmt.java`
- `src/main/java/hfr/plugin/my5g/service/Session_mgmt/function/RestInvokerFunction_Session_mgmt.java`
- `src/main/java/hfr/plugin/my5g/service/Session_mgmt/worker/WorkerRemoteSessionManager.java`
- `src/main/java/hfr/plugin/my5g/service/Session_mgmt/worker/WorkerNOCSessionManager.java`
- `src/main/java/hfr/plugin/my5g/service/Session_mgmt/worker/WorkerProcessRequestRest.java`
- `src/main/java/hfr/plugin/my5g/service/Session_mgmt/worker/UserAuthorityStateManager.java`, `UserAuthorityState.java`
- `home/jar/session_mgmt.inf`

공통:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.Common/src/main/java/hfr/common/worker/WorkerLocalRestSessionManager.java`
