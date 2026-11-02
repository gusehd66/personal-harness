# Six-Gen React NE Configuration Performance Parity

## 범위

React 이식 대상은 `NE Configuration > Configuration > Performance > Performance`다.

- URL: `/ne-configuration/configuration/sixgen_performance`
- 기준 viewport: `1440x960`
- 모바일은 parity 범위에서 제외한다.
- Angular COT/RT Performance provider와 six-gen active site route를 기준으로 한다.

## 화면 및 Authority 계약

- 화면은 `Slot`, `Statistics`, `Optic`, `Bias`, `Voltage`, `Temp`, `L1`, `L2` 순서다.
- authority target은 root `NeConfiguration`이다.
- `None`은 `/nopermission`으로 이동하고 `View`와 `Editable`은 조회를 허용한다.
- Angular 화면과 같이 Set 동작은 없으며 각 data panel에 Refresh만 제공한다.
- NE가 바뀌면 slot과 모든 panel 상태를 초기화하고 새 대상의 데이터를 다시 조회한다.

## Command 계약

초기 조회는 다음 순서를 직렬로 유지한다.

1. `455 SlotList` Get
2. `421 PerformanceFCS` Get
3. `419 PerformanceConfiguration` Get: Optic
4. `419 PerformanceConfiguration` Get: Bias
5. `419 PerformanceConfiguration` Get: Voltage
6. `419 PerformanceConfiguration` Get: Temp
7. `417 PerformanceL1` Get
8. `418 PerformanceL2` Get

- 모든 command에는 `shelf-id`, `slot-id`를 포함한다.
- Angular `ConfigurationPanel.processCommandExecute()` Get 경로와 같이 `configKey`는 보내지 않는다.
  `NeConfigurationObject`처럼 별도로 `configKey`를 설정하는 경로와 혼동하지 않는다.
- 모든 요청의 `access_token` header와 body `client_session_id`는 로그인 `session_id`와 같다.
  별도 `client_session_id` HTTP header는 보내지 않는다.
- Slot Refresh는 `455` 조회 뒤 첫 slot을 다시 선택하고, 같은 slot이어도 모든 data panel을 다시 조회한다.
- panel Refresh는 해당 command만 다시 조회한다.
- Slot 변경은 전체 data panel을 같은 직렬 순서로 다시 조회한다.
- NE, slot, 개별 panel request generation을 각각 비교해 늦게 도착한 이전 응답이 현재 화면을 덮지 않게 한다.

## 데이터 규칙

- `455` 결과에서 MCU를 제외하고 slot ID 숫자 오름차순으로 정렬한다.
- Slot label은 `${id} (UnitType:${name})` 형식이며 첫 항목을 기본 선택한다.
- Statistics의 `FCS`, byte, frame 값은 JavaScript number로 변환하지 않고 문자열로 보존한다.
- `port-id`와 `name`의 `/` 뒤 값을 화면 key로 사용하고 빈 key 행은 제거한다.
- 행 정렬은 숫자 문자열을 고려한 natural sort다.
- `bin-type`은 `:` 뒤 값, `bin-info`는 세 번째 comma token을 표시한다.
- `stats-l0`은 Optic 0..7, Bias 8..11, Voltage 12..15, Temp 16..19로 나눈다.
- Optic/Bias/Voltage/Temp의 `0.00`은 Angular와 같이 `-`로 표시한다.
- L1은 `stats-l1` 3개를 사용한다.
- L2는 command `418 PerformanceL2`를 요청하지만 Angular component의 현재 cast/display 동작대로
  응답의 `stats-l1` 27개를 표시한다. 이 규격은 의도적으로 개선하지 않는다.
- L2 컬럼의 기존 오탈자 `Fragement`도 규격으로 유지한다.

## 데스크톱 UI 계약

- `1440x960`에서 Slot control 너비는 `309px`다.
- data panel 높이는 `250px`, panel 간격은 Angular root `13px` 기준 Bootstrap `mb-2`와 같은 `6.5px`다.
- panel은 border `2px` + header `38px` + body padding `10px` + table `200px` 구조다.
- table은 panel 좌우에서 `5px` 안쪽에 배치한다.
- table header 높이는 `32px`이며 마지막 컬럼만 남은 너비를 채운다.
- table header는 `12px/#828a99`, body는 `13px/#000`, cell padding은 `4px 17px`다.
- 각 panel의 `4px` 청록색 header rail과 `1px` body rail은 panel 끝까지 이어진다.
- header shadow는 `0 1px 6px rgb(0 0 0 / 23%)`이며 뒤따르는 form/table 위에 겹쳐 보인다.
- Slot select는 좌우 `11px`, 아래 `1px` padding과 `3px` radius를 유지한다.
- 공용 readonly table의 치수와 색상도 `--ne-configuration-readonly-*` 및 공통 table token을 사용한다.
- React 구현은 `DESIGN.md`의 `--ne-configuration-performance-*` token을 사용한다.

## 검증

- 모델/API: `nePerformanceModel.test.ts`, `nePerformanceApi.test.ts`
- 브라우저: `npm run qa:six-gen-ne-configuration-performance`
- 검증 항목: route/tree, authority, panel 순서, command 순서, slot filter/sort, parser,
  BigInteger 문자열, session header/body, panel Refresh, slot reload, stale 응답 차단,
  Set 미노출, panel/table geometry, full-height rail과 outer chrome
- Angular 기준 캡처와 React 캡처는 모두 `1440x960`으로 생성하고 `visual-qa.mjs image-diff`로 비교한다.
- live 인증 gate는 실제 `admin/admin` 로그인에서 `Login -> WebSocket -> keepalive 응답 -> environment -> user`
  순서와 보호 요청의 `access_token/client_session_id=session_id`를 함께 확인한다.
- live Angular는 Slot 선택과 panel 초기화가 겹쳐 data command를 세 번 반복하며, 첫 transient cycle은
  `slot-id.value`가 없다. React는 이 불완전/중복 요청을 재현하지 않고 Slot 확정 후 유효 sequence만 한 번 보낸다.
- `npm run qa:six-gen-ne-configuration-performance-live`는 동일한 COT/slot, 모든 유효 command signature,
  응답 status, 7개 panel header/row, table geometry/computed style, Set 미전송을 비교한다.
- command signature는 `message_type`, `version`, `request_id`, `request_type`, `work_type`,
  `operationSubFunction`, `product_type`, `configKey`, `data`를 포함한다. 세션 값은 raw artifact에 저장하지 않고
  `access_token` header와 body `client_session_id`의 일치 여부만 기록한다.
- live gate는 모든 command 응답의 HTTP `200`, JSON body, application status `1`을 강제한다. redacted count,
  status, session/wrapper 비교 결과는
  `.codex/visual-qa/ne-performance-live/performance-live-result.json`에 저장한다.
- live 장비 응답은 모든 command가 성공했지만 row가 비어 있었다. populated row 변환과 정렬은 fixture gate로
  검증하며, live populated row parity는 장비 데이터가 생길 때까지 미검증으로 남긴다.
- fresh `1440x960` full-screen image diff는 `94/100` similarity와 intact alpha다. shell icon/font 차이는
  별도로 남고 Performance panel의 측정 geometry와 style은 일치한다. 상세 diff는
  `.codex/visual-qa/ne-performance-live/performance-live-image-diff.json`에 저장한다.
