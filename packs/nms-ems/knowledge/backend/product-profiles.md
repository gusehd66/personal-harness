# Product Profile 지식

## 기본 profile
backend 하네스는 기본적으로 전체 product profile을 대상으로 한다.

- `my5G`
- `openhaul`
- `sixgen-ems`

## profile별 확인 항목
각 product profile마다 아래를 확인한다.

- start project
- `ServerStart_*`
- service name
- manual worker loader
- product/service `ServerConfig`
- REST content book
- request type 정의
- worker/process/function 목록
- DB mapper와 migration
- external adapter
- scheduler
- product-specific runtime branch

## my5G
주요 후보:
- `Project_start_systemmanagement`
- `Project_start_usermanagement`
- `Project_start_reportmgmt`
- `Project_start_NOC`
- `Product_my5g`
- `my5g.Service.vran_ems`
- `my5g.Service.wifi`
- `my5GBox`

주요 product domain 확인 대상:
- `<BACKEND_ROOT>/product/my5G/my5g.Service.vran_ems/src/main/java/hfr/plugin/my5g/service/vran_ems/ServiceStart_vran_ems.java`
- `<BACKEND_ROOT>/product/my5G/my5g.Service.vran_ems/src/main/java/hfr/plugin/my5g/service/vran_ems/vRan_emsServerConfig.java`
- `<BACKEND_ROOT>/product/my5G/my5g.Service.vran_ems/src/main/java/hfr/plugin/my5g/service/vran_ems/function/RestInvokerFunction_vran_ems.java`
- `<BACKEND_ROOT>/product/my5G/my5g.Service.wifi/src/main/java/hfr/plugin/my5g/service/wifi/ServerConfig4wifi.java`

기록 대상:
- vRAN, CU, DU, RU, vCore, WiFi/APC 모델
- `FunctionDiscovery*`
- `FunctionAlarmDataSync*`
- `FunctionPerformance*`, `FunctionPerformanceMonitoring*`
- `Worker4baseNe*`, `Worker4Performance*`
- NETCONF/O1/vCore notification parser

## openhaul
주요 후보:
- `Project_start_openHaul`
- `Project_start_openHaul_pf_debug`
- `my5g.Service.openHaul`

주요 product domain 확인 대상:
- `<BACKEND_ROOT>/product/openhaul/my5g.Service.openHaul/src/main/java/hfr/plugin/my5g/service/openHaul_ems/ServiceStart_openHaul_ems.java`
- `<BACKEND_ROOT>/product/openhaul/my5g.Service.openHaul/src/main/java/hfr/plugin/my5g/service/openHaul_ems/openHaul_emsServerConfig.java`
- `<BACKEND_ROOT>/product/openhaul/my5g.Service.openHaul/src/main/java/hfr/plugin/my5g/service/openHaul_ems/function/RestInvokerFunction_openHaul_ems.java`

기록 대상:
- openHaul, FSH/FSH2 모델
- `FunctionDiscovery_openHaul`, `FunctionDiscoveryIP_openHaul`, `FunctionDiscoveryRefresh_openHaul`
- `FunctionAlarmDataSync_openHaul`, `FunctionAlarmDataSyncFSH2`
- `FunctionPerformance4NeOpenHaul`
- `FunctionShelfViewOpenHaul`, `ShelfViewNeProcessFSH`, `ShelfViewNeProcessOpenHaul`
- `Worker4baseNeopenHaul`, `WorkerRestAPI_openHaul`

## sixgen-ems
주요 후보:
- `Project_start_six_gen`
- `my5g.Service.six_gen_flexible`

주요 product domain 확인 대상:
- `<BACKEND_ROOT>/product/sixgen-ems/my5g.Service.six_gen_flexible/src/main/java/hfr/plugin/my5g/service/six_gen_flexible/ServiceStart_six_gen_flexible.java`
- `<BACKEND_ROOT>/product/sixgen-ems/my5g.Service.six_gen_flexible/src/main/java/hfr/plugin/my5g/service/six_gen_flexible/SixGenFlexibleServerConfig.java`
- `<BACKEND_ROOT>/product/sixgen-ems/my5g.Service.six_gen_flexible/src/main/java/hfr/plugin/my5g/service/six_gen_flexible/function/RestInvokerFunction_six_gen_flexible.java`

기록 대상:
- six-gen-flexible 모델
- `FunctionDiscovery_six_gen`, `FunctionDiscoveryIP_six_gen`, `FunctionDiscoveryRefresh_six_gen`
- `FunctionAlarmDataSyncSixGen`
- `FunctionPerformance4NeSixGenFlexible`
- `FunctionShelfViewSixGen`, `ShelfViewNeProcessSixGen`
- `Worker4baseNeSixGen`, `WorkerRestAPI_six_gen`
- `NotificationParser_sixGen_netconf`

## 하네스 기준
product profile fixture는 아래를 포함한다.

- product name
- service name
- start class
- run mode
- manual worker loader
- server config
- rest content source
- key request type
- 주요 worker/process
- DB/external/scheduler 위험 지점

## 공통영역 영향 확인
product profile 분석 시 공통영역 변경 후보가 나오면 아래를 반드시 확인한다.

- 해당 변경이 `BuildCommon`, 공통 `library`, `Platform_framework`, `uNMS_Common`에 들어가는가?
- my5G, openhaul, sixgen-ems 중 어떤 product가 같은 class/method/config를 사용하는가?
- product-specific class에서 상속 또는 override로 분리할 수 있는가?
- product-specific `ServerConfig`, `RestContentBook`, `WorkerRestContentManager`, worker/function으로 대체 가능한가?
- 공통영역 변경 없이 adapter wrapper나 fixture/mock 교체로 해결 가능한가?
- 공통영역 변경이 불가피하면 기존 product의 default behavior가 유지되는가?

## Performance/PM product profile 확인
Performance 기능은 product별 장비, 파일명, parser, Mongo model, category/table mapping 차이가 크므로 profile별로 아래를 별도 확인한다.

- 실시간 PM: product별 `AbsFunctionPerformanceMonitoringOx` 구현과 `performanceMonitoring` function 등록 여부
- FTP 파일 PM: product별 `AbsFunctionPerformance4Ne` 구현, 파일명 판별, `getCsvFiles4Performance(...)`, `parsePmFile(...)`
- 저장 경로: product별 `WorkerPerformance` 또는 `Worker4Performance`, Mongo collection, MyBatis mapper statement
- 공통 변경 회피: `PMStatisticsSchedulerJob`, `NeStatisticsScheduleJob`, `AbsWorker4Performance`, `AbsFunctionPerformance4Ne` 직접 변경 전에 product 단 상속/override 가능성

상세 분석 기준은 `<HARNESS_ROOT>/packs/nms-ems/knowledge/backend/services/performance.md`를 따른다.

## product domain fixture 기준
- product code와 NE model
- protocol/session 상태
- discovery 결과
- alarm/PM/notification raw input
- shelfview 상태
- product별 expected DB/Mongo/API response

## product domain 변경 위험
- product별 장비 모델 식별 실패
- 공통 서비스에서 product 전용 필드 누락
- alarm/PM parser mismatch
- shelfview slot/index mapping 오류
- protocol adapter timeout/retry 증가
