# Six-Gen React Software Image Parity

## Scope

- Angular source of truth:
  `sites/six-gen-flexible/contents.module.ts` and
  `EquipmentContentProviderModule`
- Active route: `/equipment/sofrware-image`
- Authority: `Equipment.Software Image`
- 데스크톱 viewport는 1440x960이며 모바일은 현재 이식 범위 밖이다.

## Request Contract

- 목록 조회와 메타데이터 변경은 `basedata/v1/software-image`를 사용하며 request type은
  `software-image`, work type은 `GetList`, `Add`, `Modify`, `Delete`다.
- 파일 업로드는 `softwareupgrade-mgmt/v1/image-upload`를 사용한다.
- 파일 삭제는 `softwareupgrade-mgmt/v1/image`와 work type `Delete`를 사용한다.
- 보호 요청은 통신 `session_id`를 헤더 `access_token`과 본문 `client_session_id`에 모두 사용한다.
- 업로드는 `access_token=session_id`를 사용하며 `client_session_id` HTTP 헤더는 추가하지 않는다.

## Angular Model Serialization

- `Software_image`는 `BaseData`를 확장한다. Modify는 다음 알려진 필드가 있으면 복제해
  유지한다: `id`, `name`, `description`, `creator`, `create_time`, `updater`,
  `update_time`, `delete_state`, `model`, `module`, `sw_type`, `image`,
  `file_path`, `version`, `isdefault`.
- Angular 복제 생성자는 알려진 필드만 복사하므로 Modify는 Angular `Software_image` 또는
  `BaseData` 모델에 속하지 않는 응답 전용 속성을 전달하지 않는다.
- Delete는 선택한 캐스팅된 응답 객체를 직접 전송하므로 React Delete 사전 검사는 원본 전송 응답 객체를 유지한다.
- Add는 Angular 클래스 기본값 `creator=0`, `updater=0`, `delete_state=false`,
  `model=0`, `sw_type=OS`, `isdefault=false`로 시작한다. 정의되지 않은 `id`와
  `file_path`는 값이 제공될 때까지 생략한다.
- 수정하지 않은 null, 빈 값, enum, 날짜 값은 그대로 유지한다. 적용할 모듈이 없으면
  Angular 편집기가 `module`을 `UNKNOWN`으로 설정한다.
- 새 기본 이미지를 선택하면 이전 기본 이미지에 대한 Modify를 먼저 순차 전송한 뒤
  편집한 이미지에 대한 Add 또는 Modify를 전송한다.

## UI Contract

- 정식 route 표기는 의도적으로 `sofrware-image`다.
- 컬럼은 Model 120px, Version 80px, Default 80px, Software Image 300px,
  가변 너비 Description, Action 90px다.
- 헤더 높이는 32px이며 글자색은 `rgb(130, 138, 153)`, 굵기는 600,
  배경색은 `rgb(247, 247, 247)`이다.
- 패널 반경은 8px이고 테두리는 `rgb(233, 233, 235)`이다.
- Search는 250x32px이다. Add, Column Settings, Download는 32x32px이고 간격은 3px이다.
- 목록이 비어 있어도 Download는 활성 상태를 유지하며, 행 수가 0이면 클릭 핸들러는 파일을 만들지 않고 반환한다.
- Add/Modify/Delete에는 Editable 권한이 필요하다. Search, 컬럼 표시와 CSV 기능은 View 권한으로 사용할 수 있다.

## Verification

- Fixture browser gate: `npm run qa:six-gen-software-image`
- Live read-only gate: `npm run qa:six-gen-software-image-live`
- Model/API tests:
  `vitest run src/screens/software-image/softwareImageModel.test.ts src/screens/software-image/softwareImageApi.test.ts --maxWorkers=1`
- Live evidence: `.codex/visual-qa/software-image-live/`
- 2026-07-19 실환경 백엔드는 0건을 반환했다. 실제 데이터가 있는 행과 실제 변경은
  미검증이며, 브라우저 요청 가로채기 QA와 계약 테스트로 확인 절차, 순서와 payload 사전 검사를 검증했다.
