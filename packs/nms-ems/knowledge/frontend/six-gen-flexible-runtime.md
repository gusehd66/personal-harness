# six-gen-flexible FE Runtime Flow

> ⚠ checkout 주의(2026-07 실측): `six-gen-flexible` site는 `unms_develop_7.0` branch checkout의 `angular.json`/`sites/`에 존재하지 않는다 — 별도 branch 전용이다. 현재 checkout의 site 목록/교체 체계는 `site-specific-build.md`를 본다. 이 문서는 six-gen-flexible site가 포함된 branch에서만 그대로 적용된다.

## 목적
`<FRONTEND_ROOT>` checkout에서 `six-gen-flexible` 작업을 할 때 실제 런타임 파일과 메뉴/provider 흐름을 같은 기준으로 추적하도록 한다.

## 핵심 결론
`six-gen-flexible` 작업에서는 generic 파일만 보면 안 된다.

먼저 다음 순서로 확인한다.

1. `package.json` script
2. `angular.json` build/serve configuration
3. `sites/six-gen-flexible/app.prepare.ts`
4. `sites/six-gen-flexible/contents.module.ts`
5. content provider module
6. `ContentsManagerService`
7. authority/filtering

## 실행/빌드 경로
현재 checkout 기준:

- `package.json`의 `serve:six-gen-flexible`은 `ng serve -c six-gen-flexible --serve-path unms/`를 사용한다.
- `angular.json` serve configuration의 `six-gen-flexible`은 build target으로 `uems-fe:build:six-gen-flexible-dev`를 사용한다.
- `six-gen-flexible-dev`는 `baseHref: /unms/`와 site-specific fileReplacement를 사용한다.
- production 성격의 `six-gen-flexible` build configuration은 `baseHref: /`를 사용하므로 dev serve와 혼동하지 않는다.

## 주요 fileReplacement
`angular.json`의 `six-gen-flexible` / `six-gen-flexible-dev` configuration은 다음 파일을 교체한다.

| replace | with |
| --- | --- |
| `src/app/app.prepare.ts` | `sites/six-gen-flexible/app.prepare.ts` |
| `src/app/content/contents/contents.module.ts` | `sites/six-gen-flexible/contents.module.ts` |
| `src/app/common/common/product.ts` | `sites/six-gen-flexible/product.ts` |
| `src/tsconfig.app.json` | `sites/six-gen-flexible/tsconfig.app.json` |
| `src/app/app.routes.ts` | `sites/six-gen-flexible/app.routes.ts` |
| index | `sites/six-gen-flexible/index.html` |
| style | `src/assets/scss/style-six-gen-flexible.scss` |

## App prepare 흐름
`sites/six-gen-flexible/app.prepare.ts`에서 start sequence와 제외 메뉴를 확인한다.

중요 지점:
- `registerExcludeMenu(...)`에서 performance menu 일부를 제외한다.
- `registerStartSequence(...)`에서 communication, login, base data load, `ContentServicePrepareProcess`를 등록한다.
- `registerExcludeAuthority(...)`는 authority 제외/필터 영향이 있을 수 있으므로 메뉴 미노출 조사 시 확인한다.
- React 인증 startup과 `session_id` wire parity는 `packs/nms-ems/knowledge/frontend/six-gen-react-auth-authority-parity.md`를 함께 읽는다.
- React 로그인 license 조회, WebSocket/keepalive 선행 순서, `UResponse` 직렬화 parity는 `packs/nms-ems/knowledge/frontend/six-gen-react-auth-license-bootstrap-parity.md`를 함께 읽는다.
- React Preference 메뉴, mutation API, MessageBox, desktop geometry parity는 `packs/nms-ems/knowledge/frontend/six-gen-react-preference-parity.md`를 함께 읽는다.
- React Preference System Info의 제품 정보, 빈 `wc-card` title 영역, font/geometry parity는 `packs/nms-ems/knowledge/frontend/six-gen-react-preference-system-info-parity.md`를 함께 읽는다.
- React Alarm Definition의 full-row Update, row selection, fixed Action 합성 규칙은 `packs/nms-ems/knowledge/frontend/six-gen-react-preference-alarm-definition-parity.md`를 함께 읽는다.
- React Log Level의 optional `id`, startup site cache label, Set payload, MessageBox parity는 `packs/nms-ems/knowledge/frontend/six-gen-react-preference-log-level-parity.md`를 함께 읽는다.
- React Server Log Download의 Blob 요청, 600초 timeout, Cancel 후 응답 무시, WcLoading, tooltip parity는 `packs/nms-ems/knowledge/frontend/six-gen-react-preference-server-log-download-parity.md`를 함께 읽는다.
- React Security User의 조회/변경 service 분리, full model 직렬화, action parity는 `packs/nms-ems/knowledge/frontend/six-gen-react-local-user-parity.md`를 함께 읽는다.
- React Security User Session의 active Angular 호출 경로, `session-mgmt/v1/connect` 계약, `account` 응답 필드, table parity는 `packs/nms-ems/knowledge/frontend/six-gen-react-user-session-parity.md`를 함께 읽는다.

## Content module 흐름
`sites/six-gen-flexible/contents.module.ts`가 실제 active content module이다.

현재 관찰된 provider import/registration 축:
- `DashboardContentProviderModule`
- `EquipmentContentProviderModule`
- `NeConfigurationContentProviderModule`
- `FaultContentProviderModule`
- `HistoryContentProviderModule`
- `SecurityContentProviderModule`
- `PerformanceContentProviderModule`
- `OperationContentProviderModule`
- `PreferenceContentProviderModule`
- `LocalUserMgmtContentProviderModule`
- `SixGenFlexibleContentProviderModule`
- `ShelfViewSixGenFlexibleProviderCOTModule`
- `ShelfViewSixGenFlexibleProviderRTModule`

주의:
- `My5GContentProviderModule`, `openHaulContentProviderModule` 등은 주석 처리될 수 있다.
- 메뉴가 안 보일 때는 provider import 여부를 먼저 확인한다.
- `src/app/content/contents/contents.module.ts`가 아니라 `sites/six-gen-flexible/contents.module.ts`를 먼저 본다.

## Dashboard Topology override

`six-gen-flexible`의 첫 Dashboard 화면은 generic Main Dashboard가 아니다.

- `SixGenFlexibleContentProvider`가 Main Dashboard를 비활성화하고 Equipment Topology를 `Dashboard > Topology`로 등록한다.
- 기본 URL은 `/dashboard/topology`이며, 실제 화면은 `SpaceTopologyComponent`이다.
- 최초 group은 `NeGroupService.getAuthorizedGroup(0)`으로 선택하고 선택 후 URL은 `/dashboard/topology/<group_id>`가 된다.
- 좌측 group panel은 250px, 우측 Detail panel은 400px이며 `Topology/TopologyView` user-profile의 `groupVisible`, `detailVisible` 값으로 표시 상태를 복원한다.
- `site` 또는 일반 group은 `EquipmentTopologyComponent.checkTopologyType()` 결과에 따라 `TopologyMap`, `Topology2D`, `Topology3D`로 분기한다. 특히 `site` 화면은 Cesium 기반 `TopologyMapComponent`이므로 React 이식 시 저장 `topology-item` SVG로 대체하면 기능 parity가 아니다.
- 좌측 tree의 `vCore`, `Floor`, `Block`, `Zone`은 `groupDisplayChecker`에서 제외한다. 권한 root나 `site_group` 노출 여부는 실제 `getAuthorizedGroup` 결과와 live menu/tree baseline으로 확인한다.
- panel 버튼은 group panel, Detail panel, 양쪽 panel 순서이며 변경할 때마다 같은 user-profile 계약으로 저장한다.

React parity 검증에서는 route/tree/panel 상태만 보지 말고 `site`의 지도 타일, building marker, alarm 색상, click detail, camera profile 저장까지 확인한다.

## Shell main menu state

좌측 main menu의 60px/250px 상태는 브라우저 로컬 설정이 아니라 사용자 profile 계약이다.

- state target: `new ControlStateTarget('home', 'main-menu')`
- API: `system/v1/user-profile`
- Get data: `{ user_id, type: 'ControlState', target: 'home.main-menu' }`
- value: `{"mode":"Icon"}` 또는 `{"mode":"General"}`
- `Icon`은 60px 기본 폭이며 hover 중에만 250px overlay로 펼쳐진다.
- `General`은 250px pinned 상태로 작업 영역 폭에 반영된다.
- pin 변경 저장은 기존 profile id가 있으면 `Modify`, 없으면 `Add`를 사용한다.
- React에서도 `access_token`과 request `client_session_id`는 활성 communication `session_id`를 사용해야 한다.

desktop browser parity에서는 현재 profile mode를 먼저 확인한다.

- 1440px viewport와 workspace padding 20px 기준 content 시작 x는 `Icon=80`, `General=270`이다.
- Icon mode의 menu label/child는 hover 전 CSS로 숨겨진다. Playwright에서 label 기반 role selector를 사용하려면 먼저 side menu를 hover해 Angular 사용자 동작을 재현한다.
- 화면별 geometry assertion에 `80` 또는 `270`을 고정하기 전에 fixture의 `home.main-menu` 응답이 `Icon`인지 `General`인지 확인한다.

live parity에서는 로그인 후 Get 응답의 mode가 초기 폭에 반영되는지와
`Icon -> General -> Icon` 변경이 같은 profile id로 저장되는지를 확인한다.

## Shell menu localization

좌측 menu와 workspace title은 `CurrentLanguage`의 `ko`, `en`, `ja` 값을 사용한다.

- 번역 기준은 Angular `src/assets/lang/*_{ko,en,ja}.json`이다.
- active six-gen menu는 top-level, child, footer를 합쳐 23개 항목이다.
- `Role`, `Service Status Log`처럼 active provider에서 숨긴 route title도 같은 번역 계약을 유지한다.
- route, authority code, provider id는 번역하지 않고 표시 label만 변환한다.
- live parity는 각 언어로 다시 로그인한 뒤 menu 순서와 active workspace title을 함께 비교한다.
- React live gate는 `npm run qa:six-gen-shell-menu-localization-live`이며,
  `en`, `ko`, `ja` 로그인 후 60px icon 상태와 250px hover 상태를 모두 비교한다.
- Angular는 접혀서 잘린 submenu DOM도 bounding box를 유지할 수 있다. 의미상
  menu inventory는 전체 DOM label로 비교하고, overflow/overlap/geometry는
  `elementFromPoint`로 실제 그려진 행만 판정해야 한다.

## ContentsManagerService 흐름
`src/app/common/content/content-define/content-manager.service.ts`가 content provider를 모으고 메뉴/UI를 초기화한다.

핵심 흐름:
1. `ContentProviderModule` 생성자에서 provider module이 `ContentsManagerService.addContentProviderModule(...)`로 등록된다.
2. `initializeProduct()`에서 등록된 module의 provider를 수집한다.
3. provider enabled 여부와 content authority를 기준으로 provider list를 필터링한다.
4. `initializeContentProvider(...)`에서 menu provider, UI provider, widget provider, extension provider를 초기화한다.
5. logout 시 content provider list와 UI 상태를 clear한다.

## Current user menu provider 흐름

desktop current-user menu의 활성 범위는 `main-toolbar.component.ts` 한 파일만으로 판단하지 않는다.

- 공통 `MainToolbarComponent`는 `User Preference`를 등록한다.
- `SixGenFlexibleContentProvider`는 `Edit Account`, `Change Password`를 `Main_CurrentUserMenu`에 주입한다.
- `HomeProvider.defineCurrentUserMenuDisplay()`가 최종 표시 순서를 `Edit Account -> Change Password -> User Preference`로 고정한다.
- 따라서 React 이식 baseline은 공통 toolbar, six-gen 제품 provider, `HomeProvider` 순서 정의와 로그인 후 live DOM을 함께 비교해야 한다.
- account/password panel은 메뉴 label뿐 아니라 readonly/disabled 필드, validation, confirmation, request payload와 `access_token`/`client_session_id` 계약까지 같은 gate에서 검증한다.

`six-gen-flexible` React 포팅 완료 범위는 `sites/six-gen-flexible/contents.module.ts`에서 활성화된 provider와 그 런타임 출력으로 고정한다. 제품에서 비활성화된 NOC 전용 메뉴, Cluster의 System Backup/Keycloak 화면은 완료 gate에서 제외하되, 공통 모델 직렬화 필드와 실제 startup sequence/WebSocket keepalive 의존성은 유지한다.

완료 route 목록을 React 메뉴 배열에서 다시 만들면 React 누락이 gate에서도 함께 누락되는 순환 검증이 된다. 따라서 baseline extractor는 active `NgModule.imports`를 AST로 읽고 provider module에서 `ContentProvider` source를 해석한 뒤, `defineMenu`와 user-facing `addMenuItem`의 교집합 menu ID를 저장한다. React scope는 이 Angular menu ID와 명시적 route map으로만 완료 route를 만든다. 새 source menu ID에 route 또는 구조 메뉴 `null` 매핑이 없으면 fail-closed로 gate를 중단한다.

`app.prepare.ts`와 product provider가 제외한 menu ID는 조상 관계까지 적용한다. 특히 `E_PreferenceMenuID.Cluster`가 비활성이면 Cluster Management, Alarm Threshold, System Backup, restore/upgrade 및 Keycloak 하위 route 전체를 aggregate parity에서 제외한다. 제외 화면의 공통 모델을 삭제하거나 request 직렬화와 session/startup 계약을 약화해서는 안 된다.

## React parity coverage gate

React 프로젝트에서는 범위 정의와 완료 증거를 분리한다.

- `scripts/angular-sixgen-source-extractor.mjs`: active provider와 source menu
  registration 추출
- `src/domain/sixGenMenuRouteMap.json`: Angular menu ID의 route/구조 메뉴 검토 경계
- `src/domain/sixGenPortingScope.ts`: source menu ID, active/excluded route와 provider 기준
- `src/domain/sixGenParityCoverage.ts`: 활성 shell, Preference, extension,
  NE Configuration inner route와 활성 current-user menu ID를 fixture/live QA와 연결
- `src/domain/sixGenRuntimeCoverage.ts`: request 직렬화, session field,
  첫 keepalive, startup BaseData 의존성을 실행 가능한 gate와 연결
- `npm run qa:six-gen-scope`: Angular source drift와 제외 범위 검증
- `npm run qa:six-gen-parity-coverage`: 활성 route 누락, 제외 route 유입,
  QA command 오타, evidence artifact 누락, runtime contract 누락 검증
- `npm run qa:six-gen-browser`: scope와 coverage를 순서대로 통과한 뒤 화면
  fixture suite를 실행

한 QA가 여러 route를 검증할 수 있다. 예를 들어 User Session live gate는
`/security/user-session`과 메뉴에 노출되지 않는 `/sixGenFlexible`을 함께
비교한다. 따라서 script 이름만으로 누락을 판단하지 말고 coverage profile의
route 목록과 durable artifact를 확인한다.

route coverage만으로 shell 완료를 판정하면 안 된다. `Edit Account`,
`Change Password`, `User Preference`처럼 route 밖에서 provider가 주입하는
current-user menu ID를 coverage profile에 명시하고, 인증 startup, account/password,
screen lock, shell localization, error page의 fixture/live command와 artifact를 함께
연결한다. 예상 ID를 source scope에서 그대로 복사해 자동 충족시키지 말고 검토된
ID를 명시해, Angular에 새 current-user item이 활성화되면 gate가 fail-closed로
중단되게 한다.

## 메뉴 미노출 조사 순서
1. active configuration이 `six-gen-flexible`인지 확인한다.
2. `angular.json` fileReplacement가 `sites/six-gen-flexible/contents.module.ts`를 가리키는지 확인한다.
3. 해당 provider module이 `sites/six-gen-flexible/contents.module.ts` imports에 있는지 확인한다.
4. `registerExcludeMenu(...)`나 `registerExcludeAuthority(...)`에서 제외되는지 확인한다.
5. provider의 `getEnabled()`, `useContentAuthority`, `authorityCode`를 확인한다.
6. `authorityCheck`가 걸린 menu/action인지 확인한다.
7. 브라우저에서 실제 권한과 메뉴 상태를 확인한다.

## NE Configuration 내부 flow
`NE Configuration > Configuration`은 일반 메뉴와 다르게 NE 선택 후 내부 configuration tree가 열린다.

장비 설정 조회/설정은 Command class 기반으로 동작하므로, 화면을 바꾸기 전에 `packs/nms-ems/knowledge/domain/ne-configuration-command.md`를 함께 읽는다.
React System 이식과 parity 검증은 `packs/nms-ems/knowledge/frontend/six-gen-react-ne-configuration-system-parity.md`를 함께 읽는다.
React PM Report 이식과 FTP/Set parity 검증은 `packs/nms-ems/knowledge/frontend/six-gen-react-ne-configuration-pm-report-parity.md`를 함께 읽는다.
React Port/Slot/FlexPort 이식과 command parity 검증은 `packs/nms-ems/knowledge/frontend/six-gen-react-ne-configuration-port-parity.md`를 함께 읽는다.
React Port Status/Statistics와 Unit/Module Inventory 이식은 `packs/nms-ems/knowledge/frontend/six-gen-react-ne-configuration-readonly-table-parity.md`를 함께 읽는다.
React SyncE/Service Interface/Time Configuration 이식은 `packs/nms-ems/knowledge/frontend/six-gen-react-ne-configuration-advanced-parity.md`를 함께 읽는다.
React Performance의 command 순서, parser, Angular 초기 중복 요청, live parity 검증은 `packs/nms-ems/knowledge/frontend/six-gen-react-ne-configuration-performance-parity.md`를 함께 읽는다.
React Fault/Active Alarm 이식과 조회·초기화·권한 parity 검증은 `packs/nms-ems/knowledge/frontend/six-gen-react-ne-configuration-active-alarm-parity.md`를 함께 읽는다.
React `Fault > Active Alarm`의 filter, Ack, Action History, 리소스, table/grid parity 검증은 `packs/nms-ems/knowledge/frontend/six-gen-react-fault-active-alarm-parity.md`를 함께 읽는다.
React NE Reboot 이식과 명령/권한/confirmation parity 검증은 `packs/nms-ems/knowledge/frontend/six-gen-react-ne-reboot-parity.md`를 함께 읽는다.
React ConfigBackup/Restore 이식과 backup/restore/download/delete parity 검증은 `packs/nms-ems/knowledge/frontend/six-gen-react-ne-configuration-backup-parity.md`를 함께 읽는다.
React MEF Service Management 이식과 COT/RT 조회, 서비스/인터페이스 명령 parity 검증은 `packs/nms-ems/knowledge/frontend/six-gen-react-mef-service-parity.md`를 함께 읽는다.
Angular `WCTextBox`의 one-way `[value]`는 `readonly`를 뜻하지 않는다. 부모가
`valueChange`를 구독하지 않으면 화면 입력만 바뀌고 요청 모델은 원래 값을
유지할 수 있으므로, React 이식 시 실제 입력 상태와 wire payload를 함께 비교한다.
React Preference Storage Retention의 system-config 직렬화, numberbox 동작, live parity 검증은 `packs/nms-ems/knowledge/frontend/six-gen-react-preference-storage-period-parity.md`를 함께 읽는다.
React User Preference의 ScreenLock/AlarmSoundConfig request, Default/Apply,
WC control token, audio runtime parity 검증은
`packs/nms-ems/knowledge/frontend/six-gen-react-user-preference-parity.md`를 함께 읽는다.

중요 경로:
- 공통 module: `src/app/content/contents/neconfiguration/content/configuration/configuration-main-content.module.ts`
- six-gen provider: `src/app/content/contents/neconfiguration/content/configuration/six-gen-flexible/provider/`
- six-gen content: `src/app/content/contents/neconfiguration/content/configuration/six-gen-flexible/content/`

현재 코드상 six-gen provider는 COT/RT 계열 module/provider로 나뉘며, `NeCodeBook`의 NE code와 연결된다.

내부 메뉴는 provider의 `createMenuNodes()`에서 생성하고, 각 menu는 `authorityCheck`와 `addUIInfo(...)`로 실제 UI module/component에 연결된다.

각 설정 화면 component는 보통 `ConfigurationPanel` 계열을 상속하고, `getCommands(...)`에서 `ButtonType.refresh` 조회 command와 `ButtonType.set` 설정 command를 구성한다. 응답은 `display(...)`에서 form/table 상태로 매핑한다.

## 신규 기능 추가 시 필수 체크
- 새 top-level 메뉴인가, 기존 content provider 아래 sub menu인가?
- active site content module에 provider import가 필요한가?
- `ContentsManagerService` filtering/authority 영향이 있는가?
- `authorityCheck`가 필요한 action/menu인가?
- route/content module lazy loading이 필요한가?
- 기존 `wc-table`, `wc-form`, `wc-toolbar`, `wc-split-panel` 패턴으로 구현 가능한가?
- NE Configuration 설정 기능이면 Command class, `NeCommandDefineArray`, `getCommands(...)`, `display(...)` read/write path가 맞는가?
- 매뉴얼 캡처 갱신이 필요한 화면 변경인가?

## React Dashboard Topology parity

React의 활성 2D Topology 분기, node/link/detail/zoom/panel profile 및 browser gate는 `packs/nms-ems/knowledge/frontend/six-gen-react-dashboard-topology-parity.md`를 함께 확인한다.

## React 19 상태와 반복 입력 ref 규칙

- render 중 `ref.current`를 prop 값으로 대입하지 않는다. prop-to-ref 동기화는
  effect에서 수행한다.
- WC numberbox처럼 pointer hold가 같은 event loop에서 최신 값을 이어서
  사용해야 하는 control은 commit 함수에서 state와 ref를 함께 갱신한다.
- 함수형 state updater 안에서 다른 state setter, storage 쓰기, request
  cancel 같은 side effect를 실행하지 않는다. 먼저 next state를 계산한 뒤
  관련 side effect를 updater 밖에서 순서대로 실행한다.
- 비동기 effect가 첫 `await` 전에 만든 observer, viewer, timer, event
  subscription은 effect 로컬 소유권에 즉시 기록한다. cleanup이 최종 runtime
  ref 공개 여부에만 의존하면 초기화 도중 unmount에서 리소스가 남을 수 있다.
- `ResizeObserver`, Cesium `Viewer`, emitter listener 정리는 여러 경로에서
  호출되어도 안전하게 만든다. 이벤트 등록 helper는 실제 unsubscribe 함수를
  반환하고 effect가 그 핸들을 직접 보관하는 방식을 우선한다.
- updater 밖에서 storage를 저장할 때 연속 입력의 최신값이 필요하면 latest-value
  ref로 next state를 계산하고 `persist -> ref -> state` 순서를 고정한다.
- 이 규칙을 적용해도 Angular의 500ms/50ms held-repeat, Enter/blur commit,
  blank-blur restore, panel close cancel/reset 순서는 그대로 유지해야 한다.
- 활성 Six-Gen 화면의 React 19 오류는 fixture와 실제 Angular/React browser
  parity를 함께 통과해야 수정 완료로 판단한다.
