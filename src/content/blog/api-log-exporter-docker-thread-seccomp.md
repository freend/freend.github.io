---
title: "[Docker] HTTP 요청 하나에 서버가 멈췄다: Python thread 생성 실패와 seccomp 추적기"
description: "Docker Compose 개발서버에서 HTTP 요청 직후 thread 생성이 실패한 원인을 자원 제한부터 seccomp 호환성까지 단계적으로 좁혀간 기록입니다."
pubDate: "Sep 19 2026"
heroImage: "/src/assets/blog-placeholder-5.jpg"
---

Docker Compose로 개발 서버를 띄운 뒤 컨테이너가 정상적으로 기동하는 것까지 확인했습니다. 이미지 빌드도 성공했고, 프로세스도 살아 있었습니다.

그런데 HTTP 요청을 하나 보내자 서버가 응답하지 않았습니다.

```text
RuntimeError: can't start new thread

curl: (52) Empty reply from server
```

처음에는 흔히 생각하는 원인부터 의심했습니다. 프로세스 수 제한, 메모리 부족, Docker의 PID 제한, 호스트의 thread 상한이 차례로 후보에 올랐습니다. 하지만 여러 제한을 확인한 결과, 자원은 남아 있었습니다.

최종적으로 문제를 재현한 경계는 애플리케이션 코드가 아니라 **Docker 기본 seccomp 프로파일과 Python/glibc의 thread 생성 경로 사이**였습니다.

다만 이 글에서 seccomp 프로파일 내부의 단일 syscall까지 근본 원인으로 확정하지는 않습니다. 동일 이미지를 `seccomp=unconfined`로 실행했을 때 문제가 사라지는 것까지 검증했고, 그 결과를 바탕으로 호환성 문제의 범위를 좁힌 기록입니다.

## 먼저 확인한 증상

개발 서버의 Compose 빌드와 컨테이너 기동은 정상적으로 끝났습니다. 문제는 요청을 처리하려는 순간 발생했습니다.

```text
HTTP request
  -> Python handler
  -> new thread creation
  -> RuntimeError
  -> response 전에 프로세스가 연결을 종료
```

컨테이너 내부에서 health endpoint를 호출하면 클라이언트에는 `Empty reply from server`가 보였습니다. 이 메시지만 보면 네트워크, 포트 매핑, 애플리케이션 예외를 모두 의심할 수 있습니다.

하지만 핵심은 서버가 HTTP 상태 코드를 반환하지 못했다는 점입니다. 요청 처리 중 thread를 만들려다 예외가 발생했고, 응답을 작성하기 전에 연결이 끊긴 상황에 가까웠습니다.

## 자원 부족부터 배제하기

`can't start new thread`라는 문구는 자원 부족처럼 보입니다. 그래서 애플리케이션 설정을 바꾸기 전에 호스트와 컨테이너의 제한을 확인했습니다.

### 호스트에서 확인한 항목

- 현재 프로세스와 thread 수
- 사용자별 `ulimit -u`
- 커널의 `threads-max`
- `pid_max`
- 사용 가능한 메모리

여기서 thread가 상한에 도달했거나 메모리가 고갈된 정황은 발견되지 않았습니다.

### 컨테이너에서 확인한 항목

컨테이너의 PID cgroup도 확인했습니다.

```text
pids.max     = max
pids.current = 3
```

컨테이너에는 소수의 프로세스만 존재했고, PID 제한에 걸린 상태도 아니었습니다. 따라서 다음과 같은 단순한 가설은 우선순위에서 내려놓을 수 있었습니다.

- 호스트의 thread 상한 초과
- 컨테이너 PID 제한 초과
- 프로세스가 너무 많이 생성된 상태
- 메모리 부족으로 인한 일반적인 thread 생성 실패

여기서 중요한 교훈은 오류 메시지를 원인으로 착각하지 않는 것입니다.

```text
can't start new thread
  != 항상 thread 수가 부족하다는 뜻
```

thread 생성이라는 동작이 실패했다는 결과만 보여줄 뿐, 실패를 유발한 정책이나 시스템 경계까지 알려주지는 않습니다.

## 재현 조건을 바꿔 경계를 찾다

다음 단계는 애플리케이션과 이미지를 바꾸지 않고 컨테이너 보안 프로파일만 바꿔 보는 것이었습니다.

기본 Docker seccomp 프로파일로 실행했을 때는 HTTP 요청 처리 과정에서 같은 오류가 재현됐습니다.

반면 동일한 이미지를 다음 옵션으로 실행하면 요청이 정상 처리됐습니다.

```bash
docker run \
  --security-opt seccomp=unconfined \
  ...
```

이 환경에서는 health endpoint가 정상적으로 `HTTP 200 OK`와 응답 JSON을 반환했습니다.

이 비교가 중요했던 이유는 바뀐 조건이 하나뿐이었기 때문입니다.

| 항목 | 기본 seccomp | `unconfined` |
| --- | --- | --- |
| 애플리케이션 이미지 | 동일 | 동일 |
| Python 코드 | 동일 | 동일 |
| 컨테이너 리소스 | 동일 | 동일 |
| HTTP 요청 | 동일 | 동일 |
| 결과 | thread 생성 실패 | `200 OK` |

따라서 문제의 범위는 애플리케이션 로직보다 컨테이너 보안 정책과 실행 환경의 호환성 쪽으로 이동했습니다.

## 왜 seccomp가 thread 생성에 영향을 줄 수 있을까?

seccomp는 컨테이너 프로세스가 호출할 수 있는 시스템 콜을 정책으로 제한합니다. Python에서 thread를 하나 만드는 일은 단순히 언어 런타임 안에서 객체 하나를 생성하는 작업이 아닙니다.

Python 런타임과 glibc는 운영체제에 thread 생성을 요청하고, 이 과정에서 여러 시스템 콜과 저수준 동기화 동작을 거칩니다.

```text
Python threading API
  -> CPython runtime
  -> glibc / pthread
  -> kernel syscall
  -> Docker seccomp filter
```

이 경로에서 Docker 엔진, libseccomp, glibc, 커널의 조합이 어긋나면 애플리케이션은 단순히 “새 thread를 만들 수 없다”는 형태로 실패할 수 있습니다.

이번 사례에서는 기본 프로파일을 사용할 때 실패하고 `unconfined`에서 성공했으므로, 이 계층의 호환성 문제가 강하게 의심됐습니다. 다만 어떤 syscall이 직접 거부됐는지까지 확인하지 않은 상태에서 특정 syscall 하나를 범인으로 단정하면 안 됩니다.

## 개발 환경에 적용한 조치

개발서버의 Compose 설정에는 다음 호환성 옵션을 추가했습니다.

```yaml
services:
  api-log-exporter:
    security_opt:
      - seccomp=unconfined
```

컨테이너는 호스트의 `10370` 포트와 컨테이너의 `8080` 포트를 연결하고 있었습니다.

```yaml
ports:
  - "10370:8080"
```

설정 적용 후 Compose를 다시 기동하고 내부 health endpoint를 호출했습니다.

```bash
docker compose up -d --build
curl http://localhost:10370/health
```

이후 컨테이너는 정상적으로 요청을 처리했고, health response도 `200 OK`로 돌아왔습니다.

## `seccomp=unconfined`를 운영 해법으로 착각하지 않기

이번 조치는 개발서버에서 원인을 분리하고 작업을 계속하기 위한 호환성 우회책입니다. seccomp를 해제하면 컨테이너의 시스템 콜 제한이 사라지므로, 보안 관점에서는 방어 계층을 약화시킵니다.

따라서 운영 환경에 그대로 복사하면 안 됩니다.

운영 배포 전에는 다음 순서로 개선하는 편이 안전합니다.

1. 실제로 거부된 시스템 콜을 audit 로그나 syscall tracing으로 확인한다.
2. 사용 중인 Docker Engine, libseccomp, glibc, 커널 버전을 함께 기록한다.
3. 필요한 시스템 콜만 허용하는 제한적 seccomp 프로파일을 작성한다.
4. 기본 프로파일과 제한적 프로파일에서 thread 생성 및 health check를 반복 검증한다.
5. 이미지와 런타임을 업데이트한 뒤 `unconfined` 제거 가능성을 다시 확인한다.

특히 `unconfined` 상태에서 정상이라는 결과는 “seccomp가 관련 있다”는 강한 증거이지, “모든 seccomp 정책이 문제다”라는 뜻은 아닙니다. 보안 정책을 완전히 해제하는 대신 실패 지점을 관측하고 필요한 권한만 복원해야 합니다.

## 이번 장애에서 배운 것

이번 문제는 애플리케이션 코드 한 줄을 고쳐 해결한 장애가 아니었습니다. 오히려 실행 환경의 경계를 하나씩 분리하면서 원인을 좁혀간 사례에 가깝습니다.

```text
thread 생성 실패
  -> 호스트 자원 확인
  -> 컨테이너 PID 제한 확인
  -> 동일 이미지로 보안 프로파일만 변경
  -> seccomp 경계에서 재현 조건 확인
```

정리하면 다음과 같습니다.

- 컨테이너가 기동했다는 사실만으로 요청 처리까지 정상이라고 볼 수 없다.
- `can't start new thread`는 자원 부족 외에도 런타임·보안 정책 호환성 문제일 수 있다.
- 하나의 설정만 바꾼 A/B 테스트가 원인 범위를 빠르게 좁혀준다.
- `seccomp=unconfined`는 진단과 개발 환경의 임시 우회책이지 운영 기본값이 아니다.
- 운영에서는 재현 가능한 증거를 바탕으로 최소 권한 정책으로 되돌아가야 한다.

컨테이너 환경의 장애를 만났을 때 중요한 질문은 “코드가 잘못됐나?”에서 끝나지 않습니다.

> 이 요청이 애플리케이션에 도착하기 전에, 실행 환경의 어느 경계에서 막혔는가?

이 질문을 기준으로 호스트, cgroup, 런타임, seccomp를 차례로 분리하면 막연한 Docker 오류도 훨씬 빠르게 구조화할 수 있습니다.
