# Six-Gen React 포팅 완료 범위

## 기준 소스

`six-gen-flexible` React 포팅의 우선순위와 완료 판단 기준은 Angular의
`sites/six-gen-flexible/contents.module.ts`이다.

다음 항목은 서로 구분해서 관리한다.

- `CONTENTS_ROUTES`에 실제 등록된 route
- `@NgModule.imports`에 직접 등록된 provider module
- 활성 provider가 사용하는 COT/RT NE Configuration provider 의존성
- `SixGenFlexibleContentProvider`의 제품 필터 적용 후 노출되는 사용자 메뉴

React에서는 `src/domain/sixGenPortingScope.ts`가 실행 가능한 범위 계약이며,
`npm run qa:six-gen-scope`가 Angular baseline과 aggregate browser gate의 drift를
검사한다.

`sites/six-gen-flexible/app.prepare.ts`의 `addExcludeMenu`, `addStartSequence`,
`addBaseDataLoader` 호출은 baseline extractor가 직접 읽는다. React runtime은
생성 baseline을 그대로 사용하며 별도 startup 목록을 중복 관리하지 않는다.
Angular에 새 비활성 메뉴 ID가 추가되면 명시적인 제외 route 매핑을 검토하기
전까지 scope gate가 실패해야 한다.

## 완료 Gate 제외

다음 화면은 현재 Six-Gen 활성 제품 범위가 아니므로 포팅 우선순위와 aggregate
parity 완료 gate에서 제외한다.

- NOC 전용 Consign NOC
- 비활성 `Preference > Cluster` 하위 화면
- System Backup과 그 안의 Keycloak 상태
- 제품에서 비활성화한 Role, SMTP Server, Service Status Log, Performance Dashboard
- 그 밖에 `setMenuDisable(..., true)`로 비활성화한 화면

공통 구현이나 direct route가 React 저장소에 남아 있는 것은 허용하지만, 해당
화면의 구현 여부를 Six-Gen 완료 근거로 계산하지 않는다.

## 반드시 보존할 공통 계약

화면 제외는 공통 runtime 계약을 완화하지 않는다.

- Angular model/request 기본 필드, null/빈 값, enum/date 직렬화
- 보호 요청의 `access_token=session_id`
- request body의 `client_session_id=session_id`
- 로그인 후 WebSocket 연결과 첫 keepalive 응답 대기
- 첫 keepalive 이후 실행되는 실제 startup BaseData loader 의존성

## 공통 오류 경로

`sites/six-gen-flexible/app.routes.ts`의 401/403/404/500, Exception, Login
Error, No Permission은 `AuthGuard` 밖의 공통 기능이다. React에서도 이
경로를 인증 shell 안의 임시 패널로 구현하지 않고 공개 full-page route로
유지한다. No Permission의 Logout만 기존 세션의 `access_token`과
`client_session_id` 계약을 사용한다.

활성 화면의 REST/WebSocket parity는 위 공통 계약을 항상 통과해야 한다.
