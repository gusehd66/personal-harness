# Six-Gen React NE Configuration Active Alarm Parity

## 범위

React 이식 대상은 `NE Configuration > Configuration > Fault > Active Alarm`이다.

- URL: `/ne-configuration/configuration/sixgen_active_alarm`
- 기준 viewport: `1440x960`
- 모바일은 parity 범위에서 제외한다.
- Angular source 기준은 COT/RT 공통 Active Alarm module과 현재 six-gen provider다.

## 화면 구성

화면은 Angular와 같은 순서의 두 panel로 구성한다.

1. `Alarm Initialize`: Slot ID 선택, Refresh, Set
2. `Active Alarm`: Refresh와 alarm table

NE가 선택되기 전에는 configuration layout이 자식 route를 렌더링하지 않는다. 화면 단위
authority나 button 상태를 브라우저에서 검증할 때는 먼저 Selected NE를 선택해야 한다.

## Command 계약

| 동작 | Command | 필드 | operation metadata |
| --- | --- | --- | --- |
| Slot 조회 | `455 SlotList` Get | `shelf-id` | 없음 |
| Active Alarm 조회 | `446 ActiveAlarm` Get | 없음 | 없음 |
| Alarm Initialize | `447 AlarmStatusClear` Set | `shelf-id`, `slot-id` | `Configuration / Fault>Active Alarm` |

- Set 전 확인 문구는 `Are you sure you want to initialize?`다.
- destructive Set은 브라우저 QA의 mock backend에서 confirmation과 payload까지만 검증한다.
- 모든 요청의 `access_token` header, `client_session_id` header/body는 로그인 `session_id`와 같다.
- Angular 직렬화 기준으로 `455`와 `447`은 `_parsedValues: {}`를 포함하지만,
  값을 읽지 않는 `446 ActiveAlarm` Get은 `_parsedValues` 필드를 생략한다.
- `447 Set` 응답 대기 중 Selected NE가 바뀌거나 화면이 해제되면 이전 응답의 message/busy 상태를 반영하지 않는다.
  요청 시작 시 화면 generation을 캡처하고 success/error/finally 상태 변경을 현재 generation에만 적용한다.

## 데이터 규칙

- 현재 Angular module의 `ConfigSlotProvider(I200)`를 기준으로 `455` 응답에서 I200 slot만 남긴다.
- slot은 ID 숫자 오름차순으로 정렬하고 첫 항목을 기본 선택한다.
- Slot label은 `${id} (UnitType:${name})` 형식이다.
- `446` 응답에서 `resource`가 없거나 빈 문자열인 행은 제거한다.
- severity는 대문자로 표시한다.

## Authority 계약

- 화면 authority code는 `NeConfiguration.Configuration.Active Alarm`이다.
- COT/RT configuration provider에서 IEEE 1588v2의 세 메뉴와 Fault > Active
  Alarm 메뉴도 같은 authority code를 사용한다.
- Angular `WCMenuTreeNode.enabled`는 authority state가 `Disabled`이면 `false`를
  반환한다. 이때 노드는 숨기지 않고 tree에 남기며 클릭/선택만 막는다.
- React도 denied 메뉴를 제거하지 않는다. `aria-disabled=true` 상태로 15개
  전체 메뉴 순서를 유지하고 route navigation을 발생시키지 않는다.
- React의 `useAdvancedConfigurationTarget(authorityCode)`는 메뉴 authority가
  `None`이면 NE command target을 만들지 않는다. IEEE Global, Clock, Status와
  Active Alarm 직접 URL은 모두 `/nopermission`으로 이동한다.
- `None`은 `/nopermission`으로 이동한다.
- `View`는 조회를 허용하고 Alarm Initialize Set을 비활성화한다.
- `Editable`은 confirmation 이후 Set을 허용한다.
- session resume QA는 `session_id -> WebSocket -> first keepalive -> user/role` 순서를 통과한 뒤 authority snapshot을 사용한다.

## 데스크톱 UI 계약

- Alarm Initialize 다음에 Active Alarm panel이 배치된다.
- table content는 `640px`이고 기준 캡처에 포함된 outer chrome `10px`를 별도 token으로 더한다.
- 컬럼은 Alarm Severity 150px, Resource Name 120px, Cause 250px, Text 200px,
  Time Created 순서다.
- severity badge는 흰 글자, 6px radius, 회색 text shadow를 사용한다.
- 색상은 critical `#EA0606`, major `#F76F00`, minor `#FFE600`, warning `#0055FF`, 그 외 black이다.
- React 구현은 `DESIGN.md`의 `--ne-configuration-active-alarm-*` token을 사용한다.

## 검증

- 모델/API: `neActiveAlarmModel.test.ts`, `neActiveAlarmApi.test.ts`
- 브라우저: `npm run qa:six-gen-ne-configuration-active-alarm`
- non-master denied fixture에서 15개 메뉴 유지, IEEE 1588v2 3개와 Active
  Alarm disabled, click navigation 차단, 네 direct route의 `/nopermission`
  이동을 검증한다.
- 검증 항목: route/tree, command ID, slot/filter/sort, table columns, severity style,
  confirmation, `447` payload, operation metadata, read-only, no-permission, session header/body,
  지연된 `447` 응답 도중 NE 변경 시 stale dialog 미표시
- 동적 alarm 문자열과 발생 시간은 기준 캡처와 달라질 수 있으므로 screenshot diff에서는 panel,
  column, row geometry와 상태 색상을 별도로 확인한다.

## 인증 실환경 검증

- 명령: `npm run qa:six-gen-ne-configuration-active-alarm-live`
- Angular와 React 모두 `system_COT`을 선택하고 `455/446`을 각각 한 번씩
  전송했으며 URL, header, body, 기본 필드와 세션 필드가 일치했다.
- 두 앱 모두 HTTP `200`, application status `1`을 받았고 동일한 10개 행,
  panel title, toolbar icon/state, Slot control과 `1073x640` table을 렌더링했다.
- `1440x960` full-screen mean absolute ratio는 `0.026468`, table-only ratio는
  `0.003230`이었고 browser error는 없었다.
- 실장비 `447 AlarmStatusClear` Set은 전송하지 않았다. confirmation, payload,
  진행/완료 메시지와 stale response는 fixture backend에서 검증한다.
- 증적은 `D:/sixgen-react/.codex/visual-qa/ne-active-alarm-live/latest-qa.json`과
  `D:/sixgen-react/.omo/evidence/ne-configuration-active-alarm-live-parity.md`에 둔다.
- 수동 QA matrix와 실행 기록은
  `D:/sixgen-react/.omo/evidence/ne-configuration-active-alarm-manual-qa.md`,
  `D:/sixgen-react/.omo/evidence/ne-configuration-active-alarm-verification-20260719.md`에 둔다.
- locale 단위 테스트의 expected text는 React 구현 편의를 위한 문구가 아니라
  Angular 영문/한글/일문 resource 원문을 고정한 외부 contract snapshot이다.
- 최신 code-quality, clone-fidelity, completion-gate review는 모두 blocker 없이
  `APPROVE`했다.
