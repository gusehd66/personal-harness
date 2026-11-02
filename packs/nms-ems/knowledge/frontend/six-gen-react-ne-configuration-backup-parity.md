# Six-Gen React NE Configuration Config Backup Restore Parity

## 목적

Angular `six-gen-flexible`의 `NE Configuration > ConfigBackup/Restore`를 React로
이식할 때 route, 요청 모델, 권한, 확인 절차, 파일 검증을 같은 계약으로 유지한다.

## 기준 계약

- route: `/ne-configuration/backup-restore`
- child authority: `NeConfiguration.ConfigBackup/Restore`
- service: `nebackuprestore-mgmt`
- list: `config-backup-file/GetList`, data `{ne_id, type: 'Manually'}`
- backup: `request-neconfig-backup`, 숫자 `work_type`에 선택 NE ID 사용
- detail: `request-neconfig-detail/Get`
- restore: `request-neconfig-restore/null`
- download: `request-neconfig-download/GetList`
- delete: `config-backup-file/Delete`
- external restore: `request-neconfig-restore-external/null`
- upload: `ne-configuration/v1/local-neconfig-upload`

Angular 모델의 undefined field는 body에서 생략하고, `Auto_Config_backup_file`의
기본값 `ne_id: 0`, `type: 'Manually'`를 유지한다. 보호 요청은 활성 login의
`session_id`를 `access_token` header와 body `client_session_id`에 동일하게 넣는다.

## UI와 권한

- 화면 진입만으로 목록을 요청하지 않고 사용자가 Refresh를 실행할 때 조회한다.
- Backup, Restore, Download, Delete는 `Editable` 권한에서만 활성화한다.
- 저장 파일 Restore는 detail 조회 후 editor를 열고 confirmation 뒤 요청한다.
- Delete와 destructive restore는 자동 live 검증하지 않고 confirmation과 payload까지만 확인한다.
- six-gen 장비는 Angular validator mapping이 없으므로 external backup 파일을 허용하지 않는다.
- 기준 viewport는 `1440x960`이며 모바일은 현재 범위가 아니다.

## Live 검증 주의사항

로그인 이후 `page.goto`로 문서를 다시 열면 WebSocket이 닫혀 backend communication
session이 해제될 수 있다. live QA는 로그인 후 side menu를 클릭해 SPA 내부 이동을
유지하고, Refresh GET만 실행해 session field와 응답을 검증한다.

## 완료 게이트

- route/menu/title diff 0
- 요청 URL, request type, work type, 기본 필드, session field diff 0
- authority별 visible/enabled/disabled 상태 일치
- confirmation 취소 시 destructive request 미전송
- `1440x960` table/editor screenshot 검토
- 실제 인증 session에서 Refresh GET 성공과 destructive request 0건 확인
