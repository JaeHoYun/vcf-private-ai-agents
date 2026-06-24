# 07 — 운영과 Day-2

[← 목차로](../README.md)

에이전트를 만들었다면 이제 운영입니다. 이 문서는 에이전트·모델 워크로드를 어디에 배포하고, 무엇을 관측하며, 업그레이드 때 무엇을 조심하고, 어떤 알려진 이슈와 비용이 있는지를 다룹니다. 용량·TCO의 정밀 계산은 ⑥에, 플랫폼 Day-2 운영 전반은 ①에 위임하고, 여기서는 **에이전트 워크로드에 직접 닿는 운영**에 집중합니다.

> 본 문서의 수치·동작은 작성 시점(2026-06) VCF 9.1 / PAIF 9.1 / PAIS 2.1 기준이며, 적용 전 공식 문서로 재확인하시기 바랍니다.

---

## 7.1 배포 토폴로지

- **실행 기반** — 모델 엔드포인트·에이전트는 Supervisor의 vSphere Namespace에 프로비저닝된 **VKS 클러스터** 위에서 실행되며, ESXi 호스트의 GPU에 연결됩니다([01 §1.4](01-foundations.md)). PAIS는 VKr 1.33·NVIDIA GPU Operator 25.10.1 기준으로 동작합니다(작성 시점).
- **두 경로** — 프로토타이핑·노트북 작업은 **DLVM(Deep Learning VM)**, 프로덕션 모델 엔드포인트·에이전트는 **VKS 클러스터**에 둡니다.
- **고가용성** — 모델 엔드포인트는 서로 다른 워커 노드에 복제본 2 이상을 두기를 권장합니다. 단일 zone 배포에서는 VKS 컨트롤 플레인 등 핵심 구성요소가 단일 인스턴스로 배치돼 가용성 제약이 따릅니다.
- **사이징 위임** — 프로덕션 모델 서빙에 필요한 최소 GPU 호스트 수·GPU 메모리·시스템 RAM 비율 등 용량 산정은 [⑥ VKS 클러스터 사이징](https://github.com/JaeHoYun/vcf-private-ai-sizing-cost/blob/main/docs/04-vks-cluster-sizing.md)에 위임합니다. 공식 디자인 문서도 구체 수치를 별도 사이징 자료로 위임합니다.

## 7.2 관측성

PAIS 2.1은 추론·GPU·에이전트를 아우르는 관측을 VCF Operations의 AI 지표 화면으로 제공하며, OpenTelemetry 기반 LLM 추적을 지원합니다([01 §1.2.5](01-foundations.md)).

**모델·서비스 지표**

| 지표 | 의미 |
|------|------|
| Time to First Token (TTFT) | 첫 토큰까지 걸린 시간 — 체감 응답성 |
| Token throughput | 초당 토큰 처리량 — 처리 용량 |
| End-to-end latency | 요청 종단 지연 |
| Tokens per request | 요청당 토큰 수 — 비용·길이 |
| Cache utilization | 캐시 활용도 |

**GPU 지표** — 사용률·온도·전력·메모리 온도·메모리 클럭 등.

**에이전트 추적** — OpenTelemetry LLM 추적으로 "사용자 ↔ 모델 ↔ 에이전트 ↔ 지식베이스" 상호작용을 따라갑니다. 백엔드는 Prometheus·Grafana를 씁니다. 추적이 보이지 않으면 구성 오류일 수 있습니다(§7.4).

> 평가([06](06-evaluation-guardrails.md))가 "출시 전 품질"이라면, 관측성은 "운영 중 품질·비용"입니다. TTFT·종단 지연·토큰 처리량을 기준선으로 잡아 두고 이탈을 감시하십시오.

## 7.3 업그레이드 — 다운타임 주의

> **반드시 알아둘 운영 리스크** — PAIS 2.0.x → 2.1 업그레이드는 **모델 엔드포인트를 호스팅하는 VKS 클러스터를 삭제·재생성**합니다. 그 과정에서 노드가 재생성되고 모델을 다시 내려받는 동안 **다운타임**이 발생합니다.

대비:

- 업그레이드 창(window)을 다운타임 전제로 계획하고 이해관계자에 사전 공지합니다.
- 모델 재다운로드 시간을 고려해 충분한 창을 잡습니다(모델 크기·대역폭 의존).
- 업그레이드 후 모델 엔드포인트·에이전트·지식베이스·MCP 연결이 정상 복구되는지 검증 절차를 둡니다.
- 플랫폼 LCM 업그레이드 순서·롤백 등 전반은 [① Day-2 운영](https://github.com/JaeHoYun/vcf-private-ai-guide/blob/main/docs/10-operations.md)의 §10.1 LCM 런북에 위임합니다.

## 7.4 알려진 이슈

작성 시점 PAIS 2.1에서 보고된 대표 이슈입니다(릴리스 노트 기준, 변동 가능).

| 증상 | 방향 |
|------|------|
| GPU 파드가 `CDI device injection failed`로 실패 | GPU Operator Helm 값 조정(CDI 관련 설정) |
| 업그레이드 후 모델 엔드포인트가 메모리 부족으로 실패 | 2.1에서 VRAM 요구량 증가 — 자원 재산정([⑥ 컴퓨트·메모리 사이징](https://github.com/JaeHoYun/vcf-private-ai-sizing-cost/blob/main/docs/03-compute-memory-sizing.md)) |
| OpenTelemetry LLM 추적이 표시되지 않음 | 추적 구성 점검(§7.2) |
| 네임스페이스당 모델 엔드포인트 복제본 상한(작성 시점 최대 15) | 복제본·엔드포인트 수 설계 시 상한 고려 |

증상→진단→조치 형태의 트러블슈팅 런북 패턴은 [① Day-2 운영](https://github.com/JaeHoYun/vcf-private-ai-guide/blob/main/docs/10-operations.md)의 §10.2 트러블슈팅 런북을 참조하십시오.

## 7.5 비용

에이전트 자체는 모델·도구를 잇는 경량 계층이고, 무거운 비용은 **모델 추론(GPU)** 에서 발생합니다.

- **GPU 라이선스** — vGPU(가상 GPU)를 쓰면 NVIDIA AI Enterprise(NVAIE) 라이선스가 필요합니다(호스트·게스트 드라이버 모두). GPU 패스스루(DirectPath) 모드는 NVAIE가 불필요하나 vMotion 등 기능 제약이 따릅니다. 정확한 라이선스 과금 단위는 NVIDIA·Broadcom 라이선싱 자료로 확인하시기 바랍니다.
- **CPU 추론 대안** — 작은 모델·저부하 작업은 llama.cpp CPU 추론으로 GPU 비용을 줄이는 선택지가 있습니다([05 §5.2](05-models-serving.md)). 성능 trade-off를 평가로 확인하십시오.
- **토큰·단계 비용** — 에이전트는 도구 호출·재시도로 단계가 늘어 단일 호출보다 토큰을 많이 씁니다. 종료 조건([06 §6.4](06-evaluation-guardrails.md))과 세션 요약([02 §2.4](02-design-patterns.md))으로 토큰 폭증을 통제하십시오.
- **TCO 위임** — GPU·노드·스토리지 비용의 정밀 산정과 활용도 회수는 [⑥ TCO와 비용 모델](https://github.com/JaeHoYun/vcf-private-ai-sizing-cost/blob/main/docs/07-tco-cost-model.md)에 위임합니다.

## 7.6 운영 점검 리듬

에이전트 워크로드에 맞춘 가벼운 점검 리듬을 제안합니다(플랫폼 전반 리듬은 ①).

- **일상** — TTFT·종단 지연·오류율·GPU 사용률 기준선 이탈 감시, 에이전트 추적 표본 점검.
- **주기** — 평가 묶음 회귀([06](06-evaluation-guardrails.md)) 재실행, MCP 도구·자격증명 유효성 점검, 토큰·비용 추세 검토.
- **변경 시** — 모델·지시문·도구 변경 후 평가·관측 기준선 갱신, 업그레이드는 다운타임 창 계획(§7.3).

## 7.7 백업과 복구

에이전트 서비스는 stateless가 아닙니다 — 여러 stateful 자산이 흩어져 있어, 무엇을 백업하는지부터 정리해야 합니다.

| 자산 | 저장 위치 | 백업·복구 |
|------|-----------|-----------|
| 지식베이스 벡터·데이터 | pgvector(외부 PostgreSQL) | PostgreSQL 백업([②](https://github.com/JaeHoYun/vcf-dsm-vectordb-guide)) |
| 모델 아티팩트 | Model Gallery(Harbor) | 레지스트리 백업 또는 AMT로 재미러([05 §5.4](05-models-serving.md)) |
| 에이전트·도구 구성 | Agent Builder 구성 코드 | 형상관리(Git)에 보관·재현([03 §3.8](03-agent-builder.md)) |
| MCP 서버 등록·승인 | Tool Gallery 등록 정보 | 등록 절차를 문서화해 재등록 가능하게([04 §4.4](04-mcp-tools.md)) |
| 세션 상태 | 배포 단위 확인 필요 | 영속성·복구 가능 여부는 공식 문서로 확인([01 §1.4](01-foundations.md)) |

- **복구 우선순위** — 구성(Git)과 모델(재미러)은 재현이 쉽고, **지식베이스 데이터는 원본 재인덱싱 비용이 크므로** 백업 가치가 가장 높습니다.
- **DR 위임** — 멀티사이트 재해복구·RTO/RPO·백업 주기 등 플랫폼 DR 정책은 [① Day-2 운영](https://github.com/JaeHoYun/vcf-private-ai-guide/blob/main/docs/10-operations.md)의 §10.3 백업·복구에 위임합니다. 이 가이드의 몫은 **무엇이 stateful한지 식별**해 DR 범위에서 빠뜨리지 않게 하는 것입니다.

## 7.8 스케일·알람·SLO·온콜

§7.2가 무엇을 관측하는지였다면, 여기서는 그 위에 임계·목표·대응을 얹습니다.

- **스케일** — 모델 엔드포인트는 복제본으로 가용성·처리량을 늘리되 네임스페이스당 상한이 있습니다(§7.4). 에이전트는 도구 호출 루프·동시 세션 급증으로 부하가 갑자기 치솟으므로, 복제본을 수동으로 늘릴지 부하 기반 자동 확장이 되는지는 공식 문서로 확인하십시오. 토큰 폭증 통제는 종료 조건·세션 요약으로 합니다(§7.5).
- **알람** — §7.2 기준선 위에 임계를 정합니다 — TTFT·종단 지연 P95(95 백분위), 오류율, GPU 포화, 복제본 가용성. VCF Operations 알람으로 거는 것이 1차이며, 외부 온콜 도구 연동은 앱·외부 계층입니다.
- **목표 수준(SLO)** — 가용성·TTFT·오류율 같은 지표에 목표값을 정해 둡니다. 기준선을 실측한 뒤 현실적인 값으로 잡고, 이탈이 잦으면 용량([⑥ 용량 계획과 운영](https://github.com/JaeHoYun/vcf-private-ai-sizing-cost/blob/main/docs/06-capacity-planning.md))이나 설계를 재검토합니다.
- **온콜·에스컬레이션** — 1차 대응은 알려진 이슈·트러블슈팅(§7.4)으로, 해소되지 않으면 플랫폼 운영([① Day-2 운영](https://github.com/JaeHoYun/vcf-private-ai-guide/blob/main/docs/10-operations.md) §10.4)·보안([⑤](https://github.com/JaeHoYun/vcf-private-ai-security-governance))으로 에스컬레이션합니다.

이로써 구축에서 운영까지의 흐름을 마칩니다. 용어·참조·변경 이력은 부록에 정리했습니다.

---
[← 이전: 06 평가와 가드레일](06-evaluation-guardrails.md) · [목차](../README.md) · [다음: A1 부록 →](../appendix/A1-reference.md)
