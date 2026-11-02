# Six-Gen React Equipment Parity

## 범위

- Angular 기준: `C:/khd/6gworkspace/UnifyEMS/uems_fe`
- React 대상: `D:/sixgen-react`
- 대상 화면: `/equipment/equipment`
- 화면 기준: 데스크톱 `1440x960`; 모바일은 현재 범위 밖이다.

## Angular 기준 소스

- 화면/toolbar/action: `src/app/content/contents/equipment/content/equipment/equipment.component.ts`
- table/tree/state: `src/app/content/common/equipment/group-ne-table/group-ne-table.component.ts`
- NE editor: `src/app/content/contents/equipment/content/equipment/ne-editor/ne-editor.component.ts`
- six-gen MCU slot lookup: `src/app/content/common/six-gen-flexible/six-gen-flexible-common.ts`
- IP/IP Range: `src/app/content/common/equipment/ne-editor/ne-editor-ip.service.ts`
- NE Change: `src/app/content/contents/equipment/content/equipment/ne-change/`
- Recovery API: `src/app/content/common/recovery/recovery.service.ts`
- Backup API: `src/app/dbdata/unify_nebackup/nebackup.service.ts`

## 유지할 요청 계약

- Group/NE CRUD: `basedata/v1/ne-group`, `basedata/v1/ne`
- 단일 IP/수동 검색: `nediscovery-mgmt/v1/nediscovery-ip`, `nediscovery-refresh`
- IP Range: `nediscovery-ip-range` 검색 후 선택 후보를 `nediscovery-ip-range-add`로 등록한다.
- NE Change backup: `nebackuprestore-mgmt/v1/config-backup-file`
- NE Change process: `recovery/v1/change-ne`에 `restore`, `check-restart`, `change` 순서로 요청한다.
- 신규 NE는 Angular `Ne` 생성자의 타입을 유지한다. `model=-1`은 number,
  `state=false`는 boolean이며 `site_id=1`, `productcode=2`를 전송한다.
- Equipment에서 six-gen COT/RT의 이름 또는 설명을 수정하면 `455 SlotList`
  Get으로 MCU slot을 찾고, 해당 slot으로 `413 MCUSystem` Set이 성공한 뒤
  `basedata/v1/ne` Modify를 전송한다. 장비 Set 실패 시 basedata를 수정하지 않는다.
- 일반 그룹 삭제는 `basedata/v1/ne-group Delete` 뒤에 같은 `group_id`의
  `topology Delete`를 전송한다. `Block`/`Zone` 그룹은 부모 topology의
  `topology_id`와 그룹 `target_id`가 일치하는 `topology-item Delete`를 전송한다.
- 삭제용 `Topology` 기본값은 `creator=0`, `updater=0`, `delete_state=false`,
  `tp_type=site`, `bg_type=Color`, `drawing_width=0`, `drawing_height=0`이다.
  `TopologyItem` 기본값은 `creator=0`, `updater=0`, `delete_state=false`,
  `item_type=none`, `target_type=none`, `zorder=0`이다.
- runtime cache가 비어 있는 복구 세션에서는 관련 `topology` 또는
  `topology-item GetList`를 먼저 수행한 뒤 primary group delete와 관련 delete를
  순서대로 실행한다.
- 모델 `24000`, `25000`, `25001`, `25002` NE를 삭제한 뒤에는
  `ne-configuration/v1/express-cell Get`에 `product_type=vran-ems`와 `site_id`를
  보내고, 응답 중 `cudu_id`가 삭제한 NE id와 같은 행만 순차적으로 Delete한다.
- NE Configuration의 System 화면은 command 생성 중 `updateNeNameDesc()`를
  비동기로 호출하므로 Equipment 편집 순서와 같다고 가정하면 안 된다.
- 모든 요청은 기존 `URequest`의 `message_type`, `version`, `client_session_id`, `request_id`, `request_type`, `work_type`, `data` 필드를 유지한다.

## UI 기준

- 기준 캡처: `C:/khd/playwright/reports/20260623-112608-six-gen-flexible/screenshots/generated-action.spec.ts-Generated-action-YAML-verification-generated-equipment-list-parity-Equipment-list-readonly-parity-1782181630086.png`
- Angular와 동일한 `font-awesome@4.7.0`과 `uems-icon` 폰트를 사용한다.
- NMS에서는 `site_group`를 표시하지 않고 NOC에서만 유지한다. 같은 부모의
  child group을 모두 배치한 뒤 해당 부모의 NE를 이어 붙인다.
- live active alarm의 미해제 행을 NE별로 집계해 가장 높은 severity를 상태로
  사용하고, 그 결과로 상위 site 상태를 계산한다. 일반 그룹 폴더는 검정이며
  NE와 site 아이콘만 집계 상태 색상을 사용한다.
- destructive action 자동화는 실제 backend에 전송하지 않고 confirmation과 payload preflight까지만 검증한다.
- `Edit`, `Delete` 행 action은 `Equipment.Equipment`의 `Editable` 권한이 필요하다.
- `Configuration` 행 action은 `NeConfiguration.Configuration`의 `View` 권한이 필요하다.
- non-master group target은 지정 그룹과 하위 그룹/NE를 노출하고, 상위 그룹은 tree context로 표시하되 편집 action을 비활성화한다.
- active Angular toolbar 정의의 `Group`, `NE`, `Change`, `Manual NE Discovery`는 현재 `authorityCheck`가 없으므로 React도 같은 노출 상태를 유지한다.
- group tree는 기본 expanded 상태이며 `+`/`-` toggle로 자손 행을 접고 펼칠 수 있어야 한다.
- 데스크톱 shell은 header `45px`, 고정 side menu `250px`, icon mode `60px`, Equipment title `50px`, content bottom gap `20px`을 유지한다.
- side menu pin 해제 시 workspace는 `60px`만 차지하고, menu hover 중에만
  `250px` overlay로 펼쳐진다. 클릭 focus가 남아도 route 이동 뒤 메뉴가 계속
  열린 상태가 되면 안 된다. hamburger는 pin 상태와 별개로 menu 전체 표시를 제어한다.
- Equipment table은 toolbar `50px`, header `32px`, row `38px`이며 body column 세로선은 표시하지 않는다.

## React 구현 위치

- 화면/행 action: `src/screens/equipment/EquipmentScreen.tsx`, `EquipmentTable.tsx`
- six-gen NE identity 수정: `src/screens/equipment/equipmentNeUpdate.ts`
- 연관 데이터 삭제 순서: `src/screens/equipment/equipmentDelete.ts`
- 생성자 직렬화: `src/screens/equipment/equipmentCrudModel.ts`
- 권한/target 계산: `src/screens/equipment/equipmentAuthorityModel.ts`
- tree 상태 계산: `src/screens/equipment/equipmentTreeModel.ts`
- session별 active alarm cache: `src/runtime/faultDataRuntime.ts`
- 권한/target/tree 회귀 테스트: `src/screens/equipment/equipmentModel.test.ts`
- identity sequence 회귀 테스트: `src/screens/equipment/equipmentNeUpdate.test.ts`
- delete sequence 회귀 테스트: `src/screens/equipment/equipmentDelete.test.ts`
- 브라우저 권한 QA: `scripts/qa-sixgen-equipment-authority.mjs`

## React 검증

- `npm run parity:six-gen-flexible`
- `npm run build:six-gen-flexible`
- `npm run qa:six-gen-browser`
- `npm run qa:six-gen-equipment`
- `npm run qa:six-gen-equipment-live`
- `npm run qa:six-gen-equipment-authority`
- Equipment 캡처: `D:/sixgen-react/.codex/visual-qa/desktop-equipment-parity.png`
- 권한 캡처: `D:/sixgen-react/.codex/visual-qa/desktop-equipment-readonly-authority.png`, `desktop-equipment-editable-authority.png`
- `2026-07-18` live gate는 실제 `admin/admin`으로 Angular와 React에 순차
  로그인해 `ne`, `ne-group`, `Active_Alarm`의 URL/body/session fields, 8개 행의
  순서와 상태, 모든 visible action, toolbar, computed style, header/action
  geometry를 비교한다. 증거는
  `D:/sixgen-react/.codex/visual-qa/equipment-live/latest-qa.json`과 같은 폴더의
  Angular/React PNG에 저장한다.
- `scripts/qa-sixgen-equipment.mjs`는 pin/unpin/hover overlay, tree 접기/펼치기,
  검색, list mode, CRUD confirmation, `455 -> 413 -> basedata/ne` 수정 순서,
  `ne-group Delete -> topology Delete` 연관 정리, discovery/change payload
  preflight, Configuration route를 검증한다.

## 남은 전체 이식 위험

- 공통 인증, 사용자 role, 메뉴/route authority 계약은 `six-gen-react-auth-authority-parity.md`를 따른다. Equipment 행 action과 group target은 연결됐지만 다른 화면의 toolbar/table/dialog/form action은 계속 화면별 확인이 필요하다.
- 실제 backend의 Equipment read-only 초기 상태와 API/session 계약은
  `admin/admin` live gate로 확인했다. destructive mutation은 계속 실행하지 않는다.
- Equipment group의 topology/topology-item 정리와 모델별 Express Cell 정리는
  fixture contract로 확인했다. 실제 backend의 destructive cleanup과 장비별
  데이터 상태는 live 검증이 필요하다.
- COT/RT identity 수정의 fixture 요청 순서는 검증됐지만 실제 장비의
  `SlotList`/`MCUSystem` 성공 응답과 basedata 반영은 live 검증이 필요하다.
- Equipment live gate는 핵심 computed style과 geometry를 `3px` 이내로
  비교한다. font anti-aliasing 차이 때문에 full-screen pixel diff 0을 완료
  조건으로 사용하지 않는다.
