# Six-Gen React Security User Session Parity

## Active non-menu provider route

- `sites/six-gen-flexible/contents.module.ts`는 `/sixGenFlexible`을 활성 지연 로딩 route로 등록한다.
- `SixGenFlexibleContentModule`이 `UserSessionModule`을 import하므로 빈 하위 route도 동일한 User Session 표를 렌더링한다.
- 이 route는 화면에 보이는 메뉴 항목이 아니며 사이드 메뉴 목록에 추가하면 안 된다.
- `SixGenFlexibleContentProvider`는 `useContentAuthority=false`와 `authorityCode=null`을 사용한다. React 이식에 대체 메뉴 권한을 상속하면 안 된다.
- 이 route는 제목 텍스트가 비어 있는 전역 50px 제목 영역을 유지한다.
- fixture QA는 Dashboard 권한이 비활성화된 비 master 세션을 포함해 `/sixGenFlexible`이 `nopermission`으로 리다이렉트되지 않음을 확인해야 한다.
- 실환경 QA는 Angular/React 직접 URL, 보호 요청, 행, 계산된 스타일과 데스크톱 배치를 비교해야 한다.

## 사용 시점

`Security > User Session`의 API, 세션 필드, 응답 모델, 테이블 렌더링을
Angular와 React 사이에서 비교하거나 수정할 때 사용한다.

## Active Angular 경로

- 화면: `UserSessionComponent`
- 실제 조회: `UserMgmtContentService.getSessionList()`
- 화면에 주입된 `UserSessionService.getList()` 호출은 주석 처리되어 있으므로
  active runtime 계약으로 사용하지 않는다.
- 응답은 Angular `UserCollection`으로 cast하며 화면에는 `last_login_time`과
  `account`를 표시한다.

## 요청 계약

- URL: `rest-api/session-mgmt/v1/connect`
- header `access_token`: 활성 communication `session_id`
- body `client_session_id`: 같은 `session_id`
- `message_type`: `Request`
- `version`: `v1`
- `request_id`: `0`
- `request_type`: `connect`
- `work_type`: `local-user-session`
- `data`: `{}`

보호 요청이므로 로그인 응답 직후 조회하지 않는다. 공통 startup runtime에서
WebSocket 연결과 첫 keepalive 응답이 완료되어 communication session이 활성화된
뒤 실행해야 한다.

## 화면 계약

- read-only table이며 toolbar, action button, dialog가 없다.
- 열 순서는 `Last Login Time`, `User Name`이다.
- 첫 열 너비는 250px이다.
- Angular 실측 기준 header 32px, row 38px, cell padding 4px 17px,
  header text `#828a99`/600, body text `#000000`이다.
- `last_login_time`은 `yyyy-MM-DD HH:mm:ss`로 표시한다.

## 검증

- unit: `src/screens/security/userSessionModel.test.ts`
- fixture browser gate: `npm run qa:six-gen-user-session`
- authenticated live gate: `npm run qa:six-gen-user-session-live`
- live evidence: `D:/sixgen-react/.codex/visual-qa/user-session-live/latest-qa.json`

live gate는 Angular와 React의 URL, body, session field 일치, 응답 성공, 계정 행,
computed style, 1440x960 geometry를 함께 비교한다. 순차 로그인 때문에 로그인
시각 값 자체는 달라질 수 있으므로 timestamp 형식과 계정 순서를 비교한다.
