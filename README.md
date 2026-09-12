# VCF Private AI 앱과 에이전트 서비스 가이드

> **이 가이드를 읽기 전에** — 임베딩, 벡터, 토큰, RAG, 쿠버네티스(VKS) 같은 용어가 낯설다면, 먼저 [VCF Private AI 입문 (Primer)](https://github.com/JaeHoYun/vcf-private-ai/tree/main/00-foundations)에서 기초 어휘를 잡으시길 권합니다. 이 가이드는 그 개념들을 이미 아는 것으로 전제합니다.

> VMware Cloud Foundation(VCF) 9.1.x Private AI Services(PAIS) 3.0 위에서 기업이 LLM 앱, RAG 앱, 배치 파이프라인, 에이전트 서비스를 **기획, 설계, 구축, 검증, 운영**하는 실행 계층 가이드

[① 인프라](https://github.com/JaeHoYun/vcf-private-ai/tree/main/01-infra), [② VectorDB](https://github.com/JaeHoYun/vcf-private-ai/tree/main/02-vectordb), [③ 서빙 API](https://github.com/JaeHoYun/vcf-private-ai/tree/main/03-serving-api), [④ RAG](https://github.com/JaeHoYun/vcf-private-ai/tree/main/04-rag), [⑤ 보안과 거버넌스](https://github.com/JaeHoYun/vcf-private-ai/tree/main/05-security), [⑥ 사이징과 비용](https://github.com/JaeHoYun/vcf-private-ai/tree/main/06-sizing-cost), [⑦ 통합 설계](https://github.com/JaeHoYun/vcf-private-ai/tree/main/07-design)는 Private AI **플랫폼**을 세우고 보호하고 산정하고 설계합니다. 그 위에서 아직 답하지 않은 질문이 있습니다 — **"이 플랫폼 위에서 실제 서비스 하나를, 기획부터 운영까지 어떻게 만들고 책임지나?"** 이 가이드가 그 답입니다.

플랫폼(시리즈 ①~⑦)이 인프라팀과 플랫폼팀의 관점이라면, 이 가이드는 그 플랫폼을 **소비하는 쪽** — 서비스를 기획하는 담당자, 설계하는 아키텍트, 만드는 앱 팀, 출시를 심사하는 보안과 법무, 운영하는 팀 — 의 관점입니다. 에이전트는 이 가이드가 다루는 네 가지 서비스 유형(챗과 Q&A RAG, 문서 처리 배치, 기존 시스템에 심는 코파일럿, 에이전트) 중 하나이며, PAIS 3.0의 Agent Builder, MCP, Model Runtime으로 구현하는 방법은 구축 편(08~10)에서 그대로 다룹니다.

> **VCF Private AI 가이드 시리즈 위에 올리는 실행 계층 가이드**입니다. 시리즈 본편(①~⑦)은 [시리즈 허브](https://github.com/JaeHoYun/vcf-private-ai)에서, 상위 전략은 [AX 방법론](https://github.com/JaeHoYun/enterprise-ax-methodology)에서 다룹니다. 프로필의 **AX(전략) → Private AI(인프라) → 앱과 에이전트(실행)** 3단계 중 실행 편입니다. 에이전트 전용 9편 체제였던 이전 구조는 태그 [`baseline-agents-v1`](https://github.com/JaeHoYun/vcf-private-ai-apps/tree/baseline-agents-v1)에서 그대로 읽을 수 있습니다.

---

## 기반 버전 (Source of Truth)

> 본 가이드는 PAIS 3.0 위의 서비스 구현에 집중합니다. 광범위한 인프라 버전(vSphere, NSX, vSAN 등)은 단정하지 않고 형제 가이드의 버전 단일 기준 문서를 기준선으로 삼습니다 → [① README 버전표](https://github.com/JaeHoYun/vcf-private-ai/tree/main/01-infra#기반-버전-source-of-truth). 모든 수치는 작성 시점(2026-06) 기준이고 2026-09에 VCF 9.1.1 / PAIS 3.0 GA(2026-09-03) 내용을 반영했으며, 엔진, CLI, 기능 동작은 릴리스마다 바뀌므로 적용 전 공식 문서로 재확인하시기 바랍니다.

| 구분 | 버전 | 비고 |
|------|------|------|
| VMware Cloud Foundation / PAIF | 9.1.1 | 9.1 GA 2026-05, 9.1.1 GA 2026-09. PAIS 3.0은 VCF 9.1.x 호환 |
| Private AI Services (PAIS) | 3.0 | 6개 모듈 — Model Gallery, Model Runtime, Data Indexing and Retrieval, MCP Servers and Tool Gallery, Agent Builder, Observability. 3.0에서 공유 모델 호스팅, 원격 클라우드 모델, API 토큰, 지식베이스 복제 추가 |
| 서빙 엔진 (Model Runtime) | vLLM 0.20.0, llama.cpp b9309, Infinity 0.0.76 | vLLM 0.20.0은 CUDA 13.0 기본(드라이버 580 이상). 2.1은 vLLM 0.11.2, llama.cpp b7739 |
| 실행 기반 (VKS) | VKr 1.34, ClusterClass builtin-generic-v3.5.0, NVIDIA GPU Operator 25.10.1(기본) 또는 26.3.1 | 모델 엔드포인트와 에이전트 실행. 2.1은 VKr 1.33, v3.2.0 |

> **2.1 환경을 운영 중이라면** — 본문에서 "PAIS 3.0부터"로 표기한 대목만 건너뛰면 됩니다. 기능별 도입 버전은 [① 00 What's New의 버전별 기능 이력](https://github.com/JaeHoYun/vcf-private-ai/blob/main/01-infra/docs/00-whats-new.md#08-버전별-기능-이력-pais-2089--21--30)에, 2.1 기준으로 쓰인 2026-06 시점 문서 전체는 태그 [`baseline-pais-2.1`](https://github.com/JaeHoYun/vcf-private-ai-apps/tree/baseline-pais-2.1)에 있습니다.

## 이 가이드의 관점 — 서비스 수명주기

플랫폼 편이 "무엇이 있는가"를 계층별로 설명한다면, 이 가이드는 서비스 하나가 태어나서 퇴역할 때까지의 순서를 따릅니다. 다섯 부로 나뉩니다.

- **기획과 선정** — 어떤 일에 써야 성과가 나는지, 위험 등급은 어디에 놓이는지, 어느 서비스 유형으로 풀지 정합니다.
- **설계** — 사용자 신원이 어디까지 따라가는지, 플랫폼의 게이트웨이와 쿼터를 어떻게 소비하는지, 어떤 데이터 소스를 어떤 조건으로 들여오는지, 사내 시스템에 쓰기까지 허용할지 설계합니다.
- **구축** — Agent Builder, MCP 도구, 모델 서빙으로 구현하고 앱에 통합합니다.
- **검증과 출시** — 서비스 단위 보안 준비, 평가 게이트, 출시 심사를 통과합니다.
- **운영과 종료** — 관측, 업그레이드, 비용, 그리고 서비스 퇴역까지 책임집니다.

각 부의 문서는 플랫폼 편(①~⑦)의 해당 절로 딥링크합니다. 플랫폼이 제공하는 통제와 사실 관계는 그쪽이 정본이고, 이 가이드는 그것을 서비스 하나에 적용하는 순서와 판단을 다룹니다.

## 문서 구성

| 부 | 순서 | 문서 | 내용 |
|----|------|------|------|
| 기획과 선정 | 00 | [개관](docs/00-orientation.md) | 이 가이드의 역할, 독자, 선행지식, 다루는 것과 다루지 않는 것, PAIS 6개 모듈 지도 |
| | 01 | [서비스 유형과 PAIS 3.0 지형](docs/01-foundations.md) | 단일 호출, RAG, 워크플로우, 에이전트의 경계, 네 가지 서비스 유형, 6개 모듈, 에이전트의 구성요소 |
| | 02 | [어디에 쓰나](docs/02-use-cases.md) | 성과가 나는 일과 실패하는 일 — 파일럿이 멈추는 다섯 실패 유형, 유스케이스 선별 기준, 사례 2종, 위험 등급 판정, 자율성 상한 |
| 설계 | 03 | [에이전트 설계 패턴](docs/03-design-patterns.md) | 언제 에이전트로 푸나, 단일과 멀티, 도구와 지식 연결, 세션, 배치 파이프라인, 롱컨텍스트 대 RAG, 4-Tier 골격 |
| | 04 | [사용자 신원과 권한 전파](docs/04-identity-propagation.md) | 두 층위의 신원, 경계마다 끊기는 자리, 세 가지 전파 패턴, 토큰 교환, MCP 도구 단의 신원, 지식베이스 권한 일치, 감사 필드 |
| | 07 | [사내 시스템 연동과 쓰기 설계](docs/07-integration-write-design.md) | API와 MCP와 RPA 선택, 작업 분류표, 승인 게이트와 승인 큐, 멱등 키와 보상, 다운스트림 보호, 도구 스키마 계약 |
| 구축 | 08 | [Agent Builder로 구축](docs/08-agent-builder.md) | 에이전트 생성, 모델 엔드포인트, 지시문, 지식베이스, 도구, 세션, Playground, REST API |
| | 09 | [MCP 도구 통합](docs/09-mcp-tools.md) | MCP 3방향(호출, 호스팅, 등록), Tool Gallery, 전송과 인증, 보안 경계 |
| | 10 | [모델과 서빙](docs/10-models-serving.md) | Model Runtime, OpenAI 호환 API, 서빙 엔진, Model Gallery, 에어갭(Artifact Mirroring Tool), CLI |
| 검증과 출시 | 13 | [평가와 가드레일](docs/13-evaluation-guardrails.md) | Playground, CI/CD 테스트, 실패 유형, 가드레일과 휴먼인더루프 경계 |
| 운영과 종료 | 14 | [운영과 Day-2](docs/14-operations.md) | 배포 토폴로지, 관측성, 업그레이드와 다운타임, 알려진 이슈, 비용 |
| 부록 | A1 | [부록](appendix/A1-reference.md) | 용어집, 참조 링크 |
| | A2 | [워크시트](appendix/A2-worksheets.md) | 유스케이스 선별 워크시트, 운영 투입 전 점검표, MCP 서버 등록 체크리스트 |

> 번호가 비어 있는 자리(05, 06, 11, 12)는 설계 편의 플랫폼 소비(게이트웨이, 온보딩, 토큰 예산)와 데이터 소스 온보딩과 보호 문서, 구축 편의 앱 통합과 신뢰 UX, 검증 편의 서비스 보안 준비가 들어갈 자리이며 순차로 채웁니다. 번호는 서비스 수명주기 순서를 따르므로 그 사이의 문서가 추가되어도 기존 번호는 바뀌지 않습니다.

## 빠른 시작

- **"처음 본다"** → [00 개관](docs/00-orientation.md) → [01 서비스 유형과 PAIS 3.0 지형](docs/01-foundations.md)
- **"이걸로 무슨 가치를 내나, 어디서 실패하나"** → [02 어디에 쓰나](docs/02-use-cases.md)
- **"사용자 권한이 도구와 검색까지 따라가게 하려면"** → [04 사용자 신원과 권한 전파](docs/04-identity-propagation.md)
- **"사내 시스템에 쓰기까지 시키려면"** → [07 사내 시스템 연동과 쓰기 설계](docs/07-integration-write-design.md)
- **"바로 하나 만들어 본다"** → [08 Agent Builder로 구축](docs/08-agent-builder.md)
- **"사내 시스템을 도구로 붙인다"** → [09 MCP 도구 통합](docs/09-mcp-tools.md)
- **"어떤 모델을 어떻게 서빙하나"** → [10 모델과 서빙](docs/10-models-serving.md)
- **"운영에 올리기 전 점검한다"** → [13 평가와 가드레일](docs/13-evaluation-guardrails.md) | [14 운영과 Day-2](docs/14-operations.md)

## 라이선스

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). 자유롭게 활용하시되 출처를 표기해 주세요. `출처: https://github.com/JaeHoYun/vcf-private-ai-apps`

## 면책

**비공식 문서** — Broadcom, NVIDIA 등 벤더의 공식 입장을 대변하지 않습니다. 본문의 구성값, 버전, 동작은 작성 시점 기준 **예시**이며, 에이전트, MCP, 서빙 기능은 릴리스마다 바뀝니다. 적용 전 반드시 공식 문서로 확인하시기 바랍니다. 언급된 제품명과 상표는 각 소유자의 자산입니다.
