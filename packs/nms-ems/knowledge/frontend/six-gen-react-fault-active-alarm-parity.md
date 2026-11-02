# Six-Gen React Fault Active Alarm Parity

## 2026-07-18 live parity 보정

- 완료 기준은 `sites/six-gen-flexible/contents.module.ts` 활성 범위이며 이
  화면은 그 범위의 `FaultContentProviderModule`에 속한다.
- 현재 알람 table은 17개 Angular 열을 모두 수평 순서대로 유지한다. 초기
  viewport에는 checkbox부터 `Creation Time`까지 보이고 `Action`은 우측에
  80px로 고정된다.
- 열 상태는 localStorage가 아니라 `system/v1/user-profile`의
  `ControlState`, target `fault.current.alarm-table`을 사용한다. 값은
  `columnStates[{columnID,visible,width}]`이며 Angular column ID를 그대로
  직렬화한다.
- 이미 인지된 행도 checkbox와 Ack 버튼은 활성 상태다. 단일 행 Ack는
  확인창 전에 반환하고, 다중 Ack는 확인 후 요청 목록에서만 제외한다.
- `npm run qa:six-gen-fault-active-alarm-live`는 실제 `admin/admin` 세션에서
  Angular/React의 event-type, alarm, table-profile 요청 URL/header/body와
  filter, severity, 15개 viewport 행, 버튼 상태, computed style, 1440x960
  geometry를 비교한다. destructive mutation은 전송하지 않는다.

## 범위

- Angular 기준: `C:/khd/6gworkspace/UnifyEMS/uems_fe`
- React 대상: `D:/sixgen-react`
- 화면: `Fault > Active Alarm`, `/fault/current-alarm`
- 화면 기준: 데스크톱 `1440x960`; 모바일은 범위 밖이다.

## Angular 기준 소스

- 화면/toolbar/filter: `src/app/content/contents/fault/content/current-fault/current-alarm/current-alarm.component.ts`
- filter helper: `current-alarm-helper.ts`
- table/action: `current-alarm-table.component.ts`, `src/app/content/common/fault/alarm-table/alarm-table.component.ts`
- Ack With Comment sidebar: `current-alarm-sidebar/current-alarm-sidebar.component.ts`
- Action History: `src/app/content/contents/fault/content/current-alarm-action/`
- API: `src/app/content/common/fault/common/fault.service.ts`
- 리소스: `src/assets/lang/common_*.json`, `fault_*.json`, `control_*.json`, `security_*.json`

## 필터 계약

- 초기 상태는 Status, Alarm Type, Reason, Severity가 모두 `All`이다.
- Alarm Type 목록은 event-code의 `alarm_type_name`을 중복 제거하고 정렬한다.
- 특정 Alarm Type의 Reason은 해당 type만 필터링한 뒤 정렬하지만 중복을 제거하지 않는다.
- Alarm Type이 `All`일 때 Reason은 backend event-code 원본 순서와 중복을 그대로 보존한다.
- Alarm Type 변경 시 Reason은 `All`로 초기화한다.
- severity count는 현재 state/type/reason 조건을 적용하되 선택된 severity 자체는 적용하기 전에 계산한다.

## 권한과 버튼

- authority code는 `Fault.Alarm`이다.
- `Editable`에서 row Ack, multi-ack, Action History Add/Modify/Delete가 활성화된다.
- Ack With Comment action은 Angular처럼 항상 표시한다.
- 이미 인지된 alarm은 row checkbox와 Ack를 비활성화한다.
- multi-ack는 선택된 alarm마다 `Acknowledge` 요청을 순차 전송한다.
- destructive action은 live backend에 자동 실행하지 않고 confirmation과 payload preflight까지만 검증한다.

## 요청 계약

- 조회: `fault/v1/alarm`, work type `Active_Alarm`
- 이벤트 코드: `fault/v1/get-event-type`
- 인지: `fault/v1/alarm`, work type `Acknowledge`
- Action History 조회: `fault/v1/alarm`, work type `Get_Action`
- Action History mutation: `fault/v1/alarm-action`
- Ack payload는 React view model로 다시 만들지 않고 Angular와 같은 전체 raw alarm wire object를 보존한다.
- 보호 요청의 `access_token` header와 body `client_session_id`는 활성 communication `session_id`와 같다.

## UI와 리소스 계약

- toolbar 순서는 Status, Alarm Type, Reason, severity counters, multi-ack, column selector, table/grid이다.
- 기본 table은 checkbox, Severity, Network Group, Part, HostName, Location, Alarm Type, Reason, Action 순서다.
- table/grid, column selector, 확인 문구, Action History dialog/editor/user table, empty/loading/error text, CSV header는 Angular en/ko/ja 리소스를 사용한다.
- Angular `ResourceProvider.getString`은 번역 키가 없으면 key 이름을 반환한다. React도 임의 영문으로 보정하지 않는다. 예를 들어 영어와 일본어 `common.title`은 `title`로 표시한다.
- Action History CSV header도 현재 언어의 Angular 리소스 값을 사용한다.
- column visibility와 width는 Angular와 동일하게 `ControlState` type, `fault.current.alarm-table` target의 user-profile `value`에 column state JSON으로 저장한다.
- `alarm`, `alarm-Sync`, `ne-Sync` notification에서 조회를 갱신한다.

## React 구현 위치

- 화면: `src/screens/dashboard/widgets/ActiveAlarmSurface.tsx`
- model/filter: `activeAlarmModel.ts`
- 다국어 계약: `activeAlarmLabels.ts`
- table/grid: `ActiveAlarmViews.tsx`
- Action History: `ActiveAlarmActionSidebar.tsx`
- API: `activeAlarmApi.ts`
- 단위 테스트: `activeAlarmModel.test.ts`, `activeAlarmApi.test.ts`
- 브라우저 QA: `scripts/qa-sixgen-fault-active-alarm.mjs`

## 검증

- `npm run test:react`
- `npm run typecheck:react`
- `npm run build:six-gen-flexible-dev`
- `npm run qa:six-gen-fault-active-alarm`
- `npm run qa:six-gen-history-alarm`
- 영문 캡처: `D:/sixgen-react/.codex/visual-qa/desktop-fault-active-alarm.png`
- 한국어 캡처: `D:/sixgen-react/.codex/visual-qa/desktop-fault-active-alarm-ko.png`
- Angular 기준 캡처: `C:/khd/playwright/reports/20260623-112608-six-gen-flexible/screenshots/generated-action.spec.ts-Generated-action-YAML-verification-generated-fault-active-alarm-parity-Fault-active-alarm-readonly-parity-1782181637399.png`

## 남은 위험

- fixture 기반 request, interaction, authority, desktop visual gate는 통과했지만 실제 backend의 alarm/action 상태 전이는 추가 검증이 필요하다.
- live 인증 gate가 통과하기 전에는 이 화면의 live API parity나 전체 포팅 완료 근거로 사용하지 않는다.
- 같은 table/action 컴포넌트를 History Alarm이 공유하므로 리소스나 button tooltip 변경 후 두 browser QA를 함께 실행한다.
