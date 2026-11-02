# Angular 하네스 지식

## 목적
이 문서는 NMS/EMS UI 작업을 위한 Angular 하네스를 어떻게 이해하고 설계할지 설명한다.

하네스의 목적은 단순 화면 미리보기가 아니다. live backend에 기본 의존하지 않고 반복 가능한 분석, UI 검증, 기능 추가, 기능 수정, 기능 삭제, 읽기/쓰기 workflow 예행 검증을 지원해야 한다.

## 기본 범위
기본적으로 전체 제품 UI를 분석한다.

six-gen 프론트엔드 작업의 기준 루트는 아래 placeholder로 표기한다.

- `<FRONTEND_ROOT>`

공용 하네스만 갱신하는 작업에서는 해당 source tree를 수정하지 않는다.

## 하네스가 필요한 이유
NMS 프론트엔드는 보통 page가 아래 요소와 강하게 결합되어 있어 컴포넌트 단독 실행이 어렵다.

- Angular site configuration과 `fileReplacements`
- `AppModule`, `APP_INITIALIZER`, auth guard, framework startup
- `ContentProviderModule` 등록
- `ContentsManagerService.initialize()` provider filtering
- menu 정의, menu instance 생성, authority check
- base data loader와 request service
- 공통 `wc-*` UI module
- NMS, NOC, consigned NMS 같은 실행 시점 product/site state

관련 상세 문서:
- `packs/nms-ems/knowledge/frontend/architecture.md`
- `packs/nms-ems/knowledge/frontend/new-feature-flow.md`
- `packs/nms-ems/knowledge/frontend/six-gen-flexible-runtime.md`
- `packs/nms-ems/knowledge/components/wc-usage-map.md`

따라서 유용한 하네스는 컴포넌트 template만 분리해서는 부족하다. route, provider, auth, base data, service, fixture context까지 함께 제공해야 한다.

## Fixture 의미
fixture는 고정된 test scenario data이다.

fixture는 아래 질문에 답한다.

- UI가 어떤 상태에서 시작하는가?
- 각 service 호출은 무엇을 반환하는가?
- write action 이후 어떤 상태가 되어야 하는가?
- 어떤 오류, timeout, retry, permission case를 보여줘야 하는가?

fixture가 있으면 팀원과 AI agent가 같은 prompt에서 같은 UI 상태를 재현할 수 있다.

fixture group 예시:

- NE/site/group fixture
- shelf/slot/port fixture
- alarm/event fixture
- PM fixture
- sync state fixture
- user/role/session fixture
- configuration command fixture
- write-result fixture

## 하네스 분리 단위
전체 제품 하네스를 나눌 때 아래 단위를 사용한다.

- shell harness: app bootstrap, layout, route list, scenario picker
- route harness: 하나의 route와 필요한 provider, fixture binding
- feature harness: ShelfView, User, Role, Active Alarm, Alarm History, Configuration, MEF Service 같은 하나의 업무 기능
- component harness: 하나의 재사용 UI 컴포넌트와 최소 service state
- service harness: mock 기반 service contract와 request/response recorder
- write harness: mock 기반 add, edit, delete, apply, command, discovery, initialize, upgrade, sync workflow

## 읽기/쓰기 하네스 규칙
향후 AI 기반 기능 추가/수정/삭제 작업을 위해 읽기/쓰기 지원은 필수이다.

기본 mode는 mock 기반 읽기/쓰기이다. 실제 backend write mode는 opt-in이어야 한다.

각 write scenario는 아래 항목을 정의해야 한다.

- initial fixture
- user action
- expected request shape
- mock response
- expected refreshed fixture
- UI result
- 오류 처리 경로

하네스는 화면 표시만 바꾸는 변경과 실제 상태 변경 동작을 분리해야 한다.

## Six-Gen 전용 메모
six-gen 프론트엔드는 구현 전에 항상 live source file을 다시 확인한다.

- `angular.json`의 `six-gen-flexible`, `six-gen-flexible-dev`
- `sites/six-gen-flexible/app.routes.ts`
- `sites/six-gen-flexible/contents.module.ts`
- `src/app/app.module.ts`
- `src/app/common/content/content-define/content-manager.service.ts`
- `src/app/common/content/content-define/content-define.ts`
- `src/app/app.http-interceptor.ts`
- `framework_modules/framework/core/request.ts`
- `framework_modules/framework/dbdata/dbdata-process.ts`

현재 알려진 패턴:

- site별 `contents.module.ts`가 six-gen의 실제 content wiring이다.
- menu는 provider 기반이며 authority filtering을 거친다.
- menu가 보이지 않는 문제는 file replacement, provider import, `getEnabled()`, `useContentAuthority`, `checkAuthority()` 순서로 추적한다.
- 일부 request는 Angular `HttpClient`를 사용하지만, 많은 request는 `RequesterObject.sendRequestWait()`를 사용한다.

## 운영 영향 최소 기준
harness 동작은 production build 밖에 둔다.

- harness-only Angular configuration을 추가한다.
- harness-only entrypoint를 추가한다.
- site replacement가 필요하면 harness-only site file을 추가한다.
- harness-only mock provider를 추가한다.
- 좁은 dependency-injection seam이 필요한 경우가 아니면 shared production provider를 변경하지 않는다.
- 좁은 refactor가 필요하면 기존 결합이 왜 harness 분리를 막았는지 문서화한다.
