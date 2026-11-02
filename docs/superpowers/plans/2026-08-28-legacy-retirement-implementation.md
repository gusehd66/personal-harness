# Legacy Harness 제거 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

**Goal:** legacy `harness/`와 `nms-knowledge/`의 필요한 내용을 `core/`와 `packs/nms-ems/`로 이관하고, 모든 현재 사용 경로를 갱신한 뒤 두 legacy 폴더를 삭제한다.

**Architecture:** 범용 policy, review, learning, result-report는 이미 생성된 `core/`를 canonical으로 사용한다. NMS/EMS playbook, 분석 템플릿, 113개 지식 문서는 `packs/nms-ems/`에 하나의 원본으로 둔다. 삭제 전 SHA-256 기준, 민감정보 후보 판정, 현재 링크 검사, skill validator를 통과해야 한다.

**Tech Stack:** Markdown, Codex skills, Windows PowerShell, SHA-256

**Spec:** `C:\khd\ai-harness\docs\superpowers\specs\2026-08-28-legacy-retirement-design.md`

## Global Constraints

- 프로젝트 로컬 `.ai/manual/`은 삭제하거나 이동하지 않는다.
- 제품 소스 코드를 수정하지 않는다.
- `harness/` 또는 `nms-knowledge/`는 hash·링크·민감정보 검증이 끝나기 전에는 삭제하지 않는다.
- 실제 credential, cookie, private key, 운영 연결 문자열을 발견하면 값은 출력하지 않고 이관을 중단한다.
- 이 하네스는 정상 Git 저장소가 아니므로 commit, push, branch 조작을 하지 않는다.
- 역사 기록인 `docs/superpowers/**`와 `.harness/runs/**`의 legacy 문자열은 현재 사용 링크 검사에서 제외한다.
- 영구 `scripts/`는 추가하지 않는다. 반복 변환은 이관 작업에 한정된 PowerShell one-off로 수행한다.

---

## File Structure

| 경로 | 작업 | 책임 |
| --- | --- | --- |
| `docs/superpowers/migration/2026-08-28-legacy-manifest.csv` | 생성 | legacy 파일별 SHA-256, 최종 경로, 민감정보 판정 |
| `docs/superpowers/migration/2026-08-28-legacy-retirement-evidence.md` | 생성 | 이관·링크·삭제 전후 검증 증거 |
| `packs/nms-ems/knowledge/**` | 생성 | 기존 `nms-knowledge/**`의 유일한 NMS/EMS 지식 원본 |
| `packs/nms-ems/playbooks/*.md` | 생성 | NMS/EMS 전용 작업 절차 |
| `packs/nms-ems/templates/*.md` | 생성 | Angular/backend NMS 분석 보고 형식 |
| `packs/nms-ems/PACK.md` | 수정 | legacy 라우팅 표 제거, 최종 경로 기준으로 갱신 |
| `README.md`, `VERSION.md` | 수정 | legacy 없는 최종 구조 안내 |
| `skills/*.SKILL.md` | 수정 | 새 Core/Pack 경로만 참조 |
| 세 프로젝트 `AGENTS.md` | 수정 | 새 Core/Pack 경로만 참조 |
| `harness/`, `nms-knowledge/` | 삭제 | 검증 완료 후 제거할 legacy 원본 |

## Task 1: 삭제 전 기준과 민감정보 판정 생성

**Files:**
- Create: `docs/superpowers/migration/2026-08-28-legacy-manifest.csv`
- Create: `docs/superpowers/migration/2026-08-28-legacy-retirement-evidence.md`
- Read: `harness/**`, `nms-knowledge/**`

**Interfaces:**
- Consumes: legacy `harness/`의 13개 Markdown 파일과 `nms-knowledge/`의 113개 Markdown 파일
- Produces: `source_path`, `sha256`, `target_path`, `sensitive_status` 열을 갖는 삭제 전 기준

- [x] **Step 1: legacy 파일 인벤토리와 SHA-256을 생성한다.**

  Run:

  ```powershell
  $root = 'C:\khd\ai-harness'
  $rows = Get-ChildItem -LiteralPath "$root\harness", "$root\nms-knowledge" -Recurse -File -Filter '*.md' |
    ForEach-Object {
      $relative = $_.FullName.Substring($root.Length + 1).Replace('\', '/')
      $target = if ($relative.StartsWith('nms-knowledge/')) {
        'packs/nms-ems/knowledge/' + $relative.Substring('nms-knowledge/'.Length)
      } elseif ($relative -match '^harness/playbooks/(angular-ui-harness|backend-harness|frontend-feature|angular-react-framework-modules-mapping|bugfix)\.md$') {
        'packs/nms-ems/playbooks/' + [System.IO.Path]::GetFileName($relative)
      } elseif ($relative -match '^harness/templates/(angular-harness-report|backend-harness-report)\.md$') {
        'packs/nms-ems/templates/' + [System.IO.Path]::GetFileName($relative)
      } else {
        'core canonical replacement; no byte-for-byte move'
      }
      [PSCustomObject]@{
        source_path = $relative
        sha256 = (Get-FileHash -Algorithm SHA256 -LiteralPath $_.FullName).Hash
        target_path = $target
        sensitive_status = 'pending'
      }
    }
  $rows | Export-Csv -LiteralPath "$root\docs\superpowers\migration\2026-08-28-legacy-manifest.csv" -NoTypeInformation -Encoding utf8
  ```

  Expected: manifest에는 126개 source 행이 있다.

- [x] **Step 2: 값 형식 민감정보 후보를 값 노출 없이 판정한다.**

  Run:

  ```powershell
  $root = 'C:\khd\ai-harness'
  $pattern = '(?i)(password|passwd|token|secret|api[ _-]?key)\s*[:=]\s*[^\s`]+|authorization\s*:\s*[^\s]+'
  Get-ChildItem -LiteralPath "$root\harness", "$root\nms-knowledge" -Recurse -File -Filter '*.md' |
    Select-String -Pattern $pattern |
    Select-Object @{Name='Path'; Expression = { $_.Path }}, LineNumber |
    Format-Table -AutoSize
  ```

  For each candidate, write only `masked`, `identifier`, or `stop` into `sensitive_status`. Do not copy the matching line or value into the manifest. If any row is `stop`, stop the plan before Task 2 and ask the user for credential-rotation authority.

- [x] **Step 3: core canonical replacements를 기록한다.**

  In the evidence file, record that `harness/policy/{approval,security,global}.md`, `harness/playbooks/{code-review,long-term-learning}.md`, and `harness/templates/result-report.md` are intentionally replaced by existing `core/` files, not byte-for-byte copied.

- [x] **Step 4: 기준 파일의 수와 상태를 검증한다.**

  Run:

  ```powershell
  $manifest = Import-Csv 'C:\khd\ai-harness\docs\superpowers\migration\2026-08-28-legacy-manifest.csv'
  $manifest.Count
  $manifest | Group-Object sensitive_status | Select-Object Name, Count
  ```

  Expected: 126 rows, and no row has `stop` before proceeding.

## Task 2: NMS/EMS 원본을 새 Pack으로 복사하고 동일성을 검증

**Files:**
- Create: `packs/nms-ems/knowledge/**`
- Create: `packs/nms-ems/playbooks/{angular-ui-harness,backend-harness,frontend-feature,angular-react-framework-modules-mapping,bugfix}.md`
- Create: `packs/nms-ems/templates/{angular-harness-report,backend-harness-report}.md`
- Read: `docs/superpowers/migration/2026-08-28-legacy-manifest.csv`

**Interfaces:**
- Consumes: manifest의 `target_path`와 legacy source 파일
- Produces: legacy NMS/EMS 문서와 동일한 SHA-256을 갖는 Pack 사본

- [x] **Step 1: 새 Pack 목적지 경로가 legacy와 겹치지 않는지 확인한다.**

  Run:

  ```powershell
  Test-Path 'C:\khd\ai-harness\packs\nms-ems\knowledge'
  Test-Path 'C:\khd\ai-harness\packs\nms-ems\playbooks'
  Test-Path 'C:\khd\ai-harness\packs\nms-ems\templates'
  ```

  Expected: 세 경로는 아직 없거나, 이전 실패 작업의 잔여물이라면 source hash와 다를 수 있으므로 Task 2를 중단하고 사용자에게 알린다.

- [x] **Step 2: NMS knowledge와 NMS 전용 playbook·template을 새 경로에 복사한다.**

  Run:

  ```powershell
  $root = 'C:\khd\ai-harness'
  Copy-Item -LiteralPath "$root\nms-knowledge" -Destination "$root\packs\nms-ems\knowledge" -Recurse
  New-Item -ItemType Directory -Path "$root\packs\nms-ems\playbooks", "$root\packs\nms-ems\templates" -Force | Out-Null
  Copy-Item -LiteralPath "$root\harness\playbooks\angular-ui-harness.md", "$root\harness\playbooks\backend-harness.md", "$root\harness\playbooks\frontend-feature.md", "$root\harness\playbooks\angular-react-framework-modules-mapping.md", "$root\harness\playbooks\bugfix.md" -Destination "$root\packs\nms-ems\playbooks"
  Copy-Item -LiteralPath "$root\harness\templates\angular-harness-report.md", "$root\harness\templates\backend-harness-report.md" -Destination "$root\packs\nms-ems\templates"
  ```

- [x] **Step 3: 복사 직후 SHA-256 동일성을 검증한다.**

  Run:

  ```powershell
  $root = 'C:\khd\ai-harness'
  $manifest = Import-Csv "$root\docs\superpowers\migration\2026-08-28-legacy-manifest.csv" |
    Where-Object { $_.target_path -notlike 'core canonical replacement*' }
  $mismatch = foreach ($row in $manifest) {
    $targetPath = Join-Path $root ($row.target_path.Replace('/', '\'))
    if (-not (Test-Path -LiteralPath $targetPath) -or (Get-FileHash -Algorithm SHA256 -LiteralPath $targetPath).Hash -ne $row.sha256) { $row }
  }
  $mismatch | Select-Object source_path, target_path
  ```

  Expected: no output. Any mismatch blocks Task 3 and prevents deletion.

## Task 3: 새 canonical 경로로 링크와 소비자 갱신

**Files:**
- Modify: `packs/nms-ems/PACK.md`
- Modify: `packs/nms-ems/knowledge/**/*.md`
- Modify: `packs/nms-ems/playbooks/*.md`
- Modify: `packs/nms-ems/templates/*.md`
- Modify: `README.md`, `VERSION.md`
- Modify: `skills/angular-ui-harness/SKILL.md`, `skills/backend-harness/SKILL.md`, `skills/manual-update/SKILL.md`, `skills/codex-learning-loop/SKILL.md`
- Modify: `C:\khd\playwright\AGENTS.md`, `C:\khd\allproject\uems_fe\AGENTS.md`, `C:\khd\6gworkspace\UnifyEMS\uems_fe\AGENTS.md`

**Interfaces:**
- Consumes: Task 2의 Pack 사본
- Produces: 현재 사용 문서와 세 소비자가 legacy 경로 없이 Core 또는 Pack만 참조하는 상태

- [x] **Step 1: Pack 안내와 README를 최종 구조로 갱신한다.**

  `PACK.md`에서 `Legacy knowledge 호환 상태` 표를 제거한다. 작업 유형별 문서 경로는 `playbooks/`, `templates/`, `knowledge/` 아래 새 경로로 바꾼다.

  README에서는 `harness/`와 `nms-knowledge/` 설명과 링크를 제거한다. 기존 NMS 문서 목록은 `packs/nms-ems/knowledge/`와 `packs/nms-ems/playbooks/` 기준으로 바꾼다. VERSION에는 `legacy-retirement-v1` 항목과 legacy 삭제 완료 사실을 추가한다.

- [x] **Step 2: 현재 사용 NMS 문서의 내부 경로를 일괄 치환한다.**

  Run this one-off only against `packs/nms-ems/knowledge`, `packs/nms-ems/playbooks`, and `packs/nms-ems/templates`:

  ```powershell
  $roots = @(
    'C:\khd\ai-harness\packs\nms-ems\knowledge',
    'C:\khd\ai-harness\packs\nms-ems\playbooks',
    'C:\khd\ai-harness\packs\nms-ems\templates'
  )
  $map = [ordered]@{
    '.ai/nms-knowledge/' = '<HARNESS_ROOT>/packs/nms-ems/knowledge/'
    '<HARNESS_ROOT>/nms-knowledge/' = '<HARNESS_ROOT>/packs/nms-ems/knowledge/'
    'nms-knowledge/' = 'packs/nms-ems/knowledge/'
    '.ai/harness/templates/result-report.md' = '<HARNESS_ROOT>/core/templates/result-report.md'
    '<HARNESS_ROOT>/harness/templates/result-report.md' = '<HARNESS_ROOT>/core/templates/result-report.md'
    'harness/templates/result-report.md' = 'core/templates/result-report.md'
    '<HARNESS_ROOT>/harness/policy/global.md' = '<HARNESS_ROOT>/core/policy/global.md'
    '<HARNESS_ROOT>/harness/policy/security.md' = '<HARNESS_ROOT>/core/policy/security.md'
    '<HARNESS_ROOT>/harness/policy/approval.md' = '<HARNESS_ROOT>/core/policy/approval.md'
    '<HARNESS_ROOT>/harness/playbooks/' = '<HARNESS_ROOT>/packs/nms-ems/playbooks/'
    'harness/playbooks/' = 'packs/nms-ems/playbooks/'
    '<HARNESS_ROOT>/harness/templates/angular-harness-report.md' = '<HARNESS_ROOT>/packs/nms-ems/templates/angular-harness-report.md'
    '<HARNESS_ROOT>/harness/templates/backend-harness-report.md' = '<HARNESS_ROOT>/packs/nms-ems/templates/backend-harness-report.md'
    'harness/templates/angular-harness-report.md' = 'packs/nms-ems/templates/angular-harness-report.md'
    'harness/templates/backend-harness-report.md' = 'packs/nms-ems/templates/backend-harness-report.md'
  }
  Get-ChildItem -LiteralPath $roots -Recurse -File -Filter '*.md' | ForEach-Object {
    $text = [System.IO.File]::ReadAllText($_.FullName, [System.Text.UTF8Encoding]::new($false))
    foreach ($entry in $map.GetEnumerator()) { $text = $text.Replace($entry.Key, $entry.Value) }
    [System.IO.File]::WriteAllText($_.FullName, $text, [System.Text.UTF8Encoding]::new($false))
  }
  ```

- [x] **Step 3: skills를 새 Pack 경로만 참조하게 수정한다.**

  `angular-ui-harness`와 `backend-harness`의 playbook·template·knowledge 목록을 `<HARNESS_ROOT>/packs/nms-ems/...`로 치환한다.

  `manual-update`의 NMS UI/component 목록을 `<HARNESS_ROOT>/packs/nms-ems/knowledge/...`로 치환하고 `.ai/manual/` 참조는 유지한다.

  `codex-learning-loop`의 NMS 지식 행을 `<HARNESS_ROOT>/packs/nms-ems/`으로 유지하며 legacy라는 단어를 제거한다.

- [x] **Step 4: 세 소비자 AGENTS.md를 새 경로만 사용하도록 수정한다.**

  두 UEMS AGENTS.md에서는 `Policy/playbooks/templates: <HARNESS_ROOT>/harness/`와 `NMS knowledge: <HARNESS_ROOT>/nms-knowledge/` 항목을 제거한다. 기존 legacy 파일 목록은 다음으로 바꾼다.

  ```markdown
  - `<HARNESS_ROOT>/core/policy/global.md`
  - `<HARNESS_ROOT>/core/policy/security.md`
  - `<HARNESS_ROOT>/core/policy/approval.md`
  - `<HARNESS_ROOT>/core/templates/result-report.md`
  - `<HARNESS_ROOT>/packs/nms-ems/PACK.md`
  - 작업 유형에 맞는 `<HARNESS_ROOT>/packs/nms-ems/knowledge/` 또는 `playbooks/`
  ```

  Playwright AGENTS.md의 Core-first/Pack 조건은 유지하고, legacy 경로를 추가하지 않는다.

- [x] **Step 5: 링크 갱신 후 hash 기준을 새 값으로 기록한다.**

  Manifest의 이관된 행에는 `post_rewrite_sha256` 열을 추가한다. 원본 hash와 다름은 링크 경로 치환으로 인한 예상 변경이며, 파일 누락은 허용하지 않는다.

## Task 4: 삭제 전 링크·민감정보·스킬 검증

**Files:**
- Modify: `docs/superpowers/migration/2026-08-28-legacy-manifest.csv`
- Modify: `docs/superpowers/migration/2026-08-28-legacy-retirement-evidence.md`
- Test: `core/**`, `packs/nms-ems/**`, `skills/**`, 세 소비자 `AGENTS.md`

**Interfaces:**
- Consumes: Task 1~3의 manifest와 갱신된 경로
- Produces: legacy 삭제를 허용하거나 차단하는 명시적 gate 결과

- [x] **Step 1: 현재 사용 범위에 legacy 경로가 남지 않았는지 검색한다.**

  Run:

  ```powershell
  rg -n "\.ai[\\/](nms-knowledge|harness)[\\/](policy|playbooks|templates|frontend|backend|api|components|domain|features|ui)|<HARNESS_ROOT>[\\/](nms-knowledge|harness)[\\/]|nms-knowledge[\\/]|harness[\\/](policy|playbooks|templates)[\\/]" C:\khd\ai-harness\README.md C:\khd\ai-harness\core C:\khd\ai-harness\packs C:\khd\ai-harness\skills C:\khd\playwright\AGENTS.md C:\khd\allproject\uems_fe\AGENTS.md C:\khd\6gworkspace\UnifyEMS\uems_fe\AGENTS.md
  ```

  Expected: no output. Historical specs, plans, and `.harness/runs` are intentionally excluded.

- [x] **Step 2: Pack 파일 수와 파일 존재를 검증한다.**

  Run:

  ```powershell
  (Get-ChildItem -LiteralPath C:\khd\ai-harness\packs\nms-ems\knowledge -Recurse -File -Filter '*.md').Count
  Test-Path C:\khd\ai-harness\packs\nms-ems\playbooks\angular-ui-harness.md
  Test-Path C:\khd\ai-harness\packs\nms-ems\playbooks\backend-harness.md
  Test-Path C:\khd\ai-harness\packs\nms-ems\templates\angular-harness-report.md
  Test-Path C:\khd\ai-harness\packs\nms-ems\templates\backend-harness-report.md
  ```

  Expected: knowledge count 113 and all four paths return `True`.

- [x] **Step 3: 모든 skill의 YAML과 경로를 검증한다.**

  Run:

  ```powershell
  $validator = 'C:\Users\김현동\.codex\skills\.system\skill-creator\scripts\quick_validate.py'
  python -X utf8 $validator C:\khd\ai-harness\skills\safe-change
  python -X utf8 $validator C:\khd\ai-harness\skills\harness-audit
  python -X utf8 $validator C:\khd\ai-harness\skills\codex-learning-loop
  python -X utf8 $validator C:\khd\ai-harness\skills\angular-ui-harness
  python -X utf8 $validator C:\khd\ai-harness\skills\backend-harness
  python -X utf8 $validator C:\khd\ai-harness\skills\manual-update
  ```

  Expected: every command prints `Skill is valid!`.

- [x] **Step 4: 삭제 gate를 evidence에 기록한다.**

  The evidence file must contain these exact keys with actual values: `manifest_row_count`, `knowledge_file_count`, `sensitive_stop_count`, `legacy_reference_count`, `skill_validator_result`, `delete_gate`.

  Set `delete_gate: pass` only when `manifest_row_count` is 126, `knowledge_file_count` is 113, `sensitive_stop_count` is 0, `legacy_reference_count` is 0, and every validator passed.

## Task 5: 정확한 legacy 폴더 삭제와 삭제 후 검증

**Files:**
- Delete: `C:\khd\ai-harness\harness`
- Delete: `C:\khd\ai-harness\nms-knowledge`
- Modify: `docs/superpowers/migration/2026-08-28-legacy-retirement-evidence.md`

**Interfaces:**
- Consumes: `delete_gate: pass`
- Produces: Core/Pack/skills/docs만 남은 최종 하네스 구조

- [x] **Step 1: 삭제 대상 경로와 gate를 다시 확인한다.**

  Run:

  ```powershell
  $root = (Resolve-Path -LiteralPath 'C:\khd\ai-harness').Path
  $targets = @((Resolve-Path -LiteralPath "$root\harness").Path, (Resolve-Path -LiteralPath "$root\nms-knowledge").Path)
  $targets
  Select-String -LiteralPath "$root\docs\superpowers\migration\2026-08-28-legacy-retirement-evidence.md" -SimpleMatch 'delete_gate: pass'
  ```

  Expected: targets are exactly `C:\khd\ai-harness\harness` and `C:\khd\ai-harness\nms-knowledge`; gate is `pass`.

- [x] **Step 2: 두 검증된 legacy 폴더만 삭제한다.**

  Run:

  ```powershell
  Remove-Item -LiteralPath 'C:\khd\ai-harness\harness' -Recurse -Force
  Remove-Item -LiteralPath 'C:\khd\ai-harness\nms-knowledge' -Recurse -Force
  ```

- [x] **Step 3: 삭제 후 구조·링크·placeholder를 다시 검증한다.**

  Run:

  ```powershell
  Test-Path C:\khd\ai-harness\harness
  Test-Path C:\khd\ai-harness\nms-knowledge
  rg -n "\.ai[\\/](nms-knowledge|harness)[\\/](policy|playbooks|templates|frontend|backend|api|components|domain|features|ui)|<HARNESS_ROOT>[\\/](nms-knowledge|harness)[\\/]|nms-knowledge[\\/]|harness[\\/](policy|playbooks|templates)[\\/]" C:\khd\ai-harness\README.md C:\khd\ai-harness\core C:\khd\ai-harness\packs C:\khd\ai-harness\skills C:\khd\playwright\AGENTS.md C:\khd\allproject\uems_fe\AGENTS.md C:\khd\6gworkspace\UnifyEMS\uems_fe\AGENTS.md
  rg -n "TBD|TODO|PLACEHOLDER|implement later|fill in details" C:\khd\ai-harness\core C:\khd\ai-harness\packs\nms-ems\PACK.md C:\khd\ai-harness\packs\nms-ems\policy C:\khd\ai-harness\packs\nms-ems\playbooks C:\khd\ai-harness\packs\nms-ems\templates C:\khd\ai-harness\skills
  ```

  Expected: both `Test-Path` results are `False`; both searches have no output. `packs/nms-ems/knowledge/` is excluded from the placeholder scan because it may accurately document source-code TODO or stub observations.

## Task 6: 완료 기록과 비범위 보고

**Files:**
- Modify: `docs/superpowers/migration/2026-08-28-legacy-retirement-evidence.md`
- Modify: `docs/superpowers/migration/2026-08-28-harness-baseline.md`
- Modify: `C:\khd\ai-harness\.harness\runs\2026-08-28-harness-audit\EVIDENCE.local.md`

**Interfaces:**
- Consumes: Task 5의 삭제 후 검증 결과
- Produces: legacy 제거 완료, 민감정보 판정, 미검증 범위가 명확한 이관 기록

- [x] **Step 1: 기준본의 v0 판정을 최종 구조 판정으로 갱신한다.**

  Replace the v0 legacy-retention statement with:

  ```markdown
  - Core: canonical 사용
  - NMS/EMS Pack: canonical 사용
  - legacy `harness/`, `nms-knowledge/`: 제거 완료
  ```

- [x] **Step 2: 감사 증거에 legacy 제거와 남은 비범위를 기록한다.**

  Record: deleted paths, link scan result, skill validator result, sensitive-information decision, no product build/browser/API execution, and no Git commit because the root is not a repository.

- [x] **Step 3: plan completion criteria를 재확인한다.**

  Verify the final layout has `core/`, `packs/nms-ems/`, `skills/`, `docs/`, and does not have `harness/` or `nms-knowledge/`.

## Execution Completion Criteria

- Legacy source 126 rows are inventoried and each moved file has a matching destination before rewrite.
- Pack knowledge has 113 Markdown files.
- All live references use Core or Pack paths only.
- No unresolved sensitive-information candidate blocks the move.
- All six skills pass UTF-8 validator checks.
- `harness/` and `nms-knowledge/` are absent after the delete gate passes.
- Product build, browser, API, and Git integration remain explicitly unverified or unavailable.
