# Frontend Feature Playbook

## 사용 시점
- NMS/EMS FE에 새 메뉴, 새 화면, 새 table/form 기능을 추가할 때 사용한다.
- 기존 화면의 flow를 이해한 뒤 최소 변경으로 확장해야 할 때 사용한다.

## 목표
팀원이 같은 순서로 frontend flow를 읽고, 기존 common component와 NMS 지식을 재사용해 일관된 결과물을 만들도록 한다.

## 절차
1. Configuration 확인
   - `angular.json` build/serve configuration 확인
   - fileReplacement 확인
   - active site/product 파일 확인

2. 기존 유사 화면 찾기
   - 메뉴 유형과 UI 유형을 분류한다.
   - 기존 route/module/provider/component/service/API 흐름을 추적한다.
   - 유사 화면이 있으면 그 pattern을 우선 따른다.

3. 메뉴 노출 경로 확인
   - menu definition
   - provider import
   - `ContentsManagerService` filtering
   - `authorityCheck`/role/permission

4. 공통 UI 컴포넌트 채택 gate
   - 사진·목업은 시각 기준으로만 사용하고, 실제 구현 계약은 현재 프로젝트의 component source와 기존 consumer로 확인한다.
   - 화면 유형별 기존 유사 화면, `wc-*` selector/API, import 방식, 상태·이벤트·권한 처리를 먼저 찾는다.
   - 대응 WC가 확인되면 화면 레벨에서 native HTML 또는 별도 custom component로 같은 역할을 재구현하지 않는다.
   - native HTML은 WC 내부 구현, 기존 화면의 의도된 native 사용, 또는 대응 WC 부재가 source 근거로 확인된 경우에만 사용한다.

5. UI 구현 기준 선택
   - 공통 컴포넌트 탐색 시작점: `packs/nms-ems/knowledge/components/wc-usage-map.md`
   - table/list: `packs/nms-ems/knowledge/components/wc-data-display.md`
   - form/filter/detail: `packs/nms-ems/knowledge/components/wc-form-controls.md`
   - layout/dialog/toolbar: `packs/nms-ems/knowledge/components/wc-layout.md`
   - topology/shelfview: `packs/nms-ems/knowledge/components/topology-canvas.md`

6. Data flow 확인
   - initial load
   - search/filter
   - selection change
   - refresh/reload
   - create/update/delete/command action

7. 위험 보고
   - route/baseHref/proxy 영향
   - 권한/세션 영향
   - API request/response 영향
   - null/empty/error case
   - manual update/capture 필요 여부

8. 검증
   - build/test/manual verification 중 실제 수행한 것만 기록한다.
   - 수행하지 않은 항목은 `Not verified`에 남긴다.

## 결과 보고 추가 항목
기본 result-report 외에 다음을 포함한다.

- UI pattern:
- Flow traced:
- Common components used:
- WC evidence (source/consumer):
- Native markup exception (없음 또는 근거):
- Manual update needed:
- Skill candidate:
