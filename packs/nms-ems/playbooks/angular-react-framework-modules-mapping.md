# Angular-to-React framework_modules mapping playbook

## Purpose

Angular NMS/EMS 프런트엔드를 React로 이식하거나 리팩터링할 때 Angular 프로젝트 구조를 개발자가 알아볼 수 있도록 유지해야 한다면 이 플레이북을 사용한다. 구조와 책임의 정본은 Angular 저장소이며 React에서는 컴포넌트 문법, 생명주기, 바인딩만 변경한다.

## Mandatory mapping gate

React 프레임워크 모듈을 생성·이동·재작성하기 전에 다음을 수행한다:

1. 활성 Angular 빌드/site 설정과 정확한 Angular 소스 루트를 확인한다.
2. 현재 편집 중인 파일뿐 아니라 전체 Angular `framework_modules` 트리의 목록을 작성한다.
3. 각 후보 파일의 Angular 경로, 활성 사용처, 책임, 소스 상태, React 대상 경로와 이식 결정을 기록한다.
4. 모든 파일에 대해 독립적인 두 분류를 기록한다:
   - Angular 구현 상태: `active-implementation`, `legacy-comment-only`, `reference-only`.
   - React 이식 상태: `not-ported`, `same-path-unverified`, `react-implementation-conflict`, `legacy-not-ported`, `reference-not-ported`, `react-only-unclassified`, `angular-specific-root` 중 하나.
   - `active-implementation`은 소스 파일에 활성 코드가 존재한다는 뜻이며, 활성 런타임 사용처가 있다는 주장은 아니다. 구현 전에 사용처를 추적한다.
5. 매핑 기록에서 Angular의 책임과 React의 대상 위치를 모두 확인하기 전에는 구현하지 않는다.

## Required directory contract

Angular에 루트 수준의 `framework_modules`가 있으면 React에서도 매핑된 모듈을 저장소 루트에 배치해야 한다:

```text
framework_modules/
  framework/
  library/
  wccontrol/
```

이 위치에는 대응하는 Angular 루트가 있는 항목만 둔다. React 전용 Context, hook, Provider, JSX 생명주기 어댑터와 애플리케이션 구성은 `framework_modules`가 아닌 `src/app/react-adapter` 또는 애플리케이션 계층에 둔다.

Angular `NgModule` 구현을 그대로 유지하지 않는다. 대신 책임, 유사한 이름, 공개 import 방향과 런타임 순서를 유지한다.

## Dependency and behavior rules

```text
sites / app / screens
          ↓
       wccontrol
          ↓
       framework
          ↓
        library
```

- `framework`는 `wccontrol`, 화면, site 또는 product 코드를 import하면 안 된다.
- `wccontrol`은 `framework`와 `library`에 의존할 수 있지만 product 메뉴, 장비 모델 또는 도메인 규칙에는 의존하면 안 된다.
- 주석만 남은 레거시 Angular 파일을 대응 폴더가 있다는 이유만으로 새 범용 React 서비스로 만들면 안 된다. 해당 경로에 이미 React 구현이 있다면 `react-implementation-conflict`로 분류하고 확장하지 않는다.
- `framework_modules`에서 발견한 React 전용 구현은 분류·이동·제거할 예외이며 Angular 동등성의 근거가 아니다.
- Angular `framework_modules/wccontrol` SCSS를 바이트 단위로 동일하게 유지하고 원본 변수 및 통합 진입 파일을 컴파일한다. Angular에 이미 있는 WC 규칙에 대해 React 전용 `styles/*.css`, 중복 WC 사용자 정의 속성 또는 화면 수준 재정의를 추가하지 않는다.
- 활성 화면 스타일은 정확한 Angular 컴포넌트 경로에서 한 화면씩 이식하고 복사한 SCSS를 바이트 단위로 동일하게 유지한다. Angular `::ng-deep` 제거 같은 프레임워크 문법 변환은 복사한 SCSS가 아닌 React 빌드 파이프라인에 둔다.
- Angular 템플릿에서 의도적인 사용이 확인된 경우에만 화면의 네이티브 컨트롤을 유지한다. 시각적으로 다른 React 컨트롤로 바꾸지 말고 WC 적용 점검에 정확한 경로와 컨트롤을 기록한다.
- `src/framework`와 `framework_modules/framework` 양쪽에 같은 책임을 유지하지 않는다. 이식 중에는 하나의 구현과 임시 호환성 export만 사용한다.

## Required evidence artifacts

광범위한 리팩터링 전에 React 저장소에 다음을 갖춘다:

1. 각 파일의 소스 해시, Angular 구현 상태와 React 이식 상태를 포함한 Angular framework-modules 생성 목록.
2. 루트 계약과 알려진 예외를 설명하는 사람이 읽을 수 있는 매핑 문서.
3. 소스 트리 또는 매핑 상태의 변경을 감지하는 확인 명령.
4. 매핑의 구현 계약이 정의된 후에만 작성하는 집중 테스트.

목록은 이식 입력 자료이며 완료 근거가 아니다. 상대 경로가 같은 파일도 책임과 사용처를 확인해야 한다.

## Review checklist

- Angular 소스 경로가 실제로 존재하고 활성 프로젝트 설정에 속하는가?
- 소스가 활성 구현, 주석만 남은 구현, 참조 전용 또는 Angular 전용 중 무엇인가?
- React 대상은 이름만 비슷한 것이 아니라 같은 책임을 갖는가?
- 명시적인 Angular 매핑 예외가 없는 React 전용 파일은 모두 `framework_modules` 밖에 있는가?
- WC 크기·상태, Font Awesome 색상, 표와 페이징을 React 전용 CSS 복사본이 아닌 변경하지 않은 Angular SCSS에서 불러오는가?
- `framework`, `library`, `wccontrol`에 site·장비·route·화면 업무 규칙이 없는가?
- 기존 `src/framework` 구현을 중복하지 않고 단일 소유 경로로 이관하는가?

## Six-Gen reference

`D:\sixgen-react`의 Angular 소스 루트는 `C:\khd\6gworkspace\UnifyEMS\uems_fe`를 사용한다. React 프로젝트는 루트 수준의 `framework_modules/framework`, `library`, `wccontrol` 구조를 유지해야 한다. 적용 기준과 알려진 예외는 `packs/nms-ems/knowledge/frontend/angular-react-framework-modules-mapping.md`를 참조한다.
