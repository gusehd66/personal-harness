# Six-Gen React Software Upgrade Parity

## Scope

- Angular source of truth:
  `sites/six-gen-flexible/contents.module.ts`, the active Equipment provider,
  and `content/task/software-upgrade`.
- Active route: `/equipment/sofrware-upgrade`.
- Authority: Editable `Equipment.Software Upgrade`.
- 데스크톱 viewport는 1440x960이며 모바일은 이식 범위 밖이다.

## Startup And Request Contract

- user/role/BaseData에 의존하는 Software Upgrade 조회 전에 로그인 세션 초기화와 첫 WebSocket keepalive가 완료돼야 한다.
- 보호 REST 요청은 `session_id`를 헤더 `access_token`과 본문 필드 `client_session_id`에 모두 사용한다.
- 초기 상태는 `softwareupgrade-mgmt/v1/upgrade`, work `Get`, `data.user`를 사용한다.
- 이미지 조회는 `basedata/v1/software-image`, work `GetList`를 사용한다.
- Check는 request type `upgrade`, work `Check`와 선택한 `neIDs`, `module_type=HL7302`를 사용한다.

## Angular Serialization Contract

- Upgrade는 상속된 BaseData 필드와 null/빈 값/날짜 값을 수신한 그대로 포함한 원본 `Software_image` 응답 객체를 전송한다.
- work 수준의 `bank`는 Check 상태에서 찾은 현재 ACTIVE bank다.
- 각 Upgrade 대상은 자신의 STANDBY bank를 사용한다.
- Six-Gen Upgrade는 module 필드를 생략한다.
- Reboot는 image와 work 수준 module을 생략하고 work 수준에서 현재 ACTIVE bank를 유지하며,
  대상에는 반대 bank와 `module_type=None`, `module_id=0`을 전송한다.
- JSON에서 undefined 필드를 생략하는 것은 계약상 의미가 있다.

## UI Contract

- 단계는 Select, Check, Upgrade, Reboot다. 도구 모음 순서는 Previous, Next,
  Complete이며 각 버튼은 32x32px이고 간격은 3px이다.
- Select는 41px 행, 180px 레이블, 30px 컨트롤과 Angular 트리·체크박스 스프라이트를 사용한다.
- GroupNeTree는 권한이 있는 모든 그룹을 먼저 만들고 선택 모델에 따라 NE 노드만 필터링한다.
  빈 그룹은 계속 표시되고 선택할 수 있다.
- Site, site-group, group, NE 아이콘은 기존 EquipmentTable 매핑
  `icon-uems-site`, `icon-uems-site-group`, `fa fa-folder`, `fa fa-building-o`를 따른다.

## Verification And Safety

- Fixture gate: `npm run qa:six-gen-software-upgrade`.
- Live read-only gate: `npm run qa:six-gen-software-upgrade-live`.
- Evidence: `.codex/visual-qa/software-upgrade-live/`.
- 실환경 QA에서는 Upgrade, Reboot, Cancel 또는 장비 변경을 확정하면 안 된다.
  확인 절차와 정확한 payload 사전 검사는 가로챈 브라우저 요청과 집중 테스트로 검증한다.

