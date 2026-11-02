# Equipment 기능 노트

## 사용 시점
Equipment, shelfview, port, card, COT/RT display, NE dashboard 동작 변경 시 이 문서를 사용한다.

## 현재 FE 주의점
- `six-gen-flexible`은 `sites/six-gen-flexible/contents.module.ts`를 통해 shelfview provider를 import한다.
- COT와 RT shelfview provider는 서로 다른 mapping behavior를 가질 수 있다.
- Equipment action은 active NE, shelf, slot, port, provider selection에 의존할 수 있다.

## 변경 전 확인
- active site file replacement를 확인한다.
- 선택된 NE와 equipment object shape를 식별한다.
- 변경이 COT, RT 또는 둘 다에 영향을 주는지 확인한다.

## 위험
- 잘못된 shelf/slot/port mapping은 action이 잘못된 equipment를 대상으로 하게 만들 수 있다.
- provider mismatch는 한 equipment type을 고치면서 다른 type을 깨뜨릴 수 있다.
- UI-only 변경도 operational action을 숨길 수 있다.
