# A1 — 부록

[← 목차로](../README.md)

이 부록은 본문에 쓰인 약어·기술 용어를 풀어 둔 용어집과 참조 링크, 변경 이력을 담습니다. 본문에서 모르는 용어를 만나면 여기에서 찾으십시오.

## A1.1 용어집

### A1.1.1 에이전트·도구

- **에이전트(Agent)** — 사용자 요청에 대해 검색·도구 호출·종료를 스스로 판단하며 여러 단계를 잇는 워크로드. 경로가 입력에 따라 달라지는 점이 단일 호출·RAG와 다르다.
- **Agent Builder** — 모델 엔드포인트·지시문·지식베이스·도구·세션을 묶어 에이전트를 구성하고 채팅 완성 엔드포인트로 노출하는 PAIS 모듈.
- **MCP(Model Context Protocol)** — 모델·에이전트가 외부 도구·데이터에 접근하는 개방 프로토콜. PAIS는 호출·호스팅·등록 세 방향으로 지원한다.
- **Tool Gallery** — MCP 서버를 조직 차원에서 중앙 등록·관리하는 PAIS 2.1 신규 기능.
- **SSE(Server-Sent Events)** — 서버가 클라이언트로 이벤트를 흘려보내는 전송 방식. MCP 연결은 Streamable HTTP 또는 SSE를 요구한다.
- **휴먼인더루프(Human-in-the-loop)** — 되돌리기 어려운 행동 전에 사람의 확인을 두는 통제. PAIS 내장 기능은 확인되지 않아 애플리케이션 계층에서 설계한다.
- **가드레일(Guardrail)** — 에이전트의 입출력·행동을 제한하는 안전 통제. PAIS 맥락에서는 리소스 쿼터·거버넌스 의미로 주로 쓰이며, 콘텐츠 가드레일은 애플리케이션 계층 설계가 필요하다.
- **골든셋(Golden set)** — 대표 입력과 기대 동작을 묶은 평가용 데이터셋. 회귀·A/B 비교의 기준이 된다.
- **LLM 채점(LLM-as-judge)** — 별도 모델에 채점 기준을 주어 자유 서술 답의 품질을 점수화하는 평가 방법. 편향이 있어 사람 표본 검수로 보정한다.

### A1.1.2 모델·서빙

- **Model Runtime** — completion·embedding 모델을 추론 엔진으로 실행해 OpenAI 호환 엔드포인트로 노출하는 PAIS 모듈.
- **Model Gallery** — 모델 아티팩트의 중앙 저장소. Harbor(OCI 레지스트리) 기반.
- **Harbor** — OCI 호환 컨테이너 레지스트리. Model Gallery의 저장소 구현.
- **vLLM · llama.cpp · Infinity** — Model Runtime의 추론 엔진. 작성 시점 vLLM 0.11.2(생성·임베딩), llama.cpp b7739(CPU 추론), Infinity 0.0.76(임베딩 전용).
- **chat completion / OpenAI 호환 API** — `/chat/completions`·`/completions`·`/embeddings` 등 OpenAI 규약을 따르는 추론 API. 에이전트는 채팅 완성 엔드포인트로 노출된다.
- **Artifact Mirroring Tool** — 에어갭 환경에 모델·아티팩트를 미러링해 반입하는 PAIS 2.1 도구. `vcf pais` 하위 명령이 아니라 별도 스크립트로 수행한다.
- **NIM(NVIDIA Inference Microservices)** — NVIDIA가 제공하는 컨테이너형 추론 모델. Model Gallery로 반입해 관리할 수 있다.

### A1.1.3 데이터·검색

- **Data Indexing and Retrieval** — 데이터소스를 인덱싱해 지식베이스로 묶고 검색을 제공하는 PAIS 모듈.
- **지식베이스(Knowledge Base)** — 에이전트가 검색하는 인덱싱된 문서 집합.
- **pgvector** — 벡터 임베딩 저장·검색을 지원하는 PostgreSQL 확장. PAIS 지식베이스의 벡터 저장에 쓰인다.
- **임베딩(Embedding)** — 텍스트를 벡터로 변환한 표현. 인덱싱과 질의에 같은 임베딩 모델을 써야 검색이 일관된다.
- **Top-K** — 검색에서 가져올 상위 문서 수. 유사도 컷오프와 함께 검색 정밀도·잡음을 조절한다.
- **RAG(검색 증강 생성)** — 답하기 전 지식베이스를 검색해 컨텍스트로 넣는 고정 흐름. 상세는 시리즈 ④.
- **환각(Hallucination)** — 검색·도구 근거 없이 그럴듯하게 지어낸 답.

### A1.1.4 실행·운영

- **VKS(vSphere Kubernetes Service)** — Supervisor의 vSphere Namespace에 프로비저닝되는 쿠버네티스 클러스터. 모델 엔드포인트·에이전트 실행 기반.
- **VKr(vSphere Kubernetes release)** — VKS 클러스터에 쓰는, VMware가 서명·지원하는 쿠버네티스 배포 릴리스. 버전은 쿠버네티스 마이너 버전을 따른다(예: VKr 1.33 = 쿠버네티스 1.33 기반).
- **Supervisor / vSphere Namespace** — VKS·PAIS가 동작하는 vSphere 상의 쿠버네티스 제어·격리 단위.
- **DLVM(Deep Learning VM)** — 프로토타이핑·노트북용 딥러닝 VM 이미지. 프로덕션은 VKS를 쓴다.
- **GPU Operator** — VKS에서 GPU 드라이버·자원을 관리하는 쿠버네티스 오퍼레이터. PAIS 2.1 기준 25.10.1.
- **Observability** — 추론·GPU·에이전트 상호작용을 추적·관측하는 PAIS 모듈. 2.1에서 확장.
- **OpenTelemetry / LLM trace** — 표준 관측 프레임워크와 그 위의 LLM 추적. 사용자·모델·에이전트·지식베이스 상호작용을 따라간다.
- **TTFT(Time to First Token)** — 첫 토큰까지 걸린 시간. 체감 응답성 지표.
- **종단 지연(End-to-end latency)** — 요청부터 최종 응답까지 걸린 전체 시간.
- **SLO(Service Level Objective)** — 서비스가 지켜야 할 목표 수준(가용성·지연·오류율 등). 관측 기준선 위에 정해 이탈을 감시한다.
- **NVAIE(NVIDIA AI Enterprise)** — NVIDIA의 엔터프라이즈 AI 소프트웨어 라이선스. vGPU 사용 시 필요하며 패스스루 모드에서는 불필요하다.

## A1.2 참조 링크

작성 시점 1차 출처입니다. 기능·버전·동작은 릴리스마다 바뀌므로 적용 전 최신 공식 문서로 재확인하시기 바랍니다.

- [Private AI Services 릴리스 노트 (Broadcom TechDocs)](https://techdocs.broadcom.com/us/en/vmware-cis/private-ai/foundation-with-nvidia/9-0/private-ai-release-notes/vmware-private-ai-services-release-notes.html)
- [Private AI Services 상세 디자인 (VCF 9.1 디자인 라이브러리)](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/design/design-library/private-ai-platform-detailed-design/private-ai-services.html)
- [What is Private AI Services (Broadcom TechDocs)](https://techdocs.broadcom.com/us/en/vmware-cis/private-ai/foundation-with-nvidia/9-0/private-ai-foundation-9-x/what-is-private-ai-services.html)
- [RAG 애플리케이션용 에이전트 배포 (Broadcom TechDocs)](https://techdocs.broadcom.com/us/en/vmware-cis/private-ai/foundation-with-nvidia/9-0/private-ai-foundation-9-x/what-is-private-ai-services/deploy-an-agent-for-a-rag-application.html)
- [VCF CLI v9 — pais 명령 레퍼런스 (Broadcom TechDocs)](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-consumption/latest/consumer-interfaces-in-vcf/installing-and-using-vcf-cli-v9/command-reference2/pais2.html)
- [Streamline, Simplify and Protect all your AI workloads with VCF 9.1 (blogs.vmware.com)](https://blogs.vmware.com/cloud-foundation/2026/05/05/streamline-simplify-and-protect-all-your-ai-workloads-with-vcf-9-1/)
- [Model Gallery — JupyterLab Notebooks (blogs.vmware.com)](https://blogs.vmware.com/cloud-foundation/2026/02/26/model-gallery-how-to-use-jupyterlab-notebooks-to-simplify-model-deployment-and-management/)

시리즈 형제 가이드는 [시리즈 허브](https://github.com/JaeHoYun/vcf-private-ai-series)에서 모두 볼 수 있습니다.

## A1.3 변경 이력

| 버전 | 일자 | 내용 |
|------|------|------|
| v0.1 | 2026-06 | 초판 — PAIS 2.1 에이전트 서비스 가이드 신설. 개관·기초·설계 패턴·Agent Builder·MCP·모델 서빙·평가/가드레일·운영(00~07) + 부록(A1) |
| v0.1.1 | 2026-06 | 교정 — docs/06 문자 혼입('役'→'역할'), A1 용어집에 VKr 추가 |
| v0.2 | 2026-06 | 보강 — docs/00 §0.3에 '앱·외부 계층 책임 경계' 표 추가(PAIS·시리즈·앱 3계층 경계 명시) |
| v0.3 | 2026-06 | 보강 — docs/02 §2.6 신설(관리형 Agent Builder vs 자체 앱 직접 오케스트레이션 결정 분기) |
| v0.4 | 2026-06 | 보강 — docs/03 §3.9 신설(신원 두 층위: 서비스 인증 vs 최종 사용자 신원·권한 전파·테넌트 격리), 00 §0.3 신원 행 연결 |
| v0.5 | 2026-06 | 보강 — docs/04 §4.8(사내 MCP 서버 호스팅·운영)·§4.9(도구 호출 이그레스와 네트워크 경로) 신설 |
| v0.6 | 2026-06 | 보강 — docs/07 §7.7(백업·복구 stateful 자산 식별)·§7.8(스케일·알람·SLO·온콜) 신설, A1에 SLO 추가 |
| v0.7 | 2026-06 | 보강 — docs/06 §6.7 신설(평가 방법: 골든셋·LLM 채점·회귀 게이트·A/B·온라인 피드백 루프), A1에 골든셋·LLM 채점 추가 |
| v0.8 | 2026-06 | 보강 — docs/03 §3.10(엔드포인트 소비: 인증·스트리밍·재시도·멱등) 신설, docs/05 §5.3에 파인튜닝=PAIS 밖 경계 추가 |
| v0.9 | 2026-06 | 보강 — 형제 가이드 위임 링크를 repo 루트에서 해당 문서로 딥링크(⑤ ID·인증/네트워크·테넌트 격리/앱 가드레일, ⑥ GPU·메모리·VKS·용량·TCO, ① Day-2 운영 §10.x). 카탈로그·전체-가이드 포인터는 루트 유지, 07 트러블슈팅 참조의 누락 링크 추가 |
| v0.10 | 2026-06 | 보강 — docs/01 §1.5 '에이전트의 동작 루프'(인지→판단/계획→행동→관찰·ReAct·종료 조건) 신설, docs/06 §6.6 휴먼인더루프에 구현 패턴(제안→승인·대기 큐·드라이런) 추가. 곁들여 §6.7 헤더가 앞 문장과 한 줄에 붙어 있던 렌더링 버그 교정 |

---
[← 이전: 08 산업 유스케이스](../docs/08-use-cases.md) · [목차](../README.md)
