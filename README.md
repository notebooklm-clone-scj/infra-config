# infra-config

NotebookLM Clone 전체 시스템의 인프라 실행, 운영 배포, 아키텍처 문서를 관리하는 저장소입니다.

이 저장소는 단순히 Docker 설정만 모아두는 곳이 아니라, 멀티레포 프로젝트를 어떤 방식으로 연결하고 실행하는지 설명하는 허브 역할을 담당합니다.

## Repository Role

이 프로젝트는 아래 4개 저장소로 분리되어 있습니다.

| Repository | Role |
| --- | --- |
| `frontend-ui` | Next.js 기반 사용자 인터페이스 |
| `core-api-spring` | 인증, 노트북, 문서, 채팅 API 및 비즈니스 로직 |
| `ai-worker-fastapi` | PDF 파싱, 요약, 임베딩, 벡터 검색, LLM 연동 |
| `infra-config` | Docker Compose, Nginx, 배포 문서, 아키텍처 문서 |

## Architecture

```mermaid
flowchart LR
    U[User Browser]
    N[Nginx]
    F[frontend-ui<br/>Next.js]
    S[core-api-spring<br/>Spring Boot]
    A[ai-worker-fastapi<br/>FastAPI]
    P[(PostgreSQL + pgvector)]
    R[(Redis)]
    G[Gemini API]

    U --> N
    N --> F
    N --> S
    F --> S
    S --> P
    S --> R
    S --> A
    A --> P
    A --> G
```

## Document Analysis Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend as frontend-ui
    participant Spring as core-api-spring
    participant Worker as ai-worker-fastapi
    participant DB as PostgreSQL/pgvector

    User->>Frontend: PDF 업로드
    Frontend->>Spring: 업로드 요청 전달
    Spring->>DB: Document(status=PROCESSING) 저장
    Spring->>Worker: PDF 분석 요청
    Worker->>Worker: 텍스트 추출, 요약 생성
    Worker->>DB: 임베딩 및 메타데이터 저장
    Worker-->>Spring: 분석 결과 반환
    Spring->>DB: Document(status=COMPLETED/FAILED) 갱신
    Frontend->>Spring: 문서 목록 재조회
```

## Chat Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend as frontend-ui
    participant Spring as core-api-spring
    participant Worker as ai-worker-fastapi
    participant DB as PostgreSQL/pgvector
    participant Gemini as Gemini API

    User->>Frontend: 질문 입력
    Frontend->>Spring: 채팅 요청
    Spring->>DB: 최근 대화/문서 조회
    Spring->>Worker: question + history 전달
    Worker->>DB: 유사 문서 청크 검색
    Worker->>Gemini: 컨텍스트 기반 답변 요청
    Gemini-->>Worker: 답변 반환
    Worker-->>Spring: answer + references 반환
    Spring->>DB: ChatHistory + ChatReference 저장
    Spring-->>Frontend: 응답 반환
```

## Services

| Service | Role | Default Port |
| --- | --- | --- |
| `nginx` | 외부 진입점, reverse proxy | `80` |
| `frontend-ui` | 사용자 인터페이스 | `3000` |
| `spring-api` | 메인 비즈니스 API | `8080` |
| `ai-worker` | PDF 처리, 임베딩, LLM 연동 | `8000` |
| `postgres` | 관계형 데이터 및 벡터 저장소 | `5432` |
| `redis` | refresh token 및 캐시 저장 | `6379` |

## Files

이 저장소에서 주로 사용하는 파일은 다음과 같습니다.

- `docker-compose.yml`: 로컬 인프라만 띄우는 간단한 Compose 파일
- `docker-compose.prod.yml`: 전체 멀티서비스 운영 배포용 Compose 파일
- `.env.example`: 로컬 PostgreSQL 실행용 예시 환경 변수
- `.env.prod.example`: 운영 배포용 예시 환경 변수
- `nginx/nginx.conf`: reverse proxy 설정
- `docs/architecture.md`: 전체 아키텍처 상세 문서

## Local Infra

PostgreSQL과 Redis만 먼저 띄워서 로컬 개발 환경의 기반으로 사용할 수 있습니다.

```bash
cd infra-config
cp .env.example .env
docker compose up -d
```

구성:

- PostgreSQL + pgvector
- Redis

## Production Deployment

운영 배포는 이 저장소의 `docker-compose.prod.yml` 기준으로 수행합니다.

이 섹션의 명령은 모두 아래 위치에서 실행한다고 가정합니다.

```bash
cd ~/notebooklm/infra-config
```

또한 Compose 프로젝트 이름은 기존 배포와 볼륨 이름을 유지하기 위해 `notebooklm`으로 고정합니다.

### 1. 디렉터리 구조

운영 서버에는 아래처럼 4개 저장소가 같은 상위 디렉터리 아래 있어야 합니다.

```txt
~/notebooklm
├── frontend-ui
├── core-api-spring
├── ai-worker-fastapi
└── infra-config
```

`docker-compose.prod.yml`은 `infra-config` 안에 있지만, 각 서비스 이미지는 형제 디렉터리의 소스코드를 사용하도록 되어 있습니다.

### 2. 운영 환경 변수 파일 생성

```bash
cp .env.prod.example .env
```

실제 운영 값은 `.env`에 입력하고, 이 파일은 Git에 커밋하지 않습니다.

### 3. 주요 환경 변수

```env
POSTGRES_USER=chanjin
POSTGRES_PASSWORD=replace-with-a-strong-db-password
POSTGRES_DB=notebooklm_db

SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/notebooklm_db
SPRING_DATASOURCE_USERNAME=chanjin
SPRING_DATASOURCE_PASSWORD=replace-with-a-strong-db-password
SPRING_DATA_REDIS_HOST=redis
SPRING_DATA_REDIS_PORT=6379
JWT_SECRET=replace-with-a-32-byte-or-longer-secret
AI_WORKER_URL=http://ai-worker:8000
APP_CORS_ALLOWED_ORIGINS=http://your-domain.example

CORE_API_BASE_URL=http://spring-api:8080
SESSION_COOKIE_SECURE=true

GEMINI_API_KEY=replace-with-your-gemini-api-key
DATABASE_URL=postgresql://chanjin:replace-with-a-strong-db-password@postgres:5432/notebooklm_db
```

### 4. 전체 서비스 실행

이미지까지 다시 빌드하면서 전체 서비스를 실행합니다.

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml up -d --build
```

### 5. 상태 확인

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml ps
```

### 6. 로그 확인

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml logs --tail=200 spring-api
docker-compose -p notebooklm -f docker-compose.prod.yml logs --tail=200 ai-worker
docker-compose -p notebooklm -f docker-compose.prod.yml logs --tail=200 frontend-ui
```

### 7. 종료

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml down
```

## Operations

운영 중 자주 쓰는 정지/재기동 명령은 아래 기준으로 사용합니다.

이 섹션의 명령도 모두 아래 위치에서 실행합니다.

```bash
cd ~/notebooklm/infra-config
```

### 1. 컨테이너를 완전히 내리기

컨테이너와 네트워크를 정리하고 나중에 다시 올리고 싶을 때 사용합니다.

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml down
```

다시 시작:

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml up -d
```

코드 변경까지 반영하면서 다시 시작:

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml up -d --build
```

### 2. 컨테이너를 잠깐 멈추기

컨테이너를 삭제하지 않고 잠깐 멈췄다가 빠르게 다시 켜고 싶을 때 사용합니다.

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml stop
```

다시 시작:

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml start
```

### 3. 서버(VM) 자체를 종료할 때

권장 순서는 아래와 같습니다.

1. 먼저 애플리케이션 컨테이너를 내립니다.
2. 그 다음 VM을 종료합니다.
3. VM이 다시 켜진 뒤 Compose로 서비스를 다시 올립니다.

예시:

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml down
shutdown -h now
```

서버 재기동 후:

```bash
cd ~/notebooklm/infra-config
docker-compose -p notebooklm -f docker-compose.prod.yml up -d
```

### 4. 주의사항

- `down -v`는 PostgreSQL 볼륨까지 삭제할 수 있으므로 운영 환경에서는 사용하지 않는 것을 권장합니다.
- 인증서는 `certbot/conf`에 저장되므로 일반적인 `down`/`up`만으로는 사라지지 않습니다.
- `infra-config` 디렉터리에서 실행하더라도 `-p notebooklm`를 붙여야 기존 프로젝트 이름과 볼륨 이름을 유지할 수 있습니다.

## HTTPS with Let's Encrypt

도메인 DNS가 서버 IP를 가리키고 있다면, Certbot으로 무료 HTTPS 인증서를 발급받을 수 있습니다.

현재 저장소에는 두 단계 구성이 들어 있습니다.

- `nginx/nginx.conf`: 최초 인증서 발급용 HTTP 설정
- `nginx/nginx.https.conf`: 인증서 발급 후 사용할 HTTPS 설정

이 섹션의 명령도 모두 아래 위치에서 실행합니다.

```bash
cd ~/notebooklm/infra-config
```

### 1. 443 포트 열기

클라우드 방화벽에서 `443/TCP`를 허용해야 합니다.

### 2. 현재 HTTP 설정으로 서비스 실행

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml up -d
```

### 3. 인증서 발급

루트 도메인만 쓸 경우:

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml run --rm --entrypoint certbot certbot certonly --webroot --webroot-path=/var/www/certbot -d dev-scj.site --email your-email@example.com --agree-tos --no-eff-email
```

`www`도 함께 쓸 경우:

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml run --rm --entrypoint certbot certbot certonly --webroot --webroot-path=/var/www/certbot -d dev-scj.site -d www.dev-scj.site --email your-email@example.com --agree-tos --no-eff-email
```

주의:

- `www.dev-scj.site`를 같이 발급받으려면 해당 DNS 레코드도 서버를 가리켜야 합니다.
- DNS가 아직 전파되지 않았으면 인증서 발급이 실패할 수 있습니다.
- `certbot` 서비스는 기본 entrypoint가 `renew` 루프이므로, 신규 발급 시에는 반드시 `--entrypoint certbot`를 붙여야 합니다.

### 4. HTTPS 설정으로 전환

인증서 발급이 끝나면 아래처럼 HTTPS 설정 파일로 교체합니다.

```bash
cp nginx/nginx.https.conf nginx/nginx.conf
docker-compose -p notebooklm -f docker-compose.prod.yml restart nginx
```

### 5. 확인

브라우저에서 아래 주소로 접속합니다.

- `https://dev-scj.site`
- `https://www.dev-scj.site`

인증서가 정상이라면 자물쇠가 표시됩니다.

## Access

배포 완료 후 일반적으로 외부 사용자는 Nginx를 통해 접속합니다.

- 사용자 화면: `http://<server-domain-or-ip>`
- Spring API 내부 주소: `http://spring-api:8080`
- AI Worker 내부 주소: `http://ai-worker:8000`

보통 외부에는 `80`, `443`, `22`만 열고, 내부 서비스 포트는 직접 노출하지 않습니다.

## Recommended Network Policy

외부 공개 권장 포트:

- `80` HTTP
- `443` HTTPS
- `22` SSH

비공개 유지 권장 포트:

- `5432` PostgreSQL
- `6379` Redis
- `8000` AI Worker
- `8080` Spring API

## Troubleshooting

### 1. frontend build 실패

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml logs --tail=200 frontend-ui
```

Next.js production build 타입 오류나 서버 런타임 오류를 먼저 확인합니다.

### 2. AI 분석 실패

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml logs --tail=200 ai-worker
```

주요 확인 포인트:

- `GEMINI_API_KEY`가 유효한지
- `DATABASE_URL` 값이 올바른지
- PostgreSQL 연결이 정상인지
- 문서 분석 요청이 `POST /api/v1/pdf/extract`에서 실패하는지

### 3. Spring API 통신 오류

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml logs --tail=200 spring-api
```

주요 확인 포인트:

- `AI_WORKER_URL` 값이 올바른지
- JWT/Redis 설정이 정상인지
- DB 연결 및 JPA 테이블 생성이 정상인지

### 4. docker-compose v1 재생성 오류

일부 환경에서는 컨테이너 재생성 중 `ContainerConfig` 오류가 발생할 수 있습니다.

이 경우 전체 종료 후 다시 실행하면 해결되는 경우가 많습니다.

```bash
docker-compose -p notebooklm -f docker-compose.prod.yml down
docker-compose -p notebooklm -f docker-compose.prod.yml up -d --build
```

## Security Notes

- `docker-compose.prod.yml`은 Git에 커밋해도 됩니다.
- `.env`, `.env.prod` 같은 실제 비밀값 파일은 Git에 커밋하면 안 됩니다.
- 운영 환경에서는 `SESSION_COOKIE_SECURE=true`를 권장합니다.
- 운영 환경에서는 강한 `JWT_SECRET`과 DB 비밀번호를 사용해야 합니다.

## Related Documents

- [docs/architecture.md](./docs/architecture.md)
- [frontend-ui](https://github.com/notebooklm-clone-scj/frontend-ui)
- [core-api-spring](https://github.com/notebooklm-clone-scj/core-api-spring)
- [ai-worker-fastapi](https://github.com/notebooklm-clone-scj/ai-worker-fastapi)
