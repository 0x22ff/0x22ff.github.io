---
layout: post
title: "SLO/SLI/Error Budget, 운영에서 진짜로 쓰는 방식 정리"
date: 2026-03-14 20:10:00 +0900
categories: [backend]
tags: [sre, slo, sli, error-budget, prometheus, grafana]
thumbnail: /assets/images/thumb-slo-sli-error-budget.svg
---

모니터링 도구는 다 붙였는데, 막상 장애가 나면 이런 상황이 자주 나온다.

- 알람은 많이 오는데 뭐가 진짜 급한지 모르겠고
- 팀마다 "이 정도면 괜찮다" 기준이 다르고
- 배포 멈출지 계속 갈지 결정이 감으로 흘러간다

여기서 기준선을 잡아주는 게 `SLO/SLI/Error Budget`이다.

이번 글은 이론 설명보다, **운영에서 바로 적용 가능한 형태**로 정리했다.

## 1) 용어부터 딱 맞춰두기

![SLO SLI Budget overview](/assets/images/slo-sli-budget-overview.svg)

- `SLI` (Service Level Indicator): 측정값
  - 예: 요청 성공 비율, p95 지연 시간
- `SLO` (Service Level Objective): 목표값
  - 예: 30일 기준 가용성 99.9%
- `Error Budget`: 목표를 기준으로 허용 가능한 실패량
  - 수식: `1 - SLO`

핵심은 단순하다.  
**측정(SLI) -> 목표(SLO) -> 허용 실패량(Error Budget)** 순서다.

## 2) 숫자로 보면 훨씬 명확해진다

예시(계산 예시):

- SLO: 30일 가용성 99.9%
- 허용 에러율: `1 - 0.999 = 0.001 (0.1%)`
- 30일 총 분: `30 * 24 * 60 = 43,200분`
- 허용 장애 시간(완전 장애 기준): `43,200 * 0.001 = 43.2분`

![Error budget math example](/assets/images/slo-budget-math-example.svg)

즉 30일 동안 완전 다운 시간이 약 43.2분을 넘기면 이 SLO는 깨진다.

## 3) SLI는 식을 먼저 확정해야 한다

SLI가 애매하면 SLO도 애매해진다.  
그래서 분자/분모를 코드 리뷰 가능한 수준으로 명확히 정의하는 게 중요하다.

예시(요청 기반 가용성 SLI):

```text
SLI = successful_requests / total_requests
```

Spring + Prometheus 환경에서 자주 쓰는 형태는 아래다.

```promql
sum(rate(http_server_requests_seconds_count{status!~"5.."}[5m]))
/
sum(rate(http_server_requests_seconds_count[5m]))
```

주의할 점:

- `4xx`를 실패로 볼지 말지는 서비스 정책에 따라 다름
- 중요한 건 팀 내에서 정의를 고정하고 계속 유지하는 것

## 4) Burn Rate를 붙이면 알람 품질이 달라진다

Error Budget을 "얼마나 남았는지"만 보면 늦을 때가 있다.  
그래서 운영에서는 "얼마나 빨리 타고 있는지"를 같이 본다.

`burn_rate` 기본식:

```text
burn_rate = current_error_rate / allowed_error_rate
```

예시:

- 허용 에러율 0.1% (`0.001`)
- 현재 에러율 1% (`0.01`)
- `burn_rate = 0.01 / 0.001 = 10`

![Burn rate concept](/assets/images/slo-burn-rate-concept.svg)

`burn_rate > 1`이면 예산을 계획보다 빠르게 소모 중이라는 뜻이다.

## 5) 알람은 멀티 윈도우로 가는 게 안정적이다

짧은 윈도우만 보면 스파이크 노이즈에 취약하고,  
긴 윈도우만 보면 감지가 늦다.

그래서 실무에선 짧은/긴 윈도우를 같이 본다.

![Multi-window alerting](/assets/images/slo-alerting-windows.svg)

예시(개념 예시):

- short window: 5분 burn rate
- long window: 1시간 burn rate
- 두 조건이 같이 나쁠 때 페이지 알람 발송

PromQL 예시(99.9% SLO, 허용 에러율 0.001 가정):

```promql
(
  sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
  /
  sum(rate(http_server_requests_seconds_count[5m]))
) / 0.001
```

운영에서는 보통 이 값을 recording rule로 저장하고 alert rule에서 재사용한다.

## 6) Grafana 대시보드는 이 3개가 핵심이다

- 현재 SLI 추세
- Burn rate(short/long)
- Error budget remaining

![SLO dashboard layout](/assets/images/slo-dashboard-layout.svg)

처음부터 패널을 많이 만들기보다,  
온콜 상황에서 바로 의사결정 가능한 화면부터 만드는 게 낫다.

## 7) 배포/운영 의사결정에 연결해야 의미가 있다

SLO를 만든 뒤 운영에 안 붙이면 그냥 숫자 대시보드로 끝난다.

실무에서는 보통 이렇게 쓴다.

- budget이 빠르게 줄면 신규 변경 배포를 보수적으로 진행
- 경보가 반복되면 원인 제거 작업(성능/안정성)을 우선순위로 올림
- 월 단위 리뷰에서 SLO 타당성(너무 느슨/빡빡)을 다시 조정

![SLO adoption roadmap](/assets/images/slo-adoption-roadmap.svg)

## 8) 시작 체크리스트

1. 사용자 영향이 큰 API 하나만 먼저 고른다
2. SLI 식(분자/분모)을 문서로 고정한다
3. SLO 목표값/기간을 합의한다
4. Burn-rate 알람(짧은+긴 윈도우)을 붙인다
5. 월 1회 SLO 리뷰를 정례화한다

---

결국 SLO/SLI/Error Budget은 "모니터링 고급 기능"이 아니라,  
**장애 대응과 배포 결정을 같은 기준으로 맞추는 운영 언어**에 가깝다.

이 기준만 맞아도,

- 알람 노이즈가 줄고
- 우선순위가 분명해지고
- 팀 의사결정이 빨라진다.

## References

- Google SRE Book - Service Level Objectives: https://sre.google/sre-book/service-level-objectives/
- Google SRE Workbook - Alerting on SLOs: https://sre.google/workbook/alerting-on-slos/
- Prometheus docs - Histograms and summaries: https://prometheus.io/docs/practices/histograms/
- Prometheus docs - Alerting rules: https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- Grafana docs - Alerting: https://grafana.com/docs/grafana/latest/alerting/
