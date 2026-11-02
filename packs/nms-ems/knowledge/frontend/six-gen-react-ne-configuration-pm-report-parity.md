# Six-Gen React NE Configuration PM Report Parity

## 목적

Angular `six-gen-flexible`의 `NE Configuration > Configuration > PM Report`를
React로 이식할 때 화면, command 441, FTP 확인, Set 예외 동작을 동일하게 유지한다.

## 기준 소스

- 메뉴/provider:
  `configuration-ui-six-gen-flexible-provider.ts`,
  `configuration-ui-six-gen-flexible-rt-provider.ts`
- 화면:
  `six-gen-ems-registration-pm.component.ts`
- command:
  `Cmd_PerformanceReport` in `system.ts`
- 공통 명령 실행:
  `configuration-panel.component.ts`
- FTP 설정:
  `SystemConfigService`, `FtpServerConfig`

## 조회 계약

PM Report 조회는 `441 PerformanceReport` 하나를
`ne-configuration/v1/process-ne-command`로 전송한다.

- `shelf-id`: 선택 NE의 `shelf_id`
- `slot-id`: 기본 MCU slot `0`
- `work_type`: `Get`
- `product_type`: 선택 NE service name
- Get에는 `operationFunction`을 넣지 않는다.
- Angular `NeCommandDefine` 기본값인 `_parsedValues: {}`와
  `isAutoCommit: true`를 유지한다.
- Angular `ConfigurationPanel` 경로에는 `configKey`가 없으므로 React에서도
  추가하지 않는다.

활성 PM provider/module에서는 Slot component 등록이 주석 처리되어 있다. 또한
Angular PM component constructor가 구독 초기 replay 뒤 `selectedSlotID = 0`을
설정하므로 command 441의 `slot-id: 0`은 하드코딩 누락이 아니라 현재 MCU 전용
runtime 계약이다.

응답 `data`는 UTF-8 JSON command array의 base64 값이다. `enabled` field를
boolean으로 복원해 ON/OFF radio에 표시한다.

## FTP 확인 계약

화면 생성 시 `system/v1/system-config`에 다음 조회를 보낸다.

- `work_type: Get`
- `data.site_id: 0`
- `data.type: FtpServer`

첫 config row의 JSON `value.address`가 비어 있지 않으면 FTP가 설정된 것으로
판단한다. 이 확인은 장비 command 조회와 독립적으로 진행한다.

## 설정 계약

Set은 변경 여부와 관계없이 항상 `441 PerformanceReport`를 전송한다.
Angular 화면은 `usechangedFieldCheck = false`이므로 변경값이 없어도 요청을
차단하지 않는다.

FTP가 있고 `enabled`가 변경된 경우 field list는 다음과 같다.

- `shelf-id`
- `slot-id`
- `enabled`

FTP가 있어도 변경값이 없으면 `shelf-id`, `slot-id`만 전송한다.

FTP가 없으면
`Please configure FTP server settings before enabling PM report.` 경고를 표시하지만
Set을 취소하지 않는다. `enabled`를 제외한 `shelf-id`, `slot-id` command를 계속
전송한다. 일반적인 사전조건 실패처럼 return을 추가하면 Angular parity가 깨진다.

Set operation log metadata는 다음과 같다.

- `operationFunction: Configuration`
- `operationSubFunction: PM Report`

성공 메시지를 닫은 뒤 command 441을 다시 조회한다.

Set 진행 중 selected NE가 바뀌면 이전 NE의 성공/실패 메시지와 busy 해제가 현재
NE 화면에 반영되지 않아야 한다. Get과 Set 모두 같은 장비 선택 세대 검사를
사용하고, 장비 전환 시 이전 dialog queue를 비운다.

Set 응답이 operation response 형식으로 parse되지 않으면 성공으로 간주하지
않고 실패 메시지를 표시한다.

## 데스크톱 UI 계약

- 모바일 레이아웃은 현재 범위가 아니다.
- 기준 viewport는 `1440x960`이다.
- 아코디언 header는 `40px`, form row는 `40px`이다.
- PM Report form은 공통 NE form과 달리 `13px` 글꼴, `#828a99` label,
  검은 radio text를 사용한다.
- 아코디언은 Angular의 `#e2e3e5` border, `4px #49b7e5` active rail,
  `0 1px 6px rgb(0 0 0 / 23%)` shadow를 유지한다.
- label width는 `180px`이다.
- radio는 Angular `icon_form_30.png`의 `20px` sprite를 사용한다.
- ON/OFF label 간격과 checked/hover/disabled sprite position을 유지한다.
- Set 버튼은 editable authority에서 변경 여부와 관계없이 활성 상태를 유지한다.
- View authority에서는 Set과 ON/OFF radio를 disabled 상태로 유지하고 Set 요청을
  보내지 않는다.

## 완료 게이트

- Get command ID `441`, shelf/slot, product type
- base64 `enabled` 응답 parsing
- 변경 있음/없음 Set field diff
- Set의 `client_session_id`, request/product/NE metadata,
  `_parsedValues: {}`, `isAutoCommit: true`, `configKey` 부재
- FTP 있음/없음 Set field diff와 경고 순서
- operation log metadata
- 성공 메시지 종료 후 재조회
- 빠른 NE 전환 stale response 방지
- Set 응답 대기 중 NE 전환 시 stale dialog/refresh 억제
- authority에 따른 Set/radio disabled 상태
- `1440x960` Angular/React screenshot 비교
- `qa:six-gen-ne-configuration-pm-report-live`에서 Angular 로그아웃 후 React를
  순차 로그인하고 command 441/FTP Get의 URL, body, session field, HTTP/application
  상태를 비교한다.
- live gate는 Set을 실행하지 않고 두 앱의 Set 요청 합계가 `0`인지 확인하며,
  실제 token/session/password를 evidence에 저장하지 않는다.
