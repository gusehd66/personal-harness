# Six-Gen React NE Configuration Port Parity

## 목적

Angular `six-gen-flexible`의 `NE Configuration > Configuration > Port`를
React로 이식할 때 Slot, Port, FlexPort 화면과 command 실행 순서를 동일하게 유지한다.
모바일 레이아웃은 이 parity 범위에 포함하지 않는다.

## 기준 소스

- 메뉴/provider: `configuration-ui-six-gen-flexible-provider.ts`, RT provider
- Slot: `six-gen-slot.component.ts`, `slot-selection.service.ts`
- Port: `six-gen-port-configuration.component.ts`, `port-command.ts`
- FlexPort: `six-gen-flexport-configuration.component.ts`,
  `six-gen-flexport-editor.component.ts`, `flexport-command.ts`
- command: `system.ts`, `serviceinterface.ts`, `mefservice.ts`
- 공통 실행: `configuration-panel.component.ts`, `configuration-sidebar.component.ts`

## Slot 계약

`455 SlotList`를 `shelf-id`만 넣어 조회한다. 응답은 Port module의
`ConfigSlotProvider.onlyI200AndSwu()`에 따라 I200과 SWU만 남기고 slot id 오름차순으로
정렬한다. 첫 항목을 자동 선택하고 다음 형식으로 표시한다.

`17 (UnitType:I200)`

Slot 선택이나 Slot 새로고침으로 첫 항목이 다시 선택되면 Port와 FlexPort 조회를
다시 실행한다. FlexPort panel은 선택 unit type이 I200일 때만 표시한다.

## Port 조회 계약

선택 shelf/slot에 대해 다음 command를 순차 조회한다.

1. `456 PortConfiguration`
2. `454 PortStatus`

`PortConfiguration.port-id`와 `PortStatus.name`을 shelf/slot 안에서 병합한다.
`1/N` 형식만 남기고 N 숫자순으로 표시한다. Port Id는 N만 화면에 보인다.
description이 `n_a`이면 빈 문자열로 표시한다.

편집 필드는 Admin Status, Signal, Desc, Port Speed이다. I200에서는 Port Speed 선택지를
다음처럼 제한한다.

- 1~10: `SPEED_1GB`, `SPEED_10GB`, `SPEED_25GB`
- 11~12: `SPEED_100GB`

## Port 설정 계약

변경된 행만 처리하고 command 종류별로 분리해 순차 Set한다.

- `456`: shelf-id, slot-id, port-id + 변경된 admin-status/signal/desc
- `454`: shelf-id, slot-id, name + 변경된 enabled/port-speed

한 행에서 두 command 필드가 모두 바뀌면 456 다음 454 순서로 두 요청을 보낸다.
Set metadata는 `operationFunction: Configuration`, `operationSubFunction: Port`이다.
변경 필드가 없으면 요청을 보내지 않고 `error_no_change` 메시지를 표시한다.

## FlexPort 계약

I200 slot에서 `466 FlexportConfiguration`을 shelf-id/slot-id로 조회한다. 빈 name은
backend echo row이므로 제외하고 name을 숫자 자연 정렬한다. Detach row의
Active Port List는 `-`로 표시한다.

연필 action은 600px `Edit FlexPort` sidebar를 연다. 원본 `max-speed=100`은 NNI
group으로 간주한다.

- ethernet NNI speed: 100GB만
- ethernet non-NNI speed: 10GB, 25GB
- cpri speed: CPRI7, CPRI8, CPRI10
- ethernet NNI breakout: NONE, 10Gx4, 25Gx4
- 그 외 breakout: NONE
- Member 2는 port-list가 두 개 이상일 때만 표시

Member를 선택하면 member와 attach를 함께 전송한다. Member가 `(No change)`이면
attach가 바뀐 경우에만 attach를 전송한다. port-type/max-speed는 Attach일 때만
변경 필드를 보내고 breakout은 attach와 관계없이 변경 시 전송한다.

## FlexPort 점유 검증

FlexPort Set 전에 다음 command를 순차 조회한다.

1. `410 UNIInterface`
2. `409 NNIInterface`
3. `411 MEFService`

UNI/NNI의 `interface` 물리 포트가 현재 FlexPort `port-list`에 포함되면 Set을 차단하고
`err_usage_serviceinterface` 메시지를 표시한다. Angular는 점유 조회 중 예외가 발생하면
빈 점유 집합으로 처리해 Set 검증을 계속하므로 React도 fail-open 동작을 유지한다.
점유 조회를 기다리는 동안 Slot/NE가 바뀔 수 있으므로 조회 직후 selection generation을
다시 확인하고, stale context이면 점유 dialog와 466 Set을 모두 중단한다.

FlexPort sidebar Set은 공통 panel의 `Configuration > Port` metadata를 물려받지 않고
sidebar 기본 operation function을 사용한다. Port table Set metadata를 강제로 재사용하면
operation log parity가 달라진다.

## 데스크톱 UI 계약

- 기준 viewport: `1440x960`
- accordion header: `40px`
- Slot form row: `42px`, label width `180px`
- Port table header: `32px`, row: `38px`
- Port table body: `542px` 기준
- FlexPort editor: 오른쪽 고정 `600px`
- View authority: Port Set, editable cells, FlexPort 연필 action 비활성화

## 완료 게이트

- 455 -> 456 -> 454 -> 466 초기 조회 흐름
- I200/SWU Slot 필터와 첫 항목 자동 선택
- 456/454 병합, `1/N` 필터, `n_a` 변환
- 변경 필드별 456/454 순차 Set
- I200 포트별 speed option
- FlexPort label, option, member+attach, breakout mapping
- 410 -> 409 -> 411 점유 검증 후 466 Set
- 점유 차단과 점유 조회 실패 fail-open
- no-change Set 억제
- stale Get/Set 응답 억제
- View authority disabled 상태
- SWU 선택 시 FlexPort panel/request 미생성
- `1440x960` Angular/React screenshot 비교
