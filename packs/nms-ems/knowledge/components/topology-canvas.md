# Topology Canvas Component

## 역할
Topology, ShelfView, MEF Service Management에서 장비 또는 service 대상의 시각적 선택 영역으로 사용된다.

## 관찰된 화면별 차이
- `Dashboard > Topology`: group/NE object를 배치하고 선택하면 우측 Detail panel이 갱신된다.
- `Equipment > ShelfView`: COT/RT 장비 shelf 구성을 보여주고 unit 선택에 따라 Detail panel이 갱신된다.
- `NE Configuration > MEF Service Management`: 좌측 topology 선택 영역으로 사용되고, 우측 table/list 조건과 연결된다.

## 작업 기준
- canvas object 선택은 조회 기준에서 허용된다.
- `Group`, `NE`, edit, delete, background, drawing 변경 버튼은 매뉴얼 조사 중 누르지 않는다.
- 화면 설명은 canvas만 설명하지 말고 tree/toolbar/detail 또는 table과의 연결을 함께 설명한다.

