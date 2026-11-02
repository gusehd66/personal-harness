# WC Layout 컴포넌트

## 포함 대상
- `wc-card`
- `wc-toolbar`
- `wc-split-panel`
- `wc-scroll-panel`
- `wc-dialog`
- `wc-window`
- `wc-tab`
- page/content panel layout

## 사용 시점
- 화면 분할, toolbar, modal/dialog, tab, scroll 영역을 수정할 때 사용한다.
- 매뉴얼 캡처에서 어떤 panel을 포함해야 하는지 판단할 때 사용한다.
- table/form/canvas를 배치하는 공통 레이아웃을 바꿀 때 사용한다.

## 코드 위치 힌트
- sample: `src/app/temp/wccontrol/wc-card/`
- sample: `src/app/temp/wccontrol/wc-toolbar/`
- sample: `src/app/temp/wccontrol/wc-splitpanel/`
- sample: `src/app/temp/wccontrol/wc-dialog/`
- sample: `src/app/temp/wccontrol/wc-window/`
- sample: `src/app/temp/wccontrol/wc-tab/`

## 대표 사용 화면
- `Dashboard > Topology`: 좌측 tree, 중앙 canvas, 우측 Detail panel
- `Equipment > ShelfView`: 좌측 tree, 중앙 shelfview, 우측 Detail panel
- `NE Configuration > MEF Service Management`: topology 영역과 service table 영역
- `Preference`: 내부 tree와 우측 form/table content

## 변경 전 확인
- 기존 split 비율과 scroll 영역이 화면 크기에 맞게 동작하는지 확인한다.
- toolbar button은 icon-only인 경우 tooltip 또는 코드 근거로 기능을 확인한다.
- modal/dialog는 저장/적용/삭제 같은 side effect action을 포함할 수 있으므로 조사 중 함부로 실행하지 않는다.
- content title, left menu, main panel, right detail panel이 매뉴얼 캡처에서 잘 보이는지 확인한다.
- route/baseHref/serve-path/proxy assumption을 layout 변경과 함께 건드리지 않는다.

## 주의할 위험
- split/scroll 변경은 table header, toolbar, detail panel clipping을 만들 수 있다.
- toolbar button 위치 변경은 운영자가 command/action을 잘못 누르게 만들 수 있다.
- dialog size 변경은 form validation 메시지나 button footer를 가릴 수 있다.
- tab 변경은 lazy loading, refresh timing, selected state에 영향을 줄 수 있다.

## Anti-patterns
- 한 화면만 보고 shared layout style을 바꾸는 것.
- 기존 `wc-card`, `wc-toolbar`, `wc-split-panel` 패턴이 있는데 별도 layout을 만드는 것.
- icon-only button을 문서화할 때 기능을 추측해서 적는 것.

