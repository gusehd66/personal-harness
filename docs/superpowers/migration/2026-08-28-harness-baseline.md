# 하네스 이관 기준본

## 기준 시각

- 2026-08-28, Asia/Seoul

## 삭제 전 구조

- `harness/`: 정책, NMS/EMS 작업 playbook, 결과 템플릿
- `nms-knowledge/`: NMS/EMS 도메인, frontend, backend, API, UI, component 지식
- `skills/`: `angular-ui-harness`, `backend-harness`, `codex-learning-loop`, `manual-update`
- `docs/superpowers/`: 범용 하네스 재구성 설계와 구현 계획

## 기존 소비자와 참조 경로

| 소비자 | 현재 참조 | 이관 상태 |
| --- | --- | --- |
| `C:\khd\playwright\AGENTS.md` | `C:\khd\ai-harness`를 필요할 때 참고 | Core-first 이관 대상 |
| `C:\khd\allproject\uems_fe\AGENTS.md` | `<HARNESS_ROOT>/harness/`, `nms-knowledge/`, `skills/` | Core + Pack 병행 이관 대상 |
| `C:\khd\6gworkspace\UnifyEMS\uems_fe\AGENTS.md` | `<HARNESS_ROOT>/harness/`, `nms-knowledge/`, `skills/` | Core + Pack 병행 이관 대상 |

검색하지 않은 다른 프로젝트 또는 외부 문서는 기존 경로를 참조할 수 있으므로, 이 표는 전체 참조 목록이 아니다.

## 새 canonical 경로

- 범용 정책, 절차, 결과 형식: `core/`
- NMS/EMS 적용 조건과 도메인 안전 규칙: `packs/nms-ems/`
- 재사용 절차: `skills/`
- 작업별 원시 기록: 각 프로젝트의 `.harness/runs/`

## 최종 구조

- 범용 정책·절차·결과 형식: `core/`
- NMS/EMS 지식·전용 절차·전용 템플릿: `packs/nms-ems/`
- 재사용 절차: `skills/`
- 작업별 원시 기록: 각 프로젝트의 `.harness/runs/`

## 민감정보 점검 결과

- 정책·인증·세션·사용자 관리 설명에 `password`, `token`, `secret` 등의 용어가 다수 존재한다. 용어 자체는 보안 규칙 또는 구현 설명일 수 있으므로 값 노출 없이 구분 검토가 필요하다.
- 값 형식 후보는 아래 기존 parity 문서에서 확인됐다. 공개 배포 또는 대량 이관 전에는 실제 값 여부를 별도 승인된 환경에서 비식별화 검토한다.
  - `nms-knowledge/frontend/six-gen-react-auth-license-bootstrap-parity.md:56`
  - `nms-knowledge/frontend/six-gen-react-dashboard-topology-parity.md:69`
  - `nms-knowledge/frontend/six-gen-react-ne-advanced-parity.md:43`
  - `nms-knowledge/frontend/six-gen-react-ne-configuration-readonly-table-parity.md:92`
  - `nms-knowledge/frontend/six-gen-react-operation-log-parity.md:16`
  - `nms-knowledge/frontend/six-gen-react-porting-scope.md:44`
  - `nms-knowledge/frontend/six-gen-react-software-image-parity.md:20`
- 후보 문자열의 실제 값은 이 문서와 실행 기록에 복사하지 않는다.

## Git 상태

정상 Git 저장소 아님; 커밋 및 원격 변경 금지.

## 이관 차단 조건

- 검색하지 않은 소비자의 legacy 경로 참조
- 민감정보 후보의 비식별화 검토 미완료
- 새 Core 및 NMS/EMS Pack의 실제 소비자 검증 미완료
- 정상 Git 기준 및 초기 기준본 부재

## legacy 제거 이관 판정

- Core: canonical 사용
- NMS/EMS Pack: canonical 사용
- legacy `harness/`, `nms-knowledge/`: 제거 완료 (2026-08-28 사용자 수동 삭제 후 구조·링크·placeholder·skill 검증 통과)
