# em_emulator Service (stub)

## 서비스 책임
외부 EMS 장비(pcf, smp, vcore, wifi/APC)를 에뮬레이션해 실제 장비 없이 EMS 연동 REST 인터페이스를 시뮬레이션하는 서비스. **현재 상태 = skeleton/stub**: Function들이 하드코딩 JSON을 반환하고, 인증/초기화 로직은 `if(1==1){return;}`으로 비활성화되어 있다. 실 운영 기능이 아니라 개발/테스트 지원 목적.

## 시작점과 실행 설정
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.em_emulator/src/main/java/hfr/plugin/my5g/service/em_emulator/ServiceStart_em_emulator.java`
- `tServiceName = "em_emulator"`, `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- DB: isDebugMode=true면 127.0.0.1, 운영은 `platformRatCD.SupportService.maria_db` + `LOCAL_DOMAIN`. 자체 mapper/XML 없음 — `CommonMybatisDBConfiguration`은 공통 프레임워크 의존용.
- `serverRunningConfig`에 `manualWorkerLoader4Service` 수동 로더 등록. Mongo/scheduler 없음.

## 아키타입
**D(manualWorkerLoader)** — `../feature-extension.md` 참조. ②③ 등록 위치 = `manual/manualWorkerLoader4Service.processManualWorkerLoad()`. product override(⑤) 구조 없음(모든 Function이 `AbsRestInvokerFunction_Service` 직접 상속, product 분기 없음).

## REST contract
request type: `test-run` (pcf/vcore/wifi의 `processMicroServiceRestRequest()` 분기), `EMS` (WorkerRest4vCore control 경로). 단 **manualWorkerLoader4Service에는 `RestInvokerFunction_smp`만 `addFunction()` 등록**되어 있어 pcf/vcore/wifi endpoint는 실제 구동 시 비활성일 수 있다 — 활성화하려면 로더에 addFunction 추가 필요.

## 주요 Worker/Function 흐름
- Function: `RestInvokerFunction_pcf/smp/vcore/wifi` — 하드코딩 JSON 응답 stub. wifi는 `EmsServiceCode.APC`, smp는 `EmsServiceCode.SM_PLATFORM` 사용.
- Worker: `Worker_em_emulator`, `WorkerRest4vCore`(vcore REST 제어, EMS 분기 본문 비어 있음) + 프레임워크 공통 Worker들(`Worker4ExecuteRestAPI`, `WorkerLockManager`, `Worker4ProcessFunction`, `Worker4NeCommand`, `Worker4VirtualTerminal`, `Worker4PerformanceMonitoring`, `WorkerNePropertyManager`, `Worker4RebootCheckProcess`) — 모두 `isExistWorker()` 체크 후 setWorker 패턴.

## 기능 확장점 (feature seams)
| seam | 추가 방법 | 등록 위치/근거 |
| --- | --- | --- |
| ① 새 에뮬레이션 장비 endpoint | `function/`에 `AbsRestInvokerFunction_Service` 상속 `RestInvokerFunction_{장비명}` 생성 → `processMicroServiceRestRequest()`에 aRequestType 분기 | `manualWorkerLoader4Service.processManualWorkerLoad()`에 `aWorkflowMap.addFunction(new RestInvokerFunction_{장비명}())` 한 줄 추가 |
| ② 새 Worker | `worker/`에 `abstractWorker` 상속 `Worker_{기능}` 생성, `isExistWorker()` 체크 후 등록 패턴 유지 | `manualWorkerLoader4Service`에 `aWorkflowMap.setWorker(new Worker_{기능}(aWorkflowMap))` |
| 외부 adapter (복잡한 장비 제어) | `WorkerRest4vCore` 패턴 참고: `Worker4{장비명}Control` 생성, Function의 `initialize()`에서 `m_WorkflowMap.getWorker(...)`로 주입 | `worker/WorkerRest4vCore.java` + manualWorkerLoader4Service setWorker |
| data-model 상수 | `em_emulatorCD.WorkTypeBook`에 static String 상수 추가(현재 비어 있음) | `common/data/em_emulatorCD.java` |

## 죽은/미구현 코드
- pcf/vcore/wifi Function의 `initialized()` — `if(1==1){return;}`로 인증/콘텐츠 교환 초기화 전부 skip. 실제 장비 연동 전환 시 이 블록 제거 필요.
- `WorkerRest4vCore.processControl()`의 `EMS` 분기 — 본문 stub, 빈 응답 반환.
- `em_emulatorCD.WorkTypeBook` — 비어 있음.

## mock/fake 지점
- `RestInvokerFunction_*.processMicroServiceRestRequest()` — 하드코딩 JSON 반환이므로 응답값 직접 교체 가능(별도 mock 불필요).
- 외부 EMS 실호출은 `WorkerRestContentManager.sendHttpRequest()` 한 곳 — mock 대상 핵심.

## 변경 위험
- smp 외 Function 미등록 상태를 모르고 pcf/vcore/wifi 수정만 하면 "추가했는데 안 됨" — 로더 addFunction 등록 여부 먼저 확인.
- `if(1==1)` 비활성 패턴을 정상 로직으로 오인해 그 아래 dead code를 수정하는 실수 주의.

## 근거 파일
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.em_emulator/src/main/java/hfr/plugin/my5g/service/em_emulator/ServiceStart_em_emulator.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.em_emulator/src/main/java/hfr/plugin/my5g/service/em_emulator/manual/manualWorkerLoader4Service.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.em_emulator/src/main/java/hfr/plugin/my5g/service/em_emulator/function/` (RestInvokerFunction_pcf/smp/vcore/wifi)
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.em_emulator/src/main/java/hfr/plugin/my5g/service/em_emulator/worker/WorkerRest4vCore.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.em_emulator/src/main/java/hfr/plugin/my5g/service/em_emulator/common/data/em_emulatorCD.java`
