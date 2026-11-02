# NMS WC 공통 컴포넌트 우선 적용 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 사진 기반 NMS/EMS UI 구현이 native HTML 조립으로 흐르지 않고, 현재 프로젝트의 `wc-*` 공통 컴포넌트와 실제 소비자 패턴을 우선 사용하도록 공용 Pack을 보강한다.

**Architecture:** `PACK.md`는 사진 기반 UI 구현을 일반 화면 구현 흐름으로 명확히 라우팅한다. `frontend-feature.md`는 구현 전 공통 컴포넌트 채택 gate를 강제하고, `wc-usage-map.md`는 확인해야 할 WC API·사용처·예외 근거를 유지한다. 제품별 `AGENTS.md`에는 이 규칙을 중복하지 않는다.

**Tech Stack:** Markdown, Codex shared harness, Angular WCControl

**Spec:** 2026-08-31 사용자 승인 요구 — NMS Pack에 WC 공통 컴포넌트 우선 규칙을 중앙화하고 새 스킬을 만들지 않는다.

## Global Constraints

- 새 skill, 새 plugin, 새 프로젝트 `AGENTS.md` 규칙을 만들지 않는다.
- Core에는 WC 또는 NMS 전용 규칙을 넣지 않는다.
- 사진은 시각 기준일 뿐이며, 실제 source·활성 구성·기존 consumer가 UI 구현 계약의 기준이다.
- native HTML 자체를 금지하지 않는다. WC 내부 구현, 기존 의도된 native 사용, 대응 WC 부재는 source 근거가 있으면 허용한다.
- 실제 `wc-*` selector, API, module/standalone 구성은 작업 대상 프로젝트에서 다시 확인한다.

---

## File Structure

| 경로 | 작업 | 책임 |
| --- | --- | --- |
| `packs/nms-ems/PACK.md` | 수정 | 사진 기반 UI 구현과 설계·분석 요청의 Pack 라우팅을 구분하고 WC 우선 원칙을 선언 |
| `packs/nms-ems/playbooks/frontend-feature.md` | 수정 | 일반 NMS UI 구현 전 공통 컴포넌트 채택 gate와 결과 보고 근거를 추가 |
| `packs/nms-ems/knowledge/components/wc-usage-map.md` | 수정 | `wc-button`을 포함한 화면-level WC 채택·예외 확인 기준을 기록 |

## Task 1: Pack 라우팅과 공통 UI 원칙 고정

**Files:**
- Modify: `packs/nms-ems/PACK.md`

- [x] **Step 1: 작업 유형 표를 설계·분석과 구현으로 구분한다.**

  `NMS UI 설계·분석`은 `angular-ui-harness.md`, `NMS UI 구현·사진 기반 화면 변경`은 `frontend-feature.md`와 WC 지식 문서를 읽도록 적는다.

- [x] **Step 2: 화면-level WC 우선 원칙과 native 예외를 추가한다.**

  사진을 시각 기준으로 한정하고, 대응하는 active-project WC가 있으면 화면에서 native HTML 또는 one-off custom 컴포넌트로 재구현하지 않도록 명시한다.

## Task 2: 일반 UI 구현 절차에 채택 gate 추가

**Files:**
- Modify: `packs/nms-ems/playbooks/frontend-feature.md`

- [x] **Step 1: UI 구현 기준 선택 전에 공통 컴포넌트 채택 gate를 추가한다.**

  유사 화면, WC source/API, 실제 consumer, 상태·이벤트·권한 처리를 먼저 확인하게 한다.

- [x] **Step 2: 결과 보고에 WC 근거와 native 예외를 기록하도록 한다.**

  사용 컴포넌트만 나열하지 않고 source path 또는 consumer와 예외 사유를 남기게 한다.

## Task 3: WC 사용 지도와 검증

**Files:**
- Modify: `packs/nms-ems/knowledge/components/wc-usage-map.md`

- [x] **Step 1: 화면-level WC 채택 원칙과 `wc-button` 근거를 추가한다.**

  `wc-button` source와 sample/consumer 확인을 안내하고, 내부 native `<button>`은 WC 구현 책임임을 분리한다.

- [x] **Step 2: 문서 경로와 규칙 일관성을 검증한다.**

  Run:

  ```powershell
  $validator = 'C:\\Users\\김현동\\.codex\\skills\\.system\\skill-creator\\scripts\\quick_validate.py'
  Get-ChildItem C:\\khd\\ai-harness\\skills -Directory | ForEach-Object {
      python -X utf8 $validator $_.FullName
  }
  rg -n -i "사진 기반|공통 UI|wc-button|native HTML|frontend-feature" packs\\nms-ems
  ```

  Expected: 모든 skill validator가 통과하고, Pack·playbook·usage map에서 같은 채택 원칙과 예외 경계가 확인된다.
