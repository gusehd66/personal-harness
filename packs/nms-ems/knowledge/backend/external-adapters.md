# External Adapter 지식

## 주요 외부 연동
- Keycloak admin/client role/session/token
- GitLab release information read/write
- ArgoCD/Kubernetes workload/version/upgrade
- SNMP/NETCONF/Telnet/SSH 장비 통신
- REST outbound service 호출
- WebSocket/session mapping
- file download/upload/SFTP

## 분석 순서
1. 외부 client class를 찾는다.
2. endpoint, credential source, token/header 구성 방식을 확인한다.
3. read path와 write path를 분리한다.
4. payload shape와 response shape를 기록한다.
5. timeout/retry/permission/error body 처리 방식을 확인한다.
6. 실제 운영 endpoint나 secret은 문서에 남기지 않고 마스킹한다.

## 하네스 기준
adapter fixture는 아래를 포함한다.

- 외부 service type
- 호출 method
- endpoint path pattern
- request payload
- response payload
- 인증 header/token은 마스킹된 placeholder
- success/error/timeout/retry/permission case

## 특별 주의
- Keycloak은 list/mapping/detail endpoint payload shape가 다를 수 있다.
- role name path segment는 encoding ownership을 확인해야 한다.
- GitLab은 read path와 write path를 분리해야 한다.
- ArgoCD/Kubernetes 변경은 upgrade/prune/rollback 영향이 크므로 real-write 금지 기본값을 유지한다.
- SNMP/NETCONF command는 장비 상태 변경 가능성이 있으므로 mock command result를 기본으로 사용한다.
