# NE Configuration Command Flow

## 목적
`NE Configuration > Configuration` 기능에서 장비 설정을 조회하거나 설정할 때 사용하는 Command 기반 흐름을 설명한다.

이 문서는 화면 배치 설명이 아니라 장비와 주고받는 명령 계약을 다룬다.

## 핵심 개념
- NE Configuration은 단순 form/table 화면이 아니라 장비 설정 read/write 기능이다.
- Command class는 장비에 보낼 조회/설정 명령과 field 구조를 미리 정의하는 계약이다.
- 화면 component는 직접 장비 protocol을 만들지 않고, 필요한 Command를 `NeCommandDefineArray`로 구성한다.
- refresh 계열 동작은 장비에서 값을 조회한다.
- set/delete 계열 동작은 form 값을 Command field에 반영한 뒤 장비에 설정 명령을 보낸다.

## 주요 코드 축
프로젝트마다 경로는 다를 수 있으나, Angular NMS FE에서는 다음 축을 우선 확인한다.

| 역할 | 확인 대상 |
| --- | --- |
| Command object/field 모델 | `src/app/common/ne-config/ne-command.ts` |
| Command define 모델 | `src/app/common/ne-config/ne-command-define.ts` |
| 장비별 Command class | `src/app/common/ne-config/commands/` |
| 공통 configuration panel | `src/app/content/contents/neconfiguration/content/configuration-ui/` |
| configuration provider/manager | `src/app/common/ne-config/content/ui-define/` |
| 제품/NE별 provider | `src/app/content/contents/neconfiguration/content/configuration/**/provider/` |
| 제품/NE별 화면 component | `src/app/content/contents/neconfiguration/content/configuration/**/content/` |

## Command 모델
- `NECommandClass`
  - command object와 object field를 담는 기본 모델이다.
  - 장비에서 읽거나 쓸 field 이름, 값, object instance 흐름을 확인한다.
- `AbstractNeCommandDefine`
  - 하나의 장비 명령 단위를 표현하는 기반 class다.
  - command define id, command object, protocol별 metadata를 확인한다.
- `NeCommandDefine`, `NeCommandDefineSnmp`, `NeCommandDefineRestAPI`, `NeCommandDefineDas`
  - SNMP, REST, DAS 등 장비 연동 방식에 따른 구현이다.
  - 새 기능에서 protocol이 무엇인지 먼저 확인한다.
- `NeCommandDefineArray`
  - 화면이 조회/설정에 사용할 command define 목록이다.
  - component의 `getCommands(...)`에서 구성되고 request 처리 흐름으로 전달된다.

## UI Panel 흐름
대부분의 설정 화면은 공통 `ConfigurationPanel` 계열을 따른다.

1. component가 `ConfigurationPanel` 또는 관련 base class를 상속한다.
2. 사용자가 `Refresh`를 누르거나 화면 진입 refresh가 발생한다.
3. component의 `getCommands(ButtonType.refresh)`가 조회용 command 목록을 만든다.
4. 공통 panel이 command를 장비 요청으로 처리한다.
5. response command 목록이 `display(...)`로 전달된다.
6. component가 command field 값을 form/table 상태로 매핑한다.

설정 흐름은 다음과 같다.

1. 사용자가 `Set`, `Apply`, `Delete` 계열 action을 실행한다.
2. component의 `getCommands(ButtonType.set)` 또는 관련 button type이 설정용 command 목록을 만든다.
3. form/table의 변경 값을 command field에 넣는다.
4. 공통 panel이 설정 command를 장비로 보낸다.
5. 설정 성공 후 필요한 경우 `onConfigurationChanged()` 또는 refresh 흐름으로 화면 값을 다시 조회한다.

## Provider/Menu 흐름
NE Configuration 내부 메뉴는 일반 top-level menu와 다르게 NE model/provider에 의해 결정된다.

1. `ConfigurationUIProviderModule`이 provider module을 manager에 등록한다.
2. `ConfigurationUIManagerService`가 NE code와 provider를 연결한다.
3. provider의 `createMenuNodes()`가 configuration tree menu를 만든다.
4. provider의 `addUIInfo(...)`가 menu id를 module/component 정보에 연결한다.
5. menu에는 `authorityCheck`가 걸릴 수 있으므로 권한 누락과 UI 누락을 구분한다.

## 새 NE Configuration 기능 추가 순서
1. 대상 NE model과 active site/product configuration을 확인한다.
2. 같은 장비 계열의 기존 component와 provider를 먼저 찾는다.
3. 필요한 장비 명령이 기존 `commands/`에 있는지 확인한다.
4. 없으면 장비 protocol과 field 정의를 확인한 뒤 Command class를 추가한다.
5. provider의 menu node와 `addUIInfo(...)` 연결이 필요한지 판단한다.
6. component에서는 `getCommands(...)`와 `display(...)`를 중심으로 read/write path를 구현한다.
7. form/table은 기존 `wc-*` 공통 컴포넌트와 validation 흐름을 우선 사용한다.
8. set/delete 계열이면 장비 영향, 권한, confirm, refresh-after-set 필요 여부를 명시한다.
9. 매뉴얼 화면이나 조작 설명이 바뀌면 manual update skill 사용 여부를 판단한다.

## 변경 전 확인
- 이 기능은 조회 전용인가, 장비 설정을 변경하는가?
- target NE code와 provider가 맞는가?
- COT/RT, DU/RU, shelf/slot/port 등 장비 모델별 차이가 있는가?
- command define id와 object field name이 기존 장비 규격과 일치하는가?
- `ButtonType.refresh`, `ButtonType.set`, `ButtonType.setDelete` 등 action 구분이 맞는가?
- response가 없거나 일부 field가 null/empty일 때 UI가 어떻게 표시되는가?
- set 이후 즉시 refresh해야 하는가, 장비 반영 지연을 고려해야 하는가?
- 권한이 없는 사용자는 menu/action이 어떻게 보이는가?

## 위험
- 잘못된 Command class나 field mapping은 장비에 다른 설정을 보낼 수 있다.
- read path와 write path를 섞으면 조회 화면에서 의도치 않은 설정 명령이 실행될 수 있다.
- NE model/provider 연결이 틀리면 특정 장비에서만 메뉴가 사라지거나 다른 화면이 열린다.
- 설정 후 refresh timing이 빠르면 장비 반영 전 값이 다시 표시될 수 있다.
- delete, initialize, discovery, command 실행류 action은 운영 영향이 있으므로 manual 조사 중 누르지 않는다.

## 검증 기준
- 기존 유사 화면의 `getCommands(...)`, `display(...)`, validation pattern과 비교했다.
- refresh path와 set path를 분리해서 설명했다.
- target NE model/provider/menu 연결을 확인했다.
- 설정 command가 있으면 영향 범위와 rollback/manual recovery 가능성을 보고했다.
- 실행하지 않은 장비 연동 검증은 통과했다고 말하지 않고 `Not verified`에 남겼다.
