# Backend 기능 확장 가이드 (feature seams)

## 목적
"누가 기능을 요청하면 해당 서비스에서 기존 구조를 바꾸지 않고 빠르게 추가/수정"하기 위한 공통 기준 문서다.
모든 backend 서비스 문서는 이 문서를 먼저 참조하고, 서비스별 차이만 각 서비스 문서에 적는다.

NMS-core 서비스는 거의 모두 동일한 **기능 추가 5-seam**을 따른다. 단, ②③(Worker/Schedule)의 **등록 방법**은 서비스 **아키타입**에 따라 달라진다. 기능 요청을 받으면 먼저 (1) 어느 서비스인지, (2) 그 서비스가 어느 아키타입인지, (3) 5-seam 중 어디에 끼우는지 순서로 분해한다.

근거 경로는 모두 `<BACKEND_ROOT>`(backend 저장소 루트) 기준이며, 각 서비스 폴더는 `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/<service>` 이다.

## 기능 추가 5-seam (전 서비스 공통)

| seam | 추가 위치 | 비고 |
| --- | --- | --- |
| **① REST request type** | `RestInvokerFunction_<Svc>.getContentList()`에 `new RestContent(this, <type>, HttpMethodBook.POST, getServiceName(), "v1")` 한 줄 + `processMicroServiceRestRequest()` if-else 분기 추가 + `RequestTypeStringBook`(보통 `MicroServiceCommonCD.RequestTypeStringBook` 또는 `<Svc>CD`)에 상수 선언 | 전 서비스 동일. 내부 전용 API는 `ScopeTypeBook.Inner` |
| **② Worker / Function** | `Worker_<Svc>`에 public 메서드 추가, 또는 `abstractWorker` 상속 신규 Worker 작성 | **등록 방법은 아키타입별 (아래 §아키타입)** |
| **③ Scheduler job** | `ScheduleAdapterRat` 상속 → `execute()/isAutoRun()/getInterval()` 구현 | 등록 방법은 아키타입별 |
| **④ DB mapper** | `uEMS.plugin.my5g.Common`의 `hfr/plugin/my5g/common/mybatis/unify_<domain>/mybatis_*.xml` 추가 + `unify_<domain>.getMapperList()`에 `MybatisMapperBook.Dao_*` 등록 + `updatesql/update.sql`에 ALTER/CREATE 추가(`processDBUpgrade`에서 자동 실행) | unify 계열(아키타입 A) |
| **⑤ Product override** | product 서비스(`product/my5G`, `product/openhaul`, `product/sixgen-ems`)에서 `AbsFunction*`/`Worker4*` 상속 + `FunctionTypeBook` 또는 `manualWorkerLoaderEx_*` 또는 `InvokerAdapterManager`에 등록 | **공통영역 무수정**으로 product별 차이 구현 |

> 공통영역(`Framework`, `Platform_framework`, `uNMS_Common`, 공통 `library`) 변경 전에 항상 ⑤ product override로 대체 가능한지 먼저 검토한다. 자세한 원칙은 `architecture.md` 참조.

## 서비스 아키타입 5종

기능 요청 시 ②③ 등록 절차가 아키타입에서 갈린다. 한 서비스가 여러 아키타입을 겸할 수 있다(예: Performance = A+E).

### A. unify_* 자동등록 (DB-CRUD)
대상: `BaseData`(unify_base) · `Fault`(unify_fault) · `Performance`(unify_performance) · `System`(unify_system) · `Work_mgmt`(unify_workmgmt) · `NeBackupRestore_mgmt`(unify_nebackup)

새 엔티티 추가 절차(코드 거의 없이 CRUD가 동작):
1. `common/data/unify_<domain>/<entity>/<Entity>.java`(IData/Auto_* 패턴) 생성
2. `common/worker/unify_<domain>/<entity>/Worker<Entity>.java`(AbsBaseDataWorker 등) 생성
3. `unify_<domain>.java`의 *Auto Generated Code* 영역 — `toDataRequestTypeListAuto()`, `toWorker4Auto()`, `toDataClass4Auto()`, `getMapperList()`, `getJavaTypeList()`에 등록
4. ④ mapper XML 추가

→ 등록만 하면 `processCommonData()` 범용 경로(Get/GetList/Search/Add/Modify/Delete/DeleteEx…)로 **자동 라우팅**. "구조 안 바꾸고 추가"가 가장 잘 되는 유형.

### B. 코디네이터 / 디스패처 (직접 DB 안 함)
대상: `DBBackup_mgmt`→database_backup/system · `Recovery`→nebackuprestore/ne_configuration · `NeDiscovery_mgmt`→EMS/product · `embeddedEMS`→URL세그먼트 dispatch · `Gateway_Uems`→NMS/EMS/NOC 포워딩 · `NeConfig_Backup`→southbound

기능 추가 = ① request type 분기 + `m_WorkerRestContentManager.executeRestAPI(getClass(), SupportService.<svc>, requestType, json)` 위임 대상 추가/교체. 위임 서비스명은 `MicroServiceCommonCD.SupportService`에 상수. embeddedEMS는 URL 첫 세그먼트로 `AbsEmbeddedEMSRestInvokerFunction` 구현체 라우팅.

### C. .inf 플러그인 로더
대상: `Session_mgmt` · `Notify_mgmt` · `ulog_mgmt` · `Database_Backup` · `NeConfig_Backup`

②③ 등록 = `home/jar/<svc>.inf`(또는 `home/config`)에 새 Worker/Schedule 클래스 **FQCN 한 줄** 추가 → `runMicroService`의 `loadStaticClass`가 자동 로드. **이 한 줄을 빠뜨리면 클래스를 만들어도 로딩 안 됨**(흔한 실수).

### D. manualWorkerLoader 로더
대상: `UserManagement`(D·C) · `SystemManagement`(D·C) · `em_emulator` · `Report_mgmt` · `update_service`

②③ 등록 = `manualWorkerLoader_<Svc>.processManualWorkerLoad()`에서 `aWorkflowMap.setWorker(new <Worker>(aWorkflowMap))`.
product 확장 = `manualWorkerLoaderEx_<Svc>`(서브클래스)에서 `super.processManualWorkerLoad()` 호출 후 product 전용 Worker 추가 등록 → **공통영역 무수정**. URL 라우팅은 `WorkerRestContentManager_<Svc>.getURL()` override.

주의(실측): `UserManagement`·`SystemManagement`는 **하이브리드(D·C)**다 — manualWorkerLoader는 framework 공통 worker만 등록하고, 서비스 고유 `Worker_<Svc>`/`RestInvokerFunction_<Svc>`/Schedule은 `home/jar/<svc>.inf` FQCN 로딩이다. 이 두 서비스에서 서비스 고유 클래스를 추가할 때는 `.inf`에 등록해야 한다.

### E. 외부연동 / southbound adapter
대상: `Performance`(SFTP/장비), `vCore_mgmt`(vCore OpenAPI), `SoftwareUpgrade_mgmt`(NE image), `Gateway_Uems`/`WebCli`(WebSocket), `Report_mgmt`(SFTP/NBI), `Ne_Configuration`(ShelfView/InvokerAdapter)

확장 = `AbsFunction*`/adapter 상속한 product/기능 클래스 작성 + `FunctionTypeBook` 코드로 등록하면 스케줄러/Invoker가 자동 수집.
주요 등록 코드 예: `performanceMonitoring`, `funtion_sftp`, `shelfViewFunction`, `nediscovery_ip`, `command_PathGuide`, `alarmDataSync`, `fault`. WebSocket은 `*.xml`의 `<listener>`/`<path>`로 어댑터 지정.

## 등록 위치 cheat-sheet
"추가했는데 안 됨"의 대부분은 등록 누락이다. 서비스별 ②③ 등록 위치:

| 등록 메커니즘 | 위치 | 해당 서비스 |
| --- | --- | --- |
| unify_* Auto 영역 | `unify_<domain>.java` | BaseData, Fault, Performance, System, Work_mgmt, NeBackupRestore_mgmt |
| `.inf` FQCN | `home/jar/<svc>.inf` | Session_mgmt, Notify_mgmt, ulog_mgmt, Database_Backup, NeConfig_Backup, **UserManagement·SystemManagement의 서비스 고유 Worker/Schedule** |
| manualWorkerLoader | `manualWorkerLoader[_Ex]_<Svc>` | em_emulator, Report_mgmt, update_service, UserManagement·SystemManagement(framework 공통 worker만) |
| ServerConfig / 프레임워크 scan | `<Svc>ServerConfig`, plugin loader | Performance(스케줄), DBBackup_mgmt 등 혼용 |
| Function 어댑터 | `FunctionTypeBook` + Worker4ProcessFunction | 아키타입 E 전반 |

## ⚠ 죽은 / 미구현 코드 (기능 수정 시 함정)
"있는데 안 도는" 코드. 수정 요청 전에 실제 동작 여부를 먼저 확인한다.

- `File_mgmt`, `apc_mgmt` — skeleton. `processMicroServiceRestRequest()` null 반환, `WorkTypeBook` 전부 주석
- `Ne_Configuration` → `ExpressSettingProcess` — 처리 메서드 전부 주석(사실상 빈 클래스)
- `UserManagement` → `KeyCloakUserBackUpSchedule.execute()` — 본문 주석 = **no-op**(실제 백업은 `user_backup`/InstantBackup 경로)
- `Gateway_Uems` → `ScheduleKeepAlive.execute()` — 빈 메서드
- `ulog_mgmt` → `ScheduleShrinkOperationLogData`, `NeConfig_Backup` → `ScheduleShrinkNeConfigBackupData` — 구현됐으나 `.inf` 미등록 = **비활성**
- `ulog_mgmt` → `ConvertToLog_OperationLog` — null 반환 미구현
- `Session_mgmt` → `ScheduleKeepAlive` — `.inf` 선언, **소스 없음**

## 사용 순서
1. 요청이 가리키는 서비스 문서를 먼저 읽는다(`services/<service>.md`).
2. 서비스의 아키타입을 확인하고, 위 cheat-sheet에서 ②③ 등록 위치를 잡는다.
3. 5-seam 중 추가 지점을 정하고, 공통영역 수정이면 ⑤ product override로 대체 가능한지 먼저 본다.
4. 변경 전 `architecture.md`의 공통영역 변경 원칙과 real-write 위험(`db-and-migration.md`, `external-adapters.md`, `scheduler.md`)을 확인한다.
