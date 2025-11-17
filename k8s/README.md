# Kubernetes Deployment for Hospital Backend

This directory contains all the Kubernetes manifests needed to deploy the Hospital Backend application on a Kubernetes cluster.

## Prerequisites

- Kubernetes cluster (v1.24+)
- kubectl installed and configured
- Docker image pushed to a registry (Docker Hub, ECR, GCR, etc.)
- Helm (optional, for advanced deployments)

## Directory Structure

```
k8s/
├── namespace.yaml              # Kubernetes namespace for the application
├── configmap.yaml              # Application configuration
├── secret.yaml                 # Sensitive data (passwords, API keys)
├── serviceaccount.yaml         # Service account and RBAC rules
├── mysql-deployment.yaml       # MySQL database deployment
├── hospital-deployment.yaml    # Hospital backend deployment
├── hospital-service.yaml       # Service to expose the application
├── hospital-ingress.yaml       # Ingress for external access
├── hpa.yaml                    # Horizontal Pod Autoscaler
├── network-policy.yaml         # Network security policies
├── kustomization.yaml          # Kustomize configuration
└── README.md                   # This file
```

## Prerequisites Setup

### 1. Build and Push Docker Image

```bash
# Build the Docker image
docker build -t srivishnu1011/hospital-backend:latest .

# Push to Docker Hub
docker push srivishnu1011/hospital-backend:latest
```

### 2. Update Secret Values

Before deploying, update `secret.yaml` with your actual credentials:

```bash
# Generate a base64 encoded string for your JWT secret
echo -n "your-secret-key" | base64
```

Edit `k8s/secret.yaml` and update the values:
- `db-username`: Database username
- `db-password`: Database password
- `db-root-password`: MySQL root password
- `jwt-secret`: JWT secret for authentication

### 3. Update Image Registry

Edit `hospital-deployment.yaml` and update the image field if using a different registry:

```yaml
image: your-registry/hospital-backend:latest
```

## Deployment Options

### Option 1: Manual Deployment (Individual Manifests)

```bash
# Create namespace
kubectl apply -f k8s/namespace.yaml

# Create secrets and config
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/configmap.yaml

# Deploy database
kubectl apply -f k8s/mysql-deployment.yaml

# Wait for MySQL to be ready
kubectl wait --for=condition=ready pod -l app=mysql -n hospital-app --timeout=300s

# Deploy backend application
kubectl apply -f k8s/serviceaccount.yaml
kubectl apply -f k8s/hospital-deployment.yaml
kubectl apply -f k8s/hospital-service.yaml
kubectl apply -f k8s/hospital-ingress.yaml
kubectl apply -f k8s/hpa.yaml
kubectl apply -f k8s/network-policy.yaml
```

### Option 2: Using Kustomize

```bash
# Deploy all resources using Kustomize
kubectl apply -k k8s/

# Wait for deployments to be ready
kubectl wait --for=condition=available --timeout=300s deployment/hospital-backend -n hospital-app
```

### Option 3: Using Kustomize with Overlays (Production)

For production deployments with different configurations, use overlays:

```bash
# Deploy with production overlay
kubectl apply -k k8s/overlays/production/
```

## Verification

### Check Deployment Status

```bash
# Check namespace
kubectl get ns

# Check all resources in hospital-app namespace
kubectl get all -n hospital-app

# Check deployments
kubectl get deployments -n hospital-app
kubectl describe deployment hospital-backend -n hospital-app

# Check pods
kubectl get pods -n hospital-app
kubectl logs -f deployment/hospital-backend -n hospital-app

# Check services
kubectl get svc -n hospital-app

# Check MySQL is running
kubectl exec -it deployment/mysql -n hospital-app -- mysql -u root -p -e "SHOW DATABASES;"
```

### Test the Application

```bash
# Get the external IP of the service
kubectl get svc hospital-service -n hospital-app

# If using minikube
minikube service hospital-service -n hospital-app

# If using LoadBalancer, check external IP
kubectl get svc hospital-service -n hospital-app -w
```

## Scaling

### Manual Scaling

```bash
# Scale the deployment
kubectl scale deployment hospital-backend --replicas=4 -n hospital-app
```

### Automatic Scaling (HPA)

The HPA is already configured to:
- Maintain minimum 2 replicas
- Scale up to 5 replicas
- Scale based on CPU (70%) and Memory (80%) utilization

Check HPA status:

```bash
kubectl get hpa -n hospital-app
kubectl describe hpa hospital-backend-hpa -n hospital-app
```

## Updating the Application

### Rolling Update

```bash
# Update the image tag in hospital-deployment.yaml
# Then reapply the deployment
kubectl apply -f k8s/hospital-deployment.yaml

# Or use kubectl set image
kubectl set image deployment/hospital-backend hospital-backend=srivishnu1011/hospital-backend:v1.1.0 -n hospital-app

# Check rollout status
kubectl rollout status deployment/hospital-backend -n hospital-app

# Rollback if needed
kubectl rollout undo deployment/hospital-backend -n hospital-app
```

## Environment Configuration

### Development Environment

```bash
# Use replicas: 1 and lower resource limits
kubectl apply -k k8s/overlays/dev/
```

### Production Environment

```bash
# Use replicas: 2+ and higher resource limits
kubectl apply -k k8s/overlays/production/
```

## Monitoring and Logging

### View Logs

```bash
# View logs from all backend pods
kubectl logs -f deployment/hospital-backend -n hospital-app

# View logs from a specific pod
kubectl logs pod-name -n hospital-app

# View logs from MySQL
kubectl logs deployment/mysql -n hospital-app
```

### Port Forwarding (for local testing)

```bash
# Forward local port 8081 to service
kubectl port-forward svc/hospital-service 8081:80 -n hospital-app

# Access at http://localhost:8081
```

## Security Considerations

1. **Secrets Management**
   - Use external secret management tools (Sealed Secrets, HashiCorp Vault)
   - Never commit secrets to git
   - Rotate credentials regularly

2. **RBAC**
   - Service account has minimal required permissions
   - Review and adjust roles based on application needs

3. **Network Policies**
   - Pod-to-pod communication is restricted
   - Database access only from application pods
   - DNS access for external communication

4. **Resource Limits**
   - CPU and memory limits prevent resource exhaustion
   - Requests ensure pod scheduling

## Troubleshooting

### Pod Not Starting

```bash
# Check pod status and events
kubectl describe pod <pod-name> -n hospital-app

# Check recent logs
kubectl logs <pod-name> -n hospital-app
```

### Database Connection Issues

```bash
# Test MySQL connectivity from application pod
kubectl exec -it <pod-name> -n hospital-app -- sh
# Inside pod:
nc -zv mysql-service 3306
```

### Service Not Accessible

```bash
# Check service endpoints
kubectl get endpoints hospital-service -n hospital-app

# Check ingress status
kubectl describe ingress hospital-ingress -n hospital-app
```

## Cleanup

### Delete all resources

```bash
# Delete specific namespace (removes all resources in it)
kubectl delete namespace hospital-app

# Or delete individual manifests
kubectl delete -k k8s/
```

## Production Deployment Checklist

- [ ] Update all credentials in `secret.yaml`
- [ ] Update Docker image registry and tag
- [ ] Configure ingress hostname for your domain
- [ ] Set up TLS/SSL certificates (Cert Manager or manual)
- [ ] Configure monitoring (Prometheus, ELK stack)
- [ ] Set up log aggregation
- [ ] Configure backup strategy for MySQL
- [ ] Test disaster recovery procedures
- [ ] Set up alerts for critical metrics
- [ ] Document all configurations and passwords securely

## Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Spring Boot on Kubernetes](https://spring.io/projects/spring-cloud-kubernetes)
- [Kustomize Documentation](https://kustomize.io/)
- [MySQL on Kubernetes Best Practices](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/)

## Support

For issues or questions about the Kubernetes deployment:
1. Check the logs: `kubectl logs deployment/hospital-backend -n hospital-app`
2. Describe resources: `kubectl describe deployment hospital-backend -n hospital-app`
3. Review events: `kubectl get events -n hospital-app`
