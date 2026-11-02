# Six-Gen React Log Level Parity

## 범위

`D:/sixgen-react`의 `/preference/system/log-level`을 Angular
`six-gen-flexible`의 `LogLevelComponent` 및 `LogLevelService` 계약과 비교한다.
모바일은 범위 밖이며 desktop `1440 x 960`을 기준으로 한다.

## 요청 계약

- URL: `/unms/rest-api/to-ems/v1/log-level`
- request type: `log-level`
- Get: `work_type=Get`, `data={ domain: '', service: '' }`
- Set: 변경된 로그 레벨만 `SiteLogLevel -> DomainLogLevel -> LogLevel` 순서로 보낸다.
- 보호 요청의 `access_token`과 body `client_session_id`는 모두 활성화된
  communication `session_id`를 사용한다.
- Angular 응답에 `LogLevel.id`가 없으면 Set payload에도 `id`를 넣지 않는다.
  누락 값을 `0`으로 정규화하면 다른 모듈 인스턴스를 의미할 수 있으므로 금지한다.

## BaseData 및 트리 계약

- `site_id=0`은 `NOC`로 표시한다.
- 그 외 site는 startup `BaseDataLoadProcess`가 채운 `site` cache에서
  `id`로 찾고 `customer_id`를 표시한다. `Site <id>` 같은 React 전용 라벨을 만들지 않는다.
- 트리 순서는 Site, Domain, Service, Module 순서다.
- `id >= 0 && service !== type`인 항목만 `${type}-${id}` module row로 표시한다.
- service row와 module row의 라디오 값은 `fatal`, `error`, `warning`, `info`,
  `debug`, `trace` 중 하나이며 변경 행만 Set 대상이 된다.

## MessageBox 계약

Angular `WCMessageBox.createMessageBox`는 title bar를 숨기고 `INFO` icon/text,
400px dialog, 64 x 34 OK 버튼을 사용한다. React Preference 화면은 별도 dialog를
만들지 않고 공통 `WcMessageBox` primitive를 사용한다.

## 안전한 live 검증

```powershell
cd D:\sixgen-react
npm run qa:six-gen-preference-log-level-live
```

검증은 Angular와 React를 순차 로그인하고 각 세션을 Logout한다. Get은 실제 backend로
보내지만 Set은 browser route에서 가로채 성공 응답만 반환하므로 backend 로그 레벨을
변경하지 않는다. URL/header/body, live tree rows, radio edit, Apply, dialog, desktop
geometry와 screenshot을 비교한다.

이 화면의 gate 통과만으로 전체 Angular-to-React 이식 완료를 선언하지 않는다.
