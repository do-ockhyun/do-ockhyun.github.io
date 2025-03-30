# RAG 시스템 설계 (2024-03-26)

## 1. 시스템 아키텍처

```
[Frontend (채널팀 - JS)]
    │
    ├── Next.js/React 애플리케이션
    │   ├── 사용자 인터페이스
    │   └── API 클라이언트
    │
    └── API Gateway
        │
[Backend (AI팀 - Python)]
    │
    ├── RAG 서비스 (FastAPI)
    │   ├── LangChain 파이프라인
    │   ├── 벡터 DB
    │   └── LLM 통합
    │
    └── API 서버 (FastAPI)
        ├── 인증/인가
        └── 비즈니스 로직
```

## 2. API 통신 구조

### Frontend (채널팀)
```typescript
// api/rag.ts
interface RAGResponse {
  answer: string;
  sources: Array<{
    content: string;
    metadata: Record<string, any>;
  }>;
}

export const ragApi = {
  async query(question: string): Promise<RAGResponse> {
    const response = await fetch('/api/rag/query', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ question }),
    });
    return response.json();
  }
};

// components/Chat.tsx
const Chat = () => {
  const handleSubmit = async (question: string) => {
    try {
      const response = await ragApi.query(question);
      // 응답 처리
    } catch (error) {
      // 에러 처리
    }
  };
  // ...
};
```

### Backend (AI팀)
```python
# rag_service.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Dict, Any

app = FastAPI()

class QueryRequest(BaseModel):
    question: str

class Source(BaseModel):
    content: str
    metadata: Dict[str, Any]

class RAGResponse(BaseModel):
    answer: str
    sources: List[Source]

class RAGService:
    def __init__(self):
        self.vector_store = self._init_vector_store()
        self.llm = self._init_llm()
    
    async def process_query(self, question: str) -> RAGResponse:
        # RAG 로직 구현
        docs = await self.vector_store.similarity_search(question)
        answer = await self.llm.generate(docs, question)
        return RAGResponse(
            answer=answer,
            sources=[Source(content=doc.content, metadata=doc.metadata) for doc in docs]
        )

rag_service = RAGService()

@app.post("/api/rag/query", response_model=RAGResponse)
async def query_endpoint(request: QueryRequest):
    try:
        response = await rag_service.process_query(request.question)
        return response
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## 3. 서비스 간 통신

### API Gateway 설정
```yaml
# nginx.conf
server {
    listen 80;
    server_name api.yourdomain.com;

    # Frontend 요청을 Backend로 라우팅
    location /api/rag/ {
        proxy_pass http://rag-service:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Docker Compose 설정
```yaml
# docker-compose.yml
version: '3.8'

services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - API_URL=http://api.yourdomain.com

  rag-service:
    build: ./rag-service
    ports:
      - "8000:8000"
    environment:
      - VECTOR_STORE_URL=your-vector-store-url
      - LLM_API_KEY=your-api-key
```

## 4. 개발 프로세스

### API 계약 정의
```typescript
// shared/types.ts (공유 타입 정의)
export interface RAGQuery {
  question: string;
}

export interface RAGResponse {
  answer: string;
  sources: Array<{
    content: string;
    metadata: Record<string, any>;
  }>;
}
```

### OpenAPI 문서화
```python
# rag_service.py
from fastapi.openapi.utils import get_openapi

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    
    openapi_schema = get_openapi(
        title="RAG Service API",
        version="1.0.0",
        description="RAG Service API documentation",
        routes=app.routes,
    )
    
    app.openapi_schema = openapi_schema
    return app.openapi_schema

app.openapi = custom_openapi
```

## 5. 모니터링 및 로깅

```python
# rag_service.py
import logging
from fastapi import Request
import time

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    duration = time.time() - start_time
    
    logging.info(
        f"Method: {request.method} Path: {request.url.path} "
        f"Duration: {duration:.2f}s Status: {response.status_code}"
    )
    return response
```

## 주요 고려사항

1. 각 팀이 선호하는 기술 스택 사용 가능
2. 서비스 간 독립적인 개발과 배포 가능
3. API를 통한 명확한 인터페이스로 팀 간 협업 용이
4. 각 서비스의 독립적인 스케일링 가능

## 추가 고려사항

- 서비스 간 인증/인가 처리
- 에러 처리 및 재시도 로직
- 캐싱 전략
- 로드 밸런싱
- 서비스 디스커버리 (필요한 경우) 