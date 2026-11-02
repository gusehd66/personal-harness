# Legacy Harness 제거 증거

## 삭제 전 기준

- legacy Markdown manifest: 126행
- `harness/`: 13개 Markdown 파일
- `nms-knowledge/`: 113개 Markdown 파일
- manifest: `2026-08-28-legacy-manifest.csv`

## 민감정보 판정

- 값 형식 후보: 7개 행
- `stop`: 0개
- `identifier`: 7개
- `none`: 119개

후보는 모두 NMS parity 문서의 field name 또는 코드 식별자 형식으로 판정했다. 실제 값은 이 문서에 기록하지 않았다.

## Core canonical replacement

다음 legacy 문서는 byte-for-byte 복사하지 않는다. 이미 생성된 Core 문서가 범용 내용을 canonical으로 대체한다.

- `harness/policy/approval.md` → `core/policy/approval.md`
- `harness/policy/security.md` → `core/policy/security.md`와 `packs/nms-ems/policy/domain-safety.md`
- `harness/policy/global.md` → `core/policy/global.md`와 `packs/nms-ems/PACK.md`
- `harness/playbooks/code-review.md` → `core/playbooks/code-review.md`
- `harness/playbooks/long-term-learning.md` → `core/playbooks/long-term-learning.md`
- `harness/templates/result-report.md` → `core/templates/result-report.md`

## 삭제 gate

- manifest_row_count: 126
- knowledge_file_count: 113
- sensitive_stop_count: 0
- legacy_reference_count: 0
- moved_missing_target_count: 0
- skill_validator_result: pass (safe-change, harness-audit, codex-learning-loop, angular-ui-harness, backend-harness, manual-update)
- delete_gate: pass
- delete_execution: completed manually by user after the gate passed

## 삭제 후 검증

- 새 Core와 NMS/EMS Pack 경로는 현재 사용 문서와 세 소비자에 반영됐다.
- legacy 경로 참조 검색은 0건이다.
- `harness_exists: False`
- `nms_knowledge_exists: False`
- 새 하네스 범위의 placeholder 검색은 0건이다. Pack knowledge의 기존 소스 TODO/stub 관찰은 분석 사실이므로 검사 대상에서 제외했다.
- 여섯 개 skill UTF-8 validator 결과는 모두 pass다.
- 제품 build, browser, API 실행과 Git commit은 이 문서 이관 범위 밖이다. 루트는 정상 Git 저장소가 아니다.

## 현재 상태

- legacy `harness/`, `nms-knowledge/`는 제거 완료다.
- Pack knowledge의 기존 소스 TODO/stub 관찰은 분석 사실이며, 새 하네스 문서 placeholder 검사 대상에서 제외한다.
