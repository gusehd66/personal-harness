# Toolbar Tooltip Pattern

## Rule
- 아이콘만 있는 도구 모음 UI에 툴팁을 추가하기 전에 같은 도구 모음의 기존 항목을 먼저 추적한다.
- `wc-toolbar-button`은 `toolTipText`를 제공하며, 내부 도구 모음 항목은 `[tooltip]`으로 `TooltipDirective`를 적용한다.
- `wc-toolbar-component` 내부의 사용자 정의 DOM에서는 네이티브 `title`보다 `TooltipDirective`를 import하고 `[tooltip]` / `[tooltipIsVisible]`을 사용하는 방식을 우선한다.
- 대상이 WC UI 패턴 밖에 있거나 브라우저 기본 툴팁 동작을 허용할 명확한 이유가 있는 경우에만 네이티브 `title`을 사용한다.

## Why
- WC 툴팁은 기존 도구 모음 버튼과 위치, 스타일, 클릭 시 숨김 및 비활성 상태 동작을 일관되게 유지한다.
- 네이티브 `title`을 임의로 사용하면 UI 동작이 달라지고 도구 모음 코드가 점차 일회성 구현으로 변한다.

## Code Pointers
- `framework_modules/wccontrol/wc-toolbar/wc-toolbar-item.component.ts`
- `framework_modules/wccontrol/wc-tooltip/wc-tooltip.component.ts`
- Existing app toolbar example: `src/app/home/main-toolbar/main-toolbar.component.ts`

