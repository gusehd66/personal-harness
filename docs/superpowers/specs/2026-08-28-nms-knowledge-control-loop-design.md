# NMS 지식 하네스 제어 루프 설계

## 목표

paperthin의 작은 판단 스킬과 변경 후 검증·학습 루프를 적용하되, NMS/EMS 지식은 범용 스킬에 복제하지 않고 `packs/nms-ems/`의 단일 원본으로 유지한다. 각 제품 프로젝트는 전역 설치 대신 해당 프로젝트의 `AGENTS.md`에서 이 하네스를 우선 참조한다.

## 비목표

- Codex 전역 skill catalogue 설치 또는 symlink 생성
- 제품 소스, 실행 설정, 실제 장비·DB·배포 상태 변경
- paperthin의 전체 skill catalogue 복제
- 장기 반복 개발 루프를 지금 추가하는 일

## 계층과 소유권

```text
프로젝트 AGENTS.md
  -> core/playbooks/harness-routing.md
      -> core/                         범용 판단·변경·결과 규칙
      -> packs/nms-ems/                NMS/EMS 지식과 운영 안전 규칙
      -> skills/                       작고 독립적인 제어 절차
      -> <PROJECT_ROOT>/.harness/runs/ 작업별 원시 증거
```

- `core/`는 모든 프로젝트에서 적용 가능한 정책과 라우팅 원본이다.
- `packs/nms-ems/`는 NMS/EMS 조건, 실제 근거를 다시 확인해야 하는 지식, 운영 위험 gate의 유일한 원본이다.
- `skills/`는 어떤 문서를 언제 읽고 어떤 검증을 수행할지 결정한다. NMS 지식을 재서술하지 않는다.
- 프로젝트 `.harness/runs/`는 일회성 계획·증거·회고이며 공식 규칙이나 Pack 지식의 대체물이 아니다.

## 스킬 카탈로그

| 스킬 | 역할 | 호출 기준 |
| --- | --- | --- |
| `safe-change` | 변경 요청의 기본 진입점. 범위·현재 흐름·NMS 적용 여부를 분류하고 최소 변경을 수행한다. | 코드, 설정, 문서, 자동화 변경 |
| `harness-verify` | 변경 후 실제 검증, 참조 일관성, 미검증 범위를 점검한다. | `safe-change` 종료 전 또는 하네스 문서·스킬 변경 후 |
| `harness-retro` | 검증된 교훈을 프로젝트, Core, Pack, skill 후보 중 가장 작은 표면으로 분류한다. | 의미 있는 작업 종료 후 |
| `harness-audit` | 하네스의 경로·중복·민감정보 후보·드리프트를 읽기 전용으로 감사한다. | 명시 감사 요청 |
| `harness-refresh` | 오래된 문서·스킬을 변경 이력 없이 깨끗한 현재본으로 정리한다. | 명시 정리·갱신 요청 |
| `angular-ui-harness` | NMS Angular UI 하네스 설계·분석만 수행한다. | 명시 UI 하네스 설계·분석 요청 |
| `backend-harness` | NMS Java backend 하네스 설계·분석만 수행한다. | 명시 backend 하네스 설계·분석 요청 |
| `manual-update` | NMS 운영 매뉴얼과 화면 캡처 갱신만 수행한다. | 명시 DOCX·캡처 갱신 요청 |

기존 공유 `codex-learning-loop`은 `harness-retro`로 대체한다. Codex 전역 경로의 동명 skill은 이 작업 범위 밖이며 변경하지 않는다.

## 요청 처리 흐름

1. 프로젝트 `AGENTS.md`는 `core/playbooks/harness-routing.md`를 먼저 읽도록 지시한다.
2. 라우팅은 하나의 주 스킬만 선택한다. 일반 변경은 `safe-change`, 읽기 전용 감사는 `harness-audit`, 전문 설계·매뉴얼 요청은 해당 전문 skill을 사용한다.
3. 주 스킬은 Core를 읽고, NMS/EMS 사실이 판단·검증에 영향을 주면 `packs/nms-ems/PACK.md`와 필요한 하위 문서를 읽는다.
4. NMS 작업은 source, 활성 구성, 로그, 런타임을 현재 근거로 확인한다. Pack 문서만으로 현재 동작을 단정하지 않는다.
5. DB, 장비 명령, scheduler, 인증, 배포 등 실제 write 위험은 Pack domain-safety 규칙과 사용자 승인 경계를 먼저 적용한다.
6. 작업 후 `harness-verify`로 실행한 실제 검증과 남은 미검증 범위를 분리한다.
7. 재사용 가치가 검증된 사실만 `harness-retro`로 승격 판단한다. 자동 승격하지 않는다.

## 지식 품질 기준

새 Pack 지식 또는 갱신된 Pack 지식은 본문에 다음을 명시한다.

- 적용 범위: product, site, 기능 또는 공통 영역
- 근거: source path, 설정, 로그, API 계약, 런타임 중 확인 근거
- 최신성: 마지막 확인 시점 또는 재확인이 필요한 조건
- 위험: read-only 또는 승인 필요한 write 영향

기존 지식을 이 형식으로 일괄 재작성하지 않는다. 실제 작업에서 해당 문서를 다시 검증하거나 갱신할 때만 최소 보완한다.

## 파일 구조 원칙

각 skill 폴더는 `SKILL.md` 하나만 둔다. `scripts/`는 반복적이고 결정적인 자동화가 검증상 필요한 경우에만 추가한다. `references/`의 NMS 지식은 `packs/nms-ems/knowledge/`로 옮긴다. UI 메타데이터가 필요하지 않은 `agents/openai.yaml`은 제거한다.

## 검증 기준

- 모든 skill은 UTF-8 `quick_validate.py`를 통과한다.
- 세 소비자 `AGENTS.md`가 `harness-routing.md`를 우선 참조한다.
- 현재 공유 skill folder에는 `SKILL.md` 이외의 파일이 남지 않는다.
- `PACK.md`, README, skills, 세 소비자에서 제거된 `codex-learning-loop` 경로 참조가 없다.
- 제품 build, browser, API, 실제 운영 write는 실행하지 않는다.
