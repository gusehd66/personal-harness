# REST Content 흐름

## 기본 구조
이 backend의 REST 구조는 annotation scan보다 자체 `RestContent` 등록 흐름이 중요하다.

핵심 요소:
- `RestContentBook_*`
- `RestContentManagementFunction_*`
- `AutoRestContentBook`
- `AbsRestContentManagementFunction`
- `AbsRestRequestInvokerFunction4Process`
- `RequestTypeStringBook`
- `RestServiceNameBook`
- `new RestContent(...)`
- `WorkerRestContentManager`
- `MicroServiceRestInvoker`

## 분석 순서
1. manual worker loader가 어떤 `RestContentManagementFunction_*`을 `WorkflowMap.addFunction(...)`으로 등록하는지 확인한다.
2. `RestContentBook_*`가 `AutoRestContentBook`을 override하는지 확인한다.
3. `new RestContent(...)` 호출에서 request type, HTTP method, service name, version, scope, auth flag를 추출한다.
4. `WorkerRestContentManager.getURL(...)`, `getLocalRestHttpBaseDomain(...)` override 여부를 확인한다.
5. service name이 debug/NOC/uEMS/microservice mode에 따라 어떤 URL로 변환되는지 추적한다.

## 하네스 기준
REST contract fixture는 아래를 포함한다.

- service name
- request type
- version
- HTTP method
- scope
- auth 필요 여부
- path segment
- local/microservice URL 분기
- 처리 invoker/function/worker

## 주의점
- 같은 request type이라도 product profile과 service name에 따라 처리 worker가 달라질 수 있다.
- `AutoRestContentBook`은 생성 코드일 수 있으므로 원본 generator와 생성 결과 중 실제 runtime에 쓰이는 파일을 구분한다.
