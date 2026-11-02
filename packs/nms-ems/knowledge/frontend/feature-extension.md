# NMS FE 기능 확장점 표준

## 목적
NMS FE(uems_fe)에 기능을 추가/수정할 때 "어디에 끼우는가"를 표준화한다. 기존 구조를 바꾸지 않고 필요한 확장점(seam)에만 코드를 추가하도록 안내한다.

- 이 문서: **어디에 끼우는가** (확장점, 등록 위치, 등록 실패 원인)
- `packs/nms-ems/knowledge/frontend/new-feature-flow.md`: **어떤 순서로 하는가** (절차, 산출물)
- 구조 이해가 먼저 필요하면 `packs/nms-ems/knowledge/frontend/architecture.md`
- BE 확장점은 `packs/nms-ems/knowledge/backend/feature-extension.md` (FE 대칭 문서)

모든 경로는 uems_fe 저장소 루트 기준이다.

## 핵심 registry
FE의 화면/메뉴/위젯 등록은 전부 `src/app/common/content/content-define/content-manager.service.ts`의 `ContentsManagerService`로 모인다.

- `ContentProviderModule` 생성자가 `ContentsManagerService.addContentProviderModule(this)`를 호출해 자기 자신을 static 목록에 등록한다. 즉 **provider module이 NgModule import로 로드되기만 하면 자동 등록**된다.
- 기동 시퀀스(`ContentServicePrepareProcess`, 같은 파일)가 `initializeProduct()` → `initialize()`를 호출한다.
  - `initializeProduct()`: 각 module의 `getContentProvider()`를 수집. `useContentAuthority`가 true인데 `authorityCode == null`이면 error 로그 후 **제외**.
  - `initialize()`: `getEnabled()`(NMS/NOC 타겟 필터) → `authService.checkAuthority(authorityCode)`(권한 없으면 제외) → 통과한 provider마다 `getMenuProvider()`/`getUIProvider()`/`getWidgetProvider()`/`getExtensionProviderList()`를 초기화.
  - 메뉴 등록 시 `excludeMenuIDs` 포함 여부와 `WCMenuDefine.authorityCheck`(`isMenuAuthorized` → `authService.checkAuthorityView`)를 검사한다.

## 기능 요청 분해 순서
요청을 받으면 다음 순서로 좁힌다.

1. **어느 화면/도메인인가** — 기존 content(feature) 폴더에 속하는가(`src/app/content/contents/<feature>/`), 아니면 새 content인가. 기존 화면 수정이면 새 seam이 필요 없을 수 있다.
2. **어느 seam인가** — 아래 seam 표에서 메뉴/route/화면/서비스/위젯/권한 중 무엇을 추가하는지 결정한다. 대부분의 "새 메뉴 + 새 화면" 요청은 seam 2~5를 함께 쓴다.
3. **등록 위치는 어디인가** — cheat-sheet 절 참조. 특히 provider module import와 route는 **generic 파일이 아니라 active site 파일**에 있다.
4. **site 영향 확인** — `angular.json`의 build configuration별 `fileReplacements`를 확인해 어느 site 빌드에 노출되는지, `registerExcludeMenu`로 제외되는 site가 있는지 확인한다.

## 확장 seam 표
아래 seam은 전부 실제 코드로 확인된 것이다. "근거 파일"의 fault content를 표준 예시로 삼는다.

| # | seam | 추가 방법 | 등록 위치 / 근거 파일 |
| --- | --- | --- | --- |
| 1 | Content Provider (새 content 도메인) | `src/app/content/contents/<feature>/provider/<feature>-content-provider.module.ts`에 `ContentProviderModule` 상속 NgModule 작성. `contentType`, `getContentProvider()`, (필요 시) `getContentExternalRoutes()`, `getBaseDataLoaderList()` 구현. provider 본체는 `<feature>-content-provider.ts`에서 `ContentProvider` 상속: `contentType`/`name`/`authorityCode` + `getMenuProvider()`/`getUIProvider()`/`getWidgetProvider()` | 예시: `src/app/content/contents/fault/provider/fault-content-provider.module.ts`, `fault-content-provider.ts`. module을 **active site의 `contents.module.ts` imports에 추가해야 등록**됨 |
| 2 | 메뉴 (`MenuProvider`) | provider의 `getMenuProvider()`가 반환하는 `MenuProvider` 하위 클래스의 `initializeMenu()`에서 `defineMenu(menuID, text, icon, isToggle, url)`로 정의하고 `addMenuItem(E_HomeMenuTarget.Main_Menu, parentMenuID, menuID)`로 트리에 배치. 권한 제어는 `tMenu.authorityCheck = <AuthorityCode>`. 배지는 `badgeManager.addBadgeMapping`. toolbar 버튼은 `addToolbarButton`/`addToolbarMenu` | `MenuProvider` API: `src/app/common/content/content-define/menu-common.ts`. 예시: `FaultContentMenuProvider.initializeMenu()` (`fault-content-provider.ts`) |
| 3 | Route / LazyLoad | (a) content 최상위 route: **active site의** `sites/<site>/contents.module.ts`의 `CONTENTS_ROUTES`에 `{ path: <Feature>ContentCommon.s_Content_Url, loadChildren: () => import(...) }` 추가 (`src/app/app.routes.ts`가 `CONTENTS_ROUTES`를 children으로 사용). (b) 다른 content 밑에 끼우는 route: provider module의 `getContentExternalRoutes()`에서 `RouteDefineCollection.addRoute(대상ContentType, parentPath, ROUTES)` — 대상 content provider가 `ContentsManagerService.serviceInstance.getExternalRoutes(...)`로 회수 | 예시: fault가 History/Preference content 밑에 route를 끼움 (`fault-content-provider.module.ts`의 `HISTORY_ROUTES`, `PREFERENCE_ROUTES`). 회수 예시: `src/app/content/contents/history/provider/history-content-provider.ts` |
| 4 | 화면 컴포넌트 | `src/app/content/contents/<feature>/content/<screen>/`에 component + lazy module 작성. 공통 `wc-*` 컴포넌트 재사용 우선 (`packs/nms-ems/knowledge/components/wc-usage-map.md`) | 예시: `src/app/content/contents/fault/content/current-fault/current-alarm/current-alarm.component.ts` |
| 5 | Content Service / API | content 상태·화면 helper: `src/app/content/common/<feature>/<feature>-content.service.ts` — `ContentService` 상속, `contentUrl`/`contentType`/`contentAuthorityCode` 구현. 상수·enum(`s_ContentType`, `s_Content_Url`, `E_<Feature>MenuID`, `<Feature>AuthorityCodes`)은 `<feature>-content.ts`에 정의. REST/데이터 계층은 도메인별 데이터 서비스(`src/app/dbdata/<domain>/<entity>/<entity>.service.ts`) | 예시: `src/app/content/common/fault/fault-content.service.ts`, `fault-content.ts`, `src/app/dbdata/unify_fault/alarm/alarm.service.ts` |
| 6 | Site별 override | site 빌드는 `angular.json` `fileReplacements`로 5개 파일을 통째로 교체: `src/app/app.prepare.ts` → `sites/<site>/app.prepare.ts`, `src/app/content/contents/contents.module.ts` → `sites/<site>/contents.module.ts`, `src/app/common/common/product.ts` → `sites/<site>/product.ts`, `src/app/app.routes.ts` → `sites/<site>/app.routes.ts`, `src/tsconfig.app.json` → `sites/<site>/tsconfig.app.json` (+ `index`는 `sites/<site>/index.html`). site별 메뉴 제외는 `app.prepare.ts`(AppPrepareModule)에서 `registerExcludeMenu()` override → `aContentManager.addExcludeMenu(E_...MenuID.xxx)` | site 목록: `sites/`(default, noc, das, hiems, mobile, mobile-cpe, user-mgmt, customer-mgmt, noc-user-mgmt, system-mgmt, openHaul). 제외 예시: `sites/das/app.prepare.ts`. base class: `src/app/app.common.ts`의 `AppPrepareModuleBase` |
| 7 | 권한 (AuthorityCode) | `src/app/common/auth/authority-code.ts`의 `AuthorityCode` 클래스(enum 아님). content별 코드 묶음 클래스에 `new AuthorityCode(contentType, group, authority, displayText)` static 필드로 추가하면 `AuthorityCode.manager`에 자동 등록. provider의 `authorityCode`(content 진입 권한)와 메뉴의 `authorityCheck`(메뉴 노출 권한)에 사용 | 예시: `FaultAuthorityCodes` (`src/app/content/common/fault/fault-content.ts`) |
| 8 | BaseData entity | `src/app/basedata/<entity>/`에 entity + `<entity>.service.ts`(`BaseDataServiceSync` 등 상속, `baseDataType`/`notificationType` 구현). 로딩 등록 2가지: (a) site `app.prepare.ts`의 `registerBaseDataLoader()`에서 `BaseDataLoadProcess.addBaseDataLoader(XxxService)`, (b) provider module의 `getBaseDataLoaderList()` 반환 — `ContentProviderModule` 생성자가 자동으로 `addBaseDataLoader` 호출 | 예시: `src/app/basedata/ne/ne.service.ts`, 등록: `sites/default/app.prepare.ts` |
| 9 | Dashboard Widget | provider의 `getWidgetProvider()`가 반환하는 `WidgetProvider` 하위 클래스의 `initializeWidget()`에서 `defineWidget(category, widgetID, title, width, height, ComponentInfo)` 정의 후 `addWidget(E_DashboardTarget.Main_Dashboard, widgetID)`로 배치 | API: `src/app/common/content/content-define/dashboard-common.ts`. 예시: `FaultContentWidgetProvider` (`fault-content-provider.ts`) |
| 10 | Preference 화면/섹션 | (a) Preference content 밑 화면: provider module `getContentExternalRoutes()`로 `PreferenceContentCommon.s_ContentType`에 route 추가 + `MenuProvider`에서 `addMenuItem(E_HomeMenuTarget.Main_Preference, ...)`로 메뉴 등록. (b) User Preference 사이드 섹션: `UIProvider.initializeUI()`에서 `defineUI(...)` 후 `addUI(E_HomeUIID.UserPreference, null, uiID)` | 예시: fault의 `PREFERENCE_ROUTES`(`fault-content-provider.module.ts`), `FaultUIProvider`의 `FaultPreferenceAlarmSound` 등록 (`fault-content-provider.ts`) |
| 11 | UI 확장 슬롯 (`UIProvider`) | 다른 content가 열어 둔 UI target에 component를 끼움: `UIProvider.initializeUI()`에서 `defineUI(uiID, text, image, new ComponentInfo(Component, contentService))` → `addUI(targetID, parentID, uiID)` | API: `src/app/common/content/content-define/ui-common.ts` |
| 12 | Content Extension Provider | content 간 확장: provider의 `getExtensionProviderList()` 반환 목록이 `contentExtesionProviderList`에 등록되고, 대상 content가 `getExtensionProviderList(contentType, providerTarget)`로 회수 | `content-manager.service.ts`의 `initializeContentProvider()` / `getExtensionProviderList()` |
| 13 | 실시간 push (WebSocket notification) | 데이터 서비스가 `INotificationReceiver`(`framework/core/process`)를 구현하거나 `BaseDataServiceSync`의 `notificationType`/`notificationSyncType`으로 서버 notification을 수신. 화면/서비스에서 수동 등록은 `ContentService.addNotificationReceiver(receiver)` | 예시: `AlarmService implements INotificationReceiver` (`src/app/dbdata/unify_fault/alarm/alarm.service.ts`), `NeService`의 `notificationType`. WebSocket 사용 여부는 `sites/<site>/product.ts`의 `isUseWebSocket` |

## 등록 위치 cheat-sheet

### provider module / route는 site 파일에 있다 (가장 중요)
- generic `src/app/content/contents/contents.module.ts`는 **빈 stub**이다(`CONTENTS_ROUTES = []`, imports 없음). 실제 빌드에서는 `angular.json` `fileReplacements`가 이 파일을 `sites/<site>/contents.module.ts`로 교체한다.
- 따라서 새 provider module import와 새 `CONTENTS_ROUTES` 항목은 **대상 site의 `sites/<site>/contents.module.ts`에 추가**해야 한다. 여러 site에 노출해야 하면 각 site 파일에 각각 추가한다.
- site별 imports 차이 예: `sites/noc/contents.module.ts`에는 `NOCContentProviderModule`, `GponContentProviderModule`, `WorkMgmtContentProviderModule`이 있으나 `sites/default/contents.module.ts`에는 없다(default는 `NOCClientContentProviderModule` 사용).

### initializer / 기동 등록
- start sequence, BaseData loader, exclude menu: `sites/<site>/app.prepare.ts`의 `AppPrepareModule` (base: `src/app/app.common.ts`의 `AppPrepareModuleBase` — `registerExcludeMenu` → `registerStartSequence` → `registerBaseDataLoader` 순으로 호출).
- content provider 초기화 자체는 `ContentServicePrepareProcess` start sequence가 수행하므로 별도 등록 불필요 — provider module이 import 되어 있기만 하면 된다.

### 권한
- 새 content면 provider `authorityCode` 필수 (없으면 `initializeProduct()`에서 error 로그 후 provider 제외). 권한 개념이 없는 content만 `useContentAuthority` getter를 override해 false 반환.
- 메뉴 단위 노출 제어는 `WCMenuDefine.authorityCheck`에 `AuthorityCode` 할당. 미할당 시 항상 노출(`isMenuAuthorized`가 true 반환).
- 새 `AuthorityCode`는 `<feature>-content.ts`의 `<Feature>AuthorityCodes` 클래스에 static 필드로 추가 (생성자에서 자동 등록). BE 권한 데이터와 코드 문자열이 일치해야 하므로 권한 추가는 BE 영향으로 함께 보고한다.

### 상수/식별자 컨벤션 (`src/app/content/common/<feature>/<feature>-content.ts`)
- `<Feature>ContentCommon.s_ContentType = new ContentType('<Feature>')`, `s_Content_Url`
- `E_<Feature>MenuID`, `E_<Feature>UIID`, `E_<Feature>WidgetID` enum
- `<Feature>AuthorityCodes`, `<Feature>ContentLogger`

## "추가했는데 안 보임" 흔한 원인
전부 코드에서 확인된 메커니즘이다. 위에서부터 순서대로 점검한다.

1. **provider module을 generic contents.module.ts에만 import** — generic 파일은 빈 stub이고 빌드 시 `sites/<site>/contents.module.ts`로 교체된다. active site 파일에 import했는지 확인한다. (`angular.json`의 해당 configuration `fileReplacements` 확인)
2. **provider module import 누락** — `ContentProviderModule`은 NgModule import로 로드될 때만 `addContentProviderModule`이 실행된다. import가 없으면 어떤 로그도 없이 그냥 빠진다.
3. **`authorityCode` 누락** — `useContentAuthority`가 true(기본)인데 `authorityCode`가 null이면 `initializeProduct()`가 `'There is no authority code.'` error를 남기고 provider를 제외한다.
4. **사용자 권한 부족** — `initialize()`에서 `checkAuthority(authorityCode) === E_Authority.None`이면 provider 전체 제외. 메뉴 단위로는 `authorityCheck` 실패 시 `defineMenu`/`addMenuInstance`가 스킵된다. 콘솔 `'The content authority rating.'` 로그로 판정값을 확인한다.
5. **`getEnabled()` 필터** — provider의 `contentProviderTargets`(기본 NOC+NMS)와 서버 환경(`SystemEnvironmentService.instance.isNOC`)이 안 맞으면 제외된다. NOC 전용/NMS 전용 provider인지 확인한다. (`content-define.ts`의 `getEnabled()`)
6. **site exclude** — `sites/<site>/app.prepare.ts`의 `registerExcludeMenu()`에서 `addExcludeMenu(menuID)`로 제외된 메뉴는 define/instance 등록 자체가 스킵된다 (`'The menu define is excluded.'` 로그). 예: das site의 Performance 메뉴.
7. **메뉴 define만 하고 instance 누락(또는 반대)** — `defineMenu`는 정의만, `addMenuItem`이 실제 트리 배치다. 둘 다 필요하다. parent menuID가 다른 content 소유일 수 있음(예: fault가 `E_HistoryMenuID.Main_Menu_History` 밑에 추가).
8. **route 누락** — 메뉴는 보이는데 클릭 시 빈 화면이면 `CONTENTS_ROUTES`(site 파일) 또는 `getContentExternalRoutes()` 등록을 확인한다. 다른 content 밑 route는 대상 provider가 `getExternalRoutes(...)`를 호출해줘야 반영된다.
9. **다른 site 빌드/serve 확인** — `angular.json`의 configuration(default/noc/das/...)별로 `baseHref`, `outputPath`, 교체 파일이 다르다. 지금 serve 중인 configuration이 수정한 site와 같은지 확인한다.

## 검증 시 확인한 근거 파일 (재검증용)
- `src/app/common/content/content-define/content-manager.service.ts` (`ContentsManagerService`, `ContentProviderModule`, `ContentServicePrepareProcess`, `ContentService`)
- `src/app/common/content/content-define/content-define.ts` (`ContentProvider`, `ContentType`, `getEnabled`)
- `src/app/common/content/content-define/menu-common.ts` / `ui-common.ts` / `dashboard-common.ts`
- `src/app/content/contents/fault/provider/` + `src/app/content/common/fault/`
- `sites/default/contents.module.ts`, `sites/noc/contents.module.ts`, `sites/default/app.prepare.ts`, `sites/das/app.prepare.ts`, `sites/default/product.ts`, `src/app/app.common.ts`, `angular.json`
