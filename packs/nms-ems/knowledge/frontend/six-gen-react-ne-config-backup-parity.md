# Six-Gen React NE Configuration ConfigBackup/Restore parity

## 범위

`sites/six-gen-flexible/contents.module.ts`에서 활성화된 COT/RT
`ConfigBackup/Restore` 화면의 Angular 계약을 React에 유지한다. NOC 전용
메뉴와 Cluster System Backup/Keycloak 화면은 이 완료 gate에 포함하지 않는다.

## 요청 계약

- 목록 URL: `nebackuprestore-mgmt/v1/config-backup-file`
- 목록 envelope: `request_type=config-backup-file`, `work_type=GetList`
- 목록 data: `{ne_id, type: "Manually"}`
- Angular `site_ne_id`의 기본값 `0`은 유효한 wire 값이다. JavaScript의 falsy
  처리로 선택 NE ID를 대신 넣으면 안 된다.
- 모든 보호 요청은 통신 `session_id`를 `access_token` header와
  `client_session_id` body에 동일하게 넣는다.
- backup, restore, download, delete, external restore도 같은 공통 request
  serializer와 communication session을 사용한다.

## 동작 계약

- refresh, manual backup, row selection, restore, download, delete, 파일 검증,
  external restore, confirmation, loading text, readonly authority를 Angular와
  동일하게 유지한다.
- 실제 장비/파일을 변경하는 동작은 자동 parity gate에서 실행하지 않는다.
  confirmation과 payload preflight까지만 검증한다.
- 비동기 목록 응답은 현재 NE/generation과 일치할 때만 화면에 반영한다.

## 데스크톱 UI 계약

- 기준 viewport: `1440x960`; 모바일은 범위 밖이다.
- 한국어 header: 그룹, 장비명, 모델, 버전, 백업시간, 파일명, 타입, 동작
- toolbar tooltip: `refresh`, `backup`, `restore`, `download`, `delete`
- Angular의 고정 열 너비, 테이블 frame, uploader track/Open File button,
  disabled/read-only 색상과 panel geometry를 보존한다.

## 검증

- Fixture gate: `npm run qa:six-gen-ne-config-backup`
- Authenticated live gate: `npm run qa:six-gen-ne-config-backup-live`
- live gate는 Angular와 React를 순차 로그인하고 같은 NE를 선택한 뒤 URL,
  header, body, session fields, response status, labels, button states, computed
  styles, geometry, screenshots를 비교한다.
- 결과는 `.codex/visual-qa/ne-config-backup-live/latest-qa.json`에 저장하며
  destructive request 수는 반드시 `0`이어야 한다.
