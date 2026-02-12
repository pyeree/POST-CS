# 06. Agentic RAG

> **핵심 질문:** 에이전트가 검색을 "똑똑하게" 하려면?

---

## 1. 기본 RAG의 한계

03에서 배운 기본 RAG는 이런 흐름이었다:

```
쿼리 → 검색 → 결과를 LLM에 전달 → 답변 생성
```

**단 한 번의 검색, 단 한 번의 생성.** 이게 왜 문제일까?

### 한계 1: 검색 결과가 부족해도 모름

```
쿼리: "Python에서 GIL 없이 진짜 병렬 처리하는 최신 방법"
검색 결과: [2년 전 글만 나옴]
LLM: (오래된 정보 기반으로) "subinterpreters를 사용하면..."
→ 최신 PEP 703 (GIL 제거 프로젝트) 정보가 누락됨
```

기본 RAG는 검색 결과가 부족해도 **그대로 답변**한다. "더 찾아봐야겠다"는 판단이 없다.

### 한계 2: 쿼리가 모호하면 검색이 안 됨

```
쿼리: "저번에 쓴 그 글 있잖아, 락 관련된 거"
→ 벡터 검색이 "그 글"을 이해 못함
→ 쿼리를 재작성해야 함: "Python GIL lock 블로그 글"
```

### 한계 3: 여러 소스를 조합해야 하는 경우

```
쿼리: "GIL과 Docker 네트워크를 비교하는 글을 써줘"
→ GIL 관련 문서도 찾고, Docker 관련 문서도 찾아야 함
→ 기본 RAG는 하나의 쿼리로 한 번만 검색
```

---

## 2. Agentic RAG = 에이전트 + RAG

**에이전트가 검색 과정을 자율적으로 제어한다.**

```
[기본 RAG]
쿼리 ──▶ 검색 ──▶ 생성
(일직선, 한 번)

[Agentic RAG]
쿼리 ──▶ 에이전트 ──┬──▶ 검색 1 ──▶ "충분한가?" ──┬── Yes ──▶ 생성
                    │                              │
                    │                              └── No ──▶ 쿼리 재작성 ──▶ 검색 2 ──▶ ...
                    │
                    ├──▶ 웹 검색 (최신 정보 필요 시)
                    │
                    └──▶ DB 쿼리 (정확한 수치 필요 시)
```

---

## 3. Agentic RAG의 핵심 패턴

### 3.1 쿼리 재작성 (Query Rewriting)

사용자의 원래 쿼리를 검색에 더 적합한 형태로 바꾼다.

```
사용자: "저번에 쓴 락 관련 글이랑 겹치지 않게 동시성 글 써줘"

에이전트 (Thought):
  "저번에 쓴 락 관련 글"이 뭔지 먼저 찾아야 한다.
  쿼리를 재작성하자.

재작성된 쿼리들:
  1. "Python lock 블로그 글"
  2. "GIL mutex 동시성"
  3. "threading lock 사용법"

→ 3개 쿼리로 각각 검색 → 결과를 합침
```

```python
def rewrite_query(original_query: str, context: str) -> list[str]:
    """LLM을 사용해서 검색에 적합한 쿼리들을 생성"""
    prompt = f"""다음 사용자 쿼리를 벡터 검색에 적합한 구체적인 쿼리 3개로 재작성하세요.

원래 쿼리: {original_query}
컨텍스트: {context}

각 쿼리는 서로 다른 관점에서 검색되도록 하세요.
"""
    response = llm.generate(prompt)
    return parse_queries(response)  # ["Python lock 블로그", "GIL mutex", ...]
```

### 3.2 자기 평가 (Self-Evaluation)

검색 결과가 충분한지 에이전트가 스스로 판단한다.

```
[검색 결과를 받은 후]

에이전트 (Thought):
  "검색 결과 3개를 받았다. 질문에 답하기에 충분한가?"

  결과 1: Python GIL 기본 설명 → 관련 있음 ✓
  결과 2: Docker 볼륨 마운트   → 관련 없음 ✗
  결과 3: 멀티프로세싱 기초    → 부분적으로 관련 ✓

  "GIL 우회 방법에 대한 구체적 정보가 부족하다."
  "쿼리를 바꿔서 다시 검색하자."

새 쿼리: "GIL 우회 방법 multiprocessing asyncio subinterpreters"
→ 재검색
```

```python
def evaluate_results(query: str, results: list[str]) -> dict:
    """검색 결과의 충분성을 평가"""
    prompt = f"""검색 결과가 다음 질문에 답하기에 충분한지 평가하세요.

질문: {query}

검색 결과:
{chr(10).join(results)}

다음 JSON 형식으로 답하세요:
{{"sufficient": true/false, "missing": "부족한 정보", "new_query": "재검색 쿼리 (부족할 때만)"}}
"""
    return llm.generate(prompt)
```

### 3.3 라우팅 (Routing)

쿼리의 성격에 따라 다른 검색 전략을 선택한다.

```
사용자: "Python GIL의 최신 변경사항"
         │
         ▼
    에이전트 (Router)
         │
         ├─ "최신" 키워드 감지
         │  → 벡터 DB에는 최신 정보가 없을 수 있음
         │  → 웹 검색으로 라우팅
         │
         ├─ "GIL" 키워드
         │  → 벡터 DB에서 기존 글 검색
         │
         └─ 두 결과를 합쳐서 LLM에 전달
```

```python
def route_query(query: str) -> list[str]:
    """쿼리를 분석해서 적절한 검색 소스를 선택"""
    prompt = f"""다음 쿼리에 답하기 위해 어떤 검색 소스를 사용해야 하는지 판단하세요.

쿼리: {query}

사용 가능한 소스:
- vector_db: 기존 블로그 글 검색 (과거 데이터)
- web_search: 최신 정보 검색
- notion_db: Notion에 저장된 초안/메모

JSON 배열로 답하세요: ["vector_db", "web_search"]
"""
    return llm.generate(prompt)
```

### 3.4 다단계 검색 (Multi-step Retrieval)

하나의 질문을 여러 하위 질문으로 나누어 각각 검색한다.

```
사용자: "GIL과 asyncio의 관계를 설명하는 글을 써줘"

에이전트:
  하위 질문 1: "Python GIL이란 무엇인가?"
  하위 질문 2: "asyncio의 동작 원리는?"
  하위 질문 3: "GIL이 asyncio에 미치는 영향은?"

  → 각 하위 질문으로 검색
  → 3개의 검색 결과를 모두 합쳐서 LLM에 전달
  → 포괄적인 글 생성
```

---

## 4. Post-CS에서의 Agentic RAG 흐름

### 블로그 글 생성 시

```
사용자: "Python 동시성 프로그래밍 글을 생성해줘"
    │
    ▼
[에이전트 시작]
    │
    ├── Step 1: 쿼리 분석
    │   "동시성 프로그래밍은 GIL, threading, multiprocessing, asyncio를 다뤄야 한다"
    │
    ├── Step 2: 기존 글 검색 (ChromaDB)
    │   쿼리 1: "Python GIL threading"
    │   쿼리 2: "Python multiprocessing 병렬 처리"
    │   쿼리 3: "Python asyncio 비동기"
    │   → 기존 글 5개 발견
    │
    ├── Step 3: 검색 결과 평가
    │   "GIL 글은 있지만, asyncio 글은 없다"
    │   "asyncio 관련 정보가 부족하다"
    │
    ├── Step 4: 부족한 부분 보완 검색
    │   추가 쿼리: "Python asyncio 이벤트 루프 코루틴"
    │   → 관련 정보 추가 확보
    │
    ├── Step 5: 중복 체크
    │   기존 GIL 글과 중복되는 부분 식별
    │   → "GIL 기본 설명은 빼고 링크를 걸자"
    │
    ├── Step 6: 글 생성 (LLM)
    │   프롬프트에 포함:
    │   - 기존 관련 글 요약 (중복 방지용)
    │   - 부족한 asyncio 참고 자료
    │   - 프롬프트 템플릿 (tone: tutorial)
    │   → 초안 생성
    │
    └── Step 7: 사후 검증
        생성된 초안의 각 문단을 벡터 검색
        → 기존 글과 유사도 > 0.9인 문단이 있으면 경고
        → 톤 일관성 체크
```

### MCP 도구와의 연결

```python
# 에이전트가 사용하는 MCP 도구들

# ChromaDB 서버
@mcp.tool()
def search_blog_posts(query: str, n: int = 5) -> str:
    """기존 블로그 글에서 유사한 내용 검색"""

@mcp.tool()
def check_paragraph_duplicates(text: str) -> str:
    """문단의 중복 여부 확인"""

@mcp.tool()
def check_tone_consistency(text: str) -> str:
    """기존 글과의 톤 일관성 점수"""

# Notion 서버
@mcp.tool()
def get_draft_content(post_id: str) -> str:
    """Notion에서 초안 내용 가져오기"""

@mcp.tool()
def upload_draft(post_id: str, content: str) -> str:
    """Notion에 초안 업로드"""

# 에이전트가 이 도구들을 ReAct 루프에서 자율적으로 사용
```

---

## 5. 기본 RAG vs Agentic RAG 비교

| 측면 | 기본 RAG | Agentic RAG |
|------|---------|-------------|
| **검색 횟수** | 1번 | 필요한 만큼 반복 |
| **쿼리** | 사용자 입력 그대로 | 재작성/분해/라우팅 |
| **결과 평가** | 없음 | 충분성 자기 평가 |
| **다중 소스** | 벡터 DB만 | 벡터 DB + 웹 + DB + API |
| **에러 복구** | 없음 | 쿼리 변경 후 재시도 |
| **비용** | LLM 1회 호출 | LLM 여러 회 호출 |
| **레이턴시** | 빠름 (1~2초) | 느림 (5~30초) |
| **정확도** | 보통 | 높음 |

### 언제 뭘 쓰나?

```
기본 RAG로 충분한 경우:
- 단순 Q&A (자주 묻는 질문)
- 검색 소스가 하나
- 실시간 응답이 필요

Agentic RAG가 필요한 경우:
- 복잡한 질문 (여러 정보 조합 필요)
- 검색 결과의 품질이 중요
- 여러 소스를 탐색해야 함
- Post-CS처럼 자동화 파이프라인
```

---

## 6. 구현 아키텍처 (Post-CS)

### 전체 구조

```
┌──────────────────────────────────────────────────────┐
│                   Blog Agent (Host)                   │
│                                                       │
│  ┌─────────┐    ┌──────────┐    ┌──────────────────┐ │
│  │  LLM    │───▶│  ReAct   │───▶│  MCP Client      │ │
│  │  (추론)  │◀───│  Loop    │◀───│  (도구 호출)      │ │
│  └─────────┘    └──────────┘    └────────┬─────────┘ │
│                                          │           │
└──────────────────────────────────────────┼───────────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    │                      │                      │
            ┌───────▼──────┐    ┌──────────▼───┐    ┌────────────▼──┐
            │ MCP Server   │    │ MCP Server   │    │ MCP Server    │
            │ ChromaDB     │    │ Notion       │    │ Velog         │
            │              │    │              │    │               │
            │ - search     │    │ - get_posts  │    │ - publish     │
            │ - check_dup  │    │ - upload     │    │ - check_spam  │
            │ - check_tone │    │ - update     │    │ - login       │
            └───────┬──────┘    └──────────┬───┘    └────────────┬──┘
                    │                      │                      │
            ┌───────▼──────┐    ┌──────────▼───┐    ┌────────────▼──┐
            │   ChromaDB   │    │  Notion API  │    │  Velog API    │
            └──────────────┘    └──────────────┘    └───────────────┘
```

### 실행 예시

```bash
# CLI에서 에이전트 모드 실행
python main.py agent "Python 동시성 프로그래밍 글을 생성해서 Notion에 올려줘"

# 에이전트가 자율적으로:
# 1. ChromaDB에서 관련 글 검색
# 2. 중복 체크
# 3. LLM으로 글 생성
# 4. 톤 일관성 검증
# 5. Notion에 업로드
# 6. 결과 보고
```

---

## 7. 직접 해보기

### 실습: Agentic RAG 시뮬레이션

```python
# 기본 RAG vs Agentic RAG의 차이를 직접 체감해보기

import chromadb

# 문서 준비
client = chromadb.Client()
collection = client.create_collection("blog")
collection.add(
    documents=[
        "Python GIL은 CPython의 핵심 메커니즘으로, 한 번에 하나의 스레드만 실행한다.",
        "멀티프로세싱은 GIL 영향 없이 CPU 코어를 활용한다.",
        "asyncio는 단일 스레드에서 비동기 I/O를 처리한다.",
        "Docker compose로 멀티 컨테이너를 관리할 수 있다.",
    ],
    ids=["1", "2", "3", "4"]
)

# === 기본 RAG ===
print("=== 기본 RAG ===")
query = "GIL 없이 Python에서 병렬 처리하는 방법"
results = collection.query(query_texts=[query], n_results=2)
print(f"쿼리: {query}")
print(f"검색 결과: {results['documents'][0]}")
print("→ 이걸 LLM에 넣어서 답변 생성 (1번 검색, 1번 생성)\n")

# === Agentic RAG ===
print("=== Agentic RAG ===")

# Step 1: 쿼리 분해
sub_queries = [
    "Python GIL이란",
    "Python 멀티프로세싱 병렬",
    "Python asyncio 비동기 처리",
]
print(f"원래 쿼리: {query}")
print(f"분해된 쿼리: {sub_queries}\n")

# Step 2: 각각 검색
all_results = []
for sq in sub_queries:
    r = collection.query(query_texts=[sq], n_results=1)
    all_results.extend(r["documents"][0])
    print(f"  쿼리: '{sq}' → {r['documents'][0][0][:50]}...")

# Step 3: 결과 평가
print(f"\n총 {len(all_results)}개 결과 수집")
print("평가: GIL, 멀티프로세싱, asyncio 모두 커버됨 ✓")
print("→ 더 포괄적인 정보로 LLM에 전달 가능")
```

---

## 핵심 정리

| 개념 | 한 줄 요약 |
|------|-----------|
| Agentic RAG | 에이전트가 검색 과정을 자율적으로 제어하는 RAG |
| 쿼리 재작성 | 검색에 더 적합한 형태로 쿼리를 변환 |
| 자기 평가 | 검색 결과가 충분한지 에이전트가 스스로 판단 |
| 라우팅 | 쿼리 성격에 따라 다른 검색 소스 선택 |
| 다단계 검색 | 하나의 질문을 여러 하위 질문으로 나눠 검색 |

---

## 전체 학습 정리

```
01. 임베딩      → 텍스트를 숫자(벡터)로 변환
02. 벡터 DB     → 벡터를 저장하고 빠르게 검색
03. RAG         → 검색 결과를 LLM 프롬프트에 넣어서 답변 강화
04. MCP         → LLM이 외부 도구를 쓰는 표준 프로토콜
05. 에이전트    → LLM이 자율적으로 도구를 사용하는 루프
06. Agentic RAG → 에이전트 + RAG = 검색도 똑똑하게, 행동도 자율적으로

이 모든 것을 Post-CS에 합치면:
에이전트가 ChromaDB에서 기존 글을 검색(RAG)하고,
MCP로 Notion/벨로그를 조작하며,
블로그 글 생성~발행을 자율적으로 수행하는 시스템
```
