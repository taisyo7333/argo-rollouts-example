# argo-rollouts-example

Sample manifests for trying a canary deployment with Argo Rollouts on OpenShift GitOps.

The Rollout runs `argoproj/rollouts-demo:blue` with 5 replicas and shifts traffic to a new version in steps: 20% → 40% → 60% → 80%. The 20% step pauses until it is resumed manually. Later steps continue automatically after 45 seconds, 20 seconds, and 10 seconds.

## Resources

| File | Description |
| --- | --- |
| `manifests/rollout.yaml` | Rollout with a canary strategy |
| `manifests/service.yaml` | Service for the Rollout (port 80 → 8080) |
| `manifests/route.yaml` | OpenShift Route with edge TLS termination |

# Tested Environments
* OpenShift GitOps 1.22
* OpenShift 4.22

# Reference URLs

https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.22/html/argo_rollouts/getting-started-with-argo-rollouts
