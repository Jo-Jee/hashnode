---
title: "[BariBot] Day 0 - "
datePublished: 2026-04-08T23:35:04.518Z
cuid: cmnqor7er00j02blz6h9m6oqu
slug: baribot-day-0
cover: https://cdn.hashnode.com/uploads/covers/69c777487cf2706510c47f9a/8bd9ee87-a7eb-4774-8d9e-6c7f23bafe5f.png

---

> Kubernetes 환경에서 AI로 로그를 분석하고 알림하는 서비스를 만들기로 했다. 코드를 한 줄도 쓰기 전에, 어떤 결정들을 내렸는지 기록한다.

## 왜 만드는가

Kubernetes 환경에서 로그를 수집, 분석, 적재하고 알림까지 보내는 전체 프로세스를 **Kubernetes native하게** 처리하는 도구를 만들고 싶었다. 기존 도구들은 각각의 영역은 잘 하지만, 전체를 하나의 흐름으로 엮으면서 K8s 생태계에 자연스럽게 녹아드는 건 드물다.

최종 목표는 수집 → 분석 → 적재 → 알림의 전체 파이프라인을 커버하는 것이다. 하지만 순서가 중요하다. **분석이 주력**이기 때문에 분석에 최적화된 구조를 먼저 설계하고, 그 구조에 맞춰서 수집과 적재를 가장 효율적으로 붙이려고 한다.

수집을 먼저 만들고 분석을 나중에 얹으면, 수집 구조가 분석에 맞지 않아 나중에 뜯어고쳐야 할 수 있다. 분석이 필요로 하는 데이터 형태, 전처리 수준, 컨텍스트 그룹핑을 먼저 정의하고, 거기에 최적화된 수집 파이프라인을 구성하는 게 순서상 맞다고 판단했다.

적재도 마찬가지다. 단순히 S3에 쌓는 건 쉽지만, 나중에 검색이 안 되면 의미가 없다. 분석 패턴이 확정된 후에 인덱싱 전략(Loki 스타일 라벨 인덱스 + 청크 등)을 설계하고 적재를 붙일 예정이다.

## 아키텍처: 왜 Agent + Server 구조인가

처음에는 서버가 K8s API로 로그를 Pull하는 방식을 생각했다. 단순하니까. 그런데 두 가지 문제가 있었다.

첫째, K8s API의 pod 로그 엔드포인트(`/api/v1/.../log`)는 `sinceTime`이나 `sinceSeconds` 같은 시간 기반 필터만 지원하고, **바이트 오프셋 기반의 정확한 이어받기가 불가능하다.** 같은 초에 여러 로그가 찍히면 중복이나 누락이 생길 수 있다. 반면 노드 로컬의 `/var/log/containers/*.log`를 직접 tail하면 파일 오프셋으로 정밀하게 resume할 수 있다.

둘째, 네트워크 비용 문제다. 모든 노드의 로그가 중앙으로 올라오는데, 대부분은 AI가 볼 필요 없는 로그다. 전처리로 충분히 걸러낼 수 있는 health check, 반복 noise 같은 로그까지 네트워크를 타는 건 낭비다.

그래서 각 노드에 DaemonSet으로 **Agent를 배포**하기로 했다. Agent가 로컬 로그 파일을 직접 tail하면서 1차 필터링하고, 의미 있는 로그만 중앙 Server로 보내는 구조다.

![](https://cdn.hashnode.com/uploads/covers/69c777487cf2706510c47f9a/a9ea0174-e02b-4161-a4dc-43ce1212e66f.png align="center")

## 바이너리 구조: 하나인가 둘인가

코드베이스는 하나, 바이너리는 둘로 빌드한다.

```plaintext
cmd/agent/main.go  → bari-bot-agent
cmd/server/main.go → bari-bot-server
```

`internal/` 아래 공통 로직(필터 엔진, 로그 모델 등)은 양쪽이 공유한다. Agent 바이너리에는 AI/알림 코드가 포함되지 않아 가볍다.

처음에는 단일 바이너리에 서브커맨드로 분기하려 했는데, CRD 컨트롤러까지 서버에 포함시키면 바이너리가 무거워진다는 우려가 있었다. 결국 코드 레벨에서는 합치되 빌드를 분리하는 방향으로 결정했다.

## 전처리: AI 비용을 줄이는 핵심

AI 호출 비용을 최적화하기 위해 전처리를 **보내는 쪽(Agent)과 받는 쪽(Server) 둘 다**에서 한다.

**Agent (1차 필터링 — IngestionRule):**

기본 동작은 **아무 로그도 수집하지 않는 것**이다. 명시적으로 지정한 대상만 읽는 default-deny 방식이다.

*   포함 규칙: label 기반으로 특정 pod의 로그를 수집하거나, location으로 특정 파일 경로를 직접 지정
    
*   제외 규칙: 포함된 로그 중에서도 health check, noise 등 불필요한 로그 드롭
    
*   샘플링: 제외되지 않은 로그에 비율 제어 적용
    
*   평가 순서: 포함 → 제외 → 샘플링
    

"일단 다 읽고 필요 없는 걸 빼는" 방식이 아니라, "필요한 것만 읽고 그 안에서도 한 번 더 거르는" 방식이다. 이래야 Agent의 리소스 사용량을 예측 가능하게 유지할 수 있다.

**Server (2차 전처리):**

*   Fingerprint 기반 중복 제거 (5분 윈도우, LRU 10만 항목)
    
*   Mute 규칙으로 알림 억제
    
*   같은 pod/namespace 로그를 30초 윈도우로 그룹핑하여 AI에 전달
    

## Fingerprinting: 수치가 다른 로그도 같은 로그다

`"Connection timeout after 3542ms to 10.0.1.5"`와 `"Connection timeout after 1203ms to 10.0.2.8"`은 본질적으로 같은 에러다. 숫자, IP, UUID, 타임스탬프 같은 변수 부분을 플레이스홀더로 치환한 뒤 해싱하면 같은 fingerprint가 나온다.

```plaintext
원본: "Connection timeout after 3542ms to 10.0.1.5"
정규화: "Connection timeout after <NUM>ms to <IP>"
해시: a1b2c3d4...
```

이 fingerprint를 Agent에서 생성하고, Server에서 dedup 기준으로 사용한다.

## 멀티라인 조합은 반드시 Agent에서

Stack trace는 여러 줄로 쪼개져서 로그 파일에 기록된다. 이걸 하나의 LogEntry로 합치는 건 **반드시 Agent에서** 해야 한다.

왜? Server를 나중에 HA로 구성하면, Agent가 배치로 보낸 로그가 로드밸런서를 통해 서로 다른 Server pod에 분산될 수 있다. Stack trace의 앞부분은 Server A에, 뒷부분은 Server B에 도착하면 조합이 불가능하다.

Agent는 노드 로컬에서 파일을 순차적으로 읽으니 줄 순서가 보장된다.

## 이중 알림 구조

알림은 두 갈래로 나뉜다:

1.  **규칙 기반 즉시 알림**: FATAL, OOMKilled 같은 건 AI 분석 기다릴 필요 없이 바로 Slack으로
    
2.  **AI 분석 결과 알림**: 분석이 끝나면 같은 이슈의 Slack 메시지에 **스레드로** 결과를 추가
    

이렇게 하면 긴급한 건 즉시 받고, AI의 심층 분석은 컨텍스트가 연결된 상태로 나중에 확인할 수 있다.

Slack 스레드 연결을 위해 webhook이 아닌 **Web API** (token 기반 `chat.postMessage`)를 사용한다. Webhook은 `thread_ts`를 반환하지 않아서 스레드 매핑이 불가능하다.

## 상태 관리: 왜 CRD인가

필터 규칙, 알림 규칙, AI 설정 같은 동적 상태를 어디에 저장할지 고민했다.

| 선택지 | 장단점 |
| --- | --- |
| BoltDB | Pod 재시작 시 유실. PV 붙이면 번거로움 |
| Redis | 외부 의존성 추가 |
| ConfigMap | CRD 없이 가능하지만 수동 관리 |
| **CRD** | K8s 네이티브, kubectl로 관리 가능, watch로 실시간 반영 |

CRD를 선택하면 컨트롤러가 필요하고, 그러면 바이너리가 무거워진다는 우려가 있었다. 하지만 컨트롤러를 Server 바이너리에 포함시키고, Agent는 IngestionRule CRD만 K8s API로 직접 watch하면 양방향 gRPC도 불필요하고 깔끔하다.

정의한 CRD 세 가지:

*   **IngestionRule**: 1차/2차 필터링 규칙, mute 규칙
    
*   **AlertRule**: 규칙 기반 즉시 알림 조건
    
*   **AIConfig**: 모델 설정, 예산 한도
    

## 멀티 모델 지원

특정 AI 모델에 종속되고 싶지 않았다. 모델별로 강점이 다르고 비용도 다르니까.

*   간단한 분류 → 저렴한 모델 (Haiku 급)
    
*   이상 탐지 → 중급 모델 (Sonnet 급)
    
*   근본 원인 분석 → 고급 모델 (Opus 급)
    

용도별 기본 모델이 있되, CRD에서 오버라이드할 수 있다. 어댑터 패턴으로 Claude, GPT, Ollama를 동일한 인터페이스 뒤에 구현했다. Circuit breaker도 모델별로 독립적이라 Claude가 장애나도 GPT는 계속 동작한다.

## AI 비용 제어

AI 비용이 무한히 커지는 걸 막기 위해:

*   일일/시간별 예산 한도를 AIConfig CRD에 설정
    
*   OpenTelemetry로 토큰 사용량, 비용 추적
    
*   한도 초과 시 AI를 끄고 규칙 기반 처리만 수행
    
*   예산 리셋 시 자동 재개
    

## 기술 스택

| 영역 | 선택 | 이유 |
| --- | --- | --- |
| 언어 | Go | 고성능 로그 처리, K8s 생태계 |
| 통신 | gRPC | 효율적, 스트리밍 지원 |
| 상태 | K8s CRD | 외부 의존성 없음, kubectl 관리 |
| 알림 | Slack Web API | 스레드 지원 |
| 관측 | OpenTelemetry | 벤더 무관 |
| 배포 | Helm | K8s 표준 |

## MVP에서 빠진 것들

의도적으로 빠진 것들이다. 각각 "안 할 것"이 아니라 "지금은 안 할 것":

*   **로그 적재/아카이빙**: S3에 단순 적재하면 나중에 검색이 안 됨. 인덱싱 전략(Loki 스타일 라벨 인덱스 + 청크 등)을 설계한 후 추가
    
*   **웹 대시보드**: Slack + Prometheus/Grafana로 충분한 단계
    
*   **서버 HA**: 단일 레플리카로 시작. 재시작 시 dedup 캐시/Slack 스레드 매핑은 유실되지만 영향 제한적. HA는 추후 구성
    
*   **멀티 클러스터**: 단일 클러스터 먼저