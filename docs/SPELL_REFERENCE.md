# Spell Reference Guide

Complete reference for writing kast-system spells (application definitions).

## Table of Contents

1. [Basic Configuration](#basic-configuration)
2. [Container Configuration](#container-configuration)
3. [Scaling](#scaling)
4. [Networking](#networking)
5. [Health Checks](#health-checks)
6. [Storage](#storage)
7. [Security](#security)
8. [Vault Integration](#vault-integration)
9. [Istio Integration](#istio-integration)
10. [cert-manager Integration](#cert-manager-integration)

## Basic Configuration

### Minimal Spell

```yaml
name: my-app

image:
  name: nginx
  tag: latest

ports:
  - name: http
    containerPort: 80

service:
  enabled: true
  type: ClusterIP
  ports:
    - name: http
      port: 80
```

### Required Fields

- `name`: Unique identifier for the application
- `image.name`: Container image name
- `image.tag`: Container image tag
- `ports`: At least one port definition

## Container Configuration

### Image

```yaml
image:
  name: myapp
  tag: v1.0.0
  pullPolicy: IfNotPresent  # Always, Never, IfNotPresent
  pullSecrets:
    - name: registry-credentials
```

### Environment Variables

```yaml
env:
  # Simple value
  - name: APP_ENV
    value: "production"

  # From Secret
  - name: DATABASE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password

  # From ConfigMap
  - name: CONFIG_VALUE
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: setting
```

### Resources

```yaml
resources:
  requests:
    cpu: 100m        # Minimum guaranteed
    memory: 128Mi
  limits:
    cpu: 500m        # Maximum allowed
    memory: 512Mi
```

### Command and Args

```yaml
command:
  - /app/entrypoint.sh
args:
  - --config
  - /etc/config/app.yaml
  - --verbose
```

## Scaling

### Manual Replicas

```yaml
replicas: 3
```

### Horizontal Pod Autoscaler

```yaml
replicas: 2  # Initial/minimum

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

## Networking

### Service

```yaml
service:
  enabled: true
  type: ClusterIP  # ClusterIP, NodePort, LoadBalancer
  ports:
    - name: http
      port: 80           # Service port
      targetPort: 8080   # Container port (or name)
      protocol: TCP
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
```

### Ports

```yaml
ports:
  - name: http
    containerPort: 8080
    protocol: TCP
  - name: metrics
    containerPort: 9090
    protocol: TCP
```

### Network Policy

```yaml
networkPolicy:
  enabled: true
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: database
      ports:
        - protocol: TCP
          port: 5432
```

## Health Checks

### Liveness Probe

Determines if container should be restarted.

```yaml
livenessProbe:
  enabled: true
  httpGet:
    path: /health/live
    port: http
    scheme: HTTP
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  successThreshold: 1
  failureThreshold: 3
```

### Readiness Probe

Determines if pod should receive traffic.

```yaml
readinessProbe:
  enabled: true
  httpGet:
    path: /health/ready
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
```

### Startup Probe

Gives slow-starting containers time to start.

```yaml
startupProbe:
  enabled: true
  httpGet:
    path: /health/startup
    port: http
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 30  # 30 * 10s = 5 minutes max
```

### Probe Types

```yaml
# HTTP Get
httpGet:
  path: /health
  port: http
  httpHeaders:
    - name: Custom-Header
      value: value

# TCP Socket
tcpSocket:
  port: 8080

# Exec Command
exec:
  command:
    - cat
    - /tmp/healthy
```

## Storage

### Volumes and Volume Mounts

```yaml
volumeMounts:
  - name: data
    mountPath: /app/data
  - name: config
    mountPath: /app/config
    readOnly: true
  - name: cache
    mountPath: /tmp/cache

volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-data
  - name: config
    configMap:
      name: app-config
  - name: cache
    emptyDir: {}
```

### Persistent Volume Claims

```yaml
persistence:
  enabled: true
  storageClass: local-path
  accessMode: ReadWriteOnce  # ReadWriteOnce, ReadOnlyMany, ReadWriteMany
  size: 10Gi
  mountPath: /app/data
```

### ConfigMaps

```yaml
configMaps:
  app-config:
    data:
      config.yaml: |
        server:
          port: 8080
        database:
          pool_size: 10
      settings.json: |
        {
          "debug": false
        }
```

## Security

### Security Context

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 2000
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
```

### Pod Security

```yaml
podSecurityContext:
  fsGroup: 2000
  supplementalGroups:
    - 3000
```

## Vault Integration

### Basic Secret

```yaml
vault:
  db-credentials:
    path: secret/data/production/database
    outputType: secret
    keys:
      - username
      - password
      - connection_string
```

### Environment Variable Integration

```yaml
vault:
  app-secrets:
    path: secret/data/production/app
    outputType: secret
    keys:
      - api_key

env:
  - name: API_KEY
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: api_key
```

## Istio Integration

### VirtualService

```yaml
istio:
  app-vs:
    # Use gateway from lexicon
    selector:
      access: external
      default: book
    hosts:
      - app.example.com
    routes:
      - match:
          - uri:
              prefix: /api
        route:
          - destination:
              host: my-app
              port: 80
            weight: 90
          - destination:
              host: my-app-canary
              port: 80
            weight: 10
```

### DestinationRule

```yaml
istio:
  app-dr:
    host: my-app
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 100
        http:
          http1MaxPendingRequests: 50
          maxRequestsPerConnection: 2
      loadBalancer:
        simple: LEAST_REQUEST
      outlierDetection:
        consecutiveErrors: 5
        interval: 30s
```

### PeerAuthentication

```yaml
istio:
  app-pa:
    mode: STRICT  # STRICT, PERMISSIVE, DISABLE
```

## cert-manager Integration

### Certificate

```yaml
certManager:
  app-cert:
    dnsNames:
      - app.example.com
      - www.app.example.com
    # Use issuer from lexicon
    selector:
      default: book
    # Or specify directly
    issuer: letsencrypt-prod
    duration: 2160h      # 90 days
    renewBefore: 360h    # 15 days before expiry
```

## Advanced Features

### Init Containers

```yaml
initContainers:
  - name: migration
    image: myapp:v1.0.0
    command: ["./migrate.sh"]
    env:
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: db-credentials
            key: url
```

### Sidecar Containers

```yaml
sidecars:
  - name: log-forwarder
    image: fluent/fluent-bit:2.0
    volumeMounts:
      - name: logs
        mountPath: /var/log
```

### Affinity

```yaml
affinity:
  # Pod anti-affinity (spread across nodes)
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: my-app
          topologyKey: kubernetes.io/hostname

  # Node affinity (prefer certain nodes)
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
            - key: node-type
              operator: In
              values:
                - high-memory
```

### Tolerations

```yaml
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
```

### Custom Labels and Annotations

```yaml
podLabels:
  app.kubernetes.io/component: backend
  app.kubernetes.io/part-of: my-system
  tier: application

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "9090"
```

## ArgoCD Configuration

### Sync Policy

```yaml
appParams:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

## Complete Example

See `full-stack-example.yaml` for a comprehensive spell using all features.

## Best Practices

1. **Always specify resource requests and limits**
2. **Use health checks for all applications**
3. **Run as non-root user**
4. **Use read-only root filesystem when possible**
5. **Store secrets in Vault, not in Git**
6. **Use semantic versioning for image tags**
7. **Add labels for better organization**
8. **Configure HPA for production workloads**
9. **Use network policies to restrict traffic**
10. **Set up proper monitoring and logging**
