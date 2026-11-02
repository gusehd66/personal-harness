# 범용 승인 정책

## 사람 승인 필요 작업

shared, remote, production-like 상태를 바꿀 수 있는 작업은 실행 전에 명시적인 사람 승인을 받는다.

- `git push`, merge, release branch 변경
- deploy, release, remote pipeline 실행
- DB update, delete, migration, truncate, manual data repair
- remote resource 생성·변경·삭제·restart
- 요청 범위 밖 파일의 제거 또는 광범위한 이동

## 변경 전 보고

승인이 필요한 작업은 다음을 먼저 설명한다.

- 정확한 대상과 영향 범위
- 되돌리는 방법 또는 복구 기준
- 비가역 가능성
- 사전에 수행한 read-only 확인

## 데이터 구조 변경

DB 관련 코드나 schema를 바꾸는 경우에는 migration 필요 여부, 호환성 영향, rollback 고려사항을 분리해 보고한다.
