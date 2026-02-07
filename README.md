# Laravel Backend Helm Chart

A production-ready Helm chart for deploying Laravel applications on Kubernetes with PHP-FPM and Nginx sidecar, queue workers, Redis, and ConfigMap support.

## Features

- **PHP-FPM + Nginx Sidecar**: Optimized two-container setup for Laravel applications
- **Queue Workers**: Built-in Laravel queue worker deployment support
- **Redis**: Optional Redis deployment for caching and queues
- **ConfigMap/Secret Support**: Flexible environment variable management
- **Horizontal Pod Autoscaling**: CPU and memory-based autoscaling
- **Health Probes**: Startup, liveness, and readiness probes configured
- **Ingress Support**: Traefik and NGINX ingress controller compatible
- **Multi-Environment**: Separate values files for dev and production

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- PV provisioner support in the underlying infrastructure (optional)

## Chart Structure

```
laravel_backend_helm/
├── Chart.yaml                 # Chart metadata
├── values.yaml                # Default values
├── values-dev.yaml            # Development overrides
├── values-prod.yaml           # Production overrides
└── templates/
    ├── deployment.yaml        # Main application deployment
    ├── queue-deployment.yaml  # Queue worker deployment
    ├── configmap.yaml         # ConfigMap generation
    ├── service.yaml           # Service definition
    ├── ingress.yaml           # Ingress configuration
    ├── redis-deployment.yaml  # Redis deployment (optional)
    ├── redis-service.yaml     # Redis service (optional)
    ├── hpa.yaml               # Horizontal Pod Autoscaler
    └── serviceaccount.yaml    # ServiceAccount
```

## Installation

### Add Repository (if hosted)

```bash
helm repo add myrepo https://charts.example.com
helm repo update
```

### Install Chart

**Development:**
```bash
helm install laravel-backend . \
  -f values-dev.yaml \
  --namespace laravel-dev \
  --create-namespace
```

**Production:**
```bash
helm install laravel-backend . \
  -f values-prod.yaml \
  --namespace laravel-prod \
  --create-namespace
```

### Upgrade Existing Deployment

```bash
helm upgrade laravel-backend . \
  -f values-dev.yaml \
  --namespace laravel-dev
```

## Configuration

### Environment Variables

This chart supports two methods for providing environment variables:

1. **ConfigMap** (Recommended for most values) - All environment variables can be defined in the ConfigMap
2. **Secrets** (For sensitive data) - Use Kubernetes secrets for passwords, API keys, etc.

By default, the chart uses **ConfigMap only** and requires no secrets. This works out of the box for development.

For production, you may want to create secrets for sensitive data:

**Create a secret for sensitive values:**
```bash
kubectl create secret generic laravel-backend-env-prod \
  --from-literal=DB_PASSWORD='your_secure_password' \
  --from-literal=APP_KEY='base64:your_app_key' \
  --from-literal=API_SECRET='your_api_secret' \
  --namespace laravel-prod
```

Then, in your `values-prod.yaml`, uncomment the secret references:

```yaml
containers:
  - name: backend
    envFrom:
      - secretRef:
          name: laravel-backend-env-prod  # Uncomment this
      - configMapRef:
          name: laravel-backend-helm-laravel-app-config
```

**Note:** Sensitive values like `DB_PASSWORD` should be removed from the ConfigMap when using secrets.

### Key Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas | `1` |
| `imageBackend.repository` | Backend container image | `jed.ocir.io/...` |
| `imageBackend.tag` | Backend image tag | `1.0.4` |
| `imageNginx.repository` | Nginx image | `nginx` |
| `imageNginx.tag` | Nginx image tag | `alpine` |
| `config.enabled` | Enable ConfigMap creation | `false` |
| `queue.enabled` | Enable queue worker | `false` |
| `redis.enabled` | Enable Redis deployment | `false` |
| `autoscaling.enabled` | Enable HPA | `false` |

### Environment Variables via ConfigMap

To use ConfigMap-based environment variables, enable and configure the `config` section:

```yaml
config:
  enabled: true
  data:
    - name: laravel-app-config
      config:
        APP_ENV: "production"
        APP_DEBUG: "false"
        DB_CONNECTION: "mysql"
        DB_HOST: "mysql-service"
        # ... more env vars
```

Then reference it in your containers:

```yaml
containers:
  - name: backend
    envFrom:
      - secretRef:
          name: laravel-backend-env
      - configMapRef:
          name: laravel-backend-helm-laravel-app-config
```

### Custom Nginx Configuration

The chart includes a default nginx configuration. You can customize it via the `nginx.config` value in `values.yaml` or by providing your own ConfigMap.

### Queue Worker Configuration

The queue worker runs Laravel's queue:work command. Configure it via:

```yaml
queue:
  enabled: true
  replicas: 2
  command:
    - "php"
    - "artisan"
    - "queue:work"
    - "--tries=3"
    - "--timeout=90"
```

## Values Files

### values.yaml
Default values that apply to all environments.

### values-dev.yaml
Development-specific overrides:
- Single replica
- Debug enabled
- Resource limits optimized for dev
- Separate dev ConfigMaps and Secrets

### values-prod.yaml
Production-specific overrides:
- Multiple replicas with HPA
- Debug disabled
- Higher resource limits
- TLS ingress configuration
- Production-specific ConfigMaps

## Networking

### Service

The chart creates a ClusterIP service exposing port 80.

```yaml
service:
  type: ClusterIP
  port: 80
```

### Ingress

Enable ingress to expose your application:

```yaml
ingress:
  enabled: true
  className: traefik
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: tls-secret
      hosts:
        - api.example.com
```

## Autoscaling

Enable horizontal pod autoscaling based on CPU and memory:

```yaml
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

## Resource Limits

Configure resource limits per container:

```yaml
containers:
  - name: backend
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
```

## Troubleshooting

### Check Pod Status

```bash
kubectl get pods -n laravel-dev
kubectl describe pod <pod-name> -n laravel-dev
```

### View Logs

```bash
# Backend logs
kubectl logs -f deployment/laravel-backend -n laravel-dev -c backend

# Nginx logs
kubectl logs -f deployment/laravel-backend -n laravel-dev -c nginx

# Queue worker logs
kubectl logs -f deployment/laravel-backend-queue -n laravel-dev
```

### Check Environment Variables

```bash
kubectl exec -it <pod-name> -n laravel-dev -c backend -- env | grep APP_
```

### Common Issues

**CreateContainerConfigError: secret "laravel-backend-env" not found:**
- This error occurs when your values file references a secret that doesn't exist
- **Solution 1 (Quick):** Remove or comment out the `secretRef` from your values file and use only ConfigMap
- **Solution 2 (Production):** Create the missing secret:
  ```bash
  kubectl create secret generic laravel-backend-env \
    --from-literal=DB_PASSWORD='your_password' \
    --namespace your-namespace
  ```

**ConfigMap not found:**
- Ensure `config.enabled: true` in your values file
- Check that ConfigMap reference names include the chart prefix: `laravel-backend-helm-<name>`
- Verify ConfigMaps were created: `kubectl get configmaps -n your-namespace`

**Pods in CrashLoopBackOff:**
- Check logs for database connection errors
- Verify secrets are created: `kubectl get secrets -n laravel-dev`
- Ensure Redis is enabled if using cache/queue drivers
- Check environment variables are being loaded correctly

**502 Bad Gateway from Nginx:**
- PHP-FPM container may not be ready
- Check PHP-FPM logs: `kubectl logs <pod> -c backend`
- Verify PHP-FPM is listening on port 9000

## Uninstall

```bash
helm uninstall laravel-backend -n laravel-dev
```

## Development

### Linting

```bash
helm lint .
```

### Template Rendering (Dry Run)

```bash
helm template laravel-backend . -f values-dev.yaml
```

### Testing

```bash
helm test laravel-backend -n laravel-dev
```

## Security Considerations

1. **Secrets Management**: Never commit secrets to git. Use Kubernetes secrets or external secret managers (HashiCorp Vault, AWS Secrets Manager, etc.)
2. **Image Tags**: Use specific image tags, not `latest`
3. **Resource Limits**: Always set resource limits to prevent runaway containers
4. **Network Policies**: Consider adding network policies to restrict pod-to-pod communication
5. **RBAC**: Review and minimize ServiceAccount permissions

## Contributing

When making changes to this chart:

1. Update the `Chart.yaml` version
2. Update this README with any new parameters
3. Test with both `values-dev.yaml` and `values-prod.yaml`
4. Run `helm lint` and `helm template` to validate

## License

This Helm chart is provided as-is for deployment of Laravel applications.

## Support

For issues and questions:
- Check the Troubleshooting section above
- Review Kubernetes pod logs and events
- Verify your values file configuration


helm upgrade --install laravel-backend  ./laravel-helm-deployment  --values ./laravel-helm-deployment/values-dev.yaml --namespace dev --create-namespace