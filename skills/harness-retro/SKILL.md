---
name: harness-retro
description: 완료된 작업에서 현재 근거로 검증된 교훈을 프로젝트, Core, Pack, skill 후보 중 가장 작은 저장 위치로 분류할 때 사용한다.
---

# 목적

작업 중 얻은 사실을 무분별하게 전역화하지 않고, 다음 작업에 실제로 도움이 될 최소 범위의 지식만 남긴다.

## 먼저 읽을 문서

- `<HARNESS_ROOT>/core/playbooks/long-term-learning.md`
- NMS/EMS 사실이면 `<HARNESS_ROOT>/packs/nms-ems/PACK.md`

# 절차

1. 작업 기록과 `harness-verify`의 실제 근거를 확인한다.
2. Core playbook의 기준으로 가장 작은 저장 표면을 선택한다.
3. 선택한 문서가 이미 같은 사실을 갖는지 확인하고 중복 추가를 피한다.
4. 업데이트가 필요 없으면 이유를 결과에 남긴다.

# 규칙

- 추측, 미검증 로그 해석, 특정 site 예외를 Core나 Pack에 승격하지 않는다.
- password, token, cookie, 개인정보, 고객 데이터, 운영 식별값을 저장하지 않는다.
- Codex memory는 사용자가 명시적으로 요청한 경우에만 현재 세션의 memory 규칙을 따른다.
- 자동 승격하지 않는다. 문서 변경에는 해당 작업의 승인 범위를 적용한다.
