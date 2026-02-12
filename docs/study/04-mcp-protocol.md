# 04. MCP 프로토콜

> **핵심 질문:** LLM이 외부 시스템을 어떻게 조작하는가?

---

## 1. MCP란?

**MCP = Model Context Protocol (모델 컨텍스트 프로토콜)**

LLM이 외부 도구, 데이터, 서비스에 접근할 수 있게 해주는 **표준 프로토콜**.

### 비유로 이해하기

```
[MCP 없이]
LLM은 "방에 갇힌 천재" — 엄청 똑똑하지만 바깥 세상과 소통할 수 없음
→ Notion DB를 못 읽고, 벨로그에 글을 못 올리고, 파일을 못 만듦

[MCP 있이]
LLM에게 "전화기"를 줌 — 필요하면 외부에 전화해서 정보를 가져오거나 행동을 요청
→ "Notion에서 Ready 상태인 글 목록 줘" → MCP 서버가 Notion API 호출 → 결과를 LLM에게 전달
→ "이 글을 벨로그에 발행해" → MCP 서버가 벨로그 GraphQL API 호출
```

### USB 비유 (Anthropic 공식 비유)

```
[USB 표준 이전]
각 기기마다 다른 케이블 → 프린터 전용 케이블, 카메라 전용 케이블, ...
= 각 LLM에 맞는 커스텀 통합 코드를 매번 작성

[USB 표준 이후]
하나의 포트로 모든 기기 연결
= MCP 하나로 모든 LLM이 모든 도구에 연결
```

---

## 2. MCP 아키텍처

### 핵심 구성 요소

```
┌─────────────────────────────────────────┐
│              MCP Host                    │
│  (Claude Desktop, IDE, 커스텀 앱 등)     │
│                                          │
│  ┌──────────────────────────────────┐   │
│  │          MCP Client               │   │
│  │  (Host 안에 내장된 프로토콜 클라이언트)│   │
│  └──────┬───────────┬───────────┬───┘   │
│         │           │           │        │
└─────────┼───────────┼───────────┼────────┘
          │           │           │
    ┌─────▼────┐ ┌────▼─────┐ ┌──▼────────┐
    │MCP Server│ │MCP Server│ │MCP Server  │
    │ (Notion) │ │ (Velog)  │ │ (ChromaDB) │
    └─────┬────┘ └────┬─────┘ └──┬────────┘
          │           │           │
    ┌─────▼────┐ ┌────▼─────┐ ┌──▼────────┐
    │Notion API│ │벨로그 API │ │ChromaDB   │
    └──────────┘ └──────────┘ └───────────┘
```

### 역할 정리

| 구성 요소 | 역할 | Post-CS 예시 |
|-----------|------|-------------|
| **Host** | MCP를 사용하는 애플리케이션 | 블로그 파이프라인 에이전트 |
| **Client** | Host와 Server 사이 통신 관리 | 에이전트 내 MCP 클라이언트 |
| **Server** | 특정 기능을 노출하는 경량 프로그램 | Notion 서버, Velog 서버, ChromaDB 서버 |

---

## 3. MCP의 3가지 핵심 기능

MCP 서버가 노출할 수 있는 것은 3가지:

### 3.1 Tools (도구)

LLM이 **호출**할 수 있는 함수. 실행하면 부수효과(side effect)가 발생할 수 있다.

```json
{
  "name": "publish_to_velog",
  "description": "마크다운 글을 벨로그에 발행합니다",
  "inputSchema": {
    "type": "object",
    "properties": {
      "title": { "type": "string", "description": "글 제목" },
      "body": { "type": "string", "description": "마크다운 본문" },
      "tags": { "type": "array", "items": { "type": "string" } }
    },
    "required": ["title", "body"]
  }
}
```

LLM이 이 도구를 보고: "아, 벨로그에 글을 올릴 수 있구나. title과 body를 넣으면 되겠다."

**Post-CS Tools 예시:**
- `search_similar_posts` — ChromaDB에서 유사한 글 검색
- `upload_to_notion` — Notion에 초안 업로드
- `publish_to_velog` — 벨로그에 발행
- `check_duplicate` — 중복 문단 체크
- `get_notion_status` — Notion DB 상태 조회

### 3.2 Resources (리소스)

LLM이 **읽을** 수 있는 데이터. 파일, DB 레코드 등.

```json
{
  "uri": "notion://posts/2026-02-12-python-gil",
  "name": "Python GIL 완전 정복 (Notion 초안)",
  "mimeType": "text/markdown"
}
```

LLM이: "이 리소스를 읽으면 Notion에 있는 초안 내용을 볼 수 있겠다."

**Post-CS Resources 예시:**
- `notion://posts/{id}` — Notion에 있는 특정 초안
- `drafts://local/{id}` — 로컬 drafts 폴더의 초안
- `config://topics` — topics.yaml 내용

### 3.3 Prompts (프롬프트)

미리 정의된 프롬프트 템플릿. LLM이 **선택**해서 사용할 수 있다.

```json
{
  "name": "generate_blog_post",
  "description": "기술 블로그 글 생성용 프롬프트",
  "arguments": [
    { "name": "topic", "required": true },
    { "name": "tone", "required": true },
    { "name": "keypoints", "required": true }
  ]
}
```

**Post-CS Prompts 예시:**
- `generate_blog_post` — 톤별 글 생성 프롬프트
- `check_tone_consistency` — 톤 일관성 체크 프롬프트
- `generate_seo_title` — SEO 제목 생성 프롬프트

---

## 4. MCP 통신 방식

### JSON-RPC 2.0

MCP는 내부적으로 JSON-RPC 2.0 프로토콜을 사용한다.

```
[Client → Server] 요청
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_similar_posts",
    "arguments": {
      "query": "Python 동시성",
      "n_results": 3
    }
  }
}

[Server → Client] 응답
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "관련 글 3개를 찾았습니다:\n1. Python GIL 완전 정복 (유사도: 0.92)\n..."
      }
    ]
  }
}
```

### 전송 계층 (Transport)

MCP 메시지를 어떻게 주고받는가:

| Transport | 방식 | 특징 | 용도 |
|-----------|------|------|------|
| **stdio** | 표준 입출력 | 로컬 프로세스 간 통신. 가장 간단 | 로컬 개발, CLI 도구 |
| **HTTP + SSE** | HTTP 스트리밍 | 원격 서버 연결. 인증/방화벽 통과 | 원격 서버, 프로덕션 |

**Post-CS에서는:**
- 로컬 실행이면 `stdio` 사용 (간단)
- 원격 서버 연동이면 `HTTP + SSE` 사용

---

## 5. MCP 서버 만들기 (Python)

### 설치

```bash
pip install mcp
```

### 기본 MCP 서버 구조

```python
# src/mcp_servers/chromadb_server.py

from mcp.server.fastmcp import FastMCP

# MCP 서버 생성
mcp = FastMCP("chromadb-search")

# Tool 정의
@mcp.tool()
def search_similar_posts(query: str, n_results: int = 3) -> str:
    """ChromaDB에서 쿼리와 유사한 블로그 글을 검색합니다.

    Args:
        query: 검색할 텍스트 (예: "Python 동시성 프로그래밍")
        n_results: 반환할 결과 수
    """
    import chromadb
    client = chromadb.PersistentClient(path="./chroma_data")
    collection = client.get_collection("blog_posts")

    results = collection.query(
        query_texts=[query],
        n_results=n_results
    )

    # 결과를 텍스트로 포맷팅
    output = []
    for i, (doc, meta) in enumerate(zip(
        results["documents"][0],
        results["metadatas"][0]
    )):
        output.append(f"{i+1}. [{meta.get('title', 'Untitled')}]\n   {doc[:200]}...")

    return "\n\n".join(output)


@mcp.tool()
def check_duplicate(text: str, threshold: float = 0.1) -> str:
    """텍스트가 기존 글과 중복되는지 확인합니다.

    Args:
        text: 중복 체크할 텍스트
        threshold: 유사도 임계값 (낮을수록 엄격)
    """
    # ... ChromaDB 검색 로직 ...
    return "중복 없음" or "⚠️ 중복 발견: ..."


# Resource 정의
@mcp.resource("config://topics")
def get_topics() -> str:
    """topics.yaml의 현재 내용을 반환합니다."""
    import yaml
    with open("config/topics.yaml") as f:
        return yaml.dump(yaml.safe_load(f), allow_unicode=True)


# 서버 실행
if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### Notion MCP 서버 예시

```python
# src/mcp_servers/notion_server.py

from mcp.server.fastmcp import FastMCP

mcp = FastMCP("notion-blog")

@mcp.tool()
def get_posts_by_status(status: str) -> str:
    """Notion DB에서 특정 상태의 글 목록을 가져옵니다.

    Args:
        status: 글 상태 (Draft, In Review, Ready to Publish, Published)
    """
    from notion_client import Client
    import os

    notion = Client(auth=os.environ["NOTION_TOKEN"])
    results = notion.databases.query(
        database_id=os.environ["NOTION_DATABASE_ID"],
        filter={"property": "Status", "status": {"equals": status}}
    )

    posts = []
    for page in results["results"]:
        title = page["properties"]["Name"]["title"][0]["plain_text"]
        post_id = page["properties"]["Topic ID"]["rich_text"][0]["plain_text"]
        posts.append(f"- {title} (ID: {post_id})")

    return "\n".join(posts) if posts else f"'{status}' 상태의 글이 없습니다."


@mcp.tool()
def upload_draft_to_notion(post_id: str, title: str, markdown: str, tags: list[str]) -> str:
    """마크다운 초안을 Notion DB에 업로드합니다.

    Args:
        post_id: 글 ID (topics.yaml의 id)
        title: 글 제목
        markdown: 마크다운 본문
        tags: 태그 목록
    """
    # ... Notion API 호출 로직 ...
    return f"✅ '{title}' 업로드 완료 (Notion URL: ...)"


if __name__ == "__main__":
    mcp.run(transport="stdio")
```

---

## 6. MCP 클라이언트에서 서버 연결

### 에이전트(Host)에서 MCP 서버 사용

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

# MCP 서버 연결
server_params = StdioServerParameters(
    command="python",
    args=["src/mcp_servers/chromadb_server.py"]
)

async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()

        # 사용 가능한 도구 목록 확인
        tools = await session.list_tools()
        print(tools)
        # [Tool(name="search_similar_posts", ...), Tool(name="check_duplicate", ...)]

        # 도구 호출
        result = await session.call_tool(
            "search_similar_posts",
            arguments={"query": "Python 동시성", "n_results": 3}
        )
        print(result)
```

---

## 7. MCP의 핵심 설계 원칙

### 7.1 서버는 가볍게

각 MCP 서버는 **하나의 관심사**만 담당한다.

```
❌ 하나의 거대한 서버에 모든 기능
   → notion + velog + chromadb + file system

✅ 관심사별로 분리
   → notion_server.py     (Notion 전담)
   → velog_server.py      (Velog 전담)
   → chromadb_server.py   (검색 전담)
```

### 7.2 Tool 설명이 중요

LLM이 도구를 선택하는 근거는 **도구의 설명(description)**이다.

```python
# ❌ 설명이 부족
@mcp.tool()
def search(q: str) -> str:
    """검색합니다."""
    pass

# ✅ 설명이 충분
@mcp.tool()
def search_similar_posts(query: str, n_results: int = 3) -> str:
    """ChromaDB에서 쿼리와 의미적으로 유사한 기존 블로그 글을 검색합니다.
    새 글 작성 시 중복 확인이나 관련 글 링크 생성에 사용하세요.

    Args:
        query: 검색할 텍스트 (예: "Python 동시성 프로그래밍")
        n_results: 반환할 결과 수 (기본 3개)
    """
    pass
```

### 7.3 에러를 명확하게

```python
@mcp.tool()
def publish_to_velog(title: str, body: str) -> str:
    try:
        url = velog_publish(title, body)
        return f"✅ 발행 완료: {url}"
    except AuthExpiredError:
        return "❌ 벨로그 세션이 만료되었습니다. `python main.py login`으로 재로그인하세요."
    except SpamFilterError:
        return "⚠️ 스팸 필터에 걸렸을 수 있습니다. 벨로그에서 글 상태를 확인하세요."
```

---

## 8. 직접 해보기

### 실습 1: 가장 간단한 MCP 서버

```python
# simple_server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("hello-world")

@mcp.tool()
def greet(name: str) -> str:
    """이름을 받아 인사합니다."""
    return f"안녕하세요, {name}님!"

@mcp.tool()
def add(a: int, b: int) -> str:
    """두 숫자를 더합니다."""
    return f"{a} + {b} = {a + b}"

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

```bash
# 테스트 (mcp 패키지의 inspector 사용)
mcp dev simple_server.py
```

### 실습 2: Claude Desktop에 연결해보기

`~/Library/Application Support/Claude/claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "hello-world": {
      "command": "python",
      "args": ["/path/to/simple_server.py"]
    }
  }
}
```

Claude Desktop을 재시작하면 "greet"과 "add" 도구가 나타난다.

---

## 핵심 정리

| 개념 | 한 줄 요약 |
|------|-----------|
| MCP | LLM이 외부 도구/데이터에 접근하는 표준 프로토콜 |
| Host | MCP를 사용하는 애플리케이션 (에이전트) |
| Client | Host 내부에서 서버와 통신하는 프로토콜 클라이언트 |
| Server | 특정 기능을 노출하는 경량 프로그램 |
| Tool | LLM이 호출할 수 있는 함수 (행동) |
| Resource | LLM이 읽을 수 있는 데이터 (정보) |
| Prompt | 미리 정의된 프롬프트 템플릿 |

**다음 단계:** MCP 도구를 활용해서 자율적으로 판단하고 행동하는 AI 에이전트 → `05-ai-agents.md`
