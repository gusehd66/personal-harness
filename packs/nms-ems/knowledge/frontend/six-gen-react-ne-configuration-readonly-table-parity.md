# Six-Gen React NE Configuration Readonly Table Parity

## 목적

Angular `six-gen-flexible`의 Port Status, Port Statistics, Unit Inventory,
Module Inventory를 React로 이식할 때 조회 command, 필터, 정렬, 표 표시를 동일하게 유지한다.
모바일 레이아웃은 parity 범위에 포함하지 않는다.

## 2026-07-19 latest-only request sequence

- React Port Status는 Angular의 `414 -> 415 -> 408` 명령 순서를 유지한다.
- slot 또는 route 전환 시 이전 요청의 다음 명령을 실행하기 전에 해당 요청을 중단한다.
  generation 검사는 이전 응답이 현재 표를 갱신하지 못하도록 계속 방지한다.
- 회귀 테스트는
  `src/screens/ne-configuration/neReadonlyTablesApi.test.ts`다.
- Playwright 연속 10회 실행에서 뒤늦은 이전 `415/408` 요청이 발생하지 않았고,
  전체 활성 화면 브라우저 테스트는 한 번의 467.4초 실행에서 통과했다.

## Port Status

`ConfigSlotProvider.exceptMcu()` 기준으로 `455 SlotList` 응답에서 MCU만 제외한다.
선택한 shelf/slot에 대해 다음 command를 순차 조회한다.

1. `414 PortStatusModuleInventory`: `management-shelf-id`, `management-slot-id`
2. `415 PortStatusInterface`: `interface-shelf-id`, `interface-slot-id`
3. `408 MCUAlarms`: `shelf-id`, `slot-id`

415의 `name`을 기준 행으로 사용한다. 414의 `port-id`, 408의 `resource`는 대소문자를
무시하고 병합한다. Admin Status는 enabled를 ACT/DEACT로 변환하고 Port ID는 `1/N`의
N만 표시한다. description `n_a`는 빈 문자열이다. alarm-type-id namespace를 제거한 뒤
Angular `convertSpecific_problem` 매핑을 적용한다. MCU v2.15 구형 PM alarm은 Angular의
`isMCUv215PMAlarm`과 같이 빈 상태로 숨긴다.

## Port Statistics

Slot 필터는 Port Status와 동일하다. 선택한 shelf/slot으로 `422 PortStatistics`를 조회한다.
`name`이 정확히 `1/N`인 행만 남기고 N을 숫자 오름차순으로 정렬한다. 화면에는 N만
표시한다. ULong counter는 JavaScript number로 변환하지 않고 문자열로 유지해 64-bit
값의 정밀도를 보존한다.

## Unit Inventory

Slot panel 없이 shelf-id만 포함한 `465 UnitInventory4Class`를 조회한다.
`hardware-class === ianahw:container`인 행만 표시하고 slot-id 숫자 오름차순으로 정렬한다.

표 컬럼은 Slot ID, Name, MFG Name, Model Name, Serial Number, MFG Date,
Hardware Revision, Software Revision이다.

## Module Inventory

현재 Angular module의 `ConfigSlotProvider(I200)`를 기준으로 `455 SlotList`에서 I200만
표시한다. 선택한 shelf/slot으로 `416 ModuleInventory`를 조회한다.
`status === EQUIP`인 행만 남기고 port-id 자연 정렬 후 `1/N`의 N만 표시한다.

## 비동기 계약

Slot/NE 전환 중 이전 Get 응답이 늦게 도착할 수 있다. React 화면은 Slot 목록과 표 조회에
각각 generation을 두고 현재 generation과 다른 응답을 폐기한다. 414/415/408은 backend와
Angular command 순서를 보존하기 위해 병렬화하지 않는다.

## 데스크톱 UI 계약

- 기준 viewport: `1440x960`
- Main menu profile: `General`, side menu width `250px`
- accordion header: `40px`
- Slot form row: `42px`, label width `180px`
- Slot ID 컨트롤 너비: 읽기 전용 표는 `309px`, 다른 일반 Slot 폼은
  `214px`를 유지하며, Performance도 별도로 관찰한 `309px` 토큰을 유지한다.
- table header: `32px`, row: `38px`
- 고정 컬럼 폭과 가로 스크롤 유지
- 조회 화면에는 Set button과 편집 control이 없어야 함

## 완료 게이트

- command ID와 selector field diff 0
- Slot provider 필터와 첫 항목 선택 일치
- Port Status 3-command 병합 및 stale 응답 억제
- Port Statistics `1/N` 필터, 숫자 정렬, 64-bit counter 보존
- Unit container 필터와 slot 정렬
- Module EQUIP 필터와 자연 정렬
- refresh button 재조회
- `1440x960` 네 화면 screenshot 비교

## 2026-07-19 authenticated live parity contract

- 완료 기준은 `sites/six-gen-flexible/contents.module.ts`에서 import한 활성 provider와
  route이며, product에서 비활성화된 화면은 이 범위의 종합 게이트에 포함하지 않는다.
- Angular와 React를 순차 실행한다. 장비 통신 세션을 공유하지 않도록 React 로그인 전에
  Angular Logout 계약을 완료한다.
- 모든 보호 명령은 `access_token=session_id`와 `client_session_id=session_id`를
  사용해야 한다. URL, 헤더, 본문, 명령 순서, 기본 필드, 행과 응답 적용 상태를 비교한다.
- Port Status는 정확한 대체 문구 `NOT-EQUIP`을 사용한다.
- Port Statistics 카운터는 10진수 문자열로 유지한다. 누락된 카운터는 Angular
  `getBigInteger` / `getInteger` 동작과 같이 `0`으로 표시한다.
- Port Statistics에는 의도적으로 `full-size` 프레임이 없다. Port Status, Unit
  Inventory, Module Inventory는 Angular 테두리, 8px 반경, 흰 배경과 옅은 그림자를 유지한다.
- Slot 기반 읽기 전용 표는 330px 값 영역 안에 브라우저에서 관찰한 309px 컨트롤을
  사용한다. 바깥 폼뿐 아니라 컨트롤 상자 자체를 비교한다.
- 실환경에서 관찰한 표 높이는 Slot 기반 화면 543px, Unit Inventory 640px다.
  Angular 컴포넌트 소스의 host 높이는 542px이지만 Slot 값은 브라우저 실측 레이아웃을 유지한다.
- 표 헤더 높이는 32px이다. 본문 행은 38px이며 셀 안쪽 여백은 세로 4px, 가로 17px다.
  헤더 글자는 12px/600/#828a99, 본문 글자는 13px/#000이다. 표는 Angular의 Segoe UI
  대체 글꼴 목록과 18.2px 줄 높이를 사용하며 행 테두리와 교차 배경은 Angular alpha 색상을 사용한다.
- 실환경 게이트는 `npm run qa:six-gen-ne-configuration-readonly-tables-live`다.
  `.codex/visual-qa/ne-readonly-tables-live/latest-qa.json`에 비식별 증적만 기록하며
  Set 요청이 0건이어야 한다.
