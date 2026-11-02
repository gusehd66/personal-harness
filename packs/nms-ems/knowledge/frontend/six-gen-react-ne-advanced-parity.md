# Six-Gen React NE Advanced Parity

## 범위

- 기준 provider: Six-Gen Flexible COT/RT Configuration UI provider
- 대상 route: `sixgen_synce`, `sixgen_service_interface`, `sixgen_time_configuration`
- 완료 범위는 `sites/six-gen-flexible/contents.module.ts`의 활성 provider 의존성 안에서만 판단한다.

## Angular 호환 규칙

### SyncE 빈 enum

장비의 command `442` 응답에 `global-enable`, `esmc-enable` 값이 없더라도 Angular `wc-radio`의 첫 항목 `selected=true`가 화면 draft를 `enable`로 만든다.

- 원본 응답 baseline은 빈 값으로 보존한다.
- 화면 draft만 `enable`로 초기화한다.
- 사용자가 즉시 Set하면 `esmc-enable`, `global-enable` 두 필드가 Angular 순서로 전송된다.

### Timezone 빈 값

command `468`의 `timezone-name`이 비어 있으면 Angular autocomplete는 검색 placeholder를 유지한다. React select가 첫 timezone을 자동 표시하거나 선택한 값으로 취급하면 안 된다.

- 표시 문자열은 언어별 timezone label을 사용하고 request에는 canonical timezone name을 저장한다.
- 닫힌 combobox에서 첫 `ArrowDown`은 목록의 첫 항목을 활성화해야 한다.

### 안정 상태와 표 규격

- Service Interface의 loading은 request 진행 중에만 표시하고, 빈 응답 완료 후에는 제거한다.
- live screenshot은 Angular `wc-loading`과 React loading overlay가 모두 사라진 뒤 저장한다.
- SyncE 표는 `1071px`, 열은 `100/100/120/150/100/100/401`, header는 `32px`, table panel은 `470px`를 기준으로 비교한다.
- Service UNI/NNI panel은 각각 `320px`, Time/NTP panel은 `93px/450px`를 기준으로 비교한다.

### Operation metadata 소유 경로

- 일반 Configuration panel의 Set과 delete command는 `operationFunction=Configuration` 및 현재 메뉴 subfunction을 보존한다.
- Service Interface UNI/NNI editor와 NTP create editor는 Angular `ConfigurationSidebar -> ConfigurationEditorBase` 경로를 사용한다.
- 이 sidebar editor의 `operationFunction` 기본값은 빈 문자열이며 별도 주입이 없으므로 React create/modify도 빈 `operationFunction/operationSubFunction`을 보존해야 한다.
- main panel 계약을 sidebar editor에 일괄 적용하면 operation log payload parity가 깨지므로 두 경로를 별도 fixture assertion으로 관리한다.

### Live 검증

- Angular와 React는 동일 NE/slot을 선택한다.
- command ID, `data`, `access_token=session_id`, `client_session_id=session_id`를 비교한다.
- 실제 live gate에서는 Set/Create/Delete를 보내지 않는다.
- mutation payload는 fixture interception으로만 검증한다.
- 반복 실행: `npm run qa:six-gen-ne-configuration-advanced-live`
- screenshot diff는 별도 visual QA 도구로 갱신하고 독립 검토한다. 프로젝트 live command 자체에는 아직 similarity threshold가 내장되어 있지 않다.
