# argo-rollouts-example

Sample manifests for trying a canary deployment with Argo Rollouts on OpenShift GitOps.

The Rollout runs `argoproj/rollouts-demo:blue` with 5 replicas and shifts traffic to a new version in steps: 20% → 40% → 60% → 80%. 
The 20% step pauses until it is resumed manually. Later steps continue automatically after 45 seconds, 20 seconds, and 10 seconds.

## Layout
### Tree

```
.
├── analysis-application.yaml
├── demo-application.yaml
├── manifests
│   ├── analysis
│   │   ├── analysis-template.yaml
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── prometheus-auth.yaml
│   │   ├── rollout.yaml
│   │   ├── route.yaml
│   │   ├── service-canary.yaml
│   │   └── service-stable.yaml
│   ├── base
│   │   ├── kustomization.yaml
│   │   ├── rollout.yaml
│   │   ├── route.yaml
│   │   └── service.yaml
│   └── overlays
│       └── demo
│           ├── kustomization.yaml
│           └── namespace.yaml
└── README.md
```


Kustomize base manifests live under `manifests/base`. The `demo` overlay sets the namespace to `rollouts-demo` and adds that Namespace. `manifests/analysis` is a separate demo that promotes or aborts a canary from router metrics.

| File | Description |
| --- | --- |
| `manifests/base/rollout.yaml` | Rollout with a canary strategy |
| `manifests/base/service.yaml` | Service for the Rollout (port 80 → 8080) |
| `manifests/base/route.yaml` | OpenShift Route with edge TLS termination |
| `manifests/overlays/demo/` | Overlay for the `rollouts-demo` namespace |
| `demo-application.yaml` | Argo CD Application that syncs the demo overlay |
| `manifests/analysis/` | Canary Rollout that continues or aborts from Prometheus metrics |
| `analysis-application.yaml` | Argo CD Application that syncs the analysis demo with automated sync |

## Deploy with OpenShift GitOps

`demo-application.yaml` creates an Application named `rollouts-demo` in the `openshift-gitops` namespace. It syncs `manifests/overlays/demo` from this repository into the `rollouts-demo` namespace.
It creates OpenShift objects in `rollouts-demo` namespace.

```bash
oc apply -f demo-application.yaml
```


## ArgoCD Rollouts
```sh
 oc argo rollouts list rollouts -n {namecpaces}
```

```sh
oc argo rollouts get rollout rollouts-demo -n {namecpaces} --watch
```

### promote
```sh
oc argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow
oc argo rollouts promote rollouts-demo
```

### abort

```sh
oc argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:red
oc argo rollouts abort rollouts-demo
oc argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow
```

## Metric analysis demo

This demo is separate from the manual pause demo. It runs in the `rollouts-analysis` namespace. The canary step at 20% runs `canary-success-rate`. That AnalysisTemplate reads the OpenShift Router 5xx ratio for the canary backend from Thanos Querier. The rollout moves to 100% when the ratio stays under 5%, and aborts when it does not.

`argoproj/rollouts-demo` does not publish its own Prometheus metrics. The query uses `haproxy_backend_http_responses_total` for backend `be_secure:rollouts-analysis:rollouts-demo-canary`. The `yellow` tag returns almost all HTTP 200. The `bad-yellow` tag returns HTTP 500 for most requests.

The Route in `manifests/analysis/route.yaml` is required. RolloutManager installs the Argo Rollouts controller. The OpenShift Route plugin updates the weights on this Route. The primary backend is `rollouts-demo-stable` and the alternate backend is `rollouts-demo-canary`.

### Prerequisites

* A RolloutManager is already installed, as for the pause demo.
* `argo-rollouts-config` lists the OpenShift Route plugin `argoproj-labs/openshift` at `file:/plugins/rollouts-trafficrouter-openshift/openshift-route-plugin`. When that entry is missing, add it to the RolloutManager.

### Deploy

`analysis-application.yaml` creates an Application named `rollouts-analysis` in the `openshift-gitops` namespace. Automated sync is enabled, so Argo CD syncs `manifests/analysis` into `rollouts-analysis` after this command:

```bash
oc apply -f analysis-application.yaml
```

To apply the same manifests without Argo CD:

```bash
oc apply -k manifests/analysis
```

The analysis manifests include a ServiceAccount token secret and a `cluster-monitoring-view` binding so the AnalysisTemplate can query Thanos Querier. The traffic job uses that ServiceAccount to read the Route host, then requests `https://<route-host>/color` for about 150 seconds. The Prometheus check starts after 60 seconds and runs 3 times, 30 seconds apart.

### Pass

```sh
oc argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow -n rollouts-analysis
oc argo rollouts get rollout rollouts-demo -n rollouts-analysis --watch
```

`yellow` should pass the analysis and continue to 100%.

### Abort from metrics

```sh
oc argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:bad-yellow -n rollouts-analysis
oc argo rollouts get rollout rollouts-demo -n rollouts-analysis --watch
```

`bad-yellow` should fail the analysis and abort back to the stable ReplicaSet.

## Tested Environments

* OpenShift GitOps 1.22
* OpenShift 4.22

## Reference URLs

https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.22/html/argo_rollouts/getting-started-with-argo-rollouts
