# Outline Wiki 설정 가이드

이 저장소는 [Outline Wiki](https://www.getoutline.com/)를 Docker를 사용하여 배포하기 위한 설정 파일들을 포함하고 있습니다.

## 포트 설정

현재 설정된 포트는 다음과 같습니다:

- Outline: 3000 (외부) -> 3000 (내부)
- Redis: 3031 (외부) -> 6379 (내부)
- PostgreSQL: 3032 (외부) -> 5432 (내부)

## 파일 구조

- `docker-compose.yml`: Docker 컨테이너 구성 파일
- `docker.env.example`: 환경 변수 설정 예제 파일 (실제 사용 시 docker.env로 복사하여 사용)
- `redis.conf`: Redis 설정 파일
- `storage-data/`: Outline 데이터 저장 디렉토리
- `database-data/`: PostgreSQL 데이터 저장 디렉토리

## 초기 설정 방법

### 1. 저장소 클론

```bash
git clone https://github.com/your-username/outline-docker.git
cd outline-docker
```

### 2. 필수 디렉토리 생성

```bash
mkdir -p storage-data database-data
touch redis.conf
echo "bind 0.0.0.0" > redis.conf
```

### 3. 환경 설정 파일 준비

#### 파일 복사

```bash
cp docker.env.example docker.env.prod
```

#### 심볼릭 링크 생성

```bash
ln -s docker.env.prod docker.env
```

### 4. 환경 변수 수정

필요에 따라 docker.env 파일을 편집하여 환경 변수를 수정합니다:

```bash
vim docker.env
```

주요 수정 사항:

- `URL`: 실제 서비스 URL로 변경 (예: https://wiki.example.com)
- `COLLABORATION_URL`: 실제 서비스 URL과 동일하게 설정 (WebSocket 연결에 사용됨)
- `SECRET_KEY` 및 `UTILS_SECRET`: 보안을 위해 새로운 값으로 변경
- `OIDC_CLIENT_ID` 및 `OIDC_CLIENT_SECRET`: SSO 인증 정보 설정

### 5. Redis 설정 파일 생성

기본 Redis 설정 파일이 없는 경우 생성합니다:

```bash
echo "bind 0.0.0.0" > redis.conf
```

## Docker 컨테이너 실행

```bash
docker-compose up -d
```

## 서비스 상태 확인

```bash
docker-compose ps
docker-compose logs outline
```

## 접속 방법

- 로컬 접속: http://localhost:3000
- 외부 접속: http://[서버IP]:3000
- 프록시 설정 시: https://wiki.comin.com

## 환경 변수 설정

주요 환경 변수:

- `URL`: Outline 서비스의 기본 URL
- `PORT`: 내부 서비스 포트 (3000으로 설정)
- `COLLABORATION_URL`: 실시간 협업을 위한 WebSocket URL (URL과 동일하게 설정)
- `HOST`: 바인딩할 호스트 (0.0.0.0으로 설정하여 모든 인터페이스에서 접근 가능)
- `FORCE_HTTPS`: HTTPS 강제 여부 (false로 설정)
- `DATABASE_URL`: PostgreSQL 연결 문자열
- `REDIS_URL`: Redis 연결 문자열

## 문제 해결

### WebSocket 연결 오류

WebSocket 연결 오류가 발생하는 경우 (예: `WebSocket connection to 'ws://localhost:3000/collaboration/...' failed`), 다음 설정을 확인하세요:

1. `docker.env` 파일에서 `COLLABORATION_URL`이 실제 서비스 URL과 동일한지 확인:

   ```
   COLLABORATION_URL=https://wiki.example.com
   ```

2. 역방향 프록시를 사용하는 경우, WebSocket 연결을 허용하도록 설정:
   - Nginx 예시:
     ```
     location / {
       proxy_pass http://localhost:3000;
       proxy_http_version 1.1;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
       proxy_set_header Host $host;
     }
     ```

### 접속 문제

1. 방화벽 설정 확인
   ```bash
   sudo ufw status
   ```
2. 포트 사용 여부 확인

   ```bash
   netstat -tulpn | grep 3000
   ```

3. Docker 네트워크 확인
   ```bash
   docker network ls
   docker network inspect outline-network
   ```

### 로그 확인

```bash
docker-compose logs -f outline
```

## 백업 및 복원

### 데이터베이스 백업

```bash
docker-compose exec postgres pg_dump -U user outline > outline_backup.sql
```

### 데이터베이스 복원

```bash
cat outline_backup.sql | docker-compose exec -T postgres psql -U user outline
```

## 설정 변경 시 주의사항

1. 포트 변경 시 `docker-compose.yml`의 포트 매핑만 수정하세요.
2. 내부 서비스 간 통신은 Docker 네트워크를 통해 기본 포트(Redis: 6379, PostgreSQL: 5432)로 이루어집니다.
3. 환경 변수 변경 후에는 반드시 서비스를 재시작하세요.
   ```bash
   docker-compose down
   docker-compose up -d
   ```

## 보안 고려사항

1. 프로덕션 환경에서는 `docker.env` 파일의 보안 키를 반드시 변경하세요.
2. 민감한 정보가 포함된 `docker.env` 파일은 git에 커밋하지 마세요.
3. 외부에서 접근 가능한 서버에 배포할 경우, 방화벽 설정을 통해 필요한 포트만 개방하세요.
