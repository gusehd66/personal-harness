# 공통 UI 레이아웃

## 사용 시점
NMS 화면 설명, 매뉴얼 캡처, UI 변경, 메뉴 추가, 화면 구조 분석 작업에서 이 문서를 먼저 확인한다.

## App Shell
- 상단 `app-main-toolbar`에는 제품명, 언어 선택, 사용자 메뉴, logout 아이콘이 있다.
- 좌측 `app-left-menu`는 product menu 영역이다.
- 중앙 `app-content-main-panel`은 현재 선택한 메뉴의 실제 업무 화면이다.
- page title은 `app-content-main-title` 또는 `.content-title`에 표시된다.

## 기본 화면 분할
현재 `six-gen-flexible` UI는 `wc-split-panel`, `wc-card`, `wc-toolbar`, `wc-scroll-panel` 조합을 자주 사용한다.

대표 구조:

```text
App Toolbar
Left Product Menu
Content Title
Main Content Panel
  - optional left selector/tree/search panel
  - main workspace/table/form panel
  - optional right Detail panel
```

## 화면 유형
브라우저 조회 기준으로 다음 화면 유형을 확인했다.

| 화면 유형 | 예시 메뉴 | 특징 |
| --- | --- | --- |
| Workspace/Canvas | `Dashboard > Topology`, `Equipment > ShelfView` | 좌측 tree, 중앙 시각화 영역, 우측 Detail panel을 함께 사용한다. |
| Topology + Table | `NE Configuration > MEF Service Management` | 좌측 topology 선택 영역과 우측 조건/목록 table을 함께 사용한다. |
| Table/List | `Equipment > Equipment`, `Security > User`, `Security > User Session` | 상단 toolbar와 목록 table 중심이다. Action column이 있어도 조회 작업에서는 클릭하지 않는다. |
| Search + Table | `Fault > Active Alarm`, `History > Alarm History`, `History > Performance` | 조회 조건, Search/Reset류 필터, 결과 table을 함께 사용한다. |
| Select Required + Nested Tree | `NE Configuration > Configuration` | 진입 직후 `Select the NE.`가 표시되고, NE 선택 후 내부 configuration tree와 form/table이 열린다. |
| Preference/Form | `Preference` | 설정 tree와 form/table content를 사용한다. 조회 기준으로만 확인하고 저장/적용 동작은 수행하지 않는다. |

## 매뉴얼 작성 원칙
- 메뉴명만 쓰지 말고 화면이 어떤 레이아웃 유형인지 함께 설명한다.
- 캡처는 가능하면 좌측 menu, page title, 주요 panel, table/header 또는 detail panel이 보이도록 잡는다.
- Workspace 화면은 좌측 tree, 중앙 workspace, 우측 Detail panel을 분리해서 설명한다.
- Table 화면은 toolbar/filter, table header, Action column 존재 여부를 설명한다.
- Preference/설정 화면은 조회 기준으로만 캡처하고, 값 변경이나 저장 동작은 수행하지 않는다.

## 관련 문서
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/ui/menu.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/ui/current-menu-observation.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/ui/topology-workspace.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/ui/configuration-workspace.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/ui/preference.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/ui/toolbar.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/ui/detail-panel.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/wc-components.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/wc-data-display.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/wc-form-controls.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/wc-layout.md`
