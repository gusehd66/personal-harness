# Six-Gen React Operation Log Parity

## Active scope

- Angular source: `HistoryContentProviderModule` registered by
  `sites/six-gen-flexible/contents.module.ts`
- Route: `/history/operation`
- 데스크톱 기준은 `1440x960`이며 모바일은 이 이식 범위 밖이다.

## Request contract

- Operation API: `ulog-mgmt/v1/operation-log`
- Work types: `GetFilterCode`, `Search`, `TotalCount`
- User가 비어 있으면 `data`에서 생략하며 `null`로 직렬화하지 않는다.
- Start/End는 Angular의 밀리초 정밀도와 로컬 시간대 오프셋을 유지한다.
- 모든 보호 요청은 `access_token=session_id`와 본문
  `client_session_id=session_id`를 사용한다.

행 수는 `system/v1/user-profile`에 type `RowPerPage`, target `Common`으로 저장한다.
Get은 `{ id: 0, user_id, type: 'RowPerPage', target: 'Common' }`을 전송한다.
저장 시 다른 화면 값을 유지하고 불러온 profile ID에 따라 Add 또는 Modify를 선택한다.

## UI contract

- Create Time 조건 하나에 날짜 유형, Start, End 컨트롤이 포함된다.
- Reset/Search는 두 번째 검색 행에 함께 배치된다.
- 데이터가 비어 있으면 표 헤더는 DOM에 유지하지만 헤더 행은 시각적으로 숨긴다.
- 행이 0건이어도 Download는 활성 상태를 유지하며 핸들러는 요청을 보내지 않는다.
- 패널, 컨트롤, 버튼과 빈 상태의 배치를 `1440x960`에서 Angular와 직접 비교한다.

## Verification

- Fixture: `node scripts/qa-sixgen-operation-log.mjs`
- Live: `npm run qa:six-gen-operation-log-live`
- Evidence: `.codex/visual-qa/operation-log-live/`
- Latest visual result: `98/100` similarity, `0.0245` diff ratio

실환경 게이트는 인증과 요청 캡처를 실제로 수행한다. 실환경 Search 엔드포인트가
Angular Loading 상태에서 60초 타임아웃까지 머물 수 있으므로, 각 Search/TotalCount의
URL·헤더·본문을 캡처한 뒤에만 Angular와 React에 같은 빈 성공 응답을 제공한다.

이 범위의 검증은 전체 Six-Gen 이식 완료의 근거가 아니다. 전체 완료 판단은
`sites/six-gen-flexible/contents.module.ts`의 활성 route/menu/provider 목록에 따른다.
