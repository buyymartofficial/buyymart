# BuyyMart — Roadmap from plan to monitoring

**Audience:** First cloud deployment. Read this before opening the AWS console.  
**Region:** `ap-south-1` (Mumbai)  
**Related:** [APP-SPECIFICATION.md](./APP-SPECIFICATION.md), [SERVICES.md](./SERVICES.md), [AWS-ARCHITECTURE.md](./AWS-ARCHITECTURE.md)

This is the order of work. Build the platform once. Add services in the order checkout needs them. Every service is deployed to **dev**, then **stage**, then **prod**. The same container image moves forward. Only configuration and secrets change.

---

## 1. Words used in this document

| Word | Meaning for BuyyMart |
| --- | --- |
| Environment | A full copy of the system with its own data and secrets. BuyyMart keeps three: dev, stage, prod. |
| AWS account | A separate AWS login boundary. A mistake in dev cannot delete prod, because they are different accounts. |
| Container image | The packaged service (for example `order-service`) built once by CI and stored in ECR. |
| Cluster | The Kubernetes (EKS) cluster that runs those images. Each cloud environment has its own cluster. |
| Deploy | Telling the cluster to run a specific image digest. Argo CD does this from the gitops repo. |
| Pipeline | The automatic path: push code, run tests, build the image, open a gitops change, deploy. |
| Monitor | Logs, metrics, traces, and alarms that tell you the system is healthy after it is deployed. |

---

## 2. The three environments

Same services in all three. Different size, data, and keys.

| | Dev | Stage | Prod |
| --- | --- | --- | --- |
| Purpose | Build and try a change the same day | Prove the exact image that will go live | Real customers and real money |
| Where | AWS account `buyymart-dev` | AWS account `buyymart-staging` | AWS account `buyymart-production` |
| Also on the laptop | Docker Compose, for the service you are editing | — | — |
| URL shape | `dev.buyymart.com`, `api.dev.buyymart.com` | `stage.buyymart.com`, `api.stage.buyymart.com` | `buyymart.com`, `api.buyymart.com` |
| Data | Fake users and products. Safe to wipe | Fake catalogue and test orders. Kept between deploys | Real orders, payments, KYC. Retained for tax |
| Payments | Gateway sandbox, or no gateway until checkout exists | Gateway **test mode** only | Gateway **live** keys |
| SMS and email | Log the message. Do not send to real phones | Provider sandbox, or a fixed test number | Real SMS and email |
| Who can deploy | Any engineer, on merge to a `dev` branch or a manual sync | Automatic when `main` merges and CI is green | A person approves after stage has run that image |
| Who gets alarms | Email to the engineer who deployed. No night pages | Email. No night pages | Email and chat. Payment and order alarms page someone |
| Size | 1 small EKS node, 1 Aurora cluster min 0.5 ACU, Redis micro. Stop the node outside working hours | 2 `m6g.large` nodes, 1 Aurora cluster min 0.5 ACU | Launch size in [AWS-ARCHITECTURE.md](./AWS-ARCHITECTURE.md) section 12 |
| Gitops folder | `gitops/dev/` | `gitops/staging/` | `gitops/production/` |

Rules that keep the three environments apart:

- An image is built once. Dev, stage, and prod run that digest. Prod is never a separate build.
- Secrets live in Secrets Manager inside that environment’s account: `buyymart/dev/{service}`, `buyymart/stage/{service}`, `buyymart/prod/{service}`.
- Live payment keys, live SMS keys, and production KYC files exist only in the production account.
- Database restores, experiments, and schema trials happen in dev. Stage is treated as a dress rehearsal. Prod is changed only by gitops.
- Terraform describes the accounts, networks, and data stores. Clicking in the AWS console is for reading, not for changing size or keys.

Shared pieces sit in `buyymart-shared` and `buyymart-management`, not inside the three environments: the container registry (ECR), the public DNS zone, Terraform state, billing, and human login (IAM Identity Center with MFA).

```text
Laptop (Docker Compose)
        │  push
        ▼
GitHub Actions  →  build image once  →  ECR (shared account)
        │
        ├─ gitops/dev/         →  Argo CD  →  buyymart-dev
        ├─ gitops/staging/     →  Argo CD  →  buyymart-staging
        └─ gitops/production/  →  Argo CD  →  buyymart-production
                                      ▲
                               manual approval
```

---

## 3. Roadmap

Work top to bottom. A later phase starts when the previous phase has a demo you can click.

### Phase 0 — Plan (no AWS yet)

Freeze the product choices that change screens and licences. They are listed as open decisions in the app specification: launch categories, launch pin codes, whether COD is on, and the legal name on invoices.

Read, in this order:

1. [APP-SPECIFICATION.md](./APP-SPECIFICATION.md) — what the three web apps must do.
2. [SERVICES.md](./SERVICES.md) — which service owns which data.
3. [AWS-ARCHITECTURE.md](./AWS-ARCHITECTURE.md) — accounts, network, databases, Kubernetes.
4. [STARTUP-DOCUMENTS.md](./STARTUP-DOCUMENTS.md) — company, GST, and policies. The public site needs these before prod takes an order.

Done when: launch city, categories, and COD are written down, and you can name the service that owns an order (`order-service`).

### Phase 1 — Accounts and access

Create AWS Organizations. Humans sign in with IAM Identity Center and MFA. No one keeps a long-lived access key on a laptop.

| Account | What it is for |
| --- | --- |
| `buyymart-management` | Billing, SSO, CloudTrail, GuardDuty. No workloads. |
| `buyymart-shared` | ECR, Route 53, Terraform state, CI role. |
| `buyymart-dev` | Dev cluster and disposable data. |
| `buyymart-staging` | Stage cluster and test-mode payments. |
| `buyymart-production` | Prod cluster and live keys. Create this account in Phase 1, and deploy workloads into it only in Phase 6. |

Engineers get admin on dev, admin on stage, and read-only on prod. Production deploy is a GitHub Actions role, plus a break-glass SSO role with MFA.

Done when: you can sign in to dev and stage with MFA, and prod has no permanent access keys.

### Phase 2 — Platform, empty of BuyyMart services

Build the platform with Terraform in `buyymart-platform`. Apply it to **dev first**, then copy the same modules to stage with the stage sizes, then to prod when you are ready to go live.

One-time platform pieces:

| Piece | Why you need it before any service |
| --- | --- |
| VPC, private subnets, NAT | Services run on a private network. Only the load balancer is public. |
| ECR repositories | CI needs a place to push images. |
| EKS cluster | Runs the services. |
| ALB controller and ExternalDNS | Gives `api.dev.buyymart.com` (and stage, prod) a path into the cluster. |
| External Secrets | Puts Secrets Manager values into pods. |
| Aurora, Redis, S3, EventBridge, SQS | Data, cache, files, events. Dev and stage can start with one Aurora cluster and one database per service. |
| Argo CD | Watches `gitops/{env}` and deploys. |
| Fluent Bit, Managed Prometheus, Grafana, X-Ray | Logs, metrics, and traces from the first deploy, including an empty cluster. |
| GitHub OIDC role | CI assumes a role. CI does not store an AWS key. |

Local Docker Compose is part of this phase. It runs the same service names so you can code without waiting for the cluster. Use LocalStack only when you are testing S3 or SQS.

Done when: Argo CD in dev shows a healthy cluster, Grafana has a board, and a hello image answers `GET /health/ready` on the dev API host.

### Phase 3 — Services, in the order checkout needs them

Do not start all 17 services at once. Each step below is one deployable slice. Finish it in dev, promote that image to stage, and only then start the next step. Prod stays empty until Phase 6.

| Step | Build these | You can demo |
| --- | --- | --- |
| 1 | `identity-service`, `customer-bff` | A phone OTP login on the dev storefront. Session survives a refresh. |
| 2 | `catalogue-service`, `media-service`, `search-service`, customer web product page | Upload an image, see a product page, search finds it. |
| 3 | `inventory-service`, `cart-service` | Add to cart. Stock on the page matches inventory. Guest cart merges after login. |
| 4 | `order-service`, `payment-service`, payment webhook | Checkout in gateway **test mode**. Order stays `payment_pending` until the webhook. A repeated webhook does not double-charge. |
| 5 | `seller-service`, `seller-bff`, `admin-bff`, Seller Centre, Admin Console | Staff approve KYC. Seller lists a product. Admin can see the order. |
| 6 | `fulfilment-service`, `notification-service` | Pin code returns a fee. After payment, a courier label exists and the customer gets an SMS in stage (log line in dev). |
| 7 | `promotion-service`, `support-service` | A coupon changes the total. A ticket opens on an order. |
| 8 | `settlement-service` | A payout statement matches the test order: sale, commission, tax, net. |

Web apps deploy as static files to S3 and CloudFront in the same step as the BFF they call. Customer web starts at step 1. Seller Centre and Admin Console start at step 5.

Every service, from the first one, ships with:

- `GET /health/live` and `GET /health/ready`
- JSON logs with `time`, `level`, `service`, `requestId`, `traceId`
- `/metrics`
- A database migration job that runs before the new pods
- A row in `gitops/dev/` and, after the demo works, `gitops/staging/`

### Phase 4 — Pipeline as the only way to deploy

After step 1, stop deploying by hand. The path for every later service:

```text
feature branch
  → unit tests and contract tests
  → docker build
  → image scan, fail on a critical CVE that has a fix
  → push digest to ECR
  → pull request updates gitops/dev with that digest
  → Argo CD syncs dev
  → you click through the demo on dev.buyymart.com
merge to main
  → pull request updates gitops/staging with the same digest
  → Argo CD syncs stage
  → smoke test on stage (section 5)
manual approval
  → pull request updates gitops/production with the same digest
  → Argo CD syncs prod
  → canary for order-service and payment-service (10%, then 100%)
  → other services rolling update, maxUnavailable 0, maxSurge 1
```

`payment-service` and `settlement-service` need a second reviewer on the production gitops pull request.

Rollback is a git revert of the digest. Argo CD runs the previous image. Database migrations are forward-only: the old code must still run on the new schema.

### Phase 5 — Monitoring before real traffic

Turn alarms on in stage as soon as checkout exists (Phase 3 step 4). Copy the same alarms to prod before the first live payment.

| Signal | Where it goes | What you use it for |
| --- | --- | --- |
| Logs | stdout JSON → Fluent Bit → CloudWatch `/buyymart/{env}/{service}` | An incident. Find one `requestId` across services. |
| Metrics | `/metrics` → Amazon Managed Prometheus → Grafana | Latency, errors, CPU, queue depth. One board per service, one board for checkout. |
| Traces | OpenTelemetry → ADOT collector → X-Ray | Follow one checkout through BFF, order, inventory, and payment. |
| Alarms | Grafana or CloudWatch → SNS → email. Prod also notifies a chat channel | Wake someone only in prod, and only for the rows below. |
| Audit | Domain database row, plus CloudTrail for AWS API calls | Who refunded, who approved KYC, who changed infrastructure. |

Alarms that page someone in **prod** (stage sends email only):

| Condition | Why it matters |
| --- | --- |
| Payment webhook 5xx over 1% for 5 minutes | Orders sit unpaid |
| SQS oldest message over 2 minutes on order or payment queues | Status is drifting |
| Any dead-letter queue depth at least 1 for 10 minutes | A consumer is rejecting events |
| `order-service` or `payment-service` available replicas under 2 | Checkout is at risk |
| Aurora CPU over 80% for 15 minutes, or free storage low | Writes will stall |
| NAT Gateway packets dropped | Gateway, SMS, and courier calls fail |
| WAF blocked spike on the OTP route | SMS cost or a credential attack |

Logs omit OTP values, tokens, PAN, Aadhaar, and full bank account numbers. Payment and order logs in prod stay searchable for 30 days and archive to S3 for 1 year.

A daily habit once stage is up: open the checkout Grafana board and the dead-letter queues. Both should be quiet.

### Phase 6 — Go live

Prod gets its first services only after stage has run the checkout smoke test on the image you intend to ship.

Before the first live rupee:

- Policies, legal name, and grievance officer are on the public site.
- Production gateway is in live mode. Stage is still test mode.
- WAF is on, with a rate limit on OTP.
- Payment and order backups: 35 days point-in-time recovery, copy to `ap-south-2`.
- Restore drill: restore the payment database to a throwaway cluster and read one payment row.
- Canary rollout is on for `order-service` and `payment-service`.
- The prod alarms in Phase 5 notify a person, and that person has the break-glass role.
- Finance can match one stage-day of test payments to the gateway test settlement. The same report runs in prod on day one.

Done when: a phone on the real site can search, pay, and receive a shipment in the launch pin codes, and a refund in admin shows on the gateway and on the order.

### Phase 7 — After launch

Scale by changing Terraform and gitops, then letting the pipeline apply it. Festival week: raise Aurora max ACU, raise autoscaler limits, and raise Karpenter limits the week before.

Later product work, from the app specification, waits until daily orders reconcile to the bank:

- Android, then iOS, both calling `customer-bff`
- Hindi
- Bulk catalogue upload
- Automated settlement file, still with a finance approval
- Warehouse Console only if BuyyMart stores stock

---

## 4. What you do on a normal day

1. Pull `main`. Create a branch.
2. Run the service in Docker Compose. Point it at local Postgres or the dev database, never at stage or prod.
3. Open a pull request. CI runs tests and builds an image.
4. Merge the dev gitops change. Click the flow on `dev.buyymart.com`.
5. Merge to `main`. Stage deploys the same digest. Run the smoke test.
6. Approve production only when that smoke test passed on stage.

A database change ships inside the service image as a migration job. You do not run SQL by hand against stage or prod.

---

## 5. Smoke test that must pass on stage

Run this against `api.stage.buyymart.com` before any production approval, once checkout exists.

1. Request an OTP for a test phone. Log in.
2. Open a product. Confirm stock.
3. Add it to the cart. Apply a coupon if that service is deployed.
4. Check out with a launch pin code. Confirm the shipping fee.
5. Pay with a gateway **test** card or test UPI.
6. Confirm the order becomes `paid` from the webhook, including when the webhook is delivered twice.
7. Confirm a label or a “label pending” state, and a notification log or test SMS.
8. In Admin, open the order and issue a test refund. Confirm the gateway shows the refund.

If a step has no service yet, skip it and write that in the pull request. Do not skip the payment webhook check once `payment-service` is deployed.

---

## 6. Where to look when something is wrong

| What you see | First place to look |
| --- | --- |
| The site loads and the API does not | ALB target health, then `customer-bff` logs for that `requestId` |
| Login never sends a code | `identity-service` and `notification-service` logs. In dev, the code is in the log, not on a phone |
| Cart or product page is empty | `catalogue-service`, then `search-service` index, then `inventory-service` |
| Pay button fails | `payment-service` logs and the gateway dashboard for that environment |
| Order stays `payment_pending` | Payment webhook deliveries, then the payment dead-letter queue |
| Order paid, seller sees nothing | `OrderConfirmed` queue age, then `fulfilment-service` |
| Stage works, prod does not | Compare the image digest in `gitops/staging` and `gitops/production`. Then compare secrets, not code |

---

## 7. Suggested calendar

One engineer, learning cloud while building. Dates slip if company paperwork or the payment-gateway review slips. The order does not.

| Weeks | Focus | Environment you should have |
| --- | --- | --- |
| 1 | Phase 0 decisions, AWS accounts, MFA, empty dev cluster, hello health check, Grafana | Dev |
| 2–3 | Identity and customer BFF, CI, gitops dev and stage | Dev and stage |
| 4–5 | Catalogue, media, search, product page | Dev and stage |
| 6–7 | Inventory and cart | Dev and stage |
| 8–9 | Order, payment, webhook, stage smoke test | Dev and stage |
| 10–11 | Seller, admin, KYC | Dev and stage |
| 12–13 | Fulfilment and notification | Dev and stage |
| 14 | Coupons, support, settlement statement | Dev and stage |
| 15 | Prod platform, restore drill, live gateway, first real order | Dev, stage, and prod |

Prod exists as an account from week 1 and as a running store from week 15.
