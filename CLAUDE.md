# Post-CS: 벨로그 반자동 블로그 시스템

## 프로젝트 개요
취업 준비용 기술 블로그를 반자동으로 생성 및 발행하는 시스템.
사용자가 주제와 키포인트만 입력하면 AI가 초안을 생성하고, Notion에서 검토 후 벨로그에 자동 업로드한다.

## 핵심 워크플로우
1. 사용자가 `config/topics.yaml`에 주제 + 키포인트 작성
2. `python main.py generate` → AI가 마크다운 초안 + 다이어그램 + 썸네일 생성
3. `python main.py upload` → Notion DB에 초안 업로드 (코드 블록 + Mermaid 렌더링)
4. Notion에서 초안 검토/수정 (모바일 포함, 칸반 보드 상태 관리)
5. Notion Status → "Ready to Publish"로 변경
6. `python main.py publish` → Notion에서 마크다운 추출 → 벨로그 자동 발행

## 기술 스택
- **언어:** Python 3.10+
- **에이전트 프로토콜:** MCP (Model Context Protocol) — 직접 구현, 프레임워크 미사용
- **RAG:** ChromaDB + 하이브리드 검색 (Dense + BM25) + 리랭킹
- **검토 플랫폼:** Notion (중간 계층 — 초안 검토/상태 관리)
- **브라우저 자동화:** Playwright (벨로그 발행용, 폴백)
- **벨로그 발행:** GraphQL API (기본) + Playwright (폴백)
- **LLM:** 온프레미스 로컬 모델 (커스텀 API)
- **이미지 생성:** 온프레미스 로컬 이미지 모델 (커스텀 API)
- **임베딩:** 온프레미스 임베딩 모델 (BGE-M3 등) 또는 LLM 서버 임베딩 API
- **다이어그램:** Mermaid (Notion + 벨로그 모두 자체 렌더링)
- **Vector DB:** ChromaDB (RAG 검색, 톤 일관성, 중복 탐지)
- **CI/CD:** GitHub Actions (자동 생성/발행 파이프라인)
- **실행 환경:** 로컬 Mac + GitHub Actions

## AI 서버 정보
- LLM과 이미지 모델은 별도 온프레미스 서버에서 운영
- 커스텀 API 형식 (OpenAI 호환이 아님)
- API 엔드포인트, 인증 방식은 `.env`에서 설정
- `src/api/` 모듈의 어댑터 패턴으로 호출

## 대상 플랫폼
- **발행:** 벨로그(Velog) 단일 타겟
  - 비공식 GraphQL API (`writePost` mutation) 기본 사용
  - GraphQL 실패 시 Playwright 브라우저 자동화로 폴백
  - 스팸 필터 주의: AI 생성 글이 자동 비공개 처리될 수 있음
- **검토:** Notion (중간 계층)
  - 공식 API로 안정적 연동
  - 코드 블록 + Mermaid 네이티브 렌더링
  - 칸반 보드 기반 상태 관리

## 프로젝트 구조
```
Post-CS/
├── config/
│   ├── topics.yaml              # 주제 + 키포인트 입력
│   ├── .env                     # API 엔드포인트, 인증키
│   └── prompt_templates/        # tone별 프롬프트 템플릿
├── src/
│   ├── cli.py                   # CLI 인터페이스
│   ├── config_loader.py         # YAML, .env 로딩
│   ├── api/
│   │   ├── llm_client.py        # LLM 서버 어댑터
│   │   ├── image_client.py      # 이미지 모델 서버 어댑터
│   │   └── embedding_client.py  # 임베딩 모델 서버 어댑터
│   ├── generator/
│   │   ├── content_gen.py       # 본문 생성 (RAG 컨텍스트 포함)
│   │   ├── diagram_gen.py       # Mermaid 다이어그램 처리
│   │   └── thumbnail_gen.py     # 썸네일 생성
│   ├── rag/                     # RAG 파이프라인
│   │   ├── indexer.py           # 문서 청킹 + 벡터화 + 인덱싱
│   │   ├── retriever.py         # 하이브리드 검색 (Dense + BM25)
│   │   ├── reranker.py          # 검색 결과 리랭킹
│   │   └── context_builder.py   # 검색 결과 → LLM 프롬프트 컨텍스트 구성
│   ├── mcp_servers/             # MCP 서버 (각 외부 시스템별)
│   │   ├── chromadb_server.py   # ChromaDB 검색/중복체크 MCP 서버
│   │   ├── notion_server.py     # Notion CRUD MCP 서버
│   │   └── velog_server.py      # 벨로그 발행 MCP 서버
│   ├── agent/                   # 에이전트 오케스트레이션
│   │   ├── blog_agent.py        # ReAct 루프 기반 블로그 에이전트
│   │   ├── mcp_client.py        # MCP 클라이언트 (서버 연결 관리)
│   │   └── prompts.py           # 에이전트 시스템 프롬프트
│   ├── notion/                  # Notion 중간 계층
│   │   ├── notion_client.py     # Notion API 래퍼
│   │   ├── md_to_blocks.py      # 마크다운 → Notion 블록 변환
│   │   ├── blocks_to_md.py      # Notion 블록 → 마크다운 변환
│   │   ├── database_manager.py  # Notion DB CRUD + 상태 관리
│   │   └── image_uploader.py    # Notion 파일 업로드 API
│   ├── publisher/
│   │   ├── velog_graphql.py     # 벨로그 GraphQL API 발행 (기본)
│   │   ├── velog_playwright.py  # 벨로그 Playwright 발행 (폴백)
│   │   └── auth.py              # 벨로그 로그인/세션 관리
│   ├── seo/
│   │   ├── keyword_analyzer.py  # 검색 의도 분석 + 키워드 밀도 체크
│   │   ├── internal_linker.py   # 기존 글 기반 내부 링크 자동 삽입 (RAG 활용)
│   │   └── title_optimizer.py   # 제목 A/B 후보 생성
│   ├── memory/
│   │   ├── vector_store.py      # ChromaDB 기반 글 벡터화/검색
│   │   ├── tone_checker.py      # 톤 일관성 검증
│   │   └── dedup_detector.py    # 중복 문단 탐지
│   ├── analytics/
│   │   ├── stats_collector.py   # 조회수/반응 수집
│   │   └── feedback_loop.py     # 성과 기반 자동 개선 판단
│   └── utils/
│       ├── markdown_utils.py    # 마크다운 파싱/변환
│       └── logger.py            # 로깅
├── drafts/                      # 생성된 초안 (로컬 백업)
├── logs/                        # 실행 로그
├── docs/
│   ├── PLAN.md                  # 상세 계획서
│   └── study/                   # RAG/MCP/에이전트 학습 자료
├── .github/
│   └── workflows/
│       └── auto_publish.yml     # GitHub Actions 자동 파이프라인
├── main.py
└── requirements.txt
```

## 개발 규칙
- 커스텀 API 연동은 반드시 어댑터 패턴 사용 (나중에 모델 교체 용이하게)
- 모든 설정은 `.env` 또는 `topics.yaml`에서 관리 (하드코딩 금지)
- 벨로그 DOM 셀렉터는 `src/publisher/` 내에서만 관리
- 초안은 항상 `drafts/{id}/` 디렉토리 단위로 로컬 백업
- Notion이 검토의 중심 — 상태 관리는 Notion DB 우선, topics.yaml은 입력 전용
- 벨로그 발행은 GraphQL 기본 → Playwright 폴백 이중 구조
- MCP 서버는 관심사별로 분리 (Notion/Velog/ChromaDB 각각 독립 서버)
- MCP는 프레임워크 없이 `mcp` SDK로 직접 구현 (학습 목적)
- RAG 파이프라인은 `src/rag/`에서 관리, 에이전트는 `src/agent/`에서 관리
- 에이전트는 단일 에이전트(ReAct 루프)로 시작, 필요 시 점진적 확장

## 현재 진행 상황

### 코어 시스템 (MVP + RAG/MCP)
- [x] 프로젝트 계획 수립
- [x] 상세 계획 문서 작성 (`docs/PLAN.md`)
- [x] RAG/MCP/에이전트 학습 자료 작성 (`docs/study/`)
- [ ] Phase 1: 프로젝트 기반 구축 (디렉토리, 설정 파일, CLI)
- [ ] Phase 2: LLM 연동 + 콘텐츠 생성
- [ ] Phase 3: Vector DB + RAG 파이프라인 (ChromaDB, 임베딩, 하이브리드 검색)
- [ ] Phase 4: 이미지 파이프라인 (다이어그램 + 썸네일)
- [ ] Phase 5: Notion 연동 (초안 업로드 + 검토 워크플로우)
- [ ] Phase 6: 벨로그 발행 자동화 (GraphQL + Playwright 폴백)
- [ ] Phase 7: MCP 에이전트 레이어 (MCP 서버 구축 + ReAct 에이전트)
- [ ] Phase 8: 통합 테스트 + 안정화

### 확장 시스템 (선도형 AI 콘텐츠 운영)
- [ ] Phase 9: SEO 자동화 (키워드 분석, 내부 링크, 제목 A/B)
- [ ] Phase 10: GitHub Actions 자동 파이프라인 (CI/CD + 예약 발행)
- [ ] Phase 11: 성과 피드백 루프 (조회수 기반 리라이트, 품질 점수)
