# WC Data Display 컴포넌트

## 포함 대상
- `wc-table`
- `wc-treetable`
- `wc-treeview`
- table column/action/renderer
- row selection, pagination, sorting, filtering
- list/table 기반 조회 화면

## 사용 시점
- `Equipment`, `Security`, `Fault`, `History`, `Preference`, `NE Configuration`의 table/list 화면을 수정할 때 사용한다.
- table column, action column, renderer, row selection, pagination, filter/search 동작을 바꿀 때 사용한다.
- 새 목록 화면을 만들기 전에 기존 table 패턴을 확인할 때 사용한다.

## 코드 위치 힌트
- sample: `src/app/temp/wccontrol/wc-table/`
- sample: `src/app/temp/wccontrol/wc-treetable/`
- sample: `src/app/temp/wccontrol/wc-treeview/`
- module import 예: `wccontrol/wc-table/wc-table.module`
- component import 예: `wccontrol/wc-table/wc-table.component`
- column/action import 예: `wccontrol/wc-table/wc-table-column`
- renderer import 예: `wccontrol/wc-table/wc-table-renderer`

## 대표 사용 화면
- `Equipment > Equipment`
- `Fault > Active Alarm`
- `History > Alarm History`
- `History > Performance`
- `Security > User`
- `Security > User Session`
- `Preference > Alarm Definition`
- `NE Configuration > Configuration` 내부 Port/Inventory/Alarm/Performance table

## 변경 전 확인
- 기존 table/base component 또는 renderer를 재사용할 수 있는지 확인한다.
- column 순서와 label은 매뉴얼, export/download, backend response contract와 연결될 수 있다.
- `Action` column은 기능을 단정하지 말고 코드와 tooltip 근거를 확인한다.
- row identity, selection behavior, checked state, multi-select 여부를 확인한다.
- filter/search 조건과 pagination이 함께 동작하는지 확인한다.
- severity/status renderer가 있으면 기존 renderer를 유지한다.

## 주의할 위험
- selection bug는 잘못된 NE, alarm, shelf, port에 action을 실행하게 만들 수 있다.
- filter/pagination 오류는 데이터가 없는 것처럼 보이게 만들 수 있다.
- shared renderer 변경은 여러 화면의 색상, severity, status 표시를 동시에 바꿀 수 있다.
- action column 변경은 조회 화면처럼 보여도 실제로는 수정/삭제/명령 실행으로 이어질 수 있다.

## Anti-patterns
- 새 화면마다 one-off table component를 만드는 것.
- 기존 renderer를 무시하고 화면별로 severity/status 표시를 새로 구현하는 것.
- table column 변경 후 매뉴얼/캡처/API contract 영향을 보고하지 않는 것.
- 조회 작업 중 `Action` column button을 직접 눌러 기능을 확인하는 것.

