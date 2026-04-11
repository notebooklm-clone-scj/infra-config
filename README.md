# notebooklm-clone-scj

문서를 업로드하고, 요약을 생성하고, 문서 기반 질의를 수행하는 NotebookLM 스타일 멀티서비스 프로젝트입니다.

이 저장소는 단순 인프라 설정만 모아둔 폴더가 아니라, 전체 프로젝트를 실행하고 설명하는 허브 역할도 함께 담당합니다.

```txt
사용자 요청
  -> frontend-ui
  -> core-api-spring
  -> ai-worker-fastapi
  -> PostgreSQL / pgvector
  -> 응답 저장 및 재조회
```

## 프로젝트 목표

- 문서를 업로드하면 비동기로 분석을 수행한다.
- 문서 요약과 문서 기반 채팅을 하나의 노트북 단위로 묶는다.
- AI 답변의 근거를 저장하고, 과거 채팅에서도 다시 확인할 수 있게 한다.
- Spring, Python, AI, Docker가 연결된 구조를 실제 서비스처럼 구성한다.

## 서비스 구성

| 저장소 | 역할 | 주요 기술 |
| --- | --- | --- |
| `core-api-spring` | 사용자, 노트북, 문서, 채팅 API 및 비즈니스 로직 | Spring Boot, JPA, JWT |
| `ai-worker-fastapi` | PDF 파싱, 임베딩, 유사도 검색, LLM 응답 생성 | FastAPI, LangChain, Gemini, PGVector |
| `frontend-ui` | 사용자 화면, 인증 처리, Spring API 프록시 | Next.js, TypeScript, Docker |
| `infra-config` | 공용 인프라 실행과 전체 아키텍처 문서 | Docker Compose, PostgreSQL, Redis |

## 핵심 기능

### 1. 노트북 단위 문서 관리

- 사용자는 노트북을 생성할 수 있습니다.
- 노트북마다 여러 PDF 문서를 업로드할 수 있습니다.
- 문서 목록과 요약은 노트북 단위로 관리됩니다.

### 2. 비동기 문서 분석

문서 업로드는 즉시 요약을 반환하지 않습니다.

```txt
1. 사용자가 PDF 업로드
2. Spring이 문서 레코드를 먼저 생성
3. Spring이 FastAPI 분석 작업을 비동기로 전달
4. FastAPI가 텍스트 추출, 요약, 임베딩 저장 수행
5. Spring이 문서 상태와 요약을 갱신
6. 프론트가 문서 목록을 다시 조회해 상태를 반영
```

### 3. 문서 기반 채팅

- 채팅은 특정 문서 하나가 아니라 노트북 전체 문서 집합을 기준으로 수행됩니다.
- Spring은 최근 대화 일부만 AI Worker에 전달해 컨텍스트 길이를 제한합니다.
- FastAPI는 pgvector에서 유사한 청크를 찾아 답변과 근거를 생성합니다.

### 4. AI 답변 근거 저장

- FastAPI는 `reference_chunks`를 함께 반환합니다.
- Spring은 AI 답변과 근거를 별도 엔티티로 저장합니다.
- 이후 채팅 이력 조회 시 과거 AI 답변의 근거도 다시 확인할 수 있습니다.

### 5. API 에러 응답 표준화

- Spring은 공통 예외 응답 구조를 사용합니다.
- 잘못된 입력, 잘못된 요청 본문, 외부 AI 서버 장애를 일관된 JSON 형식으로 내려줍니다.

예시:

```json
{
  "status": 400,
  "code": "C001",
  "message": "잘못된 입력값입니다.",
  "errors": [
    {
      "field": "email",
      "message": "이메일 형식이 올바르지 않습니다."
    }
  ]
}
```

## 실행 순서

### 1. 인프라 실행

```bash
cd /Users/seochanjin/workspace/notebooklm/infra-config
docker compose up -d
```

기본 인프라:

- PostgreSQL + pgvector
- Redis

### 2. AI Worker 실행

```bash
cd /Users/seochanjin/workspace/notebooklm/ai-worker-fastapi
docker compose up --build
```

### 3. 프론트 실행

```bash
cd /Users/seochanjin/workspace/notebooklm/frontend-ui
docker compose up --build
```

### 4. Spring API 실행

Spring은 현재 로컬 실행 기준으로 연결되는 구성이며, 필요 시 Dockerfile로 이미지 빌드가 가능합니다.

```bash
cd /Users/seochanjin/workspace/notebooklm/core-api-spring
./gradlew bootRun
```

## 문서

- 전체 아키텍처: [docs/architecture.md](/Users/seochanjin/workspace/notebooklm/infra-config/docs/architecture.md)
- Spring API 설명: [core-api-spring README](/Users/seochanjin/workspace/notebooklm/core-api-spring/README.md)
- FastAPI AI Worker 설명: [ai-worker-fastapi README](/Users/seochanjin/workspace/notebooklm/ai-worker-fastapi/README.md)
- 프론트 설명: [frontend-ui README](/Users/seochanjin/workspace/notebooklm/frontend-ui/README.md)

## 현재 한계와 다음 단계

현재 레퍼런스는 청크 단위라 가독성이 높지 않습니다.
추후에는 아래 방향으로 확장할 수 있습니다.

- 오래된 대화의 요약 메모리 전략 구현
- SSE 기반 스트리밍 응답
- PDF 위치 정보 기반 하이라이트 표시
- 레퍼런스 품질 개선
- 테스트 범위 확대 및 관측성 보강
