# NMS FE Architecture 이해 가이드

## 목적
NMS/EMS Angular frontend를 분석하거나 새 기능을 추가할 때 AI와 개발자가 같은 순서로 구조를 이해하도록 돕는다.

이 문서는 특정 화면 하나의 설명이 아니라, NMS FE에서 기능이 화면에 노출되고 동작하는 공통 흐름을 설명한다.

## 기본 원칙
- 먼저 현재 flow를 식별한 뒤 변경한다.
- generic source만 보지 말고 active site/product configuration을 확인한다.
- 기존 component/service separation을 유지한다.
- common component와 provider 구조를 먼저 확인한 뒤 새 UI를 추가한다.
- route, `baseHref`, serve-path, proxy assumption을 변경하지 않는다.

## 분석 순서
새 기능, 메뉴, 화면, 버그를 다룰 때 다음 순서로 확인한다.

1. Active configuration
   - `angular.json`의 build/serve configuration을 확인한다.
   - fileReplacement가 있으면 실제 런타임에서 쓰이는 파일을 먼저 읽는다.
   - site/product별 `sites/<site>/` 파일이 있으면 generic 파일보다 우선한다.
   - `six-gen-flexible` 작업이면 `packs/nms-ems/knowledge/frontend/six-gen-flexible-runtime.md`를 함께 읽는다.

2. Menu exposure
   - 메뉴 정의 파일을 확인한다.
   - provider import가 실제로 포함되어 있는지 확인한다.
   - `ContentsManagerService` 또는 유사 filtering layer를 확인한다.
   - `authorityCheck`나 role/permission 조건을 확인한다.
   - 메뉴가 보이지 않는다고 menu definition 누락으로 바로 단정하지 않는다.

3. Route and content module
   - route path와 content module 연결을 확인한다.
   - lazy loading 또는 provider 기반 화면 등록이 있는지 확인한다.
   - active site replacement가 route/content module을 바꾸는지 확인한다.

4. Component and service flow
   - component는 UI state와 event 처리 중심으로 본다.
   - service는 API 호출, data transform, shared state 중심으로 본다.
   - 기존 service가 있으면 새 API wrapper를 만들기 전에 재사용 가능성을 확인한다.
   - subscription, refresh, change detection timing을 변경할 때 영향 범위를 설명한다.

5. API contract
   - request/response field, pagination, error response, null/empty case를 확인한다.
   - backend contract가 불명확하면 UI에서 임의로 추정하지 않는다.
   - table column이나 form field 변경은 API contract와 매뉴얼 영향을 함께 본다.

6. Common UI component
   - table/list 화면은 `wc-data-display.md`를 먼저 확인한다.
   - form/filter/detail 화면은 `wc-form-controls.md`를 먼저 확인한다.
   - split panel, toolbar, dialog, tab은 `wc-layout.md`를 먼저 확인한다.
   - 실제 사용처를 찾을 때는 `wc-usage-map.md`를 함께 확인한다.
   - topology/shelfview 계열은 `topology-canvas.md`, `group-tree.md`, `detail-panel.md`를 함께 확인한다.

## NMS FE 화면 유형
| 유형 | 대표 화면 | 먼저 확인할 지식 |
| --- | --- | --- |
| Workspace/Canvas | Topology, ShelfView | `ui/topology-workspace.md`, `components/topology-canvas.md` |
| Table/List | Equipment, User, User Session | `components/wc-data-display.md` |
| Search + Table | Active Alarm, Alarm History, Performance | `components/wc-data-display.md`, `components/wc-form-controls.md` |
| Select Required + Nested Tree | NE Configuration > Configuration | `ui/configuration-workspace.md` |
| Preference/Form | Preference | `ui/preference.md`, `components/wc-form-controls.md` |

## 변경 전 질문
- 이 기능은 어떤 site/product configuration에서 활성인가?
- 실제 런타임 파일은 generic 파일인가, fileReplacement된 파일인가?
- 메뉴 노출은 menu definition, provider, filtering, authority 중 어디서 결정되는가?
- 기존 common component로 구현 가능한가?
- API contract와 backend 영향이 있는가?
- manual capture 또는 운영 가이드 갱신이 필요한가?

## 금지
- active configuration 확인 없이 generic 파일만 수정하지 않는다.
- 새 common component를 성급하게 만들지 않는다.
- route/baseHref/proxy 설정을 기능 작업과 함께 임의 변경하지 않는다.
- 테스트를 실행하지 않았으면 통과했다고 말하지 않는다.
