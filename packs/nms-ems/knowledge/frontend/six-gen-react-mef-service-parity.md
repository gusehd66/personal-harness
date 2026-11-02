# Six-Gen React MEF Service Management Parity

## 범위

- Angular 기준: `NE Configuration > MEF Service Management`
- React 경로: `/ne-configuration/mef-service-management`
- 데스크톱 기준: `1440x960`
- 모바일 parity는 현재 범위 밖이다.

## 조회 순서

1. topology에서 COT NE를 선택한다.
2. `request-mefservice/Get`으로 command ID `455` `SlotList`를 요청한다.
3. 응답에서 첫 `I200` slot을 선택한다.
4. 선택 COT와 `cot_ne_id`가 일치하는 RT를 찾는다.
5. 각 NE에 `request-mefservice/Search` command ID `411` `MEFServiceInfoExt`를 요청한다.
6. 결과는 `svc_id#s_vlan#svc_type#service_category`로 묶는다.

SlotList 요청은 단순 shelf 객체가 아니다. Angular command model 전체를 유지한다.

- protocol: `NETCONF`
- targetModel: `0`
- messageType: `Request`
- object defineName: `SlotList`
- field: `shelf-id` `UInteger`
- isAutoCommit: `true`
- commandDefineID: `455`
- `_parsedValues`: `{}`

## 변경 계약

- Create/Update: `six-gen-flexible/v1/request-mefservice`
- Create data: 직렬화된 MEF service 배열
- Update data: 변경 배열, `old_data`: 기존 배열
- UNI/NNI reference: command model 전체가 아니라 ID 필드만 남긴 참조 객체
- NNI create: command ID `409`
- UNI create: command ID `410`
- Delete: `ne-configuration/v1/process-ne-command`, command ID `432` `MEFServiceID_Delete`

모든 보호 요청은 로그인 `session_id`를 header `access_token`과 body
`client_session_id`에 동일하게 사용한다.

## 화면 및 권한

- 메인 화면은 topology/list split이며 별도 title/provider card를 추가하지 않는다.
- 서비스, endpoint, interface editor 폭은 각각 `1300px`, `1100px`, `800px`다.
- Add/Modify/Delete는 root `NeConfiguration` 권한을 사용한다.
- `View`에서는 버튼을 숨기지 않고 disabled 상태로 유지한다.
- 생성 시 체크된 setting이 0개면 요청하지 않고 편집기를 유지한다.
- Delete는 확인창 승인 전에는 command ID `432`를 보내지 않는다.

### 편집기 Action 무동작 계약

서비스 편집기의 setting table은 Details/Set/Delete를 모두 표시하지만 Angular
`onActionClick`은 Set만 처리한다. Details와 Delete는 편집기를 유지하고 하위
편집기를 열지 않으며 request도 보내지 않는 의도적 no-op이다. endpoint
편집기에서도 Delete branch는 비어 있다. React 전환 과정에서 이 버튼에 새 동작을
추가하거나 숨기거나 disabled로 바꾸면 Angular parity 위반이다.

live QA는 Angular와 React에서 서비스 편집기 Details/Delete를 직접 클릭하고
editor visible, endpoint editor count 0, mutation delta 0을 함께 비교한다.
endpoint 편집기 Delete도 양쪽에서 직접 클릭해 editor visible, row count 유지,
mutation delta 0을 확인하고 기존 Cancel 경로로 닫히는지 검증한다. mutation
intercept의 JSON 파싱이 실패하면 fixture/live 모두 빈 객체로 대체하지 않고 QA를
실패시킨다.

### Description 표시 전용 입력 계약

Angular 서비스 편집기의 Description은 `WCTextBox`에 one-way `[value]`만
전달한다. 따라서 입력 자체는 가능하고 사용자가 입력한 문자는 화면에 남지만,
부모 `editData`에는 반영되지 않아 Update의 `user-label`은 조회 당시 원래 값을
유지한다.

React에서는 표시용 로컬 state와 request draft를 분리한다. `readOnly`를 추가하거나
입력값을 request draft에 반영하면 둘 다 Angular 계약 위반이다. 브라우저 검증은
입력 가능 여부, 화면 값, intercepted Update payload를 한 시나리오에서 확인한다.

### WC 시각 계약

MEF 편집기의 기준은 브라우저 기본 form UI가 아니라 활성
`wc-variables-30.scss`와 WC component의 computed style이다.

- font: Segoe UI/Malgun Gothic stack, `13px`, line-height `18.2px`
- label: `#828a99` / `#f7f7f7`
- input: `#e3e4e6`, border `#dcdde0`, radius `3px`, height `30px`
- readonly: `rgba(0, 4, 38, 0.35)`
- accordion: border `#e2e3e5`, header `38px`, active rail `#49b7e5`
- table: header `12px/600`, body `13px/400`, padding `4px 17px`
- footer buttons: `64x34px`; primary `#00a674`
- editor: top `45px`, width `1300px`, header `40px`, footer `52px`, body
  padding `18px 20px`
- service-setting table: header `33px`, row `38px`, 각 고정 column은 이름 있는
  width token을 사용한다.
- header/close/footer, base/detail grid, combo/number button 영역, settings
  toolbar/table, RoE control, endpoint edge, message state도 이름 있는
  `--mef-*` token을 사용한다.

native `select` caret, number spinner, checkbox는 운영체제/브라우저마다 달라져
Angular WC와 동일하게 보이지 않는다. React에서는 값 입력부와 icon 영역을
분리한 공통 control을 사용하고, checkbox는 기존 `icon_form_30.png` sprite의
normal/hover/checked 좌표를 그대로 사용한다. 이 primitive는 표시만 재현하며
기존 `onChange`, disabled, min/max, request draft 계약을 바꾸지 않아야 한다.

시각 비교는 shell 차이가 섞인 전체 화면과 1300x914 editor surface를 분리한다.
2026-07-20 실제 인증 세션 기준 editor `95/100`, full screen `94/100`이며,
동작 완료 판단은 이미지 점수와 별도로 read/update/delete/dialog contract가 모두
일치하고 browser error가 0인지 확인한다.

## 검증

```powershell
$env:SIXGEN_REACT_URL='http://localhost:4202/unms'
npm run qa:six-gen-mef-service
```

QA는 Add, endpoint/interface 선택, NNI 생성 preflight, Create, Modify/Update,
Description 표시 전용 입력, 권한 disabled, delete confirmation/cancel을 검증한다.
실제 Update/Delete는 live parity에서 전송 직전에 가로채 장비 상태를 변경하지 않는다.
