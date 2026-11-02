# NMS/EMS 도메인 안전 규칙

## multi-site concurrency

- 확인할 현재 흐름: 활성 site/product 구성, 요청이 향하는 대상, 공유 상태와 동시 실행 경로
- 보고할 영향: 다른 site의 설정·표시·권한에 미치는 영향과 경합 가능성
- 승인 없이 하지 않을 동작: 여러 site에 동시에 쓰기, 공통 구성 일괄 변경

## scheduler overlap과 queue/backpressure

- 중복 동기화를 유발할 수 있는 변경을 피한다.
- 확인할 현재 흐름: job 등록, trigger 조건, 중복 방지, retry, queue 소비 속도
- 보고할 영향: 중복 실행, 누락, 지연, resource 고갈, 순서 역전 가능성
- 승인 없이 하지 않을 동작: scheduler 강제 실행, 대기열 비우기, 운영 주기 변경

## stale display와 data loss

- 확인할 현재 흐름: 조회 cache, refresh, 저장 시점, 실패 복구, 데이터 보존 기간
- 보고할 영향: 오래된 표시, 부분 저장, 중복 집계, 되돌릴 수 없는 손실 가능성
- 승인 없이 하지 않을 동작: 운영 데이터 삭제, truncate, 수동 정합성 보정

## NE, site, shelf, port mapping

- 확인할 현재 흐름: 식별자 생성, mapping 조회, parent-child 관계, 장비 명령 대상 선택
- 보고할 영향: 잘못된 장비 또는 포트에 적용될 가능성, 화면과 장비 상태의 불일치
- 승인 없이 하지 않을 동작: 실제 장비 명령, mapping 재생성, 대량 설정 반영

## 인증·세션·WebSocket mapping

- 확인할 현재 흐름: Keycloak login, token expiry/refresh, logout, role filtering, WebSocket session mapping
- 보고할 영향: multi-session, stale token, 권한 누수, 연결 해제 후 상태 불일치
- 승인 없이 하지 않을 동작: 인증·권한 우회, 운영 계정으로 진단, 세션 강제 삭제

## real-write 경계

- 확인할 현재 흐름: DB/Mongo write, 장비 명령, GitLab write, ArgoCD/Kubernetes 변경, scheduler trigger의 호출 조건과 승인 상태
- 보고할 영향: 대상 환경, 대상 데이터 또는 장비, rollback 가능성, 비가역성
- 승인 없이 하지 않을 동작: DB update/delete/migration 실행, SNMP/NETCONF 명령, deploy/sync/restart, 운영 장비 제어
