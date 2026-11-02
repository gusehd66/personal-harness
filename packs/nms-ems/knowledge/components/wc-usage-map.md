# WC Component Usage Map

## 목적
NMS FE에서 공통 `wc-*` 컴포넌트를 새로 만들거나 수정하기 전에 실제 사용 패턴을 빠르게 찾기 위한 지도다.

## 기본 원칙
- `wc-*`를 새로 감싸는 custom component를 만들기 전에 기존 sample과 product 사용처를 확인한다.
- table/list는 `wc-data-display.md`, form/filter/detail은 `wc-form-controls.md`, layout/dialog/toolbar는 `wc-layout.md`를 우선 읽는다.
- 화면 변경은 common component 영향과 매뉴얼 캡처 영향까지 함께 판단한다.

## 화면-level 공통 컴포넌트 탐색

- 화면-level WC 우선 정책과 native 예외 경계는 `packs/nms-ems/PACK.md`의 `공통 UI 컴포넌트 우선 원칙`을 따른다. 이 문서는 그 판단에 필요한 실제 selector, 입력값, 이벤트, 권한, disabled/readonly 동작과 consumer를 찾는 지도다.
- 사진·목업에서 식별한 button/action, 입력·선택, table/list, dialog/tab, toolbar/layout 역할마다 아래 대응 `wc-*`와 실제 사용처를 먼저 확인한다.

| 화면 역할 | 우선 확인 대상 | 함께 확인할 근거 |
| --- | --- | --- |
| button/action | `wc-button`, `wc-buttonbar`, toolbar action | 권한, disabled/readonly, icon, width, click handler |
| 입력·선택 | `wc-form`, `wc-form-row`, `wc-form-value`, combobox/input 계열 | validation, blank/null, readonly, 변경 이벤트 |
| table/list | `wc-table`, `wc-treetable`, renderer, action column | selection, pagination, action side effect |
| dialog/tab/layout | `wc-dialog`, `wc-window`, `wc-tab`, `wc-card`, `wc-split-panel`, `wc-scroll-panel` | close/save action, scroll/split state, clipping |

- native 예외가 필요하면 `PACK.md`의 허용 조건을 적용하고, 확인한 source 또는 consumer 경로를 결과에 남긴다.

### Button/action 근거

- 현재 UEMS source의 `framework_modules/wccontrol/wc-button/wc-button.component.ts`는 `wc-button` selector를 제공한다. 화면 구현 전에 이 component와 실제 consumer를 확인한다.
- `wc-button`은 내부적으로 native `<button>`을 렌더링한다. 따라서 WC 내부의 native markup은 화면-level WC 우회가 아니다.
- 확인 대상 input과 동작의 예: `authorityCheck`, `disabled`, `readonly`, `imageClass`, `isFormButton`, `fixWidth`, `dataToggle`, `dataTarget`, click handler. 실제 적용 가능 여부는 대상 checkout의 component source와 기존 화면에서 다시 확인한다.
- 대표 sample: `src/app/temp/wccontrol/wc-form/wc-form-sample.component.ts`. 대표 product consumer: `src/app/content/contents/neconfiguration/content/ne-dashboard/ne-gallery/ne-gallery-editor.component.ts`.

## Sample 위치
대표 sample 경로:

- `src/app/temp/wccontrol/wc-table/`
- `src/app/temp/wccontrol/wc-form/`
- `src/app/temp/wccontrol/wc-validation/`
- `src/app/temp/wccontrol/wc-toolbar/`
- `src/app/temp/wccontrol/wc-splitpanel/`
- `src/app/temp/wccontrol/wc-dialog/`
- `src/app/temp/wccontrol/wc-window/`
- `src/app/temp/wccontrol/wc-treeview/`
- `src/app/temp/wccontrol/wc-treetable/`

## Table 계열
사용 전 확인:
- `WCTableModule` import
- `WCTable` component reference
- `wc-table` template
- column 정의
- row selection
- action column
- custom renderer
- pagination/search 연동

대표 실제 사용 범위:
- Equipment 목록
- Fault/Alarm 목록
- History/Performance 목록
- Security/User 목록
- Preference/Alarm Definition 목록
- NE Configuration 내부 Port/Inventory/Alarm/Performance table

위험:
- action column은 조회처럼 보여도 수정/삭제/명령 실행으로 이어질 수 있다.
- renderer 변경은 severity/status 색상과 의미를 바꿀 수 있다.
- column 변경은 manual, export/download, backend response contract에 영향을 줄 수 있다.

## Form 계열
사용 전 확인:
- `WCFormModule` import
- `wc-form`, `wc-form-row`, `wc-form-label`, `wc-form-value`
- `wcRequire`
- readonly/disabled
- validation component/pattern
- blank/null/duplicate 처리

대표 실제 사용 범위:
- Preference 설정 form
- User create/edit form
- Equipment/ShelfView detail panel
- NE Configuration 설정 panel
- search/filter panel

위험:
- disabled/readonly 누락은 조회 화면에서 값 변경을 유발할 수 있다.
- validation 우회는 잘못된 운영 설정 저장으로 이어질 수 있다.
- duplicate check는 blank/null 값을 별도로 다뤄야 한다.

## Layout/Toolbar 계열
사용 전 확인:
- `WCToolbarModule`
- `WCSplitPanelModule`
- `wc-split-panel`, `wc-split-left`, `wc-split-right`
- `wc-toolbar-component`
- `wc-card`, `wc-scroll-panel`, dialog/window/tab 계열

대표 실제 사용 범위:
- Topology/ShelfView의 좌측 tree + 중앙 workspace + 우측 detail panel
- Preference의 내부 tree + 우측 content panel
- Equipment/Fault/Security list의 toolbar
- Configuration 내부 tree/content layout

위험:
- split/scroll 변경은 table header, toolbar, detail panel clipping을 만들 수 있다.
- toolbar icon button은 tooltip/code 근거 없이 기능을 추측하지 않는다.
- dialog/window는 저장/적용/삭제 side effect가 있을 수 있다.

## 새 화면 추가 기준
1. 화면 유형을 분류한다.
2. 유사 화면의 module/component/service를 찾는다.
3. 해당 화면의 `wc-*` import와 template 구조를 복사 가능한 기준으로 삼는다.
4. action/authority/API/validation만 기능에 맞게 최소 변경한다.
5. manual update 필요 여부를 결과 보고에 남긴다.
