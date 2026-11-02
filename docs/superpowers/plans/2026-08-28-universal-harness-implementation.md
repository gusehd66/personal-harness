# 범용 Core + NMS/EMS Pack 하네스 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

**Goal:** 기존 NMS/EMS 중심 하네스를 범용 Core와 선택형 NMS/EMS Pack으로 병행 확장하고, 프로젝트 로컬 실행 기록과 최소 스킬 두 개를 도입한다.

**Architecture:** `core/`에는 모든 프로젝트에 적용되는 정책·절차·보고 형식을 둔다. `packs/nms-ems/`에는 NMS/EMS 위험 규칙과 기존 지식의 읽기 진입점을 둔다. 기존 `harness/`, `nms-knowledge/`, 기존 스킬은 소비자 호환성을 위해 유지하며, 새 경로가 실제 프로젝트에서 검증될 때까지 대량 이동하거나 삭제하지 않는다.

**Tech Stack:** Markdown, Codex skill discovery, Windows PowerShell, 기존 Angular/Java/NMS 문서

**Spec:** `C:\khd\ai-harness\docs\superpowers\specs\2026-08-28-universal-harness-design.md`

## Global Constraints

- 새 스킬은 각 폴더의 `SKILL.md` 한 파일로 시작한다. `scripts/`, registry, plugin 패키징, YAML 상태 머신을 만들지 않는다.
- `harness/`와 `nms-knowledge/`의 기존 파일을 이동·삭제·일괄 이름 변경하지 않는다.
- 범용 Core에는 NE, PM, alarm, sync, product/site 같은 NMS 전용 사실을 넣지 않는다.
- NMS/EMS Pack에는 일반적인 Git, 테스트 보고, 최소 변경 원칙을 복제하지 않는다.
- `.harness/runs/`에는 비밀번호, 토큰, cookie, 실제 세션값, 개인정보, 운영 DB 값 또는 실제 장비 식별값을 남기지 않는다.
- 테스트·브라우저·API 검증은 실제 실행한 것만 성공으로 기록한다. 실패 기록은 자동 삭제하거나 성공으로 바꾸지 않는다.
- 이 하네스 루트는 현재 정상 Git 저장소가 아니다. 구현 중 `git commit`, `git reset`, `git checkout`, 원격 push는 수행하지 않는다.
- 이 계획은 `C:\khd\ai-harness`와 명시된 세 개의 프로젝트 `AGENTS.md` 및 `.gitignore`만 대상으로 한다. 제품 소스 코드는 수정하지 않는다.

---

## File Structure

| 경로 | 작업 | 책임 |
| --- | --- | --- |
| `docs/superpowers/migration/2026-08-28-harness-baseline.md` | 생성 | 이관 전 파일·소비자·보안 점검 기준본 |
| `core/policy/{global,security,approval}.md` | 생성 | 모든 프로젝트에 적용되는 안전·보안·승인 정책 |
| `core/playbooks/{safe-change,code-review,long-term-learning}.md` | 생성 | 변경·리뷰·지식 승격의 범용 절차 |
| `core/templates/result-report.md` | 생성 | 범용 작업 완료 보고 형식 |
| `packs/nms-ems/PACK.md` | 생성 | NMS/EMS Pack의 범위, 적용 조건, 기존 지식 진입점 |
| `packs/nms-ems/policy/domain-safety.md` | 생성 | NMS/EMS의 동시성·세션·데이터 위험 규칙 |
| `skills/safe-change/SKILL.md` | 생성 | 범용 변경의 분석·기록·검증·승격 절차 |
| `skills/harness-audit/SKILL.md` | 생성 | 하네스 경로·중복·민감정보·드리프트 읽기 전용 점검 |
| `README.md`, `VERSION.md` | 수정 | 새 구조와 기존 호환 경로의 역할을 한 곳에서 안내 |
| `skills/{angular-ui-harness,backend-harness,manual-update,codex-learning-loop}/SKILL.md` | 수정 | NMS 작업과 지식 승격에서 Core 또는 Pack 진입점을 먼저 읽도록 라우팅 |
| `C:\khd\playwright\AGENTS.md`, `.gitignore` | 수정 | Core-only 실행 기록의 첫 번째 소비자 검증 |
| `C:\khd\allproject\uems_fe\AGENTS.md` | 수정 | Core + NMS/EMS Pack 읽기 경로 병행 안내 |
| `C:\khd\6gworkspace\UnifyEMS\uems_fe\AGENTS.md` | 수정 | Core + NMS/EMS Pack 읽기 경로 병행 안내 |

## Task 1: 이관 기준본과 안전 경계 고정

**Files:**
- Create: `docs/superpowers/migration/2026-08-28-harness-baseline.md`
- Read: `README.md`, `VERSION.md`, `harness/**`, `nms-knowledge/**`, `skills/**/SKILL.md`
- Read: `C:\khd\playwright\AGENTS.md`, `C:\khd\allproject\uems_fe\AGENTS.md`, `C:\khd\6gworkspace\UnifyEMS\uems_fe\AGENTS.md`

**Interfaces:**
- Consumes: 설계 문서의 2절, 3절, 8절, 9절
- Produces: 이후 작업이 참조할 `legacy retained`, `new canonical`, `consumer migration pending` 경계와 보안 점검 결과

- [x] **Step 1: 현재 하네스 파일 목록과 소비자 참조를 수집한다.**

  Run:

  ```powershell
  rg --files C:\khd\ai-harness | Sort-Object
  rg -n -i "C:[\\/]khd[\\/]ai-harness|<HARNESS_ROOT>|\.ai[\\/]harness|\.ai[\\/]nms-knowledge|\.ai[\\/]skills" C:\khd\playwright\AGENTS.md C:\khd\allproject\uems_fe\AGENTS.md C:\khd\6gworkspace\UnifyEMS\uems_fe\AGENTS.md
  ```

  Record each existing consumer path exactly; do not infer that an unscanned repository has no dependency.

- [x] **Step 2: 민감정보 후보를 값 노출 없이 검사한다.**

  Run:

  ```powershell
  rg -n -i --only-matching --glob '*.md' "password|passwd|token|api[ _-]?key|authorization:|cookie:|secret" C:\khd\ai-harness
  ```

  In the baseline, record only file path, line number, and `검토 필요` or `마스킹 확인`; never copy a matching value into the baseline or terminal output.

- [x] **Step 3: `2026-08-28-harness-baseline.md`를 작성한다.**

  Include these exact sections:

  ```markdown
  # 하네스 이관 기준본

  ## 기준 시각
  ## 현재 구조
  ## 기존 소비자와 참조 경로
  ## 새 canonical 경로
  ## 유지할 호환 경로
  ## 민감정보 점검 결과
  ## Git 상태
  ## 이관 차단 조건
  ```

  Set `Git 상태` to `정상 Git 저장소 아님; 커밋 및 원격 변경 금지`. Set `이관 차단 조건` to include unresolved consumer references and unresolved sensitive-information findings.

- [x] **Step 4: 기준본의 사실성을 검증한다.**

  Run:

  ```powershell
  Get-Content -LiteralPath C:\khd\ai-harness\docs\superpowers\migration\2026-08-28-harness-baseline.md -Raw
  git -C C:\khd\ai-harness rev-parse --is-inside-work-tree
  ```

  Expected: 기준본에는 세 소비자 경로와 기존 세 최상위 경로가 나오며, Git 명령은 현재 실패 상태를 보여야 한다. Git 실패를 계획 실패로 바꾸지 않는다.

## Task 2: 범용 Core 정책·절차·보고 형식 추가

**Files:**
- Create: `core/policy/global.md`
- Create: `core/policy/security.md`
- Create: `core/policy/approval.md`
- Create: `core/playbooks/safe-change.md`
- Create: `core/playbooks/code-review.md`
- Create: `core/playbooks/long-term-learning.md`
- Create: `core/templates/result-report.md`
- Modify: `README.md`
- Modify: `VERSION.md`

**Interfaces:**
- Consumes: `harness/policy/global.md`, `harness/policy/security.md`, `harness/policy/approval.md`, `harness/playbooks/code-review.md`, `harness/playbooks/long-term-learning.md`, `harness/templates/result-report.md`
- Produces: NMS 지식 없이도 읽을 수 있는 `core/`의 canonical 규칙과 `<HARNESS_ROOT>/core/...` 참조 경로

- [x] **Step 1: Core에 올릴 내용과 Pack에 남길 내용을 분리한다.**

  `core/policy/global.md`에는 현재 흐름 확인, 최소 변경, dirty work 보존, 실제 검증만 보고, 미검증 범위 명시, 새 라이브러리 최소화만 둔다.

  다음 표현은 Core에서 제거하고 Task 3의 Pack 정책으로만 둔다: `NE`, `site`, `shelf`, `port`, `PM`, `alarm`, `sync`, `scheduler`, `WorkflowMap`, `Keycloak`, `fileReplacement`, product 이름.

- [x] **Step 2: 세 정책 파일을 작성한다.**

  `core/policy/security.md`에는 secret·개인정보·운영 데이터 비기록, 민감값 마스킹, 운영 credential 미사용을 둔다. NMS/EMS 고유의 Keycloak/WebSocket 설명은 넣지 않는다.

  `core/policy/approval.md`에는 push, merge/release, deploy, DB write/migration, destructive filesystem action, production-like resource 변경 전 명시 승인 규칙을 둔다.

  `core/policy/global.md` 끝에는 `작업 종료 시 프로젝트 지식·Core 지식·도메인 Pack 지식·스킬 후보 중 하나로 분류한다`를 둔다.

- [x] **Step 3: 범용 playbook과 결과 템플릿을 작성한다.**

  `core/playbooks/safe-change.md`에는 다음 순서를 번호 목록으로 작성한다: 요청·현재 흐름·소비자 확인 → 경량/전체 사이클 분류 → 최소 변경 → 집중 검증 → 실제 증거와 미검증 기록 → 지식 승격 판단.

  `core/playbooks/code-review.md`의 심각도 순서는 `회귀`, `보안/데이터 노출`, `null·empty·timeout·retry·permission`, `검증 공백`, `유지보수 위험`으로 둔다.

  `core/playbooks/long-term-learning.md`은 저장 대상을 프로젝트 `AGENTS.md`/`docs`, `core`, 도메인 `packs`, `skills`, live lookup으로 분류한다. Codex memory는 recall layer이며 필수 규칙의 유일한 저장소가 아님을 명시한다.

  `core/templates/result-report.md`에는 `원인`, `수정`, `변경 파일`, `위험`, `검증`, `미검증`, `수동 테스트`, `지식/문서 업데이트`, `스킬 후보` 항목을 둔다.

- [x] **Step 4: README와 VERSION을 새 구조로 갱신한다.**

  README 첫 문장을 `범용 Core와 선택형 NMS/EMS Pack을 제공하는 공용 AI 하네스`로 바꾼다. 구조 표에는 `core/`, `packs/nms-ems/`, `skills/`, `harness/`와 `nms-knowledge/`의 호환 역할을 명시한다.

  VERSION에는 `2026-08-28 universal-core-v0` 항목을 추가하고, 이 버전에서 기존 경로는 제거하지 않았음을 기록한다.

- [x] **Step 5: Core가 NMS 전용 용어를 포함하지 않는지 검증한다.**

  Run:

  ```powershell
  rg -n -i "\bNE\b|\bPM\b|alarm|sync|scheduler|workflowmap|keycloak|filereplacement|six-gen|openhaul|my5g" C:\khd\ai-harness\core
  ```

  Expected: 결과 없음. `NMS/EMS Pack`이라는 분류명 외의 도메인 사실이 나오면 Task 2 문서를 수정한다.

## Task 3: NMS/EMS Pack 진입점과 도메인 안전 규칙 추가

**Files:**
- Create: `packs/nms-ems/PACK.md`
- Create: `packs/nms-ems/policy/domain-safety.md`
- Modify: `skills/angular-ui-harness/SKILL.md`
- Modify: `skills/backend-harness/SKILL.md`
- Modify: `skills/manual-update/SKILL.md`
- Modify: `skills/codex-learning-loop/SKILL.md`

**Interfaces:**
- Consumes: `nms-knowledge/**`, `harness/playbooks/angular-ui-harness.md`, `harness/playbooks/backend-harness.md`, `harness/playbooks/frontend-feature.md`
- Produces: NMS 작업이 Pack을 통해 위험 규칙과 현재 legacy 지식 경로를 선택하도록 하는 단일 진입점

- [x] **Step 1: `PACK.md`의 적용 조건과 읽기 순서를 작성한다.**

  `PACK.md`에는 다음 항목을 둔다.

  ```markdown
  # NMS/EMS Pack

  ## 적용 조건
  ## 적용하지 않는 경우
  ## 필수 안전 규칙
  ## 작업 유형별 읽기 순서
  ## Legacy knowledge 호환 상태
  ## 지식 승격 기준
  ```

  적용 조건에는 NMS/EMS의 NE, PM, alarm, sync, session, Angular site 설정, Java backend/service, 운영 매뉴얼을 포함한다. 적용하지 않는 경우에는 범용 문서·도구·비도메인 코드 작업을 적는다.

- [x] **Step 2: `domain-safety.md`에 NMS/EMS 고유 위험을 작성한다.**

  반드시 다음을 포함한다: multi-site concurrency, scheduler overlap, queue/backpressure, stale display/data loss, NE/site/shelf/port mapping, Keycloak/token/session/WebSocket mapping, DB/Mongo/장비 명령/ArgoCD/Kubernetes의 real-write 경계.

  각 위험에는 `확인할 현재 흐름`, `보고할 영향`, `승인 없이 하지 않을 동작`을 한 항목씩 둔다.

- [x] **Step 3: Pack과 legacy 지식의 관계를 명시한다.**

  대량 이동은 하지 않는다. `PACK.md`의 `Legacy knowledge 호환 상태` 표에 아래 경로와 책임을 정확히 넣는다.

  | 현재 경로 | 현재 책임 | v0 처리 |
  | --- | --- | --- |
  | `nms-knowledge/` | 제품/도메인/UI/API 사실 | 소비자 이관 전 유지 |
  | `harness/playbooks/angular-ui-harness.md` | Angular NMS 분석 절차 | Pack에서 라우팅 |
  | `harness/playbooks/backend-harness.md` | Java NMS 분석 절차 | Pack에서 라우팅 |
  | `harness/playbooks/frontend-feature.md` | NMS FE 확장 절차 | Pack에서 라우팅 |

- [x] **Step 4: 기존 NMS 스킬의 참조 순서를 최소 수정한다.**

  각 기존 스킬의 `필수 참조` 첫 줄에 `<HARNESS_ROOT>/packs/nms-ems/PACK.md`를 추가한다. 그 뒤에는 현재의 구체적 legacy 문서 목록을 그대로 둔다.

  `manual-update`에는 Pack 경로를 추가하되, 프로젝트 로컬 `.ai/manual/`은 운영 매뉴얼 산출물이라 유지한다는 문장을 보존한다.

  `codex-learning-loop`은 `<HARNESS_ROOT>/core/playbooks/long-term-learning.md`를 기준 문서로 사용하고, 범용 규칙은 `core/`, NMS/EMS 지식은 `packs/nms-ems/`, 프로젝트별 사실은 프로젝트 `AGENTS.md` 또는 `docs/`로 분류하도록 갱신한다.

- [x] **Step 5: NMS Pack이 Core 규칙을 복제하지 않는지 검증한다.**

  Run:

  ```powershell
  Get-Content -LiteralPath C:\khd\ai-harness\packs\nms-ems\PACK.md -Raw
  Get-Content -LiteralPath C:\khd\ai-harness\packs\nms-ems\policy\domain-safety.md -Raw
  rg -n "packs/nms-ems/PACK.md" C:\khd\ai-harness\skills\angular-ui-harness\SKILL.md C:\khd\ai-harness\skills\backend-harness\SKILL.md C:\khd\ai-harness\skills\manual-update\SKILL.md
  rg -n "core/playbooks/long-term-learning.md|packs/nms-ems" C:\khd\ai-harness\skills\codex-learning-loop\SKILL.md
  ```

  Expected: 세 NMS 스킬은 Pack 진입점을, `codex-learning-loop`은 Core 지식 승격 경로를 참조하며, 기존 개별 NMS 문서는 삭제되지 않는다.

## Task 4: 최소 스킬과 프로젝트 로컬 실행 기록 도입

**Files:**
- Create: `skills/safe-change/SKILL.md`
- Create: `skills/harness-audit/SKILL.md`
- Modify: `C:\khd\playwright\.gitignore`
- Modify: `C:\khd\allproject\uems_fe\.gitignore`
- Modify: `C:\khd\6gworkspace\UnifyEMS\uems_fe\.gitignore`

**Interfaces:**
- Consumes: `core/policy/*.md`, `core/playbooks/safe-change.md`, `core/templates/result-report.md`, `packs/nms-ems/PACK.md`
- Produces: 모든 프로젝트에 적용 가능한 `safe-change`, 읽기 전용 `harness-audit`, 그리고 로컬 실행 기록 제외 규칙

- [x] **Step 1: `safe-change` 스킬을 한 파일로 작성한다.**

  Frontmatter는 아래처럼 작성한다.

  ```yaml
  ---
  name: safe-change
  description: 현재 흐름과 영향 범위를 확인한 뒤 최소 변경, 실제 검증, 프로젝트 로컬 실행 기록, 지식 승격 판단을 수행할 때 사용한다.
  ---
  ```

  본문에는 `Goal`, `Workflow`, `Rules`, `Verification` 네 제목을 사용한다. Workflow는 `<PROJECT_ROOT>/.harness/runs/<YYYY-MM-DD>-<work-name>/` 아래의 `PLAN.local.md`, `EVIDENCE.local.md`, `RETRO.local.md`를 전체 사이클에만 만들고, 경량 작업은 `RETRO.local.md` 한 파일만 만드는 기준을 포함한다.

- [x] **Step 2: `harness-audit` 스킬을 한 파일로 작성한다.**

  Frontmatter의 이름은 `harness-audit`으로 한다. 본문에는 읽기 전용이라는 범위와 다음 점검 항목을 포함한다: 오래된 `.ai` 참조, 존재하지 않는 경로, Core/Pack 경계 위반, 중복·모순 규칙, 민감정보 후보, 스킬 발견 위치 중복, 날짜가 지난 관찰 기록.

  Audit 결과는 파일·행·위험 이유를 기록하되, 민감값 자체는 절대 복사하지 않도록 한다.

- [x] **Step 3: 세 pilot 프로젝트의 로컬 실행 기록을 기본 제외한다.**

  `C:\khd\playwright\.gitignore`, `C:\khd\allproject\uems_fe\.gitignore`, `C:\khd\6gworkspace\UnifyEMS\uems_fe\.gitignore`에 다음 한 줄이 없으면 각각 추가한다.

  ```gitignore
  /.harness/runs/
  ```

  기존 ignore 항목의 순서와 내용을 바꾸지 않는다.

- [x] **Step 4: 두 스킬의 구조를 검증한다.**

  Run:

  ```powershell
  python C:\Users\김현동\.codex\skills\.system\skill-creator\scripts\quick_validate.py C:\khd\ai-harness\skills\safe-change
  python C:\Users\김현동\.codex\skills\.system\skill-creator\scripts\quick_validate.py C:\khd\ai-harness\skills\harness-audit
  Get-ChildItem -LiteralPath C:\khd\ai-harness\skills\safe-change -Force
  Get-ChildItem -LiteralPath C:\khd\ai-harness\skills\harness-audit -Force
  ```

  Expected: 두 validator가 `Skill is valid!`를 출력하고, 각 신규 스킬 폴더에는 `SKILL.md`만 존재한다.

## Task 5: 첫 소비자 두 유형에서 새 경로를 병행 검증

**Files:**
- Modify: `C:\khd\playwright\AGENTS.md`
- Create: `C:\khd\playwright\.harness\runs\2026-08-28-core-only-pilot\RETRO.local.md`
- Modify: `C:\khd\allproject\uems_fe\AGENTS.md`
- Modify: `C:\khd\6gworkspace\UnifyEMS\uems_fe\AGENTS.md`
- Create: `C:\khd\allproject\uems_fe\.harness\runs\2026-08-28-nms-pack-pilot\RETRO.local.md`

**Interfaces:**
- Consumes: `core/`, `packs/nms-ems/`, existing compatibility paths, `skills/safe-change/SKILL.md`
- Produces: Core-only 사용 증거와 Core + Pack 사용 증거; 세 소비자에서 새 경로가 기존 경로를 대체하지 않고 안내하는 상태

- [x] **Step 1: Playwright의 shared harness 안내를 Core-first로 바꾼다.**

  `C:\khd\playwright\AGENTS.md`의 `공통 하네스 사용 기준`에 다음 의미를 추가한다: 일반 Playwright/TypeScript 변경에는 `<HARNESS_ROOT>/core/`와 `safe-change`를 우선 사용하고, NMS 화면·도메인 판단이 필요한 경우에만 `<HARNESS_ROOT>/packs/nms-ems/PACK.md`를 추가로 읽는다.

  기존 `현재 프로젝트 코드와 실제 실행 동작을 우선한다` 규칙은 그대로 둔다.

- [x] **Step 2: Core-only 경량 pilot 기록을 만든다.**

  `C:\khd\playwright\.harness\runs\2026-08-28-core-only-pilot\RETRO.local.md`에 다음 필드를 채운다: `요청과 변경 요약`, `읽은 Core 문서`, `읽지 않은 NMS Pack 문서`, `실행한 검증`, `미검증 범위`, `반복 가능한 교훈`.

  이 pilot은 실제 제품 테스트를 실행하지 않는다. `문서 경로와 skill 형식만 확인함`을 검증으로, `브라우저 및 NMS 도메인 동작 미검증`을 미검증으로 기록한다.

- [x] **Step 3: 두 UEMS 소비자의 읽기 경로를 Core + Pack 병행 구조로 바꾼다.**

  두 `AGENTS.md`에서 기존 `harness/`와 `nms-knowledge/` 경로를 삭제하지 않는다. 기존 `작업 전 읽을 파일` 앞에 아래 순서를 추가한다.

  1. `<HARNESS_ROOT>/core/policy/global.md`
  2. `<HARNESS_ROOT>/core/policy/security.md`
  3. `<HARNESS_ROOT>/core/policy/approval.md`
  4. `<HARNESS_ROOT>/packs/nms-ems/PACK.md`
  5. 작업 유형에 대응하는 기존 세부 legacy 문서

- [x] **Step 4: NMS Pack 경량 pilot 기록을 만든다.**

  `C:\khd\allproject\uems_fe\.harness\runs\2026-08-28-nms-pack-pilot\RETRO.local.md`에 Core 세 정책, `PACK.md`, `domain-safety.md`를 읽은 사실을 적는다. 검증에는 AGENTS 경로 존재 확인만 적고, 제품 build·browser·API는 실행하지 않았다고 명확히 적는다.

- [x] **Step 5: 소비자 참조와 untracked 기록을 검증한다.**

  Run:

  ```powershell
  rg -n "core[\\/]policy[\\/]global.md|packs[\\/]nms-ems[\\/]PACK.md" C:\khd\playwright\AGENTS.md C:\khd\allproject\uems_fe\AGENTS.md C:\khd\6gworkspace\UnifyEMS\uems_fe\AGENTS.md
  Test-Path C:\khd\playwright\.harness\runs\2026-08-28-core-only-pilot\RETRO.local.md
  Test-Path C:\khd\allproject\uems_fe\.harness\runs\2026-08-28-nms-pack-pilot\RETRO.local.md
  Select-String -LiteralPath C:\khd\playwright\.gitignore,C:\khd\allproject\uems_fe\.gitignore,C:\khd\6gworkspace\UnifyEMS\uems_fe\.gitignore -SimpleMatch '/.harness/runs/'
  ```

  Expected: 세 AGENTS.md 모두 새 Core/Pack 경로를 보이며, 두 RETRO 파일이 존재하고 세 `.gitignore`에 `/.harness/runs/` 규칙이 있다. Git 저장소인 프로젝트에서만 `git check-ignore`로 추가 확인한다.

## Task 6: 호환성·중복·보안 감사와 이관 보류 결정

**Files:**
- Create: `C:\khd\ai-harness\.harness\runs\2026-08-28-harness-audit\EVIDENCE.local.md`
- Create: `C:\khd\ai-harness\.harness\runs\2026-08-28-harness-audit\RETRO.local.md`
- Modify: `docs/superpowers/migration/2026-08-28-harness-baseline.md`

**Interfaces:**
- Consumes: `skills/harness-audit/SKILL.md`, Task 1 기준본, Task 2~5 산출물
- Produces: 기존 경로를 계속 유지할지, 특정 소비자를 이관할 준비가 되었는지에 대한 근거 기반 판단

- [x] **Step 1: 새·기존 경로의 중복과 오래된 참조를 읽기 전용으로 검사한다.**

  Run:

  ```powershell
  rg -n -i "\.ai[\\/]harness|\.ai[\\/]nms-knowledge|\.ai[\\/]skills|harness[\\/]policy|nms-knowledge[\\/]" C:\khd\ai-harness C:\khd\playwright\AGENTS.md C:\khd\allproject\uems_fe\AGENTS.md C:\khd\6gworkspace\UnifyEMS\uems_fe\AGENTS.md
  rg -n -i --only-matching "password|passwd|token|api[ _-]?key|authorization:|cookie:|secret" C:\khd\ai-harness --glob '*.md'
  ```

  Do not print matching values in the evidence file. Store path, line number, category, and remediation decision only.

- [x] **Step 2: audit 실행 증거와 회고를 작성한다.**

  `EVIDENCE.local.md`에는 명령, 실행 시각, 결과 요약, 발견 파일/행, 미검증 범위를 기록한다.

  `RETRO.local.md`에는 `호환 경로 유지`, `대량 이동 보류`, `다음 이관 조건`을 적는다. 다음 이관 조건은 `알려진 소비자에서 새 경로 검증`, `legacy 참조 검색 결과 검토`, `민감정보 후보 처리`, `정상 Git 기준 확보` 네 가지다.

- [x] **Step 3: 기준본을 이관 판정으로 갱신한다.**

  `2026-08-28-harness-baseline.md` 끝에 `## v0 이관 판정`을 추가하고 다음 세 값을 정확히 기록한다.

  ```markdown
  - 새 Core: 사용 가능
  - NMS/EMS Pack: 병행 사용 가능
  - legacy 대량 이동 및 삭제: 보류
  ```

- [x] **Step 4: 전체 구조와 금지 범위를 최종 검증한다.**

  Run:

  ```powershell
  rg --files C:\khd\ai-harness\core C:\khd\ai-harness\packs C:\khd\ai-harness\skills\safe-change C:\khd\ai-harness\skills\harness-audit
  rg -n "TBD|TODO|PLACEHOLDER|implement later|fill in details" C:\khd\ai-harness\core C:\khd\ai-harness\packs C:\khd\ai-harness\skills\safe-change C:\khd\ai-harness\skills\harness-audit C:\khd\ai-harness\docs\superpowers\migration
  Test-Path C:\khd\ai-harness\harness
  Test-Path C:\khd\ai-harness\nms-knowledge
  ```

  Expected: Core, Pack, 두 신규 스킬이 존재하고 placeholder 검색 결과가 없으며, 기존 `harness/`와 `nms-knowledge/`도 계속 존재한다.

## Execution Completion Criteria

- 범용 Playwright 변경은 Pack을 읽지 않고 Core와 `safe-change`로 기록할 수 있다.
- NMS UEMS 변경은 Core와 NMS/EMS Pack을 함께 읽고, 세부 legacy 지식을 필요할 때만 접근한다.
- 신규 스킬 두 개는 `SKILL.md`만으로 validator를 통과한다.
- 프로젝트 실행 기록은 로컬로 남고 Playwright repository에서 ignore된다.
- 기존 `harness/`와 `nms-knowledge/`는 삭제되지 않는다.
- 감사 결과의 민감정보 후보는 값 노출 없이 경로·행·판정으로만 보고된다.
- 결과 보고에는 실행한 검증과 미검증 범위가 분리되어 있다.

## Deferred Follow-up

다음 항목은 이 계획의 완료 조건이 아니다. v0 audit 결과와 사용자 승인 후 별도 계획으로 다룬다.

1. 정상 Git repository 전환 및 초기 기준 커밋
2. 모든 `nms-knowledge/**`와 NMS 전용 `harness/**` 문서의 Pack 내부 이관
3. 남은 소비자의 AGENTS.md 경로 이관
4. legacy 경로의 pointer 전환 또는 삭제
5. 반복적이고 결정적인 검사에만 한정한 `scripts/` 추가
