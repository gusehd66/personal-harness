# Toolbar UI

## 사용 시점
상단 toolbar, workspace toolbar, table toolbar, search/filter toolbar를 설명하거나 수정할 때 사용한다.

## Toolbar 종류

| 종류 | 위치 | 예시 |
| --- | --- | --- |
| App toolbar | 최상단 | 제품명, 언어 선택, 사용자 메뉴, logout |
| Content title toolbar | content 상단 | 현재 화면 제목 |
| Workspace toolbar | Topology/ShelfView/MEF workspace 내부 | zoom, Display Options, Drawing, Background |
| Table toolbar | table 상단 | Total List count, Search/Reset, table actions |
| Filter toolbar | Search + Table 화면 상단 | Target Type, Severity, Time Type, Period 등 |

## Topology 계열 toolbar
관찰된 버튼/컨트롤:

- zoom percentage input
- full screen 또는 fit view 아이콘
- undo/return 계열 아이콘
- `Display Options`
- `Drawing`
- `Background`
- `Group`
- `NE`
- edit/pencil 아이콘
- delete/trash 아이콘
- disabled 상태의 장비/설정/refresh류 아이콘

## Table/List toolbar
관찰된 화면:

- `Equipment > Equipment`
- `Security > User`
- `Security > User Session`
- `Fault > Active Alarm`
- `History > Alarm History`
- `History > Performance`
- `NE Configuration > MEF Service Management`

주요 패턴:

- `Total List : N rows` 형태의 목록 수 표시
- Search/Reset 또는 조건 필터
- Action column이 있는 table

## 안전 기준
- 매뉴얼 조사 중에는 조회 조건 입력, 검색, reset 정도만 신중하게 사용한다.
- `Add`, `Create`, `Edit`, `Delete`, `Apply`, `Save`, `Discovery`, `Command`, `Upgrade`, `Initialize` 버튼은 누르지 않는다.
- 아이콘만 있는 버튼은 기능을 확인하기 전까지 클릭하지 않는다. DOM class, tooltip, 주변 label, 기존 코드명을 먼저 확인한다.
- disabled 버튼은 현재 선택 상태 또는 권한 조건에 따라 활성화될 수 있으므로 문서에는 disabled 상태였음을 명시한다.

