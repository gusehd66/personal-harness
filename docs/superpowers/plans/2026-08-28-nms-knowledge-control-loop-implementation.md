# NMS Knowledge Control Loop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

**Goal:** 프로젝트 `AGENTS.md` 우선 참조 방식으로 paperthin식 변경·검증·회고 제어 루프와 NMS/EMS Pack 지식을 결합한다.

**Architecture:** `core/playbooks/harness-routing.md`를 단일 요청 라우팅 원본으로 만든다. 범용 제어 skill은 `skills/`에 얇게 두고, NMS 지식·운영 위험 규칙은 `packs/nms-ems/`에만 둔다. 세 소비자 `AGENTS.md`는 이 라우팅 문서를 먼저 읽도록 연결한다.

**Tech Stack:** Markdown, Codex skills, Windows PowerShell, Python UTF-8 skill validator

**Spec:** `C:\khd\ai-harness\docs\superpowers\specs\2026-08-28-nms-knowledge-control-loop-design.md`

## Global Constraints

- Codex 전역 skill catalogue, symlink, plugin 설정은 변경하지 않는다.
- 제품 소스, 실행 설정, DB, 장비, scheduler, 배포 상태를 변경하지 않는다.
- `C:\khd\allproject\uems_fe`와 `C:\khd\6gworkspace\UnifyEMS\uems_fe`의 기존 dirty work는 보존하고 `AGENTS.md`만 수정한다.
- 각 skill folder에는 완료 시 `SKILL.md`만 남긴다.
- NMS 지식은 `packs/nms-ems/`에만 보관하고 skill 본문에 복제하지 않는다.
- 이 하네스 루트는 정상 Git 저장소가 아니므로 commit, push, branch 조작을 하지 않는다.

---

## File Structure

| 경로 | 작업 | 책임 |
| --- | --- | --- |
| `core/playbooks/harness-routing.md` | 생성 | 요청 유형별 주 skill, Core/Pack 읽기 순서, 종료 루프의 단일 원본 |
| `skills/harness-verify/SKILL.md` | 생성 | 변경 후 검증과 미검증 범위 구분 |
| `skills/harness-retro/SKILL.md` | 생성 | 검증된 지식의 저장 표면 분류 |
| `skills/harness-refresh/SKILL.md` | 생성 | 명시 요청된 문서·skill의 clean-v0 갱신 |
| `skills/{safe-change,harness-audit,angular-ui-harness,backend-harness,manual-update}/SKILL.md` | 수정 | 얇은 제어 절차와 Pack 참조만 유지 |
| `packs/nms-ems/knowledge/manual/my5g-docx-structure.md` | 생성 | NMS DOCX 구조 지식의 Pack 단일 원본 |
| `skills/codex-learning-loop/`, 각 `skills/*/agents/`, `skills/manual-update/references/` | 삭제 | 중복 learning skill, UI metadata, skill 내부 NMS reference 제거 |
| `README.md`, `packs/nms-ems/PACK.md` | 수정 | 라우팅과 NMS 전문 skill 위치 안내 |
| 세 소비자 `AGENTS.md` | 수정 | `harness-routing.md` 우선 참조와 프로젝트 코드 우선 규칙 |

## Task 1: Core routing 원본과 Pack 진입점 정리

**Files:**
- Create: `core/playbooks/harness-routing.md`
- Modify: `README.md`
- Modify: `packs/nms-ems/PACK.md`

**Interfaces:**
- Consumes: `core/policy/global.md`, `core/playbooks/safe-change.md`, `packs/nms-ems/PACK.md`
- Produces: 프로젝트 `AGENTS.md`와 모든 skill이 참조할 요청 라우팅 원본

- [x] **Step 1: 요청 유형, 주 skill, Core/Pack 읽기 순서를 표로 작성한다.**

`harness-routing.md`에는 다음 행을 포함한다.

| 요청 | 주 skill | 추가 문서 |
| --- | --- | --- |
| 코드·설정·문서 변경 | `safe-change` | NMS이면 `packs/nms-ems/PACK.md` |
| 하네스 읽기 전용 감사 | `harness-audit` | 소비자 `AGENTS.md` |
| NMS UI/backend 하네스 설계 | 해당 전문 skill | Pack 전용 playbook·template |
| NMS 매뉴얼 갱신 | `manual-update` | Pack manual/UI knowledge |
| 문서·skill clean-v0 갱신 | `harness-refresh` | target과 관련 canonical 문서 |

- [x] **Step 2: NMS write 위험과 종료 검증·회고 연결을 명시한다.**

`safe-change` 또는 전문 skill의 완료 전 `harness-verify`를 읽고, 반복 가치가 검증된 지식이 있을 때만 `harness-retro`를 읽도록 기록한다. DB, 장비 명령, scheduler, 인증, 배포는 `packs/nms-ems/policy/domain-safety.md`와 사용자 승인 경계를 먼저 적용한다고 기록한다.

- [x] **Step 3: README와 PACK.md에서 routing 원본을 단일 진입점으로 연결한다.**

README는 일반 변경의 첫 문서를 `core/playbooks/harness-routing.md`로 바꾼다. Pack의 매뉴얼 행은 Pack manual knowledge와 `skills/manual-update/SKILL.md`를 함께 가리키고, 전문 skill은 명시 설계/갱신 요청에만 선택한다고 기록한다.

- [x] **Step 4: Core와 Pack의 제거된 learning skill 참조를 검색한다.**

Run:

```powershell
rg -n 'codex-learning-loop|harness-routing' C:\khd\ai-harness\README.md C:\khd\ai-harness\core C:\khd\ai-harness\packs C:\khd\ai-harness\skills
```

Expected: `harness-routing` 참조는 새 routing 원본과 의도된 소비자에만 있고, 제거된 shared `codex-learning-loop` 참조는 없다.

## Task 2: 얇은 제어 skill catalogue 구현

**Files:**
- Create: `skills/harness-verify/SKILL.md`
- Create: `skills/harness-retro/SKILL.md`
- Create: `skills/harness-refresh/SKILL.md`
- Modify: `skills/safe-change/SKILL.md`
- Modify: `skills/harness-audit/SKILL.md`
- Modify: `skills/angular-ui-harness/SKILL.md`
- Modify: `skills/backend-harness/SKILL.md`
- Modify: `skills/manual-update/SKILL.md`
- Delete: `skills/codex-learning-loop/**`
- Delete: `skills/{angular-ui-harness,backend-harness,manual-update}/agents/openai.yaml`
- Delete: `skills/manual-update/agents/openai.yaml`

**Interfaces:**
- Consumes: `core/playbooks/harness-routing.md`, `core/`, `packs/nms-ems/PACK.md`
- Produces: 한 skill당 하나의 명확한 역할과 `SKILL.md` 하나만 갖는 skill folder

- [x] **Step 1: 세 신규 skill을 각각 하나의 목적과 4~6단계 절차로 작성한다.**

`harness-verify`는 실제 실행 증거, 참조·경로, 테스트 종류, 미검증 범위를 점검한다. `harness-retro`는 작업 기록·프로젝트 문서·Core·Pack·skill 후보를 분류하며 자동 승격을 금지한다. `harness-refresh`는 대상과 canonical sibling을 읽고 현재 사실만 남기는 명시 정리 절차를 수행한다.

- [x] **Step 2: safe-change를 변경 요청의 주 skill로 갱신한다.**

`safe-change`는 routing 문서를 먼저 읽고, NMS 영향 여부를 분류한 뒤 필요한 Pack을 읽는다. 완료 전 `harness-verify`, 재사용 지식이 확인된 경우 `harness-retro`를 사용하도록 한다. 전문 UI/backend/manual 요청은 주 skill을 교체하며 중복 호출하지 않는다고 적는다.

- [x] **Step 3: 기존 전문 skill에서 지식 나열을 제거한다.**

`angular-ui-harness`, `backend-harness`, `manual-update`는 적용 조건, Pack에서 읽을 시작 문서, 핵심 산출물, 금지/승인 조건, 검증만 남긴다. `harness-audit`은 읽기 전용 감사 역할을 유지하되 routing 문서를 참조한다.

- [x] **Step 4: manual DOCX 지식을 Pack으로 이동하고 skill 내부 보조 파일을 삭제한다.**

`skills/manual-update/references/my5g-docx-structure.md`의 55줄 내용을 `packs/nms-ems/knowledge/manual/my5g-docx-structure.md`로 옮긴다. `manual-update`는 새 Pack 경로를 참조한다. 이후 모든 `agents/`, `references/`, `codex-learning-loop/` 빈 폴더를 제거한다.

- [x] **Step 5: skill validator를 실행한다.**

Run:

```powershell
$validator = 'C:\Users\김현동\.codex\skills\.system\skill-creator\scripts\quick_validate.py'
$skills = 'safe-change','harness-audit','harness-verify','harness-retro','harness-refresh','angular-ui-harness','backend-harness','manual-update'
foreach ($skill in $skills) { python -X utf8 $validator "C:\khd\ai-harness\skills\$skill" }
```

Expected: 모든 command가 `Skill is valid!`를 출력한다.

## Task 3: 세 프로젝트 AGENTS.md에 우선 라우팅 연결

**Files:**
- Modify: `C:\khd\playwright\AGENTS.md`
- Modify: `C:\khd\allproject\uems_fe\AGENTS.md`
- Modify: `C:\khd\6gworkspace\UnifyEMS\uems_fe\AGENTS.md`

**Interfaces:**
- Consumes: `core/playbooks/harness-routing.md`, 각 프로젝트의 기존 하네스 위치 선언
- Produces: Codex가 프로젝트별 규칙보다 먼저 공용 routing을 읽되, 현재 소스·실행 동작을 우선하는 지시

- [x] **Step 1: 각 AGENTS.md의 공용 하네스 섹션에 동일한 우선 절차를 추가한다.**

삽입할 문구:

```markdown
## 공용 하네스 요청 라우팅

요청을 처리하기 전에 `<HARNESS_ROOT>/core/playbooks/harness-routing.md`를 먼저 읽어 주 skill과 필요한 Core/Pack 문서를 결정한다. NMS/EMS 사실이 판단·검증에 영향을 주면 `<HARNESS_ROOT>/packs/nms-ems/PACK.md`를 추가로 읽는다.

공용 문서는 판단·검증 기준이며, 현재 프로젝트의 source, 활성 구성, 로그, 실제 실행 동작과 충돌하면 현재 프로젝트 근거를 우선한다. 실제 DB·장비·scheduler·인증·배포 write는 Pack 안전 규칙과 사용자 승인을 먼저 적용한다.
```

- [x] **Step 2: 기존 중복 safe-change 및 Pack 안내를 새 routing 흐름에 맞게 축소한다.**

Playwright의 절대경로 안내와 두 UEMS의 `<HARNESS_ROOT>` 안내는 유지한다. 기존 상세 문서 목록과 프로젝트 특화 build/route 규칙은 변경하지 않는다.

- [x] **Step 3: 세 AGENTS.md의 경로 존재와 routing 참조를 확인한다.**

Run:

```powershell
$routing = 'C:\khd\ai-harness\core\playbooks\harness-routing.md'
Test-Path -LiteralPath $routing
rg -n 'harness-routing\.md|현재 프로젝트.*우선|source.*실행' C:\khd\playwright\AGENTS.md C:\khd\allproject\uems_fe\AGENTS.md C:\khd\6gworkspace\UnifyEMS\uems_fe\AGENTS.md
```

Expected: routing path는 `True`이고, 세 파일 모두 새 우선 절차를 가진다.

## Task 4: 구조·경로·내용 검증과 기록

**Files:**
- Modify: `VERSION.md`
- Modify: `.harness/runs/2026-08-28-harness-audit/EVIDENCE.local.md`
- Test: `skills/**`, `core/**`, `packs/**`, 세 소비자 `AGENTS.md`

**Interfaces:**
- Consumes: Task 1~3의 최종 파일
- Produces: 실제 구조와 검증 범위를 기록한 완료 상태

- [x] **Step 1: skill folder의 보조 파일 잔여물을 확인한다.**

Run:

```powershell
Get-ChildItem -LiteralPath C:\khd\ai-harness\skills -Recurse -File |
  Where-Object { $_.Name -ne 'SKILL.md' } |
  Select-Object FullName
```

Expected: no output.

- [x] **Step 2: 라우팅 참조와 삭제된 shared skill 참조를 검증한다.**

Run:

```powershell
rg -n 'skills/codex-learning-loop|skills\\codex-learning-loop|codex-learning-loop' C:\khd\ai-harness\README.md C:\khd\ai-harness\core C:\khd\ai-harness\packs C:\khd\ai-harness\skills C:\khd\playwright\AGENTS.md C:\khd\allproject\uems_fe\AGENTS.md C:\khd\6gworkspace\UnifyEMS\uems_fe\AGENTS.md
rg -n 'harness-routing\.md' C:\khd\ai-harness\README.md C:\khd\ai-harness\core C:\khd\ai-harness\packs C:\khd\ai-harness\skills C:\khd\playwright\AGENTS.md C:\khd\allproject\uems_fe\AGENTS.md C:\khd\6gworkspace\UnifyEMS\uems_fe\AGENTS.md
```

Expected: first search has no output; second search includes Core routing and all intended consumers.

- [x] **Step 3: VERSION과 local evidence에 구현 범위와 미검증 범위를 기록한다.**

기록에는 AGENTS.md 우선 참조 방식, Core/Pack/skills 역할, eight skill validator 결과, 전역 catalogue 미설치, 제품 build/browser/API/운영 write 미실행을 포함한다.

## Execution Completion Criteria

- `core/playbooks/harness-routing.md`가 주 skill, Pack 조건, verify·retro 종료 흐름의 단일 원본이다.
- 8개 skill은 각각 `SKILL.md` 하나만 갖고 validator를 통과한다.
- NMS DOCX 구조 지식은 Pack에만 존재한다.
- 세 소비자 `AGENTS.md`가 routing을 우선 참조하면서 현재 프로젝트 근거 우선 규칙을 유지한다.
- 제품 동작, 실제 운영 write, 전역 Codex 설치는 수행하지 않는다.
