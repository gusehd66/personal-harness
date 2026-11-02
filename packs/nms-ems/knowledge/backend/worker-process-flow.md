# Worker/Process 흐름

## 핵심 registry
`WorkflowMap`은 backend runtime의 중심 registry이다.

확인할 항목:
- `setWorker(...)`
- `isExistWorker(...)`
- `getWorker(...)`
- `addFunction(...)`
- `getFunctionList(...)`
- `getRestRequestInvokerFunctionMap(...)`
- `getMicroServiceRestInvokerFunction(...)`
- scheduler map
- DB driver 접근

## 분석 순서
1. manual worker loader에서 등록되는 worker를 모두 열거한다.
2. worker가 상속하는 base class와 `getDefine_class()`를 확인한다.
3. worker가 사용하는 다른 worker를 `WorkflowMap.getWorker(...)`, `isExistWorker(...)` 기준으로 추적한다.
4. request type에서 시작해 invoker function, worker, process method까지 call-chain을 작성한다.
5. DB, MongoDB, 외부 adapter, scheduler, WebSocket 호출 지점을 표시한다.

## 하네스 기준
worker/process fixture는 아래를 포함한다.

- 등록된 worker class
- 등록된 function class
- request type
- 입력 model/payload
- 호출되는 process method
- 필요 dependency worker
- 예상 response model
- 오류/timeout/retry/permission case

## 단독 실행이 어려운 이유
- `WorkflowMap` singleton 또는 global registry 의존
- `EmpContext`와 transaction 의존
- `serverRunningConfig`와 global property 의존
- DB/Mongo driver 의존
- 외부 adapter와 runtime endpoint 의존
- scheduler 또는 background worker 상태 의존

하네스는 이 의존성을 fixture와 mock/fake로 분리해야 한다.
