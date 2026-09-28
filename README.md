# 봄길요양

> 요양시설 탐색과 정보 공유를 지원하는 웹 서비스  
> Spring Modulith 기반으로 모듈 경계와 데이터 소유권을 명확하게 개선했습니다.

## 프로젝트 소개

봄길요양은 사용자가 지도에서 요양시설을 검색하고, 시설의 상세 정보를 확인할 수 있도록 지원하는 서비스입니다.

시설 즐겨찾기, 커뮤니티, 관리자와의 실시간 채팅 등 요양시설 선택에 필요한 기능을 한곳에서 제공합니다.

### 주요 기능

- 회원가입, 로그인 및 회원 정보 관리
- 지도 기반 요양시설 검색
- 시설 상세 정보 및 주변 공원 조회
- 관심 시설 즐겨찾기
- 게시글, 댓글, 좋아요 및 파일 첨부
- 관리자와의 1:1 실시간 채팅
- 사용자 및 커뮤니티 관리자 기능
- 장기요양등급 테스트

---

## 아키텍처 개선 배경

기존 백엔드에서는 다음과 같은 문제가 있었습니다.

- 도메인 사이의 직접 참조
- 다른 도메인의 Entity와 Repository 노출
- 인증 모듈과 채팅 모듈의 양방향 의존
- 여러 도메인이 하나의 데이터베이스를 공유
- 모듈 간 의존 규칙을 수동으로 관리

기존 기능은 유지하면서 변경 영향 범위를 줄일 수 있도록 다음과 같이 개선했습니다.

| 기존 구조 | 개선 구조 |
|---|---|
| 도메인 내부 객체 직접 참조 | 공개 API를 통한 단방향 참조 |
| Entity와 Repository 외부 노출 | 공개 인터페이스와 Summary 객체 제공 |
| 모듈 경계가 불분명 | Spring Modulith로 경계 정의 |
| 여러 모듈이 하나의 DB 공유 | 모듈별 논리 데이터베이스 분리 |
| JPA 및 QueryDSL 중심 데이터 접근 | MyBatis 기반 명시적 SQL |
| Refresh Token을 MySQL에 저장 | Redis와 TTL을 이용한 저장 |
| 구조 위반을 수동으로 확인 | 아키텍처 테스트로 자동 검증 |

---

## 전체 시스템 구성

```mermaid
flowchart LR
    Client[React Frontend]
    Backend[Spring Boot Backend]
    MySQL[(MySQL<br/>Module Databases)]
    Redis[(Redis<br/>Refresh Token Store)]

    Client <-->|REST API / WebSocket| Backend
    Backend --> MySQL
    Backend --> Redis
```

- 프론트엔드와 백엔드는 REST API로 통신합니다.
- 실시간 채팅은 WebSocket과 STOMP를 사용합니다.
- 업무 데이터는 MySQL의 모듈별 데이터베이스에 저장합니다.
- Refresh Token은 Redis에서 만료 시간과 함께 관리합니다.

---

## 모듈 구성

| 모듈 | 책임 | 참조 모듈 |
|---|---|---|
| `identity` | 인증, 사용자 정보, JWT 및 Refresh Token 관리 | 없음 |
| `facility` | 요양시설 검색, 상세 정보, 지도 데이터 관리 | 없음 |
| `favorite` | 사용자별 관심 시설 관리 | `identity`, `facility` |
| `community` | 게시글, 댓글, 좋아요 및 첨부파일 관리 | `identity` |
| `chat` | 사용자와 관리자 사이의 실시간 채팅 | `identity` |
| `administration` | 사용자 및 커뮤니티 관리자 기능 | `identity`, `community` |
| `infrastructure` | 공통 웹 설정과 외부 연동 설정 | 없음 |

### 모듈 의존 관계

화살표는 해당 모듈을 참조한다는 의미입니다.

```mermaid
flowchart LR
    Administration[Administration]
    Community[Community]
    Chat[Chat]
    Favorite[Favorite]
    Identity[Identity]
    Facility[Facility]
    Infrastructure[Infrastructure]

    Administration --> Identity
    Administration --> Community
    Community --> Identity
    Chat --> Identity
    Favorite --> Identity
    Favorite --> Facility
```

`identity`, `facility` 모듈은 다른 업무 모듈을 참조하지 않습니다.

모듈 간 기능이 필요한 경우 상대 모듈의 Entity, Mapper 또는 내부 Service를 직접 참조하지 않고 공개 API만 사용합니다.

---

## 모듈 공개 API

### Identity 모듈

| 공개 API | 종류 | 제공 기능 |
|---|---|---|
| `AuthenticatedUser` | Record | 현재 인증된 사용자의 ID, 이메일, 이름, 권한 제공 |
| `UserDirectory` | Interface | 사용자 ID 또는 이메일을 이용한 사용자 정보 조회 |
| `UserSummary` | Record | 다른 모듈에 필요한 사용자 정보만 전달 |
| `UserAdministration` | Interface | 일반 사용자 및 탈퇴 사용자 조회, 사용자 상태 변경 |

### Facility 모듈

| 공개 API | 종류 | 제공 기능 |
|---|---|---|
| `FacilityLookup` | Interface | 시설 ID를 이용한 단건 또는 다건 시설 정보 조회 |
| `FacilitySummary` | Record | 시설명, 주소, 위치, 평점, 정원 등 필요한 시설 정보 제공 |

### Community 모듈

| 공개 API | 종류 | 제공 기능 |
|---|---|---|
| `CommunityStatistics` | Interface | 사용자별 커뮤니티 활동 수 조회 |
| `CommunityActivityCount` | Record | 사용자의 게시글 수와 댓글 수 제공 |

### Summary 객체를 사용하는 이유

`UserSummary`, `FacilitySummary`, `CommunityActivityCount`는 단순히 내용을 요약한 객체가 아닙니다.

다른 모듈에 Entity 전체를 노출하지 않고, 해당 모듈이 실제로 필요한 정보만 전달하기 위한 모듈 간 계약 객체입니다.

이를 통해 다음과 같은 효과를 얻을 수 있습니다.

- 내부 Entity 구조 은닉
- 불필요한 데이터 노출 방지
- 모듈 간 결합도 감소
- 내부 구현 변경의 영향 범위 축소

### 공개 API 사용 예시

`favorite` 모듈은 시설 정보를 조회할 때 `FacilityMapper`나 Facility Entity를 직접 참조하지 않습니다.

```mermaid
flowchart LR
    FavoriteService -->|공개 API 호출| FacilityLookup
    FacilityLookup --> FacilityLookupService
    FacilityLookupService --> FacilityMapper
```

`FavoriteService`는 `FacilityLookup`이라는 계약만 알고 있으며, 실제 조회 방식은 `facility` 모듈 내부에 숨겨집니다.

---

## 데이터 계층

기존의 JPA와 QueryDSL을 MyBatis로 전환했습니다.

MyBatis를 통해 각 모듈의 데이터 접근 범위와 실행 SQL을 코드에서 명확하게 확인할 수 있도록 구성했습니다.

### 모듈별 데이터베이스

각 업무 모듈은 자신이 담당하는 논리 데이터베이스를 사용합니다.

| 모듈 | 데이터베이스 |
|---|---|
| Identity | `identity_db` |
| Community | `community_db` |
| Chat | `chat_db` |
| Facility | `facility_db` |
| Favorite | `favorite_db` |

현재 구성은 하나의 MySQL 서버 안에서 데이터베이스를 논리적으로 분리한 형태입니다.

### 데이터베이스 구성 요소

각 데이터베이스에는 독립적인 데이터 접근 설정이 구성됩니다.

| 구성 요소 | 역할 |
|---|---|
| `DataSource` | 해당 모듈 데이터베이스의 주소와 계정 정보를 이용해 DB 연결 제공 |
| `SqlSessionFactory` | MyBatis 설정과 Mapper 정보를 기반으로 SQL 실행 세션 생성 |
| `SqlSessionTemplate` | Spring 환경에서 MyBatis 세션을 안전하게 사용할 수 있도록 관리 |
| `TransactionManager` | 해당 모듈 데이터베이스의 트랜잭션 시작, 커밋 및 롤백 관리 |
| `Flyway` | 모듈별 스키마 생성과 변경 이력을 버전 단위로 관리 |

이 구조를 통해 한 모듈의 스키마 변경이 다른 모듈의 데이터 계층에 직접 영향을 주지 않도록 했습니다.

---

## Redis 기반 Refresh Token 관리

기존에는 Refresh Token을 MySQL에 저장했지만, 현재는 Redis에 저장하도록 변경했습니다.

### 저장 구조

```text
Key   : auth:refresh:{userId}
Value : Refresh Token
TTL   : 7일
```

Refresh Token은 7일의 TTL과 함께 저장되며, 시간이 지나면 Redis가 자동으로 삭제합니다.

### Redis를 Refresh Token에만 사용한 이유

- Access Token은 JWT 자체 검증이 가능하므로 별도 저장소 조회가 필요하지 않습니다.
- Refresh Token은 재발급 과정에서 서버 측 유효성 검증과 폐기 기능이 필요합니다.
- 로그인, 로그아웃, 토큰 재발급에 필요한 인증 상태만 Redis에서 관리합니다.
- 모든 인증 요청이 Redis에 의존하는 구조를 피할 수 있습니다.
- 인증 성능과 서버 측 통제 가능성을 함께 확보할 수 있습니다.

로그아웃 또는 회원 탈퇴 시에는 Redis에 저장된 Refresh Token을 즉시 삭제합니다.

---

## 기술 스택

### Frontend

| 기술 | 용도 |
|---|---|
| React 19 | 사용자 인터페이스 |
| Vite 8 | 개발 서버 및 빌드 |
| React Router | 페이지 라우팅 |
| Axios | REST API 통신 |
| Tailwind CSS | UI 스타일링 |
| Kakao Maps SDK | 지도 및 시설 위치 표시 |
| SockJS / STOMP | 실시간 채팅 |

### Backend

| 기술 | 용도 |
|---|---|
| Java 21 | 백엔드 개발 언어 |
| Spring Boot 4 | 애플리케이션 프레임워크 |
| Spring Modulith | 모듈 경계와 의존 관계 관리 |
| Spring Security | 인증 및 인가 |
| JWT | Access Token 및 Refresh Token |
| MyBatis | SQL 기반 데이터 접근 |
| WebSocket / STOMP | 실시간 채팅 |
| Flyway | 데이터베이스 마이그레이션 |
| Springdoc OpenAPI | API 문서화 |

### Data & Infrastructure

| 기술 | 용도 |
|---|---|
| MySQL 8.4 | 업무 데이터 저장 |
| Redis 7 | Refresh Token 저장 및 TTL 관리 |
| Docker Compose | 로컬 MySQL 및 Redis 실행 |
| H2 | Mapper 통합 테스트 |

---

## 프로젝트 구조

```text
Bomgilyoyang-Cloud
├─ backend
│  ├─ src/main/java/com/gooroomees/neulbomgil_backend
│  │  ├─ administration
│  │  ├─ chat
│  │  ├─ community
│  │  ├─ facility
│  │  ├─ favorite
│  │  ├─ identity
│  │  └─ infrastructure
│  ├─ src/main/resources
│  │  ├─ db/migration
│  │  └─ application.properties
│  ├─ src/test
│  ├─ docker/mysql/init
│  ├─ compose.yaml
│  └─ build.gradle
│
└─ frontend
   ├─ src
   │  ├─ api
   │  ├─ components
   │  ├─ context
   │  ├─ hooks
   │  ├─ pages
   │  └─ services
   ├─ package.json
   └─ vite.config.js
```

---

## 실행 방법

### 사전 요구사항

- Java 21
- Node.js 20.19 이상
- Docker 및 Docker Compose

### 1. 저장소 복제

```bash
git clone <repository-url>
cd Bomgilyoyang-Cloud
```

### 2. 백엔드 환경 변수 설정

```bash
cd backend
cp .env.example .env
```

Windows PowerShell에서는 다음 명령을 사용할 수 있습니다.

```powershell
Copy-Item .env.example .env
```

`.env` 파일에 다음 정보를 설정합니다.

```properties
IDENTITY_DB_USERNAME=
IDENTITY_DB_PASSWORD=
COMMUNITY_DB_USERNAME=
COMMUNITY_DB_PASSWORD=
CHAT_DB_USERNAME=
CHAT_DB_PASSWORD=
FACILITY_DB_USERNAME=
FACILITY_DB_PASSWORD=
FAVORITE_DB_USERNAME=
FAVORITE_DB_PASSWORD=
MYSQL_ROOT_PASSWORD=

JWT_SECRET=

REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_SSL_ENABLED=false

API_KEY=
KAKAO_KEY=
KAKAO_CLIENT_KEY=
KAKAO_MAP_API_KEY=
```

비밀번호와 API Key는 저장소에 커밋하지 않습니다.

### 3. MySQL과 Redis 실행

`backend` 디렉터리에서 실행합니다.

```bash
docker compose up -d
```

실행되는 기본 포트는 다음과 같습니다.

| 서비스 | 포트 |
|---|---:|
| MySQL | `3306` |
| Redis | `6379` |

### 4. 백엔드 실행

Windows:

```powershell
gradlew.bat bootRun
```

macOS 또는 Linux:

```bash
./gradlew bootRun
```

백엔드 서버는 기본적으로 다음 주소에서 실행됩니다.

```text
http://localhost:8088
```

### 5. 프론트엔드 환경 변수 설정

`frontend/.env` 파일을 생성하고 다음 값을 설정합니다.

```properties
VITE_API_BASE_URL=http://localhost:8088
VITE_KAKAO_MAP_API_KEY=
```

### 6. 프론트엔드 실행

```bash
cd ../frontend
npm install
npm run dev
```

프론트엔드는 다음 주소에서 실행됩니다.

```text
http://localhost:5173
```

---

## API 문서

애플리케이션 실행 후 Swagger UI에서 API를 확인하고 테스트할 수 있습니다.

```text
Swagger UI : http://localhost:8088/swagger-ui/index.html
OpenAPI JSON: http://localhost:8088/v3/api-docs
```

JWT 인증이 필요한 API는 Swagger UI의 `Authorize` 기능을 이용할 수 있습니다.

---

## 테스트 및 아키텍처 검증

### 테스트 실행

Windows:

```powershell
cd backend
gradlew.bat test
```

macOS 또는 Linux:

```bash
cd backend
./gradlew test
```

### 주요 테스트

| 테스트 | 검증 내용 |
|---|---|
| `ModulithArchitectureTest` | 모듈 경계, 허용되지 않은 의존 관계 및 순환 참조 검증 |
| `AuthServiceTest` | Refresh Token TTL 저장, 토큰 재발급 및 불일치 토큰 거부 |
| `UserAuthMapperTest` | 사용자 인증 데이터 CRUD |
| `FavoriteMapperTest` | 즐겨찾기 데이터 CRUD |
| `FavoriteServiceTest` | 즐겨찾기 등록, 중복 방지 및 삭제 |
| `FavoriteControllerTest` | 즐겨찾기 등록, 목록 조회, 삭제 및 요청값 검증 |
| `MapControllerTest` | 시설 목록, 마커, 상세 정보, 주변 공원 및 404 응답 |

Spring Modulith의 `ApplicationModules.verify()`를 이용해 다음 규칙을 자동으로 검사합니다.

- 허용되지 않은 모듈 참조
- 모듈 사이의 순환 의존
- 내부 패키지 직접 접근
- 정의된 모듈 경계 위반

---

## 개선 결과

### 경계

모듈의 역할과 허용 의존 관계를 코드로 명확하게 정의했습니다.

### 데이터

각 모듈이 자신의 데이터베이스와 Mapper를 관리하도록 구성했습니다.

### 인증

Refresh Token의 저장과 만료를 Redis와 TTL로 관리하도록 개선했습니다.

### 검증

테스트 단계에서 모듈 경계 위반과 주요 기능의 회귀를 확인할 수 있도록 구성했습니다.

---
