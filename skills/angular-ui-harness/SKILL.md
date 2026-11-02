---
name: angular-ui-harness
description: 명시적으로 요청된 NMS/EMS Angular UI 하네스의 설계 또는 읽기 전용 분석을 수행할 때 사용한다.
---

# 적용 범위

일반 Angular 기능 수정에는 사용하지 않는다. 그런 요청은 `safe-change`를 주 skill으로 사용하고 필요한 NMS Pack playbook을 읽는다.

## 먼저 읽을 문서

- `<HARNESS_ROOT>/core/playbooks/harness-routing.md`
- `<HARNESS_ROOT>/packs/nms-ems/PACK.md`
- `<HARNESS_ROOT>/packs/nms-ems/playbooks/angular-ui-harness.md`
- `<HARNESS_ROOT>/packs/nms-ems/templates/angular-harness-report.md`

## 절차

1. 현재 source, active site 구성, route, provider, component, service, mock/fixture 소비자를 읽기 전용으로 확인한다.
2. 필요한 frontend, UI, component Pack 지식만 추가로 읽는다.
3. read/write workflow와 mock 전환 지점을 실제 파일 근거로 분리한다.
4. `<HARNESS_ROOT>/packs/nms-ems/templates/angular-harness-report.md` 형식으로 구조, seam, 위험, 검증 계획을 보고한다.
5. 제품 구현이 명시 요청되면 별도의 `safe-change` 흐름에서 승인 범위와 현재 근거를 다시 확인한다.

## 규칙과 검증

- 실제 backend write는 명시 승인 없이는 mock 또는 분석 범위로 둔다.
- active site 구성보다 generic stub를 우선 근거로 사용하지 않는다.
- 결과를 마치기 전 `harness-verify`를 읽어 실행한 분석·검증과 미검증을 분리한다.
