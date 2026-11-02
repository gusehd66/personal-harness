# Six-Gen React Preference Storage Period Parity

## 범위

Angular `six-gen-flexible`의
`/preference/general/storage-period`를 React로 이식할 때 사용한다. 모바일은
범위 밖이며, 기준 viewport는 `1440x960`이다.

## Angular 기준 소스

- `data-storage-preference.component.ts`
- `db-storage-period.component.ts`
- `file-storage-period.component.ts`
- `ne-configbackup.component.ts`
- `preference-panel-server.ts`
- `system-config.service.ts`
- `db-storeage-config.ts`, `file-storeage-config.ts`, `ne-configbackup.ts`

## Request 계약

세 화면은 모두 `/unms/rest-api/system/v1/system-config`를 사용한다.

```text
message_type: Request
version: v1
request_id: 0
request_type: system-config
work_type: Get | Add | Modify
data: { site_id: 0, type: DbStoragePeriod | FileStoragePeriod | NeConfigBackup }
```

로그인 이후 header `access_token`과 body `client_session_id`는 모두 활성
communication `session_id`여야 한다.

## 직렬화 계약

- DB Storage는 alarm, event, pm, operation log, scheduled config backup의
  enable/value 10개 필드를 보존한다.
- File Storage는 loaded JSON의 property presence를 보존한다. backend 값에
  `db_backup_enable`, `db_backup`만 있으면 숨은 system log, NE backup, raw data
  기본 필드를 새로 만들지 않는다.
- NE Backup은 `numberOfBackupFile`만 직렬화한다. hash/debug 필드는 wire에
  포함하지 않는다.
- DB/File은 original과 같으면 mutation을 보내지 않는다.
- NE Backup은 Angular `equals` 결과를 그대로 따라 값이 같아도 Modify를
  만든다.

## UI 계약

- panel 높이: `210px`, `85px`, `87px`; gap `7px`; header `40px`
- label 폭 `180px`, numberbox 폭 `300px`, spinner 폭 `20px`
- DB 5개와 File 1개 numberbox에만 `Days`를 표시한다.
- checkbox는 Angular `icon_form_30.png` sprite 상태를 사용한다.
- spinner는 즉시 1회 반영 후 500ms 대기, 50ms 간격으로 반복한다.
- 빈 number input은 blur 때 기존 값을 복원한다.
- 화면별 측정값은 `src/styles.css`의 `--preference-storage-*` token으로
  선언하고 screen CSS에서 소비한다.

## 안전한 검증

```powershell
cd D:\sixgen-react
$env:SIXGEN_REACT_URL='http://127.0.0.1:4317/unms'
npm run qa:six-gen-preference-storage-period
npm run qa:six-gen-preference-storage-period-live
```

fixture gate는 Add/Modify를 intercept해 sparse File value, DB 전체 value, NE
value와 held spinner를 검증한다. live gate는 Angular와 React에 순차 로그인해
세 Get 계약, 화면 값/상태, local toggle/step/blank-blur 복원, unchanged DB
request suppression, geometry, computed style을 비교한다. backend retention을
바꾸는 live Modify는 실행하지 않는다.

## 완료 판단

이 화면 gate 통과는 전체 React 이식 완료가 아니다. 다른 활성 메뉴와 inner
menu가 route, authority, request, action, dialog/table/form, screenshot parity를
모두 통과하기 전에는 Angular runtime을 제거하지 않는다.
