---
description: "Use when writing Kubernetes manifests, Helm charts, or Kustomize overlays. Covers security contexts, resource limits, probes, network policies, and deployment strategies."
applyTo: "**/{k8s,kubernetes,helm,charts,templates}/**/*.{yaml,yml}"
---

# Kubernetes Conventions

## Security Context — Mandatory

Every pod and container must have a security context.

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

- **`runAsNonRoot: true`**: Container must not run as root. Set at pod level AND container level.
- **`allowPrivilegeEscalation: false`**: No child process can gain more privileges than its parent
- **`readOnlyRootFilesystem: true`**: Only mounted volumes are writable. If your app needs to write, mount an `emptyDir` or PVC to the specific path.
- **`capabilities.drop: [ALL]`**: Start with zero Linux capabilities, add back only what's needed
- **Never use `privileged: true`** in production

## Resource Limits — Mandatory

Every container must have resource requests and limits.

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

- **`requests`**: minimum guaranteed. Used for scheduling. Set to what the container needs at steady state.
- **`limits`**: maximum allowed. CPU throttle above limit, OOMKilled above memory limit.
- **Never omit requests**: pods without requests get BestEffort QoS and are first to be evicted under pressure
- **CPU: 100m = 0.1 core**. Memory: `Mi` = mebibytes (1024²), `Gi` = gibibytes (1024³)
- **Don't over-provision**: `requests = limits` for CPU if you want Guaranteed QoS. For memory, limit can be 1.5-2x request

## Probes — Mandatory

Every pod serving traffic needs liveness and readiness probes.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 2
```

- **`livenessProbe`**: "Should I restart this container?" Checks if the process is alive. Failure → container killed and restarted.
- **`readinessProbe`**: "Should this pod receive traffic?" Checks if the app can serve requests. Failure → pod removed from Service endpoints.
- **Different endpoints**: `/health` is lightweight (process alive). `/ready` checks dependencies (DB reachable, cache connected).
- **`initialDelaySeconds`**: gives the app time to start before probing
- **Don't probe dependencies in liveness**: restarting the pod won't fix a down database — it'll cause a restart loop

## Network Policies

Default-deny, then allow selectively.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

- **Default deny all ingress and egress**: add policies to explicitly allow required traffic
- **Don't use `namespaceSelector: {}`**: allows all namespaces. Be explicit.
- **Egress rules**: restrict outbound traffic too. Pods shouldn't phone home to the internet by default.
- **DNS egress**: pods need UDP on port 53 to `kube-dns` for name resolution

## Labels & Annotations

```yaml
metadata:
  labels:
    app.kubernetes.io/name: my-app
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: my-platform
    app.kubernetes.io/managed-by: helm
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```

- **Standard labels**: `app.kubernetes.io/name`, `app.kubernetes.io/component`, `app.kubernetes.io/part-of`, `app.kubernetes.io/managed-by`
- **`app.kubernetes.io/version`**: the application version (not the chart version)
- **Annotations**: for operational metadata (monitoring config, backup schedules, deployment notes)
- **Labels are for selection, annotations are for metadata** — don't use annotations in selectors

## Deployment Strategy

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  minReadySeconds: 10
  revisionHistoryLimit: 10
```

- **`RollingUpdate`**: default, zero-downtime. `Recreate`: faster but causes downtime (use for stateful workloads that can't run multiple instances)
- **`maxUnavailable: 1`**: one pod down during rollout. Increase for faster deploys on large clusters.
- **`minReadySeconds`**: wait after pod is ready before considering it available. Prevents traffic hitting a pod that's about to crash.
- **`revisionHistoryLimit`**: number of old ReplicaSets to keep for rollback. 10 is usually enough.
- **Use `PodDisruptionBudget`**: prevent voluntary disruptions from taking down all pods at once

## Service Types

```yaml
spec:
  type: ClusterIP       # Default — internal only
  # type: NodePort      # Exposes on every node's IP. Rarely used directly.
  # type: LoadBalancer  # Cloud load balancer. One per service = expensive.
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```

- **Use `ClusterIP` by default**: expose via Ingress or Gateway API, not per-service load balancers
- **`port`**: the Service port. `targetPort`: the container port. `nodePort` (if NodePort): 30000-32767.
- **Headless service** (`clusterIP: None`): for StatefulSets that need pod-level DNS

## Ingress / Gateway API

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

- **TLS everywhere**: HTTPS-only. Redirect HTTP to HTTPS.
- **`cert-manager`**: auto-provision and renew Let's Encrypt certificates. Don't manually manage certs.
- **Use `ingressClassName`**: the newer, required field. Not the deprecated `kubernetes.io/ingress.class` annotation.

## ConfigMaps & Secrets

```yaml
# ConfigMap — non-sensitive configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  API_TIMEOUT: "30s"

---

# Secret — sensitive data (base64-encoded at rest, NOT encrypted)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DATABASE_URL: cG9zdGdyZXM6Ly91c2VyOnBhc3NAaG9zdC9kYg==
```

- **Secrets are NOT encrypted by default**: enable etcd encryption at rest. Even better, use external secrets manager (HashiCorp Vault, AWS Secrets Manager, Sealed Secrets, External Secrets Operator).
- **Mount secrets as volumes, not environment variables**: env vars leak in crash dumps, child processes, and debug endpoints.
- **Immutable ConfigMaps/Secrets**: `immutable: true` prevents accidental modification and improves kubelet performance

## Helm Chart Conventions

```yaml
# values.yaml — defaults, documented
replicaCount: 3

image:
  repository: myapp
  tag: ""  # Empty = use Chart.appVersion
  pullPolicy: IfNotPresent

imagePullSecrets: []

serviceAccount:
  create: true
  annotations: {}
  name: ""

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

- **`values.yaml` has sensible defaults**: someone should be able to `helm install` without a custom values file
- **Comment every value**: what it does, valid range, what happens when omitted
- **`NOTES.txt`**: print useful post-install info (how to access the service, next steps)
- **Use named templates** (`helpers.tpl`): `{{ include "myapp.fullname" . }}` pattern for DRY labels and selectors
- **Never hardcode `latest` tag**: use `Chart.appVersion` or require explicit tag
