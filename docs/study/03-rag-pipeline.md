# 03. RAG 파이프라인

> **핵심 질문:** LLM이 모르는 정보를 어떻게 알려줄 수 있는가?

---

## 1. RAG란?

**RAG = Retrieval-Augmented Generation (검색 증강 생성)**

LLM에게 질문하기 전에, 관련 정보를 먼저 찾아서 같이 넣어주는 것.

### 비유로 이해하기

```
[RAG 없이]
시험 문제: "Python GIL의 내부 구현에 대해 설명하시오"
학생: (기억에 의존) "음... GIL은... 뮤텍스 같은 거... 였나?"
→ 부정확하거나 할루시네이션

[RAG 있이]
시험 문제: "Python GIL의 내부 구현에 대해 설명하시오"
시험관: "여기 관련 자료 3장 드릴게요. 읽고 답하세요."  ← 검색(Retrieval)
학생: (자료 참고) "CPython의 ceval.c에서 GIL은..."    ← 생성(Generation)
→ 정확하고 근거 있는 답변
```

### RAG의 핵심 가치

| 문제 | RAG의 해결 |
|------|-----------|
| LLM이 최신 정보를 모름 | 최신 문서를 검색해서 넣어줌 |
| LLM이 사내 데이터를 모름 | 사내 DB에서 검색해서 넣어줌 |
| LLM이 없는 걸 만들어냄 (할루시네이션) | 실제 문서를 근거로 답변하게 함 |
| LLM을 재학습시키기 비쌈 | 재학습 없이 지식을 업데이트 |

---

## 2. RAG 파이프라인 전체 흐름

```
[오프라인 - 사전 준비 (Indexing)]

원본 문서들                문서를 적절한 크기로 자름       벡터로 변환         벡터 DB에 저장
┌──────────┐          ┌─────────────────┐        ┌──────────┐       ┌──────────┐
│ 블로그 글 │──────────▶│    청킹(Chunking)  │────────▶│  임베딩   │───────▶│ 벡터 DB  │
│ 문서, PDF │          │  문단/섹션 단위 분할 │        │  모델    │       │ ChromaDB │
└──────────┘          └─────────────────┘        └──────────┘       └──────────┘


[온라인 - 실시간 질의 (Retrieval + Generation)]

사용자 질문          질문을 벡터로 변환        벡터 DB에서 유사 문서 검색
┌──────────┐      ┌──────────┐          ┌──────────────────┐
│ "GIL이    │──────▶│  임베딩   │──────────▶│  벡터 유사도 검색   │
│  뭐야?"   │      │  모델    │          │  상위 K개 문서 반환  │
└──────────┘      └──────────┘          └────────┬─────────┘
                                                 │
                                                 ▼
                                    ┌──────────────────────┐
                                    │  LLM에 프롬프트 구성     │
                                    │                       │
                                    │  "아래 자료를 참고하여    │
                                    │   질문에 답하세요:       │
                                    │   [검색된 문서 1]       │
                                    │   [검색된 문서 2]       │
                                    │   질문: GIL이 뭐야?"    │
                                    └────────┬─────────────┘
                                             │
                                             ▼
                                    ┌──────────────────┐
                                    │  LLM 답변 생성     │
                                    │  (근거 있는 답변)    │
                                    └──────────────────┘
```

---

## 3. 청킹 (Chunking)

문서를 통째로 벡터로 만들면 안 되는 이유:
1. 임베딩 모델에 입력 길이 제한이 있음 (보통 512~8192 토큰)
2. 긴 문서의 임베딩은 의미가 희석됨 ("평균"이 되어버림)
3. 검색 결과로 돌려줄 때 불필요한 내용이 너무 많음

그래서 문서를 적절한 크기의 **청크(chunk)**로 잘라야 한다.

### 청킹 전략 비교

#### 1) 고정 크기 청킹 (Fixed-size)

```
원본: "GIL은 Global Interpreter Lock의 약자이다. CPython에서 사용한다.
       멀티스레딩 시 한 번에 하나의 스레드만 바이트코드를 실행할 수 있다.
       이는 메모리 관리의 안전성을 위한 것이다..."

청크 크기: 100자
→ 청크 1: "GIL은 Global Interpreter Lock의 약자이다. CPython에서 사용한다. 멀티스레딩 시 한 번에 하나의..."
→ 청크 2: "스레드만 바이트코드를 실행할 수 있다. 이는 메모리 관리의 안전성을 위한 것이다..."
```

**문제:** 문장 중간에서 잘릴 수 있다. 의미가 끊김.

#### 2) 재귀적 청킹 (Recursive) — 가장 많이 쓰임

```
분할 우선순위:
1. 단락 경계 (\n\n)로 자르기
2. 안 되면 문장 경계 (. ! ?)로 자르기
3. 안 되면 단어 경계 (공백)로 자르기
4. 안 되면 글자 단위로 자르기

+ 오버랩(overlap): 각 청크의 끝부분을 다음 청크 시작에 포함시켜 문맥 유지
```

```python
# 예시 (개념 이해용)
def recursive_chunk(text, max_size=500, overlap=50):
    # 단락으로 먼저 분할
    paragraphs = text.split("\n\n")

    chunks = []
    current_chunk = ""

    for para in paragraphs:
        if len(current_chunk) + len(para) <= max_size:
            current_chunk += para + "\n\n"
        else:
            chunks.append(current_chunk.strip())
            # overlap: 이전 청크의 마지막 부분을 가져옴
            current_chunk = current_chunk[-overlap:] + para + "\n\n"

    if current_chunk.strip():
        chunks.append(current_chunk.strip())

    return chunks
```

#### 3) 시맨틱 청킹 (Semantic)

```
문장 1: "GIL은 CPython의 핵심 메커니즘이다."        ┐
문장 2: "멀티스레딩 시 바이트코드 실행을 직렬화한다." ├─ 유사도 높음 → 같은 청크
문장 3: "이는 메모리 안전성을 보장한다."              ┘

문장 4: "반면 Docker는 컨테이너 기술이다."           ┐
문장 5: "네트워크 격리를 제공한다."                   ├─ 유사도 높음 → 다른 청크
                                                    ┘
```

문장 간 임베딩 유사도를 계산해서, 의미가 크게 바뀌는 지점에서 자른다.
정확하지만 임베딩 계산 비용이 추가된다.

### 청킹 전략 선택 가이드

| 전략 | 장점 | 단점 | 언제 쓰나 |
|------|------|------|----------|
| 고정 크기 | 단순, 빠름 | 의미 단절 | 구조 없는 긴 텍스트 |
| 재귀적 | 의미 보존, 범용적 | 최적 크기 튜닝 필요 | **대부분의 경우 (기본 선택)** |
| 시맨틱 | 가장 정확한 의미 분할 | 느림, 비용 높음 | 정확도가 중요한 프로덕션 |

### Post-CS에서의 청킹

블로그 글은 마크다운이므로 자연스러운 분할점이 있다:
- `##` (H2) 섹션 단위로 1차 분할
- 섹션이 너무 길면 문단(`\n\n`) 단위로 2차 분할
- 코드 블록은 분할하지 않음 (코드가 잘리면 의미 없음)

---

## 4. 검색 패턴 (Retrieval)

### 4.1 기본 벡터 검색 (Dense Retrieval)

01, 02에서 배운 내용. 쿼리를 임베딩 → 벡터 DB에서 코사인 유사도 검색.

```python
results = collection.query(
    query_texts=["Python GIL 우회 방법"],
    n_results=5
)
```

**한계:** "GIL"이라는 단어가 정확히 있어야 찾기 쉽지, "전역 인터프리터 락"이라고 하면 약간 못 찾을 수 있다. (의미는 같지만 표현이 다른 경우)

### 4.2 키워드 검색 (Sparse Retrieval / BM25)

전통적인 텍스트 검색. "단어가 문서에 몇 번 나오는가"로 관련도를 계산.

```
쿼리: "GIL 우회 방법"
→ "GIL"이 3번 나오는 문서A > "GIL"이 1번 나오는 문서B
```

**한계:** "동의어"를 이해 못함. "GIL" ≠ "전역 인터프리터 락" (같은 건데 다른 단어)

### 4.3 하이브리드 검색 — 2026년의 표준

두 가지를 합치면 서로의 약점을 보완한다.

```
쿼리: "Python GIL 우회 방법"
        │
        ├───▶ 벡터 검색 (Dense)
        │     → 의미적으로 유사한 문서 찾음
        │     → "전역 인터프리터 락 우회" 문서도 찾음 ✓
        │     결과: [문서A(0.89), 문서C(0.85), 문서D(0.78)]
        │
        └───▶ 키워드 검색 (BM25/Sparse)
              → "GIL"이라는 단어가 있는 문서 찾음
              → 정확한 키워드 매칭에 강함 ✓
              결과: [문서A(8.5), 문서B(7.2), 문서E(6.1)]
        │
        ▼
   Reciprocal Rank Fusion (RRF)
   → 두 결과의 순위를 합쳐서 최종 순위 결정
   → 양쪽에서 모두 높은 문서가 최종 상위에
   최종 결과: [문서A, 문서C, 문서B, 문서D, 문서E]
```

### RRF (Reciprocal Rank Fusion) 공식

```
RRF_score(문서) = Σ  1 / (k + rank_i)

k = 상수 (보통 60)
rank_i = i번째 검색 결과에서의 순위
```

예시:
- 문서A: 벡터 검색 1위, 키워드 검색 1위 → 1/(60+1) + 1/(60+1) = 0.033
- 문서C: 벡터 검색 2위, 키워드 검색 5위 → 1/(60+2) + 1/(60+5) = 0.031
- 문서B: 벡터 검색 없음, 키워드 검색 2위 → 0 + 1/(60+2) = 0.016

→ 문서A가 최종 1위

### 4.4 리랭킹 (Reranking)

검색 결과를 한번 더 정밀하게 재정렬한다.

```
1차 검색 (ANN - 빠르지만 대략적)
    → 상위 20개 후보
        │
        ▼
리랭커 (Cross-Encoder - 느리지만 정확)
    → 쿼리와 각 문서를 쌍으로 정밀 비교
    → 상위 5개 최종 선택
```

**왜 처음부터 리랭커를 안 쓰나?**
- 리랭커는 쿼리-문서 쌍을 하나씩 비교하므로 느림
- 100만 문서를 리랭커로 다 비교할 수 없음
- 그래서 ANN으로 후보를 추린 뒤, 소수만 리랭킹

```python
# 예시 (개념)
# 1단계: 벡터 검색으로 후보 20개 추출
candidates = collection.query(query_texts=["GIL 우회"], n_results=20)

# 2단계: 리랭커로 재정렬
from sentence_transformers import CrossEncoder
reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

pairs = [(query, doc) for doc in candidates["documents"][0]]
scores = reranker.predict(pairs)

# 점수순 정렬 → 상위 5개가 최종 결과
```

---

## 5. 프롬프트 구성 (Generation)

검색된 문서를 LLM 프롬프트에 넣는 방법.

### 기본 패턴

```python
def build_rag_prompt(query, retrieved_docs):
    context = "\n\n---\n\n".join(retrieved_docs)

    prompt = f"""아래 참고 자료를 바탕으로 질문에 답하세요.
참고 자료에 없는 내용은 "자료에서 확인할 수 없습니다"라고 답하세요.

## 참고 자료
{context}

## 질문
{query}

## 답변
"""
    return prompt
```

### Post-CS에서의 RAG 프롬프트

```python
def build_content_gen_prompt(topic, keypoints, related_posts):
    """기존 글을 참고하여 새 글을 생성하는 프롬프트"""

    related_context = ""
    for post in related_posts:
        related_context += f"\n### 기존 글: {post['title']}\n{post['summary']}\n"

    prompt = f"""당신은 기술 블로그 작가입니다.

## 참고: 이전에 작성한 관련 글
{related_context}

## 지시사항
- 위 기존 글과 내용이 중복되지 않도록 하세요
- 기존 글에서 다룬 개념은 간단히 언급하고 링크를 걸어주세요
- 새로운 관점이나 심화 내용에 집중하세요

## 새 글 주제
{topic}

## 반드시 포함할 핵심 포인트
{keypoints}
"""
    return prompt
```

---

## 6. RAG 평가

RAG가 잘 작동하는지 어떻게 판단하는가?

### 핵심 지표

| 지표 | 측정 대상 | 의미 |
|------|----------|------|
| **Retrieval Recall** | 검색 단계 | 관련 문서를 얼마나 빠짐없이 찾았는가 |
| **Retrieval Precision** | 검색 단계 | 찾은 문서 중 진짜 관련 있는 비율 |
| **Faithfulness** | 생성 단계 | LLM 답변이 검색 문서에 근거하는가 (할루시네이션 체크) |
| **Answer Relevance** | 생성 단계 | 답변이 질문에 적절한가 |

### 간단한 평가 방법

```
1. 테스트 질문 10개를 만든다
2. 각 질문에 대해 "정답 문서"를 미리 정해둔다
3. RAG를 실행해서 검색 결과가 정답 문서를 포함하는지 확인
4. LLM 답변이 검색 문서의 내용에 기반하는지 확인
```

---

## 7. 직접 해보기

### 실습: 미니 RAG 시스템 만들기

```python
import chromadb

# 1. 문서 준비 (가짜 블로그 글)
docs = [
    "Python GIL은 Global Interpreter Lock이다. CPython에서 한 번에 하나의 스레드만 바이트코드를 실행하도록 한다. 이는 메모리 관리의 thread-safety를 보장하기 위함이다.",
    "멀티프로세싱은 GIL의 영향을 받지 않는다. 각 프로세스가 독립적인 Python 인터프리터를 가지기 때문이다. CPU 바운드 작업에는 멀티프로세싱이 적합하다.",
    "Docker 컨테이너 네트워크에는 bridge, host, overlay 모드가 있다. bridge는 기본값이며 컨테이너 간 격리된 네트워크를 제공한다.",
    "REST API 설계 원칙: 리소스는 명사로 표현하고, 행위는 HTTP 메서드(GET, POST, PUT, DELETE)로 표현한다. URL에 동사를 넣지 않는다.",
]

# 2. 벡터 DB에 저장
client = chromadb.Client()
collection = client.create_collection("blog_posts")
collection.add(
    documents=docs,
    ids=[f"doc-{i}" for i in range(len(docs))]
)

# 3. 검색
query = "GIL을 우회하는 방법이 있나요?"
results = collection.query(query_texts=[query], n_results=2)

# 4. 프롬프트 구성
context = "\n---\n".join(results["documents"][0])
prompt = f"""참고 자료를 바탕으로 답하세요.

## 참고 자료
{context}

## 질문
{query}

## 답변"""

print("=== 구성된 프롬프트 ===")
print(prompt)

# 5. 여기서 LLM에 prompt를 보내면 RAG 완성!
# response = llm_client.generate(prompt)
```

---

## 핵심 정리

| 개념 | 한 줄 요약 |
|------|-----------|
| RAG | 검색으로 찾은 정보를 LLM 프롬프트에 넣어서 답변을 강화하는 패턴 |
| 청킹 | 문서를 적절한 크기로 자르는 것. 재귀적 청킹이 기본 |
| 하이브리드 검색 | 벡터 검색 + 키워드 검색을 합쳐서 정확도를 높이는 방법 |
| 리랭킹 | 1차 검색 결과를 더 정밀한 모델로 재정렬 |
| RRF | 여러 검색 결과의 순위를 합치는 공식 |

**다음 단계:** LLM이 외부 도구를 사용할 수 있게 하는 표준 프로토콜 → `04-mcp-protocol.md`
