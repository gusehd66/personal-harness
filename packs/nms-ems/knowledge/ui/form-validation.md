# Form Validation UI 노트

## 사용 시점
create/edit dialog, modal form, required field, duplicate check, submit blocking behavior 변경 시 이 문서를 사용한다.

## 필수 입력
- current model value
- edit mode의 original/base value
- required field rule
- duplicate comparison set
- backend error response handling

## 기존 화면
- Local User create/edit는 account, password, role, email validation이 서로 영향을 주므로 high-risk 예시다.

## 안티패턴
- blank optional value를 duplicate로 취급하는 것.
- create와 edit flow를 모두 확인하지 않고 validation을 함께 바꾸는 것.
- backend confirmation 전에 success를 표시하는 것.
- backend validation error를 숨기는 것.

## 변경 전 확인
- validation order를 식별한다.
- null, empty, duplicate, unchanged edit value를 확인한다.
- submit 이후 async refresh timing을 확인한다.
