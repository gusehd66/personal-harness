# Preference 화면 구조

## 사용 시점
- `Preference` 화면을 설명하거나 매뉴얼 캡처를 갱신할 때 사용한다.
- 설정 화면의 내부 tree, form, table 구성을 조회 기준으로 확인할 때 사용한다.
- 설정값 변경 위험이 있는 화면을 안전하게 조사해야 할 때 사용한다.

## 기본 구조
`Preference`는 좌측 product menu 하단에서 진입하며, 내부 설정 tree와 우측 설정 content로 구성된다.

```text
Content Title: Preference
Main Panel
  - left preference tree
  - right form/table/content panel
```

## 현재 서버 관찰 기준
- 확인일: 2026-06-10
- 기준 화면: `6G Flexible EMS`
- 조사 방식: 조회 기준으로만 확인. 저장, 적용, 삭제, 다운로드, action 실행은 하지 않음.

## 내부 Tree
관찰된 Preference 내부 tree:

- `General`
  - `FTP Server`
  - `DB Backup`
  - `Storage Retention Period`
- `Fault`
  - `Alarm Definition`
- `System`
  - `Log Level`
  - `Server Log Download`
  - `System Info`

## 화면별 관찰
| 내부 메뉴 | URL | UI 유형 | 관찰 기준 |
| --- | --- | --- | --- |
| `General > FTP Server` | `/unms/preference/general/ftp-server` | Form | `FTP Server (SFTP)`, `Address`, `Port` 항목을 표시한다. |
| `General > DB Backup` | `/unms/preference/general/database-backup` | Form/Empty State | 화면에 표시되는 항목이 적다. 실제 동작은 코드와 backend contract를 확인한다. |
| `General > Storage Retention Period` | `/unms/preference/general/storage-period` | Form | `DB Storage Retention Period`, `Alarm`, `Event`, `PM History`, `Days` 항목을 표시한다. |
| `Fault > Alarm Definition` | `/unms/preference/fault/alarm-definition` | Table/List | `Total List : 6 rows`, `Alarm Type`, `Model`, `Reason`, `SeverityDefinition`, `NMS Severity`, `Category`, `Auto Acknowledge`, `Description`, `Action`. |
| `System > Log Level` | `/unms/preference/system/log-level` | Matrix/Table | `System`, `fatal`, `error`, `warning`, `info`, `debug`, `trace` 기준의 log level 설정 table을 표시한다. |
| `System > Server Log Download` | `/unms/preference/system/server-log-download` | Table/Action | `System`, `Action` column이 있다. 조사 중 download/action은 실행하지 않는다. |
| `System > System Info` | `/unms/preference/system-info` | Info | 시스템 정보 조회 화면이다. 표시 항목은 서버 상태에 따라 달라질 수 있다. |

## 변경 전 확인
- Preference는 설정 변경 위험이 크므로 form value, radio, checkbox, action button 조작을 최소화한다.
- 화면 캡처는 내부 tree, page title, 주요 label/table header가 보이도록 잡는다.
- log level, retention period, FTP/SFTP, DB backup은 운영 영향이 있으므로 변경 시 사람 승인과 영향 범위 설명이 필요하다.
- `Action` column 또는 download 버튼은 조회 작업 중 클릭하지 않는다.

## 금지
- 설정값을 저장하거나 적용하지 않는다.
- log level, retention period, backup, FTP/SFTP 설정을 변경하지 않는다.
- 서버 로그 다운로드 같은 action은 명시 요청과 승인 없이 실행하지 않는다.
