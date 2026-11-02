# vCore_mgmt Service

## 서비스 책임
5G vCore(가상 코어망 장비)와 OpenAPI 규격으로 연동하여 세션(로그인/토큰) 관리, 알람 수집·동기화, 로그 레벨 제어를 수행하고, gateway_uems에 인증/콘텐츠 등록을 통해 uEMS 플랫폼에 vCore 관리 기능을 제공하는 support 마이크로서비스. 서비스 자체 DB 없이 타 마이크로서비스(basedata, fault, gateway_uems) REST 위임으로 동작한다.

## 주요 입력
- vCore OpenAPI 응답: 로그인 토큰, 현재 알람 목록, 알람 subscription notification
- basedata의 vCore EM 목록, fault의 활성 알람 목록
- 환경변수 `VCORE_URL`(vCore 접속 주소), `LOCAL_DOMAIN`(K8s 서비스 도메인), 설정 `em.type`

## 주요 출력
- fault 서비스로 신규/해소 알람 등록(REST)
- gateway_uems 인증 및 콘텐츠(ex-notification, log-level) 등록
- vCore로 로그 레벨 설정, 알람 subscription 생성/갱신 요청

## 시작점과 실행 설정
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.vCore_mgmt/src/main/java/hfr/plugin/my5g/service/vCore_mgmt/ServiceStart_vCore_mgmt.java`, `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.vCore_mgmt/home/config/vcore_mgmt.xml`
- `tServiceName = "vcore_mgmt"`, `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- 포트: http 10452 (`vcore_mgmt.xml`)
- `em.type=site`이면 초기화 스레드 미실행. 그 외에는 `Worker_vCore_mgmt.processEMSStart()`를 백그라운드 스레드로 기동
- `CommonMybatisDBConfiguration`이 등록되어 있으나 서비스 내 Mybatis 매퍼 없음(공통 프레임워크 초기화 용도)

## 아키타입
**E(외부연동: vCore OpenAPI adapter)** — `../feature-extension.md` 참조. unify Auto/·inf/manualWorkerLoader 등록 구조가 없다.
- ② Worker: `Worker_vCore_mgmt` 단일 Worker. 신규 메서드 추가 방식으로 확장(별도 로더 등록 없음)
- ③ Scheduler: `ScheduleInterWork_vCore` — `isAutoRun()=false`, `Worker_vCore_mgmt.processEMSStart()`의 `manualRunRepeat(5000)` 호출로만 기동

## REST contract
근거: `function/RestInvokerFunction_vCore_mgmt.java` (getContentList)
- `ex-notification` (`MicroServiceCommonCD.RequestTypeStringBook.ex_Notification`, POST) — vCore 알람 subscription notification 수신용
- `log-level` (`RequestTypeStringBook.Log_level`, GET|POST) — 로그 레벨 조회/설정

⚠ 두 type 모두 `processMicroServiceRestRequest()` 내 처리 본문이 주석 처리 상태다. ex-notification 수신 처리는 현재 동작하지 않고, log-level 실제 호출은 Worker/Schedule에서 직접 수행된다. 수정 전 활성 코드 범위를 먼저 확인할 것.

## 주요 Worker/Function 흐름
근거: `worker/Worker_vCore_mgmt.java`
- `processEMSStart(customerID, emsName, account, password)`: 비트 플래그 `m_flag`(AtomicInteger)로 6단계 초기화를 관리 — 각 단계 성공 시 1/2/4/8/16/32 비트를 세우고, 전 단계(63) 완료 후 `m_CoreScheduleJob.manualRunRepeat(5000)`으로 스케줄을 기동한다. 단계에는 basedata EM 목록 조회, fault 활성 알람 조회, vCore 로그인, 초기 알람 동기화(`processCoreAlarmSync()`), gateway_uems 인증/콘텐츠 등록이 포함된다.
- `processEms_Exchange_contents_4GateEms()`: gateway_uems에 이 서비스가 제공하는 REST 콘텐츠를 `tSendJsonObject` 배열로 등록. gateway 인증 토큰(`m_my_AccessToken4GateEms`)은 메모리에만 유지된다.
- 타 서비스 의존: basedata(Em 목록), fault(알람 조회/등록), gateway_uems(인증/콘텐츠) — 모두 `WorkerRestContentManager.executeRestAPI()` 경유.

## DB/Mongo/파일 흐름
서비스 자체 Mapper/DAO 없음. DB 접근은 전부 타 마이크로서비스 REST 위임(basedata: Em, fault: Alarm). Mongo/파일 흐름 없음.

## scheduler 흐름
근거: `schedule/ScheduleInterWork_vCore.java`
- `ScheduleInterWork_vCore`: interval 5000ms, `manualRunRepeat`으로만 기동. **`execute()` 본문(정기 알람 동기화)이 주석 처리되어 현재 주기 작업은 없다** — 알람 수집은 초기화 시 `processCoreAlarmSync()` 1회만 동작.
- vCore OpenAPI 엔드포인트(모두 `getvCoreURL()` + `JsonRequest()` 패턴, `vCoreSession` 토큰 헤더 사용):
  - `/api/vcore/login/` — 로그인/토큰 발급
  - `/api/vcore/current-alarms/` — 현재 알람 조회
  - `/api/vcore/log/level/{module}/` — 로그 레벨 조회/설정
  - `/api/vcore/subscribe/alarm/`(+`/{subscriptionID}/`) — 알람 subscription 생성/갱신/확인
- vCore URL 공급: 환경변수 `VCORE_URL`(운영) 또는 디버그 모드 고정값. notification 수신 주소는 `LOCAL_DOMAIN` 기반으로 조립.

## 기능 확장점 (feature seams)
| seam 유형 | 추가 방법 | 등록 위치/근거 파일 |
| --- | --- | --- |
| ① rest-request-type | `RequestTypeStringBook`(MicroServiceCommonCD)에 상수 선언 → `getContentList()`에 `new RestContent(this, <type>, HttpMethodBook.<메서드>, getServiceName(), "v1", true)` 추가 → `processMicroServiceRestRequest()`에 type 분기 추가 | `function/RestInvokerFunction_vCore_mgmt.java` |
| ② worker-function | `Worker_vCore_mgmt`에 public 메서드 추가(단일 Worker 구조, 별도 로더 등록 불필요). Function에서 쓸 경우 `RestInvokerFunction_vCore_mgmt.initialize()`의 `m_Worker` 경유 위임 | `worker/Worker_vCore_mgmt.java` |
| ③ scheduler-job | `ScheduleInterWork_vCore.execute()`에 주기 작업 추가(주석 처리된 알람 동기화 복원 포함). interval 변경은 `getInterval()` override. 기동은 `processEMSStart()`의 `manualRunRepeat(5000)` 경유임을 유의 | `schedule/ScheduleInterWork_vCore.java` |
| external-adapter (아키타입 E) | 새 vCore API 호출은 `ScheduleInterWork_vCore`에 메서드 추가 — `getvCoreURL(em) + "/api/vcore/..."` + `JsonRequest()` 패턴, `vCoreSession` 토큰을 헤더에 사용 | `schedule/ScheduleInterWork_vCore.java` |
| data-model | vCore 수신 데이터 VO는 `common/data/` 아래 `vCoreAlarm.java` 패턴(공통 unify 모델 변환 메서드 `toAlarm()` 포함)으로 추가. 새 알람 코드(Ocode) 매핑은 `vCore_mgmtCD.AlarmMapBook`에 추가 | `common/data/vCoreAlarm.java`, `common/data/vCore_mgmtCD.java` |
| gateway 콘텐츠 노출 | 새 request type을 gateway_uems에 노출하려면 `processEms_Exchange_contents_4GateEms()`의 `tSendJsonObject` 배열에 콘텐츠 JSONObject 추가 | `worker/Worker_vCore_mgmt.java` |

## product 확장점
없음. product별 override 구조(AbsFunction/AbsWorker, product 폴더)가 이 서비스에 존재하지 않는다. `RestInvokerFunction`은 `AbsRestInvokerFunction_Service` 직접 상속, `getEmsTypecode()`는 `productCD.ProductTypeCode.plugin_comm` 고정(단일 product 전용).

## 죽은/미구현 코드
- `RestInvokerFunction_vCore_mgmt.processMicroServiceRestRequest()` — ex-notification/log-level 처리 본문 전부 주석 = 수신 REST 사실상 no-op
- `ScheduleInterWork_vCore.execute()` — 정기 알람 동기화 본문 주석 = 주기 작업 없음. 주석 해제 시 초기 `processCoreAlarmSync()`와 이중 처리 위험

## mock/fake 지점
1. `WorkerRestContentManager.executeRestAPI()` — basedata/fault/gateway_uems 호출 모킹
2. `WorkerRestContentManager.JsonRequest()` — vCore 외부 REST(/api/vcore/*) 호출 모킹
3. `EmpConfig4Server.getString()` — `em.type`, `em.url`, `site.customer_id` 등 설정 모킹
4. `System.getenv("VCORE_URL")`, `System.getenv("LOCAL_DOMAIN")` — 환경변수 모킹

## 변경 위험
- `processEMSStart()`의 비트 플래그(`m_flag`) 초기화 단계 추가 시 비트 충돌 주의(현재 1~32 사용)
- `VCORE_URL`/`LOCAL_DOMAIN` 환경변수 누락 시 subscription URL 조립 실패로 무한 대기 루프
- `execute()` 주석 해제 시 초기 알람 동기화와 이중 처리 위험
- gateway_uems 토큰이 메모리 전용 → 서비스 재시작 시 재인증 필요
- `em.type=site` 설정 시 vCore 연동 초기화가 아예 실행되지 않음 — 배포 설정 확인 필수

## 근거 파일
서비스 루트 = `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.vCore_mgmt/src/main/java/hfr/plugin/my5g/service/vCore_mgmt/`
- `ServiceStart_vCore_mgmt.java`
- `function/RestInvokerFunction_vCore_mgmt.java`
- `worker/Worker_vCore_mgmt.java`
- `schedule/ScheduleInterWork_vCore.java`
- `common/data/vCoreAlarm.java`, `common/data/vCoreSession.java`, `common/data/vCore_mgmtCD.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.vCore_mgmt/home/config/vcore_mgmt.xml`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/MicroServiceCommonCD.java` (RequestTypeStringBook; 문자열 정의는 `MicroServiceCommonCD_my5G.java`)
