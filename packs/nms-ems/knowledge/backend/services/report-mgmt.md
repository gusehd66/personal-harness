# Report_mgmt Service

## 서비스 책임
5G NMS의 OSS(NBI) 연동 리포트 서비스. CM/FM/PM 데이터를 타 마이크로서비스(basedata, performance, ne-configuration)에서 REST로 수집하여 CSV 리포트 파일을 생성하고, SFTP로 OSS/NBI 서버에 전송하며, NOTI/BULK 파일을 주기 송부한다. DB를 직접 쓰지 않는 파일·연동 중심 서비스다.

- 주요 입력: FE의 NBI 서버 설정 요청(`nbi-server`), HiEMS(vRAN)로부터의 report-nb* 데이터 push, 스케줄러 fire time, 타 서비스 REST 응답
- 주요 출력: `/data/dataOSS` 하위 CSV 리포트/NOTI/BULK 파일, SFTP 전송 결과, `envInfo.xml` 설정 파일

## 시작점과 실행 설정
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Report_mgmt/src/main/java/hfr/plugin/my5g/service/Report_mgmt/ServiceStart_Report_mgmt.java`

- `tServiceName = "report_mgmt"`, `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- 별도 ServerConfig 클래스 없음. ServiceStart 내부 `IRedirectServiceURLConfig` 익명 구현으로 redirect URL 결정(debug: `127.0.0.1:10410`, prod: `uems-backend.{LOCAL_DOMAIN}:10410`)
- `tserverRunningConfig.addManualLoadWorker(new manualWorkerLoader_Report_mgmt())` (ServiceStart 52행)

## 아키타입
**D(manualWorkerLoader) + E(외부연동 SFTP/NBI)** — `../feature-extension.md` 참조.

- ② Worker 등록: `common/WorkerLoader/manualWorkerLoader_Report_mgmt.processManualWorkerLoad()`에서 `aWorkflowMap.setWorker(...)`. 현재 등록: `Worker4ExecuteRestAPI`, `Worker4OperationLog`, `WorkerLockManager`, `Worker4ProcessFunction` (공통 인프라 Worker 4종). `Worker_Report_mgmt` 자체는 `getDefine_class()` 패턴으로 WorkflowMap 자동 싱글톤 관리 — manualWorkerLoader에 별도 등록 없음.
- ③ Scheduler 등록: `ScheduleAdapterRat` 상속 클래스가 WorkflowMap을 통해 자동 등록. `Worker_Report_mgmt.initialize()`에서 `m_WorkflowMap.getScheduleJob(...)`으로 참조 취득.
- E 확장(SFTP): 프레임워크 `UtilSFTP`(hfr.framework.server.app.sftp)를 Worker 메서드에서 직접 사용. FunctionTypeBook 어댑터 등록 없음.

## REST contract
근거: `function/RestInvokerFunction_Report_mgmt.java` (getContentList 92~106행), 상수는 `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java` (138~143행)

| request type | 상수 | 방향/용도 |
| --- | --- | --- |
| `nbi-server` | `RequestTypeStringBook.NbiServer` | FE ↔ Report_mgmt. OSS 서버 접속 설정 Get/Set |
| `report-nbbs` | `reportMgmtNbBs` | HiEMS(vRAN) → Report_mgmt. NB BS 데이터 |
| `report-nbbs-cell` | `reportMgmtNbBsCell` | HiEMS(vRAN) → Report_mgmt. NB BS Cell 데이터 |
| `report-nbru` | `reportMgmtNbRu` | HiEMS(vRAN) → Report_mgmt. NB RU 데이터 |
| `report-nbru-cell` | `reportMgmtNbRuCell` | HiEMS(vRAN) → Report_mgmt. NB RU Cell 데이터 |

`processMicroServiceRestRequest()`는 `CommunicatorCD.WorkTypeBook.Get`/`Set` 두 분기만 처리한다(57~74행). `Report_mgmtCD.WorkTypeBook`은 비어 있음(서비스 고유 WorkType 미정의) — 신규 WorkType 추가 시 이 두 파일의 관계를 먼저 확인할 것.

## 주요 Worker/Function 흐름
근거: `worker/Worker_Report_mgmt.java` (약 3,226줄 단일 파일)

핵심 public 메서드(행번호는 스팟체크 시점 기준):
- `queryInfoServer()` — `/data/dataOSS/envInfo.xml`(`ENV_FILE`, 140행)에서 `Model4InfoServer` 로드
- `queryDataCM()` (1379행) / `queryFMOssData()` (2904행, Fault 서비스에 파일 생성 요청) / `queryDataPM(context, sDateTime)` (2084행) — 타 서비스 REST 호출로 데이터 수집
- `makeReportFileCM()` (1720행), `getReportFileName()`, `genFile()` — CSV 리포트 생성
- `sendSFTP()` (952행, `UtilSFTP` 직접 생성) / `sendNoti()` (1268행) / `sendBulk()` (1183행) / `sendStartFile()` — SFTP 전송
- `deleteFile()` (604행), `deleteDataCM()` — 보관기간 경과 파일 정리
- `getConfigData()` (2950행) / `setConfigData()` (2979행) — `nbi-server` Get/Set 처리, `KhnpConfigRequestData` ↔ `Model4InfoServer` 변환

데이터 획득은 전부 `m_WorkerRestContentManager.executeRestAPI(...)` 경유 — 대상 URL은 `common/autogenerate/AutoRestContentBook_ReportMgmt.getRestContentHashMap()`에 하드코딩(basedata/performance/ne-configuration), 포트는 `getHttpsPortFromConfigFile()`에 하드코딩(10443/10445/10447).

## DB/Mongo/파일 흐름
- DB 직접 접근 없음. `CommonMybatisDBConfiguration`은 런타임 의존성으로만 포함되고 Worker는 쿼리하지 않는다. Mongo 미사용.
- 설정 저장소는 파일: `/data/dataOSS/envInfo.xml` — `readXmlFile()`/`makeXmlDocumentServer()`로 읽고 쓴다. 항목: host/id/pass/port/dir/period/retryPeriod/retryNum. 패스워드는 AES/ECB 암호화 저장.
- 리포트 산출물도 파일 시스템(`/data/dataOSS` 하위) 기반 — 컨테이너 볼륨 마운트가 전제.

## scheduler 흐름
근거: `schedule/ScheduleReport4CM.java`, `ScheduleReport4FM.java`, `ScheduleReport4PM.java` (모두 `ScheduleAdapterRat` 상속, `getInterval()=60000`(1분 tick), `isAutoRun()=true`)

- **ScheduleReport4CM**: 매 tick `queryInfoServer()` → 자정에 NOTI(type 3) 생성·`sendNoti()` → `deleteFile()`/`deleteDataCM()` 정리 → `isRetryCM()` 판단 후 `queryDataCM()` 재수집
- **ScheduleReport4FM**: `queryFMOssData()`로 Fault 서비스에 파일 생성 위임 → `sendStartFile(11)`/`sendBulk(14)` → 일 단위 NOTI(type 13) 생성·전송
- **ScheduleReport4PM**: 1분 tick이지만 분(minute)이 5/20/35/50일 때만 `queryDataPM()` 실행(15분 주기 수집) → `sendBulk(24)` → 자정 NOTI(type 23)
- 리포트 type 코드 체계: CM=1/2/3, FM=11/12/13/14, PM=21/22/23/24 (getReportFileName/sendNoti/sendBulk의 type 인자)

## 기능 확장점 (feature seams)

| seam | 추가 방법 | 등록 위치 / 근거 파일 |
| --- | --- | --- |
| ① REST request type | (1) `MicroServiceCommonCD.RequestTypeStringBook`에 상수 추가 (2) `RestInvokerFunction_Report_mgmt.getContentList()`에 `RestContent` 등록 (3) `processMicroServiceRestRequest()` if-else 분기 추가 (4) 필요 시 Worker 메서드 구현 | `function/RestInvokerFunction_Report_mgmt.java` |
| ② Worker/Function | `Worker_Report_mgmt`에 public 메서드 추가(스케줄러/RestInvoker에서 `mWorker.신메서드()` 호출). 신규 Worker 클래스면 `manualWorkerLoader_Report_mgmt.processManualWorkerLoad()`에 `setWorker` 한 줄 추가 | `worker/Worker_Report_mgmt.java`, `common/WorkerLoader/manualWorkerLoader_Report_mgmt.java` |
| ③ Scheduler job | `ScheduleAdapterRat` 상속 클래스(예: `ScheduleReport4XM`)를 `schedule/`에 작성, `getInterval()/execute()/isAutoRun()` 구현. `Worker_Report_mgmt.initialize()`에서 `getScheduleJob(...)`으로 참조. WorkflowMap 자동 등록 — 추가 설정 불필요 | `schedule/ScheduleReport4CM.java` (패턴 원본) |
| E 외부 어댑터 | SFTP 대상/프로토콜 변경은 `Worker_Report_mgmt.sendSFTP()` 수정(현재 `UtilSFTP` 직접 사용). 접속 정보 스키마 변경은 `Model4InfoServer` + `readXmlFile()`/`makeXmlDocumentServer()` 수정 | `worker/Worker_Report_mgmt.java`, `common/data/Model4InfoServer.java` |
| 데이터 모델 | OSS 설정 항목 추가는 `KhnpConfigRequestData`(FE DTO)와 `Model4InfoServer`(내부 모델)를 함께 수정 + `readXmlFile()/makeXmlDocumentServer()/getConfigData()/setConfigData()`에 필드 처리 추가 | `common/data/KhnpConfigRequestData.java` |
| 타 서비스 호출 URL | 신규 서비스 엔드포인트는 `AutoRestContentBook_ReportMgmt.getRestContentHashMap()`에 항목 추가 + `getHttpsPortFromConfigFile()`에 포트 매핑 등록 | `common/autogenerate/AutoRestContentBook_ReportMgmt.java` |

## product 확장점
없음. `Worker_Report_mgmt`는 `abstractWorker` 직접 상속 단일 concrete 클래스, `RestInvokerFunction_Report_mgmt`도 단일 구현체. product 하위 폴더(my5G/openhaul/sixgen-ems) override 없음. product별 분기가 필요해지면 아키타입 D 표준인 `manualWorkerLoaderEx_*` 서브클래스 패턴을 신설한다.

## mock/fake 지점
- `Worker_Report_mgmt.sendSFTP()` — `UtilSFTP` mock으로 전송 없이 파일 생성 경로 테스트
- `m_WorkerRestContentManager.executeRestAPI()` — basedata/performance/ne-configuration 응답 시뮬레이션
- `queryInfoServer()` — `envInfo.xml` 없이 `Model4InfoServer` 직접 주입
- `ScheduleReport4CM/FM/PM.execute()` — `TaskScheduleRat_Data.getFireTime()` stub으로 자정/특정 분(5/20/35/50) 시나리오 테스트

## 변경 위험
- `Worker_Report_mgmt`가 3,226줄 단일 파일 — CM/FM/PM 로직 혼재, 한 도메인 수정이 타 도메인에 영향 가능(분리 리팩터링 후보)
- `processMicroServiceRestRequest()`가 WorkType Get/Set만 분기 — 신규 WorkType 누락 시 무반응으로 실패(에러 없음)
- SFTP 전송·재전송·BULK 모두 로컬 파일 시스템 기반 — `/data/dataOSS` 볼륨 마운트 누락 시 데이터 유실
- `envInfo.xml` 패스워드 AES/ECB 저장 — 암호화 방식 변경 시 기존 데이터 마이그레이션 필요
- `AutoRestContentBook_ReportMgmt`에 타 서비스 포트 하드코딩(basedata:10443, performance:10445, ne-configuration:10447)
- `ScheduleReport4PM`의 5/20/35/50분 시점 하드코딩 — PM 수집 주기 변경은 코드 수정 필요

## 근거 파일
경로 prefix: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Report_mgmt/src/main/java/hfr/plugin/my5g/service/Report_mgmt/`

- `ServiceStart_Report_mgmt.java`
- `function/RestInvokerFunction_Report_mgmt.java`
- `worker/Worker_Report_mgmt.java`
- `common/WorkerLoader/manualWorkerLoader_Report_mgmt.java`
- `schedule/ScheduleReport4CM.java` / `ScheduleReport4FM.java` / `ScheduleReport4PM.java`
- `common/autogenerate/AutoRestContentBook_ReportMgmt.java`
- `common/data/KhnpConfigRequestData.java` / `Model4InfoServer.java` / `Report_mgmtCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java`
