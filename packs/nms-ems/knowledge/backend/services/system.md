# System Service

## 서비스 책임
5G NMS/EMS의 시스템 레벨 관리 서비스. 시스템 설정(System_config), DNN/TAC/NR주파수 기준정보, DB 백업/복원 조율, 사용자 프로필, NOC 위탁 관리, 시스템 업그레이드/롤백(ArgoCD), 데이터 Shrink 스케줄을 단일 마이크로서비스로 처리한다.

## 주요 입력
- Gateway/타 서비스에서 오는 REST request (전부 `ScopeTypeBook.Inner`)
- System_config/AllBackup 설정 (DB)
- 업그레이드 요청의 `E_WorkLoadName` (uNMS/SMP/PCF/UPF/Keycloak/UserMgmt/vCore/vCoreDB/My5GBox/CUDU)
- NOC 위탁 설정 (ConsignNOC)

## 주요 출력
- unify_system 9개 테이블 CRUD 결과 (MariaDB)
- AllBackup/Restore/Upgrade 진행 상태 notification
- 타 마이크로서비스 위임 호출 (`WorkerRestContentManager.executeRestAPI`)

## 시작점과 실행 설정
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.System/src/main/java/hfr/plugin/my5g/service/System/ServiceStart_System.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.System/src/main/java/hfr/plugin/my5g/service/System/SystemServerConfig.java`

확인 기준:
- `tServiceName = "system"` → `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- `SystemServerConfig extends ServerConfigOnlyDB` — `getServiceName()="system"`, `getMapperList()`/`getJavaTypeList()`는 `unify_system`에 위임
- `processDBUpgrade()`는 ClassPath의 `/hfr/plugin/my5g/service/System/updatesql/update.sql` 실행 (DDL 마이그레이션)
- MongoDB 사용 없음

## 아키타입
**A (unify 자동등록)** — `../feature-extension.md` §A 참조.
- ② Worker 등록: `unify_system.java`의 *Auto Generated Code* 영역 (`toDataRequestTypeListAuto()`, `toWorker4Auto()`, `getMapperList()`, `getJavaTypeList()`)
- ③ Schedule 등록: `Worker_System`/`Worker_Schedule`의 `initialized()` 또는 프레임워크 스캔. `ConsignNmsCheckSchedule`은 `ScheduleAdapterRat` 상속으로 자동 실행 판정
- ServerConfig는 unify_system에 위임하므로 새 엔티티 추가 시 `SystemServerConfig`는 수정 불필요

## REST contract
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.System/src/main/java/hfr/plugin/my5g/service/System/function/RestInvokerFunction_System.java` (`getContentList()` 등록 + `processMicroServiceRestRequest()` 분기, 약 2,670줄)

| 그룹 | request type |
| --- | --- |
| unify CRUD | `User_profile.requestType`, `System_config.requestType`, `System_config.requestTypeEms`, `Db_backup.requestType`, `Nr_frequency_band.requestType`, `Dnn_mgmt.requestType`, `Tac_mgmt.requestType`, `Cluster_app_server.requestType` |
| 시스템 조회 | `Request_vCore_URL`, `system_environment`, `ServerLogRequest.requestType`, `ServerInfo`, `Get_license_expire_time`, `Get_My5GBox_NeInfo`, `Cluster.requestType` |
| NOC 위탁 | `NOC_CONSIGN_MANAGEMENT`, `NOC_Request`, `NMS_Request` |
| AllBackup | `Manual_AllBackup`, `Cancel_AllBackup`, `Status_AllBackup`, `All_Backup_Notification` |
| Restore | `Data_Restore`, `user_restore`, `neconfig_restore`, `Restore_notification` |
| Upgrade | `system_upgrade`, `System_Upgrade_Notification`, `Upgrade_System_Rollback` |
| 파일 업로드 | `UploadLogFile` (processMicroServiceRestData 쪽 분기) |

상수 위치: `MicroServiceCommonCD.RequestTypeStringBook` 또는 서비스 로컬 `RequestTypeStringBook`, unify 엔티티는 각 데이터 클래스의 `requestType` static.

## 주요 Worker/Function 흐름
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.System/src/main/java/hfr/plugin/my5g/service/System/worker/Worker_System.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.System/src/main/java/hfr/plugin/my5g/service/System/worker/schedule/Worker_Schedule.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.System/src/main/java/hfr/plugin/my5g/service/System/common/worker/unify_system/unify_system.java`

- `RestInvokerFunction_System.initialize()`에서 `WorkflowMap.getWorker()`로 주입: `Worker_System`, `Worker_Schedule`, `WorkerSystem_config`, `WorkerDnn_mgmt`, `WorkerNoc_consign_management`, `WorkerCluster_app_server`, `WorkerRestContentManager` 등
- unify_system 자동등록 엔티티 9종(각각 `AutoWorker*` + `Worker*` 쌍): cluster_app_server, db_backup, dnn_mgmt, ems_db_update_info, noc_consign_management, nr_frequency_band, system_config, tac_mgmt, user_profile
- `Worker_System.initialized()`: File/DB Shrink 마지막 실행시각 초기화(DataBase −11h, NEConfigBackup −5h 등) + `initAllBackupStatus()` + `initScheduleAllBackup()` — Shrink 대상: DB백업파일/NE설정백업/이벤트/PM/OperationLog/Alarm
- 업그레이드: `GeneralUpgradeProcess.getUpgradeModelClass()`가 `E_WorkLoadName`별로 `WorkloadProcessNMS/SMP/PCF/UPF/Keycloak/UserMgmt/vCore/vCoreDB/My5GBox/CUDU`로 디스패치. 롤백은 ArgoCD client(`rollbackApplication`/`patchApplication`/`syncApplication`)
- NOC 위탁: `NOCProcessNMS`(NMS측) / `NOCProcessNOC`(NOC측)가 `NOC_Request`/`NMS_Request` 처리

## DB/파일 흐름
- MyBatis + MariaDB (`CommonMybatisDBConfiguration`, `LOCAL_DOMAIN` 환경변수 기반 DNS)
- `unify_system.getMapperList()` 등록 mapper XML 9개 — 전부 `hfr/plugin/my5g/common/mybatis/unify_system/` (uEMS.plugin.my5g.Common):
  `mybatis_cluster_app_server.xml`, `mybatis_db_backup.xml`, `mybatis_dnn_mgmt.xml`, `mybatis_ems_db_update_info.xml`, `mybatis_noc_consign_management.xml`, `mybatis_nr_frequency_band.xml`, `mybatis_system_config.xml`, `mybatis_tac_mgmt.xml`, `mybatis_user_profile.xml`
- 스키마 변경은 `updatesql/update.sql`에 ALTER/CREATE 추가 → 기동 시 `processDBUpgrade()` 자동 실행

## scheduler 흐름
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.System/src/main/java/hfr/plugin/my5g/service/System/schedule/ConsignNmsCheckSchedule.java`

- `ConsignNmsCheckSchedule extends ScheduleAdapterRat` — `getInterval()=300000`(5분). `isAutoRun()`은 `Server_RunningConfigPropertiesBook.envName_SystemType == SystemTypeBook.NMS`일 때만 true → **EMS 환경에서는 돌지 않는다**
- AllBackup 스케줄은 `Worker_System.initialized() → initScheduleAllBackup()`에서 DB의 `AllBackupSetConfig` 기반 등록 — 설정 미존재 시 미등록(무음 실패)
- Shrink 작업은 `Worker_System`의 `runShrink*` 계열이 마지막 실행시각/실행중 플래그 map으로 중복 실행 차단

## 기능 확장점 (feature seams)

| seam | 추가 방법 | 등록 위치 / 근거 파일 |
| --- | --- | --- |
| ① REST request type | `getContentList()`에 `new RestContent(this, <type>, HttpMethodBook.POST, getServiceName(), "v1", ScopeTypeBook.Inner, ...)` 추가 + `processMicroServiceRestRequest()` else-if 분기 추가 + `RequestTypeStringBook` 상수 선언. Request/Response 데이터 클래스는 `common/data/`에 작성 | `function/RestInvokerFunction_System.java` |
| ② Worker (신규 엔티티) | `common/worker/unify_system/<entity>/`에 `AutoWorker<Entity>`(CRUD 자동) + `Worker<Entity>`(비즈니스) 작성 → `unify_system.java` Auto Generated 영역의 `toDataRequestTypeListAuto()`, `toWorker4Auto()`, `getMapperList()`, `getJavaTypeList()`에 등록 → `processCommonData()` 범용 경로로 자동 라우팅. SystemServerConfig 수정 불필요 | `common/worker/unify_system/unify_system.java` |
| ③ Scheduler | `schedule/`에 `ScheduleAdapterRat` 상속 클래스(`execute`/`isAutoRun`/`getInterval` 구현) 작성. NMS/EMS 실행 조건은 `isAutoRun()`에서 `envName_SystemType`으로 분기 | `schedule/ConsignNmsCheckSchedule.java` (패턴 예시) |
| ④ DB mapper | `hfr/plugin/my5g/common/mybatis/unify_system/mybatis_<entity>.xml` 추가 + `unify_system.MybatisMapperBook`에 `Dao_<Entity>` 상수 + `getMapperList()` 등록. Enum 컬럼은 `getJavaTypeList()`에 TypeHandler 추가. DDL은 `updatesql/update.sql` | `common/worker/unify_system/unify_system.java`, `SystemServerConfig.java` |
| ⑤ 업그레이드 워크로드 | `function/`에 `AbsSubWorkloadUpgradeProcess`(HiEMS 계열은 `AbsSubWorkloadUpgradeProcess4HiEMS`) 상속 `WorkloadProcessXXX` 작성 → `GeneralUpgradeProcess.getUpgradeModelClass()`의 `E_WorkLoadName` 분기에 else-if 한 절 추가. **분기 누락 시 null 반환 → NPE** | `function/GeneralUpgradeProcess.java` |

## product 확장점
product 폴더(my5G/openhaul/sixgen-ems)의 직접 override는 이 서비스에 없다. 제품별 차이는 업그레이드 계층에서 흡수: `AbsSubUpgradeProcess` → `AbsSubWorkloadUpgradeProcess`(+ `AbsSubWorkloadUpgradeProcess4HiEMS`) 상속 구조로 워크로드별/HiEMS별 처리를 분리한다.

## 죽은/미구현 코드
- `WorkloadProcessAPC` — 클래스는 존재하나 `getUpgradeModelClass()`의 `E_WorkLoadName.APC` 분기가 주석 처리 = 비활성 (`function/GeneralUpgradeProcess.java`)
- `UE_Trace`, `upgrade_status`, `system_release_info` — `getContentList()`에 RestContent 등록은 있으나 `processMicroServiceRestRequest()` 분기 미확인 (수정 전 실제 처리 경로 확인 필요)
- `Ems_Sync_DNN/TAC/System_Config` 분기 — 전부 주석 처리

## mock/fake 지점
1. `WorkerSystem_config` — 설정 CRUD/읽기 전 구간
2. `WorkerRestContentManager.executeRestAPI(...)` — 타 마이크로서비스(fault/performance/dbbackup/usermanagement 등) 연동 전 구간
3. `NOCProcessNMS` / `NOCProcessNOC` — NOC 위탁 연동
4. `GeneralUpgradeProcess` — ArgoCD 연동 및 업그레이드 흐름
5. `WorkflowMap.getWorker()` 주입 Worker 전반

## 변경 위험
- `RestInvokerFunction_System`이 단일 파일 약 2,670줄 — else-if 체인에 분기 추가 시 merge 충돌 잦음
- `unify_system.java`의 Auto Generated 영역("Do not modify this area" 주석) — 수동 편집 시 코드 생성 도구와 동기화 깨질 수 있음. 관리 방식(도구/수동) 미확정 상태이므로 수정 전 확인
- `getUpgradeModelClass()`에 새 `E_WorkLoadName` 분기 누락 시 null 반환 → NPE
- `Worker_System.initialized()`의 AllBackup 스케줄 — DB 설정 미존재 시 무음으로 미등록
- `ConsignNmsCheckSchedule`은 NMS 타입에서만 자동 실행 — EMS 환경 회귀 테스트에서 누락되기 쉬움

## 근거 파일
`<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.System/src/main/java/hfr/plugin/my5g/service/System/` 하위:
- `ServiceStart_System.java`, `SystemServerConfig.java`
- `function/RestInvokerFunction_System.java`
- `common/worker/unify_system/unify_system.java`
- `worker/Worker_System.java`, `worker/schedule/Worker_Schedule.java`
- `schedule/ConsignNmsCheckSchedule.java`
- `function/GeneralUpgradeProcess.java`, `function/AbsSubUpgradeProcess.java`, `function/AbsSubWorkloadUpgradeProcess.java`
