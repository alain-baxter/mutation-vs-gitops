# Testing mutation webhook VS GitOps

This repo is created to test out ACM Policy Controller/OPA Gatekeeper mutation functionality VS the GitOps workflow provided by ACM Config Sync.

The findings are:
1) An `AssignMetadata` mutation will only add new labels if none found, and will not modify existing labels. As such, it will not create a conflict feedback loop with Config Sync.
2) An `Assign` mutation acting on Pods does not naturally conflict with best practices of configuring workload objects in Kubernetes. Pods are not well suited to GitOps workflow with Config Sync because many of the fields are immutable, and require delete and recreate to modify. Config Sync does not support delete and recreate for Pod objects.
3) An `Assign` mutation acting on other objects (for example, a Deployment object) can create a conflict where live cluster state does not match the expected state in the Config Sync source repository. The example of a conflict in the repository is an [Assign that sets deployment strategy](config-management/policy/mutation/assign-deployments-rolling-update.yaml) and a [Deployment that has it's own conflicting strategy](config-management/namespaces/test-ns/test-deployment.yaml). However, when monitoring the Config Sync reconciler there does not seem to be an impact on resource consumption or any log indication that it is spending more effort reconciling the conflicting deployment object state.

```
$ kubectl get deployment test-deployment -n test-ns -o yaml |yq eval '.spec.strategy' -
rollingUpdate:
  maxSurge: 1
  maxUnavailable: 0
type: RollingUpdate

$ cat config-management/namespaces/test-ns/test-deployment.yaml |yq eval '.spec.strategy' -
type: RollingUpdate
rollingUpdate:
  maxSurge: 2
  maxUnavailable: 0
```
