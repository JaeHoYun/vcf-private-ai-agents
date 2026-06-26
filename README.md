# VCF 9.1 PAIS 에이전트 서비스 가이드

> **이 가이드를 읽기 전에** — 임베딩·벡터·토큰·RAG·쿠버네티스(VKS) 같은 용어가 낯설다면, 먼저 [VCF Private AI 입문 (Primer)](https://github.com/JaeHoYun/vcf-private-ai-foundations)에서 기초 어휘를 잡으시길 권합니다. 이 가이드는 그 개념들을 이미 아는 것으로 전제합니다.

> VMware Cloud Foundation(VCF) 9.1 Private AI Services(PAIS) 2.1 위에서 에이전트 서비스를 설계·구축·운영하는 실무 가이드 — Agent Builder · MCP 도구 · Model Runtime · Day-2 운영

[① 인프라](https://github.com/JaeHoYun/vcf-private-ai-guide) · [② VectorDB](https://github.com/JaeHoYun/vcf-dsm-vectordb-guide) · [③ 서빙 API](https://github.com/JaeHoYun/vcf-paif-serving-api-guide) · [④ RAG](https://github.com/JaeHoYun/vcf-rag-reference-architecture) · [⑤ 보안·거버넌스](https://github.com/JaeHoYun/vcf-private-ai-security-governance) · [⑥ 사이징·비용](https://github.com/JaeHoYun/vcf-private-ai-sizing-cost) · [⑦ 통합 설계](https://github.com/JaeHoYun/vcf-private-ai-design)는 Private AI 플랫폼의 인프라·검색·서빙·조립·보안·사이징·설계를 다룹니다. 그 위에서 한 걸음 남습니다 — **"이 플랫폼 위에서 추론하고 도구를 호출하는 에이전트를 어떻게 만들고 운영하나?"** 이 가이드가 그 답입니다.

④가 한 워크로드(사내 Q&A RAG)를 **조립**하는 방법이라면, 이 가이드는 그 위 계층 — 모델 응답에 더해 **지식베이스를 검색하고 사내 시스템 도구를 호출하며 여러 단계를 스스로 잇는 에이전트 워크로드**를 PAIS 2.1의 Agent Builder·MCP·Model Runtime으로 구현·운영하는 방법을 다룹니다. ⑦이 플랫폼을 설계한다면, 이 가이드는 그 플랫폼 위에 에이전트 서비스를 올립니다.

> **VCF Private AI 가이드 시리즈 — ⑧ PAIS 에이전트 서비스** · 8부작 중 한 편입니다. [전체 8개 보기 — 시리즈 허브](https://github.com/JaeHoYun/vcf-private-ai-series) · 상위 전략 [AX 방법론](https://github.com/JaeHoYun/enterprise-ax-methodology)

---

## 기반 버전 (Source of Truth)

> 본 가이드는 PAIS 2.1의 에이전트 기능 구현에 집중합니다. 광범위한 인프라 버전(vSphere·NSX·vSAN 등)은 단정하지 않고 형제 가이드의 버전 단일 기준 문서를 기준선으로 삼습니다 → [① README 버전표](https://github.com/JaeHoYun/vcf-private-ai-guide#기반-버전-source-of-truth). 모든 수치는 작성 시점(2026-06) 기준이며, 엔진·CLI·기능 동작은 릴리스마다 바뀌므로 적용 전 공식 문서로 재확인하시기 바랍니다.

| 구분 | 버전 | 비고 |
|------|------|------|
| VMware Cloud Foundation / PAIF | 9.1 | GA 2026-05 |
| Private AI Services (PAIS) | 2.1 | 6개 모듈 — Model Gallery · Model Runtime · Data Indexing and Retrieval · MCP Servers and Tool Gallery · Agent Builder · Observability |
| 서빙 엔진 (Model Runtime) | vLLM 0.11.2 · llama.cpp b7739 · Infinity 0.0.76 | llama.cpp(CPU 추론)·Infinity(임베딩)는 2.1 기준 |
| 실행 기반 (VKS) | VKr 1.33 · NVIDIA GPU Operator 25.10.1 | 모델 엔드포인트·에이전트 실행 |

## 이 가이드의 관점 — 조립이 아니라 에이전트 워크로드

에이전트는 "모델에게 한 번 묻고 한 번 답받는" 호출과 다릅니다. 에이전트는 **무엇을 검색할지, 어떤 도구를 부를지, 언제 멈출지를 스스로 정하는** 워크로드입니다. PAIS 2.1은 이 워크로드를 직접 다루는 모듈(Agent Builder·MCP·Observability)을 2.1에서 처음 정식 제공했습니다.

- **관리형 위에서 만든다** — Agent Builder는 모델 엔드포인트·지식베이스·도구·세션 정책을 묶어 에이전트를 구성하고, 채팅 완성(chat completion) 엔드포인트로 노출합니다. 직접 오케스트레이션 코드를 짜는 대신 구성으로 시작합니다.
- **도구는 MCP로 붙인다** — 사내 데이터베이스·이슈 트래커·협업 도구를 Model Context Protocol(MCP) 서버로 연결하고, 관리자가 승인한 도구만 에이전트에 노출합니다.
- **운영을 전제로 설계한다** — 토큰 처리량·첫 토큰까지 시간(TTFT)·종단 지연·에이전트 추적(trace)을 관측 지표로 두고, 업그레이드·다운타임·실패 모드를 미리 설계합니다.

## 문서 구성

| 순서 | 문서 | 내용 |
|------|------|------|
| 00 | [개관](docs/00-orientation.md) | 이 가이드의 역할·독자·선행지식, 다루는 것과 다루지 않는 것, PAIS 6개 모듈 지도 |
| 01 | [에이전트 기초와 PAIS 2.1 지형](docs/01-foundations.md) | 에이전트 vs RAG vs 워크플로우 경계, 6개 모듈, 에이전트의 구성요소 |
| 02 | [에이전트 설계 패턴](docs/02-design-patterns.md) | 단일·멀티 에이전트, 도구·지식 연결, 세션 관리, 언제 에이전트로 푸나 |
| 03 | [Agent Builder로 구축](docs/03-agent-builder.md) | 에이전트 생성, 모델 엔드포인트·지시문·지식베이스·도구·세션, Playground, REST API |
| 04 | [MCP 도구 통합](docs/04-mcp-tools.md) | MCP 3방향(호출·호스팅·등록), Tool Gallery, 전송·인증, 보안 경계 |
| 05 | [모델과 서빙](docs/05-models-serving.md) | Model Runtime, OpenAI 호환 API, 서빙 엔진, Model Gallery, 에어갭(AMT), CLI |
| 06 | [평가와 가드레일](docs/06-evaluation-guardrails.md) | Playground·CI/CD 테스트, 실패 모드, 가드레일·휴먼인더루프 경계 |
| 07 | [운영과 Day-2](docs/07-operations.md) | 배포 토폴로지, 관측성, 업그레이드·다운타임, 알려진 이슈, 비용 |
| 08 | [산업 유스케이스](docs/08-use-cases.md) | 결과·KPI로 보는 에이전트 — 제조 사례(기술문서 Q&A·품질 리포트·설비 보전) + 확장 템플릿 |
| A1 | [부록](appendix/A1-reference.md) | 용어집, 참조 링크, 변경 이력 |

## 빠른 시작

- **"처음 본다"** → [00 개관](docs/00-orientation.md) → [01 에이전트 기초](docs/01-foundations.md)
- **"바로 하나 만들어 본다"** → [03 Agent Builder로 구축](docs/03-agent-builder.md)
- **"사내 시스템을 도구로 붙인다"** → [04 MCP 도구 통합](docs/04-mcp-tools.md)
- **"어떤 모델을 어떻게 서빙하나"** → [05 모델과 서빙](docs/05-models-serving.md)
- **"운영에 올리기 전 점검한다"** → [06 평가와 가드레일](docs/06-evaluation-guardrails.md) · [07 운영과 Day-2](docs/07-operations.md)
- **"이걸로 무슨 가치를 내나"** → [08 산업 유스케이스](docs/08-use-cases.md)

## 라이선스

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). 자유롭게 활용하시되 출처를 표기해 주세요. `출처: https://github.com/JaeHoYun/vcf-private-ai-agents`

## 면책

**비공식 문서** — Broadcom·NVIDIA 등 벤더의 공식 입장을 대변하지 않습니다. 본문의 구성값·버전·동작은 작성 시점 기준 **예시**이며, 에이전트·MCP·서빙 기능은 릴리스마다 바뀝니다. 적용 전 반드시 공식 문서로 확인하시기 바랍니다. 언급된 제품명·상표는 각 소유자의 자산입니다.
