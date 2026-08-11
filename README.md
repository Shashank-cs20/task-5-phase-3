Step 1 — Prerequisites + Plan

!/bin/bash
scripts/parity-precheck.sh
set -euo pipefail

PASS=0; FAIL=0

check() {
  local desc=$1 cmd=$2
  if eval "$cmd" &>/dev/null; then
    echo "✅ $desc"; ((PASS++))
  else
    echo "❌ $desc"; ((FAIL++))
  fi
}

check "staging namespace"        "kubectl get namespace staging"
check "production namespace"     "kubectl get namespace production"
check "staging pods running"     "kubectl get pods -n staging  | grep Running"
check "production pods running"  "kubectl get pods -n production | grep Running"
check "staging hpa"              "kubectl get hpa -n staging"
check "production hpa"           "kubectl get hpa -n production"
check "prometheus healthy"       "curl -sf http://prometheus:9090/-/healthy"
check "alertmanager healthy"     "curl -sf http://alertmanager:9093/-/healthy"
check "staging health"           "curl -sf https://staging.placemux.com/health"
check "prod health"              "curl -sf https://prod.placemux.com/health"
check "image matches"            "
  STAGING=$(kubectl get deployment my-app -n staging  -o jsonpath='{.spec.template.spec.containers[0].image}')
  PROD=$(kubectl get deployment my-app    -n production -o jsonpath='{.spec.template.spec.containers[0].image}')
  [ \$STAGING = \$PROD ]
"

echo ""
echo "=== $PASS passed, $FAIL failed ==="
[ $FAIL -eq 0 ] || exit 1
docs/reliability-plan.md

Stage A — The bar
Any deploy can be rolled back in minutes, and staging
genuinely predicts production behaviour.

 Blast radius

| Stage | What changes                     | Blast radius                        | Rollback                            |
|-------|----------------------------------|-------------------------------------|-------------------------------------|
| B     | Staging config parity with prod  | Staging only                        | kubectl apply previous staging yaml |
| C     | Canary deploy + auto-rollback    | Up to 10% prod traffic during canary| kubectl rollout undo                |
| D     | SLO docs + sign-off checks       | Docs only                           | git revert                          |
| E     | Full deploy demo on prod         | Brief canary window on real traffic  | Automated rollback script           |
bash scripts/parity-precheck.sh

Step 2 — Staging/production parity (Stage B)

k8s/staging/app.yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: my-app
  namespace: staging
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      
   strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
      
template:
    metadata:
      labels:
        app: my-app
        
 spec:
     topologySpreadConstraints:
       - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
             app: my-app
      containers:
        - name: my-app
          image: yourusername/my-app:latest
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 2
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: staging
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 3
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: staging
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
kubectl apply -f k8s/staging/app.yaml
bash scripts/verify-parity.sh

Step 3 — Canary deploy + auto-rollback (Stage C)

 k8s/production/canary.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-canary
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      track: canary
  template:
    metadata:
      labels:
        app: my-app
        track: canary
    spec:
      containers:
        - name: my-app
          image: yourusername/my-app:canary
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
.github/workflows/canary.yml
name: Canary Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t yourusername/my-app:${{ github.sha }} .
      - run: npm test
      - run: docker push yourusername/my-app:${{ github.sha }}

  canary:
    needs: build-and-test
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - run: bash scripts/canary-deploy.sh yourusername/my-app:${{ github.sha }}
        env:
          TOKEN: ${{ secrets.API_TOKEN }}

  Step 4 — SLO sign-off (Stage D)

docs/reliability-signoff.md

 Reliability Sign-off

Date:      ___________
Author:    ___________
Version:   ___________

 SLO Evidence

| SLO                        | Target  | Measured | Window | Pass |
|----------------------------|---------|----------|--------|------|
| Availability               | 99.9%   | ___%     | 30d    | [ ]  |
| Search p95 latency         | <200ms  | ___ms    | 30d    | [ ]  |
| Payment success rate       | 99.5%   | ___%     | 30d    | [ ]  |
| Report p95 latency         | <500ms  | ___ms    | 30d    | [ ]  |
| Error budget remaining     | >50%    | ___%     | 30d    | [ ]  |

Capacity Evidence

| Metric              | Value   | Pass |
|---------------------|---------|------|
| Safe VU capacity    | ___VUs  | [ ]  |
| Breaking point      | ___VUs  | [ ]  |
| Headroom (>30%)     | ___%    | [ ]  |
| HPA max not reached | yes/no  | [ ]  |

Rollback Evidence

| Test                        | Result  | Time    | Pass |
|-----------------------------|---------|---------|------|
| Canary auto-rollback fired  | yes/no  | ___s    | [ ]  |
| Manual rollback time        | ___s    | <300s   | [ ]  |
| Staging predicted prod      | yes/no  | —       | [ ]  |

Sign-off
All boxes checked → system is release-ready.

Signed: ___________  Date: ___________
kubectl apply -f k8s/production/canary.yaml
kubectl apply -f k8s/production/rollback-alerts.yaml

Step 5 — End-to-end demo (Stage E)

!/bin/bash
 scripts/reliability-demo.sh
set -euo pipefail

echo "=== 1. pre-check ==="
bash scripts/parity-precheck.sh

echo "=== 2. verify staging/prod parity ==="
bash scripts/verify-parity.sh

echo "=== 3. slo check ==="
bash scripts/slo-check.sh

echo "=== 4. deploy good build via canary ==="
GOOD_SHA=$(git rev-parse --short HEAD)
docker build -t yourusername/my-app:$GOOD_SHA .
docker push yourusername/my-app:$GOOD_SHA
bash scripts/canary-deploy.sh yourusername/my-app:$GOOD_SHA

echo "=== 5. verify good deploy ==="
curl -sf https://prod.placemux.com/health   | jq .
curl -sf https://prod.placemux.com/version  | jq .

echo "=== 6. deploy bad build — trigger auto-rollback ==="
cat > /tmp/Dockerfile.broken << 'EOF'
FROM node:20-slim
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "-e", "const http=require('http');http.createServer((req,res)=>{res.writeHead(500);res.end()}).listen(3000)"]
EOF
docker build -t yourusername/my-app:broken -f /tmp/Dockerfile.broken .
docker push yourusername/my-app:broken

bash scripts/canary-deploy.sh yourusername/my-app:broken || true

echo "=== 7. verify rollback fired ==="
STABLE_IMG=$(kubectl get deployment my-app -n production \
  -o jsonpath='{.spec.template.spec.containers[0].image}')
echo "stable image after rollback: $STABLE_IMG"
[ "$STABLE_IMG" != "yourusername/my-app:broken" ] && echo "✅ rollback succeeded" || echo "❌ rollback failed"

curl -sf https://prod.placemux.com/health | jq .

echo "=== 8. run load test ==="
k6 run tests/load/main.js \
  --out json=results/reliability-$(date +%Y%m%d-%H%M%S).json \
  -e TOKEN=$TOKEN

echo "=== 9. force failure path ==="
kubectl scale deployment my-app --replicas=1 -n production
sleep 10
ERROR=$(curl -s 'http://prometheus:9090/api/v1/query' \
  --data-urlencode 'query=rate(http_requests_total{status=~"5.."}[1m])/rate(http_requests_total[1m])' \
  | jq -r '.data.result[0].value[1]')
echo "error rate with 1 pod: $ERROR"
kubectl scale deployment my-app --replicas=3 -n production
kubectl rollout status deployment/my-app -n production

echo "=== 10. sign-off ==="
bash scripts/slo-check.sh
echo ""
echo "fill docs/reliability-signoff.md and get it signed"
bash scripts/reliability-demo.sh

Expected output:

pre-check       all passed ✅
parity          images match, configs match, hpa match
slo             availability >99.9%, p95 <200ms, payments >99.5%
good canary     promoted in ~2min
bad canary      auto-rollback fired, stable image restored
rollback time   <120s
load test       p95 <500ms, error rate 0%
forced failure  error rate >10% with 1 pod confirmed
recovery        200 OK after scale-up
sign-off        all SLO boxes checked

<img width="1536" height="1024" alt="ss3005" src="https://github.com/user-attachments/assets/e66f7d43-157d-494e-bb4e-977f3f884e95" />
