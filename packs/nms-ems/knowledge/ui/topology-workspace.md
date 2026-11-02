# Topology 계열 Workspace UI

## 사용되는 화면
- `Dashboard > Topology`
- `Equipment > ShelfView`
- `NE Configuration > MEF Service Management`

## 공통 구조
Topology 계열 화면은 단순한 이미지 영역이 아니라 여러 공통 UI 컴포넌트가 조합된 workspace다.

```text
Content Title
Left selector/tree or topology selection area
Main workspace/canvas or list area
Workspace toolbar
Optional right Detail panel
```

## Dashboard > Topology
- URL: `/unms/dashboard/topology/2`
- 좌측 tree: site/group 구조를 표시한다. 예: `A001001`, `1`, `2`, `3`
- 중앙 workspace: group/NE object를 배치해 topology를 표시한다.
- toolbar: zoom, Display Options, Drawing, Background, Group, NE, edit/delete류 아이콘이 있다.
- 우측 Detail panel: object를 선택하면 `Parent Group`, `Name`, `Description` 필드가 표시된다.

## Equipment > ShelfView
- URL: `/unms/equipment/shelf-view`
- 좌측 tree: site/group/NE 선택 영역이다.
- 중앙 workspace: COT/RT 등 장비 shelf 구성을 시각적으로 표시한다.
- 우측 Detail panel: 선택한 unit 기준으로 `Selected Unit`, `Unit Info`, `Slot Number`, `Unit Type`, `Equip Status`, `Admin Status`, `PM Initialize`, `LED Test`, `Information` 등을 표시한다.

## NE Configuration > MEF Service Management
- URL: `/unms/ne-configuration/mef-service-management`
- 좌측 workspace: topology object를 선택하는 영역이다.
- 우측 panel: `Selected NE`, `Slot ID` 조건과 `MEF Service List` table이 있다.
- table header: `Slot ID`, `Name`, `Category`, `Type`, `Vlan ID`, `Description`, `Action`
- 이 화면은 Topology 계열 UI를 사용하지만 우측 Detail panel 대신 조건 toolbar와 table panel이 핵심이다.

## 안전한 조회 기준
- tree node 선택, canvas object 선택, 화면 이동은 조회 작업으로 볼 수 있다.
- `Group`, `NE`, pencil, trash, save, apply, delete, discovery, command류 버튼은 매뉴얼 조사 중 누르지 않는다.
- Display Options 같은 보기 설정은 열어볼 수 있지만 checkbox/toggle 값을 바꾸지 않는다.

## 매뉴얼 작성 원칙
- Topology/ShelfView/MEF 화면은 같은 workspace 계열로 묶어 설명한다.
- 각 화면 문서에는 공통 구조를 반복하기보다 해당 화면의 차이점을 적는다.
- 캡처는 workspace만 확대하지 말고 left tree/selection, toolbar, main area, detail 또는 table panel이 함께 보이게 한다.

