# 02. 벡터 데이터베이스

> **핵심 질문:** 수만 개의 벡터에서 비슷한 것을 어떻게 빠르게 찾는가?

---

## 1. 왜 벡터 DB가 필요한가?

임베딩으로 텍스트를 벡터로 바꿨다. 이제 "비슷한 벡터 찾기"를 해야 한다.

### 단순한 방법 (Brute Force)

```python
# 모든 벡터와 하나씩 비교
query_vec = embed("Python 동시성")
for doc_vec in all_vectors:      # 10만개면 10만번 비교
    similarity = cosine_sim(query_vec, doc_vec)
```

**문제:** 벡터가 10만 개면 10만 번 비교해야 한다. 100만 개면? 느려서 못 쓴다.

### 벡터 DB의 해결책

벡터 DB는 **인덱스**를 만들어서, 전부 다 비교하지 않고도 비슷한 벡터를 빠르게 찾는다.

```
Brute Force:  O(N)     — 100만 벡터 → 100만 번 비교
벡터 DB:      O(log N) — 100만 벡터 → ~20번 비교로 근사 결과
```

---

## 2. 벡터 DB의 핵심 개념

### 2.1 ANN (Approximate Nearest Neighbor)

정확한 최근접 이웃을 찾는 대신, **거의 정확한** 근사 결과를 훨씬 빠르게 찾는다.

```
정확한 검색 (KNN):  100만 벡터, 100% 정확, 500ms
근사 검색 (ANN):    100만 벡터, 95% 정확, 5ms    ← 100배 빠르고 충분히 정확
```

### 2.2 인덱싱 알고리즘

벡터 DB마다 다른 인덱싱 알고리즘을 사용한다:

**HNSW (Hierarchical Navigable Small World)** — 가장 많이 쓰임
- 그래프 기반. 벡터들을 여러 계층의 그래프로 연결
- 상위 계층에서 대략적 위치를 찾고, 하위 계층에서 정밀 탐색

```
계층 2:  A ---- D ---- G        ← 거친 탐색 (큰 점프)
         |      |
계층 1:  A -- C -- D -- F -- G  ← 중간 탐색
         |  |  |  |  |  |  |
계층 0:  A B C D E F G H I J    ← 정밀 탐색 (모든 노드)
```

**IVF (Inverted File Index)**
- 벡터를 클러스터로 묶어놓고, 쿼리와 가까운 클러스터만 탐색
- 속도 빠르지만 클러스터 경계에 있는 벡터를 놓칠 수 있음

ChromaDB는 기본으로 HNSW를 사용한다.

### 2.3 메타데이터 필터링

벡터 검색 + 조건 필터링을 동시에 할 수 있다.

```python
# "Python" 태그가 있는 문서 중에서 의미적으로 비슷한 것을 찾기
results = collection.query(
    query_texts=["GIL 우회 방법"],
    where={"tag": "Python"},        # ← 메타데이터 필터
    n_results=5
)
```

이게 없으면 전체에서 검색 후 Python 태그만 필터링해야 해서 비효율적.

---

## 3. 주요 벡터 DB 비교

| DB | 타입 | 특징 | 적합 대상 |
|---|---|---|---|
| **ChromaDB** | 임베디드 (로컬) | 설치 간편, Python 네이티브, 개발자 친화적 | 프로토타입, 중소규모 |
| **Qdrant** | 서버형/임베디드 | Rust 기반, 메타데이터 필터링 강력 | 프로덕션, 셀프호스팅 |
| **Weaviate** | 서버형 | 지식 그래프 + 벡터, GraphQL | 복잡한 관계 데이터 |
| **Pinecone** | 클라우드 관리형 | 완전 관리형, 인프라 걱정 없음 | SaaS, 빠른 프로덕션 |
| **pgvector** | PostgreSQL 확장 | 기존 DB에 벡터 검색 추가 | 이미 PostgreSQL 쓸 때 |
| **Milvus** | 분산 서버형 | 수십억 벡터 스케일 | 초대형 시스템 |

### Post-CS에서 ChromaDB를 선택한 이유

- 로컬에서 돌아감 (서버 필요 없음)
- Python 한 줄로 시작 가능
- 블로그 글 수백~수천 개 수준에서 충분
- 나중에 Qdrant 등으로 마이그레이션 가능 (인터페이스 비슷)

---

## 4. ChromaDB 기초 사용법

### 설치

```bash
pip install chromadb
```

### 기본 흐름

```python
import chromadb

# 1. 클라이언트 생성 (로컬 파일에 저장)
client = chromadb.PersistentClient(path="./chroma_data")

# 2. 컬렉션(= 테이블) 생성
collection = client.get_or_create_collection(
    name="blog_posts",
    metadata={"hnsw:space": "cosine"}  # 코사인 유사도 사용
)

# 3. 문서 추가 (임베딩은 ChromaDB가 자동 생성!)
collection.add(
    documents=[
        "Python GIL은 멀티스레딩에서 한 번에 하나의 스레드만 실행하도록 제한한다.",
        "Docker 네트워크에는 bridge, host, overlay 세 가지 모드가 있다.",
        "REST API 설계 시 리소스는 명사로, 행위는 HTTP 메서드로 표현한다.",
    ],
    metadatas=[
        {"tag": "Python", "tone": "tutorial"},
        {"tag": "Docker", "tone": "comparison"},
        {"tag": "API",    "tone": "tutorial"},
    ],
    ids=["post-001", "post-002", "post-003"]
)

# 4. 검색
results = collection.query(
    query_texts=["파이썬 동시성 프로그래밍"],
    n_results=2
)

print(results["documents"])
# [["Python GIL은 멀티스레딩에서...", "REST API 설계 시..."]]
# → GIL 글이 가장 유사하게 나옴!
```

### 핵심 포인트

- `collection.add()` — 문서를 추가하면 자동으로 임베딩 생성 + 저장
- `collection.query()` — 쿼리 텍스트를 임베딩 → 저장된 벡터와 비교 → 가장 비슷한 것 반환
- ChromaDB는 기본 임베딩 모델을 내장하고 있어서 별도 설정 없이 바로 사용 가능
- 커스텀 임베딩 모델을 쓰려면 `embedding_function` 파라미터로 교체

### 메타데이터 필터링

```python
# Python 태그인 글 중에서만 검색
results = collection.query(
    query_texts=["멀티스레딩 성능 최적화"],
    where={"tag": "Python"},
    n_results=3
)

# 복합 필터
results = collection.query(
    query_texts=["멀티스레딩"],
    where={
        "$and": [
            {"tag": "Python"},
            {"tone": "tutorial"}
        ]
    },
    n_results=3
)
```

### 문서 업데이트 / 삭제

```python
# 업데이트
collection.update(
    ids=["post-001"],
    documents=["수정된 Python GIL 글 내용..."],
    metadatas=[{"tag": "Python", "tone": "deep-dive"}]  # 톤 변경
)

# 삭제
collection.delete(ids=["post-003"])
```

---

## 5. 커스텀 임베딩 모델 연결

ChromaDB의 기본 임베딩 모델 대신 자체 모델을 사용할 수 있다.

```python
from chromadb.utils import embedding_functions

# 방법 1: sentence-transformers 모델 사용
ef = embedding_functions.SentenceTransformerEmbeddingFunction(
    model_name="BAAI/bge-m3"
)

# 방법 2: 커스텀 API (Post-CS의 온프레미스 서버용)
class CustomEmbeddingFunction(embedding_functions.EmbeddingFunction):
    def __call__(self, input: list[str]) -> list[list[float]]:
        # 온프레미스 서버의 임베딩 API 호출
        response = requests.post(
            "http://your-server/api/embed",
            json={"texts": input}
        )
        return response.json()["embeddings"]

ef = CustomEmbeddingFunction()

# 컬렉션 생성 시 임베딩 함수 지정
collection = client.get_or_create_collection(
    name="blog_posts",
    embedding_function=ef
)
```

---

## 6. Post-CS에서의 활용 시나리오

### 시나리오 1: 중복 탐지

```python
# 새 글의 각 문단을 검색해서 기존 글과 얼마나 비슷한지 체크
for paragraph in new_post_paragraphs:
    results = collection.query(
        query_texts=[paragraph],
        n_results=1
    )
    similarity = results["distances"][0][0]  # 거리 (낮을수록 유사)

    if similarity < 0.1:    # 코사인 거리 0.1 미만 = 매우 유사
        print(f"⚠️ 중복 의심: '{paragraph[:50]}...'")
        print(f"   기존 글: {results['metadatas'][0][0]['post_id']}")
```

### 시나리오 2: 내부 링크 추천

```python
# 새 글의 주제와 관련된 기존 글 찾기
related = collection.query(
    query_texts=["Python 동시성 프로그래밍"],
    n_results=3
)
# → 관련 글 URL을 본문에 자동 삽입
```

### 시나리오 3: 톤 일관성 검사

```python
# 기존 글들의 평균 벡터와 새 글의 벡터를 비교
# → 톤이 갑자기 달라지면 경고
```

---

## 7. 직접 해보기

### 실습 1: ChromaDB 기본 CRUD

```python
# pip install chromadb

import chromadb

client = chromadb.Client()  # 인메모리 (실습용)
collection = client.create_collection("test")

# 추가
collection.add(
    documents=["Python은 인터프리터 언어다", "Java는 컴파일 언어다", "맛있는 파스타 레시피"],
    ids=["1", "2", "3"]
)

# 검색
results = collection.query(query_texts=["프로그래밍 언어"], n_results=2)
print(results["documents"])  # Python, Java 글이 나오고, 파스타는 안 나옴

# 메타데이터 필터 검색
collection.add(
    documents=["Go는 정적 타입 언어다"],
    metadatas=[{"type": "compiled"}],
    ids=["4"]
)
results = collection.query(
    query_texts=["프로그래밍"],
    where={"type": "compiled"},
    n_results=2
)
print(results["documents"])  # Go만 나옴
```

### 실습 2: 블로그 글 유사도 검색

자기가 쓴 블로그 글 3~5개를 텍스트로 넣어보고, 새로운 주제를 검색해서 어떤 글이 가장 관련 있는지 확인해보자.

---

## 핵심 정리

| 개념 | 한 줄 요약 |
|------|-----------|
| 벡터 DB | 벡터를 저장하고 비슷한 벡터를 빠르게 찾는 전문 DB |
| ANN | 정확도를 조금 희생해서 검색 속도를 극적으로 높이는 알고리즘 |
| HNSW | 가장 많이 쓰이는 인덱싱 알고리즘 (그래프 기반) |
| 메타데이터 필터링 | 벡터 유사도 + 조건 필터를 동시에 적용 |
| ChromaDB | 로컬/가벼운 벡터 DB. Python 네이티브 |

**다음 단계:** 임베딩과 벡터 DB를 활용해서 LLM의 답변을 강화하는 방법 → `03-rag-pipeline.md`
