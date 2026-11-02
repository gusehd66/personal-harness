# NMS FE Provider/Menu 등록 흐름

## 목적
새 content(메뉴 + 화면 묶음)를 추가하거나 기존 메뉴가 보이지 않는 문제를 다룰 때,
`ContentProvider`/`MenuProvider`/`ContentsManagerService` 등록 체인을 정확히 따라가도록 한다.

경로는 FE 저장소 루트(uems_fe) 기준 상대 경로로 표기한다.

핵심 파일:
- `src/app/common/content/content-define/content-manager.service.ts` — `ContentsManagerService`, `ContentProviderModule`, `ContentServicePrepareProcess`, `ContentService`, `ContentModule`
- `src/app/common/content/content-define/content-define.ts` — `ContentProvider`, `ContentType`, `E_ContentProviderTarget`, `RouteDefineCollection`
- `src/app/common/content/content-define/menu-common.ts` — `MenuProvider`, `MenuManager`
- `sites/<site>/contents.module.ts` — site별 provider module import + `CONTENTS_ROUTES` (fileReplacement 대상)
- `src/app/app.prepare.ts` — start sequence 등록
- `src/app/home/home-provider.ts` — main menu display define (메뉴 표시 순서/포함 목록)

## 등록 → 초기화 → 메뉴 노출 호출 체인
아래 단계는 전부 실제 코드에서 확인한 것이다.

### 1단계. 빌드 타임: site contents.module.ts가 provider module을 import
- `angular.json`의 fileReplacement가 `src/app/content/contents/contents.module.ts`를 `sites/<site>/contents.module.ts`로 교체한다(default/noc/das/hiems/openHaul 등 site별 항목 존재).
- `src/app/app.module.ts`가 `ContentsModule`을 import하고, `src/app/app.routes.ts`가 같은 파일의 `CONTENTS_ROUTES`를 사용한다.
- 즉 **provider가 앱에 포함되는 유일한 진입점은 active site의 `contents.module.ts` `imports` 배열**이다. 여기 없으면 이후 단계는 아예 일어나지 않는다.

### 2단계. 모듈 생성 시: ContentProviderModule 생성자에서 static 등록
`ContentProviderModule`(추상, `content-manager.service.ts`) 생성자가 수행:
1. `ContentsManagerService.addContentProviderModule(this)` → static `s_ContentProviderModuleList`에 `contentType` 키로 추가(중복 시 error 로그만 남기고 무시).
2. `getContentExternalRoutes()` 결과를 `contentManager.contentExternalRoutes`(HashMapHashtable<ContentType, parentPath, Routes>)에 병합 — 다른 content(예: History, Preference) 밑에 붙는 lazy route 등록.
3. `getBaseDataLoaderList()` 결과를 `BaseDataLoadProcess.addBaseDataLoader(...)`에 등록.

이 시점에는 provider "module"만 등록되고 provider 인스턴스는 아직 만들어지지 않는다.

### 3단계. 앱 기동 시퀀스: ContentServicePrepareProcess
`src/app/app.prepare.ts`의 `AppPrepareModule.registerStartSequence()`가 순서대로 등록:
`CommunicationProcess` → `LoginStartSequence` → `BaseDataLoadProcess` → `ContentServicePrepareProcess`.
즉 provider 초기화는 **로그인/권한 정보와 base data가 준비된 후** 실행된다.

`ContentServicePrepareProcess`(`content-manager.service.ts`):
- `initializeSequence()` → `contentService.initializeProduct()`
- `startSequence()` → `contentService.initialize()`

### 4단계. initializeProduct(): provider 인스턴스 수집
`ContentsManagerService.initializeProduct()`:
- 각 module의 `getContentProvider()` 호출(대부분 `createProvider(XxxContentProvider)` = `injector.get(...)` — 그래서 provider module의 `@NgModule({ providers: [XxxContentProvider] })`가 필수).
- provider가 null이면 `'There is no content provider.'` error 로그 후 탈락.
- `useContentAuthority`가 true인데 `authorityCode`가 null이면 `'There is no authority code.'` error 로그 후 **탈락**.
- 통과 시 static `s_ContentProviderList`에 추가하고 `productService.addProduct(tProvider.getProduct())`.
- 마지막에 `productService.initializeProducts()`.

### 5단계. initialize(): enable/권한 필터링
`ContentsManagerService.initialize()` (isInitialized guard, 로그아웃 시 `onLogout()`이 리셋):
1. 각 provider에 대해 `getEnabled()` 확인 — `content-define.ts`의 구현:
   - `contentProviderTargets`(기본 `[NOC, NMS]`, `setContentProviderTargets(...)`로 변경)와
     `SystemEnvironmentService.instance.isNOC` / `ProductInformation.config?.isNOC`를 비교해 NMS/NOC 빌드별 활성 여부 결정.
2. `useContentAuthority`이면 `authService.checkAuthority(tProvider.authorityCode)` 실행,
   결과가 `E_Authority.None`이면 provider 제외(NOC 빌드의 NOC provider는 예외적으로 강제 포함).
3. 살아남은 provider만 `s_ContentProviderList`에 재구성 후 각각 `initializeContentProvider(tProvider, ...)` 호출.
4. 끝으로 `framework.firstRedirectUrl`이 비어 있으면 `defaultMainUrl`(메뉴 display define 순서상 첫 번째 권한 있는 메뉴의 url)로 설정.

### 6단계. initializeContentProvider(): 메뉴/UI/위젯 등록
`ContentsManagerService.initializeContentProvider(aProvider, aMainMenuItemList)`:
1. `aProvider.contentManager = this; aProvider.initialize();`
2. `aProvider.getMenuProvider()` → `MenuProvider.initialize(this)`가 `initializeMenu()` + `initializeMenuDisplay()` 실행.
   - `menuDefineList`의 각 define: `excludeMenuIDs`에 있으면 제외, `isMenuAuthorized()`(define의 `authorityCheck`가 있으면 `authService.checkAuthorityView(...)`) 통과 시 `menuManager.defineMenu(tDefine)`.
   - `menuInstanceList`의 각 instance: 대응하는 define이 존재하고 권한 통과 시 `menuManager.addMenuInstance(tInstance)`.
   - `tMenuProvider.getContentExternalRoutes()`도 `contentExternalRoutes`에 병합(2단계와 동일 구조).
3. `aProvider.getUIProvider()` → `uiDefineList`/`uiInstanceList`를 `uiManager.addUIDefine`/`addUI`.
4. `aProvider.getWidgetProvider()` → `initializeWidgetProvider()`가 `dashboardManager.addWidgetDefine`/`addWidget`.
5. `aProvider.getExtensionProviderList()` → `contentExtesionProviderList`에 (targetContent, providerTarget) 키로 축적.

### 7단계. 화면에 실제로 보이는 조건: display define
메뉴 define/instance 등록만으로는 최상위 메뉴 순서가 정해지지 않는다.
`src/app/home/home-provider.ts`의 `defineMainMenuDisplay()`가
`menuManager.addDisplayDefine(E_HomeMenuTarget.Main_Menu)` 후 `tDefine.addItem(<MenuID>)`로
**표시할 메뉴 ID와 순서를 명시적으로 나열**한다(예: `tDefine.addItem(E_FaultMenuID.Fault_Current)`).
`defaultMainUrl`도 이 display define의 `itemList` 순서로 첫 URL을 고른다.
새 최상위 메뉴는 여기(또는 해당 site의 display define)에 추가하지 않으면 노출 위치가 잡히지 않는다.

## 신규 provider 등록 체크리스트
새 content `<name>`을 추가할 때 순서대로 확인한다. 기존 구조는 바꾸지 않는다.

1. [ ] **ContentType/공통 상수 정의** — `src/app/content/common/<name>/<name>-content.ts`류에
   `s_ContentType = new ContentType('...')`, `s_Content_Url`, MenuID enum, `AuthorityCode` 정의.
   (참고 예시: `src/app/content/common/fault/fault-content.ts`)
2. [ ] **ContentProvider 클래스** — `src/app/content/contents/<name>/provider/<name>-content-provider.ts`
   - `extends ContentProvider`, `contentType`/`name`/`authorityCode` getter 구현.
   - `getMenuProvider()`/`getUIProvider()` 구현(없으면 null 반환 가능), 필요 시 `getWidgetProvider()`.
   - NMS/NOC 노출 범위가 다르면 생성자에서 `setContentProviderTargets(...)`.
3. [ ] **MenuProvider 클래스** — 같은 파일 내 `extends MenuProvider`, `initializeMenu()`에서
   `defineMenu(menuID, text, icon, isToggle, url)` + `addMenuItem(E_HomeMenuTarget.Main_Menu, parentID, menuID)`.
   메뉴별 권한은 define의 `authorityCheck`에 `AuthorityCode` 지정.
4. [ ] **ContentProviderModule** — `src/app/content/contents/<name>/provider/<name>-content-provider.module.ts`
   - `@NgModule({ providers: [<Name>ContentProvider] })`, `extends ContentProviderModule`.
   - `contentType` getter, `getContentProvider(): return this.createProvider(<Name>ContentProvider);`
   - 다른 content 밑에 화면을 붙이면 `getContentExternalRoutes()`에 `RouteDefineCollection.addRoute(targetContentType, parentPath, routes)`.
5. [ ] **site 등록** — 대상이 되는 **모든 active site**의 `sites/<site>/contents.module.ts`에
   - `imports` 배열에 `<Name>ContentProviderModule` 추가.
   - 독립 content면 `CONTENTS_ROUTES`에 `{ path: <Name>ContentCommon.s_Content_Url, loadChildren: ... }` 추가.
6. [ ] **display define** — 최상위 메뉴면 `src/app/home/home-provider.ts` `defineMainMenuDisplay()`에
   `tDefine.addItem(<MenuID>)` 추가(하위 메뉴는 부모 item의 `addItem`).
7. [ ] **권한** — `authorityCode`가 null이면 `initializeProduct()`에서 provider가 탈락한다.
   `AuthorityCode` 정의와 서버측 권한 데이터가 함께 있어야 한다. 서버측 확장은
   `packs/nms-ems/knowledge/backend/feature-extension.md`를 참조한다.
8. [ ] **검증** — 브라우저 콘솔에서 `Add content provider module.` / `The content authority rating.` 로그로
   module 등록과 권한 판정을 확인한다(ContentLogger 로그).

## 트러블슈팅
| 증상 | 원인 후보 | 확인 위치 |
| --- | --- | --- |
| 메뉴가 아예 안 보임 | active site `contents.module.ts` imports에 provider module 없음 | `sites/<site>/contents.module.ts`, `angular.json` fileReplacement로 active site 먼저 확인 |
| 메뉴가 아예 안 보임 | `getEnabled()` false — NMS/NOC 타겟 불일치 | `content-define.ts` `ContentProvider.getEnabled()`, provider의 `setContentProviderTargets(...)`, `SystemEnvironmentService.instance.isNOC` |
| 메뉴가 아예 안 보임 | provider 권한 `E_Authority.None` 또는 `authorityCode == null` | `ContentsManagerService.initializeProduct()`/`initialize()`의 error/info 로그, `authService.checkAuthority(...)` |
| 특정 메뉴만 안 보임 | menu define의 `authorityCheck`가 `checkAuthorityView(...)` false | `ContentsManagerService.isMenuAuthorized()`, 해당 MenuProvider의 `defineMenu(...).authorityCheck` |
| 특정 메뉴만 안 보임 | `excludeMenuIDs`에 등록됨 (`addExcludeMenu`) | `ContentsManagerService.initializeContentProvider()`의 `'The menu define is excluded.'` 로그 |
| 최상위 메뉴 위치/순서 이상 또는 미노출 | Main_Menu display define에 메뉴 ID 미등록 | `src/app/home/home-provider.ts` `defineMainMenuDisplay()` |
| 메뉴 클릭 시 빈 페이지 | `CONTENTS_ROUTES`에 route 없음 / loadChildren 경로 오류 | `sites/<site>/contents.module.ts`의 `CONTENTS_ROUTES`, `src/app/app.routes.ts` |
| 메뉴 클릭 시 빈 페이지 | menu define의 `url`과 route `path` 불일치 | MenuProvider `initializeMenu()`의 `HTMLHelper.makeUrlPath(...)` vs route path |
| 다른 content 하위 화면(예: History 밑 Alarm History)만 빈 페이지 | external route 미등록 | provider module의 `getContentExternalRoutes()`, `ContentsManagerService.getExternalRoutes()` 호출측(target content module) |
| 권한 오류/버튼 비활성 | content 권한 등급 판정 | `ContentsManagerService.getContentAuthority()`, `ContentService.isContentEditable`, `useContentAuthority` override 여부 |
| 로그인 직후만 정상, 재로그인 후 이상 | 로그아웃 시 registry 초기화 후 재초기화 실패 | `ContentsManagerService.onLogout()` (provider list/menu/UI clear, `isInitialized=false`) |

## 실제 예시: Fault content 추적
default site 빌드 기준으로 Fault 메뉴가 뜨기까지의 실제 경로.

1. **site import**: `sites/default/contents.module.ts` — `imports: [..., FaultContentProviderModule, ...]`,
   `CONTENTS_ROUTES`에 `{ path: FaultContentCommon.s_Content_Url, loadChildren: () => import('app/content/contents/fault/content/fault-content.module')... }`.
2. **provider module**: `src/app/content/contents/fault/provider/fault-content-provider.module.ts`
   - `@NgModule({ providers: [FaultContentProvider] })`, `extends ContentProviderModule`.
   - `contentType` = `FaultContentCommon.s_ContentType`.
   - `getContentExternalRoutes()`: `HISTORY_ROUTES`(alarm/event history, service status log)를
     `HistoryContentCommon.s_ContentType`에, `PREFERENCE_ROUTES`를 `PreferenceContentCommon.s_ContentType`에 등록
     — Fault 화면이지만 History/Preference content의 route 밑에 붙는 구조.
   - `getContentProvider()`: `this.createProvider(FaultContentProvider)`.
3. **provider**: `src/app/content/contents/fault/provider/fault-content-provider.ts`
   - `FaultContentProvider extends ContentProvider`; `authorityCode = FaultAuthorityCodes.fault_content`.
   - `getMenuProvider()` → `FaultContentMenuProvider`, `getUIProvider()` → `FaultUIProvider`,
     `getWidgetProvider()` → `FaultContentWidgetProvider`.
4. **메뉴 정의**: `FaultContentMenuProvider.initializeMenu()`
   - `defineMenu(E_FaultMenuID.Fault_Current, ResourceFault.fault.string, ..., tCurrentAlarmUrl)` +
     `authorityCheck = FaultAuthorityCodes.fault_content`.
   - `addMenuItem(E_HomeMenuTarget.Main_Menu, null, E_FaultMenuID.Fault_Current)` (badge 연동:
     `badgeManager.addBadgeMapping(E_FaultBadgeStateID.ActiveAlarm, ...)`).
   - 하위 메뉴 `Sub_Current_Alarm`/`Sub_Current_Event`는 각각 `fault_alarm`/`fault_event` 권한,
     History 하위(`Sub_Alarm_History` 등)는 `E_HistoryMenuID.Main_Menu_History` 부모로 등록.
5. **display define**: `src/app/home/home-provider.ts` `defineMainMenuDisplay()`의
   `tDefine.addItem(E_FaultMenuID.Fault_Current)` — Main_Menu 표시 순서에 포함.
6. **위젯**: `FaultContentWidgetProvider.initializeWidget()`이 `defineWidget(...)` 후
   `addWidget(E_DashboardTarget.Main_Dashboard, E_FaultWidgetID.CurrentAlarm)` 등으로 대시보드에 배치.

## 관련 문서
- `packs/nms-ems/knowledge/frontend/architecture.md` — 분석 순서(active configuration → menu exposure → route)
- `packs/nms-ems/knowledge/frontend/new-feature-flow.md` — 신규 기능 절차 전반
- `packs/nms-ems/knowledge/backend/feature-extension.md` — 권한/서비스 등 BE측 대응 확장 지점
