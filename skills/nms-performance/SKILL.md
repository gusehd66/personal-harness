---
name: nms-performance
description: NMS/EMS 백엔드의 Performance(PM) 수집, 파싱, 저장, 집계, 조회, 보존 기능을 분석·수정·리뷰할 때 사용한다. product별 활성 구현과 데이터 흐름을 확인하며 일반 CPU·메모리 성능 최적화나 FE 전용 작업에는 적용하지 않는다.
---

# NMS Performance 백엔드

## 대상 확정

- 현재 프로젝트의 AGENTS.md가 있으면 이를 확인하고 사용자 요청을 먼저 확인한다. `<BACKEND_ROOT>`는 대상 BE 루트다. 특정 PC 경로나 product를 기본값으로 고정하지 않는다.
- 대상 product, NE 종류, EMS/NOC 모드, 문제 또는 변경할 PM 경로를 확정한다. 질문 없이 현재 소스·실행 구성에서 확인할 수 있는 것은 먼저 확인한다.
- 여러 product가 있고 대상에 따라 결론이나 수정 범위가 달라지면 질문하고 답변 전에는 해당 작업을 멈춘다. 답변이 없다고 기본 product를 선택하지 않는다.
- [프로젝트 구조 기록](references/projects.md)에서 해당 기록을 찾고 관련 product 문서만 읽는다. 기록이 없으면 현재 소스에서 조사한다. 다른 프로젝트 기록을 그대로 적용하지 않는다.
- 수집·가공·저장 작업에서는 프로젝트의 가공·저장 상세 문서도 읽는다. parser 반환값, DB 저장 완료, 파일 이동 완료를 별도 상태로 확인하고 실제 모델의 변환·집계·collection 계약까지 추적한다.
- PM 항목 추가 또는 추가 방법 분석은 [항목 확장 절차](references/add-pm.md)를 읽는다. 대상 product의 실제 구조를 먼저 확인하며 유사 PM은 구조가 맞을 때만 구현 참고로 사용한다.

## 작업 방식

### 공통 playbook 연결

`<HARNESS_ROOT>`는 이 skill 원본을 포함하는 하네스 루트다. Codex 등록 경로가 Junction/심볼릭 링크이면 링크 대상의 원본 경로에서 확인한다. 아래 문서는 요청 유형에 해당하는 것만 읽으며, PM 전문 절차와 프로젝트별 현재 소스 근거를 함께 적용한다.

| 적용 시점 | 읽을 문서 | PM 작업에서의 역할 |
| --- | --- | --- |
| 작업 시작 | `<HARNESS_ROOT>/core/playbooks/harness-routing.md`, `core/policy/`, `packs/nms-ems/PACK.md` | 요청 범위, 공통 정책과 NMS 도메인 기준 확인 |
| 코드·설정·문서 변경 | `<HARNESS_ROOT>/core/playbooks/safe-change.md` | 최소 변경, 경량/전체 작업 기록, 검증 범위 결정 |
| PM 버그 수정 | `<HARNESS_ROOT>/packs/nms-ems/playbooks/bugfix.md` | 증상과 근본 원인, 활성 분기와 관련 소비자 추적 |
| PM 코드 리뷰 | `<HARNESS_ROOT>/core/playbooks/code-review.md` | 심각도와 파일 근거로 회귀·데이터 손실·검증 공백 보고 |
| 운영 write 관련 | `<HARNESS_ROOT>/packs/nms-ems/policy/domain-safety.md` | DB·장비·scheduler·배포 동작의 승인과 영향 확인 |

이 skill을 주 skill으로 유지하고 `safe-change` skill을 중복 호출하지 않는다. playbook의 범용 절차는 위임하며 PM의 모델·시간·집계·저장 계약은 아래 절차에서 확인한다. FE 전용 항목은 BE에 기계적으로 적용하지 않는다. 분석·리뷰만 요청되면 playbook을 읽었다는 이유로 수정이나 운영 실행을 시작하지 않는다.

### PM 흐름 확인과 검증

1. 기록의 주요 진입점과 대상 소스를 대조한다. product 폴더명, 서비스명, EmsTypecode, NE code는 서로 같다고 가정하지 않는다. 시작 클래스, classpath/JAR, 함수 등록과 초기화까지 확인하며 클래스가 존재하는 것과 실제 실행 중인 것을 구분한다.
2. 요청에 관련된 흐름만 추적한다: scheduler/외부 입력 → product 함수·parser → 데이터 모델·저장 → 조회 API. 조회 문제는 반대 방향으로 추적해도 된다. 공통 서비스와 product 구현, EMS/NOC 분기를 구분한다.
3. 수정할 함수의 호출자·등록 지점·mapper·모델·기존 테스트를 찾는다. 공통 코드 수정이면 영향받는 다른 product를 확인한다. 기존 구조와 예외 처리, 트랜잭션 경계를 유지하는 최소 변경을 한다.
4. 변경에 관련된 PM 계약을 확인한다: 수집 시각과 조회 시각의 시간대, 구간 경계와 집계 주기, counter/gauge와 단위, NE·port·index 식별자, 결측·LOS·0 구분. 주기·키·집계 방식은 현재 구현과 사양에서 확인하며 공통값을 만들어 넣지 않는다.
5. 수집 변경은 중복 실행·재시작·재처리·queue 적체·부분 저장·성공/실패 파일 이동을, 저장/조회 변경은 collection·mapper·필터·정렬·범위 경계를, 보존 변경은 삭제 대상과 시간 기준을 확인한다. 작업과 관계없는 전체 체크를 반복하지 않는다.
6. 버그는 가능한 한 기존 테스트 방식으로 실패 재현과 회귀 검증을 남긴다. parser는 비식별 fixture와 저장 대역을 사용한다. 실제 DB·장비·scheduler·서비스 main을 테스트 대신 실행하지 않는다. 빌드 명령은 현재 build 파일과 부작용을 확인한 뒤 선택한다.
7. 변경 또는 전문 분석 완료 전 `<HARNESS_ROOT>/skills/harness-verify/SKILL.md`를 읽어 실제 검증·실패·미검증을 분리한다. 근본 원인, 변경 파일, product별 영향을 보고한다. 구조 변경이 확인되면 허용된 변경 범위에서 해당 프로젝트 구조 문서만 갱신한다. 소스와 기록이 다르면 현재 소스를 우선하고 확인 날짜·근거·남은 불확실성을 수정한다.
8. 현재 근거로 확인된 재사용 지식이 있을 때만 `<HARNESS_ROOT>/skills/harness-retro/SKILL.md`와 연결된 지식 저장 playbook을 읽는다. product별 사실은 해당 프로젝트 문서에, 반복 절차는 skill에 유지하며 같은 내용을 중복 저장하지 않는다.

## 안전과 기록

- 운영 DB/Mongo 쓰기·삭제·migration, 장비 명령, scheduler 강제 실행, 배포는 해당 동작에 대한 명시 승인 없이 수행하지 않는다. 이미 받은 승인은 중복 요청하지 않는다.
- 민감정보·실제 장비 식별값·원본 운영 PM 데이터는 skill과 구조 문서에 복사하지 않는다.
- 구조 기록은 프로젝트별 사실이다. 한 product의 경로·시간대·보존 주기·등록 방식을 다른 product의 규칙으로 승격하지 않는다.
- 공통 playbook의 절차와 프로젝트의 필수 정책을 따른다. 프로젝트의 source·활성 구성·로그·실행 근거가 구조 기록과 예시보다 우선한다.
