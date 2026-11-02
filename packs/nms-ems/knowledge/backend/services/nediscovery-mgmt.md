# NeDiscovery_mgmt Service

## 서비스 책임
NE(네트워크 장비) 자동 발견(Discovery) 요청을 수신하여 단일 IP / IP 대역 / 등록 추가 / Refresh 4가지 방식으로 하위 EMS·product 서비스에 비동기 위임하고, 성공한 NE 정보를 BaseData 서비스(NMS DB)에 REST로 반영하는 **코디네이터** 서비스다. 자체 DB 테이블·Mongo collection·scheduler는 없다.

## 주요 입력
- NOC/UI에서 오는 Discovery REST 요청 4종(`nediscovery-*`)
- product별 Discovery/PathGuide function의 응답 (EMS·장비 조회 결과)
- BaseData 서비스의 NE 조회 응답

## 주요 출력
- product EMS 서비스로의 Discovery 위임 REST 호출
- BaseData 서비스로의 NE 등록/수정(`Ne.requestType` + `WorkTypeBook.Modify`)
- NOC 포맷으로 변환된 Discovery 결과 응답

## 시작점과 실행 설정
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeDiscovery_mgmt/src/main/java/hfr/plugin/my5g/service/NeDiscovery_mgmt/ServiceStart_NeDiscovery_mgmt.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeDiscovery_mgmt/home/config/nediscovery_mgmt.xml`

확인 기준:
- `tServiceName = "nediscovery_mgmt"`
- `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- HTTP 포트 `10461` (`nediscovery_mgmt.xml`의 `<port>`)
- 서버 실행 모드 `productCD.ServerRunModeBook.MicroService` 고정

## 아키타입
**B(코디네이터/디스패처) + E(외부연동)** — `../feature-extension.md` 참조.
- ② Worker 등록: `Worker_NeDiscovery_mgmt` 단일 Worker, `WorkflowMap.getWorker()`로 주입. unify/inf/manualWorkerLoader 없음.
- ③ Scheduler: 없음.
- 위임 대상 결정: product function(plugin.inf, `E_FunctionTypeCode.nediscovery_ip` / `command_PathGuide`) 동적 조회 → 없으면 `productCD.EmsServiceCode.vRAN` 폴백.

## REST contract
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeDiscovery_mgmt/src/main/java/hfr/plugin/my5g/service/NeDiscovery_mgmt/function/RestInvokerFunction_NeDiscovery_mgmt.java` (분기 line 79~178, 등록 getContentList() line 243~246)

request type 4종 (상수는 `MicroServiceCommonCD.RequestTypeStringBook`, line 274~277):
| request type | 처리 메서드 | 용도 |
| --- | --- | --- |
| `nediscovery-ip` | `executeNeDiscoveryIp` / `executeEmbeddedEMSNeDiscoveryIp` | 단일 IP Discovery |
| `nediscovery-ip-range` | `executeNeDiscoveryIpRange` | IP 대역 스캔 |
| `nediscovery-ip-range-add` | `executeNeDiscoveryIpRangeAdd` | 발견된 NE 일괄 등록 |
| `nediscovery-refresh` | `executeNeDiscoveryRefresh` | 기존 NE 재조회 |

모두 `HttpMethodBook.POST`, version `v1`.

## 주요 Worker/Function 흐름
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeDiscovery_mgmt/src/main/java/hfr/plugin/my5g/service/NeDiscovery_mgmt/worker/Worker_NeDiscovery_mgmt.java`

두 갈래 처리 경로를 구분한다:
1. **일반 NMS 경로** (`executeNeDiscoveryIp`, line 112): `Worker4ProcessFunction.getFunctionList(E_FunctionTypeCode.command_PathGuide)` 로 product function 조회 → `AbsCommandPathGuideFunction.executeRestAPI()` 호출. 처리 가능한 function이 없으면 `EmsServiceCode.vRAN` 서비스로 폴백 위임.
2. **EmbeddedEMS 경로** (`executeEmbeddedEMSNeDiscoveryIp`, line 313): `getFunctionList(E_FunctionTypeCode.nediscovery_ip)` 로 `AbsFunctionDiscovery` 구현체 조회 → `processNeDiscovery()` 호출.

비동기 패턴: `CompletableFuture` promise로 여러 product function에 병렬 위임하고 **최초 성공 응답**을 채택한다. 전부 실패하면 fallback null 처리 — 이 경합/폴백 로직이 이 Worker의 핵심 복잡도다.

NOC↔NMS 데이터 변환: request type별 `*DataMapperFunction`(AbsFunctionNocNmsDataMapper 상속)이 `WorkerNocNmsDataMapper`에 `getDataType()` 키로 자동 인덱싱된다(`<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/worker/WorkerNocNmsDataMapper.java` line 36~41).

## DB/Mongo/파일 흐름
이 서비스 안에 Mapper XML·독립 테이블·Mongo collection이 **없다**. NE 조회/갱신은 전부 `WorkerLocalBaseDataCommon`(`m_WorkerLocalBaseData`)을 경유해 BaseData 마이크로서비스에 REST 위임한다:
- 조회: `requestNeByNameKey()`, `requestNeInfo()`, `getChildNeInfoList()`
- 갱신: `processBaseDataRequest()` — `updateNeGroup()`에서 `Ne.requestType` + `CommunicatorCD.WorkTypeBook.Modify` 전송 (Worker line 249, 422, 988~1009)

## scheduler 흐름
없음.

## 기능 확장점 (feature seams)
| seam 유형 | 추가 방법 | 등록 위치 / 근거 |
| --- | --- | --- |
| ① REST request type | `RestInvokerFunction_NeDiscovery_mgmt.processMicroServiceRestRequest()`에 else-if 분기 추가 + `getContentList()`에 `new RestContent(this, RequestTypeStringBook.XXX, HttpMethodBook.POST, getServiceName(), "v1")` 추가. 상수는 `MicroServiceCommonCD.RequestTypeStringBook`에 선언 | `RestInvokerFunction_NeDiscovery_mgmt.java` line 79~178, 243~246 / `MicroServiceCommonCD.java` line 274~277 |
| ② Worker function | `Worker_NeDiscovery_mgmt`에 `execute*` public 메서드 추가. 별도 Worker 필요 시 `abstractWorker` 상속 후 `WorkflowMap.getWorker()` 주입 | `Worker_NeDiscovery_mgmt.java` initialize() line 79~89 |
| ⑤ product Discovery (EmbeddedEMS 경로) | `AbsFunctionDiscovery` 상속한 `FunctionDiscoveryIP_XXX`를 product EMS 서비스에 구현, plugin.inf에 `E_FunctionTypeCode.nediscovery_ip` 타입으로 등록 | vran 패턴: `<BACKEND_ROOT>/product/my5G/my5g.Service.vran_ems/src/main/java/hfr/plugin/my5g/service/vran_ems/hiems/function/FunctionDiscoveryIP_vran.java` |
| ⑤ product PathGuide (NMS 경로) | `AbsCommandPathGuideFunction` 상속한 `FunctionCommandPathGuide_XXX`를 product 서비스에 구현, plugin.inf에 `E_FunctionTypeCode.command_PathGuide` 타입으로 등록 | openHaul 패턴: `<BACKEND_ROOT>/product/openhaul/my5g.Service.openHaul/src/main/java/hfr/plugin/my5g/service/openHaul_ems/hiems/function/FunctionCommandPathGuide_openHaul.java` |
| data mapper (NOC↔NMS) | 기존 `NeDiscoveryIP[Range[Add]]DataMapperFunction`의 `convertToNmsFormat()/convertToNocFormat()` 수정, 또는 `AbsFunctionNocNmsDataMapper` 상속 신규 작성 후 `E_FunctionTypeCode.NocNmsDataMapperFunction`으로 plugin.inf 등록 → `getDataType()` 키로 자동 인덱싱 | `function/mapper/NeDiscoveryIP*DataMapperFunction.java` / `WorkerNocNmsDataMapper.java` line 36~41 |
| 프로토콜 세션 타입 | `common/data/neprotocol/` 아래에 `Ne_Session_XXX_Discovery` 추가 + `@ObjectClassProvider(keyFields={"protocol"}, keyValues={"N"})` 선언 → JSON 역직렬화 시 자동 선택 | `common/data/neprotocol/Ne_Session_netconf_Discovery.java` line 8 |
| DB 조작 추가 | `WorkerLocalBaseDataCommon`에 메서드 추가하거나 BaseData 서비스의 requestType을 직접 호출 (이 서비스에 mapper 만들지 말 것) | `Worker_NeDiscovery_mgmt.java` line 249, 422, 988~1009 |

## product 확장점
이 서비스 자체에는 product별 추상 클래스/override 구조가 없다. `Worker4ProcessFunction`이 plugin.inf 등록 기반으로 product function을 런타임 동적 로딩한다. 확장 지점 2종:
1. EmbeddedEMS 경로 → `AbsFunctionDiscovery` 구현체(`FunctionDiscoveryIP_vran`, `FunctionDiscoveryIP_openHaul`, `FunctionDiscoveryIP_six_gen` 등)를 `E_FunctionTypeCode.nediscovery_ip`로 등록
2. NMS 경로 → `AbsCommandPathGuideFunction` 구현체(`FunctionCommandPathGuide_openHaul` 등)를 `E_FunctionTypeCode.command_PathGuide`로 등록

폴백은 `productCD.EmsServiceCode.vRAN` 고정 — 새 product가 폴백에 걸리면 vRAN EMS로 잘못 위임된다.

## mock/fake 지점
1. `Worker4ProcessFunction.getFunctionList(command_PathGuide / nediscovery_ip)` → Mock `AbsCommandPathGuideFunction` / `AbsFunctionDiscovery`로 대체하면 외부 EMS 없이 테스트 가능
2. `WorkerLocalBaseDataCommon`(`m_WorkerLocalBaseData`) → Mock 주입으로 BaseData 서비스 없이 NE 조회/갱신 테스트
3. `WorkerRestContentManager`(`m_WorkerRestContentManager`) → Mock으로 vRAN 폴백 REST 호출 격리

## 변경 위험
- `executeNeDiscoveryIp()`의 CompletableFuture promise는 **최초 성공 응답**을 채택한다 — product function 추가 시 비동기 경쟁 조건과 fallback null 처리를 반드시 확인
- `executeNeDiscoveryIpRangeAdd()`에 `Thread.sleep(5000)` 하드코딩(Worker line 919) — 대량 NE 등록 시 지연 누적
- `RestInvokerFunction`의 `parseJSONData()`(line 267~)가 문자열 split 수동 파싱 — NE 필드명/값에 콤마·콜론이 포함되면 파싱 오류
- `executeNeDiscoveryIpRangeAdd()`는 `productCD.ProductTypeCode.valueOf(productcode)`로 위임 서비스명을 결정 — 새 product code 지원 시 `productCD` enum 수정 필요
- vRAN 폴백 고정(`EmsServiceCode.vRAN`, line 666/791/1117) — product function 미등록 시 조용히 vRAN으로 흘러간다

## 근거 파일
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeDiscovery_mgmt/src/main/java/hfr/plugin/my5g/service/NeDiscovery_mgmt/ServiceStart_NeDiscovery_mgmt.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeDiscovery_mgmt/src/main/java/hfr/plugin/my5g/service/NeDiscovery_mgmt/function/RestInvokerFunction_NeDiscovery_mgmt.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeDiscovery_mgmt/src/main/java/hfr/plugin/my5g/service/NeDiscovery_mgmt/worker/Worker_NeDiscovery_mgmt.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeDiscovery_mgmt/src/main/java/hfr/plugin/my5g/service/NeDiscovery_mgmt/function/mapper/NeDiscoveryIPDataMapperFunction.java` (동일 디렉터리 `NeDiscoveryIPRangeDataMapperFunction.java`, `NeDiscoveryIPRangeAddDataMapperFunction.java`)
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeDiscovery_mgmt/src/main/java/hfr/plugin/my5g/service/NeDiscovery_mgmt/common/data/neprotocol/Ne_Session_netconf_Discovery.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.NeDiscovery_mgmt/home/config/nediscovery_mgmt.xml`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/function/AbsFunctionNocNmsDataMapper.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/worker/WorkerNocNmsDataMapper.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java`
- `<BACKEND_ROOT>/product/my5G/my5g.Service.vran_ems/src/main/java/hfr/plugin/my5g/service/vran_ems/hiems/function/FunctionDiscoveryIP_vran.java`
- `<BACKEND_ROOT>/product/openhaul/my5g.Service.openHaul/src/main/java/hfr/plugin/my5g/service/openHaul_ems/hiems/function/FunctionCommandPathGuide_openHaul.java`
