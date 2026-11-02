# NMS FE 신규 기능 Flow

## 목적
새 메뉴, 새 화면, 새 form/table 기능을 추가할 때 필요한 확인 순서와 산출물을 표준화한다.

## 시작 전 읽기
- `core/policy/global.md`
- `core/policy/security.md`
- `core/policy/approval.md`
- `packs/nms-ems/knowledge/frontend/architecture.md`
- `packs/nms-ems/knowledge/frontend/feature-extension.md` — 이 문서가 "어떤 순서로 하는가"라면, feature-extension은 "어디에 끼우는가"(seam)를 정의한다
- `packs/nms-ems/knowledge/ui/layout.md`
- `packs/nms-ems/knowledge/components/wc-components.md`
- `packs/nms-ems/knowledge/components/wc-usage-map.md`
- NE Configuration 장비 조회/설정 작업이면 `packs/nms-ems/knowledge/domain/ne-configuration-command.md`
- 작업 유형에 맞는 domain/feature/ui/component 문서

## 신규 기능 절차
1. 기능 범위 정의
   - 사용자가 보는 메뉴명과 화면명을 분리한다.
   - read path와 write path를 나눈다.
   - 조회 기능인지 설정/명령/저장 기능인지 구분한다.

2. 기존 flow 찾기
   - 비슷한 기존 화면을 찾는다.
   - route, module, provider, component, service, API 호출 경로를 추적한다.
   - table/form/dialog/topology 중 어떤 UI pattern인지 분류한다.
   - site-specific build라면 `angular.json` fileReplacement와 `sites/<site>/` active 파일을 먼저 확인한다. site 목록과 교체 체계는 `packs/nms-ems/knowledge/frontend/site-specific-build.md`를 본다.

3. 메뉴와 권한 연결
   - active site의 menu/content module을 확인한다.
   - provider import 여부를 확인한다.
   - `ContentsManagerService` filtering과 `authorityCheck`를 확인한다.
   - role/permission 변경이 필요한 경우 security 영향으로 보고한다.
   - active site의 `sites/<site>/contents.module.ts`와 `sites/<site>/app.prepare.ts`를 우선한다. generic `src/app/content/contents/contents.module.ts`는 빈 stub이며 빌드 시 site 파일로 통째로 교체된다.
   - `six-gen-flexible` site는 현재 unms_develop_7.0 checkout의 `angular.json`/`sites/`에 존재하지 않는다(별도 branch 전용). 해당 site 작업이면 checkout branch를 먼저 확인하고 `six-gen-flexible-runtime.md`를 본다.

4. UI 구성
   - table은 기존 `wc-table` pattern을 우선 사용한다.
   - form은 기존 `wc-form`/validation pattern을 우선 사용한다.
   - toolbar/action은 기존 command/action pattern을 우선 사용한다.
   - topology/shelfview는 기존 canvas/tree/detail panel 구조를 유지한다.
   - NE Configuration 장비 설정 화면은 `ConfigurationPanel` 계열, `getCommands(...)`, `display(...)`, `NeCommandDefineArray` 흐름을 우선 확인한다.

5. Service/API 연결
   - 기존 service와 API convention을 먼저 확인한다.
   - null, empty, timeout, error response 처리를 명시한다.
   - pagination/search 조건이 있으면 request/response 영향을 설명한다.

6. 상태 갱신
   - 화면 진입 시 load path를 확인한다.
   - search, refresh, tab change, selection change의 data reload path를 구분한다.
   - RxJS 복잡도를 불필요하게 늘리지 않는다.
   - async refresh timing과 change detection 영향을 설명한다.

7. 검증
   - 변경 범위에 맞는 build/test/manual verification을 수행한다.
   - 실행하지 않은 검증은 `Not verified`에 명시한다.
   - UI 변경이면 manual capture 갱신 필요 여부를 판단한다.

## 산출물 체크리스트
- 메뉴/route/provider 연결 설명
- component/service/API flow 설명
- 사용한 common component 설명
- read/write path 구분
- 장비 command 사용 여부와 Command class 영향 범위
- 권한/세션 영향 여부
- manual update 필요 여부
- verification/not verified 기록

## Skill 후보
다음 작업이 반복되면 별도 skill로 승격한다.

- 새 table/list 화면 추가
- 새 form/dialog 추가
- 새 menu/provider 연결
- manual capture 자동 갱신
- API contract 변경 검토
- alarm/PM/sync/equipment domain-sensitive 변경 검토
