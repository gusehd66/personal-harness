# Side Menu Component

## 역할
좌측 product menu를 표시한다. 현재 checkout에서는 `wc-side-menu`와 `wc-side-menu-item` 구조로 렌더링된다.

## 관찰된 메뉴
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

## 작업 기준
- 메뉴 노출 문제는 menu definition만 보지 말고 provider import, `ContentsManagerService` filtering, `authorityCheck`를 함께 확인한다.
- 메뉴가 접힌 상태에서는 DOM에 하위 메뉴 text가 남아 있을 수 있으므로 실제 화면 노출 여부는 visible 상태와 좌표를 함께 확인한다.
- badge 숫자는 alarm/equipment 상태에 따라 변할 수 있으므로 매뉴얼에는 특정 숫자를 고정 설명하지 않는다.

