---
title: "Pod는 살아 있는데 API는 실패한다 — Kubernetes 환경에서의 JVM OOM 대응"
description: "Kubernetes 운영 환경에서 JVM Heap OOM이 발생했음에도 컨테이너가 종료되지 않았던 사례와 Liveness Probe의 감지 범위, ExitOnOutOfMemoryError를 활용한 운영 대응을 정리한다."
pubDate: "Nov 26 2026"
---

## 장애 발생

12:25에 대용량 데이터 로드 시 모든 API 요청이 504를 반환하는 장애를 확인했다.

문제는 JVM에서 OOM이 발생했음에도 컨테이너가 종료되지 않았다는 점이다.<br />
Pod는 계속 실행 중이었지만 여러 API 요청이 실패하고 있었으며, 컨테이너의 자동 재시작도 발생하지 않았다.<br />
그래서 Pod의 문제가 아니라 네트워크 문제로 예상해 파악하던 중, 컨테이너에서 아래와 같은 로그를 발견했다.

```text
Exception in thread "http-nio-80-Acceptor" java.lang.OutOfMemoryError: Java heap space
Exception in thread "http-nio-80-Poller" java.lang.OutOfMemoryError: Java heap space
Exception in thread "http-nio-80-exec-16" java.lang.OutOfMemoryError: Java heap space
...
Handler dispatch failed: java.lang.OutOfMemoryError: Java heap space
```

## JVM OOM과 Kubernetes OOMKilled의 차이

Kubernetes 환경에서 Java 애플리케이션을 운영할 때 JVM 내부에서 발생하는 OOM과 컨테이너 메모리 제한에 따른 OOMKilled는 구분할 필요가 있다.

| 구분 | JVM Heap OOM | 컨테이너 메모리 제한에 따른 OOMKilled |
|---|---|---|
| 발생 주체 | JVM | Linux Kernel |
| 발생 원인 | JVM Heap 메모리 할당 실패 | 컨테이너 메모리 한도에 따른 OOM Kill |
| 대표적인 오류 | Java heap space | OOMKilled |
| JVM 프로세스 종료 | 반드시 종료되지는 않음 | JVM 프로세스가 Kill 대상이면 종료 |
| 컨테이너 재시작 | 프로세스 종료 또는 별도의 재시작 조건 필요 | 컨테이너 종료 후 재시작 정책에 따라 동작 |

JVM에서 `OutOfMemoryError`가 발생했다고 해서 JVM 프로세스 전체가 반드시 종료되는 것은 아니다.

이번 사례에서도 JVM 프로세스는 살아 있었지만, 여러 요청에서 메모리 할당에 실패하고 있었다.

즉, 컨테이너의 실행 상태와 애플리케이션의 정상 동작 여부가 일치하지 않는 상황이었다.

## Liveness Probe는 왜 컨테이너를 재시작하지 않았을까?

당시 Kubernetes에는 Liveness Probe와 Readiness Probe가 모두 설정되어 있었다.

```yaml
livenessProbe:
  failureThreshold: 12
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 10
  successThreshold: 1
  timeoutSeconds: 2

readinessProbe:
  failureThreshold: 12
  httpGet:
    path: /
    port: 80
    scheme: HTTP
  initialDelaySeconds: 10
  periodSeconds: 10
  successThreshold: 1
  timeoutSeconds: 2
```

두 Probe 모두 `/` 경로를 10초마다 검사하도록 설정되어 있으며, 연속 12회 실패해야 각각의 조치가 이루어지도록 구성되어 있었다.

- Liveness Probe: 컨테이너 재시작
- Readiness Probe: Pod를 서비스 트래픽 대상에서 제외

이 설정에서는 Liveness Probe가 지속적으로 실패하면 재시작 조건을 충족하기까지 약 2분이 소요될 수 있다.

또한 HTTP 기반의 Probe는 JVM 내부에서 발생한 OOM 자체를 직접 감지하지 않는다.

일반 API 요청에서 OOM이 발생하더라도 Probe 요청이 정상적으로 처리되거나, 중간에 성공하여 연속 실패 조건을 충족하지 못한다면 컨테이너는 계속 실행될 수 있다.

당시 Probe의 개별 응답 결과는 확인되지 않았으므로, 자동 재시작이 발생하지 않은 정확한 이유를 단정할 수는 없다.

다만 기존 HTTP 기반의 상태 검사만으로는 JVM 내부에서 발생하는 OOM에 즉시 대응하기 어려울 수 있다는 점을 알 수 있었다.

## 운영 대응 — ExitOnOutOfMemoryError

당시 발생한 문제는 조치했지만, 대용량 데이터를 처리하는 과정에서 같은 현상이 다른 서비스에서도 발생할 수 있다고 판단했다.

이에 다음 JVM 옵션을 운영 환경에 추가했다.

```text
-XX:+ExitOnOutOfMemoryError
```

이 옵션은 이번 사례와 같은 JVM Heap OOM이 발생했을 때 JVM 프로세스를 종료하도록 하는 옵션이다.

JVM 종료가 컨테이너 종료로 이어지는 실행 구조에서는 Kubernetes Deployment의 `restartPolicy: Always`에 따라 kubelet이 컨테이너를 다시 시작한다.

### 기존 동작

```text
대용량 데이터 로드
        ↓
JVM Heap OOM 발생
        ↓
JVM 프로세스 생존
        ↓
컨테이너 실행 유지
        ↓
API 요청 실패
        ↓
자동 재시작 미발생
```

### 옵션 적용 후 기대 동작

```text
대용량 데이터 로드
        ↓
JVM Heap OOM 발생
        ↓
ExitOnOutOfMemoryError
        ↓
JVM 프로세스 종료
        ↓
컨테이너 종료
        ↓
Kubernetes 컨테이너 재시작
```

기존 Liveness Probe가 애플리케이션의 외부 응답을 기준으로 장애를 감지한다면, 이번에 추가한 JVM 옵션은 JVM 내부에서 발생한 메모리 오류를 기준으로 프로세스를 종료하도록 한다.

기존 Probe를 대체하는 것이 아니라, Probe만으로 즉시 감지하기 어려운 JVM 내부 장애에 대한 대응을 추가한 것이다.

이 방식은 OOM 자체를 예방하거나 메모리 부족의 근본 원인을 해결하는 방법은 아니다. 심각한 메모리 오류가 발생했을 때 프로세스를 종료하여 Kubernetes의 자동 복구 메커니즘이 동작할 수 있도록 하는 Fail-fast 방식의 운영 대응이다.

또한 컨테이너가 재시작되더라도 동일한 메모리 부족 상황이 반복된다면 장애가 다시 발생할 수 있다.

## 대용량 데이터 처리 구조 개선

대용량 데이터 처리 기능은 API 서버에서 분리하여 AWS Lambda 기반으로 별도 개발하고 이전하기로 했다.

이는 대용량 데이터 처리 작업이 일반 API 요청을 처리하는 JVM에 영향을 주는 구조를 개선하기 위한 것이다.

Lambda 이전 자체가 메모리 부족 문제의 해결을 보장하는 것은 아니지만, 대용량 데이터 처리 작업과 일반 API 요청의 실행 환경을 분리함으로써 장애의 영향 범위를 줄이는 것을 목적으로 한다.

이번 JVM 옵션 적용은 이러한 데이터 처리 구조 개선과는 별개로, 기존 운영 환경에서 OOM 이후에도 컨테이너가 계속 살아 있는 상황에 대비하기 위해 추가한 조치다.

## 마무리

이번 경험을 통해 Kubernetes에서 Pod가 Running 상태라는 사실만으로 애플리케이션이 정상적으로 동작한다고 판단할 수는 없다는 점을 다시 확인했다.

Kubernetes는 컨테이너의 실행 상태와 설정된 Probe를 기반으로 장애를 감지하고 복구한다. 하지만 애플리케이션 내부에서 발생하는 모든 장애를 직접 감지할 수 있는 것은 아니다.

인프라의 자동 복구 기능을 활용하려면 애플리케이션이 장애 발생 이후 어떤 상태로 남는지, 그리고 해당 장애를 인프라가 어떻게 감지할 수 있는지도 함께 고려해야 한다.

이번 대응은 JVM 내부의 장애 감지와 Kubernetes의 자동 복구 메커니즘을 연결하기 위한 운영 환경 개선이었다.

## 참고 문서

- [Oracle JDK 8u92 Release Notes](https://www.oracle.com/java/technologies/javase/8u92-relnotes.html)
- [Kubernetes — Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes — Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
