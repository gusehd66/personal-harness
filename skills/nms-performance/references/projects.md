# 프로젝트 구조 기록 찾기

구조는 대상 BE의 `docs/performance/index.md`를 단일 원본으로 관리한다. 이 skill에는 product 구조를 복제하지 않는다. `<BACKEND_ROOT>`는 사용자 지정 경로 또는 현재 작업 프로젝트에서 확인한다.

## 현재 확인한 프로젝트 계열

- 계열: `back_end` 다중 product Java BE
- 식별 근거: `Platform_framework/`, `uNMS_Common/`, `product/`, `BuildCommon/` 구조
- 기록: `<BACKEND_ROOT>/docs/performance/index.md`
- 가공·저장 상세: `<BACKEND_ROOT>/docs/performance/processing-storage.md`와 해당 product 문서의 가공·저장 절. 파일 및 장비 수집 변경 시 읽는다.
- PM 추가 지점: `<BACKEND_ROOT>/docs/performance/adding-pm.md`. 이 checkout의 사례이며 다른 구조의 product에는 현재 경로를 새로 확인한다.
- 초기 조사 product: `my5G`, `openhaul`, `sixgen-ems`
- 확인 수준: 2026-10-13 소스 정적 확인. 운영 활성 product·등록 상태·DB·장비 실행은 확인하지 않았다.

폴더 이름만 같은 다른 checkout에는 그대로 적용하지 않는다. 기록과 현재 진입점·등록 코드를 대조한다. 루트를 찾지 못하면 경로를 질문하고 멈춘다.

## 추가 또는 갱신

- 새로운 프로젝트는 해당 프로젝트의 `docs/performance/`에 기록하고 이 목록에는 식별 근거만 추가한다.
- 같은 BE의 새로운 product는 index와 해당 product 문서를 추가한다.
- 각 기록에는 확인 날짜, 상대 소스 경로와 핵심 symbol, 공통/전용 경계, 입력·저장·조회 흐름, 실행 구성 근거, 검증 수준을 남긴다.
- 등록 여부나 시간대 등 확인하지 못한 항목은 명시한다. 추정으로 채우지 않는다.
