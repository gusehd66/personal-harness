# Menu UI 노트

## 사용 시점
menu가 보이지 않거나, 중복되거나, route가 틀리거나, 잘못된 authority에 표시될 때 이 문서를 사용한다.

## 확인 순서
1. `angular.json`을 확인하고 active configuration을 식별한다.
2. `six-gen-flexible`에서는 generic contents module보다 `sites/six-gen-flexible/contents.module.ts`를 먼저 확인한다.
3. content provider module이 import되었는지 확인한다.
4. provider가 menu와 route를 정의하는지 확인한다.
5. `ContentsManagerService` filtering을 추적한다.
6. `authorityCheck`와 role/permission data를 확인한다.
7. 현재 서버에서 실제 노출되는 메뉴는 브라우저에서 직접 펼쳐 확인한다.
8. 메뉴 클릭 조사는 조회 기준으로만 수행하고, 설정/수정/삭제/저장 버튼은 누르지 않는다.

## 현재 서버 관찰 기준
2026-06-10에 Chrome에서 조회 기준으로 확인한 좌측 메뉴 인벤토리는 `<HARNESS_ROOT>/packs/nms-ems/knowledge/ui/current-menu-observation.md`에 기록한다.

현재 확인된 visible menu:

- `Dashboard > Topology`
- `Equipment > Equipment`
- `Equipment > ShelfView`
- `Fault > Active Alarm`
- `NE Configuration > Configuration`
- `NE Configuration > MEF Service Management`
- `Security > User`
- `Security > User Session`
- `History > Alarm History`
- `History > Performance`
- `Preference`

## 공통 UI 문서
메뉴가 단순 navigation인지, workspace/table/form 화면인지 판단하려면 다음 문서를 함께 읽는다.

- `<HARNESS_ROOT>/packs/nms-ems/knowledge/ui/layout.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/ui/topology-workspace.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/ui/toolbar.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/ui/detail-panel.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/side-menu.md`

## 금지 사항
- active site module을 확인하기 전에 menu definition을 수정하지 않는다.
- menu를 보이게 하려고 authority check를 제거하지 않는다.
- 매뉴얼 조사 중 `Add`, `Create`, `Edit`, `Delete`, `Apply`, `Save`, `Discovery`, `Command`, `Upgrade`, `Initialize` 류 동작을 실행하지 않는다.

## 위험
- menu가 코드에 있어도 provider filtering 또는 authority gating 때문에 숨겨질 수 있다.
- generic module 수정은 active site build에 영향을 주지 않을 수 있다.
- route/baseHref/serve-path 변경은 deep link와 refresh behavior를 깨뜨릴 수 있다.
