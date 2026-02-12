# Post-CS

벨로그 기술 블로그 반자동 생성 및 발행 시스템.

주제와 키포인트만 입력하면 RAG 기반 AI 에이전트가 초안을 생성하고, Notion에서 검토 후 벨로그에 자동 발행한다.

## 아키텍처

```
topics.yaml → [RAG + LLM] → 초안 생성 → [Notion] → 검토/수정 → [Velog] → 발행
                  ↑                                                  ↑
              ChromaDB                                        GraphQL + Playwright
            (기존 글 검색)                                      (이중 발행 구조)

                        ↕ MCP 에이전트가 전체 파이프라인 오케스트레이션
```

## 기술 스택

| 영역 | 기술 |
|------|------|
| 에이전트 | MCP (Model Context Protocol) 직접 구현 |
| RAG | ChromaDB + 하이브리드 검색 + 리랭킹 |
| LLM / 임베딩 | 온프레미스 로컬 모델 (커스텀 API 어댑터) |
| 검토 | Notion API (칸반 보드 상태 관리) |
| 발행 | Velog GraphQL API + Playwright 폴백 |
| CLI | Python + Click |

## 사용법

```bash
# 초안 생성 (RAG 컨텍스트 포함)
python main.py generate

# Notion에 업로드
python main.py upload

# 벨로그에 발행 (Notion에서 Ready to Publish 상태인 글)
python main.py publish

# 에이전트 모드 (자연어로 파이프라인 실행)
python main.py agent "Ready 상태인 글을 벨로그에 발행해줘"
```

## 설정

```bash
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
playwright install chromium
```

`.env`에 API 엔드포인트와 Notion 토큰을 설정한다. `config/topics.yaml`에 주제를 작성한다.
