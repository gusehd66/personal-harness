# Six-Gen React Security User Parity

## 적용 범위

Angular `Security > User`를 React로 이식할 때 active authentication provider와
User 모델 전체 직렬화를 보존하기 위한 기준이다.

## API 계약

- Six-Gen active provider: `LocalAuthenticationProvider`
- User 목록: `basedata/v1/user`, `GetList`
- Role 목록: `basedata/v1/user-role`, `GetList`
- Add/Modify/Password/Delete: `basedata/v1/user`
- 보호 요청의 `access_token` header와 body `client_session_id`는 모두 로그인
  `session_id`를 사용한다.
- Modify/Delete는 화면에 표시된 일부 field만 새로 만들지 않고 조회한 User
  wire object 전체를 보존한 뒤 변경 field만 덮어쓴다.
- `role_list`는 role id 배열이 아니라 Angular `ArrayList<Role>`이 직렬화한
  전체 Role object 배열을 보낸다.
- `usermanagement` 조회 경로는 Keycloak provider용 계약이다. Keycloak 화면은
  active Six-Gen 완료 범위에서 제외하지만, 공통 User/Role wire field와 session
  직렬화 계약은 변경하지 않는다.

## Add 기본값

Angular User 생성자와 동일하게 다음 값을 포함한다.

- `creator: 0`
- `updater: 0`
- `delete_state: false`
- `status: 0`
- `is_master: false`

## 동작 기준

- master user는 Password action만 허용하고 Edit/Delete는 비활성화한다.
- Add/Modify/Password는 우측 `800px` editor를 사용한다.
- Delete는 Angular confirmation 이후에만 전송한다.
- Password 변경은 현재 시간을 `password_time`에 직렬화하고 Modify로 보낸다.
- authority code는 `Security.Local User Mgmt`이며 View 상태에서는 변경 action을
  비활성화한다.

## 검증 기준

- fixture browser QA에서는 Add, Modify, Password, Delete 요청 순서와 URL,
  header, body, full model 보존을 검증한다.
- live backend에서는 Angular/React의 조회 URL, header/body session, 행/열/action
  상태를 비교하고 destructive mutation은 자동 실행하지 않는다.
- route 등록이나 화면 렌더링만으로 parity 완료 처리하지 않는다.
