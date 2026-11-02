# Six-Gen React Dashboard Topology Parity

## 범위

- Angular 기준: `C:/khd/6gworkspace/UnifyEMS/uems_fe`
- React 대상: `D:/sixgen-react`
- 대상 화면: `/dashboard/topology`, `/dashboard/topology/:groupId`
- 검증 해상도: 데스크톱 `1440x960`
- 모바일은 현재 범위 밖이다.

## Angular 기준 분기

- `SixGenFlexibleContentProvider`는 Main Dashboard를 숨기고 Equipment Topology를 `Dashboard > Topology`로 이동한다.
- 기본 화면은 첫 authorized site group을 선택한 뒤 `/dashboard/topology/<group_id>`로 이동한다.
- 화면 종류는 group type만으로 결정하지 않는다.
- Angular `EquipmentTopologyComponent`가 `Model3D` system config를 읽어 `model3DEnable`을 설정한 뒤 `checkTopologyType()`을 실행한다.
- `model3DEnable`이 false이면 site group도 `Topology2D`다.
- 2026-07-15 실제 six-gen 실행 기준은 `model3DEnable=false`였고, `/dashboard/topology/2`는 저장된 `topology`와 `topology-item`을 사용하는 2D SVG 화면이었다.

## React 구현 위치

- 화면: `src/screens/dashboard/DashboardTopologyScreen.tsx`
- 데이터/분기: `src/screens/dashboard/widgets/topologyModel.ts`
- 응답 파서: `src/screens/dashboard/widgets/topologyParsers.ts`
- 2D 렌더링: `src/screens/dashboard/widgets/TopologySurface.tsx`
- 2D 모델/선: `src/screens/dashboard/widgets/topology2dModel.ts`
- 상세 패널: `src/screens/dashboard/topologyDetailModel.ts`
- panel profile: `src/screens/dashboard/topologyViewProfile.ts`
- zoom control: `src/screens/dashboard/widgets/TopologyZoomControls.tsx`

## 유지해야 할 계약

- 제목은 `Location :`이며 group path를 뒤에 붙이지 않는다.
- group tree는 `vCore`, `Floor`, `Block`, `Zone`, `site_group`을 표시하지 않는다.
- 저장된 `topology-item`이 있으면 `bounds`의 x, y, width, height를 그대로 사용한다.
- direct child group과 direct NE를 표시하고, Floor/Zone/Block 아래 중첩 NE 규칙은 Angular visibility와 맞춘다.
- COT/RT 이미지는 각각 `6g/COT.png`, `6g/RT.png`를 사용한다.
- alarm severity와 통신 상태에 따라 Angular SVG filter id를 유지한다.
- `ne-topology`의 parent/child NE가 모두 보일 때 line을 렌더링한다.
- line center는 정수 절삭 후 양 끝 3px indent를 적용한다. parallel link는
  10px 간격과 Angular 반올림을 사용한다.
- Angular normal link의 0.56 base와 0.96 overlay 합성 alpha는 0.9824다.
  NE 선택 시 overlay는 숨고 base `stroke-opacity=0.16`만 남는다.
- NE 선택 시 선택 node opacity는 1, 나머지 node는 0.28이다.
- workspace zoom 초기값은 수동 100%다. Auto Fit, Zoom 전환, 40~300 범위,
  Reset 100%를 유지한다. 88px numberbox에는 `%` unit과 20px spinner 영역이
  있으며, auto-repeat은 500ms 후 시작해 50ms 간격으로 반복한다. 직접
  입력은 Enter/blur에서 적용한다.
- group node 더블클릭은 해당 group route로 이동한다.
- NE node 클릭은 이름, 상태, 설명, 시리얼 번호, 그룹, 모델, 버전, IP를 Detail panel에 표시한다.
- panel 상태는 `user-profile`의 `type=Topology`, `target=TopologyView`에 `groupVisible`, `detailVisible`로 저장한다.
- refresh notification은 다음 6개를 모두 처리한다.
  - `ne-group-Sync`
  - `ne-Sync`
  - `topology-Sync`
  - `topology-item-Sync`
  - `ne-topology-Sync`
  - `alarm-Sync`
- 모든 REST 요청은 활성 로그인 `session_id`를 `access_token` header와 body `client_session_id`에 동일하게 넣고 별도 `client_session_id` HTTP header를 보내지 않는다.
- active product의 `show3DModel=false`에서는 `topology-3d-model` 요청을 보내지 않는다.
- Active Alarm은 live NE id와 Angular EM id `[1,0]`을 전송한다.
- ScreenLock은 `{id:0,user_id,type:'Security',target:'ScreenLock'}` 전체 query를
  전송하며, default route의 `TopologyView` Get은 Angular와 같이 2회 발생한다.

## 검증

- mock browser gate: `SIXGEN_REACT_URL=http://localhost:<port>/unms npm run qa:six-gen-dashboard-topology`
- 실제 Angular/React Topology gate:
  `SIXGEN_TEST_USERNAME=admin SIXGEN_TEST_PASSWORD=admin SIXGEN_ANGULAR_URL=http://127.0.0.1:4201/unms SIXGEN_REACT_URL=http://127.0.0.1:4317/unms npm run qa:six-gen-dashboard-topology-live`
- live gate는 request/session, tree, asset, link path/composed alpha, node
  filter/selection, detail, zoom DOM/style/interaction, 1px geometry와 initial/
  detail screenshot을 함께 검증한다.
- 전체 browser gate: `SIXGEN_REACT_URL=http://localhost:<port>/unms npm run qa:six-gen-browser`
- 실제 destructive action은 실행하지 않고 confirmation과 payload preflight까지만 검증한다.

## 남은 주의점

- `drawing_image` overlay가 실제 운영 데이터에 존재하면 Angular 합성 순서와 별도로 비교해야 한다.
- 현재 live topology의 2개 parallel link와 1개 single link path는 검증됐다.
  다른 운영 topology shape가 추가되면 같은 path/composed-alpha gate에 fixture를
  추가해야 한다.
- `model3DEnable=true` 환경에서는 TopologyMap/Topology3D를 별도 gate로 검증해야 하며, 현재 2D 통과만으로 3D parity 완료로 처리하지 않는다.
