# NMS FE Site별 빌드 / fileReplacement 체계

## 목적
`<FRONTEND_ROOT>` (uems_fe) 하나의 코드베이스에서 여러 제품(site)을 빌드하는 구조를 설명한다.
어떤 site가 있고, 각 site가 어떤 파일을 교체(fileReplacement)하며, 신규 site 추가와 기능 추가 시 어디를 봐야 하는지를 표준화한다.

이 문서는 site 체계 전반을 다룬다. 특정 site 심층 분석은 `six-gen-flexible-runtime.md`(six-gen-flexible 전용)를 본다.

## 핵심 결론
- 하나의 Angular 프로젝트(`uems-fe`)를 `angular.json`의 build configuration으로 갈라 site별 산출물을 만든다.
- site별 차이는 `sites/<site>/` 아래 5~6개 파일을 fileReplacement로 교체해서 만든다. 나머지 소스는 전부 공유한다.
- generic `src/app/content/contents/contents.module.ts`는 빈 placeholder다(`CONTENTS_ROUTES = []`, imports 없음). 실제 메뉴/route 구성은 항상 `sites/<site>/contents.module.ts`에 있다. 따라서 **어떤 site도 지정하지 않은 순수 `ng build`는 콘텐츠가 없는 앱을 만든다.**
- 기능 추가/수정 전에 반드시 "지금 어느 site configuration이 active인가"를 먼저 확정한다.

## Site 목록 (angular.json 실측)
현재 checkout(branch `projects/unms_develop_7.0`) 기준, `angular.json`의 `uems-fe > architect > build > configurations`에 등록된 site성 configuration과 `sites/` 디렉터리는 다음과 같다.

| site | build config (prod성) | dev config | baseHref | outputPath | tsConfig | style 특이사항 |
| --- | --- | --- | --- | --- | --- | --- |
| default | `default` | `default-dev` | `/unms/` | `dist/unms` | `src/tsconfig.default.json` | 기본 `style-30.scss` |
| noc | `noc` | `noc-dev` | `/unms/` | `dist/noc` | `src/tsconfig.default.json` | 기본 |
| mobile | `mobile` | `mobile-dev` | `/m-unms/` | `dist/m-unms` | `src/tsconfig.default.json` | `style-mobile.scss` |
| das | `das` | `das-dev` | `/` | `dist/das` | `src/tsconfig.default.json` | 기본 |
| hiems | `hiems` | `hiems-dev` | `/hiems/` | `dist/hiems` | `src/tsconfig.default.json` | 기본 |
| user-mgmt | `user-mgmt` | `user-mgmt-dev` | `/usermgmt/` | `dist/user-mgmt` | `src/tsconfig.prod.json` | 기본 |
| customer-mgmt | `customer-mgmt` | `customer-mgmt-dev` | `/customermgmt/` | `dist/customer-mgmt` | `src/tsconfig.prod.json` | 기본 |
| noc-user-mgmt | `noc-user-mgmt` | `noc-user-mgmt-dev` | `/usermgmt/` | `dist/noc-user-mgmt` | `src/tsconfig.prod.json` | 기본 |
| system-mgmt | `system-mgmt` | `system-mgmt-dev` | `/systemmgmt/` | `dist/system-mgmt` | `src/tsconfig.prod.json` | 기본 |
| mobile-cpe | `mobile-cpe` | `mobile-cpe-dev` | `/unms/` | `dist/mobile-cpe` | `src/tsconfig.default.json` | `style-cpe.scss` |
| openHaul | `openHaul` | `openHaul-dev` | `/` | `dist/openHaul` | `src/tsconfig.default.json` | 기본 |

site성이 아닌 configuration:
- `production`: environment만 `environment.prod.ts`로 교체. site 파일 교체 없음(= generic placeholder 그대로).
- `development`: 최적화 옵션만 끄는 dev 플래그 묶음.

주의:
- `six-gen-flexible`은 **이 checkout의 `angular.json`/`sites/`에 존재하지 않는다** (`grep six-gen-flexible angular.json package.json` 결과 없음). 해당 site는 다른 branch/checkout에 있으며, `six-gen-flexible-runtime.md`가 그 checkout 기준으로 작성되었다. six-gen-flexible 작업 전에는 먼저 checkout branch를 확인한다.
- `sites/` 디렉터리 실측: `customer-mgmt`, `das`, `default`, `hiems`, `mobile`, `mobile-cpe`, `noc`, `noc-user-mgmt`, `openHaul`, `system-mgmt`, `user-mgmt` (11개, angular.json과 1:1 대응).

## fileReplacement 표준 세트
모든 site configuration(prod성/dev성 공통)은 다음을 교체한다.

| replace (generic) | with (site) | 역할 |
| --- | --- | --- |
| `src/app/app.prepare.ts` | `sites/<site>/app.prepare.ts` | 인증 provider, start sequence, base data loader 등록 |
| `src/app/content/contents/contents.module.ts` | `sites/<site>/contents.module.ts` | `CONTENTS_ROUTES`와 content provider module import(= 메뉴 구성) |
| `src/app/common/common/product.ts` | `sites/<site>/product.ts` | 제품명/버전/keycloak client/`ProductConfig` 플래그 |
| `src/tsconfig.app.json` | `sites/<site>/tsconfig.app.json` | 컴파일 대상/제외 조정 |
| `src/app/app.routes.ts` | `sites/<site>/app.routes.ts` | 최상위 route (error page, login, single-page 진입 등) |
| (index option) `src/index.html` | `sites/<site>/index.html` | site별 index |
| (prod성 config만) `src/environments/environment.ts` | `src/environments/environment.prod.ts` | 환경 |

예외 (실측):
- `customer-mgmt` / `customer-mgmt-dev`는 `app.routes.ts` 교체가 없고, `sites/customer-mgmt/`에도 `app.routes.ts` 파일이 없다. generic `src/app/app.routes.ts`를 그대로 쓴다.
- `mobile-cpe`는 site 디렉터리에 `proxy.config.json`을 추가로 가지며 serve configuration이 `sites/mobile-cpe/proxy.config.json`을 쓴다.
- `mobile` / `mobile-cpe`는 styles 배열 자체를 교체한다(`style-mobile.scss` / `style-cpe.scss`).

## Serve / 실행 경로 (실측)
- `angular.json` serve의 `defaultConfiguration`은 `default-dev`다. 즉 `npm run serve`(= `ng serve --serve-path unms/`, `-c` 없음)는 **default site**로 뜬다.
- proxy: 기본 `proxy.config.json`, `noc`은 `proxy.config.noc.json`, `mobile-cpe`는 `sites/mobile-cpe/proxy.config.json`.
- port: 기본 4200, `noc` 4201, `user-mgmt`/`noc-user-mgmt` 4300, `system-mgmt` 4400. (`sites/default/product.ts`의 debug용 `userManagementUrl=localhost:4300`, `systemManagementUrl=localhost:4400`과 대응)
- `package.json` script 실측: `serve`, `serve:user-mgmt`(`--serve-path usermgmt/`), `serve:noc`, `serve:noc-user-mgmt`, `build`, `build-prod`. 그 외 site는 `ng serve -c <site>`를 직접 쓴다.

## Site 3종 비교: 교체 파일이 실제로 바꾸는 것
`default`(generic 본체) / `noc` / `openHaul` 실측 비교. 다른 site를 분석할 때도 같은 축으로 보면 된다.

### app.prepare.ts — 인증과 초기 데이터
공통 뼈대: `AppPrepareModuleBase`(`src/app/app.common.ts`)를 상속하고 `getAuthenticationProvider()` / `registerStartSequence()` / `registerBaseDataLoader()`를 구현. base class는 `registerExcludeMenu()` hook도 제공한다.

| 항목 | default | noc | openHaul |
| --- | --- | --- | --- |
| 인증 | `KeycloakAuthenticationProvider` + `SSOLoginService` provider | default와 동일 | `LocalAuthenticationProvider` (SSO 없음) |
| start sequence | `CommunicationProcess` → `LoginStartSequence` → `BaseDataLoadProcess` → `ContentServicePrepareProcess` | 동일 | 동일 |
| base data loader | Ne, NeGroup, Em, Site, Topology, TopologyItem, TargetRole | default와 동일 | Ne, NeGroup, Em, Site, **NeTopology** (Topology/TopologyItem/TargetRole 없음) |
| exclude menu | 미구현(base 기본) | 미구현 | `registerExcludeMenu()` 빈 override |

### contents.module.ts — 메뉴/route 구성 (site 차이의 본체)
`CONTENTS_ROUTES`(lazy route 목록)와 `@NgModule.imports`의 `*ContentProviderModule` 목록이 그 site의 메뉴 전부를 결정한다.

| provider module | default | noc | openHaul |
| --- | --- | --- | --- |
| Dashboard/Equipment/NeConfiguration/Fault/Security/Operation/Preference | O | O | O |
| History, Performance | O | O | 주석 처리(제외) |
| SecurityKeycloak | O | O | X |
| Cluster, ServiceConfig, My5G, KhnpConfig, Wifi | O | O | X |
| NOCClient | O | X | X |
| Gpon, NOC, WorkMgmt | X | O (noc 전용) | X |
| openHaul, LocalUserMgmt | X | X | O (openHaul 전용) |

- provider module import와 `CONTENTS_ROUTES` entry는 쌍으로 관리된다. 한쪽만 넣으면 메뉴는 뜨는데 route가 없거나 그 반대가 된다.
- openHaul은 keycloak 기반 Security 화면 대신 `LocalUserMgmtContentProviderModule`(`security-local-user`)을 쓴다 — 인증 방식(local)과 일관.

### product.ts — 제품 정체성과 ProductConfig 플래그
| 항목 | default | noc | openHaul |
| --- | --- | --- | --- |
| productName / title | `uNMS` | `C-NOC` / `Network Operation Center` | `openHaul EMS` |
| version | `6.0.2` | `0.9.0` | `1.0.0` |
| keycloak | `keycloakClientID='uEMS'`, redirect/usermgmt/hiems/systemmgmt URL 구성 | `keycloakClientID='NOC'`, `isNOC=true`, `isShowLocalVersion=true` | keycloak 설정 없음 (local auth) |
| ProductConfig 분기 플래그 | - | `isNOC` | `isopenHaul`, `isShowLogo=false`, 로그인 페이지 문구 교체 |

`ProductConfig`의 `isNOC`/`isopenHaul` 같은 플래그는 공유 코드 안의 site 분기에 쓰이므로, "이 화면이 왜 이 site에서만 다르지?"를 조사할 때 grep 대상이다.

### app.routes.ts — 최상위 route
- 뼈대(Home + `CONTENTS_ROUTES` children, login, error pages, `development`, `temp`)는 세 site 모두 동일.
- `default`는 single-page 진입 route 2개가 추가: `EquipmentContentCommon.neTerminalSinglePageUrl`, `EquipmentContentCommon.neConfigSinglePageUrl`. openHaul에는 없다.
- generic `src/app/app.routes.ts`에도 이 두 route가 없다 — 즉 route 차이는 반드시 `sites/<site>/app.routes.ts` 쪽을 봐야 한다.

### tsconfig.app.json
- `sites/default`와 `sites/openHaul`은 동일.
- generic `src/tsconfig.app.json` 대비: `"types": ["node"]` 활성, `"./app/app-customer"` exclude 추가. site별 코드 제외가 필요할 때 이 파일을 쓴다.

## 신규 site 추가 절차 (체크리스트)
가장 비슷한 기존 site를 골라 복제하는 것이 기본이다. 아래 순서로 진행한다.

1. `sites/<new-site>/` 디렉터리 생성 — 기존 site(예: openHaul) 복제 후 수정:
   - `app.prepare.ts`: 인증 provider(Keycloak vs Local), base data loader 목록 결정.
   - `contents.module.ts`: 노출할 provider module import + `CONTENTS_ROUTES` entry만 남기고 정리.
   - `product.ts`: `productName`/`productTitle`/`version`/`productUrl`, keycloak client ID 또는 local auth, 필요 시 `ProductConfig` 플래그 추가.
   - `app.routes.ts`: 최상위 route (single-page 진입 필요 여부 확인). 불필요하면 customer-mgmt처럼 생략 가능하나, 생략 시 angular.json에서도 해당 replacement를 빼야 한다.
   - `tsconfig.app.json`, `index.html`.
2. `angular.json` build configurations에 2개 등록:
   - `<new-site>` (prod성): `baseHref`, `outputPath: dist/<new-site>`, environment 포함 fileReplacements 6종(또는 5종), `index.input`, `assets`(기존 site 패턴 복사), `tsConfig`(`src/tsconfig.default.json` 또는 `src/tsconfig.prod.json`), 최적화 옵션.
   - `<new-site>-dev`: environment 교체 없이 site 파일 교체만 + sourceMap/optimization dev 플래그 + `styles`에 `src/app/temp/temp.component.scss` 포함(기존 dev config 패턴).
3. `angular.json` serve configurations에 `<new-site>` 추가: `buildTarget: uems-fe:build:<new-site>-dev`, 필요 시 전용 `port`/`proxyConfig`.
4. (선택) `package.json`에 `serve:<new-site>` script 추가 — `--serve-path`는 `baseHref`와 맞춘다.
5. site 전용 스타일이 필요하면 `styles` 배열 교체(mobile/mobile-cpe 패턴), 전용 proxy가 필요하면 `sites/<new-site>/proxy.config.json`(mobile-cpe 패턴).
6. 검증: `ng serve -c <new-site>`로 로그인 → 메뉴 노출 → 대표 화면 진입 확인. `ng build -c <new-site>` 성공 확인.

## 기능 추가 시 site 영향 체크
기능을 넣기 전에 "어느 site에 노출할 것인가"를 먼저 답한다.

1. **전 site 공통 로직/버그 수정** → 공유 소스(`src/app/**`, `framework_modules/**`)를 수정한다. fileReplacement 대상 5종 파일의 generic 버전(`src/app/app.prepare.ts` 등)을 고치는 것은 **효과가 없다** — 모든 site config가 그 파일을 교체하기 때문이다.
2. **새 메뉴/화면 (일부 site에만 노출)** → 화면 구현은 공유 소스(`src/app/content/contents/<domain>/`)에 두고, 노출은 site별로 제어한다:
   - 노출할 각 site의 `sites/<site>/contents.module.ts`에 provider module import + `CONTENTS_ROUTES` entry를 추가한다.
   - 넣지 않은 site에는 자동으로 노출되지 않는다. "generic에 추가하면 모든 site에 뜬다"가 아니라, **site contents.module에 넣은 곳에만 뜬다.**
3. **기존 provider module 아래 sub menu 추가** → provider module 자체는 공유 코드이므로, 그 provider를 import하는 **모든 site**에 노출된다. 실측 예: `EquipmentContentProviderModule`은 default/noc/openHaul 전부에 있으므로 equipment sub menu 추가는 3개 site 이상에 영향. 특정 site만 제외하려면 `registerExcludeMenu(...)`(app.prepare) 또는 authority/`ProductConfig` 플래그 분기를 쓴다.
4. **site 한정 동작 분기** → 우선순위: (a) site 전용 provider module 분리(예: `NOCContentProviderModule`, `openHaulContentProviderModule`) > (b) `ProductConfig` 플래그(`isNOC`, `isopenHaul`) 분기 > (c) 공유 코드 안 하드코딩 분기(지양).
5. **초기 데이터/인증 영향** → `sites/<site>/app.prepare.ts`의 base data loader 차이를 확인한다. 예: Topology 데이터에 의존하는 기능은 `TopologyService`를 로드하지 않는 openHaul에서 깨질 수 있다.
6. **작업 후 확인 목록**:
   - 수정한 파일이 fileReplacement 대상인가? 대상이면 어느 site 버전을 고쳤는가?
   - 노출 대상 site 각각의 contents.module에 반영했는가?
   - 노출하면 안 되는 site에서 메뉴가 안 뜨는 것을 확인했는가?
   - backend 기능 확장이 동반되면 `packs/nms-ems/knowledge/backend/feature-extension.md`의 서비스 아키타입 절차를 함께 따른다.

## 메뉴가 안 보일 때 (site 관점 진단 순서)
1. 현재 실행 configuration 확인 (`ng serve -c ?`, `npm run serve`는 default-dev).
2. 그 site의 `sites/<site>/contents.module.ts`에 provider module import가 있는지 확인.
3. `CONTENTS_ROUTES`에 route entry가 있는지 확인.
4. `sites/<site>/app.prepare.ts`의 `registerExcludeMenu(...)` 확인.
5. provider `getEnabled()`/authority, `ContentsManagerService` filtering 확인 — 이후는 `architecture.md`의 Menu exposure 절차를 따른다.

## 관련 문서
- `packs/nms-ems/knowledge/frontend/architecture.md` — 분석 순서 전반 (active configuration 확인이 1순위인 이유).
- `packs/nms-ems/knowledge/frontend/new-feature-flow.md` — 신규 기능 절차 (2단계 "site-specific build" 확인이 이 문서로 연결).
- `packs/nms-ems/knowledge/frontend/six-gen-flexible-runtime.md` — six-gen-flexible site 심층(런타임/provider/NE Configuration flow). 단, 그 site는 별도 branch checkout에만 존재한다.
- `packs/nms-ems/knowledge/backend/feature-extension.md` — BE 쪽 기능 확장 아키타입. FE site 노출과 BE 서비스 확장이 한 기능에서 같이 움직일 때 상호 참조.
