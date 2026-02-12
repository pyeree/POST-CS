# 05. AI 에이전트 기초

> **핵심 질문:** LLM이 스스로 계획하고 실행하려면 무엇이 필요한가?

---

## 1. 에이전트란?

**AI 에이전트 = LLM + 도구 사용 + 자율적 판단의 루프**

일반 LLM 호출과 에이전트의 차이:

```
[일반 LLM 호출]
사용자: "GIL에 대한 블로그 글 써줘"
LLM: (한 번에) "Python GIL은..."
→ 끝. 한 번 물어보고 한 번 답함.

[에이전트]
사용자: "GIL에 대한 블로그 글을 써서 벨로그에 올려줘"
에이전트:
  1. (생각) "먼저 기존에 비슷한 글이 있는지 확인해야겠다"
  2. (행동) → search_similar_posts("Python GIL") 호출
  3. (관찰) "기존에 GIL 관련 글이 없네"
  4. (생각) "그럼 topics.yaml에서 키포인트를 가져와서 글을 생성하자"
  5. (행동) → get_topics() 호출
  6. (관찰) 키포인트 확인
  7. (생각) "이제 LLM으로 글을 생성하자"
  8. (행동) → generate_blog_post() 호출
  9. (관찰) 초안 완성
  10. (생각) "중복 체크를 해야겠다"
  11. (행동) → check_duplicate(초안 텍스트) 호출
  12. (관찰) "중복 없음"
  13. (생각) "Notion에 올리고 사용자에게 알려야겠다"
  14. (행동) → upload_to_notion() 호출
  15. (완료) "초안을 생성해서 Notion에 올렸습니다. 검토 후 발행해주세요."
```

핵심은 **에이전트가 여러 단계를 스스로 판단하고 실행**한다는 것.

---

## 2. 에이전트의 구성 요소

```
┌────────────────────────────────────────────┐
│                  AI 에이전트                  │
│                                             │
│  ┌─────────┐                                │
│  │  LLM    │  ← 두뇌. 다음에 뭘 할지 판단     │
│  │  (추론)  │                                │
│  └────┬────┘                                │
│       │                                     │
│  ┌────▼────┐   ┌──────────┐                 │
│  │  도구    │──▶│ 외부 시스템│  ← 손과 발       │
│  │ (Tools)  │◀──│ (APIs)   │                 │
│  └────┬────┘   └──────────┘                 │
│       │                                     │
│  ┌────▼────┐                                │
│  │  메모리  │  ← 이전에 한 일을 기억           │
│  │(Memory) │                                │
│  └─────────┘                                │
│                                             │
└────────────────────────────────────────────┘
```

| 구성 요소 | 역할 | Post-CS 예시 |
|-----------|------|-------------|
| **LLM** | 판단, 계획, 추론 | 온프레미스 LLM 서버 |
| **도구 (Tools)** | 외부와 상호작용 | Notion API, 벨로그 API, ChromaDB |
| **메모리** | 이전 행동/결과 기억 | 대화 히스토리, 벡터 DB |
| **계획 (Planning)** | 목표를 단계로 분해 | "글 생성 → 검토 → 발행" |

---

## 3. ReAct 패턴

가장 기본적이고 널리 쓰이는 에이전트 패턴.

**ReAct = Reasoning + Acting (추론 + 행동)**

```
[루프]
1. Thought (생각)  — "지금 상황에서 뭘 해야 하지?"
2. Action (행동)   — 도구를 호출
3. Observation (관찰) — 도구의 결과를 확인
4. → 다시 1로 (목표를 달성할 때까지)
```

### 구체적 예시: Post-CS 블로그 발행

```
사용자: "Ready to Publish 상태인 글을 벨로그에 발행해줘"

Thought 1: Notion에서 Ready to Publish 상태인 글을 먼저 확인해야 한다.
Action 1:  get_posts_by_status("Ready to Publish")
Observation 1:
  - "Python GIL 완전 정복" (ID: 2026-02-12-python-gil)
  - "Docker 네트워크 비교" (ID: 2026-02-12-docker-network)

Thought 2: 2개의 글이 있다. 하나씩 발행하자. 먼저 GIL 글의 내용을 가져오자.
Action 2:  get_notion_page("2026-02-12-python-gil")
Observation 2: [마크다운 본문 내용...]

Thought 3: 본문을 가져왔다. 벨로그에 발행하자.
Action 3:  publish_to_velog(title="Python GIL 완전 정복", body="...", tags=["Python", "GIL"])
Observation 3: ✅ 발행 완료: https://velog.io/@user/python-gil

Thought 4: GIL 글 발행 완료. Notion 상태를 Published로 업데이트하자.
Action 4:  update_notion_status("2026-02-12-python-gil", "Published")
Observation 4: ✅ 상태 업데이트 완료

Thought 5: Docker 네트워크 글도 발행해야 한다. 내용을 가져오자.
Action 5:  get_notion_page("2026-02-12-docker-network")
...

Final Answer: 2개의 글을 벨로그에 발행했습니다.
  1. Python GIL 완전 정복 → https://velog.io/@user/python-gil
  2. Docker 네트워크 비교 → https://velog.io/@user/docker-network
```

---

## 4. 에이전트 루프 구현

### 가장 단순한 에이전트 루프

```python
def agent_loop(user_request: str, tools: dict, max_steps: int = 10):
    """
    가장 기본적인 에이전트 루프.
    LLM이 도구를 호출하지 않을 때까지 반복.
    """
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": user_request}
    ]

    for step in range(max_steps):
        # 1. LLM에게 다음 행동을 물어봄
        response = llm.generate(messages, tools=tools)

        # 2. LLM이 도구 호출을 원하는가?
        if response.tool_calls:
            for tool_call in response.tool_calls:
                # 3. 도구 실행
                tool_name = tool_call.name
                tool_args = tool_call.arguments
                result = tools[tool_name](**tool_args)

                # 4. 결과를 대화에 추가 (Observation)
                messages.append({
                    "role": "tool",
                    "content": result,
                    "tool_call_id": tool_call.id
                })
        else:
            # 도구 호출 없음 = LLM이 최종 답변을 함
            return response.content

    return "최대 단계 수에 도달했습니다."
```

### 핵심 포인트

1. **LLM이 판단의 주체** — "다음에 뭘 할지"는 LLM이 결정
2. **도구 호출은 LLM의 출력** — LLM이 "이 도구를 이 인자로 호출해"라고 출력
3. **루프가 핵심** — 한 번이 아니라 목표 달성까지 반복
4. **종료 조건** — LLM이 도구를 호출하지 않으면 최종 답변으로 간주

---

## 5. 시스템 프롬프트의 중요성

에이전트의 행동은 **시스템 프롬프트**에 의해 크게 좌우된다.

```python
SYSTEM_PROMPT = """당신은 기술 블로그 자동화 에이전트입니다.

## 당신의 역할
사용자의 요청에 따라 블로그 글을 생성, 검토, 발행하는 파이프라인을 실행합니다.

## 사용 가능한 도구
- search_similar_posts: 기존 블로그 글에서 유사한 내용 검색
- get_posts_by_status: Notion DB에서 특정 상태의 글 조회
- upload_to_notion: 초안을 Notion에 업로드
- publish_to_velog: 벨로그에 발행
- check_duplicate: 중복 문단 체크

## 행동 규칙
1. 글을 생성하기 전에 항상 기존 글과의 중복을 먼저 확인하세요
2. 발행 전에 Notion 상태가 "Ready to Publish"인지 확인하세요
3. 발행 후에는 Notion 상태를 "Published"로 업데이트하세요
4. 에러가 발생하면 사용자에게 명확히 알려주세요
5. 한 번에 하나의 글만 처리하세요 (안전성)

## 행동하지 말아야 할 것
- 사용자 확인 없이 글을 삭제하지 마세요
- Draft 상태의 글을 바로 발행하지 마세요 (검토 필수)
"""
```

---

## 6. 도구 호출 (Function Calling / Tool Use)

LLM이 도구를 호출하는 메커니즘. 대부분의 LLM API가 지원한다.

### 흐름

```
[1] 도구 정의를 LLM에 전달
{
  "tools": [
    {
      "name": "search_similar_posts",
      "description": "유사한 블로그 글 검색",
      "parameters": {
        "query": {"type": "string"},
        "n_results": {"type": "integer", "default": 3}
      }
    }
  ]
}

[2] LLM이 도구 호출을 응답에 포함
{
  "role": "assistant",
  "tool_calls": [
    {
      "id": "call_001",
      "name": "search_similar_posts",
      "arguments": {"query": "Python GIL", "n_results": 3}
    }
  ]
}

[3] 애플리케이션이 도구를 실행하고 결과를 다시 LLM에 전달
{
  "role": "tool",
  "tool_call_id": "call_001",
  "content": "1. Python GIL 완전 정복 (유사도: 0.92)\n2. ..."
}

[4] LLM이 결과를 보고 다음 행동 결정
```

### MCP와의 연결

MCP의 Tool이 바로 이 도구 호출 메커니즘을 표준화한 것이다:

```
MCP Tool 정의 → LLM의 Function Calling 형식으로 변환 → LLM이 호출 → MCP 서버에서 실행
```

---

## 7. 에이전트 메모리

### 단기 메모리 (Short-term)

현재 대화의 히스토리. 이전 단계에서 뭘 했는지 기억.

```python
messages = [
    {"role": "user", "content": "글 발행해줘"},
    {"role": "assistant", "tool_calls": [...]},          # Thought 1
    {"role": "tool", "content": "Ready 상태 글 2개"},      # Observation 1
    {"role": "assistant", "tool_calls": [...]},          # Thought 2
    {"role": "tool", "content": "발행 완료"},              # Observation 2
    # ... 이 전체가 단기 메모리
]
```

**한계:** 대화가 길어지면 LLM의 컨텍스트 윈도우를 넘길 수 있다.

### 장기 메모리 (Long-term)

벡터 DB에 저장된 과거 정보. 이전 대화에서 배운 것을 기억.

```python
# 이전 세션에서 학습한 것을 벡터 DB에 저장
vector_store.add(
    documents=["사용자는 tutorial 톤을 선호한다"],
    metadatas=[{"type": "user_preference"}]
)

# 새 세션에서 검색
preferences = vector_store.query(
    query_texts=["사용자 선호도"],
    where={"type": "user_preference"}
)
```

**Post-CS에서의 장기 메모리:**
- 기존 발행 글의 스타일 (톤 벡터)
- 어떤 주제를 다뤘는지 (중복 방지)
- 성과 데이터 (어떤 글이 조회수가 높았는지)

---

## 8. 에이전트 설계 시 주의사항

### 8.1 무한 루프 방지

```python
# 반드시 최대 단계 수를 설정
for step in range(max_steps):  # max_steps = 10~20
    ...

# 또는 타임아웃
import asyncio
result = await asyncio.wait_for(agent_loop(), timeout=60)
```

### 8.2 에러 처리

```python
try:
    result = tools[tool_name](**tool_args)
except Exception as e:
    # 에러를 LLM에게 알려주면 LLM이 대안을 찾을 수 있음
    result = f"도구 실행 실패: {str(e)}"
```

### 8.3 사용자 확인 (Human-in-the-Loop)

위험한 행동 전에 사용자 확인을 받는 것.

```python
# 발행 같은 되돌릴 수 없는 행동 전에
if tool_name == "publish_to_velog":
    confirm = input(f"'{title}'을(를) 벨로그에 발행할까요? (y/n): ")
    if confirm != 'y':
        result = "사용자가 발행을 취소했습니다."
```

Post-CS의 "반자동화" 철학과 정확히 맞는다: 에이전트가 준비하되, 최종 발행은 사용자가 확인.

### 8.4 비용 관리

에이전트는 여러 번 LLM을 호출하므로 비용이 빠르게 증가한다.

```
일반 호출: 1번 = 1x 비용
에이전트:  10단계 = 10x 비용 (+ 매 단계마다 히스토리가 길어져서 토큰 증가)
```

**온프레미스 LLM의 장점:** API 과금이 없으므로 에이전트 루프의 비용 부담이 없다!

---

## 9. 직접 해보기

### 실습: 의사(pseudo) 에이전트 루프

실제 LLM 없이 에이전트 루프의 구조를 이해하는 코드:

```python
# 도구 정의
def search_posts(query: str) -> str:
    # 실제로는 ChromaDB 검색
    return f"'{query}' 관련 글 2개 발견: [GIL 정복, 동시성 패턴]"

def check_status(post_id: str) -> str:
    # 실제로는 Notion API 호출
    statuses = {
        "python-gil": "Ready to Publish",
        "docker-net": "Draft",
    }
    return f"{post_id}: {statuses.get(post_id, 'Unknown')}"

def publish(post_id: str) -> str:
    return f"✅ {post_id} 발행 완료: https://velog.io/@user/{post_id}"

tools = {
    "search_posts": search_posts,
    "check_status": check_status,
    "publish": publish,
}

# 에이전트 루프 시뮬레이션
def simulate_agent():
    print("=== 에이전트 시작 ===")
    print("사용자: Ready 상태인 글을 발행해줘\n")

    # Step 1: 상태 확인
    print("Thought: Notion에서 Ready 상태인 글을 확인해야 한다.")
    result = check_status("python-gil")
    print(f"Action: check_status('python-gil')")
    print(f"Observation: {result}\n")

    # Step 2: 발행
    print("Thought: python-gil이 Ready to Publish 상태다. 발행하자.")
    result = publish("python-gil")
    print(f"Action: publish('python-gil')")
    print(f"Observation: {result}\n")

    # Step 3: 완료
    print("Final: python-gil을 벨로그에 발행했습니다.")

simulate_agent()
```

---

## 핵심 정리

| 개념 | 한 줄 요약 |
|------|-----------|
| AI 에이전트 | LLM이 도구를 사용해서 자율적으로 목표를 달성하는 시스템 |
| ReAct | Thought → Action → Observation 루프 (가장 기본 패턴) |
| 도구 호출 | LLM이 "이 함수를 이 인자로 호출해"라고 출력하는 메커니즘 |
| 에이전트 루프 | LLM 호출 → 도구 실행 → 결과 피드백 → 반복 |
| 시스템 프롬프트 | 에이전트의 행동 규칙을 정의 |
| Human-in-the-Loop | 위험한 행동 전 사용자 확인 |

**다음 단계:** 에이전트와 RAG를 결합한 최종 형태 → `06-agentic-rag.md`
