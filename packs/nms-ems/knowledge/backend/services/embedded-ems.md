# embeddedEMS Service

## 서비스 책임
다수의 embedded EMS 제품(my5GBox 등)의 REST 요청을 단일 진입점에서 수신하고, URL 첫 번째 세그먼트(서비스명)를 키로 WorkflowMap에 등록된 `AbsEmbeddedEMSRestInvokerFunction` 구현체에 동적으로 라우팅하는 dispatcher 마이크로서비스. 서비스 자체에는 비즈니스 로직·request type이 없다 — 전부 product function이 정의한다.

## 주요 입력
- embedded EMS 제품 대상 REST 요청 (URL 첫 세그먼트 = 대상 embedded EMS 서비스명)
- `SystemConfig_Sync` notification

## 주요 출력
- 라우팅된 product function의 `processRestRequest()` 결과(`httpJsonResponseData`)
- 미등록 서비스명 요청 시 `ErrorCodeBook.NOTEXISTS` 예외

## 시작점과 실행 설정
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.embeddedEMS/src/main/java/hfr/plugin/my5g/service/embeddedEMS/ServiceStart_embeddedEMS.java`, `home/config/embeddedems.xml`
- `tServiceName = "embeddedems"`, `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- HTTP port = **10500** (`embeddedems.xml`의 `<port>`)
- 서비스 자체 ServerConfig 클래스 없음(framework `DefaultSecurityHttps` 사용). product별 서버 설정은 `AbsServerConfigEmbeddedEMS` 상속으로 해결(아래 product 확장점).

## 아키타입
**B(코디네이터/디스패처)** — `../feature-extension.md` 참조. 단, 다른 B형처럼 `executeRestAPI()` 위임이 아니라 **URL 세그먼트 기반 function 라우팅** 변형이다.
- ② Worker 등록: 별도 등록 절차 없음. `Worker_embeddedEMS`가 최초 요청 시 `WorkflowMap.getFunctionList()`를 스캔해 `platformTigerCD.FunctionTypeBook.embedded_EMS_RestInvoker_function` 타입 코드 버킷의 function들을 lazy-init(ReentrantLock)으로 서비스명 맵에 수집한다.
- ③ Scheduler: 없음.

## REST contract
근거: `function/RestInvokerFunction_embeddedEMS.java`
- 서비스 고유 request type **없음**. `Worker_embeddedEMS.getContentList()`가 등록된 각 `AbsEmbeddedEMSRestInvokerFunction.getContentHashMap()`을 집계해 전체 content 목록을 구성한다.
- `processMicroServiceRestRequest()` / `processMicroServiceRestRequestUsingInputStream()` 모두: URL 첫 세그먼트 → `getEmbeddedEMSRestInvokerFunction(서비스명)` → 해당 function의 `processRestRequest()` 호출. 미발견 시 `NOTEXISTS`.
- `getServiceName()` = `SupportService.embeddedEMS`.

## 주요 Worker/Function 흐름
근거: `worker/Worker_embeddedEMS.java`, `function/RestInvokerFunction_embeddedEMS.java`
- `Worker_embeddedEMS.getEmbeddedEMSRestInvokerFunction(String)` (line 69~): 내부 맵이 null이면 lock 잡고 `WorkflowMap.getFunctionList()` 스캔 → `embedded_EMS_RestInvoker_function` 타입 function들을 `getServiceName()` 키로 맵 구성 → 이후 요청은 맵 조회만.
- notification: `getReceive_NotificationTypes()`가 `SystemConfig_Sync`만 구독(line 139-143), `received_NotificationData()`가 등록된 모든 embedded EMS function에 브로드캐스트(line 146-151). 각 function은 `onNotification(UNotification)` 구현으로 수신.

## DB/Mongo/파일 흐름
- 자체 DB mapper/XML 없음. `CommonMybatisDBConfiguration` 전달로 프레임워크 공통 DB 설정만 사용. Mongo 미사용.
- `common/data/embeddedEMSCD.java`의 `WorkTypeBook`은 주석만 있는 빈 코드북.

## scheduler 흐름
없음.

## 기능 확장점 (feature seams)
| seam 유형 | 추가 방법 | 등록 위치/근거 파일 |
| --- | --- | --- |
| external-adapter (새 embedded EMS 제품) | `AbsEmbeddedEMSRestInvokerFunction`(EMSPlatform.Tiger) 상속 구체 클래스를 product 모듈에 작성. `getServiceName()`이 URL 첫 세그먼트와 일치하는 문자열 반환. `getTypeCode()`는 abstract 클래스가 이미 `embedded_EMS_RestInvoker_function`을 반환하므로 별도 등록 불필요 — WorkflowMap에 로드되면 `Worker_embeddedEMS`가 자동 발견 | `worker/Worker_embeddedEMS.java` (getEmbeddedEMSRestInvokerFunction), `<BACKEND_ROOT>/Platform_framework/EMSPlatform.Tiger/src/hfr/platform/Tiger/product/common/function/AbsEmbeddedEMSRestInvokerFunction.java` |
| ① rest-request-type (product function 단위) | 새 request type은 해당 product function의 `getContentHashMap()` override에 `RestContent` 추가 → `Worker_embeddedEMS.getContentList()`(line 129~)가 자동 집계. **이 서비스의 `RestInvokerFunction_embeddedEMS`는 수정하지 않는다** | `worker/Worker_embeddedEMS.java` (getContentList), `AbsEmbeddedEMSRestInvokerFunction.java` (getContentHashMap) |
| ⑤ product-override (요청 처리) | `processRestRequest()`를 product 모듈에서 구현(실제 NE 통신). product별 서버 설정 필요 시 `AbsServerConfigEmbeddedEMS` 상속 ServerConfig 작성 후 WorkflowMap 등록. 예: `ServerConfig4my5gBox` | `<BACKEND_ROOT>/Platform_framework/EMSPlatform.Tiger/src/hfr/platform/Tiger/product/server/config/AbsServerConfigEmbeddedEMS.java`, `<BACKEND_ROOT>/product/my5G/my5GBox/src/main/java/hfr/ems/embedded/product/my5GBox/ServerConfig4my5gBox.java` |
| notification 확장 | 각 function이 `onNotification(UNotification)` abstract 구현으로 `SystemConfig_Sync` 수신. 추가 notification type 구독은 `RestInvokerFunction_embeddedEMS.getReceive_NotificationTypes()` 수정(공통영역 변경 — 신중히) | `function/RestInvokerFunction_embeddedEMS.java` (line 139-151) |
| data-model | 서비스 공유 work type 상수는 `embeddedEMSCD.WorkTypeBook`에 추가(현재 완전히 비어 있음) | `common/data/embeddedEMSCD.java` |

## product 확장점
새 product 추가의 표준 패턴 (my5GBox 예):
- `<BACKEND_ROOT>/product/my5G/my5GBox/.../function/common/AbsRestInvokerFunction_my5GBox.java` — `AbsEmbeddedEMSRestInvokerFunction` 래퍼를 product 공통 기반으로 두고, 그 아래 서비스별 구현체(NeConfiguration/Performance/System 등 RestInvokerFunction_my5GBox 계열)를 배치.
- ServerConfig: `AbsServerConfigEmbeddedEMS` → `ServerConfig4my5gBox`.

## 죽은/미구현 코드
- `isVaildSession()` (line 156-158) — 항상 `Success` 반환, **세션 검증 미구현**. 코드 주석에 "차후에 정의 필요" 잔존.
- `embeddedEMSCD.WorkTypeBook` — 주석만 있고 실제 정의 없음.

## mock/fake 지점
1. `Worker_embeddedEMS.getEmbeddedEMSRestInvokerFunction(String)` — 특정 function 반환 제어
2. `AbsEmbeddedEMSRestInvokerFunction.processRestRequest()` — product function을 stub으로 교체해 라우팅 로직만 단위 테스트
3. `WorkflowMap.getFunctionList()` — `embedded_EMS_RestInvoker_function` 버킷에 mock 데이터 주입해 lazy-init 테스트

## fixture 기준
URL 세그먼트(서비스명) / WorkflowMap function 버킷 구성 / 등록·미등록 서비스명 case(NOTEXISTS) / notification 브로드캐스트 대상 목록.

## 변경 위험
- WorkflowMap에 function 미등록 시 모든 embedded EMS 요청이 `NOTEXISTS` 반환 — "추가했는데 안 됨"의 1순위 원인은 product function이 WorkflowMap에 로드되지 않은 것.
- `isVaildSession()`이 무조건 Success — 보안 요구사항 추가 시 이 지점부터 구현 필요.
- `getContentList4EmbeddedEMS()`의 fallback path(m_Worker_embeddedEMS null 시 Worker 임시 생성)에서 m_WorkflowMap이 달라 content 목록 불일치 가능.
- lazy-init 특성상 최초 요청 전 content 목록/라우팅 맵이 비어 있음 — 기동 직후 테스트 시 주의.

## 근거 파일
서비스 루트 = `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.embeddedEMS/`
- `src/main/java/hfr/plugin/my5g/service/embeddedEMS/ServiceStart_embeddedEMS.java`
- `src/main/java/hfr/plugin/my5g/service/embeddedEMS/function/RestInvokerFunction_embeddedEMS.java`
- `src/main/java/hfr/plugin/my5g/service/embeddedEMS/worker/Worker_embeddedEMS.java`
- `src/main/java/hfr/plugin/my5g/service/embeddedEMS/common/data/embeddedEMSCD.java`
- `home/config/embeddedems.xml`
- `<BACKEND_ROOT>/Platform_framework/EMSPlatform.Tiger/src/hfr/platform/Tiger/product/common/function/AbsEmbeddedEMSRestInvokerFunction.java`
- `<BACKEND_ROOT>/Platform_framework/EMSPlatform.Tiger/src/hfr/platform/Tiger/product/server/config/AbsServerConfigEmbeddedEMS.java`
- `<BACKEND_ROOT>/Platform_framework/EMSPlatform.Tiger/src/hfr/platform/Tiger/product/common/data/platformTigerCD.java` (FunctionTypeBook.embedded_EMS_RestInvoker_function)
- `<BACKEND_ROOT>/product/my5G/my5GBox/src/main/java/hfr/ems/embedded/product/my5GBox/function/common/AbsRestInvokerFunction_my5GBox.java`
- `<BACKEND_ROOT>/product/my5G/my5GBox/src/main/java/hfr/ems/embedded/product/my5GBox/ServerConfig4my5gBox.java`
