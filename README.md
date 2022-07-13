# Pod Security Admission demo

# Table of contents

- [Prerequisites](#prerequisites)
- [Demos](#demos)
  - [Demo 1: Enforce privileged](#demo-1-enforce-privileged)
  - [Demo 2: Migrate to baseline](#demo-2-migrate-to-baseline)
  - [Demo 3: Drop all capabilites](#demo-3-drop-all-capabilities)
  - [Demo 4: Dry run](#demo-4-dry-run)

# Prerequisites

Read:

- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

A running v1.23+ k8s cluster.

# Demos

## Demo 1: Enforce privileged

### Create a namespace

`kubectl create ns psa-demo`

### Label the namespace to enforce the restricted Pod Secury Standard and pin a specific version

`kubectl label ns psa-demo pod-security.kubernetes.io/enforce=restricted pod-security.kubernetes.io/enforce-version=v1.22`

### Display namespace annotations / get namespace yaml file

`kubectl get ns --show-labels`

```
NAME              STATUS   AGE     LABELS
psa-demo          Active   50s     kubernetes.io/metadata.name=psa-demo,pod-security.kubernetes.io/enforce-version=v1.22,pod-security.kubernetes.io/enforce=restricted
```

`kubectl get ns psa-demo -o yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2022-06-14T18:13:14Z"
  labels:
    kubernetes.io/metadata.name: psa-demo # <-- Got it
    pod-security.kubernetes.io/enforce-version: v1.22 # <-- Got it
    pod-security.kubernetes.io/enforce: restricted
  name: psa-demo
  resourceVersion: "1533940"
  uid: 4d5be7df-9c6e-4d67-bdbb-166809382a16
spec:
  finalizers:
    - kubernetes
status:
  phase: Active
```

### Dry run pod creation with privileged access

`kubectl run privileged -n psa-demo --dry-run=server --image=busybox --privileged`

Should get an forbidden error:

```
Error from server (Forbidden): pods "privileged" is forbidden: violates PodSecurity "restricted:latest": privileged (container "privileged" must not set securityContext.privileged=true), allowPrivilegeEscalation != false (container "privileged" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "privileged" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "privileged" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "privileged" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

### Delete namespace

`kubectl delete ns psa-demo`

## Demo 2: Migrate to baseline

Perfect setup to help your team to migrate all workloads to baseline (recommended).

### Create namespace

`kubectl create ns psa-demo`

### Label the namespace to enforce privileged and warn / audit baseline the Pod Secury Standard

`kubectl label ns psa-demo pod-security.kubernetes.io/enforce=privileged pod-security.kubernetes.io/warn=baseline pod-security.kubernetes.io/audit=baseline`

### Display namespace annotations / get namespace yaml file

`kubectl get ns --show-labels`

```
NAME              STATUS   AGE     LABELS
psa-demo          Active   17m     kubernetes.io/metadata.name=psa-demo,pod-security.kubernetes.io/audit=baseline,pod-security.kubernetes.io/enforce=privileged,pod-security.kubernetes.io/warn=baseline
```

### Create a pod with privileged access

`kubectl run privileged -n psa-demo --image=busybox --privileged`

Should get:

```
Warning: would violate PodSecurity "baseline:latest": privileged (container "privileged" must not set securityContext.privileged=true)
pod/privileged created
```

The pod created with just a warning message.

`kubectl get po -n psa-demo`

```
NAME         READY   STATUS      RESTARTS      AGE
privileged   0/1     Completed   2 (27s ago)   30s
```

### Delete namespace

`kubectl delete ns psa-demo`

## Demo 3: Drop all capabilities

### Create a namespace

`kubectl create ns psa-demo`

### Label the namespace to enforce the restricted Pod Secury Standard

`kubectl label ns psa-demo pod-security.kubernetes.io/enforce=restricted`

### Display namespace annotations / get namespace yaml file

`kubectl get ns --show-labels`

```
NAME              STATUS   AGE     LABELS
psa-demo          Active   25m     kubernetes.io/metadata.name=psa-demo,pod-security.kubernetes.io/enforce=restricted
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
  namespace: psa-demo
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
  namespace: psa-demo
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

`kubectl delete ns psa-demo`

## Demo 4: Dry run

### Create a namespace

`kubectl create ns psa-demo`

### Create a privileged pod

`kubectl run nginx -n psa-demo --image=nginx --privileged`

### Dry run: enforce baseline

`kubectl label ns psa-demo pod-security.kubernetes.io/enforce=baseline --dry-run=server --overwrite`

You should get warning messages:

```
Warning: existing pods in namespace "psa-demo" violate the new PodSecurity enforce level "baseline:latest"
Warning: nginx: privileged
```

### Delete namespace

`kubectl delete ns psa-demo`
