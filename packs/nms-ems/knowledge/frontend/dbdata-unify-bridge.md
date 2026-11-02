# FE dbdata ↔ BE unify_* 자동등록 브릿지

## 목적
FE의 `src/app/dbdata/` 모듈과 BE 아키타입 A(unify_* 자동등록, `packs/nms-ems/knowledge/backend/feature-extension.md` §아키타입 A) 서비스의 대응 관계를 정리한다.
"BE에 unify 엔티티를 추가/수정했을 때 FE에서 무엇을 어디에 추가해야 하는가"를 기존 구조를 바꾸지 않고 답할 수 있게 하는 것이 목표다.

경로 표기:
- `<FRONTEND_ROOT>` = uems_fe 저장소 루트 (`src/app/...`, `framework_modules/...`)
- `<BACKEND_ROOT>` = backend 저장소의 `uNMS_Common/uEMS.microservice/<service>` (backend 문서와 동일 규칙)
- BE 공통 데이터 플러그인 = `uNMS_Common/uEMS.plugin.my5g.Common`

## 1. dbdata 모듈 ↔ BE 서비스 매핑 (실측)
`<FRONTEND_ROOT>/src/app/dbdata/` 하위 디렉터리는 6개다. unify_* 5개가 아키타입 A 브릿지이고, `li_base`는 unify 계열이 아니다.

| FE 모듈 | 대응 BE 서비스 | BE unify 등록 파일 | FE에서 하는 일 |
| --- | --- | --- | --- |
| `dbdata/unify_fault/` | `my5g.Service.Fault` | `.../Fault/common/worker/unify_fault/unify_fault.java` | alarm/event 모델 + `AlarmService` 등 상태 캐시·noti 동기화·count 집계 |
| `dbdata/unify_performance/` | `my5g.Service.Performance` | `.../Performance/common/worker/unify_performance/unify_performance.java` | monitoring/pm-index 모델. 요청은 `content/common/performance/performance.service.ts`(serviceUrl `performance`)가 수행 |
| `dbdata/unify_nebackup/` | `my5g.Service.NeBackupRestore_mgmt` | `.../NeBackupRestore_mgmt/common/worker/unify_nebackup/unify_nebackup.java` | 백업 schedule/log 모델 + `nebackup.service.ts`(serviceUrl `nebackuprestore-mgmt`) |
| `dbdata/unify_workmgmt/` | `my5g.Service.Work_mgmt` | `.../Work_mgmt/common/worker/unify_workmgmt/unify_workmgmt.java` | work-order 계열 모델 + `common/data.service.ts`의 `DataService`(serviceUrl `work-mgmt`, `E_WorkType` Get/GetList/Search/Add/Update/Delete + 다수 커스텀 worktype) |
| `dbdata/unify_system/` | `my5g.Service.System` | `.../System/common/worker/unify_system/unify_system.java` | system-config/user-profile 등 모델 + 엔티티별 `*.service.ts`(serviceUrl `system`; 예외: `db-backup.service.ts`는 `dbbackup-mgmt`도 사용) |
| `dbdata/li_base/` | `License.Portal.Common`(li_base) | (unify 계열 아님) | License Portal용 모델(customer/site/user 등). 이 문서 범위 밖 |

### 엔티티(request type) 대사표 — FE 모델 실측
FE 각 엔티티 폴더의 `<entity>.ts`에 `public static readonly requestType = '<kebab-case>'`가 선언되고, BE `unify_<domain>.java`의 `toDataRequestTypeListAuto()`에 같은 문자열이 등록되어 있어야 왕복이 성립한다. 실측 결과:

- **unify_fault**: alarm, alarm-action, alarm-definition, event, notification-alarm-detail, notification-group, service-status, service-status-detail, service-status-log — 전부 BE 등록 확인. (BE에는 FE 모델이 없는 `alarm-def-action`도 등록되어 있음)
- **unify_performance**: ems-db-update-info, monitoring, monitoring-data, monitoring-item, pm-index — BE 등록 확인. ⚠ `mobibe-cpe-nss-ssid`/`mobibe-cpe-quality-metrics`/`mobibe-cpe-traffic` 3종은 FE 모델과 BE mybatis XML(`uEMS.plugin.my5g.Common/.../mybatis/unify_performance/mybatis_mobibe_*.xml`)만 있고 BE data/worker 클래스·자동등록이 이 소스에서 확인되지 않음 → 사용 전 BE 등록 여부부터 실측할 것.
- **unify_nebackup**: config-backup-file, schedule, schedule-apply, schedule-log, schedule-log-detail, ue-trace, ue-trace-value — 전부 BE 등록 확인.
- **unify_workmgmt**: arguide-resource, arguide-resource-map, message-log, message-template, user-push, video, work-guide, work-order, work-order-action, work-order-alarm, work-order-file, work-order-ne, work-order-user, work-remote-support — 전부 BE 등록 확인. ⚠ FE `push/push.ts`(requestType 'push')는 BE 자동등록에 없음(User_push만 등록).
- **unify_system**: cluster-app-server, db-backup, dnn-mgmt, noc-consign-management, nr-frequency-band, system-config, tac-mgmt, user-profile — BE 등록 확인. `ems-db-update-info`는 BE에서 System/Performance 양쪽에 등록되어 있으나 FE 모델은 `unify_performance/` 폴더에 있다. `system-environment`(requestType 'system-env')는 unify 자동등록이 아니라 별도 request type이다.

## 2. 엔티티 폴더의 파일 세트 규칙 (공통 패턴)
FE 엔티티 폴더는 BE의 IData/Auto_* 패턴을 그대로 미러링한 3~5개 파일로 구성된다. (예: `dbdata/unify_fault/alarm/`)

| FE 파일 | 역할 | BE 대응물 |
| --- | --- | --- |
| `auto-<entity>.ts` | `abstract class Auto_<Entity> extends DBData` — 컬럼 필드 + 복사 생성자. **generator 산출물 형식, 필드만 존재** | `common/data/unify_<domain>/<entity>/Auto_<Entity>.java` |
| `<entity>.ts` | `class <Entity> extends Auto_<Entity>` — `requestType`/`notification`/`notificationSync` 상수, `cast()`(DateTime 필드 재캐스팅), *Auto Generated Code area* 밖에 FE 전용 표시 필드(예: `Alarm.alarm_type_name`)와 `manualCast()` 추가 | `<Entity>.java` (`requestType`, `getNotificationType()`) |
| `<entity>collection.ts` | `class <Entity>Collection extends DBDataCollection<T>` — `castValue()`만 구현 | `<Entity>Collection.java` |
| `enum-<entity>.ts` | 문자열 enum (예: `E_Severity`) | `common/data/unify_<domain>/<entity>/datatype/E_*.java` |
| `<entity>.service.ts` (선택) | Angular `@Injectable` — 요청 전송, noti 수신 동기화, 로컬 캐시/이벤트 | 없음 (FE 전용) |

기반 클래스: `<FRONTEND_ROOT>/framework_modules/framework/dbdata/dbdata.ts` — `DBData`(id, `cast()`는 prototype 교체 + `manualCast()`), `DBDataCollection`(Hashtable<number, T>, `castValue()` abstract).

주의(실측): "Auto Generated Code area Start/End" 주석 블록 안은 수정 금지. FE 전용 필드/로직은 블록 밖에 추가한다(BE와 동일 규칙).

## 3. 통신 규약 (FE 요청 → BE processCommonData)
- FE는 `RequesterObject.sendRequestWait(aServiceUrl, aURequest)` 사용 → `<FRONTEND_ROOT>/framework_modules/framework/core/request.ts`에서 `Url.makeUrl(serviceUrl, version, request_type.toLocaleLowerCase())`로 URL을 만들어 **POST**. 즉 최종 URL은 `<serviceUrl>/v1/<request-type>`.
- BE는 `unify_<domain>.toDataRequestTypeList()`가 request type마다 `new RestContent(aParent, tRequestType, HttpMethodBook.POST, SupportService.<svc>, "v1", ...)`를 자동 생성 → URL이 자동으로 일치한다.
- serviceUrl 상수 위치(실측): fault=`FaultDataRequest.serviceUrl`(`content/common/fault/common/data/faultdatawork.ts`), work-mgmt=`CommonRequest.serviceUrl`(`dbdata/unify_workmgmt/common/data/datawork.ts`), nebackup=`dbdata/unify_nebackup/nebackup.service.ts`, system=`SystemConfigService.serviceUrl`, performance=`content/common/performance/performance.service.ts`.
- workType 문자열(FE enum `E_FaultWorkType`/`E_WorkType`의 Get/GetList/Search/Add/Update/Delete/Acknowledge/History_Search 등)이 BE `CommunicatorCD.WorkTypeBook` 분기와 문자열로 대응한다.

## 4. 대표 실측: unify_fault/alarm ↔ BE Alarm
| 항목 | FE | BE |
| --- | --- | --- |
| requestType | `Alarm.requestType = 'alarm'` (`dbdata/unify_fault/alarm/alarm.ts`) | `Alarm.requestType = "alarm"` (`uEMS.plugin.my5g.Common/.../unify_fault/alarm/Alarm.java`) |
| notification | `Alarm.notification = 'alarm'`, `notificationSync = 'alarm-Sync'` | `Alarm.getNotificationType()` → `"alarm"` |
| 자동등록 | — | `unify_fault.java`: `toDataRequestTypeListAuto()`에 `Alarm.requestType`, `toWorker4Auto()`→`WorkerAlarm.class`, `toDataClass4Auto()`→`Alarm.class`, `getMapperList()`→`mybatis_alarm.xml` |
| 요청 경로 | `AlarmService`가 `FaultDataRequest('alarm', E_FaultWorkType.*, data)`를 serviceUrl `fault`로 POST | `RestInvokerFunction_Fault.processMicroServiceRestRequest()` → `vaildCheckRequest()`가 `unify_fault.toDataClass()`로 테이블 단위 type을 인정 → `processCommonData()` |
| processCommonData 분기 | `E_FaultWorkType.Get`/`GetList`/`Get_Detail`/`History_Search`/`Acknowledge`/`Active_Alarm`… | `WorkTypeBook.Get`→`tDataWorker.query()`, `GetList`→`queryList()`, `Acknowledge`→ack 처리 후 `sendNotifyToClient(..., Alarm.getNotificationType(), tAlarm)` |
| 응답 처리 | `Alarm.cast()`/`AlarmCollection.cast()` — prototype 교체 후 `gen_time` 등 5개 DateTime 필드를 `DateTime.cast()` | JSON 직렬화된 Alarm/Collection |
| noti 동기화 | `AlarmService.processSynchronizeFaultData()`: `notification_type === Alarm.requestType`일 때 `work_type`(Occur/Acknowledge/Clear)별로 로컬 `faultDataList` 갱신 | Worker가 변경 시 notify 전송 |

필드 대응(실측): FE `auto-alarm.ts`의 필드(ems_id, ems_gen_id, ne_id, location, event_code, severity, state, gen_time, gen_type, … ne_severity)는 BE `Auto_Alarm.java`와 이름·순서가 거의 1:1이다. **단 완전 1:1은 아니다** — BE에는 `object_index`, `site_gen_id`가 있으나 FE `auto-alarm.ts`에는 없다. FE 표시 전용 필드(`alarm_type_name`, `is_suppression_state` 등)는 `alarm.ts`의 Auto 영역 밖에 있다.

enum 주의: FE enum은 문자열 값(`E_Severity.CRITICAL = 'CRITICAL'`), BE는 DB에 ordinal 저장(`EnumOrdinalTypeHandler`, `unify_fault.getJavaTypeList()` 등록). JSON 왕복은 이름 문자열이지만 **멤버 추가 시 BE enum 끝에만 추가**(중간 삽입 시 DB ordinal 어긋남)하고 FE enum도 같은 이름으로 맞춘다.

## 5. BE에 새 unify 엔티티를 추가했을 때 FE 체크리스트
BE 쪽 절차는 `packs/nms-ems/knowledge/backend/feature-extension.md` §아키타입 A(4단계)를 따른다. BE 등록이 끝난 후 FE:

1. **폴더 생성**: `<FRONTEND_ROOT>/src/app/dbdata/unify_<domain>/<entity>/` — 도메인은 BE 서비스와 §1 표로 맞춘다.
2. **auto-<entity>.ts**: 같은 도메인의 기존 auto-* 파일을 본떠 `Auto_<Entity> extends DBData` 작성. 필드명은 BE `Auto_<Entity>.java`(snake_case)와 동일하게. `id`는 `DBData`가 제공하므로 선언하지 않는다.
3. **<entity>.ts**: `requestType`을 BE `<Entity>.java`의 `requestType` 문자열과 **정확히 동일하게** 선언. noti를 받는 엔티티면 `notification`/`notificationSync`도 BE `getNotificationType()`과 맞춘다. DateTime 필드가 있으면 `cast()`에서 `DateTime.cast()` 재캐스팅.
4. **<entity>collection.ts**: `DBDataCollection` 상속 + `castValue()`에서 `<Entity>.cast()` 호출 한 줄.
5. **enum-<entity>.ts** (BE datatype enum이 있으면): BE enum 멤버 이름과 동일한 문자열 enum 작성.
6. **요청 연결 — 새 service를 만들지 말 것**: 도메인별 기존 통로를 재사용한다.
   - fault → `FaultDataRequest(requestType, E_FaultWorkType.*, data)` + serviceUrl `fault`
   - work-mgmt → `DataService.saveData()/commonRequestData()` (`CommonRequest`, serviceUrl `work-mgmt`)
   - system → `SystemConfigService.serviceUrl`('system') 패턴의 엔티티별 service
   - nebackup → `nebackup.service.ts`, performance → `content/common/performance/performance.service.ts`
7. **workType 확인**: 범용 CRUD(Get/GetList/Search/Add/Update/Delete)는 BE `processCommonData()`가 자동 라우팅한다. 커스텀 workType이 필요하면 BE `RestInvokerFunction_<Svc>`의 분기와 FE `E_*WorkType` enum에 같은 문자열을 양쪽에 추가한다.
8. **검증**: (a) requestType 문자열 FE/BE 일치, (b) URL = `<serviceUrl>/v1/<request-type>` 도달 여부, (c) GetList 응답이 `Collection.cast()`로 복원되는지, (d) noti 동기화 대상이면 `processSynchronize*` 경로. 실행하지 않은 검증은 Not verified로 보고한다.

흔한 실패:
- requestType 오타/케이스 불일치 → BE `vaildCheckRequest()`에서 거부. FE는 소문자로 URL을 만들므로 requestType은 처음부터 소문자 kebab-case로.
- FE 모델만 만들고 BE `toDataRequestTypeListAuto()` 미등록(또는 반대) → §1의 mobibe-*/push 사례처럼 반쪽 브릿지가 된다.
- DateTime 필드를 `cast()`에 안 넣음 → 화면에서 문자열로 남아 정렬/포맷 오류.

## 6. 상호 참조
- BE 쪽 추가 절차·아키타입 판별: `packs/nms-ems/knowledge/backend/feature-extension.md` (§아키타입 A, 5-seam)
- FE 화면 노출(메뉴/route/provider): `packs/nms-ems/knowledge/frontend/architecture.md`, `new-feature-flow.md`, `provider-and-menu-registration.md`
- 이 문서는 dbdata(모델/전송) 계층만 다룬다. 새 화면까지 필요하면 `new-feature-flow.md` 절차를 병행한다.
