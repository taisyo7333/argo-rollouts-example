# argo-rollouts-example

Sample manifests for trying a canary deployment with Argo Rollouts on OpenShift GitOps.

The Rollout runs `argoproj/rollouts-demo:blue` with 5 replicas and shifts traffic to a new version in steps: 20% → 40% → 60% → 80%. 
The 20% step pauses until it is resumed manually. Later steps continue automatically after 45 seconds, 20 seconds, and 10 seconds.

## Layout
### Tree

```
.
├── demo-application.yaml
├── manifests
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


Kustomize base manifests live under `manifests/base`. The `demo` overlay sets the namespace to `rollouts-demo` and adds that Namespace.

| File | Description |
| --- | --- |
| `manifests/base/rollout.yaml` | Rollout with a canary strategy |
| `manifests/base/service.yaml` | Service for the Rollout (port 80 → 8080) |
| `manifests/base/route.yaml` | OpenShift Route with edge TLS termination |
| `manifests/overlays/demo/` | Overlay for the `rollouts-demo` namespace |
| `demo-application.yaml` | Argo CD Application that syncs the demo overlay |

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

## Tested Environments

* OpenShift GitOps 1.22
* OpenShift 4.22

## Reference URLs

https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.22/html/argo_rollouts/getting-started-with-argo-rollouts
