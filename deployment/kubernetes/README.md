# Hook0 (mace-hooks) Deployment Guide

**Deployment:** hooks2.maceinnovation.com
**Cluster:** n8n-production-cluster (us-east-1)
**Namespace:** mace-hooks

## Architecture

```
Internet → ALB (hooks2.maceinnovation.com)
           ↓
    ┌──────────────────┐
    │  Frontend (Vue)  │ Port 80
    │  Dashboard UI    │
    └──────────────────┘
           ↓ (API calls)
    ┌──────────────────┐
    │  API (Rust)      │ Port 8081
    │  Webhook Ingress │
    └──────────────────┘
           ↓
    ┌──────────────────┐
    │  PostgreSQL 15   │ Port 5432
    │  Events + Queue  │
    └──────────────────┘
           ↓
    ┌──────────────────┐
    │  Output Worker   │
    │  Webhook Egress  │
    └──────────────────┘
```

## Prerequisites

### 1. ACM Certificate

Request a certificate for `hooks2.maceinnovation.com`:

```bash
# Request certificate via AWS Console or CLI
aws acm request-certificate \
  --domain-name hooks2.maceinnovation.com \
  --validation-method DNS \
  --region us-east-1

# Add the DNS validation record to Route53
# Wait for certificate to be issued
# Copy the certificate ARN
```

### 2. Update values-production.yaml

Edit `values-production.yaml` and update:
- `ingress.annotations.alb.ingress.kubernetes.io/certificate-arn`

### 3. Generate Secrets

```bash
# Generate strong passwords
POSTGRES_PASSWORD=$(openssl rand -base64 32)
MASTER_API_KEY=$(openssl rand -hex 32)
JWT_SECRET=$(openssl rand -base64 32)

# Create secrets.yaml from template
cp secrets.yaml.template secrets.yaml

# Edit secrets.yaml and fill in the generated values
# Or use this script:
cat > secrets.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: hook0-secrets
  namespace: mace-hooks
type: Opaque
stringData:
  POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}"
  DATABASE_URL: "postgres://hook0:${POSTGRES_PASSWORD}@postgres.mace-hooks.svc.cluster.local:5432/hook0"
  MASTER_API_KEY: "${MASTER_API_KEY}"
  JWT_SECRET: "${JWT_SECRET}"
EOF

echo "✅ Secrets generated! Save these credentials:"
echo "PostgreSQL Password: ${POSTGRES_PASSWORD}"
echo "Master API Key: ${MASTER_API_KEY}"
echo "JWT Secret: ${JWT_SECRET}"
```

## Deployment Steps

### 1. Configure kubectl

```bash
# Set context to n8n-production-cluster
aws eks update-kubeconfig --name n8n-production-cluster --region us-east-1

# Verify connection
kubectl get nodes
```

### 2. Create Namespace

```bash
kubectl create namespace mace-hooks
```

### 3. Deploy Secrets

```bash
kubectl apply -f secrets.yaml
```

### 4. Deploy Hook0 using Helm

```bash
# From the mace-hooks repository root
cd /home/shawn/dev/aws-ai-utilities/mace-hooks/self-hosting/helm

# Deploy using custom values
helm install hook0 . \
  -n mace-hooks \
  -f ../../deployment/kubernetes/values-production.yaml

# Or upgrade if already installed
helm upgrade hook0 . \
  -n mace-hooks \
  -f ../../deployment/kubernetes/values-production.yaml
```

### 5. Verify Deployment

```bash
# Check all resources
kubectl get all -n mace-hooks

# Check pods
kubectl get pods -n mace-hooks
# Expected: api, output-worker, frontend, postgres, mailpit all Running

# Check services
kubectl get svc -n mace-hooks

# Check ingress and ALB
kubectl get ingress -n mace-hooks

# Get ALB DNS name
kubectl get ingress hook0-ingress -n mace-hooks -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### 6. Configure DNS

```bash
# Get ALB hostname from ingress
ALB_HOSTNAME=$(kubectl get ingress hook0-ingress -n mace-hooks -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "Create CNAME record:"
echo "hooks2.maceinnovation.com → ${ALB_HOSTNAME}"

# Add CNAME record in Route53:
# Name: hooks2.maceinnovation.com
# Type: CNAME
# Value: <ALB_HOSTNAME>
# TTL: 300
```

### 7. Access Hook0

Wait for DNS propagation (5-10 minutes), then access:

- **Frontend Dashboard:** https://hooks2.maceinnovation.com
- **API Endpoint:** https://hooks2.maceinnovation.com/api
- **Mailpit (email testing):** Port-forward to localhost
  ```bash
  kubectl port-forward svc/mailpit 8025:8025 -n mace-hooks
  # Access: http://localhost:8025
  ```

## Testing Webhook Flow

### 1. Create Test Application via API

```bash
# Get Master API Key from secrets
MASTER_API_KEY=$(kubectl get secret hook0-secrets -n mace-hooks -o jsonpath='{.data.MASTER_API_KEY}' | base64 -d)

# Create application
curl -X POST https://hooks2.maceinnovation.com/api/v1/applications \
  -H "Authorization: Bearer ${MASTER_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Application",
    "description": "Testing webhook ingress/egress"
  }'
```

### 2. Create Webhook Endpoint

```bash
# Replace {app_id} with application ID from previous response
curl -X POST https://hooks2.maceinnovation.com/api/v1/applications/{app_id}/endpoints \
  -H "Authorization: Bearer ${MASTER_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://webhook.site/YOUR-UNIQUE-URL",
    "description": "Test endpoint"
  }'
```

### 3. Create Event Type

```bash
curl -X POST https://hooks2.maceinnovation.com/api/v1/event-types \
  -H "Authorization: Bearer ${MASTER_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "test.event",
    "description": "Test event type"
  }'
```

### 4. Send Test Webhook

```bash
# Send webhook event
curl -X POST https://hooks2.maceinnovation.com/api/v1/applications/{app_id}/events \
  -H "Authorization: Bearer ${MASTER_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "event_type": "test.event",
    "payload": {
      "message": "Hello from Hook0!",
      "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
    }
  }'
```

### 5. Verify Delivery

Check webhook.site or your test endpoint to verify the webhook was delivered.

## Monitoring

### View Logs

```bash
# API logs
kubectl logs -n mace-hooks -l app=api --tail=100 -f

# Output worker logs
kubectl logs -n mace-hooks -l app=output-worker --tail=100 -f

# Frontend logs
kubectl logs -n mace-hooks -l app=frontend --tail=100 -f

# PostgreSQL logs
kubectl logs -n mace-hooks -l app=postgres --tail=100 -f
```

### Database Access

```bash
# Port-forward PostgreSQL
kubectl port-forward svc/postgres 5432:5432 -n mace-hooks

# Connect with psql
POSTGRES_PASSWORD=$(kubectl get secret hook0-secrets -n mace-hooks -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d)
PGPASSWORD="${POSTGRES_PASSWORD}" psql -h localhost -U hook0 -d hook0

# Useful queries:
# SELECT * FROM events ORDER BY created_at DESC LIMIT 10;
# SELECT * FROM event_deliveries ORDER BY created_at DESC LIMIT 10;
```

### Check Resource Usage

```bash
kubectl top pods -n mace-hooks
kubectl top nodes
```

## Scaling

### Scale API

```bash
kubectl scale deployment api -n mace-hooks --replicas=3
```

### Scale Workers

```bash
kubectl scale deployment output-worker -n mace-hooks --replicas=4
```

### Scale Frontend

```bash
kubectl scale deployment frontend -n mace-hooks --replicas=3
```

## Troubleshooting

### Pods Not Starting

```bash
kubectl describe pod <pod-name> -n mace-hooks
kubectl logs <pod-name> -n mace-hooks
```

### Database Connection Issues

```bash
# Check PostgreSQL is running
kubectl get pods -n mace-hooks -l app=postgres

# Check PostgreSQL logs
kubectl logs -n mace-hooks -l app=postgres --tail=50

# Verify DATABASE_URL in secret
kubectl get secret hook0-secrets -n mace-hooks -o jsonpath='{.data.DATABASE_URL}' | base64 -d
```

### ALB Not Created

```bash
# Check ingress status
kubectl describe ingress hook0-ingress -n mace-hooks

# Check AWS Load Balancer Controller logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=100
```

### Certificate Issues

```bash
# Verify certificate ARN in ingress
kubectl get ingress hook0-ingress -n mace-hooks -o yaml | grep certificate-arn

# Check ACM certificate status in AWS Console
```

## Cleanup

To completely remove Hook0:

```bash
# Delete Helm release
helm uninstall hook0 -n mace-hooks

# Delete namespace (includes all resources and PVCs)
kubectl delete namespace mace-hooks
```

**Warning:** This will delete all data including the PostgreSQL database.

## Phase 2: Advanced Filtering

After basic webhook flow is working, we'll add advanced filtering:
- Array index matching (body[0-9] × customFields[0-19])
- Complex boolean logic
- Custom filter UI in dashboard

Documentation will be added in Phase 2.

## Support

- **Hook0 Documentation:** https://documentation.hook0.com/
- **Discord:** https://www.hook0.com/community
- **GitHub Issues:** https://github.com/hook0/hook0/issues
