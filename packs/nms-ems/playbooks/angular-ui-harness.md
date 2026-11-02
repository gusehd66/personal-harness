# Angular UI 하네스 플레이북

## 목적
팀원이 AI에게 Angular 기반 NMS/EMS 프론트엔드를 분석시키고, UI 기능을 독립적으로 실행/검증/추가/수정/삭제할 수 있는 하네스를 설계하거나 갱신할 때 이 플레이북을 사용한다.

사용자가 범위를 명시적으로 좁히지 않으면 기본 대상은 단일 화면이 아니라 전체 제품 UI이다.

## 문서 언어 기준
- 하네스 문서, 분석 보고서, 템플릿, 스킬 지시문은 한글로 작성한다.
- Angular, TypeScript, route, provider, service, fixture, mock, read-write처럼 코드와 직접 연결되는 기술 용어는 필요할 때 영문 원어를 병기하거나 그대로 둔다.
- 파일명, 클래스명, 메서드명, 설정명, 경로는 원문 그대로 적는다.

## 작업 경계
- 분석 전용 작업이나 공용 하네스 갱신 작업에서는 제품 소스 체크아웃을 수정하지 않는다.
- six-gen 프론트엔드는 사용자가 해당 저장소 구현을 명시적으로 요청하지 않는 한 `<FRONTEND_ROOT>`를 읽기 전용 근거 소스로 취급한다.
- 공용 규칙, 프롬프트, 재사용 지침은 `<HARNESS_ROOT>` 아래에 작성한다.
- routing, provider, service 호출, build wiring에 대한 모든 판단은 실제 파일 경로를 함께 적는다.

## 표준 프롬프트
팀 전체가 반복 가능한 분석 결과를 원할 때 아래 프롬프트를 사용한다.

```text
<frontend-root>의 Angular NMS/EMS 프론트엔드 전체 UI를 분석하고 Angular 하네스 설계안을 작성해줘.

<frontend-root>는 수정하지 마.
먼저 실제 소스 구조를 읽어.
모든 판단은 실제 파일 경로를 근거로 남겨.
하네스 문서와 결과 보고서는 한글로 작성해.

분석할 항목:
1. 라우팅 구조
2. 주요 페이지 컴포넌트
3. 공통 UI 컴포넌트
4. API/service 구조
5. mock 처리 가능한 지점
6. 컴포넌트 단독 실행이 어려운 이유
7. 하네스로 분리 가능한 단위
8. 하네스 구현 시 영향받는 파일

설계할 항목:
1. harness route 구조
2. harness page 구조
3. fixture 데이터 구조
4. mock service/provider 구조
5. 기존 service와 mock service 전환 방식
6. fixture 데이터 관리 방식
7. 운영 빌드 영향 최소화 방안
8. 구현 순서

기본 전제:
- 전체 기능을 기본 대상으로 한다.
- 기능 추가/수정/삭제를 위한 read-write 흐름을 포함한다.
- 단, 사용자가 실제 backend write를 명시적으로 허용하지 않으면 모든 write 동작은 mock 기반으로 설계한다.
- fixture는 UI 상태, service 응답, write 이후 기대 상태를 재현하기 위한 고정 시나리오 데이터로 설명한다.
- 재사용 규칙은 제품 소스가 아니라 `<HARNESS_ROOT>`에 둔다.

원하는 결과:
- 분석 요약
- 추천 하네스 구조
- 파일 추가/수정 계획
- 위험 요소
- 검증 계획
- 구현 전에 확인 필요한 질문
```

## 필수 분석 순서
1. `package.json`과 `angular.json`을 읽는다.
2. 실제 활성 build/serve configuration과 `fileReplacements`를 확인한다.
3. 활성 `sites/<site>/app.routes.ts`, `contents.module.ts`, `product.ts`, `app.prepare.ts`를 읽는다.
4. `AppModule`, `APP_INITIALIZER`, auth guard, framework startup, root route 처리 흐름을 추적한다.
5. `ContentProviderModule`, `ContentsManagerService`, content provider filtering, menu 정의, authority gating, external route 등록을 추적한다.
6. 최상위 content route와 각 route 아래 lazy module을 열거한다.
7. 공통 UI module을 확인한다. 특히 `framework_modules/wccontrol/*`를 우선 확인한다.

관련 상세 문서:
- `packs/nms-ems/knowledge/frontend/architecture.md`
- `packs/nms-ems/knowledge/frontend/new-feature-flow.md`
- `packs/nms-ems/knowledge/frontend/six-gen-flexible-runtime.md`
- `packs/nms-ems/knowledge/components/wc-usage-map.md`
8. service 계열을 확인한다.
   - Angular `HttpClient`와 HTTP interceptor
   - `RequesterObject.sendRequestWait()`
   - `DBDataServiceBase`, `BaseDataService`, `BaseDataServiceSync`, feature service
   - Angular DI로 생성되는 service와 `new`로 직접 생성되는 service
9. mock 처리 가능한 지점과 직접 생성 때문에 mock 전환이 어려운 지점을 구분한다.
10. 운영 동작을 바꾸지 않는 하네스 설계를 작성한다.

## Six-Gen 프론트엔드 근거 확인 목록
프론트엔드 루트가 six-gen 계열이면 설계 전에 아래 경로를 확인한다.

- `<FRONTEND_ROOT>/package.json`
- `<FRONTEND_ROOT>/angular.json`
- `<FRONTEND_ROOT>/src/main.ts`
- `<FRONTEND_ROOT>/src/app/app.module.ts`
- `<FRONTEND_ROOT>/sites/six-gen-flexible/app.routes.ts`
- `<FRONTEND_ROOT>/sites/six-gen-flexible/contents.module.ts`
- `<FRONTEND_ROOT>/sites/six-gen-flexible/product.ts`
- `<FRONTEND_ROOT>/sites/six-gen-flexible/app.prepare.ts`
- `<FRONTEND_ROOT>/src/app/common/content/content-define/content-manager.service.ts`
- `<FRONTEND_ROOT>/src/app/common/content/content-define/content-define.ts`
- `<FRONTEND_ROOT>/src/app/app.http-interceptor.ts`
- `<FRONTEND_ROOT>/framework_modules/framework/core/request.ts`
- `<FRONTEND_ROOT>/framework_modules/framework/dbdata/dbdata-process.ts`

현재 알려진 six-gen routing 사실은 매번 실제 소스로 재확인한다.

- `six-gen-flexible-dev`는 `sites/six-gen-flexible/` 아래 활성 site 파일을 사용한다.
- 활성 `contents.module.ts`는 dashboard, equipment, NE configuration, fault, history, security, performance, preference, local user management, six-gen flexible, six-gen shelfview provider 등을 import한다.
- root navigation은 `HomeComponent`, `AuthGuard`, `AuthService`, `FrameworkService`, `ContentsManagerService.defaultMainUrl` 흐름을 거친다.
- 메뉴는 코드에 있어도 `ContentProvider.getEnabled()`, `useContentAuthority`, `checkAuthority()`, site별 provider wiring 때문에 화면에서 사라질 수 있다.

## Fixture 정의
fixture는 live NMS backend에 의존하지 않고 UI 상태와 service 응답을 재현하기 위해 하네스가 사용하는 고정 시나리오 데이터이다.

fixture는 단순 sample JSON이 아니다. 하네스 scenario와 mock service 사이의 계약이다.

예시:
- `ne.fixture.ts`: site, group, NE 목록, 선택된 NE, 권한상 노출되는 NE 집합
- `shelfview.fixture.ts`: COT/RT shelf, slot, unit, port, alarm/equip/admin status
- `alarm.fixture.ts`: active alarm, history alarm, severity, 시간 범위, ack 상태
- `configuration.fixture.ts`: port status, port statistics, unit inventory, module inventory, SyncE, IEEE 1588v2
- `user.fixture.ts`: role, local user, 중복 account/email case, session 목록
- `write-result.fixture.ts`: 성공, validation 실패, timeout, retry, partial failure, permission denied

read-write 하네스는 초기 조회 상태와 write 이후 기대 상태를 모두 fixture로 정의해야 한다.

## 추천 하네스 구조
별도 Angular harness entrypoint와 serve/build configuration을 우선 사용한다.

```text
src/app/harness/
  main.harness.ts
  harness-app.component.ts
  harness.routes.ts
  harness.module.ts
  pages/
    harness-index.page.ts
    route-preview.page.ts
    feature-preview.page.ts
    write-action-preview.page.ts
  registry/
    harness-feature-registry.ts
    harness-scenario-registry.ts
  fixtures/
    six-gen/
      ne.fixture.ts
      shelfview.fixture.ts
      alarm.fixture.ts
      configuration.fixture.ts
      mef-service.fixture.ts
      user.fixture.ts
      write-result.fixture.ts
  mocks/
    mock-auth.service.ts
    mock-system-environment.service.ts
    mock-content-manager.service.ts
    mock-ne.service.ts
    mock-ne-group.service.ts
    mock-site.service.ts
    mock-alarm.service.ts
    mock-shelfview.service.ts
    mock-mef-service.service.ts
    mock-http.interceptor.ts
    mock-requester.ts

sites/six-gen-harness/
  index.html
  product.ts
  app.routes.ts
  contents.module.ts
  app.prepare.ts
  tsconfig.app.json
```

## 하네스 라우트 모델
모든 AI agent가 같은 구조를 만들 수 있도록 route 구조를 고정한다.

```text
/unms/harness
/unms/harness/dashboard
/unms/harness/equipment/equipment
/unms/harness/equipment/topology
/unms/harness/equipment/shelfview
/unms/harness/ne-configuration/configuration
/unms/harness/ne-configuration/mef-service-management
/unms/harness/fault/active-alarm
/unms/harness/history/alarm-history
/unms/harness/history/performance
/unms/harness/security/user
/unms/harness/security/role
/unms/harness/security/user-session
/unms/harness/preference
/unms/harness/write-actions
```

## 읽기/쓰기 안전 규칙
- 기본 write mode는 `mock-write`이다.
- 실제 backend write mode는 명시적 opt-in일 때만 허용하고 결과 보고서에 눈에 띄게 표시한다.
- mock write scenario는 성공, validation error, permission denied, timeout, retry, stale refresh를 포함한다.
- write 버튼은 scenario가 expected post-write fixture를 선언한 경우에만 하네스에서 활성화한다.
- 하네스는 read path, write path, request shape, response shape, refresh path를 기록해야 한다.
- 기본 하네스에서는 실제 `Save`, `Apply`, `Delete`, `Discovery`, `Command`, `Initialize`, `Upgrade`, sync, scheduler API를 호출하지 않는다.

## 서비스 전환 규칙
가능하면 Angular DI를 사용한다.

권장 순서:
1. harness module provider에서 feature service를 override한다.
2. auth/session/system-environment service를 override한다.
3. `HttpClient` 호출은 HTTP interceptor로 override한다.
4. framework request sending은 harness entrypoint 내부에서만 감싸거나 patch한다.
5. `new Service()` 직접 생성 코드는 사용자가 구현을 명시적으로 요청한 경우에만 좁게 refactor한다.

production service를 shared module에서 전역으로 바꾸지 않는다. 전환 지점은 harness-only module 또는 harness-only file replacement 안에 둔다.

## 결과물 계약
모든 하네스 분석 또는 구현 계획은 아래 항목을 포함한다.

- 분석한 source root
- 활성 Angular configuration
- file replacement 표
- route tree 요약
- content provider/menu/authority 흐름
- 주요 feature module
- 공통 UI module 목록
- service/mock map
- fixture map
- read-write scenario map
- production 영향 설명
- 정확한 파일 추가/수정 계획
- 검증 명령
- 미확인 질문

## 검증 기준
문서 전용 하네스 갱신:
- 변경 파일을 다시 읽는다.
- 제품 build 또는 test가 통과했다고 말하지 않는다.

향후 제품 하네스 구현:
- 해당 repo의 실제 Angular launcher를 사용한다.
- 이 Windows 환경에서 PowerShell이 `npm`을 막으면 `C:/nodejs/npm.cmd`를 우선 사용한다.
- compile 성공 후 browser에서 harness route를 확인한다.
- production build configuration이 의도치 않게 바뀌지 않았는지 확인한다.
