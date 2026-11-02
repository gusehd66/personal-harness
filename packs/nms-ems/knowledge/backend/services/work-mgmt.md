# Work_mgmt Service

## 서비스 책임
현장 작업지시서(Work Order)의 생성·배정·수락·완료·이력 관리와 AR 가이드 리소스, 작업 영상, 메시지 발송(FCM Push / SMTP 이메일), 원격 지원(Remote Support)을 통합 제공하는 작업관리 서비스다. 단일 product 구성(my5G/openhaul/sixgen-ems override 폴더 없음).

- 주요 입력: work-order 계열 REST 요청(생성/배정/수락/완료), 파일·영상·AR 리소스 업로드, 메시지 발송 요청
- 주요 출력: MariaDB CRUD 결과, FCM Push/SMTP 이메일 발송, 파일 다운로드 스트림, 타 서비스(usermanagement/system) REST 호출

## 시작점과 실행 설정
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Work_mgmt/src/main/java/hfr/plugin/my5g/service/Work_mgmt/ServiceStart_Work_mgmt.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Work_mgmt/src/main/java/hfr/plugin/my5g/service/Work_mgmt/Work_mgmtServerConfig.java`

확인 기준:
- `tServiceName = "work_mgmt"`, `setUseDataBase4MicroService(true)` (MariaDB DNS = `SupportService.maria_db` + `LOCAL_DOMAIN` 환경변수)
- `runMicroService.run(...)`에 `DefaultSecurityHttps`, `CommonMybatisDBConfiguration` 전달
- `Work_mgmtServerConfig`(`ServerConfigOnlyDB` 상속): `getServiceName()="work_mgmt"`, `getMapperList()`/`getJavaTypeList()`는 `unify_workmgmt`에 위임
- `processDBUpgrade()`가 `/hfr/plugin/my5g/service/Work_mgmt/updatesql/update.sql` 실행 (DDL 추가 위치)

## 아키타입
**A (unify 자동등록)** — `unify_workmgmt`. `../feature-extension.md` §A 절차 그대로 적용된다.

②③ 등록 위치:
- ② Worker: `common/worker/unify_workmgmt/unify_workmgmt.java`의 Auto-Generated 영역(`toWorker4Auto()`, `toDataClass4Auto()`, `toDataRequestTypeListAuto()`) — 등록만 하면 `processCommonData()` 범용 CRUD로 자동 라우팅
- ③ Scheduler: `schedule/` 폴더의 `ScheduleAdapterRat` 상속 클래스가 WorkflowMap에 등록됨. `Worker_Work_mgmt.initialize()`에서 `m_WorkflowMap.getScheduleJob(...)`으로 취득

## REST contract
근거: `function/RestInvokerFunction_Work_mgmt.java` `getContentList()` (295~316행 부근). 전부 POST + `ScopeTypeBook.Inner`가 기본, 예외는 표기.

| request type | 비고 |
| --- | --- |
| `work-order` / `work-order-user` / `work-order-action` / `work-order-ne` | Work Order CRUD 계열 |
| `work-order-file` / `file-upload` / `file-download` | 작업 첨부파일. download는 GET, 업로드는 InputStream 경로 |
| `video` / `video-upload` / `video-download` | 작업 영상. download는 GET |
| `work-guide` | 작업 가이드 |
| `work-remote-support` | 원격 지원 |
| `user-push` / `message-log` | Push 토큰·메시지 이력 |
| `arguide-resource` / `arguide-resource-upload` / `arguide-resource-zip-upload` / `arguide-resource-download` / `arguide-resource-map` | AR 가이드 리소스 |
| `request-user-ne-permission` | 유일한 non-Inner(외부 공개) endpoint. 상수는 `MicroServiceCommonCD.RequestTypeStringBook` |

주의: `work-order-alarm`, `message-template`은 도메인 엔티티·매퍼는 있으나 `getContentList()`에 RestContent 미등록 — 내부 Worker 경유로만 사용된다.

파일 업/다운로드 계열은 `RestContent` 생성자 마지막 boolean 인자로 스트림 처리(`processMicroServiceRestRequestUsingInputStream`)가 지정된다.

## 주요 Worker/Function 흐름
근거:
- `function/RestInvokerFunction_Work_mgmt.java` — request type → 분기. 표준 CRUD는 `processCommonData()`, 커스텀 WorkType은 `processMicroServiceRestRequest()` else-if 분기
- `worker/Worker_Work_mgmt.java` (1697줄, 비즈니스 로직 집중) — 핵심 메서드: `saveWorkOrder`, `updateWorkOrderStatus`, `acceptWorkOrderUser`, `resultReportingWorkOrder`, `checkWorkOrderSolve`, `addMessagelog`, `getRequestUserManagementService`
- `common/worker/unify_workmgmt/<entity>/WorkerXxx.java` — 테이블별 CRUD 워커 15종 (WorkerWork_order, WorkerWork_order_user, WorkerWork_order_action, WorkerWork_order_alarm, WorkerWork_order_file, WorkerWork_order_ne, WorkerWork_remote_support, WorkerMessage_log, WorkerMessage_template, WorkerUser_push, WorkerWork_guide, WorkerVideo, WorkerArguide_resource, WorkerArguide_resource_map)

2계층 구조: `Auto_Xxx`(자동생성 데이터) → `Xxx`(구체 데이터), `AutoWorkerXxx` → `WorkerXxx`. **Auto\* 파일은 수정 금지**, 커스터마이징은 항상 non-Auto 쪽에.

## DB/Mongo/파일 흐름
MyBatis(MariaDB) 전용, Mongo 없음. `unify_workmgmt.getMapperList()`가 14개 XML 매퍼 등록:
`mybatis_work_order.xml`, `mybatis_work_order_user.xml`, `mybatis_work_order_action.xml`, `mybatis_work_order_alarm.xml`, `mybatis_work_order_file.xml`, `mybatis_work_order_ne.xml`, `mybatis_work_remote_support.xml`, `mybatis_message_log.xml`, `mybatis_message_template.xml`, `mybatis_user_push.xml`, `mybatis_video.xml`, `mybatis_work_guide.xml`, `mybatis_arguide_resource.xml`, `mybatis_arguide_resource_map.xml`
(경로: `common/mybatis/unify_workmgmt/`, 키는 `unify_workmgmt.MybatisMapperBook.Dao_*`)

- 커스텀 쿼리: `Driver4MybatisIf`로 직접 호출. Worker는 `getMybatisNameSpace()` 반환값으로 쿼리 ID를 찾는다
- TypeHandler: `EDateTime`, `TimeSpan`
- DDL 변경: `updatesql/update.sql`에 append → 기동 시 `processDBUpgrade()` 자동 실행
- 파일: video/AR 리소스/작업 첨부는 업로드 스트림으로 저장 후 download endpoint로 서빙

## scheduler 흐름
근거: `schedule/` 폴더 5종. 모두 `ScheduleAdapterRat` 상속.

| Job | 역할 | 실행 조건 |
| --- | --- | --- |
| `WorkOrderScheduleJob` | Work Order 상태 점검(`checkWorkOrderSolve` 등) | `isAutoRun()` = `E_SystemType.NOC`일 때만 |
| `MessageSendScheduleJob` | message_log 큐 폴링 → FCM Push(`sendPush`) / SMTP 이메일(`sendEMail`) 발송 | NOC only |
| `RemoteSupportScheduleJob` | 원격 지원 세션 관리 | NOC 조건 확인 필요 |
| `ArGuideResourceScheduleJob` | AR 리소스 정리 | — |
| `UserPushScheduleJob` | Push 토큰 정리 | — |

FCM은 `GoogleCredentials.fromStream()`으로 classpath의 `hfr-push-firebase-adminsdk-fc2jt-*.json` service account 파일을 직접 로드한다.

## 기능 확장점 (feature seams)

| seam | 추가 방법 | 등록 위치 / 근거 파일 |
| --- | --- | --- |
| ① REST request type | `getContentList()`에 `new RestContent(this, "새type", HttpMethodBook.POST, getServiceName(), "v1", ScopeTypeBook.Inner)` 한 줄 + `processMicroServiceRestRequest()` else-if 분기 추가. 표준 CRUD면 `unify_workmgmt.toWorker4Auto()` 매핑도 등록. 파일 스트림이면 RestContent 마지막 boolean 인자 사용 | `function/RestInvokerFunction_Work_mgmt.java` |
| ② 신규 도메인 엔티티 | (1) `common/data/unify_workmgmt/<신도메인>/`에 `Auto_X.java`+`X.java`+`XCollection.java` (2) `common/worker/unify_workmgmt/<신도메인>/`에 `AutoWorkerX.java`+`WorkerX.java` (3) `unify_workmgmt.java`의 `toDataRequestTypeListAuto()`/`toWorker4Auto()`/`toDataClass4Auto()`/`getDataFieldList()`/`MybatisMapperBook`/`getMapperList()` 등록 (4) ServerConfig는 위임이므로 자동 반영 | `common/worker/unify_workmgmt/unify_workmgmt.java` |
| ③ Scheduler job | `schedule/`에 `ScheduleAdapterRat` 상속 클래스 작성 → `getDefine_class()`/`execute()`/`isAutoRun()`/`getInterval()` 구현. NOC 전용이면 `E_SystemType.NOC` 조건을 `isAutoRun()`에 | `schedule/WorkOrderScheduleJob.java` 패턴 복제 |
| ④ DB mapper | `common/mybatis/unify_workmgmt/`에 `mybatis_X.xml` 추가 + `MybatisMapperBook`에 Ocode + `getMapperList()`에 put + `update.sql`에 CREATE/ALTER. 기존 매퍼 쿼리 추가는 해당 xml만 수정 | `unify_workmgmt.java` (MybatisMapperBook, getMapperList) |
| 데이터 모델 필드 | `Auto_Xxx.java`는 건드리지 않고 `Xxx.java`에 필드/getter/setter 추가 + `update.sql`에 ALTER TABLE. WorkType 분기 확장은 `Work_mgmtCD.WorkTypeBook` 상수 추가 | `common/data/Work_mgmtCD.java`, `updatesql/update.sql` |
| ⑤ 외부연동 | `Worker_Work_mgmt.getRequestUserManagementService()` 패턴(=`m_WorkerRestContentManager.executeRestAPI()`)을 복제해 private 메서드 추가. 알림 채널 추가는 `MessageSendScheduleJob.sendPush()`/`sendEMail()` 영역 확장 | `worker/Worker_Work_mgmt.java`, `schedule/MessageSendScheduleJob.java` |

## product 확장점
없음. product별 override 폴더가 없는 단일 product 서비스다. 커스터마이징 경계는 Auto\*/non-Auto 2계층 분리뿐이다.

## mock/fake 지점
1. `WorkerWork_order` 등 unify 워커 — DB 호출 차단 핵심 지점
2. `Driver4MybatisIf` — 커스텀 쿼리 selectList/selectOne/insert/update/delete 결과 조작
3. `WorkerRestContentManager.executeRestAPI()` — usermanagement/system 서비스 호출 차단
4. `MessageSendScheduleJob.sendPush()`/`sendEMail()` — FCM HTTP·javaxMail 외부 발송 차단
5. `GoogleCredentials.fromStream()` — FCM credential classpath 의존성 분리

## 변경 위험
- `Worker_Work_mgmt`는 1697줄 단일 클래스에 비즈니스 로직 집중 — 수정 side effect 범위가 큼. 새 로직은 메서드 단위로 격리 추가
- `unify_workmgmt.java` Auto-Generated 주석 영역 수동 수정 시 자동생성 도구 재실행에 덮어씌워질 위험
- FCM service account JSON(`hfr-push-firebase-adminsdk-*`) classpath 누락 시 Push 전체 비활성화
- `processCommonData()`의 Search/Get_Action/HistorySearch 분기는 예외 throw 경로 — 새 requestType의 커스텀 WorkType은 반드시 function else-if 분기 추가(누락 시 EmpException)
- `WorkOrderScheduleJob`/`MessageSendScheduleJob`은 `E_SystemType.NOC`에서만 `isAutoRun()=true` — NOC 외 환경에서 스케줄러 동작 기대 금지
- `update.sql`은 단일 파일 append 방식 — 재실행 안전(idempotent)하지 않은 DDL은 재기동 오류 가능

## 근거 파일
base = `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.Work_mgmt/src/main/java/hfr/plugin/my5g/service/Work_mgmt/`

- `ServiceStart_Work_mgmt.java`, `Work_mgmtServerConfig.java`
- `function/RestInvokerFunction_Work_mgmt.java`
- `worker/Worker_Work_mgmt.java`
- `common/worker/unify_workmgmt/unify_workmgmt.java`
- `common/data/Work_mgmtCD.java`, `common/data/unify_workmgmt/work_order/Work_order.java`
- `schedule/WorkOrderScheduleJob.java`, `schedule/MessageSendScheduleJob.java`, `schedule/RemoteSupportScheduleJob.java`, `schedule/ArGuideResourceScheduleJob.java`, `schedule/UserPushScheduleJob.java`
- `updatesql/update.sql`
