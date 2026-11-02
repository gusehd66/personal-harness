# DB/Mongo/Migration 지식

## 확인 대상
- `Driver4MybatisIf`
- `Driver4Mybatis`
- `MybatisVariable`
- mapper XML
- `ServerConfig.getMapperList(...)`
- `processDBUpgrade(...)`
- `processDBUpgradeRunning(...)`
- raw query helper
- MongoDB collection/model/insert path

## 분석 순서
1. service별 `ServerConfig*`를 찾는다.
2. `getMapperList(DATABASE aDataBaseType)`에서 mapper XML 등록을 확인한다.
3. `processDBUpgrade(...)`가 어떤 SQL path를 실행하는지 확인한다.
4. worker/process에서 `EmpContext` transaction boundary를 확인한다.
5. MyBatis select/update/delete/insert와 raw query 사용 지점을 분리한다.
6. MongoDB insert/update/delete는 collection name, model, bulk insert 여부를 함께 기록한다.

## 하네스 기준
DB fixture는 아래를 포함한다.

- 초기 MariaDB/MySQL row 상태
- 초기 MongoDB document 상태
- mapper id 또는 raw query
- transaction 경계
- expected write
- rollback 또는 실패 case
- migration 적용 전/후 기대 상태

## 위험 기준
- DB update/delete/truncate
- MongoDB bulk insert/delete
- migration/upgrade SQL
- scheduler가 수행하는 batch insert
- sync 과정의 stale data 처리

이 항목은 기본 mock-write로만 검증하고, real-write는 명시 승인과 rollback 계획이 있어야 한다.
