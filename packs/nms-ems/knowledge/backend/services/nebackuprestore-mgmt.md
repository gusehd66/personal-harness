# NeBackupRestore_mgmt Service

## 서비스 책임
NE(장비) 설정 백업/복원 요청 처리, 백업 파일 CRUD, 백업 스케줄 관리, 백업 이력 기록을 담당한다.
- 주요 입력: NE 백업/복원 REST request, 백업 스케줄 CRUD request, 로컬 config 파일 업로드(InputStream), 스케줄 fire time
- 주요 출력: EMS(vran-ems 등)로의 백업/복원 REST 위임 호출, `unify_nebackup` 스키마의 백업 파일/스케줄/이력 레코드, 백업 파일 다운로드 응답

## 시작점과 실행 설정
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeBackupRestore_mgmt/src/main/java/hfr/plugin/my5g/service/NeBackupRestore_mgmt/ServiceStart_NeBackupRestore_mgmt.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeBackupRestore_mgmt/src/main/java/hfr/plugin/my5g/service/NeBackupRestore_mgmt/NeBackupRestoreServerConfig.java`

확인 기준:
- `tServiceName = "nebackuprestore_mgmt"`
- `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- DB는 MariaDB `unify_nebackup` 스키마. `NeBackupRestoreServerConfig`(ServerConfigOnlyDB 상속)의 `getMapperList()/getJavaTypeList()`가 `unify_nebackup` 팩토리에 위임
- `processDBUpgrade()`가 classpath 내 `/hfr/plugin/my5g/service/NeBackupRestore_mgmt/updatesql/update.sql` 실행

## 아키타입
**A(unify 자동등록) + C 성격 일부** — `../feature-extension.md` 기준 A가 주 아키타입.
②③ 등록 위치:
- ② Worker: `unify_nebackup.java`의 Auto Generated Code 영역(`toDataRequestTypeListAuto()`, `toWorker4Auto()`, `toDataClass4Auto()`) + `Worker_NeBackupRestore_mgmt.initialize()`의 `m_WorkflowMap.getWorker(...)` 연결
- ③ Scheduler: `ScheduleNeConfigBackup`(ScheduleAdapterRat 상속) — WorkflowMap에 schedule job으로 등록

## REST contract
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeBackupRestore_mgmt/src/main/java/hfr/plugin/my5g/service/NeBackupRestore_mgmt/function/RestInvokerFunction_NeBackupRestore_mgmt.java`

request type 상수 정의 위치: `MicroServiceCommonCD.RequestTypeStringBook`(`<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java`) + unify 엔티티별 `requestType` 상수(`common/data/unify_nebackup/<entity>/<Entity>.java`). `Change_ne_restore`는 `platformRatCD.java`에 `"restore"`로 정의.

| request type | 상수 | 동작 |
| --- | --- | --- |
| `request-neconfig-backup` | `request_NeConfig_backup` | 단일 NE 설정 백업 |
| `request-neconfig-allbackup` | `request_NeConfig_allbackup` | 전체 설정 백업 |
| `request-neconfig-restore` | `request_NeConfig_restore` | 백업 파일 복원 |
| `request-neconfig-restore-external` | `request_NeConfig_restore_external` | 외부 파일 복원 |
| `request-neconfig-detail` | `request_NeConfig_Detail` | 백업 상세 조회 |
| `request-neconfig-download` | `request_NeConfig_download` | 백업 파일 다운로드 |
| `ne-configbackup-schedule` | `ne_configbackup_schedule` | 백업 스케줄 CRUD(WorkTypeBook Get/Add/Modify/Delete/GetList 분기) |
| `schedule-config-backup-shrink` | `ScheduleConfigBackup_Shrink` | 백업 데이터 shrink 스케줄 |
| `shrink-neconfig-file` | `shrink_NeConfig_file` | 백업 파일 shrink |
| `local-neconfig-upload` | `local_NeConfig_upload` | 로컬 config 업로드(InputStream 경로: `processMicroServiceRestRequestUsingInputStream()`) |
| `restore` | `Change_ne_restore` | 다른 NE 백업 파일로 복원(NE 교체) |
| `config-backup-file` | `Config_backup_file.requestType` | 백업 파일 GetList/Delete |
| `schedule` | `Schedule.requestType` | 스케줄 GetList/Delete |
| `schedule-apply` | `Schedule_apply.requestType` | 스케줄 적용 대상 GetList |
| `schedule-log` | `Schedule_log.requestType` | 스케줄 이력 GetList |
| `schedule-log-detail` | `Schedule_log_detail.requestType` | 스케줄 이력 상세 GetList |

대부분 `ScopeTypeBook.Inner`(내부 전용)로 등록된다 — 신규 type 추가 시 scope 지정을 확인한다.

## 주요 Worker/Function 흐름
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeBackupRestore_mgmt/src/main/java/hfr/plugin/my5g/service/NeBackupRestore_mgmt/worker/Worker_NeBackupRestore_mgmt.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeBackupRestore_mgmt/src/main/java/hfr/plugin/my5g/service/NeBackupRestore_mgmt/common/worker/unify_nebackup/unify_nebackup.java`

- `Worker_NeBackupRestore_mgmt`: 백업/복원 본체. EMS로의 실제 백업/복원은 `WorkerRestContentManager.executeRestAPI()`로 위임. 파일명 생성은 `getBackupFileNameGenerate()`/`getAllBackupFileNameGenerate()`(NeCodeBook 기반 if-else, 확장자 .config/.tgz/.zip/.tar.gz/.cfg 하드코딩)
- unify 엔티티 Worker(각각 AutoWorkerXxx + WorkerXxx 쌍, `common/worker/unify_nebackup/` 하위): `WorkerConfig_backup_file`, `WorkerSchedule`, `WorkerSchedule_apply`, `WorkerSchedule_log`, `WorkerSchedule_log_detail`, `WorkerUe_trace`, `WorkerUe_trace_value`
- NOC bypass: `RestInvokerFunction`의 `processBypassRequestHandler()/processBypassResponseHandler()`가 ne_id↔site_ne_id 변환 — 신규 request type이 NE id를 다루면 여기도 처리 필요

## DB/Mongo/파일 흐름
- MariaDB(MyBatis), `unify_nebackup` 스키마. MongoDB 없음.
- 매퍼 7개(공유 모듈 `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/mybatis/unify_nebackup/`): `mybatis_config_backup_file.xml`, `mybatis_schedule.xml`, `mybatis_schedule_apply.xml`, `mybatis_schedule_log.xml`, `mybatis_schedule_log_detail.xml`, `mybatis_ue_trace.xml`, `mybatis_ue_trace_value.xml`
- 타입 핸들러: `EDateTime`, `TimeSpan`, `E_Type`(config_backup_file/ue_trace), `E_Period_type`·`E_Target_type`(schedule)
- 스키마 변경은 `updatesql/update.sql` 단일 파일에 누적(ALTER/CREATE) — `processDBUpgrade()`가 기동 시 자동 실행
- 백업 파일은 FTP 서버 경로에 저장(`FTPServerConfigProvider.getFTPServerConfig()` 참조), product별 서브경로 분기 있음(아래 product 확장점)

## scheduler 흐름
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeBackupRestore_mgmt/src/main/java/hfr/plugin/my5g/service/NeBackupRestore_mgmt/schedule/ScheduleNeConfigBackup.java`

- `ScheduleNeConfigBackup`(ScheduleAdapterRat 상속): `execute()` → `processSchedule_NeConfig_Backup()`이 스케줄 대상 NE를 순회하며 백업 수행. NE 모델별 지원 여부는 NeCodeBook 비교 if 블록으로 필터.
- Quartz one-shot 재등록 방식(`ScheduleAdapterRat.manualRunOnce()`) — 매 실행 후 다음 fire를 다시 등록한다.

## 기능 확장점 (feature seams)

| seam | 추가 방법 | 등록 위치 / 근거 파일 |
| --- | --- | --- |
| ① REST request type | `processMicroServiceRestRequest()` if-else에 새 `aRequestTypeCode` 분기 추가 + `getContentList()`에 `tReturnData.add(new RestContent(this, "새-type", HttpMethodBook.POST, getServiceName(), "v1", ScopeTypeBook.Inner))` 등록 + `MicroServiceCommonCD.RequestTypeStringBook`에 상수 선언. 파일 수신이면 `processMicroServiceRestRequestUsingInputStream()`에도 분기 | `function/RestInvokerFunction_NeBackupRestore_mgmt.java` |
| ② Worker/Function | `common/worker/unify_nebackup/` 하위에 `AutoWorkerXxx`+`WorkerXxx` 쌍 추가 → `unify_nebackup.toWorker4Auto()`, `toDataRequestTypeListAuto()`에 requestType→Worker 매핑 등록 → `Worker_NeBackupRestore_mgmt.initialize()`에서 `m_WorkflowMap.getWorker(WorkerXxx.class)` 연결 | `common/worker/unify_nebackup/unify_nebackup.java` |
| ③ Scheduler job | 백업 대상 NE 확장/제한: `ScheduleNeConfigBackup.processSchedule_NeConfig_Backup()`의 NeCodeBook 필터 if 블록 수정. 완전 신규 작업: `ScheduleAdapterRat` 상속 클래스 추가 + WorkflowMap `getScheduleJob()` 등록 | `schedule/ScheduleNeConfigBackup.java` |
| ④ DB mapper | `uEMS.plugin.my5g.Common`의 `hfr/plugin/my5g/common/mybatis/unify_nebackup/`에 `mybatis_xxx.xml` 추가 + `unify_nebackup.getMapperList()`에 Ocode 키·XML 경로 등록(ServerConfig는 위임하므로 수정 불필요) + `updatesql/update.sql`에 DDL 추가 | `common/worker/unify_nebackup/unify_nebackup.java` |
| 데이터 모델 | 새 엔티티는 `uEMS.plugin.my5g.Common`의 `hfr/plugin/my5g/common/data/unify_nebackup/` 하위에 추가(엔티티에 `requestType` 상수 선언) 후 `unify_nebackup`의 `toDataRequestTypeListAuto()/toWorker4Auto()/toDataClass4Auto()/getDataFieldList()` 등록 | `common/worker/unify_nebackup/unify_nebackup.java` |
| ⑤ Product override | 백업 서브경로 필요 여부는 `productCD.getProductFunction(aNe).isCheck(uEMSCommonCD.ProductFunctionBook.isSubPathRequiredForNeConfigBackup.getCode(), aNe)`로 위임 — 새 모델은 해당 product의 `ProductFunction_*` isCheck 분기에만 추가(Worker 본체 무수정). 단, 파일명 생성은 아직 Worker 내부 if-else — 아래 변경 위험 참조 | `worker/Worker_NeBackupRestore_mgmt.java` |

## product 확장점
- `isSubPathRequiredForNeConfigBackup` 구현체:
  - `<BACKEND_ROOT>/product/my5G/my5g.Service.vran_ems/src/main/java/hfr/plugin/my5g/service/vran_ems/function/ProductFunction_vRan_ems.java`
  - `<BACKEND_ROOT>/product/my5G/my5g.Service.wifi/src/main/java/hfr/plugin/my5g/service/wifi/function/ProductFunction_WiFi.java`
- 파일명 확장자 분기는 product로 분리되지 않고 `Worker_NeBackupRestore_mgmt` 내부 NeCodeBook if-else에 남아 있다(절반만 product 분리).

## mock/fake 지점
1. `WorkerRestContentManager.executeRestAPI()` — EMS로의 실제 백업/복원 REST 호출
2. `WorkerLocalBaseDataCommon.requestNeInfo()` — NE 정보 조회
3. `FTPServerConfigProvider.getFTPServerConfig()` — FTP 서버 설정 조회
4. `ScheduleAdapterRat.manualRunOnce()` — Quartz 스케줄 등록

## 변경 위험
- **신규 NE 모델 추가 시 3곳 동시 수정 필요**: `getBackupFileNameGenerate()` + `getAllBackupFileNameGenerate()`(중복 하드코딩) + `ScheduleNeConfigBackup`의 지원 여부 필터. 하나라도 누락하면 해당 모델 백업 누락/파일명 오류.
- `ScheduleNeConfigBackup.processSchedule_NeConfig_Backup()` — 주석 처리된 코드와 활성 if 블록이 혼재, 수정 시 실수 위험.
- 신규 request type 추가 시 `processBypassRequestHandler()/processBypassResponseHandler()`의 ne_id→site_ne_id 변환 누락 가능(NOC bypass 경로).
- `update.sql` 단일 파일 관리 — 동시 수정 시 충돌 위험.

## 근거 파일
`<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeBackupRestore_mgmt/src/main/java/hfr/plugin/my5g/service/NeBackupRestore_mgmt/` 기준:
- `ServiceStart_NeBackupRestore_mgmt.java`, `NeBackupRestoreServerConfig.java`
- `function/RestInvokerFunction_NeBackupRestore_mgmt.java`
- `worker/Worker_NeBackupRestore_mgmt.java`
- `common/worker/unify_nebackup/unify_nebackup.java`
- `schedule/ScheduleNeConfigBackup.java`
- `common/worker/unify_nebackup/config_backup_file/WorkerConfig_backup_file.java`, `common/worker/unify_nebackup/ue_trace/WorkerUe_trace.java`

공유 모듈:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/mybatis/unify_nebackup/mybatis_config_backup_file.xml` (외 매퍼 6개)
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/unify_nebackup/` (엔티티·requestType 상수)
- `<BACKEND_ROOT>/Platform_framework/EMSPlatform/src/hfr/platform/Rat/product/common/data/platformRatCD.java` (`Change_ne_restore = "restore"`)
