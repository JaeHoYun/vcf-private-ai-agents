# 05 — 모델과 서빙

[← 목차로](../README.md)

에이전트는 Model Runtime이 서빙하는 모델 위에서 동작합니다. 이 문서는 모델을 어떻게 서빙하고(엔진·엔드포인트), 어디에 보관하며(Model Gallery), 에어갭 환경에 어떻게 반입하고(AMT), CLI로 어떻게 다루는지를 다룹니다. 서빙 자체의 깊은 설계는 ③에 위임하고, 여기서는 **에이전트 관점에서 알아야 할 만큼**을 정리합니다.

> 본 문서의 수치·동작은 작성 시점(2026-06) VCF 9.1 / PAIF 9.1 / PAIS 2.1 기준이며, 적용 전 공식 문서로 재확인하시기 바랍니다.

---

## 5.1 Model Runtime — 서빙 계층

Model Runtime은 completion(생성)·embedding(임베딩) 모델을 추론 엔진으로 실행해 **OpenAI 호환 API**(`/chat/completions`·`/completions`·`/embeddings`)로 노출합니다. 엔드포인트는 ML API Gateway 뒤에 위치합니다.

- **에이전트와의 관계** — 에이전트는 이 completion 엔드포인트를 가져다 세션·검색·도구 호출을 얹어 씁니다([03 §3.2](03-agent-builder.md)). 모델 엔드포인트는 stateless이고, 에이전트가 그 위 stateful 계층입니다([01 §1.4](01-foundations.md)).
- **임베딩** — 지식베이스 인덱싱·질의에 쓰는 임베딩도 Model Runtime이 서빙합니다(상세는 [④](https://github.com/JaeHoYun/vcf-rag-reference-architecture)).
- **고가용성** — 모델 엔드포인트는 서로 다른 워커 노드에 복제본을 두는 구성(권장 복제본 2 이상)으로 가용성을 확보합니다. 단, 네임스페이스당 모델 엔드포인트 복제본 수에는 상한이 있습니다([07 §7.4](07-operations.md) 알려진 이슈).

## 5.2 서빙 엔진

PAIS 2.1 Model Runtime이 지원하는 추론 엔진과 버전입니다(작성 시점).

| 엔진 | 버전 | 용도 |
|------|------|------|
| vLLM | 0.11.2 | 생성 + 임베딩 |
| llama.cpp | b7739 | 생성 + 임베딩 (**CPU 추론**) |
| Infinity | 0.0.76 | 임베딩 전용 |

- **llama.cpp = 2.1 신규** — GPU 없이 CPU에서 추론하는 경로가 2.1에서 추가됐습니다. 작은 모델·저부하 보조 작업이나 GPU가 부족한 환경에서 선택지가 됩니다(성능·비용 trade-off는 [07](07-operations.md)·[⑥ TCO와 비용 모델](https://github.com/JaeHoYun/vcf-private-ai-sizing-cost/blob/main/docs/07-tco-cost-model.md)).
- **버전 주의** — 9.0/PAIS 2.0 계열의 vLLM 0.6.5·Infinity 0.0.43은 **옛 수치**입니다. 2.1 기준 위 버전으로 상향됐으므로 구버전 자료를 인용하지 마십시오.
- **엔진 버전 오버라이드** — 모델 엔드포인트 정의(YAML)의 `engineImage`로 엔진 이미지를 지정할 수 있습니다.

## 5.3 Model Gallery — 모델 저장소

**Model Gallery**는 모델 아티팩트의 중앙 저장소로, **Harbor**(OCI 호환 컨테이너 레지스트리)를 기반으로 Supervisor 서비스로 배포됩니다. 모델을 프로젝트·리포지토리 단위로 보관하며, 리포지토리별로 쓰기 권한을 관리합니다. (CLI·일부 인자에서는 내부적으로 `model-store`라는 용어도 함께 쓰입니다.)

모델 반입 경로:

- **NVIDIA NIM** — JupyterLab 노트북으로 NIM 모델을 Harbor로 내려받아 관리형 자산으로 만듭니다.
- **자체 모델** — `vcf pais models push`로 임의 모델 리비전을 저장소에 올립니다(§5.5).
- **Hugging Face 등** — 사설 레지스트리는 NIM 컨테이너·공개 모델 보관에 쓰일 수 있습니다. 구체 반입 절차는 공식 문서로 확인하시기 바랍니다.

> **경계 — 파인튜닝은 PAIS 밖** — Model Gallery는 모델을 **보관·반입**하지, 학습하지 않습니다. LoRA·전체 파인튜닝 같은 도메인 적응 학습은 PAIS 범위 밖이며, 외부 학습 환경(DLVM·전용 학습 파이프라인)에서 수행한 뒤 산출 모델을 `vcf pais models push`로 Gallery에 반입합니다(§5.5). 학습용 GPU·노드 사이징은 [⑥ GPU 사이징](https://github.com/JaeHoYun/vcf-private-ai-sizing-cost/blob/main/docs/02-gpu-sizing.md)·[①](https://github.com/JaeHoYun/vcf-private-ai-guide)에 위임합니다.

## 5.4 에어갭 반입 — AMT

PAIS 2.1은 **Artifact Mirroring Tool(AMT)** 로 에어갭(외부망 차단) 환경에 모델·아티팩트를 반입하는 경로를 도입했습니다. 인터넷에 연결된 준비 환경에서 컨테이너 이미지·Helm 차트·모델 파일을 미러링한 뒤, 격리망으로 옮겨 설치·운영합니다. 릴리스 노트는 NVIDIA GPU 모델 엔드포인트와 **에이전트를 포함한** 전체 Private AI 기능을 에어갭에서 운영할 수 있다고 명시합니다.

> **CLI 주의** — AMT는 별도 미러링 스크립트(`vpaifn-airgap-res-mirror.sh` 계열)로 수행하며, `vcf pais` 하위 명령이 아닙니다. 공식 CLI 레퍼런스에 `vcf pais amt` 같은 명령은 존재하지 않습니다(§5.5). 에어갭 절차는 릴리스마다 달라질 수 있으니 공식 문서·KB를 따르십시오.

## 5.5 CLI — `vcf pais models`

작성 시점 공식 CLI 레퍼런스에서 PAIS 관련 명령은 **`vcf pais models` 그룹 하나**이며, 하위 명령은 다음 여섯입니다.

| 명령 | 용도 |
|------|------|
| `vcf pais models list` | 모델 목록 |
| `vcf pais models list-revisions` | 모델 리비전 목록 |
| `vcf pais models pull` | 모델 내려받기 |
| `vcf pais models push` | 모델 올리기 |
| `vcf pais models delete` | 모델 삭제 |
| `vcf pais models delete-revision` | 리비전 삭제 |

예: `vcf pais models pull --modelStore <레지스트리>/<리포지토리> --modelName <모델> --tag <태그>`

`vcf pais amt`·`vcf pais agents` 같은 하위 명령은 레퍼런스에 없습니다. 에이전트·MCP·지식베이스 구성은 주로 UI(Agent Builder·VCF Automation)와 REST API로 다룹니다.

## 5.6 에이전트 관점의 모델 선택

에이전트가 쓸 모델을 고를 때의 기준을 모읍니다([03 §3.2](03-agent-builder.md)와 연결).

- **도구 호출 지원** — 도구를 적극적으로 쓰는 에이전트라면 네이티브 도구 호출 지원 모델을 우선합니다. 미지원 모델로도 구성은 가능하나 별도 설정이 필요합니다([02 §2.4](02-design-patterns.md)).
- **컨텍스트 길이** — 긴 대화 이력·검색 결과·도구 응답을 담을 만큼 충분해야 합니다.
- **비용·지연** — 큰 모델은 TTFT·토큰 비용이 큽니다. 보조 작업은 작은 모델(또는 CPU 추론)으로 분리하는 멀티 모델 구성을 고려하십시오.
- **임베딩 모델** — 지식베이스를 쓰면 별도 임베딩 모델이 필요합니다. 인덱싱과 질의에 같은 임베딩 모델을 써야 검색이 일관됩니다([④](https://github.com/JaeHoYun/vcf-rag-reference-architecture)).

다음 문서에서는 만든 에이전트를 운영에 올리기 전 **평가하고 가드레일을 설계**하는 방법을 다룹니다.

---
[← 이전: 04 MCP 도구 통합](04-mcp-tools.md) · [목차](../README.md) · [다음: 06 평가와 가드레일 →](06-evaluation-guardrails.md)
