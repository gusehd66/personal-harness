# FE ↔ BE 연결 흐름

## 사용 시점
FE service가 backend에 붙는 경로(REST proxy, session, WebSocket)를 분석하거나 바꿀 때 이 문서를 사용한다.
API contract 자체는 `api/conventions.md`, backend 등록 흐름은 `backend/rest-content-flow.md`를 함께 본다.

## 경로 표기
- `<FRONTEND_ROOT>`: Angular frontend root.
- `<BACKEND_ROOT>`: Java backend root.
아래 파일 경로는 각 root 기준 상대경로이며, 개인 PC 절대경로는 쓰지 않는다.

## 1. REST 연결

### FE 측
- REST client는 `framework_modules/library/network/restapi.ts`의 `RestAPIClient`다.
- 요청 URL은 `apiUri`(기본 `rest-api`) prefix + `service/version/requestType` 형태로 조립된다(`makeURL`).
- 세션은 `app.http-interceptor.ts`에서 `access_token` 헤더(`s_Header_RestSessionID`)로 실린다.
- Keycloak 등 auth ignore 대상 URL은 interceptor가 헤더를 건드리지 않는다.

### dev proxy
- `proxy.config.json`이 FE 경로를 backend로 프록시한다.
- app shell별 context(`/unms`, `/systemmgmt`, `/usermgmt`, `/customermgmt`)는 모두 `pathRewrite`로 `/rest-api`에 합류한다.
- `/hiems/rest-api`는 별도 EMS 포트로 분기한다.
- target 포트와 secure flag는 환경마다 다르므로 값 자체를 하드코딩으로 단정하지 않는다.

### BE 측
- endpoint는 `RestContentBook_*`(생성물은 `AutoRestContentBook`)에 `new RestContent(...)`로 등록된다.
- 등록 키는 `service name + version + requestType + scope + auth flag`이며, 이것이 FE 호출과 매칭되는 contract다.
- requestType별 분기는 각 서비스의 `RestInvokerFunction_<Service>`, 실제 처리는 `Worker_<Service>`다.
- 상세는 `backend/rest-content-flow.md`.

## 2. WebSocket 연결 (login 시)

### 연결 순서
1. REST login(`session-mgmt`)이 먼저 성공해 `session_id`를 발급받는다.
2. login 성공 직후 login start sequence가 `connectWebSocket(session_id)`를 호출한다.
3. FE는 `wss://<host>/[productUrl/]websocket/<session_id>`로 접속한다. REST는 헤더로 세션을 싣지만 WebSocket은 헤더를 못 쓰므로 **session_id를 URL path에 넣는다**.
4. FE는 첫 keepalive 응답을 받아야 연결을 확정한다(`waitFirstKeepalive`).

### FE 측 파일
- login → websocket 트리거: `src/app/common/login/user-login-process.ts`의 login start sequence.
- 연결 오케스트레이션: `framework_modules/framework/communication/communicationprocess.ts`의 `connectWebSocket`.
- 소켓 URL 조립: `framework_modules/library/network/websocket.ts`의 `makeConnectURL`.
- `ProductInformation.config.connectBackEnd`와 `ProductInformation.isUseWebSocket`이 모두 켜져야 동작한다. 꺼지면 websocket 없이 진행한다.

### BE 측 파일
- path → adapter 매핑: `WebSocketConfig`. `/websocket`은 jetty config(`jettywebsocket`)로 등록되어 `JettyStart2.start(...)`가 wire하고, `/nocwebsocket`은 `ServerStart_*_NOC`에서 코드로 등록한다. 둘 다 `CommonWebSocketAdapter`를 쓴다.
- 공통 동작: `WebSocketAdapterEx`. `getSessionKey()`가 URL path에서 `/websocket/` 뒤를 잘라 session_id를 추출한다.
- 세션 검증: `CommonWebSocketAdapter.isValidWebSession`이 `getJettyUserSession(session_id)`로 REST login이 만든 session 존재를 확인한다. 없으면 401.
- keepalive: `WebSocketAdapterEx.onWebSocketText` → `handleInboundRequest`에서 `requestType == Keep_alive`면 workerPool을 거치지 않고 `data="success"`를 즉시 응답한다(FE 연결 확정 트리거).
- 세션/푸시 허브: `AbsWorkerWebSocketSessionManager`(예: `WorkerRemoteSessionManager`, Session_mgmt 서비스)가 session_id별 소켓을 보관하고 alarm/event를 `sendString(...)`으로 push한다.

## 분석 순서
1. FE에서 대상 화면 component → 주입된 service → 호출 URL의 `service/version/requestType`를 추출한다.
2. `proxy.config.json`에서 해당 context가 어느 backend로 가는지 확인한다.
3. BE `RestContentBook_*`에서 같은 service/requestType의 `new RestContent(...)`를 찾는다.
4. WebSocket이 필요한 기능이면 login start sequence → `connectWebSocket` → `makeConnectURL`까지 FE path를 확인한다.
5. BE에서 `WebSocketConfig` 등록 path와 `CommonWebSocketAdapter` 세션 검증, push 경로(session manager)를 확인한다.

## 하네스 기준
연결 fixture는 아래를 포함한다.
- REST: service name, version, requestType, HTTP method, scope, auth 필요 여부, proxy context, target 분기.
- WebSocket: 연결 path(`/websocket` vs `/nocwebsocket`), session_id 전달 방식(URL path), keepalive request/response, 세션 검증 주체, push 주체(session manager).

## 주의점
- WebSocket은 REST login으로 만든 session_id에 의존한다. login 없이 websocket 단독 연결은 401로 막힌다.
- session_id는 REST에서는 `access_token` 헤더, WebSocket에서는 URL path로 전달된다. 전달 위치를 혼동하지 않는다.
- 같은 requestType이라도 product profile/service name에 따라 처리 worker가 달라질 수 있다.
- `/websocket`(일반 FE)과 `/nocwebsocket`(NOC) 분리, NOC에서 BE가 client로 접속하는 경로(`WorkerWebSocketClient4NOC`)를 일반 FE 연결과 섞지 않는다.
- proxy target/포트/secure 값은 환경 설정이므로 기능 작업과 함께 임의로 바꾸지 않는다.
