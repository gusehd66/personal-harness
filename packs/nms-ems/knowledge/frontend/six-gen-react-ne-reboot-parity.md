# Six-Gen React NE Reboot Parity

## 목적

Angular `six-gen-flexible`의 `NE Configuration > NE Reboot`를 React로 이식할 때
화면 상태, 명령 순서, 권한, 확인 절차를 같은 계약으로 유지한다.

## 기준 계약

- route: `/ne-configuration/ne-reboot`
- child authority: `NeConfiguration.NE Init/Reboot`
- Set confirmation: `Are you sure you want to reboot?`
- Set operation metadata: `operationFunction: Reboot`, `operationSubFunction: ''`
- 명령 API: `ne-configuration/v1/process-ne-command`

## 명령 흐름

System Reboot는 명령 `472`를 전송하며 `shelf-id`와 고정 `slot-id: 126`을
포함한다.

MCU Reboot는 명령 `471`로 `slot-id: 0`의 bank 정보를 조회하고, 선택한
`next-boot` 값으로 명령 `462`를 전송한다.

Unit Reboot는 명령 `471`로 bank 정보를 조회하고 명령 `455`로 slot 목록을
조회한다. `slot-id: 0`은 MCU이므로 Unit 표에서 제외하며 slot 오름차순으로
표시한다. 선택된 Unit은 각 row의 선택 bank를 사용해 명령 `462`를 순차
전송하고 row별 성공/실패 상태를 표시한다.

명령 응답의 `data`는 Angular와 동일한 base64 envelope로 해석한다. 실제
장비의 명령 `471` image 값은 `value[].fieldList` 형태이며, 이전 응답의
`value[].object.instance.fieldList` 형태도 함께 허용한다. 이 차이를 이유로
request 또는 backend payload를 변경하지 않는다.

## 공통 통신 계약

각 REST 요청은 로그인 후 활성화된 communication session의 `session_id`를
`access_token` header와 body `client_session_id`에 동일하게 사용한다. 장비
명령은 WebSocket 첫 keepalive 이전에 실행하지 않는다.

## UI와 권한

- 화면은 System, MCU, Unit 세 panel 순서다.
- MCU 기본 선택 bank는 Angular 초기값과 같은 Bank 1이다.
- Unit 조회 완료 시 Angular selection model처럼 선택 row 없이 시작한다.
- 권한이 `Editable`이 아니면 모든 Set action을 비활성화한다.
- destructive Set 자동 검증은 confirmation과 payload preflight까지만 수행한다.
- 기준 viewport는 `1440x960`이며 모바일은 현재 이식 범위가 아니다.
- System, MCU, Unit과 bank summary 문구는 shell의 Angular `ko/en/ja` resource를
  그대로 사용한다.
- live 비교에서는 세 panel의 좌표와 크기, button 수와 활성 상태, bank 값,
  선택 bank, Unit row를 Angular와 React 양쪽에서 수집한다.

## 완료 게이트

- 명령 ID, field, 요청 순서 diff 0
- authority별 Set enabled/disabled 상태 일치
- confirmation 문구와 취소 시 미전송 확인
- Unit 순차 전송과 row별 결과 표시 확인
- `access_token`, `client_session_id`, body 직렬화 계약 확인
- Angular/React `1440x960` screenshot 비교
- `npm run qa:six-gen-ne-reboot-live`에서 `471:0 -> 471:none -> 455:none`,
  session field 일치, destructive request 0건 확인

실장비의 `472`/`462` Set은 자동 완료 게이트에서 실행하지 않는다. 실제
Set 성공 여부는 운영 승인된 별도 수동 시험으로 남기며, 이 제한만으로 전체
Angular to React 전환을 완료 처리하지 않는다.
