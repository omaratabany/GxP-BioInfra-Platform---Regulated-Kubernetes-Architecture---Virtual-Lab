# PH-04 OPA Gatekeeper

> Part of [[README]] | Previous: [[PH-03 MinIO Object Storage]] | Next: [[PH-05 Falco Runtime Security]]
> CKA domains: admission controllers, ValidatingWebhookConfiguration, resource limits

**Status: NOT STARTED**
**Depends on:** [[PH-01 Forgejo]] running so all subsequent changes go through Forgejo PRs

---

## Goal

Policy-as-code admission control that enforces GxP-relevant constraints on all workloads. My first hard compliance layer -- everything from this point is validated at the API level before it runs. All constraints start in `warn` mode, I monitor for 48 hours, then flip to `deny` one at a time.

---

## GxP Mapping

| Constraint | Annex 11 Clause | What It Enforces |
|---|---|---|
| require-resource-limits | 4.4 -- system capacity | Prevents resource starvation affecting pipeline results |
| require-image-digest | 10 -- change management | Image pinning -- no silent updates, reproducible runs |
| approved-registries | 10 -- change management | Only known, controlled image sources |
| require-labels | 4.6 -- documentation | app, version, env labels -- traceability per workload |
| disallow-privileged | 12.1 -- security | Least privilege -- no privileged containers |

---

## Rollout Order

1. Deploy all ConstraintTemplates
2. Deploy all Constraints in `warn` mode
3. Monitor audit report for 48 hours -- fix violations in existing workloads
4. Flip to `deny` in this order:
   - `require-labels`
   - `require-resource-limits`
   - `disallow-privileged` (Falco DaemonSet needs exception -- add `monitoring` to excludedNamespaces)
   - `approved-registries`
   - `require-image-digest` (most disruptive -- last)

---

## Key Verification Commands

```bash
kubectl get pods -n gatekeeper-system
kubectl get constrainttemplates
kubectl get constraints -A
kubectl get constraints -A -o json | jq '.items[] | {name: .metadata.name, violations: .status.violations}'
```

---

## Exit Criteria

- Gatekeeper audit shows zero violations in the `pipelines` namespace
- Deploying a pod with a `:latest` tag in `pipelines` namespace is denied
- Deploying a pod with no resource limits in `pipelines` namespace is denied

---

## CKA Coverage

Admission controllers, ValidatingWebhookConfiguration, resource requests and limits, namespace-scoped vs cluster-scoped resources.

---

## ConstraintTemplate Rego Definitions

These are the actual OPA Rego policies. Deploy with `kubectl apply -f apps/gatekeeper/templates/` before deploying Constraints.

### Template 1 -- K8sRequireResourceLimits

```yaml
# apps/gatekeeper/templates/require-resource-limits-template.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequireresourcelimits
  labels:
    app.kubernetes.io/name: gatekeeper
    app.kubernetes.io/component: security
    app.kubernetes.io/managed-by: argocd
    env: dev
    app.kubernetes.io/version: "1.0"
spec:
  crd:
    spec:
      names:
        kind: K8sRequireResourceLimits
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequireresourcelimits

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("container <%v> has no CPU limit", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("container <%v> has no memory limit", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          not container.resources.limits.cpu
          msg := sprintf("initContainer <%v> has no CPU limit", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          not container.resources.limits.memory
          msg := sprintf("initContainer <%v> has no memory limit", [container.name])
        }
```

### Template 2 -- K8sRequireImageDigest

```yaml
# apps/gatekeeper/templates/require-image-digest-template.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequireimagedigest
  labels:
    app.kubernetes.io/name: gatekeeper
    app.kubernetes.io/component: security
    app.kubernetes.io/managed-by: argocd
    env: dev
    app.kubernetes.io/version: "1.0"
spec:
  crd:
    spec:
      names:
        kind: K8sRequireImageDigest
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequireimagedigest

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not contains(container.image, "@sha256:")
          msg := sprintf("container <%v> image <%v> does not contain a digest (@sha256:...)", [container.name, container.image])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          not contains(container.image, "@sha256:")
          msg := sprintf("initContainer <%v> image <%v> does not contain a digest (@sha256:...)", [container.name, container.image])
        }
```

### Template 3 -- K8sAllowedRepos

```yaml
# apps/gatekeeper/templates/approved-registries-template.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
  labels:
    app.kubernetes.io/name: gatekeeper
    app.kubernetes.io/component: security
    app.kubernetes.io/managed-by: argocd
    env: dev
    app.kubernetes.io/version: "1.0"
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedrepos

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not any_prefix_match(container.image, input.parameters.repos)
          msg := sprintf("container <%v> image <%v> is not from an approved registry. Approved: %v", [container.name, container.image, input.parameters.repos])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          not any_prefix_match(container.image, input.parameters.repos)
          msg := sprintf("initContainer <%v> image <%v> is not from an approved registry", [container.name, container.image])
        }

        any_prefix_match(str, prefixes) {
          prefix := prefixes[_]
          startswith(str, prefix)
        }
```

### Template 4 -- K8sRequiredLabels

```yaml
# apps/gatekeeper/templates/require-labels-template.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
  labels:
    app.kubernetes.io/name: gatekeeper
    app.kubernetes.io/component: security
    app.kubernetes.io/managed-by: argocd
    env: dev
    app.kubernetes.io/version: "1.0"
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("missing required labels: %v", [missing])
        }
```

### Template 5 -- K8sDisallowPrivileged

```yaml
# apps/gatekeeper/templates/disallow-privileged-template.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sdisallowprivileged
  labels:
    app.kubernetes.io/name: gatekeeper
    app.kubernetes.io/component: security
    app.kubernetes.io/managed-by: argocd
    env: dev
    app.kubernetes.io/version: "1.0"
spec:
  crd:
    spec:
      names:
        kind: K8sDisallowPrivileged
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdisallowprivileged

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("container <%v> is running as privileged -- not allowed", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          container.securityContext.privileged == true
          msg := sprintf("initContainer <%v> is running as privileged -- not allowed", [container.name])
        }
```

---

## Constraint Instances

Deploy these after all ConstraintTemplates are ready. All start in `warn` mode.

```yaml
# apps/gatekeeper/constraints/require-resource-limits.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireResourceLimits
metadata:
  name: require-resource-limits
spec:
  enforcementAction: warn    # flip to deny after 48-hour monitoring
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    excludedNamespaces:
    - kube-system
    - gatekeeper-system
    - local-path-storage
---
# apps/gatekeeper/constraints/require-image-digest.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireImageDigest
metadata:
  name: require-image-digest
spec:
  enforcementAction: warn
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    excludedNamespaces:
    - kube-system
    - gatekeeper-system
    - local-path-storage
---
# apps/gatekeeper/constraints/approved-registries.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: approved-registries
spec:
  enforcementAction: warn
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    excludedNamespaces:
    - kube-system
    - gatekeeper-system
    - local-path-storage
  parameters:
    repos:
    - "docker.io/"
    - "ghcr.io/"
    - "quay.io/"
    - "registry.k8s.io/"
    - "forgejo.homelab/"
    - "busybox"    # short-form docker.io images
    - "nginx"
---
# apps/gatekeeper/constraints/require-labels.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-labels
spec:
  enforcementAction: warn
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    excludedNamespaces:
    - kube-system
    - gatekeeper-system
    - local-path-storage
  parameters:
    labels:
    - "app.kubernetes.io/name"
    - "app.kubernetes.io/version"
    - "env"
---
# apps/gatekeeper/constraints/disallow-privileged.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowPrivileged
metadata:
  name: disallow-privileged
spec:
  enforcementAction: warn
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    excludedNamespaces:
    - kube-system
    - gatekeeper-system
    - monitoring     # Falco needs host-level access -- this namespace is excluded
    - local-path-storage
```
