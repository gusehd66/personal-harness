# NMS FE basedata 모듈 그룹 맵

## 목적
`<FRONTEND_ROOT>/src/app/basedata/` 아래의 기준데이터(BaseData) 모듈 40개를 도메인별로 정리하고, 표준 모듈 구조와 신규 entity 추가 절차를 제공한다.
이 문서를 따르면 기존 구조를 바꾸지 않고 필요한 모듈만 정확히 추가/수정할 수 있다.

경로 표기: `<FRONTEND_ROOT>` = uems_fe 저장소 루트. BE 경로는 `packs/nms-ems/knowledge/backend/services/basedata.md`의 `<BACKEND_ROOT>` 표기를 따른다.

## BE 대응 관계 (상호 참조)
- BE 서비스: **BaseData 서비스** (`packs/nms-ems/knowledge/backend/services/basedata.md`) — `unify_base` 스키마 36개 entity를 자동 등록(아키타입 A)으로 CRUD 처리.
- BE 확장 절차 공통 문서: `packs/nms-ems/knowledge/backend/feature-extension.md`.
- FE의 각 entity 클래스가 갖는 `requestType` 문자열이 BE `unify_base.toDataRequestTypeListAuto()`가 등록하는 request type과 1:1로 짝을 이룬다. 모든 요청은 REST POST로 microservice URL `basedata`에 전달된다.
  - 근거: `<FRONTEND_ROOT>/framework_modules/framework/basedata/basedatawork.ts` — `BaseDataRequest.serviceUrl = "basedata"` (line 54)
  - 근거: `<FRONTEND_ROOT>/src/app/basedata/ne/ne.ts` — `public static readonly requestType = 'ne'`

## 3계층 구조
| 계층 | 위치 | 역할 |
| --- | --- | --- |
| framework 공통 기반 | `<FRONTEND_ROOT>/framework_modules/framework/basedata/` | `basedata.ts`(BaseData/BaseDataCollection), `basedataprocess.ts`(service 기반 클래스 + 앱 시작 로더), `basedatawork.ts`(BaseDataRequest/Response, E_BaseDataWorkType), `basedataedit.ts`(BaseDataEditor/BaseDataEditHelper — write path), `basedatatable.ts`(BaseDataTable/BaseDataTreeTable — table 화면 기반) |
| entity 모듈 | `<FRONTEND_ROOT>/src/app/basedata/<entity>/` | entity별 model/collection/(선택)enum/(선택)service. **화면(component)은 없다** — 화면은 content 계층에 있다 |
| 소비처 | `<FRONTEND_ROOT>/src/app/content/`, `app-customer/`, `app-mobile-cpe/` 등 | 화면 component가 basedata service를 주입받아 목록/캐시를 사용 (예: `NeService`는 166개 파일에서 import) |

tsconfig path alias: `framework/*` → `../framework_modules/framework/*`, `app/*` → `./app/*` (`<FRONTEND_ROOT>/tsconfig.json`).

## 모듈 구성 규칙 (파일 5종)
`<FRONTEND_ROOT>/src/app/basedata/<entity>/` 안의 파일은 다음 규약을 따른다.

| 파일 | 필수 | 내용 |
| --- | --- | --- |
| `auto-<entity>.ts` | 필수 | `Auto_<Entity> extends BaseData` — DB 컬럼과 1:1 필드 + 복사 생성자. 코드 생성 산출물 성격이며 필드 추가 외 로직을 넣지 않는다 |
| `<entity>.ts` | 필수 | `<Entity> extends Auto_<Entity>` — `requestType`/`notification`/`notificationSync` 상수, `cast()`, `clone()`이 "Auto Generated Code area" 주석 블록 안에 있고(**Do not modify this area**), 블록 밖에 수동 헬퍼(예: `Ne.getProperty()`)를 추가한다 |
| `<entity>collection.ts` | 필수 | `<Entity>Collection extends BaseDataCollection<Entity>` — `castValue()`만 override |
| `enum-<entity>.ts` | 선택 | entity 필드용 enum (예: `ne/enum-ne.ts`의 `E_Type`, `E_Init_status`) |
| `<entity>.service.ts` | 선택 | `@Injectable({ providedIn: 'root' })` service. **조회/캐시 소비처가 있는 entity에만 존재** (40개 중 20개) |

## service 기반 클래스 3종
근거: `<FRONTEND_ROOT>/framework_modules/framework/basedata/basedataprocess.ts`

| 기반 클래스 | 의미 | 대표 |
| --- | --- | --- |
| `BaseDataServiceSync<T, C>` | 로컬 캐시 유지 + notification으로 실시간 동기화. `getList()`/`get(id)`가 동기(sync) 반환 | `NeService`, `SiteService`, `EmService`, `TopologyService`, `UserService` |
| `BaseDataService<T, C>` | 필요 시 `loadBaseData()` 후 반환. `getList()`/`get(id)`가 async | `ClusterService`, `NeGalleryService`, `RoleService`, `UeTraceService`, `NmsRegistrationService` |
| `BaseDataServiceDetail<T, C>` | `BaseDataService`와 동일 계열(Full/Local) — async 조회 | `NeTopologyService`, `SoftwareImageService`, `Topology3DModelService` |

service가 반드시 구현하는 abstract 멤버: `baseDataType`(= `<Entity>.requestType`), `notificationType`, `notificationSyncType`, `castBaseDataList()`, `castBaseData()`, `createCollection()`.

## 실측 모듈 그룹 맵 (40개 디렉터리)
`ls <FRONTEND_ROOT>/src/app/basedata/` 실측 기준. "주요 service"가 비어 있으면 model/collection만 있는 모듈이다.

### NE(장비) 코어
| 모듈 | 담당 entity(requestType) | 주요 파일 |
| --- | --- | --- |
| `ne` | `Ne` (`ne`) | `ne.service.ts` (`NeService`, Sync, 앱 시작 로더) |
| `ne-group` | `Ne_group` | `ne-group.service.ts` (`NeGroupService`, Sync, 앱 시작 로더) |
| `ne-model` | `Ne_model` | `ne-model.service.ts` (`NeModelService`, Sync) |
| `ne-gallery` | `Ne_gallery` | `ne-gallery.service.ts` (`NeGalleryService`) |
| `ne-run-config` | `Ne_run_config` | `ne-run-config.service.ts` (`NeRunConfigService`, Sync) |
| `ne-running-status` | `Ne_running_status` (`ne-running-status`) | `ne-running-status.service.ts` (Sync) |
| `ne-status-change-log` | `Ne_status_change_log` | (service 없음) |
| `ne-ems-send-log` | `Ne_ems_send_log` (`ne-ems-send-log`) | (service 없음) |
| `em` | `Em` | `em.service.ts` (`EmService`, Sync, 앱 시작 로더) |
| `cluster` | `Cluster` | `cluster.service.ts` (`ClusterService`) |

### NE 세션 (프로토콜별 접속 정보)
| 모듈 | 담당 entity | 주요 파일 |
| --- | --- | --- |
| `ne-session-ssh` | `Ne_session_ssh` | `ne-session-ssh.service.ts` (`NeSessionSSHService`) — terminal/virtual-terminal 화면이 소비 |
| `ne-session-netconf` | `Ne_session_netconf` | (service 없음) |
| `ne-session-rest` | `Ne_session_rest` | (service 없음) |
| `ne-session-serial` | `Ne_session_serial` | (service 없음) |
| `ne-session-snmp` | `Ne_session_snmp` | (service 없음) |

### 토폴로지
| 모듈 | 담당 entity | 주요 파일 |
| --- | --- | --- |
| `topology` | `Topology` | `topology.service.ts` (`TopologyService`, Sync, 앱 시작 로더) + `topology.service.spec.ts` |
| `topology-item` | `Topology_item` | `topology-item.service.ts` (`TopologyItemService`, Sync) + spec |
| `topology-3d-model` | `Topology_3d_model` | `topology-3d-model.service.ts` (`Topology3DModelService`) |
| `ne-topology` | `Ne_topology` | `ne-topology.service.ts` (`NeTopologyService`) — `ems-ne-topology.service.ts`는 클래스 선언이 주석 처리된 비활성 파일 |

### 사이트/등록/연동
| 모듈 | 담당 entity | 주요 파일 |
| --- | --- | --- |
| `site` | `Site` | `site.service.ts` (`SiteService`, Sync, 앱 시작 로더) |
| `nms-registration` | `Nms_registration` (`nms-registration`) | `nms-registration.service.ts` (`NmsRegistrationService`) — NOC 위탁관리 전용 request(`NocConsignManagementRequest`) 포함, `nms-registration-work.ts` |
| `ems-registration` | `Ems_registration` | (service 없음) |
| `ems-db-update-info` | `Ems_db_update_info` | (service 없음) |

### 사용자/권한
| 모듈 | 담당 entity | 주요 파일 |
| --- | --- | --- |
| `user` | `User` (`user`) | `user.service.ts` (`UserService`, Sync) |
| `role` | `Role` | `role.service.ts` (`RoleService`) |
| `privilege-role` | `Privilege_role` | (service 없음) |
| `target-role` | `Target_role` | `target-role.service.ts` (`TargetRoleService`, Sync) — `enum-target-role.ts`의 `E_Type`(group/ne)은 권한 검사에서 광범위하게 사용 |
| `user-group` | `User_group` (`user-group`) | (service 없음) |
| `user-group-deny-operation` | `User_group_deny_operation` | (service 없음) |

### 이미지/소프트웨어/FOTA
| 모듈 | 담당 entity | 주요 파일 |
| --- | --- | --- |
| `software-image` | `Software_image` | `software-image.service.ts` (`SoftwareImageService`) |
| `ems-image` | `Ems_image` | (service 없음) |
| `equipment-image` | `Equipment_image` (`equipment-image`) | (service 없음) |
| `fota-auto` | `Fota_auto` (`fota-auto`) | (service 없음) |
| `fota-auto-current-version` | `Fota_auto_current_version` | (service 없음) |

### NB-IoT 계열
| 모듈 | 담당 entity | 주요 파일 |
| --- | --- | --- |
| `nb-bs` | `Nb_bs` | (service 없음) |
| `nb-bs-cell` | `Nb_bs_cell` | (service 없음) |
| `nb-ru` | `Nb_ru` | (service 없음) |
| `nb-ru-cell` | `Nb_ru_cell` | (service 없음) |

### 기타
| 모듈 | 담당 entity | 주요 파일 |
| --- | --- | --- |
| `ue-trace` | `Ue_trace` (`ue-trace`) | `ue-trace.service.ts` (`UeTraceService`), `ue-trace-list.ts` |
| `backup/user` | (구버전 `User` 사본) | `user.service.ts`, `user.ts` — **백업 폴더. 수정/참조 금지, 신규 코드에서 import하지 말 것** |

### FE ↔ BE entity 대응 주의
- BE `unify_base` 자동등록 36종 중 FE에 대응 디렉터리가 **없는** entity: `Integration_info`, `Network_line`, `Ne_slot_info`, `Ne_slot_port_info`, `Ne_slot_ch_info` (FE 화면이 필요해지면 아래 추가 절차로 생성).
- FE에는 있으나 BE 문서의 36종 목록에 **없는** 모듈: `fota-auto`, `fota-auto-current-version`, `ne-ems-send-log`, `ne-running-status`, `ne-status-change-log`, `ue-trace`, `user-group`, `user-group-deny-operation`. 이들의 `requestType`도 동일 `basedata` URL로 전송되므로, 수정 전 BE 쪽 handler 존재 여부를 `RestInvokerFunction_BaseData.getContentList()` 기준으로 재확인할 것.
- service도 외부 소비처도 확인되지 않는 모듈이 다수 있다(확인 예: `ne-session-netconf`, `nb-bs`는 자기 폴더 밖 import 없음). BE entity와의 짝 유지 목적의 생성물이므로 삭제하지 말고, 기능 추가 대상도 아니다.

## 대표 모듈 상세

### 1) `ne` — 수동 확장이 많은 Sync 모듈
근거 파일: `<FRONTEND_ROOT>/src/app/basedata/ne/` (`auto-ne.ts`, `ne.ts`, `necollection.ts`, `enum-ne.ts`, `ne.service.ts`)

- `auto-ne.ts`: `Auto_Ne extends BaseData` — `site_id`, `em_id`, `ne_group_id`, `model`, `ip_address`, `properties` 등 DB 컬럼 필드 + 복사 생성자만.
- `ne.ts`: Auto 영역에 `requestType='ne'`, `notification='ne'`, `notificationSync='ne-Sync'`, `cast()`(DateTime 필드 재캐스팅). 영역 밖 수동 확장: `alarm_severity`, `getProperty()`/`getIntProperty()`(JSON `properties` 파싱).
- `necollection.ts`: `castValue()` override 한 개뿐 — 표준형.
- `ne.service.ts`(`NeService extends BaseDataServiceSync<Ne, NeCollection>`):
  - abstract 구현: `baseDataType`/`notificationType`/`notificationSyncType`/`castBaseDataList`/`castBaseData`/`createCollection`.
  - 도메인 조회 헬퍼: `getListBySite()`, `getByGroupID()`, `getAllNeByGroup()`, `getChildNeList()`.
  - 권한 필터: `getAuthorizedList()`/`getAuthorizedWriteList()` — `AuthorityCheck` + `AuthService.checkAuthority()`로 group→ne 순서 검사.
  - sync hook override: `synchronizeAdd/Modify/Delete()` — 그룹 권한에 따라 로컬 캐시 add/remove 분기, `ContentsManagerService` badge count 갱신.
  - 타 service 연동: `NeGroupService.baseDataChanged` 구독(`onGroupNotification`) — 그룹 Add 시 `getListFromRemote(RestParameterData)`로 재조회.
  - write 예시: `updateNeName()` — `new BaseDataRequest(this.baseDataType, E_BaseDataWorkType.Modify)` + `request.data = ne` + `sendRequestWait(this.getServiceUrl(), tRequest)`.
  - product 필터: `checkBaseDataUse()`가 `ProductService.isSupportModel(model)`로 미지원 모델 notification을 무시.

### 2) `site` — 최소 표준형 Sync 모듈
근거 파일: `<FRONTEND_ROOT>/src/app/basedata/site/site.service.ts`

`SiteService extends BaseDataServiceSync<Site, SiteCollection>`는 abstract 구현 6개 + 도메인 헬퍼 2개(`getByGroupID()`, `getIDByGroupID()`)가 전부다. **새 service를 만들 때 이 파일을 템플릿으로 사용하라.**

## API 호출 패턴 (공통)
근거: `basedataprocess.ts`, `basedatawork.ts`

1. **URL**: 모든 기준데이터 요청은 `getServiceUrl()` = `BaseDataRequest.serviceUrl` = `"basedata"` (microservice 이름). service별 override는 예외적(예: `NmsRegistrationService.getServiceUrl()`도 결국 `'basedata'` 상수 반환).
2. **요청**: `new BaseDataRequest(requestType, E_BaseDataWorkType, data?)` → `sendRequestWait(url, request, timeout)`. `E_BaseDataWorkType` = `Add | Modify | Delete | Get | GetList` (FE enum 기준. BE는 Search/DeleteEx 등 추가 WorkType을 처리하나 FE enum에는 없음).
3. **초기 로드**: `loadBaseData()`가 `GetList` 요청 → `castBaseDataList()`로 collection 캐스팅 → `baseDataList` 캐시.
4. **실시간 동기화**: `notificationType`과 일치하는 `UNotification` 수신 시 `work_type`에 따라 `synchronizeAdd/Modify/Delete()` → 로컬 캐시 갱신 후 `notificationSyncType`(`<type>-Sync`)으로 내부 재전파. 화면은 service의 `baseDataChanged`를 구독한다.
5. **파라미터 조회**: `getListFromRemote(RestParameterData)` — `parameter.add('ne_group_id', id)` 형태.

## 앱 시작 시 로딩 등록
근거: `<FRONTEND_ROOT>/src/app/app.prepare.ts` `registerBaseDataLoader()`

- 전역 필수 로더 5종: `TopologyService`, `NeService`, `NeGroupService`, `EmService`, `SiteService` (`BaseDataLoadProcess.addBaseDataLoader(...)`). `UserService`/`EmsNeTopologyService`는 주석 처리되어 있음 — 임의 복구 금지.
- content별 추가 로더: `<FRONTEND_ROOT>/src/app/common/content/content-define/content-manager.service.ts`의 `getBaseDataLoaderList()`를 content 정의가 override하면 `ContentServicePrepareProcess`에서 자동 등록된다.
- 로딩 순서: `AppPrepareModule.registerStartSequence()` — Communication → Login → `BaseDataLoadProcess`(등록된 로더 순차 `loadBaseData()`) → ContentServicePrepare.
- 중복 등록 시 `BaseDataLoadProcess`가 error 로그를 남기고 무시한다.

## 화면(write path) 공통 기반
- 편집 dialog/panel: `basedataedit.ts`의 `BaseDataEditor` / `BaseDataManagementPanel` / `BaseDataEditHelper` (Add/Modify/Delete 요청 전송 담당).
- 목록 화면: `basedatatable.ts`의 `BaseDataTable` / `BaseDataTreeTable` (notification 수신해 행 갱신). 사용 예: `app-customer/.../user-mgmt-table.component.ts` 등.
- 신규 table/form 화면은 이 기반 클래스 + `packs/nms-ems/knowledge/components/wc-data-display.md`/`wc-form-controls.md` 패턴을 먼저 확인한다.

## 신규 기준데이터 entity 추가 절차 (FE ↔ BE 짝 맞추기)
전제: BE에 `unify_base` entity가 먼저(또는 동시에) 추가된다. BE 쪽 4단계는 `packs/nms-ems/knowledge/backend/services/basedata.md` §기능 확장점 ②(아키타입 A)와 `packs/nms-ems/knowledge/backend/feature-extension.md`를 따른다.

BE 4단계 요약: 1) `common/data/unify_base/foo/Foo.java` 2) `common/worker/unify_base/foo/WorkerFoo.java` 3) `unify_base.java` Auto Generated Code 영역 5곳 등록 4) mapper XML + `update.sql`.

FE 체크리스트 (기존 파일 수정은 최소화, 신규 파일 위주):
1. [ ] BE data class의 request type 문자열을 확인한다 (BE `toDataRequestTypeListAuto()` 등록값). FE `requestType`은 이 문자열과 **정확히 일치**해야 한다.
2. [ ] `<FRONTEND_ROOT>/src/app/basedata/foo/` 디렉터리 생성 (kebab-case, BE entity `Foo_bar` → `foo-bar`).
3. [ ] `auto-foo.ts`: `Auto_Foo extends BaseData` (`framework/basedata/basedata` import) — DB 컬럼 필드 + 복사 생성자. `id/name/description/create_time/update_time`은 `BaseData`가 이미 가지므로 중복 선언하지 않는다.
4. [ ] `foo.ts`: `Foo extends Auto_Foo` — Auto Generated Code area 주석 블록 안에 `requestType`/`notification`/`notificationSync`/`cast()`/`clone()` 작성 (기존 모듈에서 블록 통째로 복사 후 치환). `DateTime`/enum 필드가 있으면 `cast()`에서 재캐스팅.
5. [ ] enum 필드가 있으면 `enum-foo.ts` 작성.
6. [ ] `foocollection.ts`: `FooCollection extends BaseDataCollection<Foo>` — `castValue()`만 override (`necollection.ts` 형태 그대로).
7. [ ] 화면에서 조회가 필요할 때만 `foo.service.ts` 작성:
   - 로그인 후 상시 캐시 + 실시간 동기화 필요 → `BaseDataServiceSync` (템플릿: `site.service.ts`)
   - 화면 진입 시 조회로 충분 → `BaseDataService` (템플릿: `cluster.service.ts`, `ne-gallery.service.ts`)
8. [ ] 앱 시작부터 캐시가 필요하면 로더 등록: 전역이면 `app.prepare.ts` `registerBaseDataLoader()`에 한 줄 추가, 특정 content 전용이면 해당 content 정의의 `getBaseDataLoaderList()`에 추가. **불필요한 전역 로더 추가는 로그인 후 로딩 시간을 늘리므로 금지.**
9. [ ] 화면 작업은 basedata가 아니라 content 계층에서 한다 — `packs/nms-ems/knowledge/frontend/new-feature-flow.md` 절차(메뉴/route/provider/권한)를 따르고, table/form은 `BaseDataTable`/`BaseDataEditor` 기반을 우선 검토.
10. [ ] 검증: BE 기동 상태에서 `GetList` 응답 필드가 `auto-foo.ts` 필드와 일치하는지, Add/Modify/Delete 후 notification으로 캐시가 갱신되는지 확인. 실행 못 한 검증은 `Not verified`로 보고.

## 금지/주의
- `<entity>.ts`의 "Auto Generated Code area Start/End" 블록 내부 구조를 변경하지 않는다 (상수 추가 수준만 허용). 수동 로직은 블록 밖에 둔다.
- `basedata/backup/` 폴더의 파일을 import하거나 수정하지 않는다.
- FE `requestType` 문자열을 임의로 바꾸지 않는다 — BE 자동등록 request type과의 계약이다. 변경은 BE와 동시 변경 + `packs/nms-ems/knowledge/backend/services/basedata.md` §변경 위험 검토가 필요하다.
- service 없는 모듈에 "일단 service부터" 만들지 않는다 — 소비처(화면)가 생길 때 추가한다.
- `app.prepare.ts`의 주석 처리된 로더(`UserService`, `EmsNeTopologyService`)를 임의로 활성화하지 않는다.
- basedata 모듈은 site/product fileReplacement 대상이 아니지만, 소비하는 화면은 대상일 수 있다 — `packs/nms-ems/knowledge/frontend/architecture.md`의 active configuration 확인 절차를 먼저 수행한다.

## 근거 파일
- `<FRONTEND_ROOT>/src/app/basedata/` 하위 40개 디렉터리 (실측: `ne`, `site`, `user`, `cluster` 등 — 본문 표)
- `<FRONTEND_ROOT>/framework_modules/framework/basedata/basedata.ts`, `basedataprocess.ts`, `basedatawork.ts`, `basedataedit.ts`, `basedatatable.ts`
- `<FRONTEND_ROOT>/src/app/app.prepare.ts`, `<FRONTEND_ROOT>/src/app/common/content/content-define/content-manager.service.ts`
- `<FRONTEND_ROOT>/tsconfig.json` (path alias)
- BE 대응: `packs/nms-ems/knowledge/backend/services/basedata.md`, `packs/nms-ems/knowledge/backend/feature-extension.md`
