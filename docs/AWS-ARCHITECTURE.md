# BuyyMart — AWS, Microservices, and Kubernetes Architecture

**Region:** `ap-south-1` (Mumbai)  
**Platform:** Amazon EKS  
**Style:** Domain microservices, database per service, asynchronous events between services  
**Clients:** Customer web, Seller Centre, Admin Console, and later Android and iOS  
**Related:** [APP-SPECIFICATION.md](./APP-SPECIFICATION.md), [SERVICES.md](./SERVICES.md), [ROADMAP.md](./ROADMAP.md)

This is the target production architecture. Stage uses the same shape at a smaller size. Dev is a third, smaller account. The laptop runs the same services in Docker Compose, with LocalStack only where an engineer is testing S3 or SQS. How a change moves from plan through dev, stage, prod, and monitoring is in [ROADMAP.md](./ROADMAP.md).

---

## 1. Architecture in one picture

```text
                         Internet
                            │
                     Route 53 (buyymart.com)
                            │
                     CloudFront + WAF + ACM
                            │
              ┌─────────────┼──────────────┐
              │             │              │
        S3 (web apps)   S3 (images)    ALB (EKS ingress)
        static assets   CloudFront         │
                                        namespace: edge
                            ┌──────────────┼───────────────┐
                            │              │               │
                      customer-bff   seller-bff      admin-bff
                            │              │               │
                            └──────────────┼───────────────┘
                                           │  ClusterIP + network policy
         ┌────────────┬────────────┬───────┴────────┬─────────────┐
         │            │            │                │             │
     identity    catalogue     commerce          money         sellers
     service     search        cart, order       payment       seller
                 inventory     promotion         settlement    fulfilment
                 media                                      engagement
                                                            notification
                                                            support
         │            │            │                │             │
         └────────────┴────────────┴────────────────┴─────────────┘
                           │                    │
                    Aurora PostgreSQL      EventBridge
                    (one DB per service)        │
                           │                   SQS queues
                    ElastiCache Redis      (one queue per consumer)
                           │
                    OpenSearch (products)
                           │
                    S3 (images, KYC, invoices)
```

External systems sit outside the VPC and are called only by the service that owns that relationship:

| External system | Only this service calls it |
| --- | --- |
| Payment gateway | payment-service |
| SMS provider | notification-service |
| Email (SES or provider) | notification-service |
| Courier aggregator | fulfilment-service |
| GST e-invoice (later) | settlement-service |

---

## 2. AWS account layout

Use AWS Organizations. Four accounts. No workload runs in the management account.

| Account | What runs here |
| --- | --- |
| `buyymart-management` | Organizations, billing, IAM Identity Center (SSO), CloudTrail organization trail, GuardDuty delegated admin |
| `buyymart-shared` | ECR repositories, CI/CD OIDC role, Route 53 public zone, S3 for build artifacts and Terraform state (state bucket locked to this account) |
| `buyymart-staging` | Staging VPC, staging EKS, staging data stores, test-mode payment keys |
| `buyymart-production` | Production VPC, production EKS, production data stores, live payment keys |

Humans sign in through IAM Identity Center. Engineers get staging admin and production read-only. Production deploy is an IAM role assumed by GitHub Actions and by a break-glass SSO permission set with MFA.

Infrastructure is Terraform in its own repository. Kubernetes application releases are Helm charts plus an Argo CD gitops repository. Application source never contains live keys.

---

## 3. Network

One VPC per environment in `ap-south-1`. CIDRs must not overlap so a later Transit Gateway peering stays possible.

| Environment | VPC CIDR |
| --- | --- |
| Staging | `10.20.0.0/16` |
| Production | `10.30.0.0/16` |

Production subnets across three Availability Zones (`aps1-az1`, `aps1-az2`, `aps1-az3`):

| Subnet | CIDR example in AZ a | Who lives here |
| --- | --- | --- |
| Public | `10.30.0.0/20` | ALB, NAT Gateway |
| Private app | `10.30.16.0/20` | EKS worker nodes and pods |
| Private data | `10.30.32.0/20` | Aurora, ElastiCache, OpenSearch |

Repeat the same pattern in the other two AZs (`10.30.48.0/20` public, and so on). Staging can use two AZs.

```text
AZ a                         AZ b                         AZ c
public:  ALB node, NAT       public:  ALB node, NAT       public:  ALB node, NAT
app:     EKS nodes           app:     EKS nodes           app:     EKS nodes
data:    Aurora writer/RO    data:    Aurora reader       data:    Aurora reader
         Redis primary                Redis replica                Redis replica
```

Rules:

- EKS node groups and Aurora have no public IP.
- Inbound to the VPC is only the ALB on 443.
- The ALB security group accepts 443 only from the CloudFront managed prefix list. A custom origin header (`X-Origin-Verify`) is a second check so a guessed ALB DNS name is rejected by the ingress.
- Pod egress to the internet (payment gateway, courier, SMS) goes through NAT Gateways, one per AZ.
- AWS API calls use VPC interface or gateway endpoints so they stay on the Amazon network: S3 (gateway), ECR api, ECR dkr, STS, Secrets Manager, KMS, CloudWatch Logs, SQS, EventBridge, SSM.
- Security groups are referenced by other security groups. Example: Aurora accepts 5432 only from the EKS node security group, and Kubernetes network policy narrows that further to the owning service.

DNS: Route 53 public zone `buyymart.com`. Private hosted zone `buyymart.internal` for Aurora and Redis endpoints. ExternalDNS inside the cluster manages only ingress records under `staging.buyymart.com` and the production API hosts.

---

## 4. Edge and frontends

| Host | Origin | Cache |
| --- | --- | --- |
| `buyymart.com`, `www.buyymart.com` | S3 bucket `buyymart-prod-web-customer` plus `/api/*` behavior to the ALB | Static assets cached. HTML short TTL. `/api/*` no cache |
| `seller.buyymart.com` | S3 `buyymart-prod-web-seller` plus `/api/*` to ALB | Same |
| `admin.buyymart.com` | S3 `buyymart-prod-web-admin` plus `/api/*` to ALB | No public cache on HTML. WAF geo and rate limits tighter |
| `media.buyymart.com` | S3 `buyymart-prod-images` | Long TTL. Images are content-addressed (`/p/{id}/{hash}.webp`) |
| `api.buyymart.com` | ALB only | Used by Android and iOS. No CDN cache |

CloudFront uses an ACM certificate in `us-east-1` (CloudFront requirement). The ALB uses a separate ACM certificate in `ap-south-1`.

WAF on CloudFront:

- AWS Managed Rules: Common Rule Set, Known Bad Inputs, IP Reputation
- Rate limit: 2000 requests / 5 minutes / IP on the site
- Stricter rate limit on `POST /api/v1/auth/otp` and `POST /api/v1/auth/verify`
- Admin host blocked from countries you do not operate in, once you confirm the office locations
- Body size limit so catalogue image upload goes to S3 presigned URLs, not through the API

The three web apps are static TypeScript builds. They call only their own BFF. They do not hold gateway secrets.

---

## 5. Microservice catalog

Each service is independently deployable, owns its data, and exposes a versioned HTTP API inside the cluster. Public clients talk only to a BFF. The full list, with stores, events, and callers, is in [SERVICES.md](./SERVICES.md).

### 5.1 Edge

| Service | Owns | Calls | Store |
| --- | --- | --- | --- |
| `customer-bff` | Mobile and web API for shoppers. Aggregates product, price, stock, delivery estimate | search, catalogue, inventory, cart, promotion, order, payment, fulfilment, support, identity | None. Short response cache in Redis |
| `seller-bff` | Seller Centre API | identity, seller, catalogue, inventory, order, fulfilment, settlement | None |
| `admin-bff` | Admin API. Enforces staff role on every route | All domain services through internal admin endpoints | None |

BFF duties: authentication check, request validation, response shaping, timeouts, and partial-failure behavior (search can fail open to a catalogue fallback; payment never fails open).

### 5.2 Domain services

| Service | Responsibility | Database | Publishes | Consumes |
| --- | --- | --- | --- | --- |
| `identity-service` | Customer OTP, sessions, seller login, staff login, MFA, roles, consent log | `identity` | `UserRegistered`, `UserDeleted` | — |
| `seller-service` | Seller profile, KYC status, suspension, staff of that seller | `seller` | `SellerApproved`, `SellerSuspended` | `UserRegistered` |
| `catalogue-service` | Categories, products, variants, MRP, HSN, tax rate, country of origin, images metadata | `catalogue` | `ProductChanged`, `ProductHidden` | `SellerSuspended` |
| `search-service` | Product search and filters | OpenSearch index `products` | — | `ProductChanged`, `InventoryChanged` |
| `inventory-service` | Stock on hand, reservation, release | `inventory` | `InventoryChanged`, `StockReserved`, `StockRejected` | `OrderCancelled`, `PaymentFailed` |
| `cart-service` | Cart per customer, merge on login | Redis key `cart:{userId}` plus a Postgres snapshot | `CartCheckedOut` | — |
| `promotion-service` | Coupons, validity, usage count | `promotion` | `CouponRedeemed` | `OrderCancelled` (restore usage) |
| `order-service` | Order aggregate and status machine | `order` | `OrderPlaced`, `OrderPaid`, `OrderConfirmed`, `OrderCancelled`, `ReturnRequested`, `ReturnAccepted` | `StockReserved`, `PaymentCaptured`, `PaymentFailed`, `ShipmentUpdated` |
| `payment-service` | Gateway order, webhook verification, capture, refund, idempotency | `payment` | `PaymentCaptured`, `PaymentFailed`, `RefundCompleted` | `OrderPlaced`, `RefundRequested` |
| `fulfilment-service` | Serviceability, shipping fee, courier booking, labels, tracking, COD remittance match | `fulfilment` | `ShipmentBooked`, `ShipmentUpdated`, `CodRefused` | `OrderConfirmed` |
| `settlement-service` | Ledger lines, commission, GST on commission, TCS, TDS, payout statements | `settlement` | `PayoutPrepared` | `OrderPaid`, `RefundCompleted`, `ShipmentUpdated` |
| `notification-service` | SMS, email, later push. Templates | `notification` | — | All customer-facing domain events |
| `support-service` | Tickets linked to an order id | `support` | `TicketOpened` | — |
| `media-service` | Presigned upload, image processing job, public vs KYC bucket policy | `media` metadata | `MediaReady` | — |

`order-service` is the source of truth for order status. Other services store their own ids (`paymentId`, `awb`, `ledgerEntryId`) and the order id. They do not update the order row.

### 5.3 What each service is allowed to store about an order

| Service | Stores |
| --- | --- |
| order | Order id, customer id, seller id per line, amounts in paise, GST breakup, status, address snapshot |
| payment | Order id, gateway payment id, status, method, refund ids. No card number, CVV, or UPI PIN |
| inventory | Variant id, seller id, reservation id, order id |
| fulfilment | Order id, warehouse or seller pickup address, AWB, scans, COD amount |
| settlement | Order id, ledger lines in paise, payout batch id |
| notification | Template, phone or email, provider message id, delivery status |
| support | Ticket id, order id, messages |

---

## 6. Communication rules

### Synchronous (HTTP inside the cluster)

Use sync calls when the user is waiting and the answer is needed to finish the screen.

| Caller | Callee | Timeout | If it fails |
| --- | --- | --- | --- |
| customer-bff | search-service | 400 ms | Fall back to catalogue keyword query |
| customer-bff | inventory-service | 300 ms | Show “stock unavailable”, block add-to-cart |
| order-service | inventory-service | 800 ms | Order stays `created` and returns an error. No payment session |
| order-service | promotion-service | 500 ms | Reject coupon. Do not place a discounted order you cannot prove |
| customer-bff | payment-service | 3 s | Show payment error. Order remains `payment_pending` |
| fulfilment-service | courier API | 5 s | Retry via queue. Seller sees “label pending” |

Internal HTTP:

- Base URL `http://{service}.{namespace}.svc.cluster.local`
- Header `Authorization: Bearer` service JWT, audience = callee name, lifetime 60 seconds, signed with a key in Secrets Manager
- Header `X-Request-Id` and W3C `traceparent`
- Header `Idempotency-Key` on any POST that creates money or an order
- JSON errors: `{ "code", "message", "requestId" }`
- Retries only on idempotent GETs and on POSTs that carry an idempotency key. Max 2 retries, with jitter

### Asynchronous (events)

Use events when the user does not need the result on the same click: search indexing, SMS, courier booking, ledger lines, email.

```text
Service DB transaction
    └── insert business row
    └── insert outbox row
            │
            ▼
     outbox publisher (sidecar or worker in the same service)
            │
            ▼
     Amazon EventBridge bus: buyymart-{env}
            │
            ▼
     SQS queue per consumer, with a dead-letter queue
            │
            ▼
     consumer in the target service
```

Event envelope:

```json
{
  "id": "01JABC...",
  "type": "order.paid",
  "source": "order-service",
  "time": "2026-09-28T06:30:00Z",
  "traceId": "…",
  "data": {
    "orderId": "ord_123",
    "amountPaise": 129900,
    "currency": "INR"
  }
}
```

Rules:

- Consumers are idempotent. They store `event.id` in a `processed_events` table and skip duplicates.
- Queues use a visibility timeout longer than the handler. DLQ after 5 receives. An alarm fires when DLQ depth is above zero.
- EventBridge archive keeps 30 days so a bug can be replayed into a queue.
- Publishers and consumers share JSON schemas stored in git (`contracts/events/*.schema.json`). CI rejects a schema change that breaks a consumer.

### Checkout sequence

```text
Customer
  → customer-bff
      → cart-service            read cart
      → promotion-service       price the coupon
      → fulfilment-service      fee and serviceability for the pin code
      → order-service           create order (payment_pending)
            → inventory-service reserve stock (sync)
            → publish OrderPlaced
  → payment-service             create gateway session (sync, from BFF)
Customer pays at the gateway
Gateway
  → POST /webhooks/payments  → payment-service
        verify signature, store payment, publish PaymentCaptured
order-service consumes PaymentCaptured
        status = paid, then confirmed for in-stock items
        publish OrderConfirmed
fulfilment-service consumes OrderConfirmed
        book courier, publish ShipmentBooked
notification-service consumes OrderConfirmed and ShipmentBooked
        send SMS
settlement-service consumes PaymentCaptured
        append ledger lines (sale, tax). Commission is final after delivery
search-service and inventory stay consistent through InventoryChanged
```

Payment webhooks are the source of truth for “money captured”. The browser return URL only displays status. It never marks an order paid.

---

## 7. Data layer

### 7.1 Aurora PostgreSQL

Engine: Aurora PostgreSQL Serverless v2. One **database** per service. Credentials differ per service. No service has a connection string to another service’s database.

Production clusters, multi-AZ:

| Cluster | Databases on that cluster | Min ACU | Why grouped |
| --- | --- | --- | --- |
| `bm-identity` | `identity` | 1 | Login traffic, strict access |
| `bm-commerce` | `catalogue`, `inventory`, `cart`, `promotion`, `order` | 2 | Checkout path. Separate DBs and users |
| `bm-money` | `payment`, `settlement` | 2 | Backups retained longer. Deletion protection on |
| `bm-ops` | `seller`, `fulfilment`, `notification`, `support`, `media` | 0.5 | Lower write rate |

Grouping databases on one cluster shares failover and maintenance, and keeps the Aurora minimum cost realistic. Isolation is still one database, one user, one Kubernetes secret per service. A service migration never runs against a database it does not own.

Backup:

- Payment and settlement: 35 days point-in-time recovery, daily snapshot copy to `ap-south-2`
- All other production databases: 14 days point-in-time recovery
- Deletion protection on every production cluster
- Encryption with the account KMS key

Money columns are `bigint` paise. Timestamps are `timestamptz` in UTC.

### 7.2 Redis

One ElastiCache for Redis replication group, multi-AZ, in the data subnets. Key prefix is the service name.

| Prefix | Owner | Contents | TTL |
| --- | --- | --- | --- |
| `id:` | identity | OTP hash, session | OTP 5 minutes, session 30 days sliding |
| `cart:` | cart | Cart JSON | 14 days |
| `bff:` | customer-bff | Product page fragment | 30–60 seconds |
| `inv:` | inventory | Available-to-sell cache | 10 seconds |
| `idem:` | payment, order | Idempotency responses | 24 hours |

Redis is a cache and a cart store. Orders, payments, and the ledger are in Aurora only.

### 7.3 OpenSearch

Amazon OpenSearch Service, two data nodes across AZs in production, one node in staging. The search-service is the only writer. The index document contains fields needed to filter and to render a result card: name, brand, category, price paise, rating, seller id, in-stock flag, country of origin. It does not contain customer data.

### 7.4 S3

| Bucket | Access | Contents |
| --- | --- | --- |
| `buyymart-prod-web-*` | CloudFront origin access control | Built JS and CSS |
| `buyymart-prod-images` | CloudFront, public read of objects only via CDN | Product images |
| `buyymart-prod-kyc` | Private. Seller-service task role only. Block public access | KYC documents. SSE-KMS |
| `buyymart-prod-invoices` | Private. Settlement and order roles | Invoice PDFs. Object lock governance for 8 years |
| `buyymart-prod-logs` | Log archive | CloudTrail, ALB logs, optional VPC flow logs |

Uploads use presigned PUT from media-service. The browser uploads straight to S3. A worker generates WebP renditions and publishes `MediaReady`.

---

## 8. Kubernetes platform

### 8.1 Cluster

| Setting | Production | Staging |
| --- | --- | --- |
| Product | Amazon EKS | Amazon EKS |
| Kubernetes version | Current stable EKS version, upgraded quarterly | Same, one minor ahead of prod when testing an upgrade |
| Endpoint | Private API endpoint, plus public endpoint restricted to CI and office IPs | Private + restricted public |
| CNI | Amazon VPC CNI, network policy enabled | Same |
| Nodes | Managed node groups + Karpenter | One on-demand node group |
| Service proxy | kube-proxy (default) | Same |
| Ingress | AWS Load Balancer Controller, ALB, IP mode | Same |
| GitOps | Argo CD | Argo CD |

Node groups:

| Group | Instance | Capacity | Taint | Runs |
| --- | --- | --- | --- | --- |
| `system` | `m6g.large` on-demand, 2 nodes, two AZs | On-demand | `workload=system:NoSchedule` | CoreDNS, VPC CNI, cluster-autoscaler or Karpenter, ALB controller, ExternalDNS, External Secrets, Argo CD, metrics-server |
| `apps` | `m6g.xlarge` on-demand, min 3 (one per AZ) | On-demand | none | BFFs, order, payment, inventory, identity |
| `workers` | Karpenter `m6g.large` and `c6g.large` | Spot with on-demand fallback | `workload=batch:NoSchedule` | Search indexer, image processing, notification consumers, settlement batch |

Money and order pods set a node affinity for `workload=apps` and `karpenter.sh/capacity-type=on-demand`. Spot is for work that can be killed and resumed from a queue.

Pod identity uses **IAM Roles for Service Accounts (IRSA)**. Each service account maps to one IAM role. The role’s policy names the exact queue, bucket prefix, and secret ARN.

### 8.2 Namespaces

| Namespace | Services |
| --- | --- |
| `edge` | customer-bff, seller-bff, admin-bff |
| `identity` | identity-service |
| `catalogue` | catalogue-service, search-service, media-service |
| `commerce` | cart-service, inventory-service, promotion-service, order-service |
| `money` | payment-service, settlement-service |
| `sellers` | seller-service, fulfilment-service |
| `engagement` | notification-service, support-service |
| `observability` | OpenTelemetry collector, Fluent Bit, Prometheus node exporter if used |

Resource quotas on every namespace stop one team from consuming the cluster. Production `commerce` and `money` get a higher CPU quota than `engagement`.

### 8.3 Per-service workload

Every service ships the same Kubernetes objects. The Helm library chart `buyymart-service` renders them.

| Object | Purpose |
| --- | --- |
| `Deployment` | Stateless API. Minimum 2 replicas in production, 3 for customer-bff, order-service, payment-service |
| `Deployment` (worker) | Outbox publisher and SQS consumers. Same image, command `worker`. Minimum 2 replicas |
| `Service` | `ClusterIP` port 80 → container 8080 |
| `ServiceAccount` | IRSA annotation |
| `HorizontalPodAutoscaler` | CPU 65%. Min 2, max set per service (BFF max 20, settlement max 4) |
| `PodDisruptionBudget` | `minAvailable: 1` so a node drain cannot take the service to zero |
| `ExternalSecret` | Maps Secrets Manager JSON to a Kubernetes Secret |
| `ConfigMap` | Non-secret config: log level, dependency URLs, feature flags |
| `NetworkPolicy` | Default deny ingress. Allow only named callers and the namespace DNS |
| `PodMonitor` or ServiceMonitor | Scrape `/metrics` |

Pod spec standards:

- `runAsNonRoot: true`, `runAsUser: 1000`
- `readOnlyRootFilesystem: true`, writable emptyDir only for `/tmp`
- `allowPrivilegeEscalation: false`
- Drop all Linux capabilities
- Liveness `GET /health/live` (process up)
- Readiness `GET /health/ready` (database and, for the API pods, queue connectivity)
- Startup probe on Java or larger images so a slow boot is not killed
- Requests are set from a load test. Limits are 2× CPU request and a fixed memory limit so a leak gets OOMKilled instead of evicting neighbors
- Topology spread across AZs
- Graceful shutdown: `terminationGracePeriodSeconds: 40`, preStop sleep 5 seconds so the ALB deregisters first, then the process finishes in-flight requests
- Image pulled by digest in production (`@sha256:…`), tag is only a human label

### 8.4 Ingress

One internet-facing ALB, group `buyymart-public`.

| Path | Service |
| --- | --- |
| `api.buyymart.com` `/v1/*` customer routes | customer-bff |
| `api.buyymart.com` `/seller/*` | seller-bff |
| `api.buyymart.com` `/admin/*` | admin-bff |
| `api.buyymart.com` `/webhooks/payments` | payment-service |
| `api.buyymart.com` `/webhooks/courier` | fulfilment-service |

Webhook routes skip the BFF so a gateway retry is a single hop. payment-service checks the gateway signature before any state change.

Health checks use `/health/ready`. ALB idle timeout is 60 seconds. Application HTTP server timeout is 55 seconds so the client sees an application error instead of a hang.

### 8.5 Configuration and secrets

| Kind | Examples | Where |
| --- | --- | --- |
| Secret | DB password, JWT signing key, gateway key, courier token, SMS token | Secrets Manager `buyymart/prod/{service}` |
| Config | Replica hints, dependency hostnames, tax feature flag, OTP length | ConfigMap from the gitops repo |
| Image | Container digest | Gitops repo, updated by CI |

External Secrets Operator refreshes every minute. Rotation of a database password is a Secrets Manager rotation Lambda plus a pod restart (Reloader annotation) so pods pick up the new secret.

No secret is baked into an image. No `.env` file is committed. Staging and production key paths differ by the account and the prefix.

### 8.6 Representative manifests

These show the shape. Real releases come from the Helm chart, not from copies of this file.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: commerce
  labels:
    app: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      serviceAccountName: order-service
      terminationGracePeriodSeconds: 40
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: order-service
      containers:
        - name: order-service
          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:replace
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: order-service
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: order-service
                  key: databaseUrl
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            periodSeconds: 10
          resources:
            requests:
              cpu: "250m"
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service
  namespace: commerce
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: order-service
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service
  namespace: commerce
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 65
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service
  namespace: commerce
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: edge
        - podSelector:
            matchLabels:
              app: payment-service
      ports:
        - protocol: TCP
          port: 8080
```

payment-service lives in namespace `money`, so the policy above also needs a `namespaceSelector` for `money` if the payment worker calls order-service over HTTP. Prefer the event `PaymentCaptured` so payment-service does not need ingress into order-service at all. The sample allows it only if you keep one synchronous admin query.

---

## 9. Repository and deploy layout

```text
buyymart-platform/          Terraform: accounts, VPC, EKS, Aurora, Redis, OpenSearch, S3, IAM, WAF
buyymart-gitops/            Helm releases, one directory per env
buyymart-contracts/         OpenAPI and event JSON schemas
buyymart-service-chart/     Shared Helm library chart
services/identity/          One repo or one folder per service
services/order/
services/payment/
web/customer/
web/seller/
web/admin/
```

Gitops directory:

```text
gitops/
  dev/
    edge/customer-bff/values.yaml
    commerce/order-service/values.yaml
    money/payment-service/values.yaml
  staging/
    ...
  production/
    ...
```

`values.yaml` holds replica count, CPU requests, and the image digest. Argo CD watches this repo. A merge to `production` is a deploy.

### Deploy pipeline

```text
git push to service repo
  → GitHub Actions (OIDC, no static AWS keys)
  → unit tests, contract tests against buyymart-contracts
  → docker build
  → image scan (Trivy) and fail on critical CVE with a fix available
  → push to ECR in buyymart-shared
  → open PR on buyymart-gitops changing the digest in dev
  → Argo CD syncs dev
  → merge to main opens a PR for the same digest in staging
  → Argo CD syncs staging
  → smoke test: create order against the payment gateway test mode
  → manual approval
  → PR merges digest to production
  → Argo Rollouts canary for order-service and payment-service (10%, then 100%)
  → other services rolling update, maxUnavailable 0, maxSurge 1
```

Database migrations run as a Kubernetes `Job` (Helm hook `pre-upgrade`) using the same image. The job uses a migration user that can change schema. The running app uses a user that can read and write data and cannot drop tables. A failed migration stops the rollout.

Rollback is a git revert of the digest. Argo CD deploys the previous image. Forward-only migrations are required: a rollback of code must work with the new schema, so migrations are expand then contract across two releases.

---

## 10. Observability

| Signal | Path | Used for |
| --- | --- | --- |
| Logs | stdout JSON → Fluent Bit DaemonSet → CloudWatch Logs `/buyymart/prod/{service}` | Incidents. Retention 30 days hot, 1 year in S3 for payment and order |
| Metrics | `/metrics` Prometheus format → Amazon Managed Prometheus → Amazon Managed Grafana | Latency, error rate, saturation, queue depth |
| Traces | OpenTelemetry SDK → ADOT collector → AWS X-Ray | One checkout across BFF, order, inventory, payment |
| Alarms | Amazon Managed Grafana alerts or CloudWatch alarms → SNS → email and a chat webhook | Paging |
| Audit | admin-bff and domain services write audit rows to their own DB; also a CloudTrail log of AWS API calls | Who refunded, who approved KYC, who changed infrastructure |

Every log line includes `time`, `level`, `service`, `requestId`, `traceId`, `orderId` when present. Logs omit OTP values, tokens, PAN, Aadhaar, and full bank account numbers.

### Alarms that page someone

| Condition | Why |
| --- | --- |
| payment webhook 5xx over 1% for 5 minutes | Orders will sit unpaid |
| SQS oldest message age over 2 minutes on `order` or `payment` queues | Status is drifting |
| Any DLQ depth ≥ 1 for 10 minutes | A consumer is rejecting events |
| order-service or payment-service available replicas < 2 | Checkout at risk |
| Aurora CPU over 80% for 15 minutes, or free storage low | Write path will stall |
| NAT Gateway packets dropped | External gateway calls will fail |
| WAF blocked spike on OTP route | SMS cost or credential attack |

A status page is optional. Internally, Grafana has one board per service and one board for checkout.

---

## 11. Security

| Control | Implementation |
| --- | --- |
| Identity for humans | IAM Identity Center, MFA, short role sessions |
| Identity for workloads | IRSA, one role per service, least-privilege policy |
| Identity for customers | Phone OTP in identity-service. Session token is opaque and stored hashed in Redis |
| Staff | Email, password hashed with Argon2id, TOTP MFA |
| Transport | TLS 1.2+ at CloudFront and ALB. Inside the cluster, HTTP on the VPC is acceptable at launch because network policy limits callers. Add Linkerd or Istio for pod-to-pod mTLS when a compliance review asks for it |
| Data at rest | Aurora, Redis, OpenSearch, S3, and EBS encrypted with KMS |
| KYC | Separate bucket, separate KMS key, access logged with S3 access logs, seller-bff never returns a raw document URL to the browser without a 60-second presigned GET |
| Network | Private nodes, restricted ALB, network policies default deny |
| Supply chain | ECR scan on push, image digest pin, SBOM stored next to the image |
| Detection | GuardDuty, Security Hub, CloudTrail, optional VPC flow logs on production |
| WAF | Managed rules plus OTP rate limit |
| Admin | Separate hostname, MFA, audit log, maker-checker on large refunds |
| Pod | Restricted pod security, read-only root filesystem |

PCI: card data stays on the payment gateway page or SDK. BuyyMart stores gateway ids and status. That keeps the cluster out of cardholder-data scope as long as no route logs request bodies from the gateway.

---

## 12. Environments and sizing

Three environments: **dev**, **stage**, and **prod**. Names, URLs, keys, and promotion rules are in [ROADMAP.md](./ROADMAP.md). Launch size below is for a single-city catalogue, not a festival peak. Scale the `apps` node group and Aurora ACU before a sale event.

### Dev

| Piece | Size |
| --- | --- |
| EKS | 1 node `m6g.large`, stopped outside working hours |
| Aurora | 1 Serverless v2 cluster, min 0.5 ACU, one database per service |
| Redis | `cache.t4g.micro` |
| OpenSearch | 1 × `t3.small.search`, or skip until search-service exists |
| NAT | 1 |
| Payments | Sandbox, or none until checkout is in development |

### Production launch

| Piece | Size |
| --- | --- |
| EKS | 2 system nodes `m6g.large`, 3 app nodes `m6g.xlarge` |
| Aurora | 4 Serverless v2 clusters as in section 7.1 |
| Redis | `cache.t4g.medium`, 1 primary + 1 replica |
| OpenSearch | 2 × `m6g.large.search`, 100 GB gp3 each |
| NAT | 1 per AZ (3) |
| CloudFront | One distribution, three web origins + media + API |

### Staging

| Piece | Size |
| --- | --- |
| EKS | 2 nodes `m6g.large` total, all workloads |
| Aurora | 1 Serverless v2 cluster, min 0.5 ACU, still one database per service |
| Redis | `cache.t4g.micro` |
| OpenSearch | 1 × `t3.small.search` |
| NAT | 1 |
| Payments | Gateway test mode only |

Rough always-on production cost is dominated by NAT Gateways, the EKS control plane, three `m6g.xlarge` nodes, and four Aurora minimums. Expect on the order of **USD 700–1,200 per month** before traffic, tax, support, and data transfer, at the launch size above. Re-check the AWS pricing page for `ap-south-1` before you budget. A big driver is three NAT Gateways; two AZs cut that line item and also cut Aurora’s AZ spread.

Festival peak: raise Aurora max ACU, raise HPA max, and add Karpenter limits the week before. Do not resize by clicking in the console. Change Terraform and the gitops repo.

---

## 13. Failure and recovery

| Failure | What happens |
| --- | --- |
| One AZ down | ALB, EKS pods, Aurora reader, and Redis replica already exist in other AZs. Aurora writer fails over. RTO inside the region is minutes |
| One pod crashed | Kubernetes restarts it. HPA and PDB keep capacity |
| Bad deploy | Revert the gitops digest. Canary on payment and order limits blast radius |
| Payment webhook delayed | Order stays `payment_pending`. A reconciliation worker polls the gateway by order id every 2 minutes for up to 30 minutes |
| Duplicate webhook | Idempotency table in payment-service returns the first result |
| Queue consumer bug | Messages sit in the DLQ. Fix the consumer. Replay from DLQ. EventBridge archive is the second copy |
| Database point-in-time restore | Restore a cluster to a new endpoint, point the one service’s secret at it, restart that service. Other services stay up |
| Region loss (`ap-south-1`) | Version 1 is single-region. Recovery is rebuild EKS from Terraform and gitops, restore Aurora from the `ap-south-2` snapshot, and repoint Route 53. RPO is the snapshot age (target under 1 hour once cross-region copy is on). RTO is several hours. Multi-region active-active is not part of this design |

Stock reservation expires after 15 minutes if payment never arrives, so a crashed browser does not hold inventory.

---

## 14. CI security and promotion rules

- Dev deploys from the service pull request. Stage deploys on every merge to `main`.
- Production deploys only from a gitops commit that staging has run. The image digest is the same one stage ran.
- payment-service and settlement-service require a second reviewer on the gitops PR.
- Production AWS account has no permanent access keys. GitHub OIDC role can push images and assume the gitops deploy role only.
- Terraform plan runs on pull request. Terraform apply to production is a protected workflow.

---

## 15. Service build order

Build the platform once, then add services in the order checkout needs them. Each step is deployable to staging on EKS.

| Step | Deliver |
| --- | --- |
| 1 | Organizations, VPC, endpoints, ECR, EKS, ALB controller, External Secrets, Argo CD, logging |
| 2 | identity-service and customer-bff: OTP login |
| 3 | catalogue-service, media-service, search-service: a product page |
| 4 | inventory-service and cart-service |
| 5 | order-service, payment-service, webhook, test-mode gateway |
| 6 | seller-service, seller-bff, admin-bff: KYC and catalogue moderation |
| 7 | fulfilment-service and notification-service |
| 8 | promotion-service, support-service |
| 9 | settlement-service ledger and payout statement |
| 10 | Canary rollouts, DLQ alarms, backup restore drill |

A restore drill (restore payment database to a throwaway cluster and read a payment row) is part of step 10, before the first live transaction.

---

## 16. Mapping to the apps

| App in the product spec | Runs as | Calls |
| --- | --- | --- |
| Customer website and future Android/iOS | Static web or native shell | `customer-bff` |
| Seller Centre | Static web | `seller-bff` |
| Admin Console | Static web | `admin-bff` |
| Warehouse Console (later) | Static web | `admin-bff` routes or a future `warehouse-bff` |
| Courier | Not a BuyyMart deployment | Called by fulfilment-service |

The order statuses in the product spec (`payment_pending`, `paid`, `confirmed`, `packed`, `shipped`, `delivered`, returns, refunds) are enforced only inside `order-service`. Kubernetes and the other services react to the events that status machine publishes.
