---
title: "MySQL 랜섬웨어 공격 후 PostgreSQL 마이그레이션 회고"
description: "Docker 포트 바인딩 실수로 MySQL이 랜섬웨어에 감염된 경험과, PostgreSQL 마이그레이션 + 인프라 보안 강화까지의 과정 기록"
author: laze
date: 2026-03-12 09:00:00 +0900
categories: [Dev, Database]
tags: [MySQL, PostgreSQL, Docker]
---

## 사고 요약

로컬 개발 환경의 MySQL이 랜섬웨어 공격을 받아 모든 데이터베이스가 삭제되었습니다. `RECOVER_YOUR_DATA`라는 테이블만 남아 비트코인 송금을 요구하는 전형적인 DB 랜섬웨어였습니다.

근본 원인은 세 가지가 겹친 결과였습니다.

1. **Docker 포트가 `0.0.0.0`으로 바인딩** — 호스트 IP 미지정 시 기본값이 `0.0.0.0`이라 외부 네트워크에서 3306 포트에 직접 접근 가능
2. **macOS 방화벽 비활성화** — 외부 접근을 차단할 방어선 부재
3. **취약한 비밀번호 (`root/root`)** — 공격자 입장에서 스캔 → 접속 → 삭제가 가능한 구조

## 전환 동기: 랜섬웨어 + DB 특성 재평가

사고로 스키마 재생성이 필요해진 시점에서, 기존에 미뤄왔던 DB 엔진 선택을 다시 검토했습니다. 기존에는 DB 특성을 깊이 고려하지 않고 MySQL을 기본으로 사용하고 있었는데, 서비스별 워크로드를 분석한 결과 PostgreSQL이 더 적합한 서비스가 대부분이었습니다.

### MySQL vs PostgreSQL 트레이드오프

| 항목 | MySQL | PostgreSQL                            |
|------|-------|---------------------------------------|
| **커넥션 모델** | 스레드 기반 (경량, 대량 단순 커넥션에 유리) | 프로세스 기반 (격리성 높음, 복잡 쿼리 안정적)           |
| **집계/분석** | 기본적인 집계만 지원 | Window Function, CTE, 재귀 쿼리 등 풍부      |
| **JSON 처리** | JSON 타입 지원하나 인덱싱 제한적 | JSONB + GIN 인덱스로 Document DB 수준 쿼리 가능 |
| **동시성 제어** | MVCC이나 gap lock으로 인한 데드락 빈도 높음 | MVCC 구현이 더 성숙, 읽기-쓰기 충돌 적음            |
| **확장성** | 플러그인 기반 | Extension 기반 (PostGIS, pg_trgm 등)     |
| **단순 CRUD 성능** | 스레드 모델 덕분에 가벼운 읽기/쓰기에 오버헤드 적음 | 프로세스 fork 비용이 상대적으로 큼                 |

### 서비스별 판단

| Service | 워크로드 특성 | 결정 |
|---------|-------------|------|
| auth-service | RBAC 권한 조합 쿼리, 감사 로그 집계 | **PostgreSQL** |
| shopping-service | 주문/결제/배송 상태 관리, Saga 상태 추적 | **PostgreSQL** |
| shopping-seller-service | 상품/재고/쿠폰 관리, 타임딜 동시성 제어 | **PostgreSQL** |
| shopping-settlement-service | 정산 집계(SUM/GROUP BY), 기간별 리포트 | **PostgreSQL** |
| notification-service | 단순 알림 읽기/쓰기, 복잡한 쿼리 없음 | **MySQL 유지** |

notification-service는 단순 읽기/쓰기 위주이고 복잡한 집계나 JSONB 처리가 필요 없습니다. MySQL 컨테이너가 이미 존재하고, 단순 CRUD만 수행하는 워크로드를 PostgreSQL로 옮기는 비용 대비 실질적인 이득이 없어 MySQL을 유지했습니다.

나머지 4개 서비스는 집계 쿼리, JSONB, 복잡한 트랜잭션 등 PostgreSQL의 강점이 필요한 워크로드였습니다. 이미 prism-service와 drive-service가 PostgreSQL을 사용 중이었으므로, 새 PostgreSQL 인스턴스를 추가로 띄울 필요 없이 기존 컨테이너에 DB만 추가하면 됐습니다.

## 인프라 보안 강화

### Docker 포트 바인딩

모든 Docker 서비스의 포트를 `127.0.0.1`로 바인딩하여 외부 접근을 원천 차단했습니다.

```yaml
# 변경 전 — 모든 네트워크 인터페이스에 노출
ports:
  - "3306:3306"

# 변경 후 — localhost에서만 접근 가능
ports:
  - "127.0.0.1:5432:5432"   # PostgreSQL
  - "127.0.0.1:3307:3306"   # MySQL (notification 전용)
```

### 비밀번호 강화

`root/root` 같은 기본 비밀번호를 전면 교체했습니다. 로컬 개발 환경이라도 최소한의 보안은 유지해야 합니다.

## PostgreSQL 마이그레이션

### 초기화 스크립트

PostgreSQL 컨테이너 최초 기동 시 서비스별 데이터베이스를 생성하는 스크립트입니다. Docker의 `/docker-entrypoint-initdb.d/` 경로에 놓으면 컨테이너 초기화 시 psql을 통해 자동 실행됩니다.

```sql
-- infrastructure/postgresql/init-services.sql
-- psql 메타 명령어 \gexec: 앞선 SELECT 결과를 SQL 문으로 실행

SELECT 'CREATE DATABASE auth_db'
WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = 'auth_db')\gexec

SELECT 'CREATE DATABASE shopping_db'
WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = 'shopping_db')\gexec
```

`\gexec`는 psql의 메타 명령어로, SELECT 결과로 나온 문자열을 SQL 문으로 실행합니다. PostgreSQL에는 `CREATE DATABASE IF NOT EXISTS` 구문이 없기 때문에, `pg_database` 카탈로그에서 존재 여부를 확인한 뒤 조건부로 생성하는 방식입니다.

### MySQL → PostgreSQL 전환 규칙

일괄 변환을 위해 매핑 규칙을 정했습니다.

| MySQL | PostgreSQL | 비고 |
|-------|-----------|------|
| `AUTO_INCREMENT` | `GENERATED ALWAYS AS IDENTITY` | 시퀀스 기반 ID 생성 |
| `enum('A','B')` | `VARCHAR(50)` | PostgreSQL enum은 ALTER 제약이 커서 회피 |
| `ON UPDATE CURRENT_TIMESTAMP` | Trigger 함수 | 아래에서 별도 설명 |
| `datetime` | `TIMESTAMP(6)` → `TIMESTAMPTZ(6)` | 초기 전환 시 `TIMESTAMP(6)`, 이후 타임존 대응을 위해 `TIMESTAMPTZ(6)`로 재변환 |
| `tinyint(1)` | `BOOLEAN` | 네이티브 boolean 타입 |

### updated_at 자동 갱신

MySQL에서는 컬럼 정의에 `ON UPDATE CURRENT_TIMESTAMP`를 붙이면 끝이지만, PostgreSQL은 Trigger가 필요합니다.

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
    RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 각 테이블에 Trigger 연결
CREATE TRIGGER set_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

함수를 한 번 정의하고 각 테이블에 Trigger만 걸면 되므로, 테이블 수가 많아도 관리 부담은 크지 않습니다.

### Flyway 마이그레이션 통합

DB 엔진이 변경되면 `flyway_schema_history` 테이블이 존재하지 않으므로, V1부터 새로 시작할 수 있습니다. 이 점을 이용해 기존에 V1~V5로 나뉘어 있던 파일들을 최종 스키마 상태의 `V1__init.sql` 하나로 통합했습니다.

```
-- 변경 전 (MySQL, auth-service 기준)
V1__baseline.sql
V2__membership_restructuring.sql
V3__audit_log_nullable_target.sql
V4__role_multi_include.sql
V5__role_default_membership_mapping.sql

-- 변경 후 (PostgreSQL)
V1__init.sql   ← 최종 스키마 + Seed Data
```

새 환경에서 Flyway가 V1 하나만 실행하면 되므로, 중간 상태를 거치지 않아 초기 구동이 빠르고 마이그레이션 파일 관리도 단순해집니다.

### 변환 결과 예시

auth-service의 users 테이블입니다.

```sql
CREATE TABLE IF NOT EXISTS users (
    user_id  BIGINT        NOT NULL GENERATED ALWAYS AS IDENTITY,
    uuid     VARCHAR(255)  NOT NULL,
    email    VARCHAR(255)  NOT NULL,
    password VARCHAR(255)  DEFAULT NULL,
    status   VARCHAR(50)   NOT NULL,
    last_login_at        TIMESTAMP(6) DEFAULT NULL,
    password_changed_at  TIMESTAMP(6) DEFAULT NULL,
    created_at           TIMESTAMP(6) DEFAULT NULL,
    updated_at           TIMESTAMP(6) DEFAULT NULL,
    PRIMARY KEY (user_id),
    CONSTRAINT uk_users_email UNIQUE (email),
    CONSTRAINT uk_users_uuid  UNIQUE (uuid)
);
```

V1__init.sql의 초기 스키마이며, 이후 V2에서 모든 `TIMESTAMP(6)` 컬럼을 `TIMESTAMPTZ(6)`로 변환했습니다.

## 전환 결과

| DB 엔진 | 서비스 | 선택 근거 |
|---------|--------|----------|
| PostgreSQL | auth, shopping, seller, settlement, prism, drive | 집계/JSONB/복잡 트랜잭션 |
| MySQL | notification | 단순 CRUD, PostgreSQL 전환 이득 없음 |
| MongoDB | blog | 문서 구조 데이터 |

MySQL 컨테이너는 notification-service 전용으로 축소되어 메모리 할당을 1GB에서 512MB로 줄였습니다.

## 교훈

**1. Docker 포트 바인딩은 반드시 `127.0.0.1`을 명시할 것**

호스트 IP를 생략하면 `0.0.0.0`이 기본값이 되어 외부 네트워크에 노출됩니다. 로컬 개발 환경이라도 동일합니다.

**2. DB 엔진 선택은 워크로드 특성에 맞춰야 한다**

"익숙하니까 MySQL"이 아니라, 커넥션 모델·쿼리 복잡도·동시성 요구사항을 기준으로 판단해야 합니다. 사고가 아니었다면 이 재평가를 미뤘을 가능성이 높습니다.

**3. 스키마 재생성이 필요한 상황은 마이그레이션 통합의 기회다**

Flyway 히스토리가 없는 깨끗한 상태에서 시작할 수 있으므로, 파편화된 마이그레이션 파일을 정리할 수 있습니다.
