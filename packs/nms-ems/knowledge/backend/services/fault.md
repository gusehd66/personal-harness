# Fault Service

## 서비스 책임
Fault Service는 장비(NE/EMS)로부터 수신한 Alarm·Event를 `unify_fault` DB에 저장하고, 현재 경보(active/current alarm) 관리·이력 조회·알림(notification/email/OSS 연동)·NOC/Site 동기화·NE comfail 세션 체크·데이터 shrink를 담당한다.

## 주요 입력
- 장비 alarm/event notification (product parser 경유)
- alarm/event 조회 REST request
- NOC/uEMS/Site alarm·event sync request
- NE session 상태 (comfail 체크)
- user/service status 관련 상태

## 주요 출력
- `unify_fault` DB row (alarm/event/service_status/notification 계열)
- active/current alarm 응답, alarm/event history 응답, alarm summary
- notification/email/OSS 연계 결과, NOC/Site sync 응답
- comfail 상태 (Ne state 업데이트)

## 시작점과 실행 설정
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/ServiceStart_Fault.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/FaultServerConfig.java`

확인 기준:
- `tServiceName = "fault"`, DB = `unify_fault`(MariaDB)
- `runMicroService.run("fault", new DefaultSecurityHttps(), tDBConfig)`
- `FaultServerConfig`는 `ServerConfigOnlyDB` 상속. `getMapperList()/getJavaTypeList()`를 `unify_fault.getMapperList()/getJavaTypeList()`로 위임
- `processDBUpgrade`에서 `update.sql` 실행 + `processAdd_Update_time("unify_fault")` 호출

## 아키타입
`../feature-extension.md` 기준 **A(unify 자동등록) + E(외부연동)**.
- ② Worker 등록 위치: `unify_fault.java`의 Auto Generated 영역(`toDataRequestTypeListAuto()/toWorker4Auto()/toDataClass4Auto()`)
- ③ Schedule/외부연동 Function 등록: `Worker4ProcessFunction` + `FunctionTypeBook`(`fault`, `alarmDataSync`) — product jar가 런타임에 자동 수집됨

## REST contract
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/function/RestInvokerFunction_Fault.java` (getContentList line 253~311)

직접 등록 request type:
- alarm/event 수신·조회: `Ems_alarm`, `Ems_event`, `Ems_sync_alarm`, `Ems_sync_event`, `Ems_sync_info_alarm`, `Ems_sync_info_event`, `Ems_sync_active_alarm`, `Ems_Memory_Active_Alarm`, `Ems_alarm_summary`
- 상위/외부 연동: `Uems_alarm`(Inner scope), `Raised_alarm`, `Raised_event`, `alarm_sync`, `noc_alarm_sync`, `noc_event_sync`, `Site_Data_Sync`(2026-03-09 NOC 연동, `JsonDataFormatBook.Site_Data_Sync.requestType`으로 등록), `Request_Oss_data`
- 유틸: `Get_Event_Type`, `Get_target_info`, `UserManagement_Sync`, `Alarm_Shrink`, `Event_Shrink`

unify_fault 자동등록 CRUD type (`unify_fault.toDataRequestTypeList()` 경유):
- `alarm`, `alarm_action`, `alarm_definition`, `alarm_def_action`, `event`, `notification_alarm_detail`, `notification_group`, `service_status`, `service_status_detail`, `service_status_log`

## 주요 Worker/Function 흐름
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/worker/ProcessMainWorker_Fault.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/worker/WorkerFaultDataProcess.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/function/AlarmDataProcessorFunction.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/function/EventDataProcessorFunction.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/common/worker/unify_fault/`

핵심 클래스:
- `ProcessMainWorker_Fault` / `WorkerFaultDataProcess` — fault 데이터 수신·저장 메인 경로
- `WorkerAlarm` / `WorkerEvent` — alarm/event 테이블 worker, `convert_AlarmToFaultData()`, active alarm 캐시(`queryCurrentAlarmCollection()`)
- `WorkerAlarm_action` / `WorkerAlarm_definition` / `WorkerService_status` / `WorkerNotification_group` — unify_fault 테이블별 CRUD worker
- `WorkerSMTPEmailAlarmSend`(email 발송) / `FaultNotificationSender`(notification 발송) — 알림 이중 경로 주의
- `WorkerOSSDataProcess` — `Request_Oss_data` → `getOSSAlarmData()/getOSSEventData()`
- `Work4NeTrapSerialNumber` — NE trap serial number 관리

NOC/Site 동기화 흐름:
- `noc_alarm_sync` → `AlarmDataProcessorFunction.processAlarmSyncNotification()` / `noc_event_sync` → `EventDataProcessorFunction.processEventSyncNotification()` (worker/`NocAlarmDataSyncFunction`, `NocEventDataSyncFunction` 연계)
- `Site_Data_Sync` → category(alarm/event)에 따라 `AlarmDataProcessorFunction`/`EventDataProcessorFunction`의 `processGetSiteDataJsonObject()` 호출 (RestInvokerFunction_Fault line 879~). `processBypassResponseHandler()`에서 Site NMS 응답의 alarm/event 배열을 동기화 후 무거운 data 필드를 제거하여 반환

## DB/Mongo/파일 흐름
- MariaDB `unify_fault`. Mongo 사용 없음.
- Mybatis mapper XML: `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/mybatis/unify_fault/` (mybatis_alarm.xml 등 테이블별 10개)
- 테이블: alarm, alarm_action, alarm_definition, alarm_def_action, event, notification_alarm_detail, notification_group, service_status, service_status_detail, service_status_log — 각각 Worker+Dao 쌍

## scheduler 흐름
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/schedule/`

- `NeSessionScheduleJob` — NE comfail 세션 체크. `initialized()`에서 `AbsComfailProcessFunction`을 `<producttype, <protocolCode, function>>` map으로, `AbsFunctionFindNeProtocol`을 productcode별 map으로 수집. 실행 시 NE의 productcode/protocol로 function을 찾아 comfail 판정 → Ne state 업데이트. **`m_FindNeProtocolFunctionMap` null이면 조기 return으로 세션 체크가 건너뛰어짐**
- `ScheduleAlarmSync` — `initialized()`에서 `FunctionTypeBook.alarmDataSync` 등록 function을 NE code 단위 `alarmSyncFunctionMap`으로 수집, NE별 능동 알람 동기화
- `ScheduleShrinkFaultData` — 1분 주기, `isDBShrikRunOK` 조건 확인 후 alarm/event 데이터 축약
- `ScheduleUserSync` — 사용자 정보 동기화

## 기능 확장점 (feature seams)
| seam | 추가 방법 | 등록 위치 / 근거 파일 |
| --- | --- | --- |
| ① REST request type | `RestInvokerFunction_Fault.getContentList()`에 `new RestContent(this, RequestTypeStringBook.{새값}, HttpMethodBook.POST, ...)` 한 줄 + `processMicroServiceRestRequest()` else-if 분기 + `RequestTypeStringBook` 상수 선언 | `function/RestInvokerFunction_Fault.java` (line 253~311) |
| ② Worker/Function (새 테이블) | `unify_fault.java`의 `toDataRequestTypeListAuto()/toWorker4Auto()/toDataClass4Auto()`에 requestType·Worker·Data 클래스 등록(Auto 영역) + `common/worker/unify_fault/<entity>/` 아래 Worker 클래스 생성 + `MybatisMapperBook`에 Dao 등록 | `common/worker/unify_fault/unify_fault.java` (line 39~295) |
| ③ Scheduler job | `ScheduleAdapterRat` 상속 job을 `schedule/`에 작성. `getDefine_class()/execute()/getInterval()/isAutoRun()` 오버라이드. 필요 function은 `ScheduleAlarmSync`처럼 `initialized()`에서 `Worker4ProcessFunction` 경유 주입 | `schedule/ScheduleAlarmSync.java`, `ScheduleShrinkFaultData.java`, `NeSessionScheduleJob.java` |
| ④ DB mapper | `unify_fault.MybatisMapperBook`에 Ocode 추가 → `getMapperList()`에 XML 경로 등록. XML은 `uEMS.plugin.my5g.Common/.../mybatis/unify_fault/`에 추가. `FaultServerConfig`는 위임이라 수정 불필요 | `FaultServerConfig.java` (line 29~31), `unify_fault.java` (line 233~263) |
| ⑤-a Alarm 파싱 (product) | `AbsFunctionFaultDataParser` 상속 클래스를 product 서비스에 작성, `FunctionTypeBook.fault`로 `Worker4ProcessFunction`에 등록 → `RestInvokerFunction_Fault.initialized()`의 `m_NotifyEventFunctionList`에 자동 수집 | `function/RestInvokerFunction_Fault.java` (line 215~249) |
| ⑤-b Alarm 동기화 (product) | `AbsFunctionAlarmDataSync` 상속 클래스를 product 서비스에 작성, `FunctionTypeBook.alarmDataSync`로 등록 → `ScheduleAlarmSync.initialized()`의 `alarmSyncFunctionMap`에 NE code 단위 자동 수집 | `schedule/ScheduleAlarmSync.java` (line 64~78) |
| ⑤-c comfail 판정 (product) | `AbsComfailProcessFunction` 구현 추가 (protocol·producttype별) → `NeSessionScheduleJob`이 자동 수집 | `schedule/NeSessionScheduleJob.java` (line 118~176) |

## product 확장점
공통 service 코드 수정 없이 product jar 추가만으로 확장되는 3축:
1. **`AbsFunctionFaultDataParser`** (알람 파싱): my5G/vran_ems — `FunctionFaultDataParserCUDU`·`RU`·`vCore`·`My5GBox`; openhaul — `FunctionFaultDataParserOpenHaul`·`FSH2`; sixgen — `FunctionFaultDataParserSixGen`
2. **`AbsFunctionAlarmDataSync`** (능동 동기화): my5G — `FunctionAlarmDataSyncCUDU`·`RU`·`FHM`·`My5GBox`; openhaul — `FunctionAlarmDataSync_openHaul`·`FSH2`; sixgen — `FunctionAlarmDataSyncSixGen`
3. **`AbsComfailProcessFunction`** (comfail 판정): my5G·openhaul·sixgen·wifi 각각 구현

근거:
- `<BACKEND_ROOT>/product/my5G/my5g.Service.vran_ems/src/main/java/hfr/plugin/my5g/service/vran_ems/hiems/function/alarm/` (FunctionAlarmDataSyncCUDU/RU/FHM/My5GBox)
- `<BACKEND_ROOT>/product/my5G/my5g.Service.vran_ems/src/main/java/hfr/plugin/my5g/service/vran_ems/hiems/function/notification/` (FunctionFaultDataParser*)
- `<BACKEND_ROOT>/product/openhaul/my5g.Service.openHaul/src/main/java/hfr/plugin/my5g/service/openHaul_ems/hiems/function/` (FunctionAlarmDataSync_openHaul, FunctionAlarmDataSyncFSH2, FunctionFaultDataParserFSH2)
- `<BACKEND_ROOT>/product/sixgen-ems/my5g.Service.six_gen_flexible/src/main/java/hfr/plugin/my5g/service/six_gen_flexible/hiems/function/alarm/FunctionAlarmDataSyncSixGen.java`

## 죽은/미구현 코드
- `RestInvokerFunction_Fault.getContentList()` — `system_system_config` RestContent는 주석 제거됨(`System_config.requestType`으로 대체, 2022-07-05)
- `RestInvokerFunction_Fault.processBypassRequestHandler()`의 `Site_Data_Sync` 분기 — 본문 대부분 주석(실제 동기화는 `processBypassResponseHandler()` 쪽이 담당)

## mock/fake 지점
- FaultData 입력 payload (alarm/event), `WorkerAlarm.convert_AlarmToFaultData()`
- `WorkerAlarm.queryCurrentAlarmCollection()` — active alarm 초기 상태
- `AlarmDataProcessorFunction.processAlarmSyncNotification()` — noc_alarm_sync mock
- `WorkerSMTPEmailAlarmSend.addEmailSendMessage()` — email 발송 mock
- `WorkerOSSDataProcess.getOSSAlarmData()/getOSSEventData()` — OSS 연동 mock
- `NeSessionScheduleJob.JobWorking()` — comfail 판정 function mock (`AbsComfailProcessFunction`)

## fixture 기준
- alarm/event input payload, current alarm 초기 상태, clear/ack/severity 상태
- NE session 상태 (productcode/protocol 포함 — comfail function 매칭용)
- expected active alarm/history row, notification mock 결과
- NOC/Site sync 입력 JSON (category=alarm/event)

## 변경 위험
- alarm 중복 생성 (`process_Raised_alarm`/`process_AddFaultDataList` 경로 중복)
- active alarm clear 누락 시 현재 경보 불일치
- `noc_alarm_sync`/`noc_event_sync` 처리 시 `AlarmDataProcessorFunction` 미초기화 NPE
- `ScheduleAlarmSync`의 `alarmSyncFunctionMap` null 상태 실행 시 NE 알람 동기화 누락
- comfail 오판: `NeSessionScheduleJob`의 `m_FindNeProtocolFunctionMap` null 체크 후 조기 return으로 세션 체크 건너뜀
- notification 중복 발송 (`WorkerSMTPEmailAlarmSend` + `FaultNotificationSender` 이중 경로)
- `ScheduleShrinkFaultData` 1분 주기 실행 시 `isDBShrikRunOK` 조건 미충족 중복 실행 가능성

## 근거 파일
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/ServiceStart_Fault.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/FaultServerConfig.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/function/RestInvokerFunction_Fault.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/function/AlarmDataProcessorFunction.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/function/EventDataProcessorFunction.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/common/worker/unify_fault/unify_fault.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/worker/ProcessMainWorker_Fault.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/worker/WorkerFaultDataProcess.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/schedule/NeSessionScheduleJob.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/schedule/ScheduleAlarmSync.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/schedule/ScheduleShrinkFaultData.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Fault/src/main/java/hfr/plugin/my5g/service/Fault/schedule/ScheduleUserSync.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/mybatis/unify_fault/mybatis_alarm.xml`
