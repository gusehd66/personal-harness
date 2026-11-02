# Command Toolbar Component

## 역할
화면 상단 또는 panel 내부에서 조회, 표시, 생성, 수정, 삭제, 명령 실행 기능을 제공한다.

## 관찰된 패턴
- Text button: `Display Options`, `Drawing`, `Background`, `Group`, `NE`
- Icon button: pencil, trash, refresh, monitor, setting류
- Table toolbar: `Total List : N rows`, search/filter actions

## 안전 기준
- 조회 목적에서는 표시 옵션 확인, 검색 조건 확인, table header 확인까지만 수행한다.
- 생성/수정/삭제/명령 실행 버튼은 누르지 않는다.
- icon-only 버튼은 tooltip 또는 코드 근거 없이 기능을 단정하지 않는다.

