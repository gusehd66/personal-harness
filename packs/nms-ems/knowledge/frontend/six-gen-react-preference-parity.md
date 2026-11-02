# Six-Gen React Preference Parity

## 범위

`D:/sixgen-react`의 활성 six-gen Preference 화면을 Angular
`PreferenceContentProviderModule` 계약과 비교할 때 사용한다.

활성 화면:

- `/preference/general/ftp-server`
- `/preference/general/database-backup`
- `/preference/general/storage-period`
- `/preference/fault/alarm-definition`
- `/preference/system/log-level`
- `/preference/system/server-log-download`
- `/preference/system-info`

## 공통 런타임 계약

- 모든 REST 요청은 React `RequestClient`를 통과한다.
- 로그인 이후 `access_token` header와 request body의
  `client_session_id`는 같은 Angular `session_id`를 사용한다.
- URL은 `/unms/rest-api/<service>/<version>/<request_type>`을 유지한다.
- validation, 성공, 실패 안내는 Angular `MessageBox`에 대응하는
  `PreferenceMessageDialog`를 사용한다. 화면에 request JSON이나 임시 status
  banner를 남기지 않는다.
- 메뉴와 Apply/action은 각 Preference authority의 `View/Editable` 상태를
  따른다.

## API 계약

| 화면 | service / request type | work type |
| --- | --- | --- |
| FTP Server | `system/v1/system-config` | `Get`, `Add`, `Modify` |
| DB Backup schedule | `system/v1/db-backup` | `Get`, `Modify` |
| DB Backup instant | `dbbackup-mgmt/v1/request-database-backup-user` | `Get` |
| Storage Retention | `system/v1/system-config` | `Get`, `Add`, `Modify` |
| SMTP Server (product-disabled direct route) | `system/v1/system-config` | `Get`, `Add`, `Modify` |
| Alarm Definition | `fault/v1/alarm-definition` | `GetList`, `Update` |
| Log Level | `to-ems/v1/log-level` | `Get`, changed-only `Set` |
| Server Log Download | `system/v1/server-log` | blob `Get` |

DB Backup의 `DateTime`은 offset이 포함된 Angular wire 형식을 사용하고,
`exec_time`은 `{ milliseconds }` 형태를 유지한다. 로드한 row의 화면 비노출
필드도 `Modify` payload에서 제거하지 않는다. `db-backup` notification은
`workType=Modify`일 때만 목록을 다시 조회한다. Log Level은 변경된 row만
`Site -> Domain -> log_level` payload에 포함한다.

Alarm Definition `Update`는 선택 row의 전체 Angular 모델을 전송한다.
`id`, `nms_event_code`, `event_name`, `alarm_type`, `ne_model`,
`use_nms_severity`, `nms_severity`, `category`, `auto_acknowledge`,
`nms_description`, `ne_model_name` 필드를 생략하지 않는다. NMS의 `CLUSTER`
category는 목록에서 제외하고 six-gen COT/RT 모델만 포함한 뒤 alarm type
이름과 event name 순으로 정렬한다.

six-gen의 `SixGenFlexibleContentProvider`는 `GeneralSMTPServer`를 disable하므로
SMTP Server를 활성 Preference 메뉴에 노출하지 않는다. direct route는
`Preference.SMTPServer` authority를 사용한다. 폼은
`SMTP Server/Port`, `User Name/User Password`, `From` 순서의 Angular
`wc-form-row` 배치를 유지한다. 저장 row에는 `id`, `site_id`,
`type: SMTPServer`, JSON `value`를 모두 포함한다. 성공 시 알림을 표시하지
않고 다시 `Get`하며, validation 실패나 값 미변경 시 mutation을 전송하지 않는다.
초기 `Get`이 실패하면 operation error를 표시하고 빈 form을 편집 가능하게
두지 않는다. SMTP/FTP form geometry와 색상은 공통 `--preference-form-*`
token을 사용한다.

Storage Retention의 세 Get은 `DbStoragePeriod`, `FileStoragePeriod`,
`NeConfigBackup`을 각각 조회한다. File Storage의 loaded JSON이
`db_backup_enable`, `db_backup`만 포함하면 React도 숨은 기본 필드를 추가하지
않고 같은 sparse JSON을 Modify에 유지한다. NE Backup Modify는
`numberOfBackupFile`만 직렬화한다. DB/File은 값이 같으면 요청하지 않지만,
NE Backup은 Angular 모델 equality 동작 때문에 값이 같아도 Modify를 만든다.

## 데스크톱 화면 계약

- Preference 내부 split: `250px / 7px / flexible`
- tree padding: `10px 20px`
- tree row: `28px`
- 선택 상태: 전체 row가 아니라 label span만 녹색
- DB Backup 열: `130/130/250/250/200/130/300/80px`
- DB Backup 일반 header/cell 좌측 padding: `17px`
- DB Backup Action: 오른쪽 고정 `80px`
- DB Backup editor의 table header, 5개 live row, panel/form/control/button,
  Activation radio는 `1440x960` live Angular/React gate에서 `0px` geometry
  delta를 유지한다.
- DB Backup Start/Exec는 활성 상태에서 DOM `readOnly=false`를 유지하지만
  keyboard 입력을 차단하고 picker로만 변경한다. 비활성 End는 readonly다.
  유효하지 않은 date/time은 request boundary에서 mutation 전에 거절한다.
- Alarm Definition 열: `200/280/220/130/130/180/100/140/80px`
- Alarm Definition Action: 오른쪽 고정 `80px`, read-only 권한에서는 제거하지
  않고 disabled 상태를 유지
- Alarm Definition editor: 오른쪽 `800px`, `Save/Close`
- Log Level hierarchy: `Site -> Domain -> Service -> Module`
- Log Level 선택 radio: Angular 녹색 accent
- System Info logo: `83px x 26px`
- Storage Retention panel: `210px/85px/87px`, panel gap `7px`, header `40px`
- Storage Retention numberbox: 폭 `300px`, 높이 `30px`, spinner 폭 `20px`;
  `Days`는 DB/File 6개에만 표시하고 NE Backup에는 표시하지 않는다.
- Storage Retention 전용 측정값은 `--preference-storage-*` token으로 선언하고
  화면 CSS에서 token을 소비한다.

모바일은 현재 이식 범위 밖이다.

## 검증

```powershell
cd D:\sixgen-react
npm run typecheck:react
npm run test:react
npm run build:six-gen-flexible-dev
npm run build:six-gen-flexible
$env:SIXGEN_REACT_URL='http://localhost:4202/unms'
node scripts/qa-sixgen-preference-ftp-server.mjs
node scripts/qa-sixgen-preference-database-backup.mjs
node scripts/qa-sixgen-preference-database-backup-live.mjs
node scripts/qa-sixgen-preference-storage-period.mjs
node scripts/qa-sixgen-preference-storage-period-live.mjs
node scripts/qa-sixgen-preference-smtp-server.mjs
node scripts/qa-sixgen-preference-alarm-definition.mjs
node scripts/qa-sixgen-preference-log-level.mjs
node scripts/qa-sixgen-preference-server-log-download.mjs
node scripts/qa-sixgen-preference-system-info.mjs
```

브라우저 gate는 `1440x960`에서 메뉴 선택, panel/table geometry, dialog,
mutation payload, `access_token/client_session_id` 일치까지 확인한다. DB Backup
Action과 Server Log header/body 경계는 bounding box로도 검증한다. DB Backup
live gate는 동일한 5개 row, Get URL/header/body, session field, editor state와
핵심 geometry의 `0px` delta를 검증한다. 실제 `Modify`와 Instant는 backend
상태를 변경하므로 전송하지 않고 fixture에서 confirmation과 payload까지만
검증한다.
Storage Retention live gate는 세 Get의 URL/header/body, 7개 값, 6개 checkbox,
6개 `Days`, 7개 spinner, Apply 상태, local toggle/step/blank-blur 복원,
unchanged DB request suppression, geometry, computed style을 비교한다. 실제
retention Modify는 보내지 않는다.
Alarm Definition gate는 고정 Action 열의 기본 viewport 가시성, editable action,
read-only disabled action, 필터, 전체 Update payload, CSV filename을 확인한다.

Angular와 같은 상태의 기준 스크린샷이 없으면 pixel-level 시각 parity는
완료로 판정하지 않는다. 다국어/CJK label clipping도 별도 locale 캡처 전에는
미검증으로 남긴다.

## 완료 판단

이 문서의 Preference 범위 통과는 Angular 전체 React 이식 완료를 뜻하지
않는다. 다른 provider와 전체 authenticated parity가 모두 통과하기 전에는
Angular 제거 또는 전체 포팅 완료로 처리하지 않는다.
