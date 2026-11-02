# Universal AI Harness

이 폴더는 범용 Core와 선택형 NMS/EMS Pack을 제공하는 공용 AI 하네스다.

## 구조

- `core/`: 모든 프로젝트에 적용되는 정책, playbook, 결과 보고 템플릿
- `packs/nms-ems/`: NMS/EMS 적용 조건과 도메인 안전 규칙
- `skills/`: 반복 업무를 압축한 Codex skill 원본
- `packs/nms-ems/knowledge/`: NMS/EMS 도메인, frontend, backend, API, UI, component 지식

## NMS/EMS 지식
- `packs/nms-ems/knowledge/frontend/architecture.md`: Angular NMS FE 구조와 메뉴/provider/route/component/service/API 분석 순서
- `packs/nms-ems/knowledge/frontend/new-feature-flow.md`: 새 메뉴/화면/table/form 기능 추가 절차
- `packs/nms-ems/knowledge/frontend/six-gen-flexible-runtime.md`: `six-gen-flexible` active runtime, fileReplacement, provider/menu flow
- `packs/nms-ems/knowledge/domain/ne-configuration-command.md`: NE Configuration의 Command 기반 장비 조회/설정 흐름
- `packs/nms-ems/knowledge/components/wc-usage-map.md`: 실제 NMS FE에서 `wc-*` 공통 컴포넌트를 찾고 재사용하는 기준
- `packs/nms-ems/playbooks/frontend-feature.md`: FE 기능 개발 시 따라야 할 playbook

## Backend 분석 문서
- `packs/nms-ems/playbooks/backend-harness.md`: Java NMS backend 전체 기능 분석과 하네스 설계 기준
- `packs/nms-ems/templates/backend-harness-report.md`: backend 하네스 분석/설계 결과 보고 템플릿
- `packs/nms-ems/knowledge/backend/architecture.md`: Framework, Platform, uNMS Common, product 계층 구조
- `packs/nms-ems/knowledge/backend/service-startup.md`: `ServerStart_*`, `serverRunningConfig`, `runMicroService` 시작 흐름
- `packs/nms-ems/knowledge/backend/rest-content-flow.md`: `RestContentBook`, `RestContentManagementFunction`, request type 등록 흐름
- `packs/nms-ems/knowledge/backend/worker-process-flow.md`: `WorkflowMap`, worker, function, process 추적 기준
- `packs/nms-ems/knowledge/backend/db-and-migration.md`: MyBatis, MongoDB, DB upgrade, migration 분석 기준
- `packs/nms-ems/knowledge/backend/external-adapters.md`: Keycloak, GitLab, ArgoCD/Kubernetes, SNMP, NETCONF, REST adapter 분리 기준
- `packs/nms-ems/knowledge/backend/scheduler.md`: scheduler, PM, alarm, sync job 분석 기준
- `packs/nms-ems/knowledge/backend/services/index.md`: backend 서비스별 지식 문서 인덱스
- `packs/nms-ems/knowledge/backend/product-profiles.md`: my5G, openhaul, sixgen-ems product profile 분석 기준

## FE-BE 연결 분석 문서
- `packs/nms-ems/knowledge/api/conventions.md`: request/response model, error handling, pagination 등 API 규칙
- `packs/nms-ems/knowledge/api/fe-be-connection.md`: REST proxy/`access_token` 세션, login 시 WebSocket 연결, keepalive, session manager push 흐름

## 사용 방식

전역 `AGENTS.md`에서 공용 하네스의 `AGENTS.md`를 진입점으로 연결하고, 이어서 `core/playbooks/harness-routing.md`를 따른다. 프로젝트 `AGENTS.md`가 있으면 프로젝트별 특화 규칙을 함께 적용한다. 공용 문서는 판단·검증 기준이며 현재 프로젝트의 source, 활성 구성, 로그, 실제 실행 동작보다 우선하지 않는다.

- 모든 요청의 라우팅: `core/playbooks/harness-routing.md`를 먼저 사용한다.
- 범용 변경: `skills/safe-change/SKILL.md`를 주 skill으로 사용한다.
- NMS/EMS 변경: `safe-change` 또는 전문 skill에서 `packs/nms-ems/PACK.md`와 필요한 Pack 지식을 추가로 사용한다.
- 변경 후 검증: `skills/harness-verify/SKILL.md`를 사용한다.
- 검증된 재사용 지식의 승격 판단: `skills/harness-retro/SKILL.md`를 사용한다.
- 작업별 계획·증거·회고: 호출한 프로젝트의 `.harness/runs/`에 로컬로 남긴다.

## Skill 카탈로그

| 요청 | 주 skill |
| --- | --- |
| NMS/EMS BE Performance(PM) 분석·수정·리뷰 | `nms-performance` (product별 구조는 대상 BE의 `docs/performance/`) |
| 코드·설정·문서·자동화 변경 | `safe-change` |
| 하네스 읽기 전용 감사 | `harness-audit` |
| 변경 후 검증 | `harness-verify` |
| 완료 작업의 지식 승격 판단 | `harness-retro` |
| 문서·skill 현재본 정리 | `harness-refresh` |
| 명시 NMS UI/backend 하네스 설계 | `angular-ui-harness`, `backend-harness` |
| 명시 NMS 운영 매뉴얼 갱신 | `manual-update` |

모든 skill의 상세 호출 조건과 Core/Pack 읽기 순서는 `core/playbooks/harness-routing.md`가 단일 원본이다.

## 경로 표기 원칙

- `<HARNESS_ROOT>`: 이 하네스 repository 또는 로컬 설치 위치
- `<PROJECT_ROOT>`: 현재 작업 중인 제품 repository root
- `<FRONTEND_ROOT>`: Angular frontend root. 보통 `<PROJECT_ROOT>`와 같다.
- `<MANUAL_DOCX>`: 작업 대상 운영 매뉴얼 Word 문서

공용 하네스 문서에는 개인 PC 절대경로를 쓰지 않는다. 개인 로컬 경로는 각자의 로컬 설정, 환경 변수, 또는 commit하지 않는 local note에만 둔다.

팀 배포 단계에서는 이 폴더를 별도 Git repository로 승격하고, 각 프로젝트에서 submodule/subtree 또는 repo 상대경로로 참조하도록 전환한다.
