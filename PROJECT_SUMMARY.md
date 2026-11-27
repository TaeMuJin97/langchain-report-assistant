# AI Technical Report Agent — 프로젝트 요약 보고서

> **기존 PostgreSQL 보고서 DB(`report_agent_lab`)를 읽기 전용(Read-only)으로만 재사용**하고, 독립된 Python 프로젝트에서 **LangChain `create_agent` · CUDA E5/Cross-Encoder RAG · PostgreSQL `pgvector` · FastAPI**를 통합 구현한 기술 보고서 질의·상태 검증 AI Agent 서비스입니다.

---

## 1. 프로젝트 아키텍처 및 이전 프로젝트와의 관계

이전 프로젝트([`report-agent-project`](https://github.com/TaeMuJin97/report-agent-project), Day 25)와 코드·API를 공유하지 않고 **쓰기/인덱싱 서비스와 읽기 전용 Agent 서비스의 책임을 분리(CQRS 형태)**하여 신규 구현했습니다.

| 구분 | Day 25 (`report-agent-project`) | Day 30 (`langchain-report-assistant`) |
| :--- | :--- | :--- |
| **역할** | 보고서 CRUD · 스키마 구축 · Overview 벡터 인덱싱 | **읽기 전용(Read-only)** 기술 보고서 RAG 질의 · 상태 검증 Agent |
| **DB 접근** | 쓰기 / 수정 / 삭제 / 인덱싱 | `SET TRANSACTION READ ONLY` 기반 Parameterized `SELECT` 전용 |
| **핵심 구조** | FastAPI CRUD + 단발성 검색/요약 | LangChain `create_agent` Tool Calling Loop + `ProviderStrategy` 구조화 출력 |
| **검증 방식** | API 기능 테스트 | Fail-Closed 출처/Ledger 교차검증 + 오프라인 `pytest` 41건 + Live/DB 불변성 검증 |

### 전체 실행 흐름

```text
Client (Swagger / Postman)
  └─ POST /api/v1/agent/ask
       └─ FastAPI + Pydantic v2 (입력 검증, request_id, Meter)
            └─ LangChain create_agent (ProviderStrategy[ModelAnswer] + 호출 상한 Middleware)
                 ├─ @tool rag_search
                 │    ├─ CUDA E5 Query Embedding ("query: ", 384d, L2 norm)
                 │    ├─ pgvector Cosine Top-k (ready · model · dim · norm>0 · revision · sha256)
                 │    ├─ report_read_view → LangChain Document + Metadata
                 │    ├─ CUDA Cross-Encoder Reranking (BAAI/bge-reranker-v2-m3)
                 │    └─ LCEL Runnable (RunnableLambda | RAG_PROMPT | LLM) + Citation 검증
                 └─ @tool lookup_status
                      └─ reports LEFT JOIN report_embeddings (index_status, revision, embedding_fresh)
            └─ Server-side Validator (Tool Execution Ledger & Allowed Citation IDs 교차검증)
```

---

## 2. 기술 스택 및 실행 환경

* **Framework / Agent:** Python 3.13, FastAPI, Pydantic v2, LangChain 1.x (`create_agent`, `@tool`, `RunnableLambda`, `ProviderStrategy`), `langchain-openai` (`gpt-4o-mini`, `temperature=0`)
* **Retrieval / Reranking (GPU):** PyTorch CUDA (`cuda:0`, RTX 5080), `intfloat/multilingual-e5-small` (384d), `BAAI/bge-reranker-v2-m3`
* **Database:** PostgreSQL 18 + `pgvector` 0.8.7 (`report_agent_lab`, 기존 10개 테이블 + `report_read_view` 읽기 전용)
* **Test / Verification:** `pytest` (41 tests), 단계별 검증 스크립트(`test/000~016`), Live 통합 검증(`scripts/verify_live.py`), Postman Collection, pgAdmin SQL 원문 대조

---

## 3. 핵심 구현 범위 (Basic B-01 ~ B-15)

* **환경·DB 독립성 (`B-01~B-03`):** CPU 자동 폴백 없는 CUDA 강제 검증(`preflight.py`), 읽기 전용 트랜잭션 및 SQL 파라미터 바인딩(`repository.py`)
* **GPU RAG 파이프라인 (`B-04~B-08`):** E5 `query:` 정규화 임베딩, 유효 벡터 조건(`FRESH`) 기반 `pgvector` 코사인 거리 검색, `Document`/`Metadata` 매핑, CUDA Cross-Encoder 동일 후보 재정렬, LCEL `Runnable` 체인
* **Tool Calling & Agent (`B-09~B-12`):** `rag_search`·`lookup_status` 2개 `@tool` 계약 검증, `create_agent` + `ProviderStrategy(ModelAnswer)` 네이티브 구조화 출력, FastAPI `/api/v1/agent/ask` 연결
* **신뢰성·안전장치 (`B-13~B-15`):** 본문 `[RPT-번호]`와 `source_ids`·허용 Context ID 일치 검증, 실제 실행 Ledger 기반 `tool_trace` 생성, Tool/Model 호출 상한 및 `max_retries=0`·timeout 제어

---

## 4. 검증 결과 요약 (Acceptance T-01 ~ T-10)

| 검증 수단 | 결과 | 주요 확인 내용 |
| :--- | :---: | :--- |
| **단계별 검증 (`test/000~016`)** | ✅ PASS | CUDA Tensor 연산, DB Preflight(`eligible_vectors=180`), E5/Rerank, Tool/Runtime 연결 확인 |
| **오프라인 계약 테스트 (`pytest`)** | ✅ 41 passed (skip 0) | SQL 계약, Tool/Agent 출력 계약, HTTP 200/422/503 오류 규격, 프레임워크 루프 검증 |
| **Live 통합 검증 (`verify_live`)** | ✅ `exit=0` (무료·유료) | `rag_only`·`db_only`·`both`(`completed`), `no_answer`·`missing`(`insufficient_evidence`) 전 시나리오 통과 |
| **HTTP / Postman 검증** | ✅ Basic 9/9 통과 | 무료 4건(`/health` 200, 입력 오류 422 3건) + 유료 5건(200) 통과 + pgAdmin SQL 원문·상태 수동 대조 |
| **DB 불변성 (`test/005`, `016`)** | ✅ `DB unchanged: True` | 프로젝트 시작 전후 10개 테이블 row count 및 SHA-256 해시 100% 일치 |

---

## 5. 트러블슈팅 기록 (Problem → Root Cause → Fix)

### ① 유료 Live 검증 시 `OUTPUT_CONTRACT_ERROR` (`invalid_model_json` · `rag_citation` · `agent_citation`)
* **원인:** `gpt-4o-mini`가 JSON 응답 시 `source_ids`에는 `"RPT-63"`을 넣었으나 `answer` 본문에 `[RPT-63]` 대신 `"보고서 63"`으로 서술하거나, 근거 부족 시 마크다운 코드 블록을 덧붙여 서버 측 `parse_answer()`·`citation_ids()` 검증에서 정상 차단됨.
* **해결:** 서버의 Fail-Closed 검증 규칙은 완화하지 않고 그대로 유지한 채, `app/rag/pipeline.py`(`RAG_PROMPT`)와 `app/agent.py`(`SYSTEM`)에 명확한 JSON/인용 출력 규칙과 예시를 추가하고 `ChatOpenAI(temperature=0)`을 적용하여 전 시나리오(`exit=0`) 해결.

### ② VS Code Pylance `Import "torch" could not be resolved` 표시
* **원인:** 터미널에서는 CUDA 가상환경(`.backendenv`)이 활성화되어 테스트가 통과했으나, 새 프로젝트 창에서 VS Code 에디터 인터프리터가 전역 Python으로 잡혀 있었음.
* **해결:** `Python: Select Interpreter`에서 `.backendenv` 경로를 지정해 해결.

---

## 6. 한계 및 향후 확장 계획 (Intermediate / Advanced)

* 현재 구현은 로컬 개발 전용(`127.0.0.1`) Basic 범위이며 사용자별 인증/인가(AuthN/AuthZ)는 미포함.
* 향후 Query Rewrite / Multi-query, Rerank OFF/ON A/B 평가(`Recall@k`, `MRR`, `nDCG@k`), `POST /api/v1/rag/debug` 및 실행 이력 조회(`GET /api/v1/agent/runs/{request_id}`) 확장 예정.