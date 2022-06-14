# Pod Security Policy demo

# Table of contents

- [Demo 1: Enforce privileged](#enforce-privileged)
- [Demo 2: Enforce privileged and warn / audit baseline](#enforce-privileged-warn-audit-baseline)
- [Demo 3: Enforce restricted and try to create a pod with `allowPrivilegeEscalation` set to false](#enforce-privileged-with-allowPrivilegeEscalation-set-to-false)
 
## Demo 1: Enforce privileged

### Create a namespace

`kubectl create ns psp-demo`

### Label the namespace to enforce the restricted Pod Secury Standard

`kubectl label ns psp-demo pod-security.kubernetes.io/enforce=restricted`

### Display namespace annotations / get namespace yaml file

`kubectl get ns --show-labels`

```
NAME              STATUS   AGE     LABELS
psp-demo          Active   25m     kubernetes.io/metadata.name=psp-demo,pod-security.kubernetes.io/enforce=restricted
```

`kubectl get ns psp-demo -o yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2022-06-14T18:13:14Z"
  labels:
    kubernetes.io/metadata.name: psp-demo # <-------------- Got it
    pod-security.kubernetes.io/enforce: restricted
  name: psp-demo
  resourceVersion: "1533940"
  uid: 4d5be7df-9c6e-4d67-bdbb-166809382a16
spec:
  finalizers:
    - kubernetes
status:
  phase: Active
```

### Dry run pod creation with privileged access

`kubectl run privileged -n psp-demo --dry-run=server --image=busybox --privileged`

Should get an forbidden error:

```
Error from server (Forbidden): pods "privileged" is forbidden: violates PodSecurity "restricted:latest": privileged (container "privileged" must not set securityContext.privileged=true), allowPrivilegeEscalation != false (container "privileged" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "privileged" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "privileged" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "privileged" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

### Delete namespace

`kubectl delete ns psp-demo`

## Demo 2: Enforce privileged and warn / audit baseline

Perfect setup to help your team to migrate all workloads to baseline (recommended).

### Create namespace

`kubectl create ns psp-demo`

### Label the namespace to enforce privileged and warn / audit baseline the Pod Secury Standard

`kubectl label ns psp-demo pod-security.kubernetes.io/enforce=privileged pod-security.kubernetes.io/warn=baseline pod-security.kubernetes.io/audit=baseline`

### Display namespace annotations / get namespace yaml file

`kubectl get ns --show-labels`

```
NAME              STATUS   AGE     LABELS
psp-demo          Active   17m     kubernetes.io/metadata.name=psp-demo,pod-security.kubernetes.io/audit=baseline,pod-security.kubernetes.io/enforce=privileged,pod-security.kubernetes.io/warn=baseline
```

### Create a pod with privileged access

`kubectl run privileged -n psp-demo --image=busybox --privileged`

Should get:

```
Warning: would violate PodSecurity "baseline:latest": privileged (container "privileged" must not set securityContext.privileged=true)
pod/privileged created
```

The pod created with just a warning message.

`kubectl get po -n psp-demo`

```
NAME         READY   STATUS      RESTARTS      AGE
privileged   0/1     Completed   2 (27s ago)   30s
```

### Delete namespace

`kubectl delete ns psp-demo`

## Demo 3: Enforce restricted and try to create a pod with `allowPrivilegeEscalation` set to false

### Create a namespace

`kubectl create ns psp-demo`

### Label the namespace to enforce the restricted Pod Secury Standard

`kubectl label ns psp-demo pod-security.kubernetes.io/enforce=restricted`

### Display namespace annotations / get namespace yaml file

`kubectl get ns --show-labels`

```
NAME              STATUS   AGE     LABELS
psp-demo          Active   25m     kubernetes.io/metadata.name=psp-demo,pod-security.kubernetes.io/enforce=restricted
```

### Try to create a pod with `allowPrivilegeEscalation` set to false

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: notprivileged
  name: notprivileged
  namespace: psp-demo
spec:
  containers:
  - image: busybox
    name: notprivileged
    resources: {}
    args:
    - sleep
    - "10000000"
    securityContext:
        seccompProfile:
            type: RuntimeDefault
        runAsNonRoot: true
        allowPrivilegeEscalation: false
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
EOF
```

We got an error, because we have to set explicitly `securityContext.capabilities.drop=["ALL"]`

```
Error from server (Forbidden): error when creating "STDIN": pods "notprivileged" is forbidden: violates PodSecurity "restricted:latest": unrestricted capabilities (container "notprivileged" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "notprivileged" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "notprivileged" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

### Retry to create a pod with `allowPrivilegeEscalation` set to false and `securityContext.capabilities.drop=["ALL"]`

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: notprivileged
  name: notprivileged
  namespace: psp-demo
spec:
  containers:
  - image: busybox
    name: notprivileged
    resources: {}
    args:
    - sleep
    - "10000000"
    securityContext:
        seccompProfile:
            type: RuntimeDefault
        runAsNonRoot: true
        allowPrivilegeEscalation: false
        capabilities:
            drop:
                - ALL
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
EOF
```

The pod should be created

### Delete namespace

`kubectl delete ns psp-demo`

## Bonus

### Set a specific version for each Pod Security mode

`pod-security.kubernetes.io/<MODE>-version: <VERSION>`
