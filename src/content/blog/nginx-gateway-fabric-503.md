---
title: "[Cloud Native] 정상인 Kubernetes에서 503을 반환한 NGINX Gateway Fabric"
description: "Pod와 EndpointSlice가 모두 정상인데도 NGINX Gateway Fabric이 503 fallback upstream을 사용한 장애를 계층별 관측으로 추적한 기록입니다."
pubDate: "Sep 7 2026"
heroImage: "/src/assets/blog-placeholder-4.jpg"
---

Kubernetes에서 HTTP 503을 만나면 가장 먼저 애플리케이션 Pod를 의심하게 됩니다. 배포 직후라면 새 이미지가 잘못되었거나, Service selector가 바뀌었거나, readiness probe가 실패했을 가능성을 떠올리기 쉽습니다.

이번 장애는 그 직관이 맞지 않았습니다. 애플리케이션 Pod는 `Ready`였고, Service와 EndpointSlice에도 정상 endpoint가 있었습니다. Prometheus scrape도 정상이었습니다. 그런데 외부 도메인으로 요청하면 NGINX Gateway Fabric이 즉시 503을 반환했습니다.

결정적인 단서는 NGINX가 실제 backend가 아니라 내부 503 fallback socket을 upstream으로 사용하고 있었다는 점이었습니다.

이 글은 특정 컴포넌트를 처음부터 원인으로 단정하지 않고, <b>외부 요청 → Gateway data plane → Kubernetes routing → 애플리케이션</b> 경계를 차례로 분리해 확인한 과정을 정리한 기록입니다.

## 먼저 결론부터

이번 장애에서 확인된 직접적인 장애 상태는 다음과 같습니다.

| 계층 | 장애 당시 확인 결과 |
| --- | --- |
| 애플리케이션 Pod | 정상 |
| Service / EndpointSlice | 정상 |
| HTTPRoute status | `Accepted=True`, `ResolvedRefs=True` |
| NGINX Gateway data plane | 실제 backend 대신 503 fallback upstream 사용 |
| 복구 방법 | NGF controller 재시작 후 설정 재전송 |

다만 Agent의 주기적인 `Credential watcher` 메시지를 최초 원인이라고 확정할 수는 없었습니다. 관련 공개 이슈의 유지보수자는 이 메시지가 인증서 변경뿐 아니라 Kubernetes ServiceAccount token rotation으로도 발생할 수 있으며, 약 1시간 주기는 예상 가능한 동작일 수 있다고 설명했습니다.

따라서 이 글에서 확정적으로 말할 수 있는 것은 <b>Route 상태와 실제 rendered upstream 상태가 불일치했다</b>는 점입니다. 재연결과 endpoint resolution 사이의 경합은 유력한 조사 가설이지만, 아직 코드 수준의 root cause로 단정하지 않습니다.

## 1. 장애는 애플리케이션 오류처럼 보였다

외부 도메인으로 API를 호출하자 503이 발생했습니다. 가장 자연스러운 가설은 다음과 같았습니다.

- 새 애플리케이션 이미지가 정상 기동하지 않았다.
- Service가 잘못된 포트나 selector를 가리킨다.
- EndpointSlice가 비어 있거나 Ready endpoint가 없다.
- Gateway가 backend와 연결하지 못한다.

하지만 최초 503 시각은 새 애플리케이션 Pod가 시작하기 전이었습니다.

| 시각(UTC) | 관찰 내용 |
| --- | --- |
| 2026-09-04 02:20:21 | 최초 기록된 외부 503 |
| 2026-09-04 03:44:52 | 새 애플리케이션 Pod 시작 |
| 2026-09-04 03:58:55~03:58:59 | NGF controller Pod 시작 |
| 2026-09-04 03:59:12~03:59:17 | 두 data-plane agent 연결 및 초기 설정 전송 |

따라서 새 애플리케이션 이미지가 최초 503을 만들었다는 가설은 시간 순서만으로도 배제할 수 있었습니다.

## 2. 먼저 503의 응답 주체를 찾는다

503이라는 상태 코드만으로는 애플리케이션이 반환한 것인지, Gateway가 반환한 것인지 알 수 없습니다. 첫 번째 확인 대상은 NGINX Gateway data plane의 access log였습니다.

장애 시 두 data-plane Pod가 동일한 요청에 대해 503을 반환했고, 다음과 같은 시간이 기록되어 있었습니다.

```text
request_time:          0.000
upstream_response_time: 0.000
```

이 값은 backend 애플리케이션이 처리한 뒤 503을 반환한 상황과 다릅니다. NGINX가 upstream 연결을 시도하기 전, 또는 내부 fallback 경로에서 즉시 응답했을 가능성을 보여줍니다.

이 단계에서 문제의 범위를 다음처럼 좁힐 수 있었습니다.

```text
외부 요청
  -> NGINX Gateway data plane에서 즉시 503
  -> 애플리케이션까지 요청이 전달되지 않았을 가능성
```

## 3. Prometheus가 정상이어도 외부 경로는 실패할 수 있다

다음으로 503 발생 시각의 Kubernetes 상태를 확인했습니다.

- 애플리케이션 Pod `Ready=True`
- Prometheus scrape target `up=1`
- Service에 정상적인 selector 존재
- EndpointSlice에 Pod IP와 Service port 존재
- Endpoint가 `ready=true`
- Service와 Pod에 대한 클러스터 내부 직접 요청은 HTTP 200

이 결과는 애플리케이션 프로세스가 죽었거나, Service endpoint가 사라졌거나, 애플리케이션이 모든 요청에 503을 반환한 상황과 맞지 않았습니다.

여기서 중요한 관측성의 차이는 다음과 같습니다.

```text
Prometheus up=1
  = 애플리케이션의 scrape 경로가 정상

외부 도메인 HTTP 200
  = DNS, Load Balancer, Gateway, Route, Service, Endpoint,
    애플리케이션 전체 경로가 정상
```

`up=1`은 외부 사용자 경로의 성공을 보장하지 않습니다. Gateway 앞단에 외부 blackbox HTTP 확인이 별도로 필요한 이유입니다.

## 4. 생성된 NGINX 설정이 결정적인 증거였다

상태 객체와 메트릭만으로는 “NGINX가 실제로 어느 upstream을 사용하고 있는가”를 알 수 없었습니다. 그래서 data plane 내부의 생성 설정을 확인했습니다.

장애 당시 문제가 된 upstream은 실제 Pod endpoint가 아니라 다음 fallback socket을 가리키고 있었습니다.

```nginx
upstream <namespace>_<service>_80 {
    server unix:/var/run/nginx/nginx-503-server.sock;
}
```

이 socket은 NGINX Gateway Fabric이 backend endpoint를 유효하게 구성하지 못했을 때 503을 반환하기 위한 내부 경로입니다.

컨트롤 플레인을 재시작한 뒤에는 같은 upstream이 다음처럼 바뀌었습니다.

```nginx
upstream <namespace>_<service>_80 {
    server <backend-pod-ip>:8080;
}
```

이 비교를 통해 장애 계층을 다음처럼 특정할 수 있었습니다.

| 장애 계층 | 확인 결과 |
| --- | --- |
| 애플리케이션 | 정상 |
| Service / EndpointSlice | 정상 |
| HTTPRoute status | 정상으로 표시 |
| NGINX data plane | 잘못된 fallback upstream 유지 |

단순히 “Gateway가 backend에 연결하지 못했다”고 표현하는 것보다, <b>생성된 설정의 실제 upstream</b>을 확인하는 편이 훨씬 강한 증거가 됩니다.

## 5. 재연결 성공과 routing 복구는 같은 말이 아니었다

NGINX Gateway Fabric은 controller와 data plane 사이에서 Agent를 통해 설정을 전달합니다. 장애 조사 중 다음과 같은 재연결 로그가 반복되는 것을 확인했습니다.

```text
Credential watcher has detected changes
Closing grpc connection
Agent connected
Starting new subscribe stream after connection reset
```

현재 환경에서도 한 data-plane Pod에서 약 49분 간격으로 watcher 이벤트가 관찰되었습니다. 그러나 공개 이슈의 유지보수자 설명에 따르면 Kubernetes ServiceAccount token rotation으로도 이 메시지가 발생할 수 있습니다. 그러므로 다음 두 사실을 분리해야 합니다.

- Agent gRPC 연결은 다시 수립될 수 있다.
- 연결이 다시 수립되었다고 해서 각 Route의 실제 endpoint routing 상태가 완전히 복구되었다는 뜻은 아니다.

이번 장애에서는 controller를 재시작한 직후 두 Pod가 다시 시작했고, controller 로그에서 다음 흐름이 확인되었습니다.

```text
Starting the NGINX Gateway Fabric control plane
Successfully connected to nginx agent
Sending initial configuration to agent
Successfully configured nginx for new subscription
NGINX configuration was successfully updated
```

이후 실제 upstream이 재생성되고 외부 요청이 200으로 복구되었습니다. 따라서 controller 재시작은 단순한 Pod 교체가 아니라, Kubernetes API에서 상태를 다시 읽고 data plane에 초기 설정을 재전송하는 복구 동작으로 볼 수 있습니다.

## 6. `ResolvedRefs=True`를 어디까지 믿어야 할까?

Gateway API의 `ResolvedRefs=True`는 Route가 참조한 리소스를 해석할 수 있다는 controller의 상태 표현입니다. 하지만 이번 사례에서는 이 상태가 실제 NGINX 설정의 upstream 내용과 일치하지 않았습니다.

즉, 다음과 같은 상태가 동시에 존재했습니다.

```text
HTTPRoute.status.parents.conditions:
  Accepted=True
  ResolvedRefs=True

rendered nginx.conf:
  server unix:/var/run/nginx/nginx-503-server.sock;
```

운영 관점에서 이는 매우 위험합니다. Kubernetes 리소스와 controller status만 보면 배포가 성공한 것처럼 보이지만, 사용자의 요청은 계속 503이기 때문입니다.

이때 확인해야 할 질문은 “Route 객체가 정상인가?”에서 끝나지 않습니다.

1. Route가 참조한 Service에 Ready endpoint가 있는가?
2. Controller가 계산한 endpoint와 data plane이 받은 설정이 같은가?
3. 생성된 `nginx.conf`의 upstream이 실제 endpoint를 가리키는가?
4. 외부 blackbox 요청이 실제 hostname으로 200을 반환하는가?

## 7. 비슷한 공개 이슈와 이번 사례의 차이

NGINX Gateway Fabric 공개 이슈 [#5658](https://github.com/nginx/nginx-gateway-fabric/issues/5658)도 `nginx-503-server.sock`과 `ResolvedRefs=True`를 함께 다루고 있습니다.

다만 후속 댓글에서 해당 신고자는 DNS가 이전 ingress를 가리킨 것이 원인이었을 가능성을 확인했고, 다른 환경에서는 재현되지 않았다고 답변했습니다. 따라서 같은 문자열이 등장한다는 이유만으로 두 사례를 동일한 원인으로 묶으면 안 됩니다.

이번 사례에서 확인한 차이는 다음과 같습니다.

- 활성 NGF data plane의 생성 설정 자체에서 fallback upstream을 확인했다.
- 장애 대상 Service와 EndpointSlice에 Ready endpoint가 존재했다.
- 외부 요청의 `request_time`과 `upstream_response_time`이 0.000이었다.
- NGF controller 재시작 직후 초기 설정 전송과 설정 적용 성공이 기록되었다.
- 재시작 후 실제 backend endpoint upstream으로 바뀌며 정상화되었다.

이 조사 결과를 바탕으로 [NGINX Gateway Fabric Issue #5869](https://github.com/nginx/nginx-gateway-fabric/issues/5869)를 등록했습니다. 이슈에는 내부 IP와 계정 정보는 제외하고, 환경 버전·상태·설정 형태·복구 로그·확인 질문만 담았습니다.

## 8. 장애 대응 순서

유사한 503이 발생하면 다음 순서로 확인하는 것이 효율적입니다.

```text
1. 외부 hostname의 HTTP 상태와 최초 발생 시각 기록
2. NGINX access log에서 status, request_time, upstream_response_time 확인
3. HTTPRoute와 Gateway status 확인
4. Service와 EndpointSlice의 endpoint 및 ready 상태 확인
5. Service/Pod 직접 요청으로 애플리케이션 경로 확인
6. data plane의 nginx -T에서 실제 upstream 확인
7. controller/Agent의 연결 재설정과 config apply 로그 대조
8. Pod 시작 시각, restart counter, Deployment 변경 시각 비교
9. 필요하면 EKS audit log로 Kubernetes API 변경 주체 확인
10. 외부 요청을 다시 확인하고 복구 여부를 검증
```

여기서 6번이 핵심입니다. `kubectl get httproute`가 정상이라는 이유만으로 NGINX routing 상태가 정상이라고 결론 내리면 안 됩니다.

## 9. 운영 환경에 추가할 관측성

이번 장애는 일반적인 Kubernetes health check만으로는 감지하기 어려웠습니다. 다음 항목을 함께 구성하는 것이 좋습니다.

### 외부 blackbox HTTP 모니터링

애플리케이션 scrape와 별개로 실제 사용자 hostname을 호출해야 합니다. HTTP 5xx뿐 아니라 응답 시간과 body 형태도 함께 기록하면 NGINX 내부 fallback인지 backend 오류인지 구분하는 데 도움이 됩니다.

### NGINX access log 보존

최소한 다음 값을 Loki나 CloudWatch에 보존해야 합니다.

```text
host
status
request_time
upstream_response_time
upstream_addr
request_id
```

### 생성 설정 스냅샷

장애가 사라진 뒤 `nginx -T`를 실행하면 정상 설정만 남을 수 있습니다. 배포나 Route 변경 직후, 그리고 5xx 발생 시점에 생성 설정을 스냅샷으로 남기는 것이 좋습니다.

### Agent 연결 이벤트 알림

`connection closing`, `connection reset`, `subscribe stream`, `Config apply failed`, `rollback` 같은 이벤트를 알림 대상으로 삼을 수 있습니다. 다만 연결 재수립 성공만으로 정상화를 판정하지 말고, 실제 외부 HTTP와 upstream 상태까지 확인해야 합니다.

### EKS audit logging

누가 언제 HTTPRoute, Service, EndpointSlice 또는 관련 Secret을 변경했는지 확인하려면 EKS control-plane audit logging이 필요합니다. 이 기능은 활성화 이후의 이벤트만 수집하므로 장애가 발생한 뒤 소급해서 과거 변경 이력을 복원할 수는 없습니다.

## 마무리

이번 장애의 교훈은 503의 원인을 애플리케이션에서 시작할 필요가 없다는 뜻이 아닙니다. 더 정확하게는, <b>503이라는 결과와 그 결과를 만든 계층을 분리해서 확인해야 한다</b>는 뜻입니다.

Pod가 Ready이고 Prometheus가 `up=1`이어도 외부 경로는 실패할 수 있습니다. Route가 `ResolvedRefs=True`여도 data plane의 upstream이 실제 endpoint를 사용한다는 보장은 이번 사례에서 확인되지 않았습니다.

따라서 Kubernetes Gateway를 운영할 때는 다음 세 가지를 함께 봐야 합니다.

```text
Kubernetes status
  + generated NGINX configuration
  + real external HTTP request
```

이 세 가지가 모두 일치할 때 비로소 “배포가 정상적으로 외부에 반영되었다”고 말할 수 있습니다.

### 참고 자료

- [NGINX Gateway Fabric Issue #5658](https://github.com/nginx/nginx-gateway-fabric/issues/5658)
- [이번 장애 문의 Issue #5869](https://github.com/nginx/nginx-gateway-fabric/issues/5869)
- [NGINX Gateway Fabric Troubleshooting](https://docs.nginx.com/nginx-gateway-fabric/troubleshooting/)
- [Amazon EKS control plane logging](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html)
- [Amazon EKS auditing and logging best practices](https://docs.aws.amazon.com/eks/latest/best-practices/auditing-and-logging.html)
