# GitOps 管理系コンポーネントのリソース

2026-11-13 時点の計測です。使用量は `oc adm top pod` の瞬間値で、Request / Limit は Pod spec の値です。対象は Red Hat OpenShift GitOps Operator 1.22.3 と、それが `openshift-gitops` に置いている管理 Pod です。`rollouts-demo` と `rollouts-analysis` のアプリケーション Pod は含めていません。

CPU とメモリーの Request が無い Pod の QoS Class は `BestEffort` です。ephemeral-storage の Request は QoS に入りません。

## 使用量と CPU / メモリーの Request / Limit

| Namespace | Pod | コンテナ | CPU 使用量 | メモリー使用量 | CPU Request | CPU Limit | メモリー Request | メモリー Limit | QoS |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `openshift-gitops-operator` | `openshift-gitops-operator-controller-manager-84dbcb758b-fq5mm` | `manager` | 2m | 292Mi | なし | なし | なし | なし | BestEffort |
| `openshift-gitops` | `argo-rollouts-cd5f69d79-4g4wj` | `argo-rollouts` | 1m | 26Mi | なし | なし | なし | なし | BestEffort |
| `openshift-gitops` | `argo-rollouts-cd5f69d79-qzwsq` | `argo-rollouts` | 2m | 43Mi | なし | なし | なし | なし | BestEffort |
| `openshift-gitops` | `gitops-plugin-8f9ddfb89-tgxmt` | `gitops-plugin` | 1m | 55Mi | 250m | 500m | 128Mi | 256Mi | Burstable |
| `openshift-gitops` | `openshift-gitops-application-controller-0` | `argocd-application-controller` | 30m | 663Mi | 250m | 2 | 1Gi | 2Gi | Burstable |
| `openshift-gitops` | `openshift-gitops-applicationset-controller-5fbc95947c-rvpps` | `argocd-applicationset-controller` | 1m | 83Mi | 250m | 2 | 512Mi | 1Gi | Burstable |
| `openshift-gitops` | `openshift-gitops-dex-server-87dc6f968-skwgc` | `dex` | 1m | 137Mi | 250m | 500m | 128Mi | 256Mi | Burstable |
| `openshift-gitops` | `openshift-gitops-redis-7f7d8b5c8c-7pkvj` | `redis` | 2m | 19Mi | 250m | 500m | 128Mi | 256Mi | Burstable |
| `openshift-gitops` | `openshift-gitops-repo-server-66c5c844d7-8bqrw` | `argocd-repo-server` | 1m | 95Mi | 250m | 1 | 256Mi | 1Gi | Burstable |
| `openshift-gitops` | `openshift-gitops-server-7d4c64bb5-pnkb8` | `argocd-server` | 3m | 178Mi | 125m | 500m | 128Mi | 256Mi | Burstable |

Argo Rollouts は 2 Pod です。`RolloutManager` の `spec.ha.enabled: true` により、active / passive の 2 レプリカになっています。

## ephemeral-storage

Argo Rollouts の 2 Pod だけ、ephemeral-storage の Request と Limit がどちらも `1Gi` です。Deployment に設定されているのは Limit `1Gi` だけで、Request は Limit と同じ値として Kubernetes が補完しています。他の管理 Pod に ephemeral-storage の Request / Limit はありません。

## 起動時だけの initContainer

実行中の使用量には含まれません。Pod 起動時だけ次の Request / Limit が付きます。

| Pod | initContainer | CPU Request | CPU Limit | メモリー Request | メモリー Limit |
| --- | --- | --- | --- | --- | --- |
| `openshift-gitops-dex-server-87dc6f968-skwgc` | `copyutil` | 250m | 500m | 128Mi | 256Mi |
| `openshift-gitops-repo-server-66c5c844d7-8bqrw` | `copyutil` | 250m | 1 | 256Mi | 1Gi |

## 同じ Namespace にある管理系以外の Pod

`openshift-gitops` には、標準の Argo CD コンポーネント名ではない Pod が 2 つあります。

| Pod | イメージ | CPU 使用量 | メモリー使用量 | CPU Request | CPU Limit | メモリー Request | メモリー Limit | QoS |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `cluster-b77677fbc-ftjd2` | `registry.redhat.io/openshift-gitops-1/gitops-rhel9` | 0m | 18Mi | 250m | 500m | 128Mi | 256Mi | Burstable |
| `cf-demo-repo-plugin-b65dcfd4f-hb9b9` | `registry.access.redhat.com/ubi9/python-311:1` | 4m | 35Mi | 20m | なし | 64Mi | 128Mi | Burstable |

`cluster` は Deployment `cluster` の Pod で、公式 GitOps イメージを使っています。`cf-demo-repo-plugin` は UBI の Python イメージです。

## 読み取れること

GitOps Operator と Argo Rollouts には、CPU とメモリーの Request / Limit がありません。計測時の使用量は、Operator が CPU 2m / メモリー 292Mi、Argo Rollouts の 2 Pod が合計 CPU 3m / メモリー 69Mi です。

Argo CD 各コンポーネントは Request に対して使用量が小さいものがほとんどです。application controller は CPU Request 250m に対して使用量 30m、メモリー Request 1Gi に対して使用量 663Mi です。repo-server は CPU Request 250m に対して使用量 1m、メモリー Request 256Mi に対して使用量 95Mi です。
