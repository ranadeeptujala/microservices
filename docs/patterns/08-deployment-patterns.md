# Deployment Patterns

Patterns for deploying and running microservices.

---

## Sidecar Pattern

**A helper container that runs alongside the main service container.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    SIDECAR PATTERN                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                   ┌──────────────────────────────────┐          │
│                   │            POD                    │          │
│                   │  ┌──────────────┐ ┌────────────┐ │          │
│                   │  │     MAIN     │ │  SIDECAR   │ │          │
│                   │  │  APPLICATION │ │ CONTAINER  │ │          │
│                   │  │              │ │            │ │          │
│                   │  │  (Your code) │ │ • Logging  │ │          │
│                   │  │              │ │ • Proxy    │ │          │
│                   │  │              │ │ • Metrics  │ │          │
│                   │  │              │ │ • Auth     │ │          │
│                   │  └──────────────┘ └────────────┘ │          │
│                   │        ▲              │          │          │
│                   │        └──── localhost ┘         │          │
│                   └──────────────────────────────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Common Sidecar Use Cases:

| Sidecar | Purpose |
|---------|---------|
| **Envoy** | Service mesh proxy, load balancing, mTLS |
| **Fluentd/Fluent Bit** | Log collection and forwarding |
| **Vault Agent** | Secret injection |
| **Prometheus Exporter** | Custom metrics export |

### Kubernetes Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      containers:
        # Main application
        - name: order-service
          image: order-service:v1.0
          ports:
            - containerPort: 8080
        
        # Sidecar for logging
        - name: fluentbit
          image: fluent/fluent-bit:latest
          volumeMounts:
            - name: logs
              mountPath: /var/log
        
        # Sidecar for proxy (if not using service mesh)
        - name: envoy
          image: envoyproxy/envoy:v1.28
          ports:
            - containerPort: 9901
```

---

## Ambassador Pattern

**A proxy that handles outbound connections from a service.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    AMBASSADOR PATTERN                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────┐       │
│   │                        POD                           │       │
│   │  ┌──────────────┐      ┌──────────────────────────┐ │       │
│   │  │     MAIN     │      │      AMBASSADOR          │ │       │
│   │  │  APPLICATION │─────►│                          │ │       │
│   │  │              │      │  Handles:                │ │       │
│   │  │ Calls        │      │  • Retries               │ │       │
│   │  │ localhost    │      │  • Circuit breaking      │ │       │
│   │  │              │      │  • Routing               │ │       │
│   │  │              │      │  • TLS termination       │ │       │
│   │  └──────────────┘      │  • Protocol translation  │ │       │
│   │                        └────────────┬─────────────┘ │       │
│   └─────────────────────────────────────┼───────────────┘       │
│                                         │                        │
│                                         ▼                        │
│                              External Services                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Difference from Sidecar:

| Pattern | Direction | Purpose |
|---------|-----------|---------|
| **Sidecar** | Both/Inbound | Extend app functionality |
| **Ambassador** | Outbound | Handle external connections |

---

## Strangler Fig (Deployment)

**Gradually migrate from monolith to microservices by routing traffic incrementally.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    STRANGLER FIG DEPLOYMENT                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌─────────────────┐                          │
│                    │   API Gateway   │                          │
│                    │    (Router)     │                          │
│                    └────────┬────────┘                          │
│                             │                                    │
│              ┌──────────────┼──────────────┐                    │
│              │              │              │                    │
│              ▼              ▼              ▼                    │
│        ┌─────────┐   ┌─────────┐   ┌─────────────┐             │
│        │ New     │   │ New     │   │  MONOLITH   │             │
│        │User Svc │   │Order Svc│   │  (Legacy)   │             │
│        └─────────┘   └─────────┘   │             │             │
│                                    │  Products   │             │
│        Migrated ✅   Migrated ✅   │  Payments   │             │
│                                    │  (remaining)│             │
│                                    └─────────────┘             │
│                                                                  │
│   Route by path:                                                │
│   /users/* → User Service                                       │
│   /orders/* → Order Service                                     │
│   /* → Monolith (default)                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Blue-Green Deployment

**Two identical production environments, switch traffic instantly.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    BLUE-GREEN DEPLOYMENT                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌─────────────────┐                          │
│                    │  Load Balancer  │                          │
│                    └────────┬────────┘                          │
│                             │                                    │
│              ┌──────────────┴──────────────┐                    │
│              │                             │                    │
│              ▼                             ▼                    │
│   ┌───────────────────┐       ┌───────────────────┐            │
│   │   BLUE (v1.0)     │       │   GREEN (v1.1)    │            │
│   │   ████████████    │       │   ░░░░░░░░░░░░    │            │
│   │   (Current)       │       │   (New version)   │            │
│   └───────────────────┘       └───────────────────┘            │
│                                                                  │
│   1. Deploy new version to GREEN                                │
│   2. Test GREEN environment                                     │
│   3. Switch traffic: BLUE → GREEN                               │
│   4. BLUE becomes standby (instant rollback)                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Canary Deployment

**Gradually roll out to a subset of users.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    CANARY DEPLOYMENT                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Phase 1: 5% Canary                                            │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │▓▓▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│   │
│   │ Canary (5%)                Stable (95%)                 │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Phase 2: 25% Canary (if metrics good)                         │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│   │
│   │ Canary (25%)               Stable (75%)                 │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Phase 3: 100% (Full rollout)                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│   │
│   │ New Version (100%)                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Automatic rollback if error rate increases!                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Istio Canary Configuration:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service
  http:
    - route:
        - destination:
            host: order-service
            subset: stable
          weight: 95
        - destination:
            host: order-service
            subset: canary
          weight: 5
```

---

## Rolling Update

**Gradually replace old instances with new ones.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ROLLING UPDATE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Step 1: Start with all v1                                     │
│   [v1] [v1] [v1] [v1] [v1]                                      │
│                                                                  │
│   Step 2: Replace one at a time                                 │
│   [v2] [v1] [v1] [v1] [v1]    ← 1 new, 4 old                   │
│                                                                  │
│   Step 3: Continue rolling                                      │
│   [v2] [v2] [v2] [v1] [v1]    ← 3 new, 2 old                   │
│                                                                  │
│   Step 4: Complete                                              │
│   [v2] [v2] [v2] [v2] [v2]    ← All new                        │
│                                                                  │
│   Pros: Zero downtime, uses less resources than Blue-Green     │
│   Cons: Both versions run together (compatibility required)     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## AWS Deployment Strategies

**AWS provides built-in support for all major deployment strategies.**

```
┌─────────────────────────────────────────────────────────────────┐
│              AWS DEPLOYMENT OPTIONS BY SERVICE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┬───────────┬─────────────┬──────────────────┐  │
│  │ Service      │ Rolling   │ Blue-Green  │ Canary           │  │
│  ├──────────────┼───────────┼─────────────┼──────────────────┤  │
│  │ ECS          │ ✅ Default│ ✅ CodeDeploy│ ✅ CodeDeploy   │  │
│  │ EKS          │ ✅ K8s    │ ✅ K8s/Istio│ ✅ Istio/Argo   │  │
│  │ Lambda       │ ❌        │ ✅ Alias    │ ✅ Alias weight │  │
│  │ Beanstalk    │ ✅ Built-in│ ✅ Swap URL│ ❌              │  │
│  │ EC2/ASG      │ ✅ Built-in│ ✅ CodeDeploy│ ✅ CodeDeploy  │  │
│  └──────────────┴───────────┴─────────────┴──────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### ECS Rolling Update (Default):

```json
{
  "deploymentConfiguration": {
    "minimumHealthyPercent": 50,
    "maximumPercent": 200
  }
}
```

```
Example: 4 tasks, min 50%, max 200%
1. Start 2 new tasks (total: 6 = 150%)
2. Stop 2 old tasks (total: 4 = 100%)  
3. Repeat until all replaced
```

### ECS Blue-Green with CodeDeploy:

```yaml
# appspec.yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:...:task-definition/app:2"
        LoadBalancerInfo:
          ContainerName: "app"
          ContainerPort: 8080
        PlatformVersion: "LATEST"

Hooks:
  - BeforeAllowTraffic: "LambdaFunctionToValidateBeforeTrafficShift"
  - AfterAllowTraffic: "LambdaFunctionToValidateAfterTrafficShift"
```

```
┌─────────────────────────────────────────────────────────────────┐
│              ECS BLUE-GREEN FLOW                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌─────────────────┐                          │
│                    │       ALB       │                          │
│                    └────────┬────────┘                          │
│                             │                                    │
│              ┌──────────────┼──────────────┐                    │
│              │              │              │                    │
│      Port 80 (Prod)         │        Port 8080 (Test)          │
│              │              │              │                    │
│              ▼              │              ▼                    │
│   ┌─────────────────┐       │   ┌─────────────────┐            │
│   │ Target Group 1  │       │   │ Target Group 2  │            │
│   │ (BLUE - v1.0)   │       │   │ (GREEN - v1.1)  │            │
│   │  ████████████   │       │   │  ░░░░░░░░░░░░   │            │
│   └─────────────────┘       │   └─────────────────┘            │
│                                                                  │
│   CodeDeploy shifts traffic:                                    │
│   1. AllAtOnce - Instant switch                                 │
│   2. Linear - 10% every minute                                  │
│   3. Canary - 10% first, then 100%                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### CodeDeploy Deployment Configs:

```
┌──────────────────────────────────────────────────────────────────┐
│              CODEDEPLOY TRAFFIC SHIFTING                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  CANARY CONFIGS:                                                 │
│  • CodeDeployDefault.ECSCanary10Percent5Minutes                  │
│    → 10% for 5 min, then 100%                                    │
│  • CodeDeployDefault.ECSCanary10Percent15Minutes                 │
│    → 10% for 15 min, then 100%                                   │
│                                                                   │
│  LINEAR CONFIGS:                                                 │
│  • CodeDeployDefault.ECSLinear10PercentEvery1Minutes             │
│    → 10% every minute (10 min total)                             │
│  • CodeDeployDefault.ECSLinear10PercentEvery3Minutes             │
│    → 10% every 3 min (30 min total)                              │
│                                                                   │
│  ALL AT ONCE:                                                    │
│  • CodeDeployDefault.ECSAllAtOnce                                │
│    → Instant switch (Blue-Green)                                 │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Lambda Canary Deployment:

```yaml
# SAM template
AutoPublishAlias: live
DeploymentPreference:
  Type: Canary10Percent5Minutes
  Alarms:
    - !Ref AliasErrorMetricGreaterThanZeroAlarm
  Hooks:
    PreTraffic: !Ref PreTrafficLambdaFunction
    PostTraffic: !Ref PostTrafficLambdaFunction
```

```
Lambda Version Weights:
┌─────────────────────────────────────────────────────────────────┐
│  Alias "live" → Version 2 (10%) + Version 1 (90%)              │
│                                                                  │
│  5 minutes later...                                             │
│                                                                  │
│  Alias "live" → Version 2 (100%)                                │
│  (if no alarms triggered)                                       │
└─────────────────────────────────────────────────────────────────┘
```

### Deployment Strategy Comparison:

| Strategy | Downtime | Rollback | Resource Cost | Risk |
|----------|----------|----------|---------------|------|
| **Rolling** | Zero | Slow (re-deploy) | Low | Medium |
| **Blue-Green** | Zero | Instant | 2x during deploy | Low |
| **Canary** | Zero | Fast | Low extra | Lowest |
| **Recreate** | Yes | Re-deploy | Low | High |

### When to Use Each:

```
┌─────────────────────────────────────────────────────────────────┐
│              CHOOSING DEPLOYMENT STRATEGY                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ROLLING UPDATE → When:                                         │
│  • Standard deployments with backward-compatible changes        │
│  • Resource-constrained environments                            │
│  • Low-risk updates                                             │
│                                                                  │
│  BLUE-GREEN → When:                                             │
│  • Need instant rollback capability                             │
│  • Database migrations (tricky - need backward compatible)      │
│  • Critical services that can't have mixed versions             │
│  • Compliance requires test before production traffic           │
│                                                                  │
│  CANARY → When:                                                 │
│  • High-traffic services (test with real traffic)              │
│  • Risky changes need gradual validation                        │
│  • Want automatic rollback based on metrics                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

[← Back to Index](./README.md) | [Next: Polyglot Microservices →](./09-polyglot-microservices.md)

