---
title: "[Technical Guide] EKS 로그 수집의 완성: Grafana Alloy API-based 스트리밍"
description: "Filesystem 방식의 한계와 에이전트 무한 루핑을 극복하고 API 기반 스트리밍으로 전환한 실전 기록"
pubDate: "Aug 19 2026"
---

안녕하세요. 이 문서는 AWS EKS 환경에서 Grafana Alloy를 사용하여 로그 수집 시스템을 구축하며 겪은 치열한 시행착오의 기록입니다. 

단순한 설정 공유를 넘어, 왜 특정 방식이 실패했는지와 그 과정에서 마주한 기술적 난제들을 [원인] -> [과정] -> [결과] 순으로 정리했습니다.

---

## 1. [원인] 왜 Filesystem 방식을 포기했는가?

대부분의 블로그와 기술 문서에서는 Promtail처럼 `/var/log/pods` 경로의 파일을 직접 읽는 방식을 권장합니다. 우리 역시 이 '표준'을 따르려 했으나, 두 가지 결정적인 문제에 봉착했습니다.

### 기술적 한계: Regex와 경로 조립의 지옥
EKS의 로그 경로는 네임스페이스, 팟 이름, UID, 컨테이너 이름이 복잡하게 뒤섞여 있습니다. 이를 Filesystem 방식으로 수집하려면 수많은 Regex 조합이 필요합니다.

### 에이전트의 무한 루핑 오류
가장 곤혹스러웠던 점은 **에이전트와의 협업 과정**이었습니다. 참고한 블로그 2곳에서 동일하게 Filesystem 방식을 다루고 있다 보니, 코딩 에이전트가 이 정보만을 학습하여 계속해서 실패하는 2가지 패턴의 코드만 무한 반복해서 생성하는 루핑에 빠졌습니다.

#### [실패했던 Filesystem 설정 예시]
```hcl
discovery.relabel "failed_path_logic" {
  targets = discovery.kubernetes.pods.targets
  rule {
    source_labels = ["__meta_kubernetes_namespace", "__meta_kubernetes_pod_name", "__meta_kubernetes_pod_uid", "__meta_kubernetes_pod_container_name"]
    
    # 실패 사례 1: 단순 조립 시도 (변수 매핑 오류)
    # replacement = "/var/log/pods/$1_$2_$3/$4/*.log" 

    # 실패 사례 2: 구분자(;)와 정밀 Regex 사용 시도 (타이밍 이슈)
    # separator = ";"
    # regex = "(.+);(.+);(.+);(.+)"
    # replacement = "/var/log/pods/${1}_${2}_${3}/${4}/*.log"
  }
}
```
결과적으로 경로는 조립되더라도 EKS의 동적 파일 생성 타이밍을 잡지 못해 `no such file` 에러가 남발되었습니다.

---

## 2. [과정] 공식 문서에서 찾은 새로운 돌파구

에이전트의 루핑을 끊기 위해 '검색된 블로그'가 아닌 **Grafana Alloy 공식 문서(Configure Alloy on Kubernetes)**를 직접 분석하기 시작했습니다. 

그 결과, 파일을 직접 읽는 방식이 아닌 **Kubernetes API 서버로부터 직접 로그 스트림을 가져오는 `loki.source.kubernetes` 컴포넌트**가 존재함을 확인했습니다. 이 방식은 복잡한 경로 조립이나 Regex 없이도 API 레벨에서 깔끔하게 로그를 낚아챌 수 있는 구조였습니다.

---

## 3. [결과] API-based 스트리밍 구현

최종적으로 완성된 설정입니다. 이 구성으로 전환한 즉시 모든 로그 수집이 100% 성공했습니다.

### 3.1. 수집기 설정: `alloy-official-values.yaml`
```hcl
alloy:
  configMap:
    create: true
    content: |
      discovery.kubernetes "pods" { role = "pod" }

      discovery.relabel "local_pods" {
        targets = discovery.kubernetes.pods.targets
        rule {
          source_labels = ["__meta_kubernetes_pod_node_name"]
          regex         = sys.env("K8S_NODE_NAME")
          action        = "keep"
        }
        rule { source_labels = ["__meta_kubernetes_namespace"]; target_label = "namespace" }
        rule { source_labels = ["__meta_kubernetes_pod_name"]; target_label = "pod" }
        rule { source_labels = ["__meta_kubernetes_pod_container_name"]; target_label = "container" }
      }

      # 💡 핵심: Filesystem 대신 API 기반 수집 컴포넌트 사용
      loki.source.kubernetes "pod_logs" {
        targets    = discovery.relabel.local_pods.output
        forward_to = [loki.write.local.receiver]
      }

      loki.write "local" {
        endpoint { url = "http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push" }
        external_labels = { collector = "alloy-official" }
      }
```

### 3.2. 저장소 설정: `loki-values.yaml`
Loki에서 로그를 받아들이기 위해 필요한 최소한의 필수 설정입니다.

```yaml
loki:
  # 💡 트러블슈팅 포인트: "no org id" 에러 해결
  # 프라이빗 클러스터라면 인증(Multi-tenancy)을 끄는 것이 효율적입니다.
  auth_enabled: false 
  
  resources:
    requests:
      ephemeral-storage: 5Gi
    limits:
      ephemeral-storage: 15Gi
```

---

## 4. 트러블슈팅 요약 (Cheat Sheet)

| 문제 현상 | 원인 및 해결 (Action) |
| --- | --- |
| **에이전트 루핑** | 블로그 정보의 한계. 공식 문서의 **API-based 방식**으로 가이드를 수정하세요. |
| **No Org ID 에러** | Loki 인증 모드 충돌. `auth_enabled: false` 설정이 필요합니다. |
| **Permission Denied** | RBAC 권한 부족. ClusterRole에 `pods/log` 권한을 반드시 추가하세요. |
| **경로 조립 실패** | Filesystem 방식의 고유 결함입니다. 미련 없이 API 방식으로 전환하세요. |

---
*Created by: Gemini CLI (2026-08-19)*
*Reference: [Grafana Alloy Official Documentation](https://grafana.com/docs/alloy/latest/configure/kubernetes/)*
