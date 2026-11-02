# NeConfig_Backup Service

## 서비스 책임
NE 설정 백업(Config Backup)·복원(Config Restore) 요청을 수신해 제품 타입별 southbound 서비스로 라우팅하고, 백업 파일을 파일 시스템에 저장·복원하며, 오래된 백업 파일/DB 레코드를 주기 삭제(shrink)하는 support 서비스. 백업 이력 DB의 실제 CRUD 주체는 `NeBackupRestore_mgmt`(unify_nebackup)이며, 이 서비스는 장비와의 백업/복원 실행 흐름을 담당한다.

## 주요 입력
- REST 백업/복원 요청 (NE id, 백업 파일 정보)
- NE 모델 코드(`WorkerLocalBaseDataCommon.requestNeInfo()`), FTP 서버 설정
- 파일 시스템의 기존 백업 파일(복원 시)

## 주요 출력
- southbound product 서비스로의 REST 위임(`WorkerRestContentManager.executeRestAPI`)
- 파일 시스템 백업 파일(`BackupData.save()`)
- shrink에 의한 백업 파일/DB 레코드 삭제

## 시작점과 실행 설정
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeConfig_Backup/src/main/java/hfr/plugin/my5g/service/NeConfig_Backup/ServiceStart_NeConfig_Backup.java`
- `tServiceName = "neconfig_backup"`, `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- DB: MyBatis(MariaDB, unify_base 공통), 로컬 mapper XML 없음. MongoDB 미사용.
- 컴포넌트 등록: `home/jar/neconfig_backup.inf`의 FQCN 12줄을 `loadStaticClass`가 자동 로드 — 공통 Worker 10개 + `RestInvokerFunction_NeConfig_Backup` + `Worker_NeConfig_Backup`

## 아키타입
**B(코디네이터/southbound 위임) + C(.inf 플러그인 로더)** — `../feature-extension.md` 참조.
- ② Worker 등록: `home/jar/neconfig_backup.inf`에 FQCN 한 줄 추가 (C형). 빠뜨리면 클래스를 만들어도 로딩 안 됨.
- ③ Scheduler 등록: 동일하게 `.inf` FQCN 등록. **`ScheduleShrinkNeConfigBackupData`는 현재 `.inf`에 미등록 = 비활성** (아래 죽은 코드 참조).
- B형 특성: 직접 DB CRUD 없이 `m_WorkerRestContentManager.executeRestAPI(...)`로 product 서비스에 위임.

## REST contract
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeConfig_Backup/src/main/java/hfr/plugin/my5g/service/NeConfig_Backup/function/RestInvokerFunction_NeConfig_Backup.java` (getContentList line 168~)

`getContentList()` 등록 request type 3개 (상수는 `MicroServiceCommonCD.RequestTypeStringBook`):
- `request-neconfig-backup` (`request_NeConfig_backup`) — 처리됨 (line 130 분기)
- `request-neconfig-restore` (`request_NeConfig_restore`) — 처리됨 (line 136 분기)
- `request-neconfig-detail` (`request_NeConfig_Detail`) — **등록만 되고 `processMicroServiceRestRequest()`에 처리 분기 없음** → null response. 실제 detail 처리자는 `NeBackupRestore_mgmt`.

backup/restore 공통 전처리(line 80~): FTP 설정 획득 → `isUseFTP(ne_id)`인 경우에만 null 체크 후 오류 응답.

## 주요 Worker/Function 흐름
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeConfig_Backup/src/main/java/hfr/plugin/my5g/service/NeConfig_Backup/worker/Worker_NeConfig_Backup.java`
- `execute_NeConfigBackup()`: NE 모델 확인 → southbound product 서비스에 `executeRestAPI()` 위임 → DAS NE(N_DAS_TU/SU/RU, line 128~137)는 응답의 백업 데이터를 `BackupData`로 파일 저장.
- `execute_NeConfigRestore()`: DAS NE(line 188~192)는 저장된 백업 파일을 읽어 요청 payload에 포함해 전송, 그 외 NE는 FTP 경유 일반 경로.
- `isUseFTP(neID)` (line 287~): DAS NE 모델이면 false — FTP 미사용 특수 경로 판별.
- southbound 대상 서비스명은 `productCD.ProductTypeCode`에서 NE product code로 결정 — 새 product 라우팅은 여기 등록 필요.
- 요청 payload 모델: `common/data/NEConfigBackupRequestData.java`(로컬). 공통 백업 데이터 모델은 `uEMS.plugin.my5g.Common`의 `NeConfigBackupData`(unify_nebackup/_manual).

## DB/Mongo/파일 흐름
- MyBatis(unify_base) 공통 접속만 사용, 서비스 로컬 mapper 없음.
- 백업 파일: 파일 시스템에 `BackupData.save()`로 저장(DAS 경로), 복원 시 파일 로드.
- DB shrink: `WorkerShrinkDataStorage.runShrink_ScheduleConfigBackup()` (`E_DBShrink_Type.ScheduleConfigBackup`) — `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/worker/WorkerShrinkDataStorage.java`

## scheduler 흐름
- `schedule/ScheduleShrinkNeConfigBackupData.java`: `WorkerShrinkDataStorage`로 오래된 백업 파일·DB 레코드 삭제(`isFileShrikRunOK`/`isDBShrikRunOK` 조건 확인 후 실행). **단, `.inf` 미등록 상태라 실제 기동되지 않는다** — shrink 기능을 살리려면 `neconfig_backup.inf`에 FQCN 한 줄 추가가 선행 조건.

## 기능 확장점 (feature seams)
| seam 유형 | 추가 방법 | 등록 위치/근거 파일 |
| --- | --- | --- |
| ① rest-request-type | `getContentList()`에 `new RestContent(this, MicroServiceCommonCD.RequestTypeStringBook.request_NeConfig_XXX, HttpMethodBook.POST, getServiceName(), "v1")` 추가 + `processMicroServiceRestRequest()`에 else-if 분기 추가 + `MicroServiceCommonCD.RequestTypeStringBook`에 상수 등록. `.inf` 수정 불필요 | `function/RestInvokerFunction_NeConfig_Backup.java` |
| ② worker-function (아키타입 B) | `Worker_NeConfig_Backup`에 public 메서드(예: `execute_NeConfigXxx()`) 추가 → RestInvoker 분기에서 `m_Worker.execute_NeConfigXxx()` 호출. southbound 위임은 `WorkerRestContentManager.executeRestAPI()` — 대상 serviceName이 `productCD.ProductTypeCode`에 등록돼 있어야 함 | `worker/Worker_NeConfig_Backup.java` |
| ② 신규 Worker (아키타입 C) | 새 Worker 클래스 작성 후 `home/jar/neconfig_backup.inf`에 FQCN 한 줄 추가 → `loadStaticClass` 자동 등록 | `home/jar/neconfig_backup.inf` |
| ③ scheduler-job (아키타입 C) | `schedule/`에 `ScheduleAdapterRat` 상속 Job 작성 + `.inf`에 FQCN 추가. `ScheduleShrinkNeConfigBackupData`가 미등록 선례이므로 등록 여부를 반드시 확인 | `home/jar/neconfig_backup.inf`, `schedule/ScheduleShrinkNeConfigBackupData.java` |
| ⑤ product-override | 새 NE 모델의 특수 경로는 `Worker_NeConfig_Backup.isUseFTP()`와 `execute_NeConfigBackup/Restore()`의 `NeCodeBook.N_XXX` 조건문에 추가. FTP 설정 소스 분기는 RestInvoker의 `Server_run_mode` 조건(line 98~)에 새 모드 추가 | `worker/Worker_NeConfig_Backup.java`, `function/RestInvokerFunction_NeConfig_Backup.java` |
| data-model | southbound 요청 payload 확장은 로컬 `NEConfigBackupRequestData`에 필드 추가. 공통 백업 모델 필드는 `uEMS.plugin.my5g.Common`의 `NeConfigBackupData`(unify_nebackup/_manual) 수정 — 공통영역이므로 side effect 확인 | `common/data/NEConfigBackupRequestData.java` |
| external-adapter | 새 파일 전송 프로토콜(SFTP 등)은 두 지점만 수정: RestInvoker의 FTP 설정 획득 블록(line 98~116) + `Worker_NeConfig_Backup`의 백업 저장/복원 전송 경로. 나머지 흐름 변경 불필요 | `function/RestInvokerFunction_NeConfig_Backup.java`, `worker/Worker_NeConfig_Backup.java` |

## product 확장점
별도 product 폴더나 Worker 하위 클래스 구조 없음 — product 분기는 Worker 내 `NeCodeBook` 조건문으로 처리.
- DAS NE(N_DAS_TU/SU/RU): FTP 미사용(`isUseFTP()=false`), 백업 데이터를 파일로 직접 저장/로드하는 특수 경로.
- BareMetal 모드(`Server_run_mode == ServerRunModeBook.BareMetal`, openhaul 등): `workerLocalSystemData.getFTPServerConfig()` 사용. 그 외 모드: `FTPServerConfigProvider.getFTPServerConfig()` (line 98~103).

## 죽은/미구현 코드
- `ScheduleShrinkNeConfigBackupData` — 구현됐으나 `neconfig_backup.inf` 미등록 = **비활성**. shrink 수정 요청 시 등록부터 확인.
- `request_NeConfig_Detail` — `getContentList()` 등록만 있고 처리 분기 없음 = null response. 이 서비스에서 미구현이며 실제 처리자는 `NeBackupRestore_mgmt`.
- RestInvoker line 90~92, 106~108, 120~121 — FTP 설정 획득 대안 경로가 주석 처리 상태. 수정 전 활성 코드 범위 확인.

## mock/fake 지점
real 장비, real FTP, real DB write는 실행하지 않는다.
1. `WorkerRestContentManager.executeRestAPI()` — southbound 응답 mock (DAS/FHM 분기 테스트)
2. `WorkerLocalBaseDataCommon.requestNeInfo()` — NE 모델 코드 mock (DAS/비DAS 경로 분기)
3. `FTPServerConfigProvider.getFTPServerConfig()` — null/non-null 반환 mock (오류 경로)
4. `WorkerShrinkDataStorage.isFileShrikRunOK()/isDBShrikRunOK()` — shrink 실행 조건 제어
5. `BackupData.save()` / 파일 로드 — 파일 시스템 I/O mock

## 변경 위험
- `.inf` 등록 누락 — 새 Worker/Schedule을 만들어도 로딩 안 됨 (`ScheduleShrinkNeConfigBackupData`가 실제 사례)
- `request_NeConfig_Detail`처럼 getContentList 등록과 처리 분기의 drift → null response
- DAS NE 판별이 `NeCodeBook` 상수 비교 하드코딩 — 새 DAS 모델 추가 시 3곳(backup/restore/isUseFTP) 모두 갱신해야 하며 누락되기 쉬움
- FTPServerConfig null 체크가 `isUseFTP()=true`일 때만 수행 — DAS NE 경로에서 `aFtpServerConfig`가 null로 Worker에 전달될 수 있음
- 공통 모델 `NeConfigBackupData`(uEMS.plugin.my5g.Common) 수정 시 `NeBackupRestore_mgmt` 등 타 서비스 side effect

## 근거 파일
서비스 루트 = `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeConfig_Backup/`
- `src/main/java/hfr/plugin/my5g/service/NeConfig_Backup/ServiceStart_NeConfig_Backup.java`
- `src/main/java/hfr/plugin/my5g/service/NeConfig_Backup/function/RestInvokerFunction_NeConfig_Backup.java`
- `src/main/java/hfr/plugin/my5g/service/NeConfig_Backup/worker/Worker_NeConfig_Backup.java`
- `src/main/java/hfr/plugin/my5g/service/NeConfig_Backup/schedule/ScheduleShrinkNeConfigBackupData.java`
- `src/main/java/hfr/plugin/my5g/service/NeConfig_Backup/common/data/NEConfigBackupRequestData.java`
- `home/jar/neconfig_backup.inf`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/worker/WorkerShrinkDataStorage.java`
