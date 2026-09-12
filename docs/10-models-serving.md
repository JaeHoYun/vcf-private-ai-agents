# 10 — 모델과 서빙

[← 목차로](../README.md)

에이전트는 Model Runtime이 서빙하는 모델 위에서 동작합니다. 이 문서는 모델을 어떻게 서빙하고(엔진과 엔드포인트), 어디에 보관하며(Model Gallery), 에어갭 환경에 어떻게 반입하고(Artifact Mirroring Tool), CLI로 어떻게 다루는지를 다룹니다. 서빙 자체의 깊은 설계는 ③에 위임하고, 여기서는 **에이전트 관점에서 알아야 할 만큼**을 정리합니다.

> 본 문서의 수치와 동작은 VCF 9.1.1 / PAIF 9.1.1 / PAIS 3.0 기준입니다(작성 2026-06, 9.1.1과 3.0 GA 반영 2026-09). 2.1 환경에서는 "PAIS 3.0부터"로 표기한 대목만 건너뛰면 됩니다. 적용 전 최신 공식 문서로 재확인하시기 바랍니다.

---

## 10.1 Model Runtime — 서빙 계층

Model Runtime은 completion(생성)과 embedding(임베딩) 모델을 추론 엔진으로 실행해 **OpenAI 호환 API**(`/chat/completions`, `/completions`, `/embeddings`)로 노출합니다. 엔드포인트는 ML API Gateway 뒤에 위치하며, 기본 경로는 `https://<PAIS FQDN>/api/v1/compatibility/openai/v1/...`입니다([근거: Private AI Services API](https://developer.broadcom.com/xapis/vmware-private-ai-service-api/latest/), 호출 예시는 [08 8.8절, 8.10절](08-agent-builder.md)).

- **에이전트와의 관계** — 에이전트는 이 completion 엔드포인트를 가져다 세션, 검색, 도구 호출을 더해 씁니다([08 8.2절](08-agent-builder.md)). 모델 엔드포인트는 stateless이고, 에이전트가 그 위 stateful 계층입니다([01 1.4절](01-foundations.md)).
- **임베딩** — 지식베이스 인덱싱과 질의에 쓰는 임베딩도 Model Runtime이 서빙합니다(상세는 [④](https://github.com/JaeHoYun/vcf-private-ai/tree/main/04-rag)).
- **고가용성** — 모델 엔드포인트는 서로 다른 워커 노드에 복제본을 두는 구성(권장 복제본 2 이상)으로 가용성을 확보합니다. 단, 네임스페이스당 모델 엔드포인트 복제본 수에는 상한이 있습니다([14 14.4절](14-operations.md) 알려진 이슈).
- **모델이 도는 자리 세 가지(PAIS 3.0부터)** — 에이전트가 고르는 completion 엔드포인트는 이제 세 종류 중 하나입니다. 이 네임스페이스의 GPU에서 도는 **로컬 모델**, 다른 PAIS 인스턴스나 네임스페이스(provider)가 서빙하는 **공유 모델**, Google Gemini나 OpenAI 호환 서비스에 연결된 **원격 클라우드 모델**입니다. Agent Builder 드롭다운에서는 셋이 같은 엔드포인트로 보이지만, 공유 모델은 provider 쪽 레플리카와 쿼터가 응답 지연을 정하고, 원격 모델은 프롬프트가 사외로 나가므로 지식베이스 검색 결과를 컨텍스트로 넣는 에이전트라면 어떤 데이터가 외부로 가는지 먼저 정해야 합니다([⑤ 데이터 거버넌스](https://github.com/JaeHoYun/vcf-private-ai/blob/main/05-security/docs/05-data-governance.md)). 연결 구조는 [③ 02 2.5.1절](https://github.com/JaeHoYun/vcf-private-ai/blob/main/03-serving-api/docs/02-serving-api-architecture.md)에 있습니다.

## 10.2 서빙 엔진

PAIS 3.0 Model Runtime이 지원하는 추론 엔진과 버전입니다([근거: PAIS 릴리스 노트](https://techdocs.broadcom.com/us/en/vmware-cis/private-ai/foundation-with-nvidia/9-1/private-ai-release-notes/vmware-private-ai-services-release-notes.html)).

| 엔진 | 3.0 | 2.1 | 용도 |
|------|-----|-----|------|
| vLLM | 0.20.0 | 0.11.2 | 생성 + 임베딩. 0.20.0은 CUDA 13.0이 기본이라 GPU 드라이버 580 이상이 필요 |
| llama.cpp | b9309 | b7739 | 생성 + 임베딩 (**CPU 추론**) |
| Infinity | 0.0.76 | 0.0.76 | 임베딩 전용 |

- **llama.cpp = 2.1 신규** — GPU 없이 CPU에서 추론하는 경로가 2.1에서 추가됐습니다. 작은 모델, 저부하 보조 작업이나 GPU가 부족한 환경에서 선택지가 됩니다(성능과 비용 트레이드오프는 [14](14-operations.md), [⑥ TCO와 비용 모델](https://github.com/JaeHoYun/vcf-private-ai/blob/main/06-sizing-cost/docs/07-tco-cost-model.md)). CPU 추론에서 MCP 도구를 함께 쓸 때 reasoning 모델이 타임아웃되던 문제는 2.1.2에서 수정됐습니다.
- **버전 주의** — 9.0/PAIS 2.0 계열의 vLLM 0.6.5, Infinity 0.0.43은 **이전 버전(2.0 계열) 기준 수치**입니다. 3.0으로 올라갈 때 vLLM이 0.11.2에서 0.20.0으로 크게 올라가므로, 모델 엔드포인트의 VRAM 요구량과 양자화 포맷 지원을 다시 확인하십시오.
- **엔진 버전 오버라이드** — 모델 엔드포인트 정의(YAML)의 `engineImage`로 엔진 이미지를 지정할 수 있습니다.
- **버전 정본** — 엔진 버전의 단일 기준은 [README 기반 버전표](../README.md#기반-버전-source-of-truth)입니다. 본문과 용어집의 버전 표기는 그 요약이며, 갱신은 README 표를 기준으로 맞춥니다.

## 10.3 Model Gallery — 모델 저장소

**Model Gallery**는 모델 아티팩트의 중앙 저장소로, **Harbor**(OCI 호환 컨테이너 레지스트리)를 기반으로 Supervisor 서비스로 배포됩니다([근거: Private AI Services 상세 디자인](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/design/design-library/private-ai-platform-detailed-design/private-ai-services.html)). 모델을 프로젝트와 리포지토리 단위로 보관하며, 리포지토리별로 쓰기 권한을 관리합니다. (CLI, 일부 인자에서는 내부적으로 `model-store`라는 용어도 함께 쓰입니다.)

모델 반입 경로:

- **NVIDIA NIM** — JupyterLab 노트북으로 NIM 모델을 Harbor로 내려받아 관리형 자산으로 만듭니다.
- **자체 모델** — `vcf pais models push`로 임의 모델 리비전을 저장소에 올립니다(10.5절).
- **Hugging Face 등 공개 모델** — 사설 레지스트리에 보관해 반입할 수 있습니다. 구체 절차는 공식 문서로 확인하시기 바랍니다.

> **경계 — 파인튜닝은 PAIS 밖** — Model Gallery는 모델을 **보관하고 반입**하지, 학습하지 않습니다. LoRA, 전체 파인튜닝 같은 도메인 적응 학습은 PAIS 범위 밖이며, 외부 학습 환경(DLVM, 전용 학습 파이프라인)에서 수행한 뒤 산출 모델을 `vcf pais models push`로 Gallery에 반입합니다(10.5절). 학습용 GPU, 노드 사이징은 [⑥ GPU 사이징](https://github.com/JaeHoYun/vcf-private-ai/blob/main/06-sizing-cost/docs/02-gpu-sizing.md), [①](https://github.com/JaeHoYun/vcf-private-ai/tree/main/01-infra)에 위임합니다.

## 10.4 에어갭 반입 — Artifact Mirroring Tool

PAIS 2.1은 **Artifact Mirroring Tool** 로 에어갭(외부망 차단) 환경에 모델과 아티팩트를 반입하는 경로를 도입했습니다. 인터넷에 연결된 준비 환경에서 컨테이너 이미지, Helm 차트, 모델 파일을 미러링한 뒤, 격리망으로 옮겨 설치하고 운영합니다. [PAIS 릴리스 노트](https://techdocs.broadcom.com/us/en/vmware-cis/private-ai/foundation-with-nvidia/9-0/private-ai-release-notes/vmware-private-ai-services-release-notes.html)는 NVIDIA GPU 모델 엔드포인트와 **에이전트를 포함한** 전체 Private AI 기능을 에어갭에서 운영할 수 있다고 명시합니다.

> **CLI 주의** — Artifact Mirroring Tool은 pais CLI 플러그인의 **`vcf pais amt pull` / `vcf pais amt push`** 명령으로 수행합니다(`vcf plugin install pais`로 설치). 단 이 `vcf pais amt` 명령은 **VCF CLI 명령 레퍼런스 페이지에는 누락**되어 있고(거기에는 `vcf pais models`만 표기, 10.5절), PAIF **Disconnected Environment 배포 문서**에만 명시돼 있으니 해당 문서를 1차 근거로 삼으십시오. 에어갭 절차는 릴리스마다 달라질 수 있으니 적용 직전 공식 문서와 KB로 재확인하십시오. ([근거: Upload the Private AI Services Components to a Disconnected Environment](https://techdocs.broadcom.com/us/en/vmware-cis/private-ai/foundation-with-nvidia/9-0/private-ai-foundation-9-x/deploying-private-ai-foundation-with-nvidia/installing-and-configuring-private-ai-services/upload-the-private-ai-services-components-to-a-disconnected-environment.html))

## 10.5 CLI — `vcf pais models`

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

위 표는 **VCF CLI 명령 레퍼런스 페이지** 기준입니다. 에어갭 반입용 `vcf pais amt`(pull/push)는 이 레퍼런스에 빠져 있으나 실재하는 명령입니다 — 근거와 주의는 10.4절에 정리했습니다. `vcf pais agents` 같은 하위 명령은 없으며, 에이전트, MCP, 지식베이스 구성은 주로 UI(Agent Builder, VCF Automation)와 REST API로 다룹니다.

CLI의 형태에 대해 한 가지 정리해 둡니다. 단독 실행 파일 형태의 `pais` CLI는 DLVM 9.1 이미지에서 제거됐고, 지금 쓰는 것은 VCF Consumption CLI의 `pais` 플러그인입니다. DLVM 9.1.1 이미지에는 VCF CLI 9.1.0과 확장된 플러그인, helm, kubectl vSphere 플러그인이 함께 들어 있습니다. PAIS 3.0부터는 이 CLI로 PAIS 관리 클러스터의 kubeconfig를 받는 절차와 지원 번들 수집이 간단해졌습니다. VCF Automation 네임스페이스에서 CLI 명령을 실행하려면 3.0부터 API 토큰이 필요합니다([08 8.9절](08-agent-builder.md)).

## 10.6 에이전트 관점의 모델 선택

에이전트가 쓸 모델을 고를 때의 기준을 한데 정리합니다([08 8.2절](08-agent-builder.md)와 연결).

- **도구 호출 지원** — 도구를 적극적으로 쓰는 에이전트라면 네이티브 도구 호출 지원 모델을 우선합니다. 미지원 모델로도 구성은 가능하나 별도 설정이 필요합니다([03 3.4절](03-design-patterns.md)).
- **컨텍스트 길이** — 긴 대화 이력, 검색 결과, 도구 응답을 담을 만큼 충분해야 합니다.
- **비용과 지연** — 큰 모델은 TTFT, 토큰 비용이 큽니다. 보조 작업은 작은 모델(또는 CPU 추론)으로 분리하는 멀티 모델 구성을 고려하십시오.
- **임베딩 모델** — 지식베이스를 쓰면 별도 임베딩 모델이 필요합니다. 인덱싱과 질의에 같은 임베딩 모델을 써야 검색이 일관됩니다([④](https://github.com/JaeHoYun/vcf-private-ai/tree/main/04-rag)).
- **공유 모델을 먼저 찾는다(PAIS 3.0부터)** — 조직에 중앙 provider 인스턴스가 있다면, 사내 표준 LLM과 임베딩 모델은 자기 네임스페이스에 새로 띄우지 말고 공유 모델을 참조하는 편이 GPU와 운영 부담을 줄입니다. 전용 파인튜닝 모델이나 규제로 분리해야 하는 데이터를 다루는 에이전트만 로컬 모델을 씁니다. 원격 클라우드 모델은 반출 정책이 허용하는 유스케이스에 한정합니다.

다음 문서에서는 만든 에이전트를 운영에 올리기 전 **평가하고 가드레일을 설계**하는 방법을 다룹니다.

---
[← 이전: 09 MCP 도구 통합](09-mcp-tools.md) | [목차](../README.md) | [다음: 13 평가와 가드레일 →](13-evaluation-guardrails.md)
