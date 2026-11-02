# WebCli Service

## 서비스 책임
브라우저 WebSocket 연결로 NE(네트워크 장비)에 대한 Web CLI(Virtual Terminal) 명령을 중계하고, 장비 CLI 실행 결과를 해당 클라이언트 WebSocket 세션으로 되돌려주는 WebCLI 게이트웨이 마이크로서비스. CLI 세션 실행 자체는 플랫폼 레이어(`Worker4VirtualTerminal`)에 위임한다.

## 주요 입력
- 브라우저 WebSocket 연결/명령 텍스트 (`/websocket-cli/{clientSessionID}`)
- REST `ems-cli-result` (POST) — 장비 측에서 CLI 결과를 push할 때
- NE 정보 조회 결과 (`WorkerLocalBaseDataCommon.requestNeInfo()`)

## 주요 출력
- WebSocket 클라이언트로의 CLI 결과 문자열 전송 (`Session.sendStringByFuture()`)
- Virtual Terminal 요청 (`Worker4VirtualTerminal.processRequest()`)

## 시작점과 실행 설정
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/unms.Service.WebCli/src/main/java/hfr/plugin/unms/service/WebCli/ServiceStart_WebCli.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/unms.Service.WebCli/home/config/webcli.xml`

확인 기준:
- `tServiceName = "webcli"`, `runMicroService.run("webcli", new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- HTTP 포트: `10462` (`webcli.xml` `<http><port>`)
- WebSocket: `webcli.xml` `<jettywebsocket>` — `<path>/websocket-cli</path>`, `<listener>hfr.plugin.unms.service.WebCli.function.WebCliWebSocketAdapter</listener>`
- `CommonMybatisDBConfiguration`을 넘기지만 서비스 자체 mapper/CD/MyBatis XML은 없음(공통 설정 로드만)

## 아키타입
**E(외부연동 — WebSocket adapter)** — `../feature-extension.md` 참조.
- ② Worker 등록: 프레임워크 `runMicroService.run()`이 WorkflowMap을 구성하고, `Worker_WebCli.initialize()`/`RestInvokerFunction_WebCli.initialize()`가 `m_WorkflowMap.getWorker()`로 의존 worker를 조회한다. 서비스 전용 `.inf`/manualWorkerLoader 없음.
- ③ Scheduler 없음.
- E형 지정 지점: WebSocket 어댑터는 `webcli.xml`의 `<listener>`/`<path>`로 지정한다.

## WorkflowMap 의존 worker
- `Worker4VirtualTerminal` (플랫폼, `AbsWorker4VirtualTerminal` 구현) — CLI 세션 실행
- `WorkerLocalBaseDataCommon` — NE 정보 조회
- `Worker_WebCli` — 본 서비스 핵심 worker (없으면 `EmpException(ERROR_NOTEXISTS)`로 기동 불가)

## REST contract
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/unms.Service.WebCli/src/main/java/hfr/plugin/unms/service/WebCli/function/RestInvokerFunction_WebCli.java` (getContentList L74~, 분기 L110)

request type:
- `ems-cli-result` (POST, v1) — 상수 `MicroServiceCommonCD.RequestTypeStringBook.Ems_Web_Cli_Result` (`<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java` L270). 장비 측 CLI 결과 push 수신용.

## 주요 Worker/Function 흐름
근거:
- `function/WebCliWebSocketAdapter.java`, `function/RestInvokerFunction_WebCli.java`, `worker/Worker_WebCli.java` (서비스 루트 하위)

WebSocket 명령 처리:
1. `WebCliWebSocketAdapter`가 연결/데이터수신/해제 이벤트를 `RestInvokerFunction_WebCli`(onSocketConnected / onSocketDataReceived / onSocketDisconnected)로 전달
2. 연결 URL에서 `m_webSocketCheckUrl`(기본 `"/websocket-cli/"`) 뒤 문자열을 `clientSessionID`로 파싱 — 불일치 시 파싱 실패로 즉시 close
3. 수신 명령 → `Worker_WebCli.processWebCliCommand(VirtualTerminalWorkRequest)` → `workerLocalBaseData.requestNeInfo(neID)`로 NE 조회 → `Worker4VirtualTerminal.processRequest()`
4. 비동기 결과: `workerVirtualTerminal.addEventVirtualTerminalListener` 콜백 → `RestInvokerFunction_WebCli.onVirtualTerminalResponse()` → `sendResult()` → WebSocket 클라이언트

세션 관리는 `ReentrantLock` + `Hashtable` 인메모리 방식이다.

## DB/Mongo/파일 흐름
직접 DB 접근 없음. 서비스 내 mapper/`*CD` 데이터 접근/MyBatis XML 없음. NE 정보는 공통 레이어(`WorkerLocalBaseDataCommon.requestNeInfo()`)에 위임.

## scheduler/queue/background 흐름
scheduler 없음.

## 기능 확장점 (feature seams)
| seam 유형 | 추가 방법 | 등록 위치/근거 파일 |
| --- | --- | --- |
| ① rest-request-type | `getContentList()`에 `new RestContent(this, RequestTypeStringBook.XXX, HttpMethodBook.POST/GET, getServiceName(), "v1")` 추가 + `processMicroServiceRestRequest()` if-else 분기 추가. type 문자열은 `MicroServiceCommonCD.RequestTypeStringBook`(또는 `WebCliCD`)에 상수 선언 먼저 | `function/RestInvokerFunction_WebCli.java` (L74~L117) |
| external-adapter (WebSocket, 아키타입 E) | 이벤트 처리 변경 시 `WebCliWebSocketAdapter` 수정 또는 서브클래스 작성 후 `webcli.xml` `<listener>`를 새 FQCN으로 교체. URL 변경 시 `<path>`와 `RestInvokerFunction_WebCli.m_webSocketCheckUrl`(setwebSocketCheckUrl)을 **함께** 수정 | `home/config/webcli.xml` (L11-13), `function/WebCliWebSocketAdapter.java` |
| ② worker-function | CLI 처리 로직 변경은 `Worker_WebCli.processWebCliCommand()` 수정. 새 하부 Worker는 `abstractWorker` 상속 작성 + `Worker_WebCli.initialize()`에서 `m_WorkflowMap.getWorker()`로 주입 — WorkflowMap 등록은 프레임워크/플러그인 설정 담당 | `worker/Worker_WebCli.java` (L42~L96) |
| data-model | 요청/응답 확장은 `WebCliRequest`(URequest 상속)/`WebCliResponse`(UResponse 상속) 수정, CLI 인증 확장은 `IModelNeSessionCLI` 구현 클래스 추가. work type 분기 확장은 `WebCliCD.WorkTypeBook`에 상수 등록 후 사용 | `common/data/WebCliRequest.java`, `WebCliResponse.java`, `IModelNeSessionCLI.java`, `WebCliCD.java` |

## product 확장점
product별 override 없음(`product/` 하위 my5G·openhaul·sixgen-ems에 WebCli 전용 클래스 없음). 가상 터미널 하부 구현은 플랫폼 레이어 `AbsWorker4VirtualTerminal`(EMSPlatform) → `Worker4VirtualTerminal`(EMSPlatform.Ox)이 담당하며 WebCli는 그대로 사용. `IModelNeSessionCLI`가 CLI 인증 모델 확장 인터페이스.

## 죽은/미구현 코드
- `Worker_WebCli.sendRequestToDevice()` — southbound REST 직접 전송 코드가 남아 있으나 호출부(L81)는 주석 처리 상태(이전 아키텍처 잔재). 기능 추가 대상으로 삼지 말 것.
- `WebCliCD.WorkTypeBook` — 주석만 있는 빈 클래스(확장 준비 상태).

## mock/fake 지점
real 장비 연결 없이 테스트할 때:
1. `Worker4VirtualTerminal.processRequest()` (AbsWorker4VirtualTerminal 구현체) 모킹 — CLI 처리 검증
2. `WorkerLocalBaseDataCommon.requestNeInfo()` 모킹 — NeID 기반 분기 검증
3. `org.eclipse.jetty.websocket.api.Session.sendStringByFuture()` 모킹 — 결과 전송 검증

## 변경 위험
- 플랫폼 `Worker4VirtualTerminal`이 WorkflowMap에 없으면 `Worker_WebCli.initialize()` 실패
- `Worker_WebCli`이 WorkflowMap에 없으면 `RestInvokerFunction_WebCli.initialize()`에서 `EmpException(ErrorCodeBook.ERROR_NOTEXISTS)` → 서비스 기동 불가
- WebSocket 세션이 인메모리(ReentrantLock + Hashtable) 관리 → 서비스 재시작 시 모든 연결 끊김
- `webcli.xml` `<path>`와 `m_webSocketCheckUrl` 불일치 → clientSessionID 파싱 실패 → 연결 즉시 close

## 근거 파일
서비스 루트 = `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/unms.Service.WebCli/`
- `src/main/java/hfr/plugin/unms/service/WebCli/ServiceStart_WebCli.java`
- `src/main/java/hfr/plugin/unms/service/WebCli/function/RestInvokerFunction_WebCli.java`
- `src/main/java/hfr/plugin/unms/service/WebCli/function/WebCliWebSocketAdapter.java`
- `src/main/java/hfr/plugin/unms/service/WebCli/worker/Worker_WebCli.java`
- `src/main/java/hfr/plugin/unms/service/WebCli/common/data/WebCliRequest.java`, `WebCliResponse.java`, `IModelNeSessionCLI.java`, `WebCliCD.java`
- `home/config/webcli.xml`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java` (RequestTypeStringBook.Ems_Web_Cli_Result)
- `<BACKEND_ROOT>/Platform_framework/EMSPlatform/src/hfr/platform/Rat/product/server/app/cliterminal/process/AbsWorker4VirtualTerminal.java`
- `<BACKEND_ROOT>/Platform_framework/EMSPlatform.Ox/src/hfr/platform/Ox/product/server/app/cliterminal/process/Worker4VirtualTerminal.java`
