# Architecture

## 전체 구조

```txt
Browser
  -> Next.js frontend-ui
  -> Next Route Handler (/app/api/*)
  -> core-api-spring
  -> ai-worker-fastapi
  -> PostgreSQL / pgvector
```

각 서비스의 역할은 다음과 같습니다.

| 계층 | 책임 |
| --- | --- |
| `frontend-ui` | 화면 렌더링, 로그인 쿠키 관리, Spring API 프록시 |
| `core-api-spring` | 사용자/노트북/문서/채팅 도메인, 상태 저장, 예외 처리 |
| `ai-worker-fastapi` | PDF 텍스트 추출, 요약, 임베딩 저장, 문서 기반 답변 생성 |
| `PostgreSQL + pgvector` | 관계형 데이터와 벡터 데이터 저장 |
| `Redis` | 캐시 및 세션성 인프라 확장 기반 |

## 문서 업로드 흐름

```txt
1. 사용자가 프론트에서 PDF 업로드
2. Next.js Route Handler가 Spring으로 요청 전달
3. Spring이 Document를 PROCESSING 상태로 저장
4. Spring이 비동기 워커에서 FastAPI로 파일 전달
5. FastAPI가 PDF 파싱, 요약, 임베딩 저장 수행
6. Spring이 문서 상태를 COMPLETED 또는 FAILED로 갱신
7. 프론트는 문서 목록 재조회로 상태 변화를 반영
```

설계 포인트:

- 업로드 요청을 오래 붙잡지 않기 위해 비동기 분석 구조 사용
- 사용자 응답 시간과 AI 처리 시간을 분리
- 문서 상태를 `PROCESSING`, `COMPLETED`, `FAILED`로 표현

## 채팅 흐름

```txt
1. 사용자가 노트북 채팅에 질문 입력
2. Spring이 최근 대화 일부만 조회
3. Spring이 question + history를 FastAPI로 전달
4. FastAPI가 pgvector에서 관련 청크 검색
5. FastAPI가 답변과 reference_chunks 생성
6. Spring이 USER / AI 채팅 이력을 저장
7. Spring이 reference_chunks를 ChatReference로 저장
8. 프론트는 채팅 이력과 references를 함께 렌더링
```

## 대화 컨텍스트 전략

현재는 전체 채팅 이력을 AI Worker에 보내지 않고 최근 대화 윈도우만 전달합니다.

```txt
전체 대화 이력
  -> 최근 N개만 추출
  -> AI Worker로 전달
```

목적:

- 장기 대화에서 payload 증가 억제
- 프롬프트 길이와 토큰 사용량의 무한 증가 완화
- 응답 품질과 비용 사이의 균형 유지

추후 확장:

- 오래된 대화 요약 메모리 저장
- summary + recent window 혼합 전략

## 레퍼런스 저장 구조

AI 답변의 근거는 응답으로만 사용하지 않고 DB에 저장합니다.

```txt
ChatHistory (role=AI, message=답변)
  -> ChatReference(pageNumber, content, sortOrder)
```

이 구조의 의미:

- 새로고침 후에도 과거 AI 답변 근거 복원 가능
- 프론트에서 답변별 `근거 보기` UI 제공 가능
- 이후 PDF 하이라이트 기능으로 확장 가능

현재 정렬은 `sortOrder` 기준입니다.
이는 AI Worker가 반환한 reference 순서를 유지하기 위함입니다.

## 예외 처리 구조

Spring은 전역 예외 처리 구조를 사용합니다.

```txt
Service / Client
  -> CustomException(ErrorCode)
  -> GlobalExceptionHandler
  -> ErrorResponse(status, code, message, errors)
```

현재 처리 범위:

- 도메인 예외
- validation 실패
- 잘못된 JSON 요청
- AI Worker 통신 실패
- AI Worker 빈 응답

예시:

```json
{
  "status": 502,
  "code": "A001",
  "message": "AI 분석 서버와 통신 중 오류가 발생했습니다."
}
```

## 데이터 모델 요약

주요 엔티티:

| 엔티티 | 설명 |
| --- | --- |
| `User` | 사용자 |
| `Notebook` | 문서와 채팅을 묶는 작업 단위 |
| `Document` | 업로드 문서와 요약 상태 |
| `ChatHistory` | USER/AI 대화 내용 |
| `ChatReference` | AI 답변의 근거 문장 |

## 현재 한계

- 레퍼런스는 문장 단위가 아니라 청크 단위라 가독성이 낮을 수 있음
- 하이라이트용 PDF 좌표 정보는 아직 저장하지 않음
- 대화 요약 메모리 전략은 아직 1차 컨텍스트 윈도우 수준
- 스트리밍 응답은 아직 미구현

## 다음 확장 방향

1. 오래된 대화를 summary memory로 압축
2. SSE 기반 스트리밍 응답 추가
3. PDF 텍스트 위치 정보 저장
4. 문장 단위 reference 품질 개선
5. 관측성 및 비용 로깅 추가
