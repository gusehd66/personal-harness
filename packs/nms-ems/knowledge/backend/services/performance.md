# Performance Service

## 서비스 책임
5G NMS PM 데이터를 수집(실시간 장비 조회 / FTP CSV 파일), MongoDB에 bulk insert하고, REST로 조회·필터링·축약(Shrink)하며 monitoring/monitoring_item/monitoring_data를 CRUD하는 Performance 전담 마이크로서비스.

## 주요 입력
- scheduler fire time, NE 목록과 product code
- 실시간 장비 PM 응답 / FTP·copy directory에 업로드된 PM 파일
- product별 PM function/parser 결과
- monitoring/PM 조회 REST request, NOC↔EMS 간 `Site_Data_Sync` bypass 요청

## 주요 출력
- MongoDB PM document (`performance_data` DB, collection별 `insertMany`)
- MyBatis `loadPerformance`, `syncListNeStatistics` 결과, monitoring 계열 CRUD 응답
- PM 파일 success/fail/raw/parsed 이동 상태

## 시작점과 실행 설정
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Performance/src/main/java/hfr/plugin/my5g/service/Performance/ServiceStart_Performance.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Performance/src/main/java/hfr/plugin/my5g/service/Performance/PerformanceServerConfig.java`

확인 기준:
- `tServiceName = "performance"`, `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- MongoDB 사용: `useMongoDB=true`, db=`performance_data` (host: `mongo_db.LOCAL_DOMAIN`, port 27017)
- MariaDB DNS = `platformRatCD.SupportService.maria_db` + `LOCAL_DOMAIN`
- `PerformanceServerConfig extends ServerConfigOnlyDB`: `getMapperList(...)` → `unify_performance.getMapperList(...)` 위임(6개 XML), `processDBUpgrade(...)` 시 `/hfr/plugin/my5g/service/Performance/updatesql/update.sql` 실행

## 아키타입
**A(unify_performance 자동등록) + E(SFTP/장비 외부연동)** — `../feature-extension.md` 참조.
- ② Worker 등록: `unify_performance.java`의 *Auto Generated Code* 영역 (`toDataRequestTypeListAuto()`, `toWorker4Auto()`, `toDataClass4Auto()`, `getMapperList()`)
- ③ Scheduler 등록: ServerConfig/plugin 등록 구조(WorkflowMap) — cheat-sheet의 "ServerConfig/프레임워크 scan" 유형
- E형 확장: `FunctionTypeBook.performanceMonitoring` / `FunctionTypeBook.funtion_sftp` 코드 등록 시 스케줄러가 자동 수집

## REST contract
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Performance/src/main/java/hfr/plugin/my5g/service/Performance/function/RestInvokerFunction_Performance.java` (getContentList)

request type:
- `collection` (timeout 5000), `performance`·`monitoring`·`monitoring_item`·`monitoring_data` (ScopeTypeBook.Inner), `monitoring_data_ne`
- `Performance_Shrink`, `Performance_Data`, `Network_Slice_PM`, `Request_Oss_data`, `Site_Data_Sync`
- unify 자동등록 경로(commonData): `ems_db_update_info`, `pm_index` (RestContent 목록이 아닌 `toDataRequestTypeListAuto()` 등록으로 처리)

`Site_Data_Sync`는 일반 분기 외에 bypass 처리 경로가 있다: `processBypassRequestHandler(...)` / `processBypassResponseHandler(...)`에서 request type 매칭 — uNMS(NOC)에서 PM 데이터 Sync를 수신하는 경로. 본문 상당수가 주석 처리되어 있으므로 수정 전 실제 활성 로직 범위를 먼저 확인한다.

## 주요 Worker/Function 흐름
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Performance/src/main/java/hfr/plugin/my5g/service/Performance/worker/WorkerPerformance.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Performance/src/main/java/hfr/plugin/my5g/service/Performance/function/PmDataProcessorFunction.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Performance/src/main/java/hfr/plugin/my5g/service/Performance/common/worker/unify_performance/unify_performance.java`

기록 기준:
- `WorkerPerformance`: mongoDBClient를 통한 PM collection `insertMany/queryList/queryAggregate/deleteMany`. Collection 이름은 `AbsMongoDBData.getCollectionName()`으로 결정(`PerformanceCD.getPerformanceTypeClassList()` 기반 동적 매핑).
- unify Worker: `WorkerMonitoring`, `WorkerMonitoring_item`, `WorkerMonitoring_data`, `WorkerEms_db_update_info`, `WorkerPm_index`, `WorkerSync_state` — 각각 `mybatis_*.xml` mapper로 CRUD. `processCommonData()` 범용 분기가 `toWorker4Auto()`/`toDataClass4Auto()` 반환값으로 자동 라우팅한다.
- `PmDataProcessorFunction`: **NOC 모드에서만**(`m_WorkerLocalSystemData.isNOC()` true 분기) `PMInsertWorker` 스레드를 2개 구동해 PM insert queue를 소비한다. EMS 모드에서는 이 스레드가 뜨지 않는다(`!isNOC()` 분기 별도).
- Plugin 추상화: `AbsPerformance_Plugin` → `Plugin_Performance`(본 서비스 내 기본 구현).

## DB/Mongo/파일 흐름
- MongoDB `performance_data`: PM collection에 class name 그룹 단위 `insertMany`. Shrink는 `WorkerShrinkDataStorage`가 PM 전체 collection 대상 삭제 실행.
- MyBatis(MariaDB): `unify_performance` 하위 6개 mapper — `mybatis_monitoring.xml`, `mybatis_monitoring_item.xml`, `mybatis_monitoring_data.xml`, `mybatis_pm_index.xml`, `mybatis_sync_state.xml`, `mybatis_ems_db_update_info.xml` (`<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/mybatis/unify_performance/`)
- 파일: PM CSV → 파싱 후 `getPmSuccessDir()`/`getPmFailDir()` 이동.

## scheduler/queue/background 흐름
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Performance/src/main/java/hfr/plugin/my5g/service/Performance/schedule/` 하위
- `PMStatisticsSchedulerJob.java` — 실시간 장비 조회 PM 수집
- `NeStatisticsScheduleJob.java` — FTP 업로드 파일 파싱 PM 수집
- `ScheduleShrinkPerformanceData.java` — PM 데이터 축약(삭제)
- `Download2SFTPScheduleJob.java` — SFTP 파일 다운로드

### 실시간 장비 조회 PM 수집 (`PMStatisticsSchedulerJob`)
1. `initialize(...)`에서 `WorkerLocalBaseDataCommon`, `WorkerPerformance`, `Worker4ProcessFunction`을 `WorkflowMap`에서 가져온다.
2. `FunctionTypeBook.performanceMonitoring`으로 등록된 `AbsFunctionPerformanceMonitoringOx` 목록을 product code(`getEmsTypecode`) 기준 `m_ProcessPerformanceFunctionMap`으로 구성한다.
3. `execute(...)`에서 전체 NE 목록을 가져와 `MCirculator`에 NE별 work item을 넣고, `runningNe` + lock으로 동일 NE 중복 수집을 막는다.
   - `runningNe`는 `putIfAbsent(...)` 성공 작업만 `finally`에서 `remove(...)`해야 한다. skip 경로가 무조건 remove하면 실행 중인 다른 NE 작업의 플래그를 지워 동일 NE 동시 수집을 허용한다.
4. `calculateDelaySeconds(...)`: NE IPv4 끝자리 기반 1~5초 delay를 계산해 장비 수집 부하를 분산한다.
5. product별 `processCreatePerformance(context, aNe)`가 장비 현재 PM을 읽는다.
6. `AbsMongoDBPerformance.getPmMemoryKey(...)` 기준으로 1분 데이터를 메모리 누적, 이전 15분 구간을 insert 대상으로 확정한다.
7. insert 대상을 class name 기준 그룹화 → `WorkerPerformance.insertBulkData(...)` → collection 단위 `insertMany(...)`.

### FTP 업로드 파일 파싱 PM 수집 (`NeStatisticsScheduleJob`)
실행 설정: `isAutoRun=true`, `getInterval=30000ms`. 파티션 집계는 `UtilPartition` 파라미터(기본 `MINUTE_15`, 파티션 카운트) 기준.
1. `initialize(...)`에서 `WorkerPerformance`를 가져오고 parser thread pool(`PerformanceThread`)을 시작한다.
2. `execute(...)`에서 `ConfigUtil.checkPauseOption(E_DebugPauseOption.NONE_PM)`으로 PM pause 상태를 먼저 확인한다.
3. `m_WorkerPerformance.getCsvFiles4Performance(context, schedule_time)`가 product별 `AbsFunctionPerformance4Ne.getCsvFiles4Performance(...)`를 모아온다.
4. 파일명 기준 `m_PmTargetListMap`으로 중복 queue 투입을 막는다(메모리 기반 — 재시작 시 중복 파싱 가능).
5. `PerformanceThread`가 queue에서 파일을 꺼내 `PerformanceCD.findPmFuncFileName(csvFile)`로 product별 parser를 선택한다.
6. `AbsFunctionPerformance4Ne.parsePmFile(IWorkerPerformance, File)` 파싱 → worker 저장 API 호출 → 성공/실패에 따라 success/fail dir 이동.
7. legacy/product별 구현에서는 `parseCsv2Performance(...)`, `loadPerformance(...)`, `syncListNeStatistics(...)`가 MyBatis statement로 이어진다.

### Shrink (`ScheduleShrinkPerformanceData`)
`WorkerShrinkDataStorage`를 WorkflowMap에서 가져와 `runShrink_PMHistory`/`isDBShrikRunOK`로 MongoDB PM collection 대상 삭제를 실행한다. shrink 기준 설정 오류 시 데이터 전량 삭제 위험.

### SFTP 다운로드 (`Download2SFTPScheduleJob`)
`initialize()`가 `tWorkerProcessFunction.getFunctionList(FunctionTypeBook.funtion_sftp)`로 product별 `AbsFunctionSFTP` 구현체를 자동 수집해 NE별 map을 구성한다. **funtion_sftp 코드 미등록 시 조용히 skip된다.**

## 기능 확장점 (feature seams)
| seam 유형 | 추가 방법 | 등록 위치/근거 파일 |
| --- | --- | --- |
| ① rest-request-type | `getContentList()`에 `new RestContent(this, "새_type", HttpMethodBook.POST, getServiceName(), "v1")` 추가 + `processMicroServiceRestRequest()` else-if 분기 추가. Worker 필요 시 `WorkflowMap.getWorker()` | `function/RestInvokerFunction_Performance.java` |
| ② worker-function (아키타입 A) | `unify_performance.java` Auto Generated Code 영역의 `toDataRequestTypeListAuto()`/`toWorker4Auto()`/`toDataClass4Auto()`/`getDataFieldList()`/`getMapperList()`에 등록 → `processCommonData()` 범용 CRUD 자동 라우팅. 새 Worker는 `common/worker/unify_performance/` 하위, mapper XML은 `uEMS.plugin.my5g.Common`의 mybatis 폴더 | `common/worker/unify_performance/unify_performance.java` |
| ⑤ product-override (실시간 PM) | `AbsFunctionPerformanceMonitoringOx` 상속 Function을 product 서비스에 추가 + `FunctionTypeBook.performanceMonitoring` 코드 등록 → `PMStatisticsSchedulerJob.initialize()`가 자동 수집 | `schedule/PMStatisticsSchedulerJob.java` |
| ⑤ product-override (FTP PM) | `AbsFunctionPerformance4Ne` 상속 + `PerformanceCD`에 파일명 매핑 등록 → `NeStatisticsScheduleJob`의 `PerformanceThread`가 `findPmFuncFileName()`으로 자동 선택 | `schedule/NeStatisticsScheduleJob.java`, `uEMS.plugin.my5g.Common/.../unify_performance/_manual/PerformanceCD.java` |
| data-model (Mongo collection) | `AbsMongoDBPerformance` 상속 클래스를 `uEMS.plugin.my5g.Common`의 mongodb/performance 패키지에 추가 + `PerformanceCD.getPerformanceTypeClassList()` 등록 → `toDataClassMongoDB()`가 반환, `WorkerPerformance` insert/query 자동 동작 | `unify_performance.java` (toDataClassMongoDB), `PerformanceCD.java` |
| ④ db-mapper (MariaDB) | `uEMS.plugin.my5g.Common/.../mybatis/unify_performance/mybatis_XXX.xml` 추가 → `unify_performance.MybatisMapperBook`에 코드 추가 → `getMapperList()` 등록 → `PerformanceServerConfig.getMapperList()`가 위임하므로 자동 적재 | `unify_performance.java`, `PerformanceServerConfig.java` |
| ③ scheduler-job | `ScheduleAdapterRat` 상속 클래스를 `schedule/` 아래 추가, WorkflowMap에 등록. `ScheduleShrinkPerformanceData`처럼 Worker를 WorkflowMap에서 가져와 쓰는 패턴을 따른다 | `schedule/ScheduleShrinkPerformanceData.java` |
| external-adapter (SFTP, 아키타입 E) | `AbsFunctionSFTP` 상속 product function을 `FunctionTypeBook.funtion_sftp` 코드로 등록 → `Download2SFTPScheduleJob.initialize()` 자동 수집 | `schedule/Download2SFTPScheduleJob.java` |

## product 확장점
product별 구현체(공통영역 무수정 확장 예):
- vRAN: `FunctionPerformance4NeCUDU`, `FunctionPerformance4NeVcore`, `FunctionPerformanceMonitoringvRAN` (`<BACKEND_ROOT>/product/my5G/my5g.Service.vran_ems/...`)
- wifi: `FunctionPerformanceMonitoringWifi`, `Worker4Performance_WiFi` (`<BACKEND_ROOT>/product/my5G/my5g.Service.wifi/...`)
- openHaul: `FunctionPerformance4NeOpenHaul` (`<BACKEND_ROOT>/product/openhaul/my5g.Service.openHaul/...`)
- sixgen: `FunctionPerformance4NeSixGenFlexible` (`<BACKEND_ROOT>/product/sixgen-ems/my5g.Service.six_gen_flexible/...`)

## 죽은/미구현 코드
- `worker/Worker_Performance_Temp.java` — 존재하나 unify Auto 등록 대상이 아닌 임시 클래스. 기능 추가 대상으로 삼지 말 것.
- `RestInvokerFunction_Performance`의 `Site_Data_Sync` 처리 본문과 `processBypassResponseHandler` 내부 상당 부분이 주석 처리 상태 — 수정 전 실제 활성 코드 범위를 확인한다.

## mock/fake 지점
real 장비, real FTP, real DB/Mongo write는 실행하지 않는다. 우선 mock/fake:
1. `WorkerPerformance.insertBulkData(ArrayList<AbsMongoDBData>)` / `IWorkerPerformance`
2. `mongoDBClient.insertMany / queryList / queryAggregate`
3. `AbsFunctionPerformanceMonitoringOx.processCreatePerformance(context, aNe)`
4. `AbsFunctionPerformance4Ne.parsePmFile(...)` / `getCsvFiles4Performance(...)`
5. `PerformanceCD.findPmFuncFileName(csvFile)` (파일명 → function 매핑)
6. `Worker4ProcessFunction.getFunctionList(FunctionTypeBook.performanceMonitoring)`
7. `WorkerShrinkDataStorage.runShrink_PMHistory / isDBShrikRunOK`
8. `AbsFunctionSFTP` (SFTP 다운로드), 파일 시스템 이동 API

## fixture 기준
실시간 PM: scheduler fire time / NE 목록·product code·IP·session 상태 / 등록된 performanceMonitoring function map / 장비 응답 mock / 이전 `PmMinuteData` memory state / expected 15분 집계·Mongo documents / bulk insert 성공·일부 실패·전체 실패 case.

FTP 파일 PM: scheduler fire time / PM pause option / directory 파일 목록 / 파일명-parser 매칭 / parser 입력 내용 / raw·success·fail·parsed 이동 기대 상태 / expected Mongo documents 또는 `loadPerformance` 입력 / expected `syncListNeStatistics` from/to type·partition index / 파일 중복·parser 미발견·권한 오류·DB session 오류 case.

## 변경 위험
- PM 중복 수집 또는 누락 — `PmMinuteData` 메모리 key 변경 시 15분 집계 손상
- Mongo bulk `insertMany` 일부 실패 시 데이터 유실(예외 전파 방식 확인)
- `PerformanceCD.getPerformanceTypeClassList()` 등록 누락 → 특정 collection이 조회·수집 대상에서 제외
- class grouping 오류 → 서로 다른 Mongo collection 데이터가 같은 `insertMany`에 섞임
- `m_PmTargetListMap`이 메모리 기반 → 프로세스 재시작 시 FTP 파일 중복 파싱
- SFTP function 미등록(funtion_sftp 코드 누락) 시 `Download2SFTPScheduleJob` 조용히 skip
- unify Auto Generated Code 미수정 상태로 새 Worker만 추가 시 `processCommonData()`에서 null 반환 → NullPointerException
- Shrink 기준 설정 오류 시 MongoDB PM 전체 collection 대상 데이터 전량 삭제 위험
- 분/시/일/월 통계 `syncListNeStatistics(...)` 중복 key 또는 partition index 오류

## 근거 파일
서비스 루트 = `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Performance/src/main/java/hfr/plugin/my5g/service/Performance/`
- `ServiceStart_Performance.java`, `PerformanceServerConfig.java`
- `function/RestInvokerFunction_Performance.java`, `function/PmDataProcessorFunction.java`
- `common/worker/unify_performance/unify_performance.java`, `common/abs/AbsPerformance_Plugin.java`
- `worker/WorkerPerformance.java`, `worker/Worker_Performance_Temp.java`
- `schedule/PMStatisticsSchedulerJob.java`, `schedule/NeStatisticsScheduleJob.java`, `schedule/ScheduleShrinkPerformanceData.java`, `schedule/Download2SFTPScheduleJob.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/unify_performance/_manual/PerformanceCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/mybatis/unify_performance/mybatis_monitoring.xml` (외 5개 XML)
