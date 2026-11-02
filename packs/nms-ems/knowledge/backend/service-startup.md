# Service 시작 흐름

## 확인 순서
1. `ServerStart_*.java`를 찾는다.
2. service name을 확인한다.
3. `serverRunningConfig` 설정을 확인한다.
4. `hframeworkCD.frameworkGlobal` 또는 product별 global property 설정을 확인한다.
5. `addManualLoadWorker(...)`로 등록되는 loader를 찾는다.
6. `setMicroServiceRestInvoker(...)`가 동기/비동기 invoker 중 무엇을 쓰는지 확인한다.
7. `runMicroService.run(...)` 또는 `runMicroService3.run(...)` 인자를 확인한다.

## 주요 설정
- DB 사용 여부: `setUseDataBase4MicroService(...)`
- MongoDB 사용 여부: `setUseMongoDB4MicroService(...)`
- session manager 사용 여부: `setUseSessionManager(...)`
- session manager class: `setSessionManagerClass(...)`
- license check: `setUseLicenceCheck(...)`
- system type/customer id: `Server_RunningConfigPropertiesBook`
- run mode: `ServerRunModeBook`, `Server_run_mode`
- debug mode: `frameworkGlobal.isDebugMode`

## 하네스 기준
- startup fixture는 service name, run mode, debug mode, DB/Mongo/session 사용 여부, manual worker loader 목록을 포함한다.
- debug mode와 microservice mode는 URL, DB host, service discovery 방식을 바꿀 수 있으므로 반드시 분리한다.
- 실제 service를 띄우기 전에 classpath와 config path가 어떻게 구성되는지 확인한다.
