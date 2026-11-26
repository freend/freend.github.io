---
title: "[Cloud Native] 쿠버네티스 리소스 관리의 물리적 원리: CPU는 '공간'이 아니라 '시간'이다"
description: "CPU를 시간의 관점에서 이해하고, CPU Request와 Limit의 차이부터 Memory와 Ephemeral Storage까지 실제 운영 경험을 바탕으로 리소스 관리 전략을 설명합니다."
pubDate: "Aug 20 2026"
heroImage: "/src/assets/blog-placeholder-3.jpg"
---

쿠버네티스 클러스터를 운영하다 보면 가장 많이 마주치는 질문이 있습니다. 
<b>"CPU 사용률은 낮은데 왜 서비스 응답 속도는 느릴까요?"</b> 혹은 <b>"메모리는 왜 조금만 넘쳐도 바로 팟(Pod)이 죽어버릴까요?"</b>

이 질문에 답하기 위해서는 쿠버네티스가 리소스를 관리하는 방식을 이해해야 합니다. 특히 CPU는 우리가 흔히 생각하는 '공간'이 아니라 '시간(Time-slicing)'의 관점에서 이해해야 합니다. 이 관점에서 CPU Request와 Limit이 실행 과정에서 어떤 역할을 하는지 살펴보겠습니다.

---

## 1. 🧊 자원별 핵심 특성 요약

| 구분 | 리소스 유형 | 관리 단위 | 부족 시 현상 | 비유 |
| :--- | :--- | :--- | :--- | :--- |
| <b>CPU</b> | <b>시간 (Time)</b> | CPU 단위 (core, m) | 경합 시 처리 지연, Limit에 따른 Throttling | 전화 상담원과 VIP 고객 |
| <b>Memory</b> | <b>공간 (Space)</b> | Byte | Limit 초과 시 OOM 가능, 노드 메모리 압박 시 Eviction 가능 | 부동산과 방 크기 |
| <b>Storage</b> | <b>디스크 공간 (Disk)</b> | Byte | DiskPressure, Pod Eviction, 쓰기 실패 | 창고의 적재 공간 |

---

## 2. 🕒 CPU: "1시간에 단 18초만 말할 수 있다면?"

Kubernetes는 CPU를 요청량과 상한으로 관리합니다. <b>CPU Request</b>는 Pod 배치와 경합 시 상대적인 배분 기준에, <b>CPU Limit</b>은 실행 중 사용할 CPU 용량의 상한에 영향을 줍니다. Linux에서는 컨테이너 런타임이 cgroup을 통해 Limit을 적용할 수 있습니다.

> 🔗 <b>공식 문서 참고</b>: [Kubernetes - How Pods with resource limits are run](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#how-pods-with-resource-limits-are-run)

### 📞 VIP 전화기 비유
*   <b>전화 상담원</b>: 처리할 수 있는 연산 용량.
*   <b>일반 고객과 VIP 고객</b>: Request는 상담 요청이 몰릴 때 적용되는 <b>상대적인 배분 기준</b>입니다. VIP는 상대적으로 더 많은 상담 기회를 받을 수 있지만, 항상 먼저 상담하거나 정해진 비율을 보장받는 것은 아닙니다.
*   <b>상담 시간 한도</b>: Limit은 작업이 사용할 수 있는 CPU의 상한입니다. 시간 잔액을 모두 사용하면 다음 잔액이 생길 때까지 기다리는 상황에 비유할 수 있습니다.

### ⚙️ 1.0 CPU의 의미: "코어 하나의 처리 용량"

Kubernetes에서 <b>CPU: 1</b>은 노드가 물리 머신인지 가상 머신인지에 따라 물리 또는 가상 CPU 코어 하나의 처리 용량을 뜻합니다. CPU: 1을 설정했다고 특정 Pod가 코어 하나를 독점하는 것은 아닙니다.

*   <b>코어 (Core)</b>: 16코어 노드는 CPU 작업을 여러 코어에서 병렬로 처리할 수 있지만, 실제 병렬성은 애플리케이션과 실행 환경에 따라 달라집니다.
*   <b>시간에 따른 사용</b>: <b>CPU: 1</b>은 시간에 걸쳐 사용할 수 있는 CPU 용량을 나타냅니다. Request와 Limit, 노드의 경합 상태에 따라 실제 사용 기회와 처리 시간은 달라집니다.

### 📉 낮은 Request(5m)는 어떤 영향을 줄까? (시간의 계산 및 비교)

상담원이 한가할 때는 일반 고객도 충분히 상담할 수 있습니다. 하지만 상담 요청이 몰리면 VIP 고객에게 상대적으로 더 많은 상담 기회가 가듯, CPU 경합 중에는 Request가 높은 작업에 더 많은 처리 기회가 배분될 수 있습니다. `5m`와 `20m`은 각각 CPU 0.005개와 0.02개의 요청량입니다.

| 설정값 | 100ms 기준 산술 환산 | 1초 기준 산술 환산 | 1시간 기준 산술 환산 |
| :--- | :--- | :--- | :--- |
| <b>5m</b> (0.005) | <b>0.5ms</b> | 5ms | 18초 |
| <b>20m</b> (0.02) | <b>2.0ms</b> | 20ms | 72초 (1분 12초) |

표의 시간은 CPU 단위를 시간 비율로 이해하기 위한 산술 환산입니다. Request가 매 구간마다 이 시간을 고정 할당한다는 뜻은 아닙니다.

#### 🗣️ "말문이 막히는" 실제 사례
애플리케이션이 특정 로직을 처리하며 "데이터 처리를 시작합니다..."라고 말하는 데 CPU 시간 <b>1ms</b>가 필요하다고 가정해 봅시다.

*   <b>5m 요청 시</b>: 상담원이 한가할 때는 필요한 처리를 할 수 있지만, 요청이 몰리면 상대 배분 기준이 낮아 처리 기회를 덜 받을 수 있습니다.
*   <b>20m 요청 시</b>: 5m보다 높은 상대 배분 기준이 적용되어, CPU 경합 상황에서 더 많은 처리 기회를 받을 수 있습니다.

제 운영 환경에서는 이런 경합 상황을 고려해 최소 CPU Request를 <b>20m</b>로 두었습니다.

---

### 🛠️ CPU Limit의 시간 잔액: Quota와 Period

Request와 달리 CPU Limit은 일정 구간에 사용할 수 있는 CPU 시간 총량을 제한합니다. Linux cgroup의 CPU bandwidth control에서는 이 총량을 <b>Quota</b>, 갱신되는 구간을 <b>Period</b>라고 부릅니다.

예를 들어 <b>Limit 500m</b>, <b>Period 100ms</b>라고 가정하면 한 주기에 사용할 수 있는 Quota는 총 <b>50ms의 CPU 시간</b>입니다. 이 50ms는 항상 연속해서 실행되는 시간이 아니라 주기 동안 누적되는 CPU 시간입니다.

상담원이 주기 초반부터 계속 통화해 시간 잔액을 다 쓰면 다음 갱신까지 기다리는 것처럼, 실행 중 Quota를 모두 소진한 cgroup은 Period가 갱신될 때까지 throttling될 수 있습니다. 다음 주기에 Quota가 갱신되면 다시 CPU를 사용할 수 있습니다. 여러 실행 흐름이 병렬로 CPU를 사용하면 잔액이 더 빨리 소진될 수도 있습니다. 100ms는 이 예시에서 가정한 Period이며, 모든 환경의 고정 주기는 아닙니다.

---

## 3. 🏠 메모리: "부동산과 방 크기" (Space-Based)

메모리는 CPU처럼 처리 시간을 나누는 자원이 아니라 사용 중인 메모리 공간의 크기로 관리합니다.

### 🏢 방 배정 비유
*   <b>Requests</b>: 스케줄러가 Pod를 배치할 때 참고하는 <b>방 크기 기준</b>입니다. 요청한 용량만큼의 물리 메모리를 독점하도록 미리 떼어 놓는다는 뜻은 아닙니다.
*   <b>Limits</b>: 컨테이너가 사용할 수 있도록 정한 <b>최대 평수</b>입니다.
*   <b>OOM (Out of Memory)</b>: 메모리 사용량이 Limit을 넘으면 커널이 메모리 압박을 감지했을 때 프로세스를 종료할 수 있습니다. 초과 즉시 항상 종료되는 것은 아니며, 노드 메모리 압박에 따른 Pod Eviction도 별도로 발생할 수 있습니다.

<b>CPU는 부족하면 처리 속도가 느려지지만, 메모리는 부족하면 프로세스가 종료될 수 있습니다.</b> 이것이 제가 메모리 관리에 더 많은 여유를 두는 이유입니다.

---

## 4. 📦 스토리지: "창고와 적재량" (Disk-Based)

스토리지는 메모리와 달리 로컬 디스크 공간을 사용하며, 장기 보존을 보장하지 않는 <b>'임시 창고'</b>와 같습니다. 여기서 구분할 점은 <b>노드 디스크 사용량</b>과 <b>Pod의 ephemeral-storage 사용량</b>이 같지 않다는 것입니다.

> 🔗 <b>공식 문서 참고</b>: [Kubernetes - Local ephemeral storage](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#local-ephemeral-storage)

로컬 ephemeral storage 용량 격리는 Kubernetes <b>v1.25(2022년)</b>에 GA가 됐습니다. 이 시점은 기능의 안정화 단계이며, 그전까지 디스크 관리가 방치됐다는 뜻은 아닙니다. 컨테이너 이미지, 로그, writable layer와 임시 파일은 노드의 로컬 파일시스템을 사용하므로 CPU·메모리와 함께 디스크 여유도 살펴야 합니다.

### 📝 기술적 장부(Ledger) vs 물리적 실체(Truth)

Kubernetes가 Pod 단위로 계산하는 사용량과 노드 파일시스템의 여유 공간은 서로 다른 기준입니다.

*   <b>Pod 사용량</b>: 지원되는 파일시스템 구성에서 kubelet은 디스크 기반 `emptyDir`, 컨테이너 로그, writable layer 등을 추적합니다. 어떤 파일시스템을 관찰하는지에 따라 측정 범위가 달라집니다.
*   <b>노드 디스크 사용량</b>: 이미지 레이어도 이미지 파일시스템 등 노드 디스크를 사용하며, 같은 레이어를 여러 이미지나 컨테이너가 재사용할 수 있습니다. Pod별 ephemeral-storage Request가 이 물리 공간을 독점 예약하는 것은 아닙니다.

스케줄러는 실제 디스크 사용량 전체를 실시간 예약 장부처럼 사용하는 대신 Pod의 Request와 노드의 allocatable을 기준으로 배치합니다. 따라서 실제 사용량이 Request를 웃돌거나 이미지 파일시스템의 여유가 줄어들면, Request만으로는 DiskPressure를 예방할 수 없습니다.

### 💡 실전 에피소드: 배포 중단으로 깨달은 스토리지의 중요성

스토리지를 리소스 설정에 포함해야 한다는 사실을 전혀 몰라 생략했다가, 배포가 중단되는 상황을 겪은 적이 있습니다. 당시 발생했던 문제는 <b>DiskPressure</b> 상태로 인한 포드 스케줄링 실패였죠.

*   <b>원인</b>: "이미지는 장부에 안 들어가니까 무시해도 되겠지?"라고 생각한 것이 화근이었습니다. 하지만 <b>이미지 원본(Origin)은 엄연히 공간을 차지하며, 압축 해제 오버헤드</b>까지 발생하면 물리 디스크는 금세 한계에 도달합니다.

이 사건을 계기로 저는 <b>[Ephemeral Storage Governance]</b> 전략을 수립했습니다.

> 🛠️ <b>제가 정한 Ephemeral Storage 산정 기준</b>
> <b>Request = (이미지 실제 크기 * 1.5) + 런타임 버퍼</b>
> *   이미지 크기가 1.5GB라면 Request를 같은 1.5GB로 두는 것이 아니라, 이 경험식에 따라 `(1.5GB * 1.5) + 런타임 버퍼`로 산정합니다.
> *   이 값은 스케줄러가 배치할 때 참고하는 요청량이지, 해당 디스크 공간을 Pod에 독점 예약하거나 DiskPressure를 막아주는 보장은 아닙니다.

---

## 5. 💡 최적화 전략: "안정적인 바닥 깔기"
### ❓ 왜 많은 프로젝트에서 스토리지를 설정하지 않을까?

실제로 많은 매니페스트에서 `cpu`와 `memory`는 설정하면서도 `ephemeral-storage`는 빼먹는 경우가 많습니다. 그 이유는 보통 다음과 같습니다.

1.  <b>노드 용량에 대한 막연한 믿음</b>: "설마 디스크가 부족하겠어?"라는 생각으로 기본값에 의존합니다.
2.  <b>로그 관리의 외주화</b>: 로그를 표준 출력(stdout)으로 쏘고 외부(CloudWatch, ELK 등)로 바로 보내기 때문에, 로컬에 저장되는 데이터가 거의 없다고 판단하기 때문입니다.
3.  <b>휘발성 특성</b>: 어차피 팟이 죽으면 사라질 데이터라는 생각에 중요도를 낮게 평가합니다.

앞서 겪은 DiskPressure는 요청량만 설정하면 실제 노드 디스크까지 보호된다고 볼 수 없다는 점을 보여줬습니다. Request는 배치 기준이고, Limit은 지원되는 구성에서 추적되는 Pod 사용량이 한도를 넘었을 때 Eviction을 유발할 수 있지만, 어느 쪽도 노드의 DiskPressure를 무조건 방지하지는 않습니다.

---

### 📝 실전! 리소스 설정 예시

아래는 리소스 설정 부분을 보여주는 발췌 예시입니다. 완전한 Deployment 매니페스트가 아닙니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-stable-app
spec:
  template:
    spec:
      containers:
      - name: app-container
        image: my-app-image:1.0.0 # 이미지 크기가 1GB라고 가정
        resources:
          requests:
            # [운영 기준] 최소 Request 20m 이상
            cpu: "50m"
            # [안정적 점유] 평균 사용량 수준
            memory: "256Mi"
            # [요청량 기준] (이미지 1GB * 1.5) + 버퍼 500MB ≈ 2GB, 올림해 2Gi
            ephemeral-storage: "2Gi"
          limits:
            # [폭발적 연산 허용]
            cpu: "500m"
            # [예시 기준] 평균 사용량의 1.5~2배
            memory: "512Mi"
            # [Pod 사용량 상한] 추적되는 로그·임시 파일 사용량 대비
            ephemeral-storage: "4Gi"
```

---

## 6. 마무리하며: '시간과 공간'의 관점으로 리소스를 바라보기

우리처럼 노드에 팟을 밀집해서 배치(Overcommit)하는 환경에서는 다음 세 가지 원칙을 기억하세요.

1.  <b>CPU는 시간의 관점에서 본다</b>: 저는 CPU 경합 시 상대 배분 기준이 너무 낮아지지 않도록 최소 CPU Request를 `20m`으로 두었습니다.
2.  <b>메모리는 생명선이다</b>: OOM 위험을 고려해 평균 사용량보다 항상 <b>30~50%</b> 이상의 여유를 `Limit`에 두는 것이 제 운영 기준입니다.
3.  <b>오버커밋의 미학</b>: 저는 <b>Requests</b> 합계를 노드 용량의 80~90% 내로 관리했습니다. <b>Limits</b>는 노드 용량을 넘겨 설정할 수 있지만, CPU Throttling과 OOM, 디스크 사용량 및 DiskPressure를 함께 살펴야 합니다.

저는 CPU는 <b>시간</b>의 관점에서, Memory는 <b>공간과 사용 한도</b>의 관점에서, Storage는 Pod의 사용량과 실제 <b>노드 디스크 공간</b>을 함께 고려해 운영 기준을 세웠습니다.

---
*Created by Gemini CLI - Infrastructure Knowledge Distillation (2026-08-20)*
