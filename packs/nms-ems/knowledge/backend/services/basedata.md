# BaseData Service

## 서비스 책임
NMS 기준 데이터(NE, Site, EM, Session, Topology, Role, User 등 `unify_base` 스키마 전 테이블)의 CRUD와 동기화를 단일 서비스로 담당하는 source-of-truth 서비스. 다른 서비스가 모두 이 데이터를 참조하므로 변경 영향 범위가 가장 크다.

## 주요 입력
- BaseData REST request (entity CRUD + 복합 워크플로우)
- EMS발 NE/EM/topology sync (`Ems_Sync_*`), notification 기반 sync (`Notification_Sync_*`)
- NOC/OSS/gateway 동기화 요청 (`Request_Oss_data`, `NOC_Request`, `Site_Data_Sync`)
- product EMS 서비스의 discovery/base worker 결과 (REST 경유)

## 주요 출력
- `unify_base` DB row (MariaDB)
- NE/site/session/topology 조회 응답
- `Worker_BaseDataSync`의 변경 notification (다른 서비스로 전파)
- gateway/product EMS로 전달되는 sync request

## 시작점과 실행 설정
근거:
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.BaseData/src/main/java/hfr/plugin/my5g/service/BaseData/ServiceStart_BaseData.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.BaseData/src/main/java/hfr/plugin/my5g/service/BaseData/BaseDataServerConfig.java`

확인 기준:
- `tServiceName = "basedata"`, `runMicroService.run(tServiceName, new DefaultSecurityHttps(), new CommonMybatisDBConfiguration())`
- MariaDB DNS = `platformRatCD.SupportService.maria_db` + `LOCAL_DOMAIN` (MicroService 모드). Mongo 사용 없음.
- `BaseDataServerConfig extends ServerConfigOnlyDB`: `getMapperList()`/`getJavaTypeList()` → `unify_base.getMapperList()`/`getJavaTypeList()` 위임
- `processDBUpgrade()`: `/hfr/plugin/my5g/service/BaseData/updatesql/update.sql` → `role_default_value.sql` 순차 실행 후 `processChangeTimeStamp("unify_base")`, `processAdd_Update_time("unify_base")` 공통 처리
- 별도 진입점 `JobStart_Live_Update_Job.java` — 서비스 기동 없이 DB 업그레이드만 단독 실행하는 job (`worker/Worker_Update_Job.java` 사용). 스키마 변경 검증 시 이 경로 존재를 기억할 것.

## 아키타입
**A(unify_base 자동등록)** — `../feature-extension.md` 참조.
- ② Worker 등록: `unify_base.java`의 *Auto Generated Code* 영역(line 91~733) — `toDataRequestTypeListAuto()`, `toWorker4Auto()`, `toDataClass4Auto()`, `getMapperList()`, `getJavaTypeList()`
- ③ Scheduler: 없음 (scheduler job 미보유)
- ⑤ product 확장: BaseData 자체에는 product 분기 코드가 없다. product EMS 서비스가 REST(`Ems_Sync_Ne` 등)로 결합 (아래 §product 확장점)

## REST contract
근거: `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.BaseData/src/main/java/hfr/plugin/my5g/service/BaseData/function/RestInvokerFunction_BaseData.java` (`getContentList()` line 1958~)

`getContentList()` 명시 등록 (모두 POST):
- 내부 전용(`ScopeTypeBook.Inner`): `data_process`, `Request_Sync_data`
- EMS sync: `Ems_Sync_Ne`, `Ems_Sync_Em`, `Ems_Sync_NeTopology`
- OSS 연동: `Request_Oss_data`, `Update_Oss_data`, `Delete_Oss_data`
- notification sync: `Notification_Sync_Ne`, `Notification_Sync_EM`, `Notification_Sync_NeTopology`
- 복합/관리: `Site_and_Em`, `user-role`(`RequestTypeStringBook.user_role`), `Request_User_TargetRole`, `NOC_Request`, `Ems_db_admin`, `Master_NeSession`, `NeSession_Search`, `MultiBaseData`, `Site_Management`, `Site_Data_Sync`(`JsonDataFormatBook` 등록, NOC 연동)

자동 등록: `getContentList()` 마지막의 `tReturnData.addAll(unify_base.toDataRequestTypeList(this))`가 **36개 엔티티 request type**(아래 표)을 일괄 추가한다. `NMS_registration`도 이 경로로 등록되며 handler 체인(line 1178~)에 Consign/ManualSync 등 전용 워크플로우 분기가 있다.

handler 체인에서만 처리되는 특수 type: `DbTimeOffset`(line 552, DB 시간 오프셋 조회), `EmDbRecovery`(line 921), `RefreshNeTopology`(line 1175, 타 서비스로 `executeRestAPI` 재위임), `topologyModel3DUpload`(REST body가 아닌 `processMicroServiceRestRequestUsingInputStream()` override, line 281 — zip 수신 후 압축 해제).

`Ems_db_admin`은 `isVaildSession()` override에서 `ckloud_Authorization` 토큰 헤더로 세션 없이 통과한다(관리용 백도어 성격 — 수정 시 주의).

## 주요 Worker/Function 흐름
근거:
- `function/RestInvokerFunction_BaseData.java` — `Inner_processMicroServiceRestRequest()`(line 524~)의 else-if 체인이 request type 라우팅의 전부
- `common/worker/unify_base/unify_base.java` — Auto 등록 허브
- `common/worker/unify_base/_manual/AbsWorker_BaseDataSync.java` / `Worker_BaseDataSync.java` — 메모리 DB + 변경 notification 전파
- `worker/Worker_BaseData.java`, `function/NeDataProcessorFunction.java`

핵심 흐름:
- **processCommonData() 자동 CRUD**: 36개 엔티티 request type은 `toWorker4Auto()`/`toDataClass4Auto()` 반환값으로 Get/GetList/Search/Add/Modify/Delete/DeleteEx/DeleteExList/addUnique WorkType이 자동 라우팅된다. 새 분기 코드 불필요.
- **MultiBaseData**: `processMultiBaseDataRequest()`(line 303~)가 `MultiBaseDataList`로 여러 엔티티를 한 요청에 일괄 처리.
- **Site_and_Em**: Site 생성 + EM 생성을 하나의 요청으로 묶는 복합 처리.
- **NMS_registration**: 위탁관리 워크플로우(Consign/ManualSync/Delete/Modify) — Site 생성 + DB commit + gateway 호출이 순차로 묶임(`WorkerNms_registration`, `WorkerSite` 사용).
- **onNotificationDataSync()** (line 2009~): NE/Ne_group Delete 시 `deleteTopologyItemByTarget()`으로 Topology_item을 자동 삭제, Ne_group 삭제 시 `deleteTopologyByGroup()`까지 연동. 기준 데이터 삭제가 topology에 전이되는 지점.
- **Worker_BaseDataSync**: `AbsWorker_BaseDataSync`(WorkflowMap 등록 기준 클래스) 구현체. 메모리 DB 로드 + `processBaseDataNotification()`/`processManualBaseDataNotification()`으로 변경 알림 발송.
- **NeDataProcessorFunction**: `AbsDataProcessorFunction` 구현, `SyncCategoryBook.NE` 기준으로 `Worker4ProcessFunction`에서 자동 선택. 새 SyncCategory가 필요하면 동일 패턴으로 구현체 추가 + `E_FunctionTypeCode.AbsDataProcessorFunction` 등록.

## DB/파일 흐름
- MyBatis + MariaDB, `unify_base` 스키마 전체 소유. Mongo 없음.
- mapper 36개 XML: `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/mybatis/unify_base/mybatis_<entity>.xml` — `unify_base.getMapperList()`(line 690~)에서 `MybatisMapperBook.Dao_<Entity>`로 등록.
- 타입 핸들러: `unify_base.getJavaTypeList()` — EDateTime, TimeSpan, 각종 Enum(E_Type, E_Status 등).
- 파일: `topologyModel3DUpload`로 수신한 3D 모델 zip을 `getTopologyModel3DPath()`에 압축 해제.

### unify_base 자동 등록 엔티티 36종
data class(`common/data/unify_base/<entity>/`) ↔ worker(`common/worker/unify_base/<entity>/Worker<Entity>` + `AutoWorker<Entity>`) ↔ mapper(`mybatis_<entity>.xml`)가 동일 이름 규약으로 1:1:1 대응:

`Cluster`, `Em`, `Ems_db_update_info`, `Ems_image`, `Ems_registration`, `Equipment_image`, `Integration_info`, `Nb_bs`, `Nb_bs_cell`, `Nb_ru`, `Nb_ru_cell`, `Ne`, `Network_line`, `Ne_gallery`, `Ne_group`, `Ne_model`, `Ne_run_config`, `Ne_session_netconf`, `Ne_session_rest`, `Ne_session_serial`, `Ne_session_snmp`, `Ne_session_ssh`, `Ne_slot_info`, `Ne_slot_port_info`, `Ne_slot_ch_info`, `Ne_topology`, `Nms_registration`, `Privilege_role`, `Role`, `Site`, `Software_image`, `Target_role`, `Topology`, `Topology_3d_model`, `Topology_item`, `User`

## 기능 확장점 (feature seams)
| seam 유형 | 추가 방법 | 등록 위치/근거 파일 |
| --- | --- | --- |
| ① rest-request-type | `getContentList()`에 `new RestContent(this, 새_type, ...)` 한 줄 + `Inner_processMicroServiceRestRequest()` else-if 체인 끝에 분기 추가. 단순 CRUD면 이 seam 대신 아래 ② 자동등록만으로 충분 | `function/RestInvokerFunction_BaseData.java` (getContentList line 1958, 체인 line 524) |
| ② worker-function (아키타입 A) | 새 엔티티 4단계: 1) `common/data/unify_base/foo/Foo.java`(IData) 2) `common/worker/unify_base/foo/WorkerFoo.java`(AbsBaseDataWorker) 3) `unify_base.java` Auto Generated Code 영역 5곳(`toDataRequestTypeListAuto`/`toWorker4Auto`/`toDataClass4Auto`/`getMapperList`/`getJavaTypeList`) 등록 4) mapper XML 추가 → `processCommonData()` CRUD + notification 자동 동작 | `common/worker/unify_base/unify_base.java` (line 91~733) |
| ④ db-mapper | `uEMS.plugin.my5g.Common`의 `hfr/plugin/my5g/common/mybatis/unify_base/mybatis_foo.xml` 추가 + `getMapperList()`에 `MybatisMapperBook.Dao_Foo` 등록 + `updatesql/update.sql`에 CREATE/ALTER 추가(`processDBUpgrade` 자동 실행) | `unify_base.java` (getMapperList line 690), `BaseDataServerConfig.java` |
| data-model (복합 워크플로우) | `NMS_registration` 처리 블록(line 1178~) 패턴 참고: 새 request type 분기에서 내부적으로 `processCommonData()` 위임 + `Worker_BaseDataSync.processBaseDataNotification()`/`processManualBaseDataNotification()`으로 변경 알림 발송 | `function/RestInvokerFunction_BaseData.java` (line 1178~1421) |
| sync-strategy | 동기화 전략/메모리 DB 로드 대상 변경은 `Worker_BaseDataSync` 수정 또는 `AbsWorker_BaseDataSync` 구현 교체(WorkflowMap 등록 기준 클래스 = `AbsWorker_BaseDataSync.class`) | `common/worker/unify_base/_manual/AbsWorker_BaseDataSync.java` |
| data-processor | 새 SyncCategory 처리기: `AbsDataProcessorFunction` 구현체 추가 + `E_FunctionTypeCode.AbsDataProcessorFunction` 등록 → `Worker4ProcessFunction`이 category 기준 자동 선택 | `function/NeDataProcessorFunction.java` |
| ⑤ product-override | BaseData 코드 무수정. product EMS 서비스에 `Worker4baseNe<Product>` 클래스 추가 — `WorkerLocalBaseDataCommon`(WorkflowMap의 `AbsWorkerLocalBaseData`) 경유로 NE 생성/갱신하고 BaseData REST(`Ems_Sync_Ne`/`Notification_Sync_Ne`)로 전파 | 아래 §product 확장점 파일들 |

## product 확장점
BaseData 서비스 내부에는 product 분기 코드가 없다. product EMS 서비스가 discovery 결과를 `Worker4baseNe*`에서 처리해 BaseData REST API로 반영한다. 신규 product는 해당 product 서비스에만 `Worker4baseNe<NewProduct>`를 추가한다.
- vRAN: `<BACKEND_ROOT>/product/my5G/my5g.Service.vran_ems/src/main/java/hfr/plugin/my5g/service/vran_ems/hiems/network/ne/Worker4baseNemy5G.java`
- wifi: `<BACKEND_ROOT>/product/my5G/my5g.Service.wifi/src/main/java/hfr/plugin/my5g/service/wifi/network/ne/Worker4baseNe_wifi.java`
- openHaul: `<BACKEND_ROOT>/product/openhaul/my5g.Service.openHaul/src/main/java/hfr/plugin/my5g/service/openHaul_ems/hiems/network/ne/Worker4baseNeopenHaul.java`
- sixgen: `<BACKEND_ROOT>/product/sixgen-ems/my5g.Service.six_gen_flexible/src/main/java/hfr/plugin/my5g/service/six_gen_flexible/hiems/network/ne/Worker4baseNeSixGen.java`

## 죽은/미구현 코드
- `getContentList()` line 1977: `Ems_Ne_Topology` 등록 주석 처리 — 비활성 request type, 기능 추가 대상으로 삼지 말 것.
- `DbTimeOffset` 분기 내 JSON accumulate 라인(line 559) 주석 — 실제 응답은 `tResponse.setData(...)` 경로만 활성.

## mock/fake 지점
real DB write / real gateway 호출은 실행하지 않는다. 우선 mock:
1. `WorkerNe.processEms_Sync_NeList()` / `processEms_Sync_Ne()` — EMS NE 동기화 입력
2. `Worker_BaseDataSync.processBaseDataNotification()` / `processManualBaseDataNotification()` — 동기화 알림 전파
3. `WorkerRestContentManager.executeRestAPI()` — gateway_uems/product EMS 외부 호출
4. `WorkerLocalRestSessionManager.getEms_ID()` — EMS 세션 ID 추출
5. `WorkerNms_registration.query/update/queryList`, `WorkerSite.create/deleteSites` — 위탁관리/Site DB
6. `unify_base.getMapperList()` 대상 MyBatis XML — DB 쿼리 단위
7. `WorkerCommonMybatis.getDbTimeOffset()` — DB 시간 오프셋

## fixture 기준
- NE/site/EM/session 초기 DB 상태, topology parent-child 관계
- EMS sync payload(`Ems_Sync_*`) / notification payload(`Notification_Sync_*`)
- MultiBaseData 일괄 요청 payload, NMS_registration WorkType별 요청
- 다른 service(gateway/product EMS) mock 응답
- write 이후 expected `unify_base` row + expected notification 발송 내역

## 변경 위험
- NE/Site source of truth 손상 시 전체 서비스 조회/동기화 연쇄 실패
- `unify_base` Auto Generated Code 영역 수정 오류 시 36개 엔티티 CRUD 라우팅 전체 중단 (영역 주석: "Do not modify this area" — 등록 추가만, 구조 변경 금지)
- `Worker_BaseDataSync` 잘못 수정 시 notification 누락/중복 → 타 서비스 캐시 stale
- NMS_registration 워크플로우는 Site 생성 + DB commit + gateway 호출이 순차 묶임 — 중간 실패 시 사이트 고아 데이터
- `processDBUpgrade`의 update.sql 변경이 기존 product DB 스키마에 side effect
- `onNotificationDataSync()`의 NE/Ne_group 삭제 → Topology_item 자동 삭제가 의도치 않게 동작 가능
- product `Worker4baseNe*` 추가 시 BaseData REST 호출 payload 계약(`Ems_Sync_Ne` 형식) 불일치면 discovery 결과가 반영되지 않음

## 근거 파일
서비스 루트 = `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/my5g.Service.BaseData/src/main/java/hfr/plugin/my5g/service/BaseData/`
- `ServiceStart_BaseData.java`, `BaseDataServerConfig.java`, `JobStart_Live_Update_Job.java`
- `function/RestInvokerFunction_BaseData.java`, `function/NeDataProcessorFunction.java`
- `common/worker/unify_base/unify_base.java`
- `common/worker/unify_base/_manual/AbsWorker_BaseDataSync.java`, `_manual/Worker_BaseDataSync.java`
- `common/worker/unify_base/<entity>/Worker<Entity>.java` (36종), `worker/Worker_BaseData.java`, `worker/Worker_Update_Job.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/mybatis/unify_base/` (mapper XML 36개)
- product: §product 확장점의 `Worker4baseNe*` 4개 파일
