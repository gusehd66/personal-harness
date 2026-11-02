# Database_Backup Service

## 서비스 책임
MariaDB/MongoDB 데이터베이스 백업 실행 전담 서비스. 스케줄/사용자/시스템 전체(uNMS·HiEMS) 백업과 업그레이드 사전 백업을 shell script(`db_backup_LINUX.sh`) 호출로 수행하고, 결과를 System 서비스에 REST notification으로 전달한다. 백업 정책·이력 관리(UI 요청 접수)는 `DBBackup_mgmt`(아키타입 B)가 담당하고, 이 서비스는 그 위임을 받아 실제 실행만 한다.

- 주요 입력: `DBBackup_mgmt`/System 서비스에서 위임되는 REST request(6종), `Db_backup` 데이터(백업 대상 타입), `UpgradeSystem` 데이터(업그레이드 백업)
- 주요 출력: `/data/dbbackup` 하위 백업 파일(`dbbackup_{DB}_{type}_{trigger}_{timestamp}.{ext}`, 전체 백업은 `/data/dbbackup/unms`), System 서비스로의 결과 notification

## 시작점과 실행 설정
- `ServiceStart_Database_Backup.main()` → `tServiceName = "database_backup"` → `runMicroService.run(tServiceName, DefaultSecurityHttps, CommonMybatisDBConfiguration)`
- HTTP 포트 **10459** (`home/config/database_backup.xml`의 `<port>`)
- 자체 MyBatis mapper 없음 — `CommonMybatisDBConfiguration`은 공통 설정만 적용

## 아키타입: C (.inf 플러그인 로더)
②③ 등록 = `home/jar/database_backup.inf`의 FQCN 목록 기반. 단, 이 .inf 파일은 debug(개발) 모드에서 `Loadplugin`이 classpath 스캔 결과로 **재생성**하는 파일이다(`Loadplugin.Load()`가 기존 .inf 삭제 후 다시 씀). 운영 배포에서는 .inf/jar 기준 로드.

⚠ 체크인된 `database_backup.inf`에는 `RestInvokerFunction_Database_Backup`, `Worker_Database_Backup`과 공통 Worker들만 있고 `ScheduleShrinkDataBase`, `WorkerShrinkDataStorage`는 **없다**. 스케줄 관련 변경 전에 배포 환경에서 재생성된 .inf(또는 plugin.inf)에 해당 클래스가 실제 적재되는지 먼저 확인하라.

## REST contract
`RestInvokerFunction_Database_Backup.getContentList()`에 6종 등록, 전부 `HttpMethodBook.POST` + `ScopeTypeBook.Inner`(내부 전용). 상수는 `MicroServiceCommonCD.RequestTypeStringBook`.

| request type | Worker 메서드 | 동작 |
| --- | --- | --- |
| `request-database-backup-schedule` | `dataBaseBackupSchedule()` | 스케줄 트리거 단일 DB 백업 |
| `request-database-backup-user` | `dataBaseBackupUser()` | 사용자 요청 단일 DB 백업 |
| `request-database-backup-unms` | `dataBaseBackupuNMS()` | uNMS 전체 DB 백업 |
| `request-database-backup-hiems` | `dataBaseBackupHiEMS()` | HiEMS 전체 DB 백업 |
| `request-database-backup-unms-upgrade` | `unmsBackup4Upgrade()` | 업그레이드 사전 백업(비동기) |
| `request-database-backup-hiems-upgrade` | `hiemsBackup4Upgrade()` | 업그레이드 사전 백업(비동기) |

## 주요 Worker/Function 흐름
- `RestInvokerFunction_Database_Backup.processMicroServiceRestRequest()` — request type if-else 분기 → `m_Worker`(Worker_Database_Backup) 메서드 호출
- `Worker_Database_Backup.executeBackupDatabase()` — 단일 DB 백업 공통 경로. `E_Type` 분기 후 `Runtime.getRuntime().exec()`로 `{EMP_HOME}/bin/db_backup_LINUX.sh` 호출
- `Worker_Database_Backup.executeAllBackupDatabase()` — 전체 백업용, DB type/name 인자로 동일 스크립트 호출
- `processBackupUNMS()` 경로 — MariaDB 4개(`unify_base`/`unify_fault`/`unify_system`/`unify_nebackup`) + MongoDB 2개(`performance_data`/`operation_data`, archive.gz) 순차 백업
- 업그레이드 백업 — 내부 `Upgrade4BackupThread`로 비동기 실행 후 `sendBackup4UpgradeResult()`가 `WorkerRestContentManager.executeRestAPI()`로 `MicroServiceCommonCD.SupportService.system`에 결과 notification 전송
- 보조 Worker: `WorkerCommonNotifier`, `WorkerRestContentManager`, `WorkerLocalRestSessionManager` (공통 플러그인, .inf 등록)

## DB/Mongo/파일 흐름
- 직접 MyBatis mapper/CD 없음(`Database_BackupCD`는 상수 정의만). 백업은 전부 shell script 위임
- MariaDB 대상: `unify_base`, `unify_fault`, `unify_system`, `unify_nebackup`, `emp_orange_hfr`(HiEMS)
- MongoDB 대상: `performance_data`, `operation_data` (mongodump archive.gz)
- 결과 파일: `/data/dbbackup` 하위(user/schedule), `/data/dbbackup/unms`(전체 백업). 파일명 `dbbackup_{DBTYPE}_{type}_{trigger}_{timestamp}.{ext}`
- OS 분기: `getOutput()`이 `FileSystems.getDefault().supportedFileAttributeViews().contains("posix")`로 Linux/Windows 판별

## scheduler 흐름
- `ScheduleShrinkDataBase` (`ScheduleAdapterRat` 상속, `isAutoRun()=true`, `getInterval()=60*1000` — 60초 주기)
- `execute()` → `WorkerShrinkDataStorage.isFileShrikRunOK(now, E_FileShrink_Type.DataBase, 12)` 체크 후 `runShrinkDB_Backup()` 위임 → 오래된 백업 파일 정리(shrink)
- `WorkerShrinkDataStorage`는 `uEMS.plugin.my5g.Common` 공통 모듈 — shrink 정책 변경은 타 서비스에도 영향
- ⚠ 위 아키타입 절의 .inf 미등록 caveat 적용 — 실동작 여부는 배포 .inf 확인 필요

## 기능 확장점 (feature seams)

| seam 유형 | 추가 방법 | 등록 위치 / 근거 파일 |
| --- | --- | --- |
| ① REST request type | (1) `MicroServiceCommonCD.RequestTypeStringBook`에 상수 추가(`request_database_backup_xxx = "request-database-backup-xxx"`), (2) `processMicroServiceRestRequest()` if-else 분기 추가, (3) `getContentList()`에 `new RestContent(this, 상수, HttpMethodBook.POST, getServiceName(), "v1", ScopeTypeBook.Inner)` 추가 | `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Database_Backup/src/main/java/hfr/plugin/my5g/service/Database_Backup/function/RestInvokerFunction_Database_Backup.java` |
| ② Worker/Function | `Worker_Database_Backup`에 public 메서드 추가 후 Function의 `m_Worker`로 호출. `executeBackupDatabase()`/`executeAllBackupDatabase()` 재사용 시 script 호출·출력 파일 처리 공통화. 신규 Worker 클래스면 `.inf` FQCN 등록 필요 | `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Database_Backup/src/main/java/hfr/plugin/my5g/service/Database_Backup/worker/Worker_Database_Backup.java`, `home/jar/database_backup.inf` |
| ③ Scheduler job | `ScheduleAdapterRat` 상속 클래스를 `schedule/`에 추가, `isAutoRun()`/`getInterval()` 구현. 로드는 Loadplugin 스캔/.inf 기반 — 배포 .inf 반영 여부 확인 필수 | `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Database_Backup/src/main/java/hfr/plugin/my5g/service/Database_Backup/schedule/ScheduleShrinkDataBase.java` |
| ④ 데이터 모델(백업 대상 DB 타입) | `E_Type` enum에 값 추가(Base(0)/Fault(2)/System(4)/Performance(5)/OperationLog(6)/HiEMS(7)/Nebackup(8)) 후 `executeBackupDatabase()` 내 타입 분기(if/else if)에 MARIADB/MONGODB 케이스 추가. 전체 백업 대상이면 `processBackupUNMS()` 순차 목록도 함께 수정 | `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/unify_system/db_backup/datatype/E_Type.java` |
| ⑤ 외부 어댑터(script) | script 교체/OS 분기 추가 시 `executeBackupDatabase()`·`executeAllBackupDatabase()`의 `Runtime.exec()` cmd 경로(`{EMP_HOME}/bin/db_backup_LINUX.sh`) 수정. OS 판별은 기존 posix 체크 패턴 재사용 | `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Database_Backup/src/main/java/hfr/plugin/my5g/service/Database_Backup/worker/Worker_Database_Backup.java` |

## product 확장점
없음. `Worker_Database_Backup`이 `abstractWorker` 직접 상속 단일 구현체. product 분기는 `E_Type` enum 값으로 내부 처리 — product override 폴더/추상 클래스 없음.

## mock/fake 지점
1. `Worker_Database_Backup.executeBackupDatabase()`의 `Runtime.exec()` 호출부 — 인터페이스 분리 시 script 없이 단위 테스트 가능
2. `WorkerRestContentManager.executeRestAPI()` — System 서비스 notification 전송 mock
3. `WorkerShrinkDataStorage.runShrinkDB_Backup()` — `ScheduleShrinkDataBase` 단위 테스트용 mock

## 변경 위험
- 새 DB 추가 시 `E_Type` 분기와 `processBackupUNMS()` 순차 목록을 **둘 다** 수정해야 함 — 하나만 고치면 전체 백업에서 누락
- `{EMP_HOME}/bin/db_backup_LINUX.sh` 부재 시 백업 전체 실패 (Runtime.exec 직접 호출)
- 결과 notification은 System 서비스(REST)로 전송 — System 다운 시 결과 유실
- 업그레이드 백업은 `Upgrade4BackupThread` 비동기 — 완료 전 프로세스 종료 시 notification 누락
- shrink 정책 변경은 공통 모듈 `WorkerShrinkDataStorage`(uEMS.plugin.my5g.Common) 수정 = 타 서비스 side effect 확인 필요
- 코드 주석에 [v6.3]/[v6.5] 변경 이력([robot]) 존재 — 운영 script와 코드 간 drift 가능성, 수정 전 실제 script 버전 확인

## 근거 파일
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Database_Backup/src/main/java/hfr/plugin/my5g/service/Database_Backup/ServiceStart_Database_Backup.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Database_Backup/src/main/java/hfr/plugin/my5g/service/Database_Backup/function/RestInvokerFunction_Database_Backup.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Database_Backup/src/main/java/hfr/plugin/my5g/service/Database_Backup/worker/Worker_Database_Backup.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Database_Backup/src/main/java/hfr/plugin/my5g/service/Database_Backup/schedule/ScheduleShrinkDataBase.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Database_Backup/src/main/java/hfr/plugin/my5g/service/Database_Backup/common/data/Database_BackupCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Database_Backup/home/config/database_backup.xml`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Database_Backup/home/jar/database_backup.inf`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/unify_system/db_backup/Db_backup.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/unify_system/db_backup/datatype/E_Type.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/worker/WorkerShrinkDataStorage.java`
- `<BACKEND_ROOT>/Platform_framework/EMSPlatform/src/hfr/platform/Rat/product/server/app/microservice/runMicroService.java`
- `<BACKEND_ROOT>/Platform_framework/Framework4/src/main/java/hfr/framework/server/init/Loadplugin.java`
