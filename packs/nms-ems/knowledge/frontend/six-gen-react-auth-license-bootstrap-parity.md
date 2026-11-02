# Six-Gen React License Bootstrap Parity

## 범위

- Angular 기준: `C:/khd/6gworkspace/UnifyEMS/uems_fe`
- React 대상: `D:/sixgen-react`
- 목적: 로그인 응답 수신과 통신 세션 활성화를 분리하고 Angular startup 순서를 그대로 유지한다.

## Angular 기준 순서

활성 six-gen 로그인 순서는 다음과 같다.

`Login -> license expiration Get -> WebSocket(session_id) -> first keepalive Response -> system-env -> current user -> role -> BaseDataLoadProcess`

- `Login` 응답의 `token`과 `session_id`는 다른 값이며 용도도 다르다.
- 보호 REST 요청의 `access_token`과 body `client_session_id`는 모두 `session_id`를 사용한다.
- WebSocket도 `session_id`로 연결한다.
- 첫 keepalive Response를 받기 전에는 environment/user/role을 조회하지 않는다.
- license 경고가 표시되면 사용자가 확인할 때까지 WebSocket bootstrap을 진행하지 않는다.
- license 조회 실패는 Angular와 동일하게 로그인 실패로 전파하지 않는다.

## License 요청 계약

- URL: `system/v1/get-license-expire-time`
- Angular 객체: `new UResponse('get-license-expire-time', 'Get')`
- `access_token`: 로그인 `session_id`
- body `client_session_id`: 같은 `session_id`

```json
{
  "message_type": "Response",
  "version": "v1",
  "client_session_id": "<session>",
  "request_id": 0,
  "request_type": "get-license-expire-time",
  "work_type": "Get",
  "status": 0
}
```

일반 `Request` envelope로 다시 만들면 안 된다. React 공통 serializer는 `UMessageBase` 계열의 `message_type`을 보존하면서 활성 `session_id`만 주입해야 한다.

## React 구현 위치

- 응답/요청 분리: `src/runtime/authApi.ts`
- 순서 제어: `src/runtime/sessionBootstrap.ts`
- 세션 상태와 경고 확인: `src/runtime/authContext.tsx`
- 공통 session 주입: `src/runtime/requestSerialization.ts`
- WebSocket/keepalive: `src/runtime/webSocketClient.ts`

## 검증

- `npm run test:request-contract`
- `SIXGEN_REACT_URL=http://127.0.0.1:<port>/unms npm run qa:six-gen-auth`
- `npm run qa:six-gen-auth-live`
- `SIXGEN_TEST_USERNAME=<user> SIXGEN_TEST_PASSWORD=<password> npm run qa:six-gen-auth-startup-live`
- startup 동등성 게이트는 React 세션을 열기 전에 Angular에서 로그아웃하고,
  첫 keepalive와 모든 보호 startup 요청 본문을 비교하며
  `.codex/visual-qa/auth-startup-live/auth-startup-parity-result.json`에
  비식별 boolean·상태 값만 기록한다.
- fixture gate는 `dDay=31` 응답으로 30일 경고 MessageBox를 실제 렌더링하고 OK 전 event가 Login/License 두 건뿐인지 확인한다.
- live gate는 license 요청을 필수 보호 요청으로 포함하고 정확한 기본 필드, `access_token`, `client_session_id`, WebSocket 이전 순서를 검증한다.
- fixture `seedBrowserSession`도 license 응답을 제공해 테스트가 실 backend로 새지 않게 한다.
- 산출물에는 password, token, 실제 session id를 저장하지 않는다.

## 완료 판단

실제 인증 성공과 startup gate가 통과하기 전에는 전체 React 이식을 완료 처리하지 않는다. 이 계약은 로그인 화면 한 건이 아니라 모든 보호 REST/WebSocket 호출의 선행 조건이다.
