# Gateway_Uems Service

## 서비스 책임
uNMS/EMS 게이트웨이 서비스. NMS↔EMS 세션 인증·콘텐츠 교환, NOC WebSocket 연결 관리(위탁 consign), Site 데이터 동기화(주기·수동), 내부 서비스 간 요청 포워딩, NOC Notification 전달을 담당한다.

## 주요 입력
- EMS/NMS 세션 인증·콘텐츠 교환 REST 요청 (`Ems_Authentication`, `Ems_session`, `Ems_contents`, `Ems_exchange_contents`)
- 내부 서비스가 포워딩을 위탁하는 요청 (`Forwareded_Request`, `Get_target_info`)
- NOC 위탁 시작/알림 (`consignStart`, `Notification_To_NOC`, `Received_Notification`)
- Site 동기화 트리거 (`Manual_SiteData_Sync`, ScheduleSiteSyncManager 주기 실행)

## 주요 출력
- 내부 서비스(basedata/fault/system/performance 등)로의 `executeRestAPI` 위임 결과
- NOC WebSocket 연결·heartbeat·notification 송신
- Site별 NE/ALARM/EVENT/Pm 동기화 요청

## 시작점과 실행 설정
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/src/main/java/hfr/plugin/my5g/service/Gateway_Uems/ServiceStart_Gateway_Uems.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/home/config/gateway_uems.xml`

확인 기준:
- `tServiceName = "gateway_uems"`
- `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())` — HTTPS + MyBatis
- ServerConfig: `home/config/gateway_uems.xml`, `<port>10456</port>`
- debugmode=on 시 Source_Path/PlugIn_Path에 `uEMS.Common`, `uEMS.plugin.my5g.Common` 포함

## 아키타입과 등록 위치
`../feature-extension.md` 기준 **B(코디네이터/포워딩) + C(.inf 로더) + E(외부연동 WebSocket)**.

- ②③(Worker/Schedule) 등록 = `home/jar/gateway_uems.inf`에 **FQCN 한 줄** 추가(아키타입 C). 현재 .inf에 `RestInvokerFunction_Gateway_Uems`, `RestInvokerFunction_to_ems`, `WorkerCommonNotifier`, `WorkerRestContentManager` 등이 등록되어 있다. **클래스를 만들어도 .inf에 안 넣으면 로딩 안 됨.**
- B: 데이터 처리 대부분을 `m_WorkerRestContentManager.executeRestAPI(getClass(), SupportService.<svc>, ...)`로 내부 서비스에 위임. 예: `Get_target_info` → `SupportService.fault`.
- E: NOC 연결은 `WorkerWebSocketClient4NOC`(JettyWebSocketClient)가 담당.
- product 폴더 분기 없음 — `SupportService.gateway_uems` / `SupportService.to_ems` 단일 구현(§product 확장점).

## REST contract
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/src/main/java/hfr/plugin/my5g/service/Gateway_Uems/function/RestInvokerFunction_Gateway_Uems.java` (service=`gateway_uems`)
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/src/main/java/hfr/plugin/my5g/service/Gateway_Uems/function/RestInvokerFunction_to_ems.java` (service=`to_ems`)

`gateway_uems` request type (getContentList 등록 순):
- `Ems_Authentication` (POST) / `Ems_exchange_contents` (POST) / `Ems_session` (GET) / `Ems_contents` (GET) — NMS↔EMS 세션·콘텐츠
- `Keep_alive` (GET) — Subscriber 헤더 기반 세션 유지
- `Forwareded_Request` (POST) — 내부 서비스 포워딩
- `Get_target_info` (GET) — `SupportService.fault`로 위임
- `consignStart` (POST) / `Notification_To_NOC` (POST) / `Received_Notification` — NOC 위탁·알림 (Received_Notification의 UserManagement_Sync는 `WorkerCommonNotifier.addNotifierBaseData_Sync` 호출)
- `Manual_SiteData_Sync` (POST, `ScopeTypeBook.Inner` — 내부 전용)

`to_ems` request type:
- `Log_level` (POST|GET) — EMS 로그레벨 조회/설정

## 주요 Worker/Function 흐름
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/src/main/java/hfr/plugin/my5g/service/Gateway_Uems/worker/ProcessMainWorker_Gateway_Uems.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/src/main/java/hfr/plugin/my5g/service/Gateway_Uems/worker/WorkerWebSocketClient4NOC.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/worker/Worker_Gateway_Uems.java`

- `ProcessMainWorker_Gateway_Uems.main()`: NMS 모드(`!isNOC()`)면 `NMS_CustomerID` 기준 Site 자동 생성(`processCreateSite`) → `NMS_SiteID` property 고정 → `WorkerWebSocketClient4NOC.processConnectNOC()`로 NOC 접속.
- `WorkerWebSocketClient4NOC`: NOC WebSocket 연결·재연결·heartbeat 응답(`processNocRevokedHeartbeatResponse`)·URL 결정(`getNOCWebSocketURL`)이 이 클래스에 집중.
- `Worker_Gateway_Uems`(uEMS.plugin.my5g.Common): EMS 세션·콘텐츠·로그레벨 처리 공통 로직. 두 RestInvokerFunction이 공유한다.
- `RestInvokerFunction_Gateway_Uems.processMicroServiceRestRequest()`: 단일 대형 if-else 체인으로 request type 분기.

## DB/Mongo/파일 흐름
직접 Mapper/CD 없음. `CommonMybatisDBConfiguration` 등록만 하고, 데이터 접근은 전부 `Worker_Gateway_Uems`·`m_WorkerRestContentManager.executeRestAPI`로 내부 서비스(basedata/fault/system 등)에 위임한다. 로컬 `common/data/Gateway_UemsCD.java`는 WorkType 상수만 정의(데이터 모델 없음).

## scheduler/queue/background 흐름
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/src/main/java/hfr/plugin/my5g/service/Gateway_Uems/schedule/ScheduleSiteSyncManager.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/src/main/java/hfr/plugin/my5g/service/Gateway_Uems/schedule/ScheduleSiteSyncWorker.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java`

- `ScheduleSiteSyncManager`: `isAutoRun()=false`, `initialized()`에서 `manualRunRepeat(60000)`로 60초 주기 자기등록. **NOC 모드에서만** 동작(`isNOC()` 아니면 return). basedata의 Site 목록(`nms_info != null`)과 `m_ActiveSiteSet`을 비교해 `ScheduleSiteSyncWorker.startSiteSyncJob(siteId)` / `stopSiteSyncJob(siteId)`로 Site별 동기화 job을 동적 등록·삭제한다.
- `ScheduleSiteSyncWorker.runSiteDataSyncAll()`: `MicroServiceCommonCD.SyncCategoryBook` 전체(현재 `NE`→basedata, `ALARM`/`EVENT`→fault, `Pm`→performance)를 순회해 카테고리별 동기화 실행. MultiQueue2로 병렬 처리.
- `Manual_SiteData_Sync`: ExecutorService(5 thread) 비동기 실행.
- `ScheduleKeepAlive`: `execute()` 본문 전체 주석 = **no-op** (§죽은 코드).

## 기능 확장점 (feature seams)

| seam 유형 | 추가 방법 | 등록 위치 / 근거 파일 |
| --- | --- | --- |
| ① REST request type (gateway_uems) | `RestInvokerFunction_Gateway_Uems.getContentList()`에 `new RestContent(this, <새type>, HttpMethodBook.*, getServiceName(), "v1", ...)` 한 줄 추가 + `processMicroServiceRestRequest()` if-else 분기 추가 + `MicroServiceCommonCD.RequestTypeStringBook`에 상수. 등록만 하면 엔드포인트 자동 노출 | `.../Gateway_Uems/function/RestInvokerFunction_Gateway_Uems.java` |
| ① REST request type (to_ems) | EMS로 보내는 제어/조회 신규 기능은 `RestInvokerFunction_to_ems`의 `getContentList()` + `processMicroServiceRestRequest()` 분기에 추가 | `.../Gateway_Uems/function/RestInvokerFunction_to_ems.java` |
| ② Worker/Function 공통 로직 | `Worker_Gateway_Uems`(uEMS.plugin.my5g.Common)에 public 메서드 추가 → 두 RestInvokerFunction에서 즉시 호출 가능 | `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/worker/Worker_Gateway_Uems.java` |
| ③ 동기화 카테고리 추가 | `MicroServiceCommonCD.SyncCategoryBook`에 `register(<categoryKey>, SupportService.<대상서비스>)` 한 줄 추가 → `runSiteDataSyncAll()` 루프에 자동 포함 | `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java` |
| ③ Scheduler job 추가 | `ScheduleAdapterRat` 상속 클래스를 `schedule/` 패키지에 작성(`execute()/isAutoRun()/getInterval()` 구현) + `home/jar/gateway_uems.inf`에 FQCN 한 줄 등록(아키타입 C). 동적 주기는 ScheduleSiteSyncManager처럼 `manualRunRepeat()` 사용 | `.../Gateway_Uems/schedule/ScheduleKeepAlive.java`, `home/jar/gateway_uems.inf` |
| ⑤ 외부연동(NOC WebSocket) | 연결·재연결·heartbeat 정책 변경은 `WorkerWebSocketClient4NOC` 직접 수정 또는 subclass — 관련 로직이 이 클래스 한 곳에 집중 | `.../Gateway_Uems/worker/WorkerWebSocketClient4NOC.java` |

## product 확장점
product별 override 구조 없음. `RestInvokerFunction_Gateway_Uems`는 `AbsRestInvokerFunction_Service`, `RestInvokerFunction_to_ems`는 `AbsRestInvokerFunction_my5GService` 상속이며 product(my5G/openhaul/sixgen-ems) 분기 없이 단일 구현. product별 차이가 필요해지면 ⑤ product override 신설을 먼저 검토한다.

## 죽은/미구현 코드
- `ScheduleKeepAlive.execute()` — 본문 전체 주석 = **no-op**. 세션 만료 처리 없음 (`../feature-extension.md` 함정 목록에도 등재).
- `home/jar/gateway_uems.inf`에 선언됐으나 repo에 소스가 없는 FQCN 3건: `...Gateway_Uems.schedule.ScheduleEmsSessionKeepAliveCheck`, `...Gateway_Uems.schedule.ScheduleSubscriberSessionKeepAliveCheck`, `...Gateway_Uems.worker.Worker_Gateway_Uems`(실제 존재하는 것은 `hfr.plugin.my5g.common.worker.Worker_Gateway_Uems`). stale 선언 가능성 — .inf 수정 전 실제 로딩 여부 확인 필요.

## mock/fake 지점
1. `Worker_Gateway_Uems` — EMS 세션·콘텐츠·로그레벨 조회 mock
2. `WorkerWebSocketClient4NOC` — JettyWebSocketClient 연결 mock
3. `WorkerLocalSystemData` — System_config ConsignNOC/isNOC 조회 mock
4. `WorkerSiteDataSync.requestSiteDataSync` — 동기화 로직 단위 테스트용 mock
5. `m_WorkerRestContentManager.executeRestAPI` — 내부 서비스 REST 호출 mock

## 변경 위험
- `WorkerWebSocketClient4NOC`는 연결·재연결·heartbeat·위탁취소가 단일 클래스에 집중 — 수정 시 NOC 접속 단절 위험
- `runSiteDataSyncAll()`은 SyncCategoryBook 전체 순회 — 카테고리 추가 시 동기화 부하 증가
- `processMicroServiceRestRequest()` 단일 대형 if-else 체인 — 신규 type 추가 시 기존 분기와 충돌 주의
- `ScheduleKeepAlive` no-op — 세션 만료 미처리 상태를 전제로 한 코드가 있을 수 있음
- `Manual_SiteData_Sync`는 5-thread ExecutorService 비동기 — 큐 무제한이라 대량 요청 시 OOM 가능성
- 최근 변경 이력이 잦은 서비스(2026-03~04 주석 기준) — 수정 전 현재 소스 재확인

## 근거 파일
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/src/main/java/hfr/plugin/my5g/service/Gateway_Uems/ServiceStart_Gateway_Uems.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/src/main/java/hfr/plugin/my5g/service/Gateway_Uems/function/RestInvokerFunction_Gateway_Uems.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/src/main/java/hfr/plugin/my5g/service/Gateway_Uems/function/RestInvokerFunction_to_ems.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/src/main/java/hfr/plugin/my5g/service/Gateway_Uems/worker/ProcessMainWorker_Gateway_Uems.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/src/main/java/hfr/plugin/my5g/service/Gateway_Uems/worker/WorkerWebSocketClient4NOC.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/src/main/java/hfr/plugin/my5g/service/Gateway_Uems/schedule/ScheduleSiteSyncManager.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/src/main/java/hfr/plugin/my5g/service/Gateway_Uems/schedule/ScheduleSiteSyncWorker.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/src/main/java/hfr/plugin/my5g/service/Gateway_Uems/schedule/ScheduleKeepAlive.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/home/config/gateway_uems.xml`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Gateway_Uems/home/jar/gateway_uems.inf`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/worker/Worker_Gateway_Uems.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java`
