# Six-Gen React Alarm Definition Parity

## 범위

`D:/sixgen-react`의 `/preference/fault/alarm-definition`은 Angular의 다음
구현을 기준으로 유지한다.

- `fault-preference-alarm-definition.component.ts`
- `fault-alarm-definition.component.ts`
- `fault-alarm-definition-sidebar.component.ts`
- `alarm-definition.service.ts`
- `framework_modules/wccontrol/wc-table/`

모바일은 현재 이식 범위 밖이며 desktop `1440 x 960`을 기준으로 한다.

## API 계약

- URL: `/unms/rest-api/fault/v1/alarm-definition`
- request type: `alarm-definition`
- work type: `GetList`, `Update`
- `access_token` header와 body `client_session_id`는 모두 communication
  bootstrap에서 활성화된 동일한 `session_id`를 사용한다.
- Update는 편집한 Angular row의 전체 wire model을 보존한다. 보이는 필드만
  새 객체로 만들거나 빈 값, date, enum 값을 임의로 정규화하지 않는다.

## 화면 동작 계약

- direct route에서는 Preference의 Fault 그룹만 펼친다.
- 필터는 Model, NMS Severity, Auto Acknowledge 순서다.
- Modify 또는 일반 row 클릭은 단일 row를 선택한다.
- editor를 Close하거나 Update 성공 후 닫아도 선택 row를 유지한다.
- Close한 draft는 같은 row를 다시 열 때 유지한다.
- read-only authority에서도 Modify action은 숨기지 않고 disabled로 둔다.
- CSV는 현재 필터 결과를 Angular filename 규칙으로 내려받는다.

## Fixed Action 규칙

Angular는 본문 table과 fixed Action table을 별도 DOM으로 렌더링한다. React는
sticky 마지막 cell을 사용하므로 반투명 alternate 배경을 그대로 적용하면
아래 Category 문자가 비친다.

- 일반 Action cell: `rgb(255, 255, 255)`
- alternate Action cell: Angular 최종 합성 픽셀인 `rgb(249, 249, 249)`
- 선택 row: `rgb(221, 235, 246)`

React에서는 Action cell만 불투명 합성색을 사용하고 본문 alternate cell은
Angular 계산값인 `rgba(247, 247, 247, 0.72)`를 유지한다.

## 검증

```powershell
cd D:\sixgen-react
$env:SIXGEN_REACT_URL='http://127.0.0.1:4317/unms'
npm run qa:six-gen-preference-alarm-definition
$env:SIXGEN_ANGULAR_URL='http://127.0.0.1:4201/unms'
npm run qa:six-gen-preference-alarm-definition-live
```

live QA의 Update는 browser route에서 응답하며 backend로 전달하지 않는다.
전체 Angular-to-React 이식 완료 여부는 이 화면의 통과만으로 판단하지 않는다.
