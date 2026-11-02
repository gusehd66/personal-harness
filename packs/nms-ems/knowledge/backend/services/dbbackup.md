# DBBackup_mgmt Service

## 서비스 책임
DB 백업 **스케줄 관리 코디네이터**. 직접 DB dump/restore를 하지 않는다.
- 백업 스케줄의 등록/갱신/동기화(Quartz `manualRunOnce`)를 담당
- 실제 백업 실행은 `database_backup` 서비스로 REST 위임
- 스케줄 목록 조회는 `system` 서비스(`Db_backup` requestType, WorkType=Get)로 REST 위임
- shrink만 예외적으로 `DataPath.getDBBackupSubPath()` 경로의 백업 파일을 직접 삭제

주의: `DBBackup_mgmt`(스케줄 관리, 이 문서)와 `Database_Backup`(실제 백업 실행, 아키타입 C, 별도 문서)은 서로 다른 서비스다.

## 시작점과 실행 설정
- `ServiceStart_DBBackup_mgmt.main()` → `tServiceName = "dbbackup_mgmt"` → `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- 포트 **10458** (`home/config/dbbackup_mgmt.xml`)
- 이 서비스는 직접 mapper를 쓰지 않는다 — MybatisDBConfiguration은 공통 설정일 뿐, DB 접근은 전부 REST 위임.

근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.DBBackup_mgmt/src/main/java/hfr/plugin/my5g/service/DBBackup_mgmt/ServiceStart_DBBackup_mgmt.java`

## 아키타입
**B (코디네이터/디스패처)** — `../feature-extension.md` §B 참조.
- ② Worker: `Worker_DBBackup_mgmt`(abstractWorker 직접 상속). runMicroService 프레임워크 scan으로 WorkflowMap에 등록(.inf/manualWorkerLoader 아님).
- ③ Schedule: `ScheduleDatabaseBackup`(ScheduleAdapterRat 상속), 같은 방식으로 scan 등록. 단 `isAutoRun()=false`라 주기 실행이 아니라 `manualRunOnce`로만 fire된다.

## REST contract
모두 `ScopeTypeBook.Inner`(내부 전용), POST, v1. `getContentList()` 4건 등록.

| requestType 상수 | URL 문자열 | 처리 |
| --- | --- | --- |
| `Db_backup.requestType` | `db-backup` | **WorkType=Modify일 때만** `process_SyncBackupSchedule()` → 스케줄 재등록. 그 외 WorkType은 Success만 반환 |
| `RequestTypeStringBook.request_database_backup_schedule` | `request-database-backup-schedule` | `execute_DataBaseBackup()` → database_backup 위임 |
| `RequestTypeStringBook.request_database_backup_user` | `request-database-backup-user` | 위와 동일 경로(사용자 수동 백업) |
| `RequestTypeStringBook.shrink_database_backup` | `shrink-database-backup` | `Threadprocess_ShrinkDataBaseFile(Shrink_Database_BackupFile_Day)` |

근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.DBBackup_mgmt/src/main/java/hfr/plugin/my5g/service/DBBackup_mgmt/function/RestInvokerFunction_DBBackup_mgmt.java`

## 주요 Worker/Function 흐름
`Worker_DBBackup_mgmt` (initialize에서 `WorkerRestContentManager`, `WorkerLocalRestSessionManager`, `ScheduleDatabaseBackup`을 WorkflowMap에서 주입):
- `processRequestDataBaseScheduleList()` — `executeRestAPI(..., SupportService.system, Db_backup.requestType, ...)`로 스케줄 목록 조회. 실패 시 null 반환(로그만)
- `execute_DataBaseBackup(Db_backup, requestType)` — `executeRestAPI(..., SupportService.database_backup, requestType, ...)`로 실제 백업 위임. 응답 null/비Success면 `EmpException` throw
- `Threadprocess_ShrinkDataBaseFile(aDay)` — **별도 Thread로 비동기 실행**. `process_ShrinkDataBaseFile`이 `DataPath.getDBBackupSubPath()` 하위 디렉터리를 순회하며 생성일이 (현재-aDay일)보다 오래된 파일을 `MFile.deleteFile()`로 삭제
- `process_SyncBackupSchedule(Db_backup)` — `ScheduleDatabaseBackup.SyncBackupSchedule()` 호출

## DB/Mongo/파일 흐름
- 직접 DB/Mongo 접근 **없음**. 전부 system/database_backup 서비스로 REST 위임.
- 파일: shrink 경로만 `DataPath.getDBBackupSubPath()` 기준 파일 직접 삭제(`MFileManager.getDirectoryList/getFileList` + `MFile.deleteFile`).

## scheduler 흐름 (ScheduleDatabaseBackup)
`isAutoRun()=false` + `manualRunOnce` 패턴. 생명주기는 `m_RunningJobHashMap`(Db_backup.ID → JobKey)이 관리한다.
1. `initialized()` — system 서비스에서 스케줄 목록을 받아 `E_Activatioin.Enable`인 항목마다 `getNextRunningDate()` 계산 → `manualRunOnce(nextDate, data)` 등록 → map에 put
2. `execute()` — fire되면 map에서 remove → `execute_DataBaseBackup(...)` 호출(**예외는 catch 후 계속 진행**) → `getNextRunningDate()`로 다음 회차 재등록
3. `SyncBackupSchedule(Db_backup)` — 기존 JobKey 있으면 map remove + `deleteJob`, Enable이면 다음 회차 재등록. Disable이면 삭제만
- `getNextRunningDate()`: start/end date 범위 확인 후 `E_Period` switch — `Daily`(+1일), `Week`(period_detail 요일 목록에서 최소 간격 계산), `Monthly`(period_detail=실행일), 그 외 +1일

## 기능 확장점 (feature seams)
| seam | 추가 방법 | 등록 위치 / 근거 파일 |
| --- | --- | --- |
| ① REST request type | `getContentList()`에 `new RestContent(this, 새타입, POST, getServiceName(), "v1", ScopeTypeBook.Inner)` 추가 + `processMicroServiceRestRequest()` if-else 분기 + `MicroServiceCommonCD.RequestTypeStringBook` 상수 선언 + Worker에 대응 메서드 | `function/RestInvokerFunction_DBBackup_mgmt.java` |
| ② Worker function | `Worker_DBBackup_mgmt`에 public 메서드 추가. 기존 의존성(RestContentManager/SessionManager/Schedule)은 이미 initialize()에서 주입됨 — WorkflowMap에서 새 의존성이 필요할 때만 initialize() 수정 | `worker/Worker_DBBackup_mgmt.java` |
| ③ Scheduler | 주기 유형 추가 → `getNextRunningDate()` switch에 `E_Period` 분기 확장. 신규 job → `ScheduleAdapterRat` 상속 클래스 작성(프레임워크 scan이 자동 등록) | `schedule/ScheduleDatabaseBackup.java`, `.../db_backup/datatype/E_Period.java` |
| 외부 위임 대상 변경/추가 | `execute_DataBaseBackup()`의 `executeRestAPI()`에서 `MicroServiceCommonCD.SupportService` 상수 교체/분기 추가. 신규 서비스명은 SupportService에 등록 | `worker/Worker_DBBackup_mgmt.java`, `MicroServiceCommonCD.java` |
| 데이터 모델 | `Db_backup` 필드 추가 = `uEMS.plugin.my5g.Common`의 `Auto_Db_backup`(자동생성)/`Db_backup` 수정(소유는 system 서비스의 unify_system). shrink 파라미터 추가 = `RestRequestPropertyBook` 상수 + Function에서 파싱 | `.../common/data/unify_system/db_backup/Db_backup.java` |

## product 확장점
없음. `product/` 하위에 DBBackup_mgmt 관련 override 클래스 없음. Worker/Function 모두 추상 계층 없이 직접 상속.

## 죽은/미구현 코드
- `process_ShrinkDataBaseFile()` — "최소 5개 파일은 남긴다"(tRemainMaxCount) 로직 전체 주석 처리 = 미동작. 현재는 기준일만으로 삭제
- `ScheduleDatabaseBackup.execute()` — 주석은 "DB에서 다시 읽어와 최신 정보로 판단"이지만 실제로는 `tNewData = tOldData` 재사용 = **재조회 미구현**. fire 사이에 스케줄이 변경돼도 execute() 재등록은 옛 데이터 기준(Modify 시 SyncBackupSchedule이 보정)

## mock/fake 지점
- `WorkerRestContentManager.executeRestAPI()` — system/database_backup REST 응답 mock으로 Worker 단위 테스트
- `MFileManager.getDirectoryList()/getFileList()`, `MFile.deleteFile()` — shrink 파일시스템 mock
- `ScheduleDatabaseBackup.manualRunOnce()` — Quartz job 등록 검증 mock
- `Db_backupCollection` 반환값 mock — `initialized()` 스케줄 초기화 경로 검증

## 변경 위험
- `DataPath.getDBBackupSubPath()` 경로 오지정 시 운영 파일 실삭제 (shrink는 real file delete)
- shrink 기준일 aDay가 0/음수여도 방어 코드 없음 — 음수면 미래 기준일이 되어 사실상 전체 파일 삭제
- `m_RunningJobHashMap`과 Quartz job 상태 불일치 시 스케줄 중복 실행/누락
- `SyncBackupSchedule`의 deleteJob→manualRunOnce 사이 타이밍에 fire가 겹치면 스케줄 꼬임 가능
- `execute_DataBaseBackup()`의 EmpException은 REST 경로에서는 500 응답, 스케줄 경로에서는 execute() 내 catch로 흡수 후 다음 회차 재등록은 계속됨(백업 실패가 조용히 지나갈 수 있음 — 로그만 남음)

## 근거 파일
`<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.DBBackup_mgmt/src/main/java/hfr/plugin/my5g/service/DBBackup_mgmt/` 하위:
- `ServiceStart_DBBackup_mgmt.java`
- `function/RestInvokerFunction_DBBackup_mgmt.java`
- `worker/Worker_DBBackup_mgmt.java`
- `schedule/ScheduleDatabaseBackup.java`
- `common/data/DBBackup_mgmtCD.java`

`<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/`:
- `unify_system/db_backup/Db_backup.java` (+ `Auto_Db_backup`, `Db_backupCollection`, `datatype/E_Period.java`, `datatype/E_Activatioin.java`)
- `MicroServiceCommonCD.java` (RequestTypeStringBook, SupportService, RestRequestPropertyBook)
- `DataPath.java` (getDBBackupSubPath)
