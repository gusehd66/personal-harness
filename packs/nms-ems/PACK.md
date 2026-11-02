# NMS/EMS Pack

## 적용 조건

다음 작업에는 Core에 더해 이 Pack을 적용한다.

- NE, site, shelf, port, PM, alarm, event, sync, session 상태를 읽거나 변경하는 작업
- NMS/EMS Angular 화면, site 설정, provider, route, authority, 공통 컴포넌트 작업
- NMS/EMS Java service, worker, process, REST, DB/Mongo, scheduler, 외부 adapter 작업
- 운영 매뉴얼, 화면 캡처, 장비 제어, 업그레이드, discovery 관련 작업

## 적용하지 않는 경우

다음 작업은 이 Pack 없이 `core/`와 필요한 프로젝트 문서만 사용한다.

- 범용 문서, 도구, 라이브러리, 비도메인 코드의 변경
- NMS/EMS 사실이 판단이나 검증 기준에 영향을 주지 않는 작은 작업

## 필수 안전 규칙

- `policy/domain-safety.md`를 먼저 읽고, 현재 코드·설정·로그를 근거로 실제 영향 범위를 확인한다.
- 실제 장비, 운영 데이터, DB, 인증, 배포, scheduler 실행은 read-only 확인과 승인 경계를 먼저 적용한다.
- NMS/EMS 지식은 현재 제품 소스·활성 구성·런타임과 다를 수 있으므로 문서만으로 동작을 단정하지 않는다.

## 작업 유형별 읽기 순서

| 작업 유형 | 먼저 읽을 문서 | 필요 시 이어서 읽을 Pack 문서 |
| --- | --- | --- |
| NMS UI 설계·분석 | `policy/domain-safety.md` | `packs/nms-ems/playbooks/angular-ui-harness.md`, `packs/nms-ems/knowledge/frontend/`, `packs/nms-ems/knowledge/ui/`, `packs/nms-ems/knowledge/components/` |
| NMS UI 구현·사진 기반 화면 변경 | `policy/domain-safety.md` | `packs/nms-ems/playbooks/frontend-feature.md`, `packs/nms-ems/knowledge/components/wc-usage-map.md`, 화면 유형별 `wc-*.md` |
| Backend | `policy/domain-safety.md` | `packs/nms-ems/playbooks/backend-harness.md`, `packs/nms-ems/knowledge/backend/`, `packs/nms-ems/knowledge/api/` |
| 매뉴얼 갱신 | `policy/domain-safety.md` | `skills/manual-update/SKILL.md`, `packs/nms-ems/knowledge/manual/`, `packs/nms-ems/knowledge/ui/`, `packs/nms-ems/knowledge/components/` |
| 도메인 진단 | `policy/domain-safety.md` | `packs/nms-ems/knowledge/domain/`, 관련 feature·API 문서 |

`angular-ui-harness`, `backend-harness`, `manual-update`는 명시적인 하네스 설계·분석·매뉴얼 갱신 요청에서만 주 skill으로 선택한다. 사진·목업·디자인 참조를 포함한 일반 NMS UI 구현은 `core/playbooks/harness-routing.md`의 `safe-change` 흐름에서 `frontend-feature.md`를 사용한다.

## 공통 UI 컴포넌트 우선 원칙

- 사진, 목업, 디자인 이미지는 layout·시각 기준일 뿐이다. 화면 markup, 상태, 이벤트, 권한, 스타일 계약은 현재 활성 프로젝트의 source와 기존 화면 소비자를 우선한다.
- NMS UI를 새로 구현하거나 변경하기 전, 화면 요소별 대응 `wc-*` 공통 컴포넌트와 실제 사용처를 확인한다. 대응 컴포넌트가 확인되면 화면 레벨에서 native HTML 또는 one-off custom component로 같은 역할을 다시 만들지 않는다.
- `wc-*` 내부가 native `<button>`, `<input>` 등을 렌더링하는 것은 공통 컴포넌트의 구현 책임이며 우회로 보지 않는다.
- native HTML은 기존 화면이 의도적으로 사용하는 경우, 대응 WC가 현재 source에 없는 경우, 또는 WC 내부 구현을 수정하는 경우에만 허용한다. 예외에는 확인한 source 또는 기존 consumer 근거를 남긴다.
- Pack의 selector·API 예시는 탐색 출발점이다. 실제 적용 전에는 대상 프로젝트의 활성 구성, component source, import 방식, 기존 consumer를 다시 확인한다.

## 지식 승격 기준

- 현재 작업의 1회성 증거는 프로젝트 `.harness/runs/`에 둔다.
- 여러 NMS/EMS 프로젝트에서 다시 검증된 사실은 이 Pack에 승격한다.
- 특정 제품·site의 예외는 공용 Pack에 일반화하지 않고 해당 프로젝트 문서에 둔다.
- 최신성이 중요한 사실은 실행 시 source, 로그, 공식 문서 또는 런타임에서 다시 확인한다.
