---
name: safe-change
description: 코드, 설정, 문서, 자동화 변경 요청에서 현재 흐름을 확인하고 Core·필요 시 NMS Pack 기준으로 최소 변경을 수행할 때 사용한다.
---

# 목적

일반 변경 요청의 주 skill으로 현재 동작과 소비자를 근거로 작은 변경을 수행한다.

# 절차

1. `<HARNESS_ROOT>/core/playbooks/harness-routing.md`를 읽고 프로젝트 `AGENTS.md`가 있으면 함께 확인하여 요청이 일반 변경인지 전문 UI/backend/manual 요청인지 분류한다.
2. 전문 요청이면 해당 전문 skill을 주 skill으로 사용하고 이 skill을 중복 적용하지 않는다.
3. 일반 변경이면 `<HARNESS_ROOT>/core/policy/`와 `<HARNESS_ROOT>/core/playbooks/safe-change.md`를 읽는다.
4. NMS/EMS 사실이 판단 또는 검증에 영향을 주면 `<HARNESS_ROOT>/packs/nms-ems/PACK.md`와 필요한 Pack 문서를 읽는다.
5. 프로젝트의 읽기·쓰기 경로, 활성 구성, 실제 소비자, 관련 기존 변경을 확인하고 필요한 작업 기록 범위를 정한다.
6. 기존 구조를 보존하는 가장 작은 수정을 적용한다.
7. 완료 전 `harness-verify`를 읽어 실제 검증과 미검증을 분리한다.
8. 재사용 가치가 현재 근거로 확인된 경우에만 `harness-retro`를 읽어 지식 저장 위치를 판단한다.

# 규칙

- `.harness/runs/`은 작업 이력이며 공식 사양이 아니다.
- password, token, cookie, 개인정보, 운영 데이터, 실제 장비 식별값은 실행 기록에 남기지 않는다.
- 실패 기록을 삭제하거나 성공으로 바꾸지 않는다.
- 프로젝트 루트를 확정할 수 없으면 실행 기록을 만들지 않고 루트 확인을 요청한다.
- 같은 작업 이름 기록이 이미 있으면 기존 기록을 덮어쓰지 않고 구분 가능한 새 이름을 사용한다.
- 배포, 실제 데이터 변경, 장비 제어, 원격 상태 변경은 관련 승인 정책을 따른다.
