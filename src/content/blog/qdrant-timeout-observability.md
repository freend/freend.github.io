---
title: "[Observability] 500 에러를 Loki로 추적해 Qdrant timeout을 확인한 과정"
description: "Kubernetes에서 발생한 500 에러를 NGINX access log와 Loki의 의존성 로그로 추적하고, Qdrant timeout 변경 전후를 검증한 기록입니다."
pubDate: "Oct 3 2026"
heroImage: "/src/assets/blog-placeholder-2.jpg"
---

Kubernetes 배포와 Pod 상태는 정상인데 추천 API에서 간헐적으로 500 Internal Server Error가 발생했다.

처음부터 Qdrant가 원인이라고 판단할 수는 없었다. 추천 요청은 MariaDB, Qdrant, Embedding API를 차례로 호출하고, 어느 단계에서든 실패하면 최종적으로는 API의 500으로 보일 수 있기 때문이다.

이번 조사는 다음 순서로 진행했다.

```text
DB 환경설정 오류 확인
  -> DB 접속 경로 정리
  -> 남은 500을 NGINX access log에서 시간축으로 확인
  -> Loki에서 Qdrant 의존성 실패 확인
  -> Qdrant timeout을 30초로 명시
  -> 변경 전후 2일의 500 비율 비교
```

이 글의 로그 시간은 기본적으로 KST로 표기한다. Loki와 NGINX 원본 로그의 UTC 표기는 필요한 곳에 함께 적었다.

## 먼저 결론부터

| 구간 | 추천 요청 수 | HTTP 500 | 500 비율 |
| --- | ---: | ---: | ---: |
| timeout 변경 전 2일 | 17 | 14 | 82.4% |
| 커밋 기준 변경 후 2일 | 14 | 3 | 21.4% |
| 실제 rollout 이후 확인분 | 10 | 1 | 10.0% |

비교 기준은 `oper`에 30초 timeout 변경이 병합된 2026-09-11 11:39:23 KST다. 로그 조회 범위는 기준 시각의 2일 전부터 2일 후까지인 2026-09-09 11:39:23 ~ 2026-09-13 11:39:23 KST로 잡았다.

다만 코드 병합과 실제 배포 사이에는 시간이 있었다. 실제 rollout 커밋은 같은 날 21:46:26 KST였기 때문에, 변경 직후의 일부 로그는 아직 이전 이미지에서 발생한 요청으로 해석해야 한다.

## 1. 500 에러만으로는 원인을 확정할 수 없다

추천 API의 흐름은 다음과 같다.

```text
Client
  -> API
  -> MariaDB
  -> Qdrant
  -> Embedding API
  -> response
```

초기에는 DB 환경설정과 접속 경로 문제가 먼저 확인됐다. DVC로 관리되는 설정과 배포 환경 설정을 정리하고, 같은 VPC 내부의 MariaDB에 불필요한 SSH tunnel을 거치지 않고 직접 연결하도록 구조를 단순화했다.

이 조치 이후에도 500이 남았기 때문에, 다음 질문을 분리해서 확인했다.

- DB 접속 단계에서 실패했는가?
- Qdrant 요청이 오래 대기했는가?
- NGINX가 upstream 응답을 기다리다 종료했는가?
- 500이 실제 서버 오류인지, client disconnect나 route 오류인지?

## 2. NGINX access log에서 5초대 응답을 확인하다

먼저 Loki에서 추천 API host와 경로를 기준으로 NGINX access log를 조회했다.

```text
{namespace="nginx-gateway",container="nginx"} |= "api-recommend.effectmall.com"
```

원본 로그에는 다음과 같이 `request_time`과 `upstream_response_time`이 함께 기록되어 있었다. 개인정보와 User-Agent는 글에서 제거하고 상태와 시간만 남겼다.

```text
2026-09-09 12:24:34 KST  POST /recommend/  500  request_time=5.748  upstream_response_time=5.748
2026-09-09 12:24:50 KST  POST /recommend/  500  request_time=5.755  upstream_response_time=5.755
2026-09-11 11:56:55 KST  POST /recommend/  500  request_time=5.769  upstream_response_time=5.768
2026-09-11 11:57:08 KST  POST /recommend/  500  request_time=5.842  upstream_response_time=5.842
```

NGINX access log에서는 약 5.7~5.8초 대기 후 500으로 종료되는 반복 패턴을 확인했다.

이때 주목한 것은 정확히 5초라는 숫자 자체보다, 요청이 매번 비슷한 시간대에 끊긴다는 사실이었다. 애플리케이션과 Qdrant가 정상적으로 떠 있다면 요청이 5초 전후에 반복해서 timeout으로 끝날 이유가 없어야 한다. 그렇다면 단순한 일시적 예외가 아니라, 요청이 완료되기 전에 어느 계층에서 기다림을 중단하는 경계가 있다는 뜻일 수 있다.

처음 세운 질문은 “NGINX route timeout이 5초로 설정된 것인가?”가 아니었다. “Qdrant가 살아 있는데 왜 5초에 timeout이 발생하는가?”였다. 이 질문을 따라 요청 경로를 확인했고, Qdrant client/server의 기본 timeout 설정을 조사하게 됐다.

Qdrant의 5초 기본 경계와 NGINX에서 관찰된 5.7~5.8초 응답은 이 가설과 맞아떨어졌다. Qdrant 요청이 기본 timeout 안에 완료되지 않고, 네트워크와 프록시 처리 시간이 더해진 뒤 NGINX access log에 5초대 응답으로 남은 것으로 해석할 수 있다. 다만 이 로그만으로 NGINX route 자체의 timeout이 5초였다고 단정할 수는 없다.

## 3. Qdrant 공식 설정과 로그 패턴을 대조하다

Qdrant 공식 서버 설정에는 HTTP client request timeout, keep-alive timeout, disconnect timeout의 기본값이 5초로 정의되어 있다. 또한 Qdrant Python client는 생성 시 `timeout` 값을 명시적으로 전달할 수 있다.

여기서 “Qdrant가 정상적으로 떠 있다”는 것과 “모든 Qdrant 요청이 5초 안에 완료된다”는 것은 같은 의미가 아니다. 프로세스와 Pod가 살아 있어도 Qdrant가 해당 요청을 처리하는 데 5초 이상 걸리거나, 네트워크 구간에서 응답이 지연되면 client timeout이 발생할 수 있다. 이번 장애에서는 바로 그 가능성을 NGINX 시간 패턴과 애플리케이션 로그로 좁혀 갔다.

따라서 이번 현상은 다음처럼 해석하는 것이 정확하다.

```text
Qdrant의 5초 기본 경계
  -> 네트워크와 프록시 처리 시간 추가
  -> NGINX에서 약 5.7~5.8초 응답으로 관찰
  -> API는 HTTP 500 반환
```

공식 문서는 “NGINX가 5초에 timeout한다”고 설명하지 않는다. Gateway API는 HTTPRoute의 request/backend timeout을 지정하지 않으면 동작이 구현체에 따라 달라질 수 있다고 정의한다. 실제 `kiat-tech-rec-api-route`에도 `timeouts` 필드가 없었고, 별도의 NGINX `ProxySettingsPolicy`도 조회되지 않았다.

## 4. Loki에서 Qdrant timeout을 직접 확인하다

애플리케이션 로그에는 관측성 개선 이후 다음과 같은 구조화 로그가 남았다.

```text
{namespace="kiat",container="tech-rec-api-server"}
  | json
  | event="dependency_failed"
  | dependency="qdrant"
```

민감한 요청 본문과 stack trace는 제외하고 핵심 필드만 정리하면 다음과 같다.

```text
2026-09-12 16:21:57.177 KST
event=dependency_failed
dependency=qdrant
operation=scroll_history_vectors
duration_ms=30035.383
error_type=ResponseHandlingException

2026-09-12 16:22:30.876 KST
event=dependency_failed
dependency=qdrant
operation=query_recommendations
duration_ms=30038.212
error_type=ResponseHandlingException
```

같은 요청은 NGINX에서 다음처럼 기록됐다.

```text
2026-09-12 16:21:57 KST  POST /recommend/  500  request_time=30.049  upstream_response_time=30.049
```

이 기록은 timeout을 30초로 늘린 뒤에도 Qdrant가 30초 이상 응답하지 않으면 API가 다시 500을 반환할 수 있다는 것을 보여준다. 동시에 문제의 위치가 단순한 “API 500”이 아니라 `qdrant / scroll_history_vectors` 단계라는 것도 확인할 수 있다.

## 5. Qdrant timeout을 코드에 명시하다

30초 변경 커밋에서는 Qdrant client 초기화 시 timeout을 명시했다.

```python
QDRANT_TIMEOUT_SECONDS = 30

client = QdrantClient(
    url=cfg.QDRANT.host,
    api_key=cfg.QDRANT.api_key if cfg.QDRANT.api_key else None,
    timeout=QDRANT_TIMEOUT_SECONDS,
)
```

동일한 변경에 timeout 값과 client keyword가 실제 소스에 존재하는지 확인하는 테스트도 추가했다.

```python
def test_qdrant_client_has_explicit_30_second_timeout():
    # QDRANT_TIMEOUT_SECONDS == 30인지 확인하고,
    # QdrantClient(timeout=QDRANT_TIMEOUT_SECONDS) 호출을 검증한다.
```

30초는 성능을 보장하는 숫자가 아니다. 요청을 무한정 기다리지 않고 서비스 요구사항에 맞는 실패 경계를 정하기 위한 값이다. 검색 시간이 30초를 넘어가는 원인은 별도로 Qdrant 성능, 네트워크, 데이터량, query 조건으로 조사해야 한다.

## 6. 변경 전후의 수치

동일한 host와 `POST /recommend/` 경로만 대상으로 Loki의 NGINX access log를 집계했다.

변경 전 2일에는 17건 중 14건이 500이었다. 그중 2건은 약 5.75초에서 실패했다.

변경 후 2일에는 14건 중 3건이 500이었다. 나머지는 200 응답 9건, 499 한 건, 503 한 건이었다. 실제 이미지 rollout 이후의 확인분만 보면 10건 중 500은 한 건이었다.

변경 후에도 30.049초에서 500이 한 번 발생했지만, 이 요청은 애플리케이션 Loki에서 Qdrant `ReadTimeout`과 함께 확인됐다. 즉 30초 변경은 500을 완전히 제거한 것이 아니라, 기존의 약 5초 실패 경계를 늘려 정상 응답을 받을 수 있는 시간을 확보했고, 여전히 오래 걸리는 요청은 30초에서 명시적으로 실패하도록 만든 것이다.

## 7. 관측성 개선이 해결을 증명하게 만들었다

이번 장애에서 중요한 변경은 timeout 값만이 아니었다. Request ID와 의존성별 처리 시간을 JSON으로 기록하면서 다음 연결이 가능해졌다.

```text
NGINX request
  -> API request_id
  -> MariaDB dependency timing
  -> Qdrant dependency timing
  -> request_failed
  -> NGINX status/request_time
```

DB 연결 문제는 설정과 접속 경로를 먼저 정리했고, 이후 남은 500은 NGINX와 Loki의 시간축을 대조해 Qdrant 단계로 좁혔다. 최초 원인 발견의 단서는 약 5.7~5.8초 후 발생한 NGINX 500이었다. 이후 timeout을 30초로 늘린 뒤에도 같은 종류의 문제가 남았고, 관측성 로그를 통해 Qdrant 단계에서 약 30초 후 실패한다는 사실을 확인했다.

- NGINX: `POST /recommend/ 500`, `request_time=30.049`
- 애플리케이션: Qdrant `scroll_history_vectors`, `duration_ms=30035.383`, `ReadTimeout`

이런 연결이 없었다면 500이 줄었다는 결과는 확인할 수 있어도, 어떤 변경이 어떤 실패 경계를 이동시켰는지는 설명하기 어려웠을 것이다.

## 왜 이 오류를 바로 찾지 못했을까?

처음부터 애플리케이션 로그에서 “Qdrant가 5초에 timeout됐다”는 한 줄을 확인할 수 있었던 것은 아니다. 당시 NGINX access log에는 다음과 같은 결과가 남아 있었다.

```text
POST /recommend/  500  request_time=5.748  upstream_response_time=5.748
POST /recommend/  500  request_time=5.755  upstream_response_time=5.755
```

하지만 애플리케이션 로그만으로는 해당 요청을 Qdrant 호출과 연결할 수 있는 요청 ID, 의존성 이름, 처리 시간, 오류 유형이 충분히 남지 않았다. 따라서 “API가 약 5초 후 500을 반환했다”는 사실은 알 수 있었지만, “Qdrant 요청이 기본 timeout 경계에서 실패했다”는 원인까지 바로 확정하기는 어려웠다.

이 문제를 특정 팀이 에러 로그의 중요성을 몰랐기 때문이라고 단정하는 것은 정확하지 않다. 더 본질적인 문제는 기능 개발에 필요한 로그와 운영 장애를 분석하는 데 필요한 로그가 달랐는데, 후자의 정보가 처음부터 설계되어 있지 않았다는 점이다.

이번 사건의 단계별 증적은 다음과 같다.

```text
최초 증상
  -> NGINX에서 약 5.7~5.8초 후 500

1차 대응
  -> Qdrant timeout을 30초로 변경

후속 확인
  -> Qdrant가 30초 이상 응답하지 않아 scroll_history_vectors 실패

2차 대응
  -> timeout을 60초로 증가
```

즉, 처음 확인해야 했던 것은 30초 timeout이 아니라 5초대에서 발생한 Qdrant 요청 timeout이었다. 30초와 60초는 그 원인을 확인한 뒤 서비스의 실패 경계를 조정한 후속 대응이다.

이 경험을 통해 에러 로그는 단순히 예외 메시지를 출력하는 기능이 아니라는 점을 확인했다. 요청 ID, 의존성, operation, 처리 시간, 오류 유형이 함께 기록되어야 NGINX의 500과 애플리케이션 내부의 Qdrant 실패를 하나의 사건으로 연결할 수 있다.

## 커밋 기록

private 저장소이므로 링크는 남기지 않고 커밋 메시지만 기록한다.

- `fix: set Qdrant timeout to 30 seconds`
- `merge: bring Qdrant timeout fix into oper`
- `feat: roll out latest image after successful build`
- `feat: increase Qdrant timeout and log safe recommend requests`

## 공식 문서 및 출처

- [Qdrant server settings](https://github.com/qdrant/qdrant/blob/master/src/settings.rs) — Qdrant HTTP client 관련 기본 timeout 설정
- [Qdrant Python client](https://github.com/qdrant/qdrant-client/blob/master/qdrant_client/qdrant_client.py) — client 생성 및 operation timeout 파라미터
- [Gateway API GEP-1742: HTTPRoute timeouts](https://gateway-api.sigs.k8s.io/geps/gep-1742/) — request/backend timeout의 의미와 미설정 시 동작
- [NGINX Gateway Fabric custom policies](https://docs.nginx.com/nginx-gateway-fabric/overview/custom-policies/) — upstream 동작과 timeout 정책을 설정하는 공식 방법

## 마무리

이번 사례의 교훈은 “500을 없앴다”가 아니다.

```text
DB 설정 문제 해결
  -> 남은 500의 시간 패턴 확인
  -> Qdrant 5초 경계와 NGINX 5.7초대 실패 연결
  -> Qdrant timeout을 30초로 명시
  -> 500 비율 감소 확인
  -> 30초 초과 요청은 다시 실패한다는 한계까지 확인
```

HTTP 500은 최종 결과일 뿐이다. 중요한 것은 요청이 어느 단계에서 멈췄고, 그 사실을 NGINX access log와 Loki의 구조화 로그로 어떻게 증명할 수 있는가다. 이 연결이 갖춰져야 간헐적인 500이 막연한 장애가 아니라 재현 가능한 운영 사건이 된다.
