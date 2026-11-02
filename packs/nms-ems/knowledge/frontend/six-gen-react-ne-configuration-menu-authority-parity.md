# Six-Gen React NE Configuration inner-menu authority parity

## 기준

- 활성 범위는 `sites/six-gen-flexible/contents.module.ts`와 COT/RT
  `ConfigurationUISixGenFlexibleProvider` 메뉴다.
- 모바일과 NOC 전용, Cluster System Backup/Keycloak, product-disabled route는
  완료 gate에서 제외한다.

## 권한

- 일반 Configuration 및 Performance 메뉴는 `NeConfiguration`을 사용한다.
- IEEE 1588v2 세 메뉴와 Fault > Active Alarm은
  `NeConfiguration.Configuration.Active Alarm`을 사용한다.
- master는 Editable, `disable`은 None, `readWrite`는 Editable,
  누락/readOnly는 View다.
- None 메뉴는 제거하지 않고 disabled 상태로 유지하며 클릭을 막는다.
- 직접 denied route는 `/nopermission`으로 이동한다.
- 화면의 redirect 권한만 확인하면 React effect가 먼저 실행될 수 있다. API를
  사용하는 화면은 같은 authority code를 request target 생성 단계에도 전달해
  denied direct route와 `?ne-id=<id>` 진입에서 command 요청이 0건인지 검증한다.

## 기본 route

Angular는 Configuration root 진입 후 System을 기본 표시해도 URL을
`/ne-configuration/configuration`에 유지한다. React index도 System component를
직접 렌더링해야 하며 `/sixgen_system_configuration`으로 redirect하지 않는다.
Dashboard/Equipment shortcut의 `?ne-id=<id>`도 root URL에 보존한다.

## 실환경 증적

- 명령: `npm run qa:six-gen-ne-configuration-menu-authority-live`
- 실제 admin 로그인에서 Angular/React 모두 master, `system_COT`, 4개 그룹,
  15개 enabled node, `설정/System` 선택 상태가 일치했다.
- tree는 `250x845`, child row는 모두 28px이며 selected style과 root URL이
  일치한다.
- user/role startup session field가 일치하고 browser error 및 Set 요청은 없다.
- 증적: `D:/sixgen-react/.codex/visual-qa/ne-configuration-menu-authority-live/`
  및 `D:/sixgen-react/.omo/evidence/ne-configuration-menu-authority-live-parity.md`

## 제한

실환경은 master 계정만 검증했다. non-master의 4개 disabled node, click 차단,
직접 route denial은 fixture에서 검증한다.
