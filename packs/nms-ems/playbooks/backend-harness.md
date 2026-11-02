# Backend 하네스 플레이북

## 목적
팀원이 AI에게 Java 기반 NMS/EMS backend를 분석시키고, 전체 기능의 개발/수정/삭제/검증을 반복 가능한 방식으로 수행할 수 있도록 공용 backend 하네스 기준을 제공한다.

기본 대상은 전체 backend 기능이다. 사용자가 범위를 명시적으로 좁히지 않으면 Framework, Platform, uNMS Common, product 계층과 모든 product profile을 함께 분석한다.

## 문서 언어 기준
- 하네스 문서, 분석 보고서, 템플릿, 스킬 지시문은 한글로 작성한다.
- Java class, package, method, request type, service name, file path는 원문을 유지한다.
- 기술 용어는 코드 추적에 필요한 경우에만 영문 원어를 그대로 둔다.

## 작업 경계
- 공용 하네스 갱신 작업에서는 제품 backend source tree를 수정하지 않는다.
- 실제 코드 변경 요청이 있을 때만 `<BACKEND_ROOT>`를 수정한다.
- 모든 판단은 실제 파일 경로를 근거로 남긴다.
- 민감값, token, password, 운영 endpoint, 실데이터는 문서에 그대로 남기지 않는다.

## 공통영역 변경 원칙
- `Framework`, `Platform_framework`, `uNMS_Common`, `BuildCommon`, 공통 `library`는 여러 product가 함께 사용하는 공통영역이다.
- 공통영역 구조 변경은 기본적으로 피한다.
- 공통영역을 바꾸면 my5G, openhaul, sixgen-ems 및 향후 추가 product까지 side effect가 생길 수 있다고 가정한다.
- 공통영역 변경이 불가피하면 변경 전 아래 항목을 보고한다.
  - 왜 product 단 구현으로 해결할 수 없는가
  - 영향받는 product profile
  - 영향받는 service/start class/worker/function/request type
  - DB/Mongo/external adapter/scheduler side effect
  - rollback 또는 compatibility 방안
- product별 동작 차이를 나눌 수 있으면 공통영역 직접 수정보다 product 단 class에서 상속, override, adapter wrapper, product-specific `ServerConfig`, product-specific worker/function으로 구현한다.
- 기존 공통 base class에 hook이 있다면 새 공통 구조를 만들기 전에 그 hook을 우선 사용한다.

## 표준 프롬프트
팀 전체가 반복 가능한 backend 분석 결과를 원할 때 아래 프롬프트를 사용한다.

```text
<BACKEND_ROOT>의 Java NMS/EMS backend 전체 기능을 분석하고 공용 backend 하네스 설계안을 작성해줘.

<BACKEND_ROOT>는 수정하지 마.
먼저 실제 소스 구조를 읽어.
모든 판단은 실제 파일 경로를 근거로 남겨.
하네스 문서와 결과 보고서는 한글로 작성해.

분석할 항목:
1. Framework, Platform, uNMS Common, product 계층 구조
2. 서비스별 책임과 입력/출력/소유 데이터
3. my5G, openhaul, sixgen-ems product profile
4. service 시작 구조와 실행 mode
5. manual worker loader와 WorkflowMap 등록 흐름
6. REST content/request type 등록 구조
7. 주요 worker/process/function 흐름
8. DB, MongoDB, mapper, migration, DB upgrade 구조
9. Keycloak, GitLab, ArgoCD/Kubernetes, SNMP, NETCONF, REST outbound adapter 구조
10. scheduler, PM, alarm, sync, session 흐름
11. mock 처리 가능한 지점
12. 기능 단독 실행이 어려운 이유
13. 하네스로 분리 가능한 단위
14. 하네스 구현 시 영향받는 파일

설계할 항목:
1. backend harness profile 구조
2. service responsibility harness 구조
3. service startup harness 구조
4. REST/request type contract harness 구조
5. worker/process harness 구조
6. DB/Mongo fixture 구조
7. external adapter mock/fake 구조
8. scheduler harness 구조
9. 기존 구현과 mock/fake 전환 방식
10. 공통영역 변경 회피 또는 product 단 확장 설계
11. 운영 빌드 영향 최소화 방안
12. 구현 순서

기본 전제:
- backend 전체 기능을 기본 대상으로 한다.
- product profile은 my5G, openhaul, sixgen-ems 전체를 포함한다.
- Framework, Platform, uNMS Common, BuildCommon, 공통 library는 모든 product에 영향을 줄 수 있는 공통영역으로 취급하고 구조 변경을 최소화한다.
- product별로 나눌 수 있는 기능은 공통영역 직접 수정 대신 product 단 상속/override/adapter wrapper 구현을 우선 검토한다.
- 기능 추가/수정/삭제를 위한 read-write 흐름을 포함한다.
- 단, 사용자가 실제 backend write를 명시적으로 허용하지 않으면 모든 write 동작은 mock 기반으로 설계한다.
- fixture는 요청 payload, WorkflowMap 등록 상태, DB/Mongo 상태, 외부 adapter 응답, write 이후 기대 상태를 재현하기 위한 고정 scenario data로 설명한다.
- 재사용 규칙은 제품 소스가 아니라 <HARNESS_ROOT>에 둔다.

원하는 결과:
- 분석 요약
- 추천 backend 하네스 구조
- 파일 추가/수정 계획
- 위험 요소
- 검증 계획
- 구현 전에 확인 필요한 질문
```

## 필수 분석 순서
1. `<BACKEND_ROOT>` 최상위 디렉터리에서 `BuildCommon`, `library`, `Platform_framework`, `uNMS_Common`, `product` 존재 여부를 확인한다.
2. `product` 아래 product profile을 확인한다. 기본 profile은 `my5G`, `openhaul`, `sixgen-ems`이다.
3. 서비스 이름으로 `ServiceStart_*` 또는 `ServerStart_*`를 찾아 서비스 책임, 입력, 출력, 소유 데이터를 먼저 기록한다.
4. `ServerStart_*` class를 찾아 service name, run mode, debug mode, DB/Mongo/session 설정, `runMicroService.run(...)` 인자를 확인한다.
5. `serverRunningConfig.addManualLoadWorker(...)`와 manual worker loader class를 추적한다.
6. manual worker loader가 `WorkflowMap.setWorker(...)`, `WorkflowMap.addFunction(...)`로 등록하는 worker/function을 열거한다.
7. `WorkflowMap`의 worker registry, function registry, REST invoker map, scheduler map, DB driver 접근 방식을 확인한다.
8. `RestContentBook`, `RestContentManagementFunction`, `AutoRestContentBook`, `new RestContent(...)`, `RequestTypeStringBook`, `RestServiceNameBook`을 따라 REST contract를 추출한다.
9. request type별로 invoker function, worker, process, DB/external adapter 호출 체인을 작성한다.
10. `ServerConfig`, `getMapperList(...)`, `processDBUpgrade(...)`, mapper XML, raw query 사용 지점을 확인한다.
11. Keycloak, GitLab, ArgoCD/Kubernetes, SNMP, NETCONF, REST outbound, WebSocket, scheduler 실행 지점을 위험도별로 분리한다.
12. mock/fake로 바꿀 수 있는 지점과 직접 생성, static singleton, global config 때문에 어려운 지점을 구분한다.
13. 변경 후보가 공통영역인지 product 영역인지 분리한다.
14. 공통영역 변경이 필요해 보이면 product 단 상속/override/adapter wrapper로 대체 가능한지 먼저 설계한다.
15. 운영 동작을 바꾸지 않는 하네스 구조를 설계한다.

## 핵심 근거 파일 유형
- 시작점: `ServerStart_*.java`
- 실행 설정: `serverRunningConfig`, `hframeworkCD.frameworkGlobal`, `propertyCodeBook.server_running_config`
- service 실행: `runMicroService`, `runMicroService3`
- 등록 지점: `manualWorkerLoader_*`, `WorkflowMap`
- REST contract: `RestContentBook_*`, `RestContentManagementFunction_*`, `AutoRestContentBook`, `RestContent`
- 업무 처리: `Worker*`, `RestInvokerFunction*`, `Function*`, `Process*`
- DB 설정: `ServerConfig*`, `getMapperList(...)`, mapper XML, `Driver4MybatisIf`
- migration: `processDBUpgrade(...)`, `processDBUpgradeRunning(...)`, SQL file path
- 외부 연동: Keycloak process, GitLab client, ArgoCD/Kubernetes client, SNMP/NETCONF/Telnet/REST adapter
- scheduler: `ScheduleJobIf`, `ScheduleAdapterRat`, `*SchedulerJob`

## 서비스별 지식 문서
서비스 책임과 주요 흐름은 `<HARNESS_ROOT>/packs/nms-ems/knowledge/backend/services/index.md`에서 시작한다.

1. 서비스 이름이 명확하면 해당 서비스 문서를 먼저 읽는다.
2. 서비스 이름이 불명확하면 `services/index.md`의 서비스 목록으로 후보를 좁힌다.
3. product별 차이는 `product-profiles.md`를 함께 확인한다.
4. PM/Performance는 `services/performance.md`를 확인한다.

## 추천 하네스 구조

```text
backend-harness/
  profiles/
    all-products.profile.yml
    my5g.profile.yml
    openhaul.profile.yml
    sixgen-ems.profile.yml
  registry/
    service-startup.registry.json
    rest-content.registry.json
    worker-function.registry.json
    scheduler.registry.json
    external-adapter.registry.json
  fixtures/
    common/
      workflow-map.fixture.json
      emp-context.fixture.json
      db-state.fixture.json
      mongo-state.fixture.json
      write-result.fixture.json
    my5g/
    openhaul/
    sixgen-ems/
  mocks/
    mock-mybatis-driver.md
    mock-mongo-store.md
    mock-keycloak-admin.md
    mock-gitlab-client.md
    mock-argocd-k8s.md
    mock-snmp-netconf.md
    mock-rest-content-manager.md
  reports/
    backend-analysis-report.md
```

이 구조는 실제 제품 repo에 바로 추가하는 구현안이 아니라, 공용 하네스 설계를 반복 가능하게 만드는 기준 구조이다. 실제 구현 시에는 제품 repo의 build/runtime 제약을 다시 확인한다.

## Fixture 정의
backend fixture는 기능 흐름을 재현하기 위한 고정 scenario data이다.

포함 항목:
- 요청 payload와 header
- service name, request type, version, HTTP method
- `WorkflowMap`에 등록된 worker/function 상태
- `serverRunningConfig`와 실행 mode
- DB/Mongo 초기 상태
- 외부 adapter 응답
- scheduler fire time과 대상 NE/site/PM/alarm 상태
- write 이후 기대 DB/Mongo/응답/UI-visible 상태
- 오류, timeout, retry, permission, stale data case

## 읽기/쓰기 안전 규칙
- 기본 write mode는 `mock-write`이다.
- 실제 DB update, Mongo insert, Keycloak 변경, GitLab write, ArgoCD/Kubernetes 변경, SNMP/NETCONF command, scheduler trigger는 명시적 opt-in일 때만 허용한다.
- write scenario는 요청 전 상태, 요청 payload, expected request shape, mock response, write 이후 기대 상태, rollback/실패 경로를 함께 정의한다.
- scheduler와 sync는 중복 실행, queue/backpressure, stale display, 데이터 손실 가능성을 반드시 보고한다.
- 운영 credential, token, password, kubeconfig, NE 접속 정보는 fixture에 실값으로 넣지 않는다.

## Mock/Fake 전환 규칙
우선순위:
1. worker/process 생성자나 setter로 주입 가능한 dependency를 fake로 대체한다.
2. `WorkerRestContentManager` 또는 REST outbound adapter를 mock한다.
3. `Driver4MybatisIf`, Mongo access layer, external client를 fake로 대체한다.
4. `WorkflowMap`과 `serverRunningConfig`는 harness fixture로 초기화한다.
5. static/global 접근은 최소 wrapper 또는 harness-only 초기화 단계에서만 격리한다.

운영 class를 전역 수정하지 않는다. 전환 지점은 harness-only profile, test fixture, adapter wrapper 안에 둔다.

## Product 단 확장 우선 설계
기능 추가/수정/삭제 설계 시 아래 순서로 판단한다.

1. product 단 `ServerStart_*`, manual worker loader, `ServerConfig`, `RestContentBook`, `WorkerRestContentManager`, worker/function override로 해결 가능한지 확인한다.
2. product-specific worker/process class를 추가하거나 기존 base class를 상속해 override할 수 있는지 확인한다.
3. 외부 adapter 차이는 공통 adapter를 직접 바꾸기보다 product 단 wrapper 또는 fake/mock 교체 지점으로 분리한다.
4. 공통영역 base class에 hook이 없어서 확장이 불가능한 경우에만 공통영역의 최소 hook 추가를 검토한다.
5. 공통 hook 추가 시 default behavior가 기존 모든 product에서 동일하게 유지되는지 검증 계획을 작성한다.

공통영역 변경 보고에는 반드시 side effect 분석을 포함한다.

- 영향받는 product: my5G, openhaul, sixgen-ems
- 영향받는 service: service name, start class, worker loader
- 영향받는 REST: request type, `RestContent`, HTTP method, version
- 영향받는 runtime: debug mode, microservice mode, NOC/uEMS 분기
- 영향받는 데이터: DB/Mongo/migration/scheduler/external adapter

## 결과물 계약
모든 backend 하네스 분석 또는 구현 계획은 아래 항목을 포함한다.

- 분석한 backend root
- product profile 목록
- 서비스별 책임/입력/출력/소유 데이터
- 시작 class와 service name 표
- run mode/debug mode/DB/Mongo/session 설정
- manual worker loader 등록 표
- `WorkflowMap` worker/function/rest/scheduler 등록 요약
- REST content/request type contract 표
- 주요 worker/process call-chain
- DB/Mongo/mapper/migration 흐름
- external adapter map
- scheduler map
- fixture map
- mock/fake 전환 지점
- read-write scenario map
- 공통영역 변경 여부와 product 단 확장 가능성
- 공통영역 변경 시 side effect 분석
- 운영 영향 설명
- 정확한 파일 추가/수정 계획
- 검증 명령 또는 수동 검증 경로
- 미확인 질문

## 검증 기준
문서 전용 하네스 갱신:
- 변경 파일을 다시 읽는다.
- 제품 build 또는 test가 통과했다고 말하지 않는다.

향후 제품 backend 하네스 구현:
- 실제 repo의 build 단위와 classpath 구성을 먼저 확인한다.
- Maven module이 없는 product/service 디렉터리는 시작 class와 runtime classpath 기준으로 검증 계획을 세운다.
- DB/Mongo/외부 adapter를 mock으로 분리한 상태에서 read/write scenario를 검증한다.
- real-write 검증은 명시 승인과 rollback 계획 없이는 수행하지 않는다.
