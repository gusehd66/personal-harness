# Backend 구조 지식

## 기본 계층
NMS/EMS backend는 일반적인 단일 Java web application 구조보다 아래 계층을 함께 봐야 한다.

- `BuildCommon`: build, launch, command, YANG/source generation 관련 공통 자원
- `library`: 공통 Java library, network, data, process, Keycloak, scheduler, SNMP, NETCONF, REST, MongoDB, MyBatis 관련 모듈
- `Platform_framework`: framework runtime, `WorkflowMap`, server, adapter, scheduler, platform 공통 기능
- `uNMS_Common`: uNMS 공통 microservice, common plugin, common data, UserManagement/SystemManagement/Performance 등 공통 service
- `product`: my5G, openhaul, sixgen-ems 등 product별 start project와 service/plugin 구현

## 분석 원칙
- build file만 보고 구조를 확정하지 않는다. product/service 중에는 Maven module처럼 보이지 않지만 runtime classpath로 로딩되는 코드가 있다.
- 먼저 시작 class와 runtime 등록 흐름을 확인한다.
- `WorkflowMap`은 worker, function, REST invoker, scheduler, DB driver가 모이는 핵심 registry이다.
- REST endpoint는 annotation보다 `RestContent`, `RequestTypeStringBook`, service name, version, HTTP method 등록 흐름으로 파악한다.

## 공통영역 변경 원칙
- `BuildCommon`, 공통 `library`, `Platform_framework`, `uNMS_Common`은 여러 product가 함께 사용하는 공통영역이다.
- 공통영역 변경은 모든 product에 영향을 줄 수 있으므로 구조 변경을 최소화한다.
- 공통영역을 바꾸기 전에 my5G, openhaul, sixgen-ems 전체 product profile의 side effect를 확인한다.
- product별 요구가 다르면 공통영역 직접 수정보다 product 단 구현을 우선한다.
- 우선순위는 product-specific class 상속, method override, adapter wrapper, product-specific `ServerConfig`, product-specific worker/function 순서이다.
- 공통 base class에 hook이 없어서 확장이 불가능한 경우에만 default behavior를 유지하는 최소 hook 추가를 검토한다.
- 공통영역 변경이 필요한 경우 영향받는 start class, manual worker loader, `WorkflowMap` 등록, `RestContent`, DB/Mongo, scheduler, external adapter를 함께 보고한다.

## 실제 코드에서 우선 확인할 파일
- `<BACKEND_ROOT>/Platform_framework/Framework4/src/main/java/hfr/framework/server/WorkflowMap.java`
- `<BACKEND_ROOT>/Platform_framework/EMSPlatform/src/hfr/platform/Rat/product/server/app/microservice/runMicroService.java`
- `<BACKEND_ROOT>/Platform_framework/EMSPlatform/src/hfr/platform/Rat/product/server/app/jersey/MicroServiceRestInvoker.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.Common/src/main/java/hfr/common/worker/WorkerRestContentManager.java`
- `<BACKEND_ROOT>/uNMS_Common/uEMS.microservice/uEMS.plugin.my5g.Common/src/main/java/hfr/plugin/my5g/common/data/autogenerate/AutoRestContentBook.java`

## 하네스 관점
backend 하네스는 “서버를 다 띄우는 도구”가 아니라, 등록된 service/function/worker/process/DB/external adapter 흐름을 재현하고 검증하는 기준이다.

하네스 설계는 공통영역을 직접 바꾸는 방향보다 product 단에서 상속/override/fake/mock으로 분리하는 방향을 먼저 제시해야 한다.
