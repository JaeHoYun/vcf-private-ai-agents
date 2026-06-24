# 00 — 개관

[← 목차로](../README.md)

이 가이드는 VCF 9.1 Private AI Services(PAIS) 2.1 위에서 **에이전트 서비스를 만들고 운영하는 방법**을 다룹니다. 이 문서는 가이드 전체의 지도입니다 — 무엇을 답하는지, 누구를 위한 것인지, 무엇을 다루고 무엇은 다른 편에 위임하는지, 그리고 PAIS의 어떤 구성요소를 어디서 설명하는지를 먼저 정리합니다.

> 본 문서의 수치·동작은 작성 시점(2026-06) VCF 9.1 / PAIF 9.1 / PAIS 2.1 기준이며, 적용 전 공식 문서로 재확인하시기 바랍니다.

---

## 0.1 이 가이드가 답하는 질문

시리즈 ①~⑦은 Private AI 플랫폼을 세우고(인프라·검색·서빙·보안·사이징) 한 워크로드를 조립(④ RAG)하고 전체를 설계(⑦)하는 데까지 옵니다. 남는 질문은 하나입니다.

> **"이 플랫폼 위에서 스스로 검색하고 도구를 호출하며 여러 단계를 잇는 에이전트를, 어떻게 구현하고 운영하나?"**

에이전트는 단일 호출과 다릅니다. 사용자의 한 요청에 대해 에이전트는 (1) 지식베이스를 검색할지 판단하고 (2) 필요하면 사내 시스템 도구를 호출하며 (3) 그 결과로 다음 행동을 정하고 (4) 충분하면 답을 종합합니다. PAIS 2.1은 이 흐름을 직접 다루는 모듈 — **Agent Builder · MCP Servers and Tool Gallery · Observability** — 을 2.1에서 정식 제공했습니다. 이 가이드는 그 모듈로 에이전트를 구성·통합·서빙·운영하는 실무를 순서대로 안내합니다.

## 0.2 독자와 선행지식

**대상 독자**

- Private AI 플랫폼 위에 에이전트·코파일럿·자동화 서비스를 올리려는 **AI 서비스 개발자·아키텍트**
- 에이전트 워크로드를 배포·관측·업그레이드해야 하는 **플랫폼 운영자**

**선행지식** — 이 가이드는 플랫폼이 이미 서 있다고 가정합니다. 다음을 먼저 보면 수월합니다.

| 알아둘 것 | 모르면 | 
|-----------|--------|
| 인프라·VKS·GPU 워크로드 도메인 | 시리즈 [① 인프라](https://github.com/JaeHoYun/vcf-private-ai-guide) |
| 모델 서빙·OpenAI 호환 엔드포인트 | 시리즈 [③ 서빙 API](https://github.com/JaeHoYun/vcf-paif-serving-api-guide) |
| RAG·지식베이스·벡터 검색 | 시리즈 [④ RAG](https://github.com/JaeHoYun/vcf-rag-reference-architecture) |
| 에이전트 일반 개념(도구 호출·계획) | [01 에이전트 기초](01-foundations.md)에서 짚습니다 |

## 0.3 다루는 것과 다루지 않는 것

**다루는 것**

- PAIS 2.1 Agent Builder로 에이전트를 구성하는 절차(모델 엔드포인트·지시문·지식베이스·도구·세션)
- MCP로 사내·외부 시스템 도구를 연결하고 승인·관리하는 방법
- 에이전트가 쓰는 모델을 Model Runtime으로 서빙하고 Model Gallery로 관리하는 방법
- 에이전트 워크로드의 평가·가드레일 한계와 운영(배포·관측·업그레이드·비용)

**다루지 않는 것 (위임)**

| 주제 | 위임 |
|------|------|
| 인프라 구축·Day-2 플랫폼 운영 | [① 인프라](https://github.com/JaeHoYun/vcf-private-ai-guide) |
| 지식베이스·RAG 파이프라인 상세(청크·임베딩·재순위) | [④ RAG](https://github.com/JaeHoYun/vcf-rag-reference-architecture) |
| 보안 통제·격리·접근·감사 상세 | [⑤ 보안·거버넌스](https://github.com/JaeHoYun/vcf-private-ai-security-governance) |
| GPU·노드 사이징·TCO | [⑥ 사이징·비용](https://github.com/JaeHoYun/vcf-private-ai-sizing-cost) |
| 플랫폼 전체 설계 결정·블루프린트 | [⑦ 통합 설계](https://github.com/JaeHoYun/vcf-private-ai-design) |

벤더중립 에이전트 설계 이론(추론·계획·평가 방법론 일반)은 이 가이드의 범위 밖이며, 필요한 만큼만 [01](01-foundations.md)·[02](02-design-patterns.md)에서 짚고 PAIS 구현으로 곧장 잇습니다.

## 0.4 PAIS 6개 모듈 지도

PAIS 2.1은 여섯 모듈로 이뤄집니다. 에이전트는 이 중 여러 모듈을 함께 씁니다.

| 모듈 | 역할 | 이 가이드 |
|------|------|-----------|
| Model Gallery | 모델 아티팩트 저장소(Harbor 기반 OCI 레지스트리) | [05](05-models-serving.md) |
| Model Runtime | 추론·임베딩 모델 서빙(OpenAI 호환 엔드포인트) | [05](05-models-serving.md) |
| Data Indexing and Retrieval | 지식베이스 인덱싱·검색(pgvector) | [03](03-agent-builder.md) 연결 · ④ 위임 |
| MCP Servers and Tool Gallery | 외부 도구를 MCP로 연결·중앙 관리(2.1 신규) | [04](04-mcp-tools.md) |
| Agent Builder | 모델·지식·도구·세션을 묶어 에이전트 구성 | [03](03-agent-builder.md) |
| Observability | 추론·GPU·에이전트 상호작용 추적·관측(2.1 확장) | [07](07-operations.md) |

세 모듈(MCP·Agent Builder·확장된 Observability)이 2.0 대비 2.1의 에이전트 이야기를 만듭니다. 자세한 모듈별 역할과 에이전트 구성요소는 [01](01-foundations.md)에서 풀어 설명합니다.

## 0.5 읽는 순서

- 개념부터 잡으려면 **00 → 01 → 02**를 차례로 읽으십시오.
- 손으로 먼저 만들어 보려면 **[03 Agent Builder](03-agent-builder.md)** 로 건너뛰고, 도구 연결이 필요할 때 **[04 MCP](04-mcp-tools.md)**, 모델 선택이 필요할 때 **[05 모델과 서빙](05-models-serving.md)** 으로 돌아오십시오.
- 운영 준비 단계라면 **[06 평가와 가드레일](06-evaluation-guardrails.md) → [07 운영과 Day-2](07-operations.md)** 가 핵심입니다.

---
[목차](../README.md) · [다음: 01 에이전트 기초와 PAIS 2.1 지형 →](01-foundations.md)
