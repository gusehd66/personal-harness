# API 규칙 노트

## 사용 시점
FE service call, request/response model, error handling, pagination, backend contract 변경 시 이 문서를 사용한다.

## 요청/응답
- backend contract가 바뀌지 않는 한 기존 field name과 payload shape를 유지한다.
- API-facing model을 변경할 때 request와 response 영향을 보고한다.
- null, empty, timeout, retry case를 명시적으로 처리한다.

## FE framework 전환 직렬화
- Angular에서 React로 전환할 때 TypeScript model 선언만으로 wire shape를 추정하지 않는다. 실제 `model -> request -> CommunicationProcess` 경로와 브라우저 payload를 함께 기준으로 삼는다.
- `undefined`로 누락된 property, 명시적인 `null`, 빈 문자열, 빈 object/array는 서로 다른 계약으로 취급하고 기존 요청에 나타난 상태를 그대로 유지한다.
- response에서 받은 미표시 metadata는 Update/Delete 전에 제거하거나 기본값으로 정규화하지 않는다. Angular의 snapshot, `old_data`, enum/date, nested command-field 직렬화를 그대로 보존한다.
- 기존 Angular 코드의 잘못된 property 접근 때문에 field는 존재하지만 `value`가 누락되는 경우도 framework-only 전환에서는 현재 wire 계약이다. backend 변경과 함께 교정하기 전에는 React가 값을 추론해 보완하지 않는다.
- protected REST/WebSocket 요청은 로그인 token과 통신 `session_id`를 구분한다. six-gen 통신 세션이 활성화된 뒤에는 header `access_token`과 body `client_session_id`가 모두 활성 `session_id`인지 contract test로 확인한다.

## 오류 처리
- 기존 service와 component의 error handling pattern을 따른다.
- operator 또는 developer가 조치해야 하는 backend error를 숨기지 않는다.
- log와 report에서 secret, token, personal data를 마스킹한다.

## 페이지네이션
- 기존 page index, page size, total count, sorting, filtering convention을 유지한다.
- 변경 전에 pagination이 client-side인지 server-side인지 확인한다.
