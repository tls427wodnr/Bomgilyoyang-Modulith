# 봄길요양 프로젝트 설정 및 실행

## 사전 요구사항

- Java 21
- Node.js 20.19 이상
- Docker 및 Docker Compose

## 1. 저장소 복제

```bash
git clone https://github.com/tls427wodnr/Bomgilyoyang-Cloud.git
cd Bomgilyoyang-Cloud
```

## 2. 백엔드 환경 변수 설정

`backend` 디렉터리에서 예제 파일을 복사해 `.env` 파일을 생성합니다.

macOS 또는 Linux:

```bash
cd backend
cp .env.example .env
```

Windows PowerShell:

```powershell
cd backend
Copy-Item .env.example .env
```

생성한 `.env` 파일의 빈 값을 실행 환경에 맞게 설정합니다.

```properties
IDENTITY_DB_USERNAME=identity_app
IDENTITY_DB_PASSWORD=
COMMUNITY_DB_USERNAME=community_app
COMMUNITY_DB_PASSWORD=
CHAT_DB_USERNAME=chat_app
CHAT_DB_PASSWORD=
FACILITY_DB_USERNAME=facility_app
FACILITY_DB_PASSWORD=
FAVORITE_DB_USERNAME=favorite_app
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

비밀번호, JWT Secret, API Key가 포함된 `.env` 파일은 저장소에 커밋하지 않습니다.

## 3. MySQL 및 Redis 실행

`backend` 디렉터리에서 다음 명령을 실행합니다.

```bash
docker compose up -d
```

| 서비스 | 기본 포트 |
|---|---:|
| MySQL | `3306` |
| Redis | `6379` |

컨테이너 상태는 다음 명령으로 확인할 수 있습니다.

```bash
docker compose ps
```

## 4. 백엔드 실행

Windows PowerShell:

```powershell
gradlew.bat bootRun
```

macOS 또는 Linux:

```bash
./gradlew bootRun
```

백엔드는 `http://localhost:8088`에서 실행됩니다.

## 5. 프론트엔드 환경 변수 설정

`frontend/.env` 파일을 생성하고 다음 값을 설정합니다.

```properties
VITE_API_BASE_URL=http://localhost:8088
VITE_KAKAO_MAP_API_KEY=
```

## 6. 프론트엔드 설치 및 실행

새 터미널에서 다음 명령을 실행합니다.

```bash
cd frontend
npm ci
npm run dev
```

프론트엔드는 `http://localhost:5173`에서 실행됩니다.
