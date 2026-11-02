# System Management Service

## 서비스 책임
SystemManagement Service는 시스템 백업/복원(uNMS·SMPlatform·vCore·Keycloak), NE Config 백업/복원, 시스템 업그레이드(ArgoCD 기반 workload별 Upgrade/Retry/Rollback), release info 관리, GitLab/ArgoCD/Kubernetes 외부 adapter 연동을 담당하는 운영 관리 서비스다.

## 주요 입력
- system/NE config backup·restore request
- upgrade request(`system_upgrade`) / 상태 조회(`upgrade_status`)
- release info 조회/파일 request
- GitLab/ArgoCD/Kubernetes 외부 adapter 응답
- operation log/lock 관련 request

## 주요 출력
- backup/restore 상태 (`E_RestoreStatus` 기반 메모리 상태)
- workload별 upgrade 상태
- release info / System_config
- Shell 명령 기반 DB restore 실행 결과
- ArgoCD patch/sync, GitLab API 호출 결과

## 시작점과 실행 설정
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SystemManagement/src/main/java/hfr/plugin/my5g/service/SystemManagement/ServiceStart_SystemManagement.java`

확인 기준:
- `tServiceName = MicroServiceCommonCD.SupportService.system_mgmt`
- `tserverRunningConfig.addManualLoadWorker(new manualWorkerLoader_SystemManagement())`
- `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- DB IP는 운영 시 `platformRatCD.SupportService.maria_db` + `LOCAL_DOMAIN` 환경변수로 결정된다.

## 아키타입과 등록 위치
`../feature-extension.md` 기준 **D(manualWorkerLoader) + E(외부연동)**. 단, 실제 등록 경로는 **두 갈래**라서 혼동하기 쉽다:

- **manualWorkerLoader 경로 (D)**: `manualWorkerLoader_SystemManagement.processManualWorkerLoad()`가 `Worker4ExecuteRestAPI`, `Worker4OperationLog`, `WorkerLockManager`, `Worker4ProcessFunction`을 `aWorkflowMap.setWorker(...)`로 등록.
- **`.inf` 경로 (C 방식 혼용)**: 핵심 클래스인 `Worker_SystemManagement`와 `RestInvokerFunction_SystemManagement`는 loader가 아니라 `home/jar/systemmanagement.inf`의 FQCN 목록으로 로드된다. 새 Worker를 만들었는데 안 뜨면 loader와 `.inf` 양쪽을 모두 확인하라.

근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SystemManagement/src/main/java/hfr/plugin/my5g/service/SystemManagement/common/WorkerLoader/manualWorkerLoader_SystemManagement.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SystemManagement/home/jar/systemmanagement.inf`

## REST contract
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SystemManagement/src/main/java/hfr/plugin/my5g/service/SystemManagement/function/RestInvokerFunction_SystemManagement.java`

`getContentList()` 등록 request type:
- `system_backup_data` / `neconfig_backup_data`
- `system_backup_restore` / `neconfig_backup_restore`
- `system_restore_status` / `ne_restore_status`
- `system_restore_status_init` / `ne_restore_status_init`
- `system_upgrade` / `upgrade_status`
- `system_release_info`
- `System_Upgrade_Notification` (`ScopeTypeBook.Inner`)
- `Restore_notification`
- `System_config` (`System_config.requestType`, `ScopeTypeBook.Inner`)

주의 — 분기는 있으나 등록이 불완전한 type:
- `release_info_file`: `processMicroServiceRestRequest()`에 분기 존재, `getContentList()` 등록은 **주석 처리** 상태 → 외부 호출 불가.
- `ServerInfo`: 분기 존재(`my5GSystemHelper.processGetServerInfo`), `getContentList()`에 **미등록** → 실질 누락.
- `Get_My5GBox_NeInfo`, `Cluster.requestType`, `Upgrade_System_Rollback` 등록 라인도 주석 처리 상태.

## 주요 Worker/Function 흐름
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SystemManagement/src/main/java/hfr/plugin/my5g/service/SystemManagement/worker/Worker_SystemManagement.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SystemManagement/src/main/java/hfr/plugin/my5g/service/SystemManagement/function/SystemUpgradeProcess.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SystemManagement/src/main/java/hfr/plugin/my5g/service/SystemManagement/function/AbsUpgradeProcess.java`

### backup/restore 흐름
- `Worker_SystemManagement`가 restore 상태를 메모리(`m_SystemRestoreStausData`)로 관리한다(`E_RestoreStatus`, `E_RestoreReturnType`, `E_TargetType` enum).
- `processRestoreSystem`에는 ReentrantLock이 없고 상태 체크만으로 중복을 막는다 → 동시 요청 race 가능.

### upgrade 흐름 (ArgoCD 기반)
- `SystemUpgradeProcess`가 Upgrade/Retry/Rollback 요청마다 내부 `UpgradeThread`를 시작하고, workload마다 `getUpgradeModelClass(E_WorkLoadName)` 분기로 `AbsUpgradeProcess` 구현체를 선택한다:
  - `UpgradeProcess4NMS` / `UpgradeProcess4UserMgmt` / `UpgradeProcess4Keycloak` / `UpgradeProcess4My5G` / `UpgradeProcess4Cluster` / `UpgradeProcess4ystemMgmt`(오타 이름 그대로가 실제 클래스명)
- `AbsUpgradeProcess`는 `prepareUpgrade()`, `upgrade()`, `rollback()` 등 훅 메서드를 제공하는 템플릿이다.
- 보조 프로세스: `UpgradeImageDownloadProcess`(폐쇄망 node 간 image 동기화), `My5GDBMigrationProcess`(업그레이드 시 DB migration).
- 외부 adapter: `ArgoCDApiClient`(patch/sync), `GitLabClient`(`gitlab/` 패키지), `KubernetesHelper`(POD 제어).

### E_SystemType.my5GBox 분기
`E_SystemType.my5GBox`일 때 `isUseNotiReceiver`, `useLicenseCheck` 동작이 달라진다(라이선스 체크·Noti 수신 비활성). box형 제품 대상 수정 시 이 분기를 먼저 확인한다.

## DB/Mongo/파일 흐름
- 이 서비스는 자체 MyBatis mapper/XML이 **없다**. `CommonMybatisDBConfiguration`은 프레임워크 기본 설정 역할.
- **MariaDB/MongoDB restore는 JDBC가 아니라 Shell 명령 직접 실행**: `Worker_SystemManagement.exec(String[])`에서 `/bin/sh -c /usr/bin/mysql ...`, `/bin/sh -c /usr/bin/mongorestore ...` 실행.
  - MariaDB restore 대상: `unify_base`, `unify_fault`, `unify_system`, `unify_nebackup`
  - MongoDB restore 대상: `operation_data`, `performance_data`
  - 자격증명(mysql root 패스워드, MongoDB 비밀번호)이 Worker 코드에 하드코딩되어 있다.
- system_config 등 다른 데이터는 내부 REST 호출(`m_WorkerRestContentManager.executeRestAPI`, systemmgmt_system 경유)로 조회/수정한다.
- release info 파일은 `E_System_config_Type`(ReleaseInfoFile/T1/T2)으로 read/write 경로가 분기된다.

## scheduler 흐름
자체 scheduler job 없음 (backup 스케줄은 별도 서비스 담당).

## 기능 확장점 (feature seams)

| seam | 추가 방법 | 등록 위치 / 근거 파일 |
| --- | --- | --- |
| ① REST request type | `processMicroServiceRestRequest()`에 else-if 분기 + `getContentList()`에 `new RestContent(this, 새타입, HttpMethodBook.POST, getServiceName(), "v1")` 추가. 내부 전용은 `ScopeTypeBook.Inner` 옵션. **분기만 추가하고 getContentList 등록을 빠뜨리면 ServerInfo처럼 죽은 API가 된다.** | `function/RestInvokerFunction_SystemManagement.java` |
| ② Worker/Function | 새 클래스 작성 후 `manualWorkerLoader_SystemManagement.processManualWorkerLoad()`에 `aWorkflowMap.setWorker(new 새워커(aWorkflowMap))` 등록. 프레임워크 공통 Worker류는 `home/jar/systemmanagement.inf`에 FQCN 한 줄 추가 방식도 병행 확인. 공통영역 무수정으로 하려면 product의 `manualWorkerLoaderEx_SystemManagement`에서 super 후 추가 등록. | `common/WorkerLoader/manualWorkerLoader_SystemManagement.java`, `home/jar/systemmanagement.inf` |
| 외부연동(E) — 새 Workload upgrade | `AbsUpgradeProcess` 상속 `UpgradeProcess4<New>` 클래스를 `function/`에 추가하고 `SystemUpgradeProcess.getUpgradeModelClass()`의 `E_WorkLoadCategory`/`E_WorkLoadName` 분기에 케이스 추가. `prepareUpgrade()/upgrade()/rollback()` 훅 오버라이드. | `function/SystemUpgradeProcess.java`, `function/AbsUpgradeProcess.java` |
| ⑤ Product override | URL 라우팅 변경은 `WorkerRestContentManager_SystemManagement.getURL()` 수정, product 전용 Worker 추가는 `manualWorkerLoaderEx_SystemManagement` 수정. 공통 `uNMS_Common` 코드는 건드리지 않는다. | `product/my5G/Project_start_systemmanagement/.../manualWorkerLoaderEx_SystemManagement.java` |
| 데이터 모델 | 새 restore 대상/업그레이드 상태 추가 시 `common/data/` 아래 data class 추가 + `Worker_SystemManagement`의 상태 관리 메서드(`setSystemRestoreStatus` 패턴) 확장. `E_RestoreStatus`, `E_RestoreReturnType`, `E_TargetType` enum에 값 추가 시 처리 흐름 전체 반영. | `common/data/datatype/E_TargetType.java` |

## product 확장점
근거:
- `<BACKEND_ROOT>/product/my5G/Project_start_systemmanagement/src/main/java/hfr/application/my5g/systemmgmt/manualWorkerLoaderEx_SystemManagement.java`
- `<BACKEND_ROOT>/product/my5G/Project_start_systemmanagement/src/main/java/hfr/application/my5g/systemmgmt/WorkerRestContentManager_SystemManagement.java`

- `manualWorkerLoaderEx_SystemManagement`(extends `manualWorkerLoader_SystemManagement`)가 super 등록 후 `WorkerRestContentManager_SystemManagement`를 추가로 WorkflowMap에 등록한다. `ServerStart_all`/`ServerStart_debug` 모두 Ex 버전을 사용한다.
- `WorkerRestContentManager_SystemManagement.getURL()` override: system_mgmt 이외 서비스 요청 URL을 `uems-backend.uems.svc.cluster.local:10410`(NOC는 `noc-backend.noc.svc.cluster.local:10410`, 로컬은 `127.0.0.1:10410`)으로 라우팅한다.

## 죽은/미구현 코드
- `release_info_file` getContentList 등록 주석 처리 → 분기 코드는 있으나 호출 불가.
- `ServerInfo` — 분기만 있고 getContentList 미등록.
- `Get_My5GBox_NeInfo`, `Cluster.requestType`, `Upgrade_System_Rollback` — 등록 라인 주석 처리 상태.

## mock/fake 지점
- `Worker_SystemManagement.exec(String[])` — Shell 명령(mysql/mongorestore) 실행 지점. **테스트 시 반드시 mock** (real DB restore 위험).
- `ArgoCDApiClient.connect()/patchApplication()/syncApplication()` — 업그레이드 외부 호출
- `GitLabClient` — GitLab API 호출
- `m_WorkerRestContentManager.executeRestAPI()` — 내부 서비스 REST 호출(System_config 조회 등)
- `m_WorkerLocalBaseData.processBaseDataGetListRequest()` — NE 목록 조회
- `KubernetesHelper` — POD 제어

## fixture 기준
- backup/restore request payload와 `E_TargetType`/`E_RestoreStatus` 조합
- upgrade request(workload 목록·timeout)와 workload별 기대 상태 전이
- release info / System_config mock 응답
- ArgoCD/GitLab/Kubernetes mock 응답
- Shell exec 성공/실패/부분 실패 case
- lock 상태, operation log expected row

## 변경 위험
- upgrade/restore 중복 실행: 메모리 상태(`m_SystemRestoreStausData`, upgradeStatus)로만 방어 → POD 재시작 시 상태 소실.
- `processRestoreSystem`에 lock 없음 → 동시 요청 race condition.
- Shell 명령 직접 실행: DB 자격증명 하드코딩 노출 + real restore 실행 위험.
- `E_System_config_Type` 잘못 전달 시 엉뚱한 release info 파일에 write.
- 백업 파일 날짜 파싱이 파일명 `split('-')` 인덱스 의존 → 파일명 규칙 변경 시 무음 오류.
- ArgoCD 업그레이드 중 POD 재배치: `UpgradeImageDownloadProcess` 실패해도 업그레이드는 성공 보고 → 다른 노드 image pull 실패 가능.

## 근거 파일
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SystemManagement/src/main/java/hfr/plugin/my5g/service/SystemManagement/ServiceStart_SystemManagement.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SystemManagement/src/main/java/hfr/plugin/my5g/service/SystemManagement/function/RestInvokerFunction_SystemManagement.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SystemManagement/src/main/java/hfr/plugin/my5g/service/SystemManagement/common/WorkerLoader/manualWorkerLoader_SystemManagement.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SystemManagement/home/jar/systemmanagement.inf`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SystemManagement/src/main/java/hfr/plugin/my5g/service/SystemManagement/worker/Worker_SystemManagement.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SystemManagement/src/main/java/hfr/plugin/my5g/service/SystemManagement/function/AbsUpgradeProcess.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SystemManagement/src/main/java/hfr/plugin/my5g/service/SystemManagement/function/SystemUpgradeProcess.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SystemManagement/src/main/java/hfr/plugin/my5g/service/SystemManagement/gitlab/GitLabClient.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.SystemManagement/src/main/java/hfr/plugin/my5g/service/SystemManagement/common/data/datatype/E_TargetType.java`
- `<BACKEND_ROOT>/product/my5G/Project_start_systemmanagement/src/main/java/hfr/application/my5g/systemmgmt/manualWorkerLoaderEx_SystemManagement.java`
- `<BACKEND_ROOT>/product/my5G/Project_start_systemmanagement/src/main/java/hfr/application/my5g/systemmgmt/WorkerRestContentManager_SystemManagement.java`
