# Six-Gen React NE Configuration System Parity

## 목적

Angular `six-gen-flexible`의 `NE Configuration > Configuration > System`을
React로 이식할 때 화면, 요청, 장비 선택 타이밍을 같은 계약으로 유지한다.

## 기준 소스

- 메뉴/provider:
  `configuration-ui-six-gen-flexible-cot-provider.ts`,
  `configuration-ui-six-gen-flexible-rt-provider.ts`
- 화면:
  `six-gen-system-configuration.component.ts`
- 공통 명령 실행:
  `configuration-panel.component.ts`
- 상단 NE 선택:
  `neconfiguration-content.component.ts`
- 공통 form:
  `wc-form.component.ts`, `wc-form-common.ts`

## 조회 계약

System 조회는 다음 명령을 순서대로 전송한다.

1. `413 MCUSystem`
2. `391 UnitInventory`
3. `464 MCUData`

각 명령은 `ne-configuration/v1/process-ne-command`로 전송한다. Get 요청은
Angular `ConfigurationPanel.processCommandExecute(...)`와 같이 operation log용
`operationFunction`을 넣지 않는다.

응답의 `data`는 UTF-8 JSON command array를 base64로 인코딩한 값이다. 세 HTTP
응답을 모두 decode한 뒤 command ID와 field name으로 System view model을 합친다.

## 설정 계약

Set은 `413 MCUSystem` 하나만 전송하며 hostname/description 중 실제 변경된
field만 포함한다. 공통 field인 `shelf-id`, `slot-id`는 유지한다.

Set 요청에는 다음 operation log metadata가 필요하다.

- `operationFunction: Configuration`
- `operationSubFunction: System`

Angular 화면은 basedata의 NE name/description 동기화를 기다리지 않고 장비 Set을
진행한다. React도 basedata 실패 때문에 장비 명령을 막지 않아야 한다.

`shelf_id`는 `0`도 유효한 장비 값이다. 누락값을 임의로 `1`로 바꾸지 않는다.

## 장비 전환 타이밍

사용자가 NE A 조회 중 NE B로 전환하면 A 응답이 늦게 도착할 수 있다. React
화면은 조회 세대 번호 또는 AbortSignal로 이전 요청을 무효화해야 한다. 선택된
NE와 form 데이터가 서로 다른 상태에서 Set 버튼이 활성화되면 잘못된 장비에
설정할 수 있으므로 이 검증은 필수다.

브라우저 QA에는 느린 A 응답과 빠른 B 응답을 인위적으로 만들고, 최종 form과
상단 선택 요약이 B 데이터인지 확인하는 시나리오를 둔다.

## 데스크톱 UI 계약

- 모바일 레이아웃은 현재 이식 범위가 아니다.
- 기준 viewport는 `1440x960`이다.
- 내부 tree는 `250px`이다.
- `wc-form` label width는 `180px`, 두 value group 사이 gap은 `13px`이다.
- 행 높이, 색상, 버튼 상태, 선택 전 empty state는 Angular screenshot과
  `D:/sixgen-react/DESIGN.md` token을 함께 기준으로 삼는다.
- 현재 route 버튼은 disabled 상태지만 hover/active UI와 다른 버튼 기능은
  그대로 유지한다.

## 완료 게이트

- request command ID/order와 Set 변경 field diff
- operation log metadata
- `shelf_id=0` 보존
- 빠른 NE 전환 stale response 방지
- authority에 따른 Set disabled 상태
- `1440x960` Angular/React screenshot 비교
- destructive command는 confirmation과 payload preflight까지만 자동 검증

