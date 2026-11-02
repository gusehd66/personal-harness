# 범용 AI 하네스 재구성 설계

## 1. 목적

`C:\khd\ai-harness`를 NMS/EMS 전용 문서 묶음에서 다음 역할을 갖는 공용 하네스로 재구성한다.

- 모든 프로젝트에서 재사용하는 작업 원칙과 절차를 제공한다.
- NMS/EMS 지식은 버리지 않고 선택형 도메인 팩으로 분리한다.
- 스킬은 기본적으로 `SKILL.md` 한 파일로 시작한다.
- 실제 작업의 계획, 증거, 실패와 교훈은 스킬 폴더가 아니라 스킬을 실행한 프로젝트에 기록한다.
- 검토된 지식만 프로젝트 문서, 범용 Core 또는 NMS/EMS Pack으로 승격한다.
- 기존 UEMS 프로젝트가 참조하는 경로는 이관이 끝날 때까지 유지한다.

## 2. 설계 원칙

### 2.1 한 사실은 한 곳에서 관리한다

- 범용 작업 규칙의 원본은 `core/`이다.
- NMS/EMS 도메인 사실의 원본은 `packs/nms-ems/`이다.
- 프로젝트에만 해당하는 사실은 해당 프로젝트의 `AGENTS.md` 또는 `docs/`에 둔다.
- 스킬에는 실행에 필요한 최소 규칙만 포함하고 큰 지식 문서를 복제하지 않는다.
- 기존 경로는 내용의 두 번째 원본이 아니라 새 원본을 안내하는 호환 표면으로 축소한다.

### 2.2 스킬과 작업 기록을 분리한다

- `SKILL.md`는 언제 무엇을 어떤 순서로 수행할지 정의한다.
- `<PROJECT_ROOT>/.harness/runs/`는 특정 작업에서 실제로 무엇을 했고 무엇을 검증했는지 기록한다.
- 실행 기록은 작업 이력이며 공식 사양이 아니다.
- 재사용 가치가 검증된 교훈만 공식 지식 계층으로 승격한다.

### 2.3 작은 구조로 시작한다

- v0에서는 `scripts/`, registry, adapter, plugin 패키징을 만들지 않는다.
- 기계 검사가 반복적이고 결정적이어야 한다는 근거가 생길 때만 스크립트를 추가한다.
- 개별 스킬은 기본적으로 `SKILL.md`만 갖는다.
- 기존 `manual-update`처럼 복잡한 참조 자료가 실제로 필요한 스킬은 현재 reference를 보존할 수 있다.

### 2.4 호환성을 먼저 보장한다

- `harness/`, `nms-knowledge/`, `skills/`의 현재 소비 경로를 즉시 삭제하거나 일괄 이동하지 않는다.
- 새 구조를 병행 추가한 뒤 소비 프로젝트를 하나씩 이관한다.
- 외부 참조가 0건이고 새 경로가 검증된 뒤에만 기존 경로를 포인터로 축소하거나 제거한다.
- 프로젝트 로컬 `.ai/manual/`은 중앙 하네스 경로와 다른 자료이므로 유지한다.

## 3. v0 폴더 구조

```text
C:\khd\ai-harness\
├─ README.md
├─ AGENTS.md
├─ VERSION.md
│
├─ core\
│  ├─ policy\
│  │  ├─ global.md
│  │  ├─ security.md
│  │  └─ approval.md
│  ├─ playbooks\
│  │  ├─ safe-change.md
│  │  ├─ code-review.md
│  │  └─ long-term-learning.md
│  └─ templates\
│     └─ result-report.md
│
├─ packs\
│  └─ nms-ems\
│     ├─ PACK.md
│     ├─ policy\
│     │  └─ domain-safety.md
│     ├─ playbooks\
│     ├─ templates\
│     └─ knowledge\
│        ├─ frontend\
│        ├─ backend\
│        ├─ api\
│        ├─ components\
│        ├─ domain\
│        ├─ features\
│        └─ ui\
│
├─ skills\
│  ├─ safe-change\SKILL.md
│  ├─ harness-audit\SKILL.md
│  ├─ codex-learning-loop\SKILL.md
│  ├─ angular-ui-harness\SKILL.md
│  ├─ backend-harness\SKILL.md
│  └─ manual-update\SKILL.md
│
├─ docs\
│  └─ superpowers\specs\
│
├─ harness\                 # 기존 소비자 호환 경로
└─ nms-knowledge\           # 기존 소비자 호환 경로
```

`packs/angular`와 `packs/java`는 v0에 만들지 않는다. 현재 Angular와 Java 문서는 NMS 제품 흐름과 강하게 결합되어 있으므로 우선 NMS/EMS Pack에 보존한다. 다른 프로젝트에서 재사용되는 공통 규칙이 실제로 확인된 뒤 별도 팩으로 추출한다.

## 4. Core와 NMS/EMS Pack의 경계

### 4.1 Core에 들어가는 내용

- 변경 전에 현재 흐름과 실제 소비자를 확인한다.
- 최소 범위로 변경한다.
- 사용자 변경과 무관한 dirty work를 보존한다.
- 실제 실행한 테스트만 통과했다고 보고한다.
- 테스트, 브라우저/API 확인, 미검증 범위를 구분한다.
- 비밀정보와 개인정보를 문서 또는 실행 기록에 저장하지 않는다.
- 삭제, 배포, 광범위 이동 등 위험 작업은 승인 경계를 따른다.
- 작업 종료 시 프로젝트 지식, 공통 지식, 스킬 후보를 분류한다.

### 4.2 NMS/EMS Pack에 들어가는 내용

- NE, site, shelf, port, PM, alarm, event, sync, session 의미와 위험
- 멀티사이트 동시성, scheduler 중복, queue backpressure
- Angular site/file-replacement, WorkflowMap, RestContent, WC component 흐름
- Java NMS service, product profile, Keycloak/session 규칙
- Six-Gen, OpenHaul, my5G 등 제품별 검증 기록과 기준
- NMS 매뉴얼 갱신 절차와 프로젝트 로컬 `.ai/manual/` 연결

## 5. 스킬 설계

### 5.1 공통 형식

각 스킬은 다음 형식을 기본으로 한다.

```markdown
---
name: <kebab-name>
description: <호출 조건과 제외 조건이 드러나는 한 문장>
---

# Goal

# Workflow

# Rules

# Verification
```

스킬 폴더에는 `SKILL.md`만 둔다. 다음 조건이 확인될 때만 보조 자원을 추가한다.

- 동일한 결정적 검사가 반복됨: `scripts/`
- 스킬 본문이 지나치게 길어짐: `references/`
- 자동 호출을 제한하거나 UI metadata가 필요함: `agents/openai.yaml`
- 재사용할 실제 템플릿 파일이 필요함: `assets/`

### 5.2 v0 신규 스킬

#### `safe-change`

범용 변경 작업을 다음 순서로 수행한다.

1. 요청, 현재 흐름, 소비자와 제약을 확인한다.
2. 작업 규모를 경량 또는 전체 사이클로 분류한다.
3. 필요한 프로젝트 로컬 실행 기록을 생성한다.
4. 가장 작은 변경을 수행한다.
5. 집중 테스트와 가능한 실제 표면 검증을 수행한다.
6. 증거와 미검증 범위를 기록한다.
7. 교훈과 승격 후보를 정리한다.

별도의 `verify-change` 스킬은 v0에 만들지 않는다. 검증은 `safe-change`의 필수 단계로 포함한다.

#### `harness-audit`

읽기 전용으로 다음 항목을 점검한다.

- 오래된 `.ai/harness`, `.ai/nms-knowledge` 참조
- 존재하지 않거나 이식 불가능한 경로
- Core와 NMS/EMS Pack 경계 위반
- 중복되거나 모순되는 규칙
- 평문 비밀번호, 토큰, 실제 장비명 등 민감정보
- 스킬 원본과 실제 발견 위치의 중복 또는 드리프트
- 날짜가 지난 관찰 기록과 검증 기준

v0에서는 별도 스크립트를 사용하지 않고 Codex의 검색·파일 읽기 기능으로 검사한다.

### 5.3 기존 스킬의 역할

- `codex-learning-loop`: 실행 기록에서 검증된 교훈을 적절한 지식 계층으로 승격한다.
- `angular-ui-harness`: NMS/EMS Pack의 Angular UI 지식을 사용한다.
- `backend-harness`: NMS/EMS Pack의 Java backend 지식을 사용한다.
- `manual-update`: NMS/EMS Pack과 프로젝트 로컬 `.ai/manual/`을 사용한다.

기존 스킬은 새 구조가 검증되기 전까지 현재 경로에서 유지한다.

## 6. 프로젝트 로컬 실행 기록

실행 기록은 스킬을 호출한 프로젝트에 생성한다.

```text
<PROJECT_ROOT>\.harness\runs\
└─ <YYYY-MM-DD>-<work-name>\
   ├─ PLAN.local.md
   ├─ EVIDENCE.local.md
   └─ RETRO.local.md
```

### 6.1 전체 사이클

- `PLAN.local.md`: 목표, 범위, 현재 흐름, 작업 순서, 완료 기준
- `EVIDENCE.local.md`: 테스트 명령, 결과, 화면/API 확인, 산출물, 미검증 항목
- `RETRO.local.md`: 유지할 것, 실패 패턴, 다음 검증 기준, 승격 후보

### 6.2 경량 작업

작은 수정은 문서 세 개를 강제하지 않는다. `RETRO.local.md` 하나에 다음만 기록한다.

- 요청과 변경 요약
- 실행한 검증과 결과
- 미검증 범위
- 반복 가능한 교훈 또는 `없음`

### 6.3 실행 기록의 성격

- `.harness/runs/`는 기본적으로 프로젝트 `.gitignore`에 추가하는 로컬 작업 이력이다.
- 비밀번호, 토큰, 실제 세션 값, 개인정보를 기록하지 않는다.
- 실패 기록은 자동 삭제하지 않고 실패 상태 그대로 보존한다.
- 공식 규칙이나 프로젝트 지식으로 승격된 내용만 추적 문서에 반영한다.

## 7. 지식 승격 규칙

작업 종료 시 다음 순서로 분류한다.

1. 한 번의 작업에만 필요한 증거인가?
   - `.harness/runs/`에만 둔다.
2. 해당 프로젝트에서 반복되는 필수 규칙인가?
   - 프로젝트 `AGENTS.md` 또는 `docs/`로 승격한다.
3. 여러 프로젝트에 적용되는 범용 원칙인가?
   - `core/`로 승격한다.
4. 여러 NMS/EMS 프로젝트에 적용되는 지식인가?
   - `packs/nms-ems/`로 승격한다.
5. 독립적으로 반복 호출할 수 있는 절차인가?
   - 새 스킬을 제안하거나 기존 `SKILL.md`를 갱신한다.
6. 최신 상태가 자주 변하는 사실인가?
   - 하네스에 고정하지 않고 실행 시 다시 확인한다.

자동 승격은 하지 않는다. 근거와 영향 범위를 확인한 뒤 최소 변경으로 반영한다.

## 8. 오류와 안전 처리

- 프로젝트 루트를 확정할 수 없으면 실행 기록을 만들지 않고 루트 확인을 요청한다.
- 동일 작업명 폴더가 있으면 덮어쓰지 않고 새 식별자를 사용한다.
- 테스트가 실패하면 성공으로 바꾸거나 기록을 정리하지 않고 실패 증거와 의미를 남긴다.
- 브라우저/API 검증을 실행하지 못했으면 정적 검사와 구분해 기록한다.
- 대규모 이동이나 삭제 전에 복구 가능한 기준본과 소비자 참조 지도를 확보한다.
- 현재 발견된 평문 인증정보와 실제 장비명 후보는 이관 전에 비식별화 또는 제거한다.

## 9. 호환 이관

### 9.1 1단계: 기준본과 보안 정리

- 현재 전체 파일과 외부 참조 목록을 보존한다.
- 평문 인증정보와 실제 장비명 후보를 검토한다.
- 정상 Git 저장소 전환 방식과 초기 기준 커밋 범위를 정한다.

### 9.2 2단계: 새 구조 병행 추가

- `core/`, `packs/nms-ems/`, 신규 스킬 두 개를 추가한다.
- 기존 `harness/`, `nms-knowledge/`, 기존 스킬 경로는 유지한다.
- 기존 문서를 즉시 대량 이동하지 않는다.

### 9.3 3단계: 한 개 수직 흐름 검증

- 범용 프로젝트 하나에서 `safe-change`와 프로젝트 로컬 실행 기록을 검증한다.
- NMS 프로젝트 하나에서 Core와 NMS/EMS Pack 조합을 검증한다.
- 스킬 자동 선택, 문서 생성, 검증 보고, 지식 승격 판단을 확인한다.

### 9.4 4단계: 소비자별 이관

1. 결합도가 낮은 Playwright 프로젝트
2. `C:\khd\allproject\uems_fe`
3. `C:\khd\6gworkspace\UnifyEMS\uems_fe`

각 소비자는 필수 문서 접근, 스킬 발견, 결과 보고를 확인한 뒤 다음 소비자로 이동한다.

### 9.5 5단계: 호환 경로 축소

- 기존 경로 참조가 0건인지 다시 검색한다.
- `harness/`와 `nms-knowledge/`는 필요한 호환 포인터만 남긴다.
- 완전 제거는 별도 승인과 복구 기준이 있을 때만 수행한다.

## 10. 검증 기준

설계 구현 완료를 주장하려면 다음을 확인해야 한다.

- 범용 프로젝트는 NMS/EMS 지식을 읽지 않고 `safe-change`를 수행할 수 있다.
- NMS 프로젝트는 Core에 더해 NMS/EMS Pack의 위험 규칙을 적용한다.
- 스킬은 기본적으로 `SKILL.md`만으로 동작한다.
- 전체 작업과 경량 작업의 실행 기록이 각각 올바른 형태로 생성된다.
- 실행 기록에는 실제 수행한 검증과 미검증 범위가 구분된다.
- 검토되지 않은 실행 기록이 자동으로 Core 또는 Pack에 들어가지 않는다.
- 기존 UEMS 프로젝트의 현재 하네스 참조가 이관 전까지 깨지지 않는다.
- 동일 이름 스킬이 여러 발견 위치에서 중복 노출되지 않는다.
- 평문 인증정보와 프로젝트 고유 실제 값이 공용 문서에 남지 않는다.

## 11. v0 비범위

- 범용 실행 엔진 또는 별도 CLI 개발
- YAML 상태 머신과 evidence database
- CI/CD 자동화
- plugin 배포와 외부 사용자 설치 패키지
- `packs/angular`, `packs/java` 조기 분리
- 모든 기존 NMS 문서의 최신 제품 코드 재검증
- 기존 호환 경로의 즉시 삭제

이 항목은 실제 필요성이 확인된 뒤 후속 설계로 다룬다.
