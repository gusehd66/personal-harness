# 공지사항 기능 구조 지식

## 적용 경계

- 공지 기능 코드는 `src/app/content/common/notice`, `src/app/content/contents/notice`, `src/app/dbdata/unify_system/notice`에 공통 기능으로 둔다.
- 현재 my5G 런타임은 `angular.json`의 `default` file replacement를 사용하며, `sites/default/contents.module.ts`에서는 공지 Provider module만 등록한다.
- 공지사항은 `Preference`의 1Depth 하위 메뉴 하나로 제공하고, Provider module의 external route로 `/preference/notice`를 연결한다.
- 공지 목록은 기존 `WCSearchPanel`과 `WCTable`을 사용한다.
- 공지 목록 row 선택은 상세·수정 화면을 열지 않으며, 등록은 상단 버튼, 수정·삭제는 action 컬럼에서만 실행한다.
- 등록·수정은 `DBDataSideEditor` 기반 `app-content-sidebar`, 로그인 공지는 `WCDialog`를 사용한다.
- 공지 Angular 컴포넌트는 기존 프로젝트 관례에 맞춰 HTML을 TS의 인라인 `template`로 두고, 규모가 있는 스타일만 SCSS로 분리한다.
- 등록·수정 화면에는 로그인 팝업 미리보기를 제공하지 않으며, `popup_enabled` 설정과 실제 로그인 팝업 동작만 유지한다.
- `HomeComponent`의 로그인 시작 순서는 변경하지 않는다.

## 로그인 팝업

- 인증 및 기본 데이터 로딩 뒤 첫 `NavigationEnd`에서 세션당 한 번 조회한다.
- 로그인 팝업 자동 실행은 공지 관리 권한과 분리하며, 로그인 상태·게시 조건·자동 표시 공지 존재 여부·오늘 숨김 정책만 적용한다.
- 오늘 숨김은 `{productName}.notice.hide.{account}` 고정 키에 `yyyyMMdd` 날짜 값을 저장해 브라우저 `localStorage` 항목이 누적되지 않게 한다.
- 상단 공지사항 버튼은 공지 권한 없이 로그인한 모든 사용자에게 제공한다.
- 상단 우측 툴바 순서는 `HomeProvider.defineMainToolbarDisplay()`에서 공지 버튼을 로그아웃 구분선보다 먼저 등록해 `공지사항 → 구분선 → 로그아웃`으로 유지한다.
- active popup API는 `publish_enabled`와 게시 기간이 유효한 공지 전체를 반환한다.
- 자동 요청은 반환 목록에 `popup_enabled=true` 공지가 하나라도 있을 때 팝업을 열고, 열린 팝업에는 게시 중인 공지 전체를 표시한다.
- 상단 버튼으로 수동 조회할 때는 세션 처리 여부·오늘 숨김 값·`popup_enabled`를 무시하고 게시 중인 공지 전체를 표시한다.
- 수동 팝업에서도 `오늘 하루 보지 않기` 체크박스를 유지하며, 체크 후 닫으면 다음 자동 로그인 팝업부터 오늘 숨김 정책을 적용한다.
- 팝업 조회 실패는 로그만 남기고 로그인 및 라우팅을 차단하지 않는다.

## 데이터 경로

- FE는 기존 `system` 요청 채널과 `DBDataServiceBase` 패턴을 따른다.
- BE는 `uNMS_Common/uEMS.microservice/my5g.Service.System`의 `unify_system` worker에 추가한다.
- 공통 모델과 MyBatis는 `uNMS_Common/uEMS.plugin.my5g.Common`에 둔다.
- DB 업그레이드는 `my5g.Service.System/.../updatesql/update.sql`에 추가한다.
- 공지 기본 키와 FE/API 모델 필드명은 공통 `DBData` 규칙에 맞춰 `id`를 사용한다.
- `site_id = 0`은 전역 공지이며 현재 사이트는 `site_id IN (0, currentSiteId)`만 조회한다.

## 상태와 권한

- 게시 상태는 `publish_enabled`, `publish_start`, `publish_end`, DB 현재시각으로 계산한다.
- 상태 갱신용 스케줄러는 만들지 않는다.
- `notice_content` 메뉴 권한의 View 등급은 조회, Edit 등급은 등록·수정·삭제에 사용한다.
- Notice content provider 자체는 권한 필터에서 제외해 상단 공지사항 버튼을 생성하고, `Preference > 공지사항` 메뉴에만 `Notice` 권한을 적용한다.
- FE는 `AuthorityCode.edit`로 변경 버튼과 편집 입력을 제한한다.
- 공지 authority code 문자열은 `Notice`이며 `authority_id`는 문자열 해시값 `-1955822856`이다.
- BaseData `7.0.0.2`에서 누락된 기본 role만 admin·operator `readWrite(1)`, monitor `disable(2)`로 추가하며, 기존 설정과 사용자 정의 role은 변경하지 않는다.
- 변경 API는 입력값·사이트 범위·`update_time` 충돌을 검사한다.
- 현재 `my5g.Service.System` 요청 컨텍스트에는 세부 역할/권한 정보가 전달되지 않아 서버의 `notice_content` Edit 권한 검사는 구현 범위에 포함하지 못했다. Gateway/세션 계약에 권한 정보가 추가되면 서버에서도 동일 권한을 강제해야 한다.

## 1차 제외

- 첨부파일
- 사용자별 읽음 이력
- 실시간 Push
- 서버 동기화형 오늘 숨김

상세 구현 계획: `C:/Users/김현동/Documents/Codex/2026-07-14/nms-design/outputs/notice-feature-implementation-plan.md`
