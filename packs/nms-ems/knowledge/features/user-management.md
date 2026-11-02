# User Management 기능 노트

## 사용 시점
Local User, Role, authority, login, password, account, user validation 변경 시 이 문서를 사용한다.

## 현재 FE 주의점
- Local User create/edit flow는 기존 validation order를 유지해야 한다.
- blank, null, duplicate account, duplicate email case를 명시적으로 처리해야 한다.
- Role과 menu visibility는 `authorityCheck`, provider registration, runtime site configuration에 따라 달라질 수 있다.

## 변경 전 확인
- form initialization path와 submit path를 추적한다.
- validation이 create-only, edit-only, shared 중 무엇인지 확인한다.
- menu 또는 button을 보이게 하려고 authority check를 우회하지 않는다.

## 위험
- 잘못된 duplicate validation은 정상 사용자를 막거나 충돌 account를 허용할 수 있다.
- authority 실수는 admin function을 잘못된 role에 노출할 수 있다.
- session/auth 변경은 login, token expiry, WebSocket session mapping에 영향을 줄 수 있다.
