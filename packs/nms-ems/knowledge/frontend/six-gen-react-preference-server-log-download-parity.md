# Six-Gen React Server Log Download Parity

## 범위

`D:/sixgen-react`의 `/preference/system/server-log-download` 화면을 Angular
`six-gen-flexible`의 `ServerLogDownloadComponent`와
`ServerLogDownloadService` 계약에 맞춘다. 모바일은 범위 밖이며 desktop
`1440 x 960`을 기준으로 검증한다.

## 요청 계약

- URL: `/unms/rest-api/system/v1/server-log`
- request type: `server-log`
- work type: `Get`
- data: `{ logType: 'uNMS', emID: 0 }`
- timeout: 600초
- 보호 요청의 `access_token`과 body `client_session_id`는 모두 활성
  communication `session_id`를 사용한다.
- 성공 Blob의 파일명은 `unms-logs.zip`이다.

## 화면과 동작 계약

- 활성 six-gen 메뉴에는 `uNMS` 한 행만 표시한다.
- Action 고정 열은 80px, Download action은 Font Awesome 18px이다.
- 클릭한 행은 `rgb(221, 235, 246)`으로 선택 상태를 유지한다.
- 단순 hover는 행을 선택하거나 배경색을 바꾸지 않는다.
- Download tooltip은 Angular `wc-tooltip`과 동일한 10px Segoe/Malgun
  font stack, 검정 배경 80% opacity, `61 x 28` geometry를 사용한다.
- HTTP 실패 문구는 Angular 영문 resource와 같은
  `Server log download failure.`이다.

## Cancel 계약

Angular Cancel은 진행 중인 HTTP 요청을 abort하지 않는다. 로딩만 즉시
숨기고 status를 canceled로 바꾼 뒤, 나중에 도착한 성공 결과를 무시한다.
React도 같은 순서를 사용해야 하며 `AbortController`로 요청을 끊으면 안 된다.

공통 `WcLoading`은 전체 viewport overlay, `Download...` 메시지, 64 x 34
Cancel 버튼, 20px spinner를 제공한다.

## 안전한 live 검증

```powershell
cd D:\sixgen-react
npm run qa:six-gen-preference-server-log-download-live
```

이 gate는 Angular와 React에 각각 로그인하고 모든 `server-log` 응답을
browser route에서 가로챈다. 첫 요청은 pending 상태에서 Cancel을 검증하고,
두 번째 요청은 fixture ZIP으로 파일명을 검증하며, 세 번째 요청은 HTTP
500으로 실패 MessageBox를 검증한다. 실제 backend log archive는 내려받지
않는다.

검증 항목은 URL/header/body/session, 메뉴와 표, 선택 상태, 로딩, Cancel,
성공 파일명, 실패 문구, tooltip style/geometry, desktop screenshot이다.
이 화면의 통과만으로 전체 Angular-to-React 이식 완료를 선언하지 않는다.
