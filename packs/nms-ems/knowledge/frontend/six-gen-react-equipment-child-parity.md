# Six-Gen React Equipment Child Parity

## 범위

- Angular 기준: `C:/khd/6gworkspace/UnifyEMS/uems_fe`
- React 대상: `D:/sixgen-react`
- 대상: Software Image, Software Upgrade, ShelfView
- 검증 화면: 데스크톱 `1440x960`; 모바일은 범위 밖이다.

## Software Image

- URL: `/equipment/software-image`
- 목록/CRUD: `basedata/v1/software-image`
- 파일 업로드: `softwareupgrade-mgmt/v1/image-upload`
- 파일 삭제: `softwareupgrade-mgmt/v1/image`
- COT/RT model code는 `10100`, `10101`이다.
- default image 변경은 기존 default를 `Modify(false)`한 뒤 대상 Add/Modify를 실행한다.
- React QA: `npm run qa:six-gen-software-image`
- 캡처: `D:/sixgen-react/.codex/visual-qa/desktop-software-image-parity.png`
- 이미지 diff: `97/100`.

## Software Upgrade

- URL: `/equipment/software-upgrade`
- 권한: `Equipment.Software Upgrade`의 `Editable`; Angular menu의 `.edit` 계약이다.
- API: `softwareupgrade-mgmt/v1/upgrade`
- work type: `Get`, `Check`, `Execute`, `Cancel`
- Check data: `neIDs`, `module_type`; COT/RT 기본 module type은 Angular와 같은 `HL7302`이다.
- COT/RT는 image bank auto-select 모델이다. ACTIVE가 아닌 standby bank를 NE별 target bank로 사용한다.
- Upgrade와 Reboot 전에 confirmation을 표시한다.
- Upgrade notification에서 `UpgradCompleted`인 NE만 Reboot target으로 유지한다.
- 상태 갱신은 polling이 아니라 WebSocket `software-upgrade` notification을 사용한다.
- Upgrade Complete는 Angular와 같은 Cancel request 후 Select Target으로 복귀한다.
- React QA: `npm run qa:six-gen-software-upgrade`
- 캡처: `D:/sixgen-react/.codex/visual-qa/desktop-software-upgrade-parity.png`
- 이미지 diff: `96/100`.

## ShelfView

- URL: `/equipment/shelf-view`
- active provider: six-gen COT/RT provider module
- 상태 API: `ne-configuration/v1/shelf-view-status`, work type `Get`
- 상태 data: `neID`, `isForceRefresh`
- 자동 갱신 주기는 Angular와 같은 10초이다.
- 명령 API: `ne-configuration/v1/process-ne-command`, work type `Set`
- 명령 ID: Unit Initialize `459`, Unit PM Initialize `460`, LED Test `461`, Admin Status `463`
- 명령 envelope는 `operationFunction: Configuration`, 빈
  `operationSubFunction`을 사용한다.
- command field는 `shelf-id`, `slot-id`, Admin Status의 `use`를 기존 `ObjectField` 구조로 직렬화한다.
- Admin Status를 선택하지 않은 Set은 Angular와 같이 `use: null`을
  직렬화한다. 임의로 `ACT` 기본값을 넣지 않는다.
- COT slot 20은 backend slot 0으로 변환한다.
- RT 기본 shelf는 chassis ID 20 미만을 사용하고, 확장 shelf는 surface install index + 18을 사용한다.
- Angular의 `assets/image/shelfview/six-gen` 70개 bitmap을 React public asset으로 그대로 사용한다.
- Angular `ShelfViewDefine`은 상태 응답과 별개인 정적 화면 계약이다.
  COT는 `1175x1511` base 위에 `(47 + 217*n, 180)` 좌표로 다섯 개
  `215x1175` `ShelfBlank`를 항상 표시한다. 상태 조회 실패 시에도 이
  기본 선반을 숨기면 안 된다.
- RT는 `1175x215` base 위에 PSU 2개, FAN 1개, `Blank` module 4개를
  기본 표시한다. RT 확장 surface 2~5는 해당 Surface/Chassis status가
  있을 때만 표시한다.
- COT `ModuleShelf` status가 `RT-Shelf`이면 shelf image와 PSU/FAN/module
  자식 정의로 교체한다. 자식 module은 `childStatuses`의 install index로
  찾고 status의 `index`는 명령 slot 표시/변환에 보존한다.
- `None` module은 숨기고, `I200`, `SWU`, `NPC`, `CMU`, `Blank2U`는 2U
  크기를 사용한다. `ModuleDisabled`와 config/actual key 불일치 overlay도
  Angular 상태 규칙을 유지한다.
- 선택 전에도 Admin Status, Set, Unit PM Initialize, Unit Initialize,
  LED Test는 Angular template과 같이 enabled 상태다. 기본 디자인만
  선택된 경우 detail은 current status가 없으므로 빈 unit 정보와
  `NOT-EQUIP`을 표시한다.
- React QA: `npm run qa:six-gen-shelf-view`
- Live Angular/React QA: `npm run qa:six-gen-shelf-view-live`
- 캡처: `D:/sixgen-react/.codex/visual-qa/desktop-shelf-view-parity.png`
- live 캡처: `desktop-shelf-view-angular-live.png`,
  `desktop-shelf-view-react-live.png`
- machine-readable live 결과:
  `D:/sixgen-react/.codex/visual-qa/shelf-view-live/latest-qa.json`
- 실제 `admin/admin` 검증에서 tree label/order, 보호 session field,
  Angular와 같은 NE building glyph, 248px tree, 400px detail, COT 기본
  선반 5개, chassis 크기 3px 이내를 비교한다. 현재 live COT 조회는
  Angular/React 모두 `status=1`,
  `errorMessage=null`이며 동일한 `neID=48`, `isForceRefresh=false` 요청과
  `605x778` chassis를 확인했다.

## 전체 검증

- `npm run typecheck:react`
- `npx vitest run --maxWorkers=1`
- `npm run build:six-gen-flexible-dev`
- `npm run qa:six-gen-browser`
- destructive command QA는 mock backend에서 confirmation과 payload까지만 검증하며 실제 NE에는 전송하지 않는다.

## 남은 위험

- 실제 backend 인증 세션에서 Software Upgrade 장시간 WebSocket reconnect/keepalive 중 상태 복구를 추가 검증해야 한다.
- live COT status 성공 경로와 기본 COT chassis는 확인했다. 다양한 실제
  payload를 사용하는 모델별 module/port/LED/alarm overlay와 장시간 갱신은
  추가 비교가 필요하다.
- 완전한 전체 이식은 아직 완료되지 않았다. 현재 문서는 Equipment 하위 세 화면의 기준만 고정한다.
