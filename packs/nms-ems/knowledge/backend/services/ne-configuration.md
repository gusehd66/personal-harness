# NE Configuration Service

## 서비스 책임
장비(NE) 조회/설정 command를 product-agnostic하게 수신하여, 요청 JSON의 `product_type`/`consumer_type`에 따라 product별 `AbsMy5GNMSInvokerAdapter`(NMS 경로) 또는 `AbsEmbeddedEMSInvokerAdapter`(embeddedEMS 경로)로 라우팅한다. 부가로 Express Setting 파일 읽기, ShelfView 상태 조회, NeConfig 파일 업로드를 처리한다. read/write 모두 장비에 직접 영향을 줄 수 있는 real-write 위험 서비스다.

## 시작점과 실행 설정
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Ne_Configuration/src/main/java/hfr/plugin/my5g/service/Ne_Configuration/ServiceStart_Ne_Configuration.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Ne_Configuration/home/config/ne_configuration.xml` (`<port>10447</port>`)

확인 기준:
- `tServiceName = "ne_configuration"`
- `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`

## 아키타입
**E(외부연동) + B(코디네이터)**. 자체 비즈니스 로직은 얇고, 요청 대부분을 product adapter로 위임한다.
- ② Worker: `RestInvokerFunction_Ne_Configuration.initialize()`에서 `m_WorkflowMap.getWorker(...)`로 참조 (`Worker_Ne_Configuration`, `WorkerNeRequest4EmbeddedEMS`).
- ③ Scheduler: 없음.
- E-확장 등록: `InvokerAdapterManager`(EmsTypecode 키) + `FunctionTypeBook.shelfViewFunction`(WorkflowMap function).

## REST contract
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Ne_Configuration/src/main/java/hfr/plugin/my5g/service/Ne_Configuration/function/RestInvokerFunction_Ne_Configuration.java`

`getContentList()` 등록 타입 (모두 POST, v1):
- `NeRequest`, `Ne_restart`, `UE_Trace_Received`
- `Express_Main`, `Express_File`, `Express_Cell`, `Express_Gnb`, `Express_Routing`, `Express_TA`, `Express_UPF`
- `Network_Slice`, `Network_Slice_Cell`
- `reportMgmtNbBs`, `reportMgmtNbBsCell`, `reportMgmtNbRu`, `reportMgmtNbRuCell`

분기 코드는 있으나 로컬 `getContentList()`에는 미등록인 타입 (라우팅 확인 필요):
- `ShelfViewStatus`("shelf-view-status") → `ShelfViewProcess.processShelfViewStatus()`
- `SetCuduPrivateMac` → `setCuduMacAddress()`(B-스타일: `m_WorkerRestContentManager.executeRestAPI()`로 NE의 product 서비스에 위임)
- 하드코딩 문자열: `openHaul-portInfo`(빈 분기), `request-otdr`, `request-otdr-file`(주석), `request-otdr-download`(빈 분기) — `product_type == "openHaul-ems"`이면 나머지는 openHaul adapter로 위임
- `local_NeConfig_upload` → InputStream 파일 업로드, `/data/backup_restore/ef/` 저장 (`processMicroServiceRestRequestUsingInputStream` override)

## 주요 Worker/Function 흐름
`processMicroServiceRestRequest()` 라우팅 순서:
1. `UE_Trace_Received` → vRAN adapter 고정 조회 후 `processRequest()`.
2. `SetCuduPrivateMac` → `setCuduMacAddress()`: NE 조회 후 product 서비스명으로 `executeRestAPI` 위임 (NE type이 EMS이면 서비스명 = NE name).
3. `Express_File` → `Worker_Ne_Configuration.readExpressInstallerFile()` (`/data/nms_express/express.json` 읽기) 응답.
4. `shelf-view-status` → `ShelfViewProcess.processShelfViewStatus()`.
5. `product_type == "openHaul-ems"` → openHaul adapter로 위임 (OTDR 계열 포함).
6. 그 외(default) → `ProductTypeCode.valueOfServiceName(product_type)`으로 EmsTypecode 결정:
   - `consumer_type == embeddedEMS` → `WorkerNeRequest4EmbeddedEMS.processNeRequest()` → `AbsEmbeddedEMSInvokerAdapter.processNeRequest/processNeListRequest`
   - 그 외 → `AbsMy5GNMSInvokerAdapter.processRequest()` (⚠ wifi는 vRAN으로 강제 변환, `report-nbbs`/`report-nbru` 포함 타입도 vRAN 강제)
7. 어느 분기도 응답을 만들지 못하면 `throw new Exception("NeRequest null")`.

`processBypassRequestHandler` override: 요청 JSON `data.ne_id`를 NE의 `site_ne_id`로 재매핑하여 `BypassRequestData(site_id, json)` 생성 — site 게이트웨이 bypass 시 NE ID 변환 지점.

## DB/Mongo/파일 흐름
- 로컬 mapper/CD 없음. `Ne_ConfigurationCD`는 `WorkTypeBook.Search` 상수만 정의. 실제 DB 접근은 위임받은 product adapter 쪽에서 수행.
- 파일: 읽기 `/data/nms_express/express.json` (Express_File), 쓰기 `/data/backup_restore/ef/` (local_NeConfig_upload, 디렉터리 자동 생성 + chmod 777).

## 기능 확장점 (feature seams)
| seam 유형 | 추가 방법 | 등록 위치/근거 |
| --- | --- | --- |
| ① rest-request-type | `getContentList()`에 `new RestContent(this, <타입>, HttpMethodBook.POST, getServiceName(), "v1")` 추가 + `processMicroServiceRestRequest()` if-else 분기 추가. 상수는 `my5gData.RequestTypeStringBook` 또는 `MicroServiceCommonCD.RequestTypeStringBook`에 선언 | `RestInvokerFunction_Ne_Configuration.java` (getContentList line 337-368, 분기 line 186-288) |
| ⑤ product adapter (NMS) | 새 product의 `AbsMy5GNMSInvokerAdapter` 구현체를 product 서비스에서 작성하고 `InvokerAdapterManager`에 해당 `EmsTypecode`로 등록. 요청 `product_type` 문자열이 `ProductTypeCode.valueOfServiceName()`으로 매핑되므로 `ProductTypeCode`에도 추가 필요 | `RestInvokerFunction_Ne_Configuration.java` (line 261, 275) / `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/invoker/AbsMy5GNMSInvokerAdapter.java` |
| ⑤ product adapter (embeddedEMS) | `AbsEmbeddedEMSInvokerAdapter` 구현체를 product 서비스에서 작성, `InvokerAdapterManager`에 EmsTypecode로 등록. `consumer_type=embeddedEMS` 요청 시 `WorkerNeRequest4EmbeddedEMS`가 조회·호출 | `WorkerNeRequest4EmbeddedEMS.java` (line 119-131, 186-192) / `<BACKEND_ROOT>/uNMS_Common/uEMS.Common/src/main/java/hfr/common/adpater/AbsEmbeddedEMSInvokerAdapter.java` |
| ⑤ product override (ShelfView) | `AbsFunctionShelfView` 상속: `getSupportModels()`에 NE_CODE 목록 반환 + `createNeProcess()` 구현. `FunctionTypeBook.shelfViewFunction` 코드로 WorkflowMap 등록하면 `ShelfViewProcess.initialize()`가 NE model별 HashMap 매핑을 자동 구성 | `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Ne_Configuration/src/main/java/hfr/plugin/my5g/service/Ne_Configuration/shelfview/ShelfViewProcess.java` (line 58-72, 105-110) |
| ② worker-function | `Worker_Ne_Configuration`에 public 메서드 추가 후 해당 request type 분기에서 호출 (`initialize()`에서 이미 `getWorker(Worker_Ne_Configuration.class)` 참조) | `Worker_Ne_Configuration.java` / `RestInvokerFunction_Ne_Configuration.java` (line 135, 212) |
| data-model | 새 WorkType 상수는 `Ne_ConfigurationCD.WorkTypeBook`, request type 문자열 상수는 `my5gData.RequestTypeStringBook` 또는 `MicroServiceCommonCD.RequestTypeStringBook` | `Ne_ConfigurationCD.java` / `<BACKEND_ROOT>/Platform_framework/my5gCommonData/src/main/java/hfr/ems/my5g/common/data/my5gData.java` |

## product 확장점
- `AbsMy5GNMSInvokerAdapter`: product별 NMS invoker. vRAN/openHaul/wifi/sixgen 각 product 서비스가 자신의 EmsTypecode로 등록. 이 서비스는 `getAdapter(EmsTypecode)`로만 조회 — product별 코드를 이 서비스에 넣지 않는 것이 설계 의도(line 259-260 주석).
- `AbsEmbeddedEMSInvokerAdapter`: embeddedEMS consumer 경로 전용.
- `AbsFunctionShelfView`: product별 구현(예: ShelfViewNeProcessOpenHaul, ShelfViewNeProcessSixGen)이 NE model code 기준으로 매핑됨.

## 죽은/미구현 코드
- `ExpressSettingProcess` — 처리 메서드 전부 주석, 생성자만 남은 사실상 빈 클래스. Express 관련 수정 요청 시 실제 동작 경로는 `Worker_Ne_Configuration` 쪽임.
- `openHaul-portInfo` 분기(line 228-229) — 빈 블록. 매칭되면 아무 처리 없이 마지막 `"NeRequest null"` 예외로 떨어진다.
- `request-otdr-download` 분기(line 240) — 빈 블록(동일 증상). `request-otdr-file` 분기는 주석 처리.
- `processRequestAPController`, `processEmsRequest`(주석) — 테스트/미사용 코드.
- `SetCuduPrivateMac` — 처리 코드는 live이나 로컬 `getContentList()`에 RestContent 미등록. 구 vRAN adapter 라우팅 코드는 주석(line 202-207).

## mock/fake 지점
- `NeConfigurationRequest` JSON payload (`product_type`, `consumer_type`, `request_type` 포함)
- `AbsMy5GNMSInvokerAdapter.processRequest()` (product별)
- `AbsEmbeddedEMSInvokerAdapter.processNeRequest/processNeListRequest()`
- `Worker_Ne_Configuration.readExpressInstallerFile()` — `/data/nms_express/express.json` 파일 mock
- `AbsFunctionShelfView.createNeProcess()` (NE model별)
- `m_WorkerLocalBaseData.requestNeInfo(neID)`

## 변경 위험
- 장비 설정 오동작(write command 잘못 라우팅) — write는 mock-write fixture 기본.
- 신규 request type을 if-else 체인에 추가 누락 시 `"NeRequest null"` 예외로 요청 전체 실패.
- `product_type` 문자열 ↔ `EmsTypecode` 매핑 불일치 시 adapter 미탐(blackBox.error 후 예외).
- wifi → vRAN 강제 변환(line 271-272), `report-nbbs`/`report-nbru` → vRAN 강제(line 262-263) 하드코딩 — wifi/report 계열 신규 기능 시 의도 확인 필수.
- `ShelfViewProcess`는 NE model 기준 HashMap 캐시 — model 코드/지원 목록 변경 시 서비스 재기동 필요.
- `openHaul-portInfo`·`request-otdr-download` 빈 분기 fall-through로 예외 발생 가능.

## 근거 파일
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Ne_Configuration/src/main/java/hfr/plugin/my5g/service/Ne_Configuration/ServiceStart_Ne_Configuration.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Ne_Configuration/src/main/java/hfr/plugin/my5g/service/Ne_Configuration/function/RestInvokerFunction_Ne_Configuration.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Ne_Configuration/src/main/java/hfr/plugin/my5g/service/Ne_Configuration/worker/Worker_Ne_Configuration.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Ne_Configuration/src/main/java/hfr/plugin/my5g/service/Ne_Configuration/worker/WorkerNeRequest4EmbeddedEMS.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Ne_Configuration/src/main/java/hfr/plugin/my5g/service/Ne_Configuration/shelfview/ShelfViewProcess.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Ne_Configuration/src/main/java/hfr/plugin/my5g/service/Ne_Configuration/function/ExpressSettingProcess.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Ne_Configuration/src/main/java/hfr/plugin/my5g/service/Ne_Configuration/common/data/Ne_ConfigurationCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Ne_Configuration/home/config/ne_configuration.xml`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/invoker/AbsMy5GNMSInvokerAdapter.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.Common/src/main/java/hfr/common/adpater/AbsEmbeddedEMSInvokerAdapter.java`
- `<BACKEND_ROOT>/Platform_framework/EMSPlatform/src/hfr/ems/product/common/function/AbsFunctionShelfView.java`
- `<BACKEND_ROOT>/Platform_framework/my5gCommonData/src/main/java/hfr/ems/my5g/common/data/my5gData.java`
