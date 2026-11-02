# Notify_mgmt Service

## 서비스 책임
마이크로서비스 간 알림(Notification)의 UniCast/BroadCast/RunProcessBroadCast 전송과 NOC(상위 관제) 연동 알림 전달을 전담하는 알림 중계(broker) 서비스. 자체 도메인 데이터는 없고, 받은 알림을 대상 서비스 또는 NOC로 재전송한다.

- 주요 입력: 타 서비스가 POST하는 `notification-request` / `notification-request-to-noc` 내부 REST 요청
- 주요 출력: 대상 마이크로서비스로의 `Received_Notification` 재전송, gateway_uems 경유 NOC 발송

## 시작점과 실행 설정
- `ServiceStart_Notify_mgmt.java` — `tServiceName = "notify_mgmt"`, `runMicroService.run("notify_mgmt", new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- HTTP 포트 `10446` — `home/config/notify_mgmt.xml` `<port>10446</port>`
- REST 서비스명은 `RestInvokerFunction_Notify_mgmt.getServiceName() = "notify-mgmt"` (하이픈 주의)
- 근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Notify_mgmt/src/main/java/hfr/plugin/my5g/service/Notify_mgmt/ServiceStart_Notify_mgmt.java`

## 아키타입
**C (.inf 플러그인 로더)** — ②③ 등록 위치는 `home/jar/notify_mgmt.inf`. 12개 FQCN 선언(공통 Worker 10개 + 서비스 고유 2개: `RestInvokerFunction_Notify_mgmt` line 11, `Worker_Notify_mgmt` line 12). **새 Worker/Function을 만들면 반드시 이 파일에 FQCN 한 줄을 추가해야 로딩된다.** scheduler는 없다.

## REST contract
등록: `RestInvokerFunction_Notify_mgmt.getContentList()` (line 245-246), 둘 다 `HttpMethodBook.POST` + `ScopeTypeBook.Inner`(내부 전용).

| request type | 상수 | 처리 |
| --- | --- | --- |
| `notification-request` | `MicroServiceCommonCD.RequestTypeStringBook.Notification_Request` | 대상 서비스로 재전송 (UniCast/BroadCast/RunProcessBroadCast 분기) |
| `notification-request-to-noc` | `...RequestTypeStringBook.Notification_To_NOC` | `NOCNotificationTask` 생성 → PendingQueue 투입 |

상수 문자열의 실제 정의는 `platformRatCD.RequestTypeStringBook` (`<BACKEND_ROOT>/Platform_framework/EMSPlatform/src/hfr/platform/Rat/product/common/data/platformRatCD.java` line 257, 260).

## 주요 Worker/Function 흐름
- `RestInvokerFunction_Notify_mgmt` (`AbsRestInvokerFunction_my5GService` 상속)
  - `processMicroServiceRestRequest()` line 148: `Notification_Request` 분기 — request_type을 `Received_Notification`으로 바꿔 `Worker_Notify_mgmt`의 send 메서드로 위임. `isConsign()`(line 182)이면 NOC 발송도 병행.
  - line 189: `Notification_To_NOC` 분기 — `NOCNotificationTask`를 `m_PendingQueue.add()`.
  - `initialize()` line 83-87: `aWorkFlowMap.getWorker(Worker_Notify_mgmt.class)` 의존성 주입 + `registerProcessor(NOCNotificationTask.class, new NOCNotificationTaskProcessor(...))` + `m_PendingQueue.start()`.
- `Worker_Notify_mgmt` — 전송 경로 3+1종:
  - `sendNotifyUniCast()` (line 96) / `sendNotifyBroadCast()` (line 145) / `sendNotifyRunProcessBroadCast()` (line 118) / `sendNotifyToNOC()` (line 85)
  - 실제 발송은 `NotifierProcess`(대상 서비스) / `NotifierCommonProcess`(→ `SupportService.gateway_uems`) 큐에 `addAction()`으로 등록.
- `NOCNotificationTaskProcessor` — notifyType별 dispatch: `BaseData_Sync`(line 38) / `AlarmData`(line 55) / `EventData`(line 68). **그 외 타입은 아무 처리 없이 done 반환.** 변환/발송은 `WorkerNocNmsDataMapper`(공통 플러그인)에 위임.

### NOC 위탁(consign) 판별
`RestInvokerFunction_Notify_mgmt.isConsign()` (line 223) — System_config의 ConsignNOC 설정 조회 결과를 `UtilCache<String, System_config>(256, 300)` (line 222, **TTL 300초**)에 캐싱. 설정 변경 후 최대 5분 지연 반영.

### PendingQueue 재시도 정책
`DefaultPendingQueue` 생성 (line 54-62): `ExponentialBackoffRetryPolicy(init=10ms, max=100ms, factor=2.0)`, worker 4개, timeout 15000ms, capacity 40, retry 5회. 로깅은 `LogListener4Notify_mgmt`, 직접 처리 핸들러는 `DirectTaskHandler4Notify_mgmt`.

## DB/Mongo/파일 흐름
자체 mapper/xml 없음. `CommonMybatisDBConfiguration` 공통 설정만 사용. DB 접근은 `NOCNotificationTaskProcessor` → `WorkerNocNmsDataMapper`(공통 플러그인) 경유 간접 접근뿐. Mongo/파일 흐름 없음. `Notify_mgmtCD.java`는 WorkTypeBook 상수 정의만 있음.

## scheduler 흐름
없음 (schedulers 0개). 백그라운드 처리는 PendingQueue와 NotifierProcess 큐가 전부.

## 기능 확장점 (feature seams)
| seam | 추가 방법 | 등록 위치 / 근거 |
| --- | --- | --- |
| ① REST request type | `getContentList()`에 `new RestContent(this, 새Type, HttpMethodBook.POST, getServiceName(), "v1", ScopeTypeBook.Inner)` 추가 + `processMicroServiceRestRequest()` if-else 분기 추가. 상수는 `platformRatCD.RequestTypeStringBook`(또는 `MicroServiceCommonCD.RequestTypeStringBook`)에 선언 | `.../function/RestInvokerFunction_Notify_mgmt.java` line 148, 189, 245-246 |
| ② Worker/Function | `Worker_Notify_mgmt`에 send 메서드 추가, 또는 `abstractWorker` 상속 신규 Worker 작성 후 **`home/jar/notify_mgmt.inf`에 FQCN 한 줄 추가**. `RestInvokerFunction_Notify_mgmt.initialize()`에서 `aWorkFlowMap.getWorker(새Worker.class)`로 주입 | `.../home/jar/notify_mgmt.inf` line 11-12; `RestInvokerFunction_Notify_mgmt.java` line 80-88 |
| NOC notifyType 추가 (external-adapter) | `AbsFunctionNocNmsDataMapper` 상속 클래스 작성 — `getDataType()`이 새 notifyType 키 반환, `convertToNocFormat`/`convertToNmsFormat`/`executeNoificationAction` 구현 — 후 `notify_mgmt.inf`에 FQCN 등록. `WorkerNocNmsDataMapper`가 자동 HashMap 등록 → `NOCNotificationTaskProcessor`에서 dispatch. **NOCNotificationTaskProcessor의 notifyType if-else에도 분기 추가 필요**(없으면 무시됨) | `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/function/AbsFunctionNocNmsDataMapper.java`; `.../common/worker/WorkerNocNmsDataMapper.java` line 38-41; `.../pendigqueue/NOCNotificationTaskProcessor.java` line 38-79 |
| PendingTask 타입 추가 | PendingTask 페이로드 클래스 작성 + `PendingTaskProcessor<새Task>` 구현 + `initialize()`에 `m_PendingQueue.registerProcessor(새Task.class, new 새Processor(...))` 추가. 기존 `NOCNotificationTask`/`NOCNotificationTaskProcessor` 쌍의 패턴을 따른다 | `RestInvokerFunction_Notify_mgmt.java` line 53-63, 86 |

## product 확장점
product별 override 폴더 없음. 공통 추상 레이어 상속만 존재:
- `RestInvokerFunction_Notify_mgmt` ← `AbsRestInvokerFunction_my5GService` (`<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/.../function/AbsRestInvokerFunction_my5GService.java`)
- NOC-NMS 데이터 변환은 `AbsFunctionNocNmsDataMapper` 구현체를 `.inf`에 등록하는 방식 — 현재 서비스 내 구체 구현체는 없고 공통 플러그인 레이어에서 주입된다.

## 죽은/미구현 코드
- `DirectTaskHandler4Notify_mgmt.handle()` — `// TODO Auto-generated method stub` 빈 구현. PendingQueue를 우회하는 직접 처리 경로는 **동작하지 않는다**.
- `Worker_Notify_mgmt.sendNotifyRunProcessBroadCast()` 내 동일 RunProcess 중복 전송 방지 로직 주석 처리(line 161 부근) — 의도된 동작 불명확, 수정 전 확인 필요.

## mock/fake 지점
1. `WorkerRestContentManager.executeRestAPI()` — 타 마이크로서비스 REST 호출 mock 대상
2. `NotifierProcess.addAction()` / `NotifierCommonProcess` — 실제 발송 전 큐 등록 지점
3. `NOCNotificationTaskProcessor.process()` — NOC 연동 단위 테스트 진입점
4. `WorkerNocNmsDataMapper.convertToNocFormat()` / `executeNoificationAction()` — 데이터 변환 mock 대상

## 변경 위험
- `NOCNotificationTaskProcessor`의 notifyType 분기: BaseData_Sync/AlarmData/EventData 외 타입은 조용히 done 반환 → 새 타입 추가 시 분기 누락하면 알림이 무시된다.
- `isConsign()` UtilCache TTL 300초 → ConsignNOC 설정 변경이 최대 5분 늦게 반영된다.
- `.inf` FQCN 미등록 → 새 Worker/Function 클래스를 만들어도 로딩되지 않음 (아키타입 C의 흔한 실수).
- PendingQueue capacity 40 / retry 5회 — 대량 NOC 알림 시 큐 포화 가능, 파라미터 변경은 line 54-62에서만.
- `DirectTaskHandler4Notify_mgmt` 미구현 상태에서 직접 처리 경로를 전제로 한 수정 금지.

## 근거 파일
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Notify_mgmt/src/main/java/hfr/plugin/my5g/service/Notify_mgmt/ServiceStart_Notify_mgmt.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Notify_mgmt/src/main/java/hfr/plugin/my5g/service/Notify_mgmt/function/RestInvokerFunction_Notify_mgmt.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Notify_mgmt/src/main/java/hfr/plugin/my5g/service/Notify_mgmt/worker/Worker_Notify_mgmt.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Notify_mgmt/src/main/java/hfr/plugin/my5g/service/Notify_mgmt/common/pendigqueue/NOCNotificationTaskProcessor.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Notify_mgmt/src/main/java/hfr/plugin/my5g/service/Notify_mgmt/common/pendigqueue/DirectTaskHandler4Notify_mgmt.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Notify_mgmt/home/jar/notify_mgmt.inf`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Notify_mgmt/home/config/notify_mgmt.xml`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/function/AbsFunctionNocNmsDataMapper.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/worker/WorkerNocNmsDataMapper.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/function/AbsRestInvokerFunction_my5GService.java`
- `<BACKEND_ROOT>/Platform_framework/EMSPlatform/src/hfr/platform/Rat/product/common/data/platformRatCD.java`
