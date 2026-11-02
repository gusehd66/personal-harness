# Six-Gen Angular-to-React framework_modules baseline

## Source of truth

- Angular source: `C:\khd\6gworkspace\UnifyEMS\uems_fe\framework_modules`
- React target: `D:\sixgen-react\framework_modules`
- Active product context: Six-Gen Flexible

목표는 Angular 개발자가 구조의 연속성을 유지할 수 있도록 하는 것이다. React는 Angular 데코레이터, 템플릿, DI 바인딩과 생명주기 문법을 대체할 수 있지만 공통 모듈의 경계를 임의로 재설계해서는 안 된다.

## Actual Angular top-level structure

```text
framework_modules/
  framework/
  library/
  ngx-tempusdominus-bootstrap/
  wccontrol/
```

React 공통 모듈 루트는 의도적으로 `framework`, `library`, `wccontrol`로 제한한다. `ngx-tempusdominus-bootstrap`은 Angular 전용 외부 연동이므로 사용처를 기준으로 React 적용 여부를 결정해야 하며, 네 번째 React 프레임워크 루트가 아니다.

## Mapping policy

| Angular state | React decision |
| --- | --- |
| Active implementation source | `not-ported` 또는 `same-path-unverified`로 표시하고 책임과 사용처 매핑을 기록한 후에만 이식한다. 활성 코드만으로 런타임 사용처가 증명되지는 않는다. |
| Comment-only legacy source | 새 React 동작으로 만들지 않는다. 매핑 기록을 유지하거나 생략한다. |
| Reference-only source | 대상을 추가하기 전에 활성 사용처를 추적한다. |
| React-only adapter | `framework_modules`가 아닌 `src/app/react-adapter` 아래에 둔다. |
| React target-only implementation | 이동·교체·제거할 예외로 취급하며 동등성의 근거로 사용하지 않는다. |
| Angular WC SCSS | 원본 파일을 바이트 단위로 동일하게 유지하고 `wc-variables-30.scss`와 `wc-control.scss`를 컴파일한다. React 전용 WC CSS를 만들거나 변수를 중복하지 않는다. |

## Known exception requiring remediation

Angular `framework/authority/authority.ts`는 주석만 남은 레거시 소스다. 사용되지 않던 범용 React `Authority` 구현은 Angular의 1:1 이식이 아니어서 제거됐다. 이후 실제 Angular authority 사용처가 이식을 요구하면 먼저 사용처를 분류하고 일치하는 책임 경계에 구현을 배치한다.

## Required workflow

1. framework-modules 이식 묶음마다 시작 전에 React 저장소의 목록 생성 명령을 실행한다.
2. 생성된 목록에서 사용처가 확인된 `active-source` 파일만 선택한다.
3. 기존 React 위치에서 책임을 하나씩 이관하고 사용처 전환 중에는 호환성 export만 유지한다.
4. 각 묶음 작업 후 매핑 문서를 갱신하고 목록을 확인한다.
5. 폴더 이름이나 빌드 성공만으로 프레임워크 동등성을 주장하지 않는다.

공통 절차는 `packs/nms-ems/playbooks/angular-react-framework-modules-mapping.md`에 있다.

## Applied React ownership correction

React 프레임워크 구성 Context와 Provider는 `src/app/react-adapter/framework-provider.tsx`가 담당한다. `src/framework/core` 아래의 중복 React core 파일은 사용처와 공개 API 테스트를 루트 `framework_modules` 소유 경로로 전환한 후 제거됐다. 이는 후속 Angular core 이식을 위한 단일 대상 경로를 확립한 것이며 현재 React core 구현이 이미 Angular 로직과 동등하다는 뜻은 아니다.

## Verified library port

`framework_modules/library/data/event.ts`는 검증된 첫 번째 활성 Angular 라이브러리 이식이다. React 파일은 `EventArg`, `CancelableEvent`, `ICancelableEvent`, `EEventEmitter`의 공개 계약을 유지한다. React에 해당 Angular 런타임 의존성이 없으므로 Angular `EventEmitter`와 RxJS 구독만 동등한 네이티브 구독 객체로 대체한다. 기존 React 전용 `EventBus`는 사용되지 않아 제거됐다.

## Partial core process port

Angular에서 `FrameworkInitialize`, `SystemDataType`, `SystemDataManager`는 `framework/data/system-data-manager.ts`가 아닌 `framework/core/process.ts`가 담당한다. React 이식에서는 해당 책임을 `framework/core/process.ts`로 옮기고 BaseDataProcess가 이를 사용하도록 전환했다. Angular requester, 통신, 알림 및 컴포넌트 브리지 계약은 별도의 React 호환 이식이 필요하므로 이 파일은 부분 이식 상태다.

## Partial start sequence port

`framework/core/startsequence.ts`는 초기화, 순차 시작, 역순 중지와 상태 수신자를 포함한 Angular `StartSequenceType`, 상태 계약, `StartSequence`, `StartSequenceProcess`를 담당한다. 기존 React 작업 형식은 FrameworkService의 내부 어댑터일 뿐이다. Router와 product 전용 시퀀스 클래스는 framework_modules 밖에 둔다.

## Partial core common port

`framework/core/common.ts`는 Angular 애플리케이션, 알림, 요청 유형과 라이선스 결과 계약을 담당한다. `library/process/logger.ts`가 아직 이식되지 않아 `FrameworkLogger`는 임시 console 어댑터이며, framework가 wccontrol을 import하지 않도록 로딩 설정은 구조적으로 정의한다.

## Partial library process port

`library/process/process.ts`는 같은 경로에서 Angular `IDisposable`, `Disposable`, `IProcess`, `Process`, `IdentifierGenerator`, `PromiseList`, `PromiseHelper` 계약을 담당한다. `PromiseList`는 이식된 Angular `ArrayList` core를 상속한다. 나머지 library/process 파일을 이식할 때까지 이 파일을 `partial-port`로 분류한다.

## Partial library data port

`library/data/data.ts`는 같은 경로에 Angular의 `E_AccessControl`부터 `E_NumericType`까지의 enum 값, `IClonable` 계약과 `Compare`를 포함한다. `DataSearcher`와 나머지 데이터 헬퍼는 아직 이식되지 않았으므로 파일 이식 완료가 아닌 `partial-port` 상태다.

## Partial response status port

`framework/core/response-status.ts`는 Angular의 `E_ResponseStatus` 값을 담당한다. `AuthApi`는 로그인·라이선스 상태 번호를 중복하지 않고 해당 enum 멤버를 사용한다. Angular `ResourceCommon` 오류 리소스 계약을 사용할 수 있을 때까지 `ResponseStatus` 자체는 미이식 상태로 둔다.

## Partial collection port

`library/data/collection.ts`는 같은 경로에서 Angular `ArrayList`, `Queue`, `Entry`, `Hashtable`, `NumberKeyHashtable`, `ArrayListHashtable`, `HashMapHashtable`, `IDCollection`, `IDHashtable`, `Stack`을 담당한다. React 구현은 브라우저 `Map`을 사용하므로 Angular의 속성 해시 JSON 복원은 알려진 `partial-port` 미완료 항목이다.

## Partial code definition port

`library/data/code-definition.ts`는 같은 경로에서 Angular `CodeDefinition`, `CodeManager`, `IAuthorityCode`, `E_Authority`, `IAuthorityCheck`, `AuthorityCheck`를 담당한다. `SystemDataType`은 코드 생성을 중복하지 않고 이 공통 구현을 확장한다. Angular 사용처가 `CodeInformationHelper`를 import하지만 해당 심볼은 Angular 소스 파일 자체에 없으므로 별도의 미이식 불일치로 유지한다.

## Verified base data port

`library/data/basedata.ts`는 같은 경로에서 Angular `AbstractBaseData`와 `DataObject`를 담당한다. Angular 전역 `setPrototype` 헬퍼가 React 런타임으로 이식되지 않아 정적 캐스팅은 네이티브 `Object.setPrototypeOf`를 사용한다.

## Verified attribute port

`library/data/attribute.ts`는 같은 경로에서 Angular `ObjectAttribute`, `ObjectAttributeCollection`, `ValueItem`, `ValueItemCollection`을 담당한다. `StringBuilder`는 네이티브 문자열 결합으로, Angular 전역 캐스팅 헬퍼는 네이티브 `Object.setPrototypeOf`로 대체한다.

## Verified enum port

`library/data/enum.ts`는 같은 경로에서 Angular `EnumValueProvider`, `EnumValue`, `EnumValueList`, `EnumValueCollection`, `Enum`, `Flag`를 담당한다. 테스트는 숫자 enum 역매핑, 문자열 enum, 등록된 컬렉션과 플래그 연산을 검증한다.

## Login screen CSS source parity

React Login은 Angular `src/app/main/login/login.component.scss`를 같은 React 경로에 바이트 단위로 동일하게 복사하고 `LoginScreen.tsx`에서 import한다. 기존 React 전용 `src/screens/login/login.css`는 제거됐다. JSX는 Angular의 `login-panel`, `form-group-input`, `kc-form-options`, `kc-login` 계층을 사용한다. Vite가 전처리 중 Angular `::ng-deep`을 변환하므로 복사된 SCSS는 변경되지 않는다.

Angular Login은 `wc-checkbox` 대신 `icon_check.svg`와 `icon_checked.svg`를 사용하는 네이티브 `rememberMe` 입력을 의도적으로 사용한다. React WC 적용 점검은 Angular 근거가 있는 정확히 이 입력만 허용한다. 프레임워크 스타일 동등성 명령은 매핑된 Login SCSS의 해시도 확인한다.
