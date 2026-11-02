# WC Form Controls 컴포넌트

## 포함 대상
- `wc-form`
- `wc-form-row`
- `wc-form-label`
- `wc-form-value`
- `WCInnerForm`
- validation sample/pattern
- combobox, autocomplete, date/time picker, input, checkbox/radio/toggle 계열

## 사용 시점
- 설정 화면, 생성/수정 dialog, detail form, search/filter form을 수정할 때 사용한다.
- required label, validation, blank/null/duplicate 처리, readonly/disabled 상태를 바꿀 때 사용한다.
- Preference, Configuration, User Management, Equipment editor 화면을 다룰 때 사용한다.

## 코드 위치 힌트
- sample: `src/app/temp/wccontrol/wc-form/`
- sample: `src/app/temp/wccontrol/wc-validation/`
- sample: `src/app/temp/wccontrol/wc-combobox/`
- sample: `src/app/temp/wccontrol/wc-datetimepicker/`
- module import 예: `wccontrol/wc-form/wc-form.module`
- component import 예: `wccontrol/wc-form/wc-form.component`
- label import 예: `wccontrol/wc-form/wc-form-label.component`

## 대표 사용 화면
- `Equipment > ShelfView` 우측 Detail/Command 영역
- `NE Configuration > Configuration` 내부 System, PM Report, SyncE, IEEE 1588v2 설정 화면
- `Preference > FTP Server`
- `Preference > Storage Retention Period`
- `Security > User` 생성/수정 dialog 계열

## 변경 전 확인
- 기존 `wc-form-row`, `wc-form-label`, `wc-form-value` 구조를 유지한다.
- required 표시는 `wcRequire` 같은 기존 패턴을 먼저 확인한다.
- readonly/disabled는 시각 표시뿐 아니라 실제 값 변경 가능 여부도 확인한다.
- blank, null, whitespace, duplicate validation을 분리해서 확인한다.
- async validation이나 중복 체크가 있으면 timing과 race condition을 확인한다.
- 설정 화면은 값 변경이 운영 영향으로 이어질 수 있으므로 저장/적용 동작을 명확히 구분한다.

## 주의할 위험
- form control 변경은 화면 표시뿐 아니라 request payload를 바꿀 수 있다.
- validation 완화는 잘못된 NE/site/user/alarm 설정 저장으로 이어질 수 있다.
- disabled/readonly 누락은 매뉴얼 조사 중에도 값 변경을 유발할 수 있다.
- date/time picker 변경은 timezone, search period, PM/alarm history 조회 결과에 영향을 줄 수 있다.

## Anti-patterns
- 기존 validation flow를 건너뛰고 template 조건만 추가하는 것.
- null과 empty string을 같은 값으로 가정하는 것.
- duplicate check에서 blank 값을 비교 대상에 포함하는 것.
- 설정 화면에서 저장/적용 동작을 조회 동작처럼 취급하는 것.

