---
title: "[Technical Guide] EKS 로그 수집의 완성: Grafana Alloy API-based 스트리밍"
description: "EKS 로그 수집 과정에서 AI 코딩 에이전트의 반복적인 해결 실패를 경험하고, 공식 문서를 통해 Filesystem 방식에서 Kubernetes API 기반 방식으로 전환한 실전 기록입니다."
pubDate: "Aug 19 2026"
---

안녕하세요. 이 문서는 AWS EKS 환경에서 Grafana Alloy로 로그 수집을 구성하며 겪은 시행착오의 기록입니다.

Filesystem 기반 설정을 반복해서 바꿔도 해결되지 않던 문제를 공식 문서에서 다른 접근 방식을 찾아 전환한 과정을 [문제 발생] -> [반복 시도] -> [방향 전환] -> [결과] 순으로 정리했습니다.

---

## 1. [문제 발생] 당시 Filesystem 기반 접근에서 해결되지 않던 문제

`/var/log/pods`의 로그 파일을 직접 읽는 방식도 Kubernetes에서 사용할 수 있는 로그 수집 방식입니다. 저희도 당시 EKS 환경에 적용하려 했지만, 두 가지 문제에 부딪혔습니다.

### 경로 조립과 파일 접근 문제
당시 설정에서는 네임스페이스, Pod 이름, UID, 컨테이너 이름을 조합해 로그 파일 경로를 만들려고 했습니다. 이 경로 규칙을 relabel과 Regex로 구성하는 과정에서 값의 매핑과 파일 접근 시점을 맞추는 데 어려움을 겪었습니다.

### 에이전트의 반복적인 해결 루프
가장 곤혹스러웠던 점은 **에이전트와의 협업 과정**이었습니다. 참고한 블로그 두 곳이 모두 Filesystem 방식을 다루고 있어, 저는 그 내용을 에이전트의 작업 컨텍스트로 제공했습니다. 하지만 에이전트는 이미 실패했던 두 가지 경로 조립 패턴을 반복해서 제안하며 같은 해결 루프에 빠졌습니다.

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
경로 문자열을 조립한 뒤에도 `no such file` 오류가 반복됐습니다. 당시에는 EKS에서 동적으로 생성되는 로그 파일의 경로와 접근 시점이 맞지 않는 것으로 보고 설정을 바꿨지만, 이 Filesystem 기반 접근으로는 문제가 해결되지 않았습니다.

---

## 2. [과정] 공식 문서에서 찾은 새로운 돌파구

기존 설정을 계속 바꾸는 것만으로는 문제가 해결되지 않자, 접근 방식 자체를 다시 살펴보기로 했습니다. 에이전트가 참고한 블로그를 반복해서 따라가는 대신 **Grafana Alloy 공식 문서(Configure Alloy on Kubernetes)**를 직접 확인했습니다.

문서에서 **Kubernetes API를 통해 Pod 로그를 가져오는 `loki.source.kubernetes` 컴포넌트**를 확인했습니다. Alloy가 노드의 로그 파일시스템에 직접 접근하지 않아도 되므로, 당시 문제였던 파일 경로 조립과 접근을 피할 수 있었습니다. 다만 API 기반 방식은 파일을 직접 읽는 방식보다 네트워크 트래픽과 Kubelet CPU 사용량이 늘어날 수 있습니다.

---

## 3. [결과] API-based 스트리밍 구현

아래는 당시 적용한 설정입니다. API 기반 방식으로 전환하자 당시 수집 대상으로 지정한 모든 로그를 즉시 정상적으로 수집할 수 있었습니다.

### 3.1. 수집기 설정: `alloy-official-values.yaml`
```yaml
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
당시 사용한 Loki Helm values의 일부입니다. `auth_enabled`는 Loki의 테넌시 모드를 제어하며, 외부 인증이나 네트워크 접근 통제를 제공하는 설정은 아닙니다. 당시에는 단일 테넌트 구성으로 운영해 `false`를 사용했습니다.

```yaml
loki:
  # 단일 테넌트 구성: X-Scope-OrgID 헤더 없이 Loki API 요청
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
| **에이전트의 반복적인 해결 루프** | 참고 자료를 따라 경로 조립을 반복 제안했지만 해결되지 않았습니다. 공식 문서를 확인한 뒤 API 기반 방식으로 전환했습니다. |
| **No Org ID 에러** | 단일 테넌트로 운영하는 경우 `auth_enabled: false`를 사용할 수 있습니다. 멀티테넌시를 사용하는 경우에는 요청에 `X-Scope-OrgID` 헤더를 포함해야 합니다. |
| **Permission Denied** | Kubernetes API 기반 수집에 필요한 RBAC를 확인합니다. Pod 로그 읽기에는 `pods/log` 권한이 필요하며, Pod 탐색 권한도 구성에 맞게 확인해야 합니다. |
| **경로 조립 실패** | 당시 환경에서는 경로를 조립한 뒤 `no such file` 오류가 반복됐습니다. 이 문제를 해결하기 위해 API 기반 방식으로 전환했습니다. |

---

## 5. 마무리하며

이번 경험을 통해 기존 접근 방식 안에서 설정을 반복해서 수정하는 것만으로는 문제를 해결하지 못할 수 있다는 점을 확인했습니다.

AI 코딩 에이전트가 제안한 코드를 검토하는 것뿐만 아니라, 에이전트가 문제를 해결하기 위해 선택한 접근 방식 자체도 검토해야 한다는 것을 배웠습니다.

---
*Created by: Gemini CLI (2026-08-19)*
*Reference: [Grafana Alloy Official Documentation](https://grafana.com/docs/alloy/latest/configure/kubernetes/)*
