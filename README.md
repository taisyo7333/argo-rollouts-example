# argo-rollouts-example

Sample manifests for canary deployments with Argo Rollouts on OpenShift GitOps.

## Summary

This repository has two examples. Both run `argoproj/rollouts-demo` with 5 replicas and start from the `blue` tag. They use the same Rollout name, `rollouts-demo`, in different namespaces. A single RolloutManager in `openshift-gitops` watches both.

### Manual pause (`rollouts-demo`)

`manifests/overlays/demo` deploys a canary that advances on pauses. `basic-application.yaml` syncs it into the `rollouts-demo` namespace. Sync is manual.

You can confirm:

- `setWeight` changes how many pods run the new version. Every request still goes to one Service, and the Route stays at weight 100 on that Service.
- The 20% step waits until you run `promote`. The later steps continue on their own after 45 seconds (40%), 20 seconds (60%), and 10 seconds (80%).
- `promote` resumes a paused step. `abort` stops the update and keeps the previous ReplicaSet.

### Metric analysis (`rollouts-analysis`)

`manifests/analysis` deploys a canary that promotes or aborts from OpenShift Router metrics. `analysis-application.yaml` syncs it into the `rollouts-analysis` namespace. Sync is automated.

You can confirm:

- The Rollout has a stable Service and a canary Service. The OpenShift Route plugin writes those weights onto the Route, so the router sends part of the traffic to the canary. At the 20% step the Route is stable 80 / canary 20.
- AnalysisTemplate `canary-success-rate` reads the canary 5xx ratio from `haproxy_server_http_responses_total` (`exported_service="rollouts-demo-canary"`). A ratio under 5% for 3 samples continues the rollout to 100%. A ratio at or above 5% aborts it, and the Route returns to the stable Service at weight 100.
- `yellow` stays near a 0% 5xx ratio and completes. `bad-yellow` returns HTTP 500 for most requests and aborts.

## Layout

```
.
├── analysis-application.yaml
├── basic-application.yaml
├── rollout-manager-application.yaml
├── docs
│   └── gitops-management-resources.md
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
│   ├── overlays
│   │   └── demo
│   │       ├── kustomization.yaml
│   │       └── namespace.yaml
│   └── rollout-manager
│       ├── kustomization.yaml
│       └── rollout-manager.yaml
└── README.md
```

`manifests/base` is the manual-pause Rollout, Service, and Route. The `demo` overlay sets the namespace to `rollouts-demo`. `manifests/analysis` is self-contained and does not use that base. `manifests/rollout-manager` installs the controller.

| File | Description |
| --- | --- |
| `manifests/base/rollout.yaml` | Canary Rollout with manual and timed pauses |
| `manifests/base/service.yaml` | Single Service for the manual-pause Rollout (port 80 → 8080) |
| `manifests/base/route.yaml` | Route to that Service, edge TLS, weight 100 |
| `manifests/overlays/demo/` | Overlay for the `rollouts-demo` namespace |
| `basic-application.yaml` | Argo CD Application for the manual-pause example |
| `manifests/analysis/` | Canary Rollout, stable and canary Services, Route, and AnalysisTemplate |
| `analysis-application.yaml` | Argo CD Application for the metric analysis example, with automated sync |
| `manifests/rollout-manager/` | RolloutManager `argo-rollout` with `spec.ha.enabled: true` |
| `rollout-manager-application.yaml` | Argo CD Application that syncs the RolloutManager into `openshift-gitops` |
| `docs/gitops-management-resources.md` | CPU and memory usage, requests, and limits for the GitOps management pods |

## Prerequisites

The Red Hat OpenShift GitOps Operator is already installed. This repository does not install the Operator.

`rollout-manager-application.yaml` creates an Application named `rollout-manager` in `openshift-gitops`. Automated sync is enabled, so Argo CD syncs `manifests/rollout-manager` into `openshift-gitops`:

```bash
oc apply -f rollout-manager-application.yaml
```

The RolloutManager is named `argo-rollout`. `spec.ha.enabled: true` runs the controller as two active/passive replicas. It does not declare a traffic or metric plugin. The Operator loads the OpenShift Route plugin into `argo-rollouts-config`. The analysis Rollout selects that plugin in `trafficRouting.plugins`. Prometheus analysis uses the built-in provider.

If a RolloutManager with a different name already exists, match `metadata.name` to that object before applying. A second name creates a second controller.

After sync, `trafficRouterPlugins` lists `argoproj-labs/openshift` at `file:/plugins/rollouts-trafficrouter-openshift/openshift-route-plugin`:

```bash
oc get cm argo-rollouts-config -n openshift-gitops -o yaml
```

## Manual pause example

`basic-application.yaml` creates an Application named `rollouts-demo` in `openshift-gitops`. It syncs `manifests/overlays/demo` into the `rollouts-demo` namespace. CreateNamespace is enabled. Sync is not automated.

```bash
oc apply -f basic-application.yaml
```

Watch the rollout. Both examples use the name `rollouts-demo`, so pass the namespace.

```sh
oc argo rollouts get rollout rollouts-demo -n rollouts-demo --watch
```

### Promote

The first step stays paused at 20% until you promote it.

```sh
oc argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow -n rollouts-demo
oc argo rollouts promote rollouts-demo -n rollouts-demo
```

### Abort

```sh
oc argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:red -n rollouts-demo
oc argo rollouts abort rollouts-demo -n rollouts-demo
oc argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow -n rollouts-demo
```

## Metric analysis example

The Route in `manifests/analysis/route.yaml` is required. Its primary backend is `rollouts-demo-stable` and its alternate backend is `rollouts-demo-canary`. Cookies are disabled and the balance is `roundrobin`, so successive requests can land on either backend while the canary weight is above 0.

`argoproj/rollouts-demo` does not publish its own Prometheus metrics. The AnalysisTemplate queries Thanos Querier on port 9091. A Job in the same analysis run requests `https://<route-host>/color` for about 150 seconds so the router series is not empty. The Prometheus check starts after 60 seconds and runs 3 times, 30 seconds apart. `successCondition` is `result[0] < 0.05`, and `failureLimit` is 1.

The manifests include a ServiceAccount token Secret and a `cluster-monitoring-view` binding for that query.

### Deploy

`analysis-application.yaml` creates an Application named `rollouts-analysis` in `openshift-gitops`. Automated sync is enabled.

```bash
oc apply -f analysis-application.yaml
```

To apply the same manifests without Argo CD:

```bash
oc apply -k manifests/analysis
```

### Pass

```sh
oc argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow -n rollouts-analysis
oc argo rollouts get rollout rollouts-demo -n rollouts-analysis --watch
```

`yellow` passes the analysis and the rollout continues to weight 100.

### Abort from metrics

```sh
oc argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:bad-yellow -n rollouts-analysis
oc argo rollouts get rollout rollouts-demo -n rollouts-analysis --watch
```

`bad-yellow` fails the analysis. The rollout aborts and the Route returns to the stable Service. The phase stays `Degraded` until you set the image back to the stable tag.

## Tested environments

* OpenShift GitOps 1.22
* OpenShift 4.22

## Reference

* [Getting started with Argo Rollouts (OpenShift GitOps 1.22)](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.22/html/argo_rollouts/getting-started-with-argo-rollouts)
* [Routing traffic by using Argo Rollouts (OpenShift GitOps 1.22)](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.22/html-single/argo_rollouts/index#routing-traffic-by-using-argo-rollouts)
