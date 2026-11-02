# Six-Gen React Operation Content Root Parity

## Active contract

- Source of truth: `sites/six-gen-flexible/contents.module.ts`.
- `OperationContentModule`은 활성 `/operation` 콘텐츠 루트를 등록한다.
- `/operation`은 작업 영역 제목이 비어 있는 활성 비메뉴 route다.
- `OperationContentProviderModule`은 `/history/operation`에 Operation Log를 별도로 추가한다.
- `/operation`을 `/history/operation`으로 리다이렉트하지 않는다. route 동작과 API 부작용이 모두 달라진다.

## Angular UI contract

Angular 루트 컴포넌트에는 `<p>Hello</p>`만 있다. 아이콘 너비의 사이드 메뉴가 있는
`1440x960` 데스크톱 기준에서 문단 상자는 `x=80`, `y=95`, `1340x18`이다.
계산된 스타일은 글꼴 `13px`, 줄 높이 `18.2px`, 검정 글자, 투명 배경과
`margin: 0 0 13px`이다.

## Request contract

`/operation` 콘텐츠 루트는 Operation Log 요청을 보내지 않는다. 이 route에서
`ulog-mgmt/v1/operation-log`를 요청하거나 Operation Log 컴포넌트를 렌더링하면
동등성 게이트는 실패해야 한다.

## Verification

- Fixture: `npm run qa:six-gen-operation-root`
- Authenticated Angular/React parity: `npm run qa:six-gen-operation-root-live`
- Evidence: `D:\sixgen-react\.codex\visual-qa\operation-root-live\`

이 범위의 검증은 전체 이식 완료의 근거가 아니다. 전체 완료 판단은 활성 route, 메뉴와 provider 목록에 따른다.
