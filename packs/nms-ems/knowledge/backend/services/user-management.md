# User Management Service

## 서비스 책임
Keycloak 기반 사용자/역할/세션 관리, 임시계정 만료 처리, Keycloak DB 백업/복원, Subscriber 알림 브로커 역할을 담당한다.
자체 RDB 없이 모든 사용자/역할 데이터를 `KeycloakAdminProcess`(HTTP REST)로 Keycloak에서 직접 읽고 쓴다.

## 주요 입력
- user/user_role/user_session/Subscriber REST request
- Keycloak admin/client REST 응답
- user_backup/user_restore request
- keepalive/expired scheduler fire

## 주요 출력
- user/role/session 응답, Keycloak user/role 변경 결과
- Keycloak DB 백업 파일(`/data/dbbackup/keycloak/`), 복원 상태
- 임시/일반 계정 만료 처리 결과(비활성화/삭제)

## 시작점과 실행 설정
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.UserManagement/src/main/java/hfr/plugin/my5g/service/UserManagement/ServiceStart_UserManagement.java`
- `<BACKEND_ROOT>/product/my5G/Project_start_usermanagement/src/main/java/hfr/application/ServerStart_all.java`

확인 기준:
- `tServiceName = "usermanagement"`
- `System.getProperty("SYSTEM_TYPE")`으로 NOC/uEMS 분기
- 공통: `addManualLoadWorker(new manualWorkerLoader_UserManagement())`
- product(ServerStart_all): `addManualLoadWorker(new manualWorkerLoaderEx_UserManagement())` — `WorkerRestContentManager_UserMGMT` 추가 등록
- `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`

## 아키타입: D + C 하이브리드
`../feature-extension.md` 기준 D(manualWorkerLoader)로 분류되지만, 실제 ②③ 등록은 두 메커니즘이 나뉜다.

- **C(.inf 로더)**: 서비스 고유 클래스(`Worker_UserManagement`, `ScheduleKeepAlive`, `RestInvokerFunction_UserManagement`)는 `home/jar/usermanagement.inf`의 FQCN 목록으로 자동 로드된다. 새 Worker/Schedule을 만들고 .inf에 한 줄을 안 넣으면 로딩되지 않는다.
  - 근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.UserManagement/home/jar/usermanagement.inf`
- **D(manualWorkerLoader)**: `manualWorkerLoader_UserManagement.processManualWorkerLoad()`는 framework 공통 worker 4종(`Worker4ExecuteRestAPI`, `Worker4OperationLog`, `WorkerLockManager`, `Worker4ProcessFunction`)만 등록한다. product 확장은 `manualWorkerLoaderEx_UserManagement`(⑤)에서 `super` 호출 후 `aWorkflowMap.setWorker(...)` 추가.

## REST contract
근거: `RestInvokerFunction_UserManagement.getContentList()` (라인 1094~)

| request type | 비고 |
| --- | --- |
| `user` | CRUD + 만료 정책 대상 |
| `user_role` | Keycloak role mapping |
| `user_session` | Delete는 **sessionId 기반**(동일 user 복수 세션 개별 처리) |
| `Subscriber` | `ScopeTypeBook.All`, 알림 브로커 |
| `user_backup` | InstantBackup → `Worker_UserManagement.execute_UsereBackup()` |
| `user_restore` | `ScopeTypeBook.All`, K8s pod 명령(`KubernetesHelper.exec`) 사용 |
| `system-config` | `System_config.requestType = "system-config"` (`ScopeTypeBook.Inner`). "System_config"가 아니라 소문자-하이픈 문자열임 |

## 주요 Worker/Function 흐름
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.UserManagement/src/main/java/hfr/plugin/my5g/service/UserManagement/function/RestInvokerFunction_UserManagement.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.UserManagement/src/main/java/hfr/plugin/my5g/service/UserManagement/worker/Worker_UserManagement.java`

- `RestInvokerFunction_UserManagement.initialized()`에서 `m_WorkflowMap.getWorker(Worker_UserManagement.class)` 조회, `KeycloakAdminProcess` 및 `ClientInfoMap` 초기화.
- `processMicroServiceRestRequest()` if-else 체인이 request type별 처리 분기.
- `Worker_UserManagement.execute_UsereBackup(name)`이 실제 즉시 백업 담당(`MFile.copyFile`로 `/data/dbbackup/keycloak/`에 저장).

### Keycloak client 매핑 (ClientInfoMap)
`UserManagementCD.keycloakClientBook` 9개 client가 `RestInvokerFunction_UserManagement.initialized()`에서 인덱스로 등록된다:
`0=user-management, 1=bizPlatform, 2=uEMS, 3=SOC, 4=clientPortal, 5=helpDesk, 6=NOC, 7=arGuide, 8=digitalTwin`
인덱스가 하드코딩이므로 새 client 추가 시 충돌 주의.
- 근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.UserManagement/src/main/java/hfr/plugin/my5g/service/UserManagement/common/data/UserManagementCD.java`

## DB/저장소 흐름
- **MariaDB/MongoDB mapper·DAO 없음.** `CommonMybatisDBConfiguration`이 주입되지만 이 서비스에 mapper 클래스가 없다. 유일한 데이터 저장소는 Keycloak(H2, REST 경유)이다.
- 백업 설정: `/data/keycloak/user-backup-config.json` (JSON 파일)
- 백업 파일: `/data/dbbackup/keycloak/` (`MFile.copyFile`)
- restore: `KubernetesHelper.exec`로 pod 직접 명령 — 클러스터 권한 의존.

## scheduler/background 흐름
| 클래스 | 방식 | 상태 |
| --- | --- | --- |
| `ScheduleKeepAlive` | `ScheduleAdapterRat` (.inf 등록) | 동작. session keepalive |
| `UserExpiredSchedule` | **`java.util.Timer` 기반 standalone** (Quartz 아님). `RestInvokerFunction_UserManagement`에서 직접 `new`. 1시간 주기(`tPeriod = 1000*60*60`) | 동작 |
| `KeyCloakUserBackUpSchedule` | Quartz형이나 **미등록 + execute() 본문 주석 = no-op** | 죽은 코드 (아래 참고) |

`UserExpiredSchedule` 만료 정책(하드코딩 상수):
- 임시계정: 만료 7일 초과 시 삭제
- 일반계정: `NormalUserDisableDay=30`일 비활성화, `NormalUserDeleteDay=60`일 삭제
- `ReentrantLock` 보호 — Keycloak 응답 지연 시 1시간 주기 lock 점유 가능.

## 기능 확장점 (feature seams)
| seam | 추가 방법 | 등록 위치 / 근거 파일 |
| --- | --- | --- |
| ① REST request type | `getContentList()`에 `new RestContent(this, "<type>", HttpMethodBook.POST, getServiceName(), "v1")` 추가 + `processMicroServiceRestRequest()` if-else 분기 + `MicroServiceCommonCD.RequestTypeStringBook` 상수 | `function/RestInvokerFunction_UserManagement.java` |
| ② Worker/Function | `abstractWorker` 상속 클래스를 `worker/`에 작성 후 **`home/jar/usermanagement.inf`에 FQCN 한 줄 추가**(현행 등록 방식). 조회는 `m_WorkflowMap.getWorker(NewWorker.class)` | `home/jar/usermanagement.inf` |
| ③ Scheduler job | `ScheduleAdapterRat` 상속 + `isAutoRun()/getInterval()/execute()` 구현 후 `.inf`에 FQCN 추가(`ScheduleKeepAlive` 패턴). Timer식 standalone이면 `UserExpiredSchedule` 패턴으로 Function에서 직접 기동 | `schedule/ScheduleKeepAlive.java`, `schedule/UserExpiredSchedule.java` |
| ⑤ product override | `manualWorkerLoaderEx_UserManagement`에서 `super.processManualWorkerLoad()` 후 product 전용 Worker `setWorker()` — 공통영역 무수정 | `product/my5G/Project_start_usermanagement/.../manualWorkerLoaderEx_UserManagement.java` |
| 외부연동 adapter | `WorkerRestContentManager` 상속 + `getURL()` override로 호출 대상 URL 재정의, Ex loader에서 등록 | `product/my5G/Project_start_usermanagement/.../WorkerRestContentManager_UserMGMT.java` |
| data-model (Keycloak client) | `UserManagementCD.keycloakClientBook`에 client ID 상수 추가 + `RestInvokerFunction_UserManagement.initialized()`의 `ClientInfoMap.put(<새 인덱스>, client)` 추가 | `common/data/UserManagementCD.java` |

> ④ DB mapper seam은 이 서비스에 해당 없음(RDB 미사용).

## product 확장점
`<BACKEND_ROOT>/product/my5G/Project_start_usermanagement/`에 두 클래스:
1. `manualWorkerLoaderEx_UserManagement` — 공통 loader 상속, `WorkerRestContentManager_UserMGMT` 추가 등록. `ServerStart_all`이 이 Ex 클래스를 사용.
2. `WorkerRestContentManager_UserMGMT` — `getURL()` override. usermanagement 이외 서비스 호출 시 NOC는 `noc-backend.noc.svc.cluster.local:10410`, uEMS는 `uems-backend.uems.svc.cluster.local:10410`으로 라우팅. debugMode 시 `127.0.0.1:10410`(uEMS) / `127.0.0.1:10510`(NOC).

debug 시작점: `ServerStart_debug_my5G.java`, `ServerStart_debug_NOC.java` (같은 디렉터리).

## 죽은/미구현 코드
- `KeyCloakUserBackUpSchedule.execute()` — 파일 복사 본문 **전부 주석 처리 = no-op**. 게다가 `usermanagement.inf`에도 미등록이라 `Worker_UserManagement.initialize()`의 `getScheduleJob(KeyCloakUserBackUpSchedule.class)`는 null. 스케줄 백업을 살리려면 (1) .inf 등록 + (2) 주석 해제가 모두 필요하며, 즉시백업 경로(`execute_UsereBackup`)와의 파일 덮어쓰기 경쟁을 먼저 검토한다.
- 실제 백업은 `user_backup`/InstantBackup → `Worker_UserManagement.execute_UsereBackup()` 경로만 동작한다.

## mock/fake 지점
1. `KeycloakAdminProcess` — Keycloak HTTP REST mock
2. `WorkerRestContentManager.executeRestAPI` — system-config 조회 mock
3. `KubernetesHelper.exec` — restore 명령 mock
4. `MFile.copyFile` / `JSON.fileToObject` — 파일 I/O mock
5. `UserExpiredSchedule.getSecurityPolicy()` 반환값 mock

## fixture 기준
- user/role/session 초기 상태, Keycloak mock 응답
- token expiry/refresh 상태, multi-session case(sessionId별)
- user backup 설정 JSON·백업 파일 mock
- keepalive/expired scheduler case(임시 7일, 일반 30/60일 경계)

## 변경 위험
- Keycloak role mapping 오류 시 전체 사용자 접근 불가
- `keycloakClientBook` client 추가 시 `ClientInfoMap` 인덱스(0~8 하드코딩) 충돌
- `KeyCloakUserBackUpSchedule` 복구 시 즉시백업과 파일 덮어쓰기 경쟁 조건
- `UserExpiredSchedule` lock 점유(Keycloak 지연 시) 및 만료 정책 상수 하드코딩(`NormalUserDisableDay=30`, `NormalUserDeleteDay=60`)
- `user_backup`/`user_restore`의 K8s pod 직접 명령 — 클러스터 권한 의존
- credential/token이 `blackBox.info`로 일부 출력될 수 있음(개인정보 노출)
- multi-session 동시 로그아웃: userSessionMap이 sessionId 키 기반이므로 동일 user 복수 세션이 모두 노출됨

## 근거 파일
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.UserManagement/src/main/java/hfr/plugin/my5g/service/UserManagement/ServiceStart_UserManagement.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.UserManagement/src/main/java/hfr/plugin/my5g/service/UserManagement/function/RestInvokerFunction_UserManagement.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.UserManagement/src/main/java/hfr/plugin/my5g/service/UserManagement/common/WorkerLoader/manualWorkerLoader_UserManagement.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.UserManagement/src/main/java/hfr/plugin/my5g/service/UserManagement/worker/Worker_UserManagement.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.UserManagement/src/main/java/hfr/plugin/my5g/service/UserManagement/schedule/KeyCloakUserBackUpSchedule.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.UserManagement/src/main/java/hfr/plugin/my5g/service/UserManagement/schedule/UserExpiredSchedule.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.UserManagement/src/main/java/hfr/plugin/my5g/service/UserManagement/schedule/ScheduleKeepAlive.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.UserManagement/src/main/java/hfr/plugin/my5g/service/UserManagement/common/data/UserManagementCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.UserManagement/home/jar/usermanagement.inf`
- `<BACKEND_ROOT>/product/my5G/Project_start_usermanagement/src/main/java/hfr/application/my5g/usermgmt/manualWorkerLoaderEx_UserManagement.java`
- `<BACKEND_ROOT>/product/my5G/Project_start_usermanagement/src/main/java/hfr/application/my5g/usermgmt/WorkerRestContentManager_UserMGMT.java`
- `<BACKEND_ROOT>/product/my5G/Project_start_usermanagement/src/main/java/hfr/application/ServerStart_all.java`
