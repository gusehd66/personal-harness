# 공용 하네스 요청 라우팅

## 목적

요청마다 하나의 주 skill을 선택하고, 범용 Core·NMS/EMS Pack·프로젝트 현재 근거를 필요한 범위에서만 읽도록 한다. 공용 문서는 판단 기준이며, 현재 프로젝트의 source, 활성 구성, 로그, 실제 실행 동작보다 우선하지 않는다.

프로젝트 `AGENTS.md`가 없으면 사용자 요청과 현재 프로젝트의 소스·설정을 기준으로 진행한다.

## 요청별 주 skill

| 요청 유형 | 주 skill | 먼저 읽을 문서 | NMS/EMS일 때 추가로 읽을 문서 |
| --- | --- | --- | --- |
| NMS/EMS BE Performance(PM) 분석·수정·리뷰 | `skills/nms-performance/SKILL.md` | 프로젝트 `AGENTS.md`가 있으면 확인, 해당 BE의 `docs/performance/index.md` | 대상 product 구조와 현재 소스; 운영 동작은 도메인 안전 규칙 |
| 코드, 설정, 문서, 자동화 변경 | `skills/safe-change/SKILL.md` | `core/policy/`, `core/playbooks/safe-change.md` | `packs/nms-ems/PACK.md`와 작업 유형별 Pack 문서 |
| 하네스 경로, 중복 규칙, 민감정보 후보 감사 | `skills/harness-audit/SKILL.md` | 이 문서와 소비자 `AGENTS.md`가 있으면 확인 | `packs/nms-ems/PACK.md` |
| NMS Angular UI 하네스 설계·분석 | `skills/angular-ui-harness/SKILL.md` | 이 문서 | `packs/nms-ems/playbooks/angular-ui-harness.md` |
| NMS Java backend 하네스 설계·분석 | `skills/backend-harness/SKILL.md` | 이 문서 | `packs/nms-ems/playbooks/backend-harness.md` |
| NMS 운영 매뉴얼 또는 화면 캡처 갱신 | `skills/manual-update/SKILL.md` | 이 문서 | `packs/nms-ems/knowledge/manual/`, UI·component knowledge |
| 오래된 문서 또는 skill을 현재본으로 정리 | `skills/harness-refresh/SKILL.md` | 대상 문서와 관련 canonical 문서 | 판단에 NMS 사실이 필요할 때만 Pack |

전문 UI/backend/manual 요청은 해당 전문 skill을 주 skill으로 사용한다. 일반 변경 요청에서 전문 skill을 중복 호출하지 말고 `safe-change`가 필요한 Pack playbook을 읽어 처리한다.

BE PM 요청은 범용 변경보다 `nms-performance`를 우선한다. 일반 CPU·메모리 성능 최적화와 FE 전용 작업에는 적용하지 않는다. 이 skill은 PM 전문 절차와 프로젝트별 구조 기록을 사용하고, 요청 유형에 따라 공통 변경·버그 수정·코드 리뷰 playbook을 읽는다. `safe-change` skill은 중복 호출하지 않는다.

## NMS/EMS 적용 판단

다음 중 하나라도 현재 판단 또는 검증에 영향을 주면 `packs/nms-ems/PACK.md`를 추가로 읽는다.

- NE, site, shelf, port, PM, alarm, event, sync, session 상태
- NMS/EMS Angular 화면, provider, route, authority, 공통 component
- NMS/EMS Java service, worker, process, REST, DB/Mongo, scheduler, 외부 adapter
- 운영 매뉴얼, 장비 제어, discovery, upgrade, 화면 캡처

그 외 범용 문서, 도구, 라이브러리, 비도메인 코드 작업은 Core와 프로젝트 문서만 사용한다.

## 실행과 종료

1. 현재 source, 활성 구성, 로그, 실제 소비자와 기존 변경을 먼저 확인한다.
2. 허용 범위에서 가장 작은 변경 또는 읽기 전용 분석을 수행한다.
3. 변경 또는 전문 분석을 마치기 전 `skills/harness-verify/SKILL.md`를 읽어 실제 검증, 실패, 미검증 범위를 분리한다.
4. 반복 가치가 있는 사실이 현재 근거로 확인된 경우에만 `skills/harness-retro/SKILL.md`를 읽어 가장 작은 저장 표면을 판단한다.
5. 프로젝트 작업 이력은 `<PROJECT_ROOT>/.harness/runs/`에 남기되, 실패 기록을 삭제하거나 성공으로 바꾸지 않는다.

## 운영 위험 gate

NMS/EMS 작업에서 DB/Mongo write, 장비 명령, discovery, upgrade, scheduler 실행, 인증·권한 변경, 배포, GitLab·ArgoCD·Kubernetes 변경이 관련되면 `packs/nms-ems/policy/domain-safety.md`를 먼저 읽는다. 실제 write는 사용자 승인이 명시되기 전에는 수행하지 않는다.
