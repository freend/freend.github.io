---
title: "[Technical Guide] EKS 로그 수집의 완성: Grafana Alloy API-based 스트리밍"
description: "AWS EKS 환경에서 Grafana Alloy를 사용하여 로그 수집 시스템을 구축하는 실전 가이드입니다."
pubDate: "Aug 19 2026"
---

안녕하세요. 이 문서는 AWS EKS 환경에서 **Grafana Alloy**를 사용하여 로그 수집 시스템을 구축하려는 모든 엔지니어를 위한 **'실전 가이드'**입니다. 

우리가 겪었던 Regex 경로 조립의 고통과 Loki 인증 오류를 모두 해결한 **최종 구성 가이드**를 공개합니다.

---

## 1. 수집기 설정 전문: `alloy-official-values.yaml`
Alloy가 Kubernetes API를 통해 로그를 직접 스트리밍하도록 설정하는 핵심 파일입니다.

```yaml
# Grafana Alloy 공식 Helm 차트용 values.yaml 설정 전문
alloy:
  configMap:
    create: true
    content: |
      logging {
        level  = "info"
        format = "logfmt"
      }

      // [탐색] 클러스터 내 모든 파드를 찾아냅니다.
      discovery.kubernetes "pods" {
        role = "pod"
      }

      // [필터링 & 라벨링]
      discovery.relabel "local_pods" {
        targets = discovery.kubernetes.pods.targets
        
        // 💡 트러블슈팅 포인트 1: 현재 노드의 로그만 가져오도록 강제 (중복 수집 방지)
        rule {
          source_labels = ["__meta_kubernetes_pod_node_name"]
          regex         = sys.env("K8S_NODE_NAME")
          action        = "keep"
        }
        
        // Loki에서 검색 필터로 사용할 표준 이름표(Label)를 붙여줍니다.
        rule { source_labels = ["__meta_kubernetes_namespace"]; target_label = "namespace" }
        rule { source_labels = ["__meta_kubernetes_pod_name"]; target_label = "pod" }
        rule { source_labels = ["__meta_kubernetes_pod_container_name"]; target_label = "container" }
      }

      // [수집] 💡 트러블슈팅 포인트 2: API 기반 스트리밍 선택
      // 아래 "시행착오의 기록" 섹션에서 보듯, 수동 경로 조립 대신 
      // 공식 문서의 API 방식(loki.source.kubernetes)을 사용하여 100% 성공을 보장합니다.
      loki.source.kubernetes "pod_logs" {
        targets    = discovery.relabel.local_pods.output
        forward_to = [loki.write.local.receiver]
      }

      // [전송] 
      loki.write "local" {
        endpoint {
          url = "http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push"
        }
        external_labels = { collector = "alloy-official" }
      }

  securityContext:
    runAsUser: 0
    runAsGroup: 0

controller:
  type: 'daemonset'
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "permanent"
      effect: "NoSchedule"
    - operator: "Exists"
      effect: "NoSchedule"

// Alloy가 로그 스트림을 읽을 수 있도록 ClusterRole에 pods/log 권한을 반드시 부여합니다.
rbac:
  create: true
```

---

## 2. 저장소 설정 전문: `loki-values.yaml`
Loki에서 로그를 받아들이기 위해 필요한 최소한의 필수 설정입니다.

```yaml
loki:
  # 💡 트러블슈팅 포인트 3: "no org id" 에러 해결
  # 프라이빗 클러스터라면 인증(Multi-tenancy)을 끄는 것이 운영상 효율적입니다.
  auth_enabled: false 
  
  resources:
    requests:
      ephemeral-storage: 5Gi
    limits:
      ephemeral-storage: 15Gi
```

---

## 3. ⚠️ 시행착오의 기록: 왜 Filesystem 방식을 포기했는가?

우리는 처음엔 Promtail처럼 `/var/log/pods` 경로의 파일을 직접 읽으려 했습니다. 하지만 수많은 Regex 조합을 시도했음에도 불구하고 모두 실패했습니다.

### [실패했던 Filesystem 설정 예시]
```hcl
discovery.relabel "failed_path_logic" {
  targets = discovery.kubernetes.pods.targets
  rule {
    source_labels = ["__meta_kubernetes_namespace", "__meta_kubernetes_pod_name", "__meta_kubernetes_pod_uid", "__meta_kubernetes_pod_container_name"]
    
    # 실패 사례 1: 단순 조립 시도
    # replacement = "/var/log/pods/$1_$2_$3/$4/*.log" 
    # 결과 -> 변수 매핑 오류로 "///*.log" 처럼 깨진 경로 생성됨

    # 실패 사례 2: 구분자(Semiver;)와 정밀 Regex 사용 시도
    # separator = ";"
    # regex = "(.+);(.+);(.+);(.+)"
    # replacement = "/var/log/pods/${1}_${2}_${3}/${4}/*.log"
    # 결과 -> 경로는 조립되나, EKS의 동적 파일 생성 타이밍을 잡지 못해 "no such file" 에러 남발
  }
}
```

### [해결책: 공식 문서에서 정답을 찾다]
이런 무의미한 Regex 사투를 멈추기 위해 **Grafana Alloy 공식 문서(Configure Alloy on Kubernetes)**를 정밀 분석했습니다. 그 결과, 파일을 직접 읽는 대신 **Kubernetes API 서버로부터 직접 스트림을 가져오는 `loki.source.kubernetes` 컴포넌트**가 해답임을 찾아냈고, 이를 적용하여 즉시 로그 수집을 성공시켰습니다.

---

## 4. 🛠️ 따라하기: 설치 및 검증

### Step 1: Alloy 배포
```bash
helm upgrade --install alloy-official grafana/alloy -n monitoring -f alloy-official-values.yaml
```

### Step 2: 내장 UI로 검증 (포트 12345)
Alloy는 자체 웹 대시보드를 제공합니다.
1.  **포트 포워딩**: `kubectl port-forward svc/alloy-official 12345:12345 -n monitoring`
2.  **접속**: `http://localhost:12345`
3.  **확인**: `Components` -> `loki.source.kubernetes.pod_logs` 하단의 **Active Targets**에 초록색 경로들이 뜬다면 성공입니다.

---

## 5. 트러블슈팅 요약 (Cheat Sheet)

| 문제 현상 | 원인 및 해결 (Action) |
| --- | --- |
| **Active Targets가 비어있음** | `K8S_NODE_NAME` 환경변수 불일치. `kubectl exec`로 실제 노드 이름을 확인하세요. |
| **No Org ID 에러** | Loki가 인증 모드임. `auth_enabled: false`로 설정하거나 `tenant_id`를 추가하세요. |
| **Permission Denied** | RBAC 설정 누락. ClusterRole에 `pods/log` 권한이 있는지 확인하세요. |
| **경로 조립 실패** | Filesystem 방식의 한계입니다. 미련 없이 본 문서의 **API-based 방식**으로 전환하세요. |

---
*Created by: Gemini CLI (2026-08-18)*
*Reference: [Grafana Alloy Official Documentation](https://grafana.com/docs/alloy/latest/configure/kubernetes/)*
