# Six-Gen React Auth And Authority Parity

## 범위

- Angular 기준: `C:/khd/6gworkspace/UnifyEMS/uems_fe`
- React 대상: `D:/sixgen-react`
- 화면 기준: 데스크톱 `1440x960`; 모바일은 현재 범위 밖이다.
- 대상 계약: 로그인, 저장 세션 재연결, 사용자/역할 조회, 메뉴/route 권한, 현재 사용자 계정/암호 변경, 화면 잠금, 언어 메뉴, 로그아웃.

## Angular 기준 소스

- startup provider/sequence: `sites/six-gen-flexible/app.prepare.ts`
- 로그인 상태 전이: `src/app/common/login/user-login-process.ts`
- 권한 계산: `src/app/base/auth/auth.service.ts`
- 로그인 UI: `src/app/common/login/`
- toolbar 사용자 메뉴: `src/app/common/main-toolbar/`
- toolbar 메뉴 검색: `src/app/home/menu-shortcut-search/`,
  `src/app/home/menu-shortcut-search.service.ts`
- six-gen 제품 사용자 메뉴 provider: `src/app/content/contents/six-gen-flexible/provider/six-gen-flexible-content-provider.ts`
- User Session UI: `src/app/content/contents/security/content/user-session/user-session.component.ts`
- User Session API/model: `src/app/common/user-mgmt/user-session.service.ts`, `src/app/common/user-mgmt/user-session.ts`
- 계정 편집 UI: `src/app/content/contents/security-local-user/content/user-account/`
- 암호 변경 UI: `src/app/content/contents/security-local-user/content/change-password/`
- 사용자 관리 API: `src/app/content/contents/security-local-user/content/common/user-mgmt/user-mgmt-content.service.ts`
- 사용자 wire model: `src/app/base/basedata/user/user.ts`, `src/app/base/basedata/user/auto-user.ts`
- 언어 목록: `src/assets/lang/languages.json`
- 화면 잠금 startup/state: `src/app/app.component.ts`, `src/app/app.module.ts`
- 화면 잠금 암호 확인: `src/app/home/user-preference/screen-lock/screen-lock-panel.component.ts`
- 화면 잠금 preference: `src/app/home/user-preference/screen-lock/preference-screen-lock.component.ts`
- 비활동 감시: `framework_modules/library/html/inactivity-checker.ts`
- 통신 상태 notification: `framework_modules/framework/communication/communicationprocess.ts`
- 공통 message box: `framework_modules/wccontrol/wc-messagebox/`
- REST proxy: `proxy.config.json`

## 유지할 인증 계약

- 로그인과 로그아웃은 `session-mgmt/v1/connect`를 사용하고 기존 `URequest` body를 유지한다.
- 로그인 성공 응답의 `token`과 `session_id`는 역할이 다르다. 보호 API는 `access_token` header와 body의 `client_session_id`에 `session_id`를 사용한다. Angular interceptor에 없는 `client_session_id` HTTP header는 추가하지 않는다.
- `token`은 로그인 재인증과 keepalive token data를 위해 별도로 보존하며 보호 REST API의 `access_token`으로 사용하지 않는다.
- startup 순서는 `Login -> session_id로 WebSocket 연결 -> 첫 keepalive Response 대기 -> system environment -> current user -> role -> BaseDataLoadProcess`이다. master 사용자는 Angular `LocalAuthenticationProvider.getUserRoles()`와 동일하게 role 상세 조회를 생략한다.
- six-gen `BaseDataLoadProcess`는 `ne -> ne-group -> em -> site -> topology -> topology-item -> target-role`을 순차 `GetList`한다. 애플리케이션 응답 실패는 다음 loader로 진행하고 전송 예외는 startup을 중단한다.
- startup BaseData 응답은 공통 runtime cache에 보관한다. 화면 기본 조회는 cache를 사용하고 명시적으로 주입된 `RequestClient`는 직접 요청을 유지한다. `request-type`과 `request-type-Sync` WebSocket notification은 해당 cache를 무효화한다.
- 첫 keepalive는 `message_type: Response`, `request_type: keep-alive`를 모두 만족할 때만 인정한다. Notification이나 request envelope로 startup을 해제하지 않는다.
- 현재 사용자는 `basedata/v1/user`의 `Get` 요청으로 조회한다.
- 현재 사용자 Get data에는 Angular `User` 모델 기본 직렬화 필드인 `creator`, `updater`, `delete_state`, `status`, `is_master`, `account`를 유지한다.
- master가 아닌 사용자는 `basedata/v1/role`의 `Get_Detail` 요청으로 역할 privilege/target을 조회한다.
- 보호 API 호출은 활성 session의 `session_id`를 `access_token` header와 body `client_session_id`에 자동 직렬화한다.

## 유지할 현재 사용자 계정 계약

- 사용자 메뉴 순서는 `Edit Account`, `Change Password`, `User Preference`이며 현재 언어에 맞는 Angular label을 사용한다.
- 계정 편집은 startup에서 받은 전체 `User` wire model을 기준으로 편집 가능한 필드만 변경한다. 저장 순서는 `ConfirmPassword -> Modify`이다.
- 암호 변경은 현재 암호, 새 암호, 새 암호 확인을 모두 요구하며 불일치 시 API를 호출하지 않는다. 저장 순서는 `ConfirmPassword -> ChangePassword`이다.
- `ConfirmPassword`, `Modify`, `ChangePassword`는 모두 `POST rest-api/basedata/v1/user`를 사용한다.
- 모든 요청의 `access_token` header와 body `client_session_id`에는 활성 `session_id`를 사용한다.
- Angular `User`의 알려지지 않은 확장 필드와 null 값도 수정 요청에서 보존한다. 화면에서 편집한 값 외에는 새 기본값으로 덮어쓰지 않는다.
- `ConfirmPassword` 요청 data는 Angular 기본 모델 필드 `creator`, `updater`, `delete_state`, `status`, `is_master`, `account`, `password`를 유지한다.
- password를 포함할 수 있는 전체 current `User` wire model은 React 메모리에서만 유지하고 local/session storage에는 저장하지 않는다.

## 유지할 화면 잠금 계약

- 인증 응답만으로 보호 API를 시작하지 않는다. `session_id` WebSocket의 첫 keepalive 응답과 environment/user/role startup이 완료된 인증 세션에서 화면 잠금 profile을 읽는다.
- profile 요청은 `POST rest-api/system/v1/user-profile`, `work_type: Get`, `type: Security`, `target: ScreenLock`, `user_id: currentAccount`를 사용한다.
- profile 요청과 `ConfirmPassword` 요청의 `access_token` header 및 body `client_session_id`는 활성 `session_id`와 같아야 한다.
- six-gen BaseData의 `ConfirmPassword` 성공 `data`는 boolean `true`가 아니라 인증된 `User` 객체일 수 있다. Angular와 동일하게 truthy `data`를 성공으로 처리하되 `false`, `null`, 누락 데이터는 실패로 유지한다.
- 화면 잠금 기본값은 30분이며 선택값은 15, 30, 60, 90, 120분이다. `mousedown`과 `keydown`은 timeout 전까지만 timer를 재시작한다.
- timeout 시 embedded user 또는 server disconnected 상태이면 잠금을 열지 않는다. embedded 여부는 최초 redirect 경로에서 결정한 뒤 `AuthSession.isEmbeddedUser`로 보존한다.
- 잠금 시 local storage `ScreenLockState`에 `{ "isLocked": true }`를 저장한다. 암호 확인 성공 시 상태를 제거하고 timer를 reset한다.
- 잠금 상태로 앱을 다시 연 경우 Angular와 동일하게 backend Logout을 시도한 뒤 서버 결과와 무관하게 로컬 인증과 `ScreenLockState`를 제거한다.
- WebSocket runtime은 `Communication.State`의 Connected(11), Reconnecting(12), Disconnected(21)를 발행한다. 최종 Disconnected 시 잠금 dialog를 닫고 서버 연결 종료 message를 확인한 뒤 로컬 로그아웃한다.
- 잠금 dialog는 Angular `WCMessageBox`의 warning icon, `WARNING`, message area, password child, 중앙 OK button 구조와 활성 six-gen style token을 따른다.
- User Preference의 Default는 확인 dialog의 Yes 이후 30분으로 바뀐다. Apply/Default/label/unit/confirmation은 en/ko/ja resource와 일치해야 하며 request JSON이나 `client_session_id`를 화면에 표시하지 않는다.
- startup controller와 User Preference 화면은 세션별 ScreenLock profile cache를 공유해 동일 profile을 중복 조회하지 않는다.

## 유지할 권한 계약

- master 사용자는 모든 authority target을 `Editable`로 계산한다.
- non-master 사용자의 privilege가 없으면 Angular와 동일하게 `View`를 기본값으로 사용한다.
- `disable`은 `None`, `readOnly`는 `View`, `readWrite`는 `Editable`이다.
- 메뉴 visibility와 직접 route 접근을 같은 authority snapshot으로 검사한다.
- `None`인 route에 직접 접근하면 `/nopermission`으로 이동한다.
- 화면 버튼은 각 Angular content provider의 authority target과 visible/enabled/readonly 조건을 별도로 연결해야 한다. 메뉴 권한 구현만으로 버튼 parity가 완료되는 것은 아니다.

## 유지할 Security User Session 계약

- `/security/user-session`은 `UserSessionComponent`에서 활성화된 `UserMgmtContentService.getSessionList()`를 사용한다. 주입된 `UserSessionService.getList()` 호출은 주석 상태다.
- 요청 URL은 `session-mgmt/v1/connect`, request type은 `connect`, work type은 `local-user-session`이며 timeout은 Angular와 같은 60초다.
- 응답은 Angular `User` 모델이며 현재 table은 `last_login_time`과 `account`를 표시한다.
- table column은 `Last Login Time` 250px, `User Name` 200px 정의를 기준으로 하고 mutation toolbar/button/dialog는 없다.
- 상세 wire 및 table parity 기준은 `packs/nms-ems/knowledge/frontend/six-gen-react-user-session-parity.md`를 따른다.
- 보호 요청의 `access_token` header와 body `client_session_id`에는 활성 communication `session_id`를 사용하고 `client_session_id` HTTP header는 보내지 않는다.

## 유지할 메뉴 바로가기 검색 계약

- 데스크톱 main toolbar의 메뉴 검색은 28px collapsed 상태에서 280px input으로 확장되고 360px popup에 최대 10개 결과를 표시한다.
- shell, Preference, NE Configuration provider 결과를 provider 순서로 합치고 key로 중복을 제거한다. 검색은 대소문자와 공백을 제거한 substring 비교를 사용한다.
- authority가 없는 항목은 검색 source에 포함하지 않는다. Enter, Escape, ArrowUp, ArrowDown, mouse 선택과 blur 지연 동작을 유지한다.
- 항목 실행은 Angular와 동일하게 `openHandler`, 현재 URL 비교, route 이동, `urlAction`, 외부 창 순서를 따른다.
- NE Configuration COT/RT shortcut은 model과 inner menu를 pending 상태로 보존한다. 현재 model, 연관 COT/RT, 동일 site, 첫 matching model 순으로 target NE를 선택하고 NE와 menu가 모두 활성화된 뒤 pending 상태를 제거한다.

## React 구현 위치

- 인증 API: `src/runtime/authApi.ts`
- 인증 startup orchestration: `src/runtime/sessionBootstrap.ts`, `src/runtime/startupSequence.ts`
- BaseData startup/cache: `src/runtime/baseDataRuntime.ts`
- 인증 상태: `src/runtime/authContext.tsx`
- 세션 저장: `src/runtime/sessionStore.ts`
- 권한 계산: `src/runtime/authorityService.ts`
- REST client: `src/runtime/requestClient.ts`
- WebSocket client: `src/runtime/webSocketClient.ts`
- 메뉴/route authority: `src/domain/sixGenMenus.ts`, `src/components/shellMenuModel.ts`
- 메뉴 바로가기 검색: `src/components/MenuShortcutSearch.tsx`,
  `src/components/menuShortcutSearchModel.ts`,
  `src/screens/ne-configuration/neConfigurationShortcutRuntime.ts`
- 로그인 UI: `src/screens/login/LoginScreen.tsx`
- 현재 사용자 계정 상태/API: `src/screens/account/currentUserAccountModel.ts`, `src/screens/account/currentUserAccountApi.ts`
- 현재 사용자 계정 UI: `src/screens/account/CurrentUserAccountPanel.tsx`
- toolbar 사용자 메뉴 연결: `src/components/primitives.tsx`, `src/components/Shell.tsx`
- 화면 잠금 공통 상태/통신 계약: `src/runtime/screenLockRuntime.ts`, `src/runtime/webSocketClient.ts`
- 화면 잠금 세션 상태: `src/runtime/sessionStore.ts`, `src/runtime/authContext.tsx`
- 화면 잠금 profile/API: `src/screens/user-preference/screenLockModel.ts`
- 화면 잠금 dialog/controller: `src/components/WcMessageBox.tsx`, `src/components/ScreenLockController.tsx`
- 화면 잠금 preference UI: `src/screens/user-preference/UserPreferenceScreen.tsx`
- Security User Session: `src/screens/security/userSessionModel.ts`, `src/screens/security/UserSessionScreen.tsx`

## 검증

- `npm run typecheck:react`
- `npm run test:react`
- `npm run build:six-gen-flexible`
- `npm run build:six-gen-flexible-dev`
- `npm run qa:six-gen-auth`
- `npm run qa:six-gen-current-user-account`
- `npm run qa:six-gen-screen-lock`
- `npm run qa:six-gen-auth-live`
- `npm run qa:six-gen-authority`
- `npm run qa:six-gen-user-session`
- `npm run qa:six-gen-browser`
- `npm run qa:six-gen-dashboard-topology`
- `qa:six-gen-auth`는 mock API로 로그인부터 7개 BaseData loader까지의 순서와 wire contract를 검증한다.
- `qa:six-gen-current-user-account`는 사용자 메뉴 순서, 800px 데스크톱 sidebar, disabled 필드, 암호 확인 실패, 요청 순서, session field, 전체 User wire model 보존을 검증한다.
- `qa:six-gen-screen-lock`은 자동 잠금, Angular WCMessageBox 구조, 빈/오류/정상 암호 확인, session field, locked-startup Logout, `Communication.State` 종료 로그아웃, embedded session 잠금 금지를 검증한다.
- `qa:six-gen-auth-live`는 실제 backend에서 로그인, WebSocket, 첫 keepalive, environment/user/role, 7개 BaseData loader 순서와 session field 일치를 검증한다. 또한 current-user 요청의 Angular 기본 직렬화 필드를 확인하며 password, token, session_id 값은 산출물에 저장하지 않는다.
- Angular 기준 서버와 병렬 실행할 때는 `SIXGEN_REACT_URL=http://localhost:<port>/unms npm run qa:six-gen-auth-live`로 React origin을 지정한다. Topology live gate까지 함께 볼 때만 `SIXGEN_VERIFY_LIVE_TOPOLOGY=true`를 추가한다.
- `qa:six-gen-dashboard-topology`는 toolbar 검색의 collapsed/expanded/keyboard 상태, Angular 검색 순서와 10-result limit, COT/RT Port breadcrumb, target NE와 inner menu 활성화, Operation Log route 이동을 fixture browser에서 검증한다.
- 화면 단위 browser QA에서도 shell startup은 BaseData 7건, `Security/ScreenLock` profile Get, `ControlState/home.main-menu` profile Get을 실행한다. 화면 자체 API가 없는지 검증할 때 이 9건을 화면 local 요청으로 오인하지 않는다.
- `user-profile` 요청 수만 세지 말고 body의 `type`과 `target`으로 ScreenLock, main-menu, 화면 profile을 분류한다. User Preference mock도 `Security/ScreenLock`과 `ControlState/home.main-menu`에 서로 맞는 응답을 반환해야 한다.
- Icon main-menu fixture에서 menu label 기반 Playwright selector를 사용할 때는 side menu hover 후 조회한다. General fixture의 250px pinned geometry와 Icon fixture의 60px geometry를 같은 절대 좌표로 검사하지 않는다.

## 남은 전체 이식 위험

- 실제 backend가 기동된 환경에서 `qa:six-gen-auth-live`를 매 변경마다 통과해야 인증 parity를 유지할 수 있다.
- `Communication.State`의 첫 연결, 1회 reconnect, 최종 disconnect mapping은 이식했지만 장시간 reconnect와 token refresh는 실제 backend 장시간 검증이 필요하다.
- Angular `BaseDataLoadProcess` 7개 loader는 React 공통 startup/cache에 이식했다. 실제 backend에서 전체 loader 응답과 장시간 notification 갱신은 live 인증 환경 복구 후 재검증해야 한다.
- 각 화면의 toolbar/table/dialog/form 버튼 authority target 연결은 화면 단위로 계속 검증해야 한다.
- destructive action은 실제 backend에 자동 실행하지 않고 confirmation과 payload preflight까지만 비교한다.
