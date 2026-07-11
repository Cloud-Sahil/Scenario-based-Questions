# AWS Scenario-Based Interview Questions
---

### 1. Your application runs fine in India but is slow/not accessible for US users. Why?
**Answer:** Most common reasons:
- App/EC2 is hosted in `ap-south-1` (Mumbai) only — no region closer to US users → high latency.
- No **CloudFront (CDN)** in front of the app, so every request travels all the way to India.
- No **multi-region deployment** or **Global Accelerator** to route US traffic to a nearby region.
- Route 53 **latency-based routing / geolocation routing** not configured.
- DNS TTL too high, so failover/routing changes take time to propagate.
**Fix:** Add CloudFront with edge locations, use Route 53 latency-based routing, and consider deploying app in a US region (e.g., `us-east-1`) with cross-region replication for DB (Aurora Global Database / DynamoDB Global Tables).

---

### 2. What are the types of IAM policies?
**Answer:**
- **Identity-based policies** – attached to Users, Groups, Roles (Managed – AWS or Customer managed, and Inline).
- **Resource-based policies** – attached directly to a resource (e.g., S3 bucket policy, SQS policy).
- **Permission boundaries** – set the max permissions an identity-based policy can grant.
- **Organizations SCP (Service Control Policy)** – applies at AWS Organization/OU level, controls max permissions for accounts.
- **Session policies** – passed during `AssumeRole`, limits permissions for that session only.
- **ACLs (Access Control Lists)** – legacy, resource-level (mostly S3), cross-account access without IAM.

---

### 3. What are the different EC2 instance types? When do you use which?
**Answer:**
- **General Purpose (T, M series)** – balanced CPU/memory. Example: web servers, small DBs. (t3, m5)
- **Compute Optimized (C series)** – high CPU. Example: batch processing, gaming servers, video encoding. (c5, c6i)
- **Memory Optimized (R, X, z series)** – large RAM. Example: in-memory DBs (Redis), big data processing. (r5, x1)
- **Storage Optimized (I, D, H series)** – high, fast local disk I/O. Example: NoSQL DBs, data warehousing. (i3, d2)
- **Accelerated Computing (P, G, Inf series)** – GPU/ML workloads. Example: ML training, video rendering. (p3, g4)
- Interview tip: they may ask "which instance type for a Jenkins master vs a build agent vs a Redis cache" — Jenkins master = general purpose (t3/m5), Redis cache = memory optimized (r5).

---

### 4. Checklist: VPC, VPC Peering, Transit Gateway, CDN, Edge Location, Database Replication Cross-Region
**Answer (explain each in 1-2 lines, this is how interviewer wants it):**
- **VPC** – Your isolated private network in AWS with subnets, route tables, IGW/NAT.
- **VPC Peering** – 1-to-1 private connection between two VPCs (same or different account/region). No transitive routing — if A↔B and B↔C peered, A cannot talk to C directly.
- **Transit Gateway (TGW)** – Hub-and-spoke model to connect many VPCs + on-prem networks centrally. Supports transitive routing (unlike peering). Used when you have 10+ VPCs to connect.
- **CDN (CloudFront)** – Caches content at edge locations close to users, reduces latency, offloads origin server.
- **Edge Location** – Physical CloudFront location (200+ globally) that caches content nearest to the end user.
- **Database Replication Cross-Region** – Copying DB data to another AWS region for DR/low-latency reads. Examples: Aurora Global Database, RDS Cross-Region Read Replica, DynamoDB Global Tables, S3 Cross-Region Replication (CRR).
**Follow-up they may ask:** "When will you use VPC Peering vs Transit Gateway?" → Peering for 2-3 VPCs, TGW for scale (many VPCs/accounts) and centralized routing/security.

---

### 5. What is Site-to-Site VPN in AWS? When do you use it?
**Answer:** Site-to-Site VPN connects your **on-premise data center** to your **AWS VPC** over the public internet using IPSec tunnels (encrypted). It needs:
- **Customer Gateway (CGW)** – represents your on-prem router/firewall.
- **Virtual Private Gateway (VGW)** or **Transit Gateway** – AWS side endpoint.
Use case: Company has a physical office/data center and wants secure, permanent connectivity to AWS resources (hybrid cloud) without using Direct Connect (which is costlier and needs physical setup).

---

### 6. What is Point-to-Site VPN? How is it different from Site-to-Site?
**Answer:**
- **Site-to-Site VPN** = network-to-network (whole office network connects to VPC).
- **Point-to-Site VPN (Client VPN in AWS)** = single device/laptop connects to VPC, e.g., a remote employee connecting from home using AWS **Client VPN endpoint** with OpenVPN-based client software.
Use case: Developer working remotely needs secure access to private resources (like a private RDS or internal app) without exposing them to the public internet — they install a VPN client and connect individually.

---

### 7. How do you transfer data from AWS to Azure (or any cross-cloud transfer)?
**Answer:** Options depend on data size and frequency:
- **Small/one-time data:** Direct API/SDK calls, `azcopy` with S3 as source, or manual download/upload.
- **Large one-time migration:** Use **AWS DataSync** (supports transferring to on-prem/other endpoints) or export to S3 → transfer via internet/Direct Connect+ExpressRoute combo.
- **Ongoing/continuous sync:** Set up a pipeline (e.g., Lambda triggered on S3 upload → pushes to Azure Blob via Azure SDK), or use third-party tools like **Rclone**.
- **Network-level:** If frequent large transfers, consider dedicated connectivity — AWS Direct Connect to a colocation point, then Azure ExpressRoute from the same location (cloud interconnect), to avoid public internet.
- **Security consideration:** Encrypt data in transit (TLS), use least-privilege IAM/Azure AD credentials, and check egress cost from AWS side (data transfer OUT is chargeable).

---

### 8. In Nexus, multiple pipelines share one artifact — how do you manage this? What will you consider?
**Answer:** (Same pattern as your Jenkins RBAC example)
- Use a **shared Nexus repository** (e.g., a common `releases` or `snapshots` repo) where the artifact is published once with a proper **version number**.
- Other pipelines/jobs **pull (download)** the artifact by version instead of rebuilding it — avoids duplicate builds and keeps consistency.
- **Access control (RBAC)** in Nexus: create roles/privileges so only the publishing pipeline has **write/deploy** access, and consumer pipelines only have **read/download** access.
- Consider: **artifact versioning strategy** (semantic versioning, avoid overwriting `latest` in prod), **retention policy** (cleanup old snapshots), **checksum verification** to ensure integrity, and **credentials management** (Nexus API token/service account per pipeline, not shared personal credentials).

---

### 9. What is the difference between S3 storage classes? When to use which?
**Answer:**
- **S3 Standard** – frequently accessed data, low latency. (active app data)
- **S3 Intelligent-Tiering** – unknown/changing access pattern, auto-moves data between tiers.
- **S3 Standard-IA / One Zone-IA** – infrequent access, cheaper storage, retrieval cost applies. (backups)
- **S3 Glacier / Glacier Deep Archive** – archival, very cheap, retrieval takes minutes to hours. (compliance/audit logs, 7-year retention)
Interview scenario: "You have logs accessed daily for 30 days then rarely touched" → Lifecycle policy: Standard → IA after 30 days → Glacier after 90 days.

---

### 10. Your EC2 instance can't connect to the internet. How do you troubleshoot?
**Answer:** Step-by-step checklist:
1. Check if instance is in a **public subnet** with a route to **Internet Gateway (IGW)** in route table.
2. Check if instance has a **public IP or Elastic IP** attached.
3. Check **Security Group** outbound rules (allow required ports/protocols).
4. Check **NACL** (Network ACL) — both inbound and outbound rules, since NACLs are stateless.
5. If in private subnet, check if there's a **NAT Gateway/NAT Instance** for outbound internet access.
6. Check OS-level firewall (iptables/firewalld) on the instance itself.
7. Check DNS resolution — `nslookup`/`dig` to rule out DNS issues.

---

### 11. What is the difference between Security Group and NACL?
**Answer:**
| Security Group | NACL |
|---|---|
| Instance level | Subnet level |
| Stateful (return traffic auto-allowed) | Stateless (must allow both directions) |
| Only "Allow" rules | Both "Allow" and "Deny" rules |
| Evaluates all rules | Evaluates rules in order (lowest number first) |

---

### 12. Your S3 bucket is public by mistake. How do you fix/prevent this?
**Answer:**
- Enable **S3 Block Public Access** at bucket/account level.
- Review **bucket policy** and **ACLs** — remove any `Principal: "*"` with public access.
- Use **AWS Config rule** (`s3-bucket-public-read-prohibited`) to continuously monitor.
- Enable **AWS Trusted Advisor / Security Hub** for automated detection.
- Use **IAM policies with least privilege** and **SCPs** to prevent public bucket creation org-wide.

---

### 13. How do you design a Highly Available (HA) architecture in AWS?
**Answer:**
- Deploy across **multiple Availability Zones (Multi-AZ)**.
- Use **Auto Scaling Group** to handle instance failure/load.
- Use **Elastic Load Balancer (ALB/NLB)** to distribute traffic.
- **RDS Multi-AZ** for database failover.
- **Route 53 health checks** for DNS failover.
- For DR, consider **multi-region** with S3 CRR / Aurora Global Database.

---

### 14. What is the difference between ALB, NLB, and CLB?
**Answer:**
- **ALB (Application LB)** – Layer 7, works with HTTP/HTTPS, supports path/host-based routing, good for microservices.
- **NLB (Network LB)** – Layer 4, handles millions of requests/sec, ultra-low latency, static IP support — used for TCP/UDP traffic.
- **CLB (Classic LB)** – legacy, Layer 4 & 7 basic support, mostly deprecated now.

---

### 15. How does Auto Scaling work? What triggers it?
**Answer:** Auto Scaling Group (ASG) maintains desired instance count based on:
- **CloudWatch alarms** (e.g., CPU > 70% for 5 min → scale out).
- **Scheduled scaling** (e.g., scale up every day at 9 AM for office hours).
- **Target tracking policy** (maintain average CPU at 50%).
- **Predictive scaling** (uses ML to forecast traffic).
Components: Launch Template, Min/Max/Desired capacity, Health checks (EC2 + ELB), Scaling policies.

---

### 16. What is the difference between EBS and Instance Store?
**Answer:**
- **EBS (Elastic Block Store)** – persistent, survives instance stop/terminate (if not deleted), can be snapshotted, attached/detached.
- **Instance Store** – temporary, physically attached to host, data lost on stop/terminate, very high I/O performance — used for cache/temp/buffer data.

---

### 17. How do you troubleshoot a Lambda function that's timing out?
**Answer:**
- Check **CloudWatch Logs** for the function to see where it's stuck.
- Increase **timeout** setting (max 15 min) if the operation genuinely needs more time.
- Check if Lambda is inside a **VPC without NAT** — trying to reach internet but stuck (common gotcha).
- Check **memory allocation** — low memory = low CPU = slow execution; increasing memory can speed up CPU-bound work.
- Check downstream dependency (DB/API) latency — the bottleneck might not be Lambda itself.
- Enable **X-Ray tracing** to pinpoint the exact slow segment.

---

### 18. What's the difference between Cross-Region Replication (CRR) and Same-Region Replication (SRR) in S3?
**Answer:**
- **CRR** – replicates objects to a bucket in a different region. Use for DR, compliance (data residency), latency reduction for global users.
- **SRR** – replicates within the same region to another bucket. Use for log aggregation, account separation, compliance backups within region.
Both require **versioning enabled** on source and destination buckets.

---

### 19. How do you secure secrets (DB passwords, API keys) in AWS?
**Answer:**
- Use **AWS Secrets Manager** (auto-rotation supported) or **Systems Manager Parameter Store (SecureString)**.
- Never hardcode secrets in code/user-data/AMI.
- Use **IAM roles** for EC2/Lambda instead of embedding access keys.
- Encrypt with **KMS**.
- In CI/CD (Jenkins), use **Jenkins Credentials plugin** and inject at runtime, not stored in Jenkinsfile.

---

### 20. Your CloudFormation/Terraform stack fails midway. How do you handle it?
**Answer:**
- Check the **event log** (CFN console "Events" tab, or `terraform plan/apply -detailed-exitcode`) for the exact failing resource and reason.
- CloudFormation: stack auto-**rollback** on failure by default (can disable for debugging with `--disable-rollback`).
- Terraform: state file may be **partially applied** — check `terraform state list`, fix the issue, re-run `apply` (Terraform is generally idempotent).
- Common causes: IAM permission missing, resource limit/quota exceeded, naming conflict (resource already exists), dependency ordering issue.
- Best practice: use **terraform plan** before every apply, and state locking (S3 + DynamoDB) to avoid concurrent modification issues.

---

### 21. What is the difference between RDS Multi-AZ and Read Replica?
**Answer:**
- **Multi-AZ** – synchronous standby copy in another AZ, used for **HA/failover** only, automatic failover during outage/maintenance.
- **Read Replica** – asynchronous copy, used for **read scaling**, can be in same or different region, can be manually promoted but isn't automatic failover.
- Scenario: "App has heavy read traffic and needs HA" → use Multi-AZ for HA + Read Replicas for scaling reads.

---

### 22. What is the difference between AWS managed keys and Customer managed keys (CMK) in KMS?
**Answer:**
- **AWS managed keys** – visible in your account (`aws/s3`, `aws/rds`), rotated automatically, limited policy control.
- **Customer managed keys (CMK)** – full control: define key policy, rotation, enable/disable, grant cross-account access. Used when compliance requires custom control over encryption keys.

---

### 23. How do you give S3 bucket access to another AWS account (cross-account access)?
**Answer:**
- **Bucket policy** on source bucket with the other account's ID/role ARN as Principal and required actions (`s3:GetObject` etc.).
- Or use an **IAM role** the other account can `AssumeRole` into.
- If encrypted with **KMS CMK**, also update the **KMS key policy** to allow the other account — a common miss causing "access denied" even when bucket policy looks correct.

---

### 24. What is the difference between CloudWatch and CloudTrail?
**Answer:**
- **CloudWatch** – monitoring: metrics, logs, alarms, dashboards (performance/health, e.g. CPU usage, error count).
- **CloudTrail** – audit: logs **who did what** (API calls) — e.g., "who deleted this S3 bucket, from which IP, at what time." Used for security auditing/compliance.

---

### 25. What is the difference between ECS and EKS? When would you pick one over the other?
**Answer:**
- **ECS** – AWS's own simpler container orchestration, tightly AWS-integrated, easier to learn.
- **EKS** – managed **Kubernetes**, more powerful/portable, steeper learning curve, more operational overhead.
- Pick ECS for AWS-only simpler workloads. Pick EKS if team already knows Kubernetes or needs portability/K8s tooling.

---

### 26. What causes Lambda cold starts and how do you reduce them?
**Answer:**
- Cold start = time to initialize a new execution environment when no warm instance is available.
- Causes: low traffic, large deployment package, VPC-attached Lambda, heavy init code.
- Reduce by: keeping package size small, using **Provisioned Concurrency**, lighter runtimes, initializing DB connections outside the handler.

---

### 27. What's the difference between a Gateway VPC Endpoint and an Interface VPC Endpoint?
**Answer:**
- **Gateway Endpoint** – only for **S3 and DynamoDB**, free, works via route table entry, no ENI.
- **Interface Endpoint (PrivateLink)** – for most other AWS services, creates an **ENI** with private IP in your subnet, charged per hour.
- Use case: keep traffic to AWS services within the VPC instead of via internet/NAT.

---

### 28. What is AWS WAF and Shield? How do they differ?
**Answer:**
- **Shield** – protects against **DDoS attacks** (L3/L4). Standard is free/automatic; Advanced adds 24/7 response team + cost protection.
- **WAF** – protects against **application layer (L7) attacks** — SQL injection, XSS, bad bots — via rules on CloudFront/ALB/API Gateway.
- Used together: Shield for network-level, WAF for app-level.

---

### 29. How do you optimize AWS costs for a production workload?
**Answer:**
- **Right-sizing** instances based on actual CloudWatch usage.
- **Reserved Instances/Savings Plans** for steady workloads; **Spot Instances** for fault-tolerant/batch workloads.
- **S3 lifecycle policies** to move infrequent data to cheaper tiers.
- Delete unused resources — unattached EBS volumes, old snapshots, idle Elastic IPs.
- Use **Cost Explorer / Trusted Advisor / Compute Optimizer**, and **tagging strategy** for cost visibility per team.

---

### 30. How do you design a secure Bastion Host setup?
**Answer:**
- Bastion in **public subnet**, Security Group allowing SSH only from known/office IP (not `0.0.0.0/0`).
- Private instances only allow SSH from Bastion's Security Group.
- Better modern alternative: **AWS Systems Manager Session Manager** — no bastion needed, no open port 22, access controlled via IAM, every session logged automatically.

---

### 31. What are the different Route 53 routing policies?
**Answer:**
- **Simple** – single resource, no health check logic.
- **Weighted** – split traffic by percentage (A/B testing, canary).
- **Latency-based** – routes to region with lowest latency (multi-region app for global users — answers your Q1 scenario).
- **Failover** – active-passive, routes to secondary if primary health check fails (DR).
- **Geolocation** – routes based on user's actual location (compliance, e.g. EU users must hit EU servers).
- **Geoproximity** – location-based with adjustable "bias".
- **Multi-value answer** – returns multiple healthy IPs, basic client-side load balancing.

---

### 32. What is the difference between Direct Connect and Site-to-Site VPN?
**Answer:**
- **Direct Connect** – dedicated physical private connection, consistent low latency & high bandwidth, expensive, takes weeks to provision, not encrypted by default.
- **Site-to-Site VPN** – over public internet, IPSec encrypted, quick setup, cheaper, variable bandwidth.
- Scenario: bank needing consistent low latency → Direct Connect (+VPN over DX for encryption). Startup needing quick secure connectivity → Site-to-Site VPN.

---

### 33. Explain AWS Disaster Recovery strategies: Backup & Restore, Pilot Light, Warm Standby, Multi-Site.
**Answer:** (Low cost/high RTO → high cost/low RTO)
- **Backup & Restore** – only backups in another region, restore on disaster. Cheapest, RTO in hours.
- **Pilot Light** – core minimal infra always running in DR region, rest spun up on demand.
- **Warm Standby** – scaled-down but functional copy always running, scaled up on failover.
- **Multi-Site (Active-Active)** – full capacity in two regions simultaneously, traffic split via Route 53. Fastest RTO, most expensive.
- Follow-up: banking app → Multi-Site/Warm Standby; internal tool → Backup & Restore.

---

### 34. Your app needs to process millions of small files uploaded to S3 in near real-time. How do you architect this?
**Answer:**
- **S3 Event Notification** → triggers **Lambda** or pushes to **SQS/SNS** for decoupled, scalable processing.
- Use **SQS** as buffer to avoid throttling, with retry/DLQ for failed items.
- For heavy multi-step processing, use **Step Functions**; for continuous streaming, use **Kinesis**.
- Consider S3 prefix design to avoid request rate hot-spotting.

---

### 35. What is the difference between Synchronous and Asynchronous invocation in Lambda?
**Answer:**
- **Synchronous** – caller waits for response (API Gateway/ALB → Lambda). Errors returned directly.
- **Asynchronous** – caller doesn't wait (S3/SNS/EventBridge → Lambda). Lambda retries automatically (2 retries by default); configure a **DLQ/on-failure destination** to capture events failing after retries.

---

### 36. How do you migrate an on-prem database to RDS with minimal downtime?
**Answer:**
- Use **AWS Database Migration Service (DMS)**: full load of existing data → **CDC (Change Data Capture)** keeps replicating ongoing changes → once in sync, do final cutover in a small maintenance window.
- For different DB engines (e.g., Oracle → PostgreSQL), use **Schema Conversion Tool (SCT)** first.

---

### 37. ASG isn't replacing an instance even though the app inside it crashed. Why?
**Answer:** ASG's default health check only checks **EC2 instance status** (is it running?), not the application. If health check type is set to "EC2" instead of "ELB," a crashed app inside a running instance won't trigger replacement. Fix: switch health check type to "ELB" so ASG considers app-level health (via target group health check hitting e.g. `/health`).

---

### 38. How do you patch an EC2-based database in a private subnet with no internet access?
**Answer:** Use **VPC Endpoints** for SSM (`ssm`, `ssmmessages`, `ec2messages`) and patch via **Systems Manager Patch Manager** — no internet access needed at all. If patches must come from external repos, route via a **NAT Gateway** with restricted outbound rules.

---

### 39. Two teams in the same AWS Organization need isolated environments but shared central logging/security. How do you structure this?
**Answer:**
- **AWS Organizations** with separate accounts per team (hard isolation).
- Central **Log Archive account** – CloudTrail/Config/VPC Flow Logs from all accounts via cross-account S3.
- Central **Security/Audit account** – GuardDuty, Security Hub aggregation.
- **SCPs** at OU level for guardrails; **IAM Identity Center (SSO)** for centralized access across accounts.

---

### 40. CloudFront is serving stale content after you updated the S3 origin. How do you fix it?
**Answer:** CloudFront caches based on TTL — origin update doesn't auto-refresh cache. Fix: create an **invalidation** (`aws cloudfront create-invalidation`). Better long-term: use **versioned/content-hash file names** instead of invalidating every time, and set proper **Cache-Control headers**.

---

### 41. What does "stateful" mean for a Security Group, in practice?
**Answer:** If you allow inbound traffic on port 443, the **response traffic is automatically allowed outbound** without an explicit outbound rule — SG "remembers" the connection. Unlike NACLs, where both directions must be explicitly allowed.

---

### 42. How do you make an S3 bucket accessible only from within your VPC, not the public internet?
**Answer:** Use a **VPC Endpoint (Gateway type)** for S3, combined with a **bucket policy** using the `aws:sourceVpce` condition to only allow requests through that specific endpoint — blocks access even for a valid IAM user going through the public internet.

---

### 43. What happens if you delete a CloudFormation stack that has an RDS instance in it? How do you prevent data loss?
**Answer:** By default, deleting the stack deletes RDS too (data loss) unless you set **DeletionPolicy: Retain** or `Snapshot` on the resource — `Snapshot` takes a final backup before deletion, `Retain` leaves the resource untouched.

---

### 44. Your app in a private subnet needs to call an external third-party API. How do you allow this securely?
**Answer:** Set up a **NAT Gateway** in a public subnet, add a route in the private subnet's route table (`0.0.0.0/0` → NAT Gateway). Instance can initiate outbound connections but remains unreachable from internet inbound.

---

### 45. What's the difference between an Elastic IP and a normal public IP on EC2?
**Answer:** A normal public IP changes if you stop/start the instance. An **Elastic IP** is static and can be remapped to another instance — useful when external systems depend on a fixed IP (though ALB + DNS is usually preferred for modern setups).

---

### 46. How do you prevent someone from accidentally deleting a production S3 bucket or RDS instance?
**Answer:** Enable **S3 Versioning + MFA Delete**, use **RDS Deletion Protection**, apply **IAM policies/SCPs** denying delete actions for non-admin roles, and use **tag-based IAM conditions** to restrict destructive actions on `Environment=Production` resources.

---

### 47. What's the difference between an internet-facing ALB and an internal ALB?
**Answer:** **Internet-facing ALB** has a public DNS resolving to public IPs, in public subnets, accessible from internet. **Internal ALB** resolves to private IPs, in private subnets, only accessible within the VPC — used for internal microservice-to-microservice traffic.

---

### 48. You need to give a vendor temporary access to specific S3 objects without creating an IAM user. How?
**Answer:** Generate a **Pre-signed URL** (`aws s3 presign`) with expiration — grants temporary, time-limited access to a specific object without AWS credentials on their side.

---

### 49. How do you audit which IAM users/roles have overly permissive (`*:*`) access?
**Answer:** Use **IAM Access Analyzer**, AWS Config rule (`iam-policy-no-statements-with-admin-access`), or **Trusted Advisor**. Also review **IAM Access Advisor** (last-used timestamps per service) to remove unused excessive permissions — enforce least privilege.

---

### 51. What is the difference between an Auto Scaling Group Launch Template and Launch Configuration?
**Answer:** **Launch Configuration** – older, immutable (can't be edited, must create new one for changes), doesn't support multiple instance types/mixed purchase options. **Launch Template** – newer, supports **versioning** (edit and create new version, roll back to old version), supports mixing On-Demand + Spot instances in one ASG, and supports newer EC2 features. AWS recommends Launch Templates for all new ASGs; Launch Configurations are being deprecated.

---

### 52. How do you set up a highly available NAT solution — what happens if a NAT Gateway's AZ goes down?
**Answer:** A NAT Gateway is **AZ-scoped** — if you only have one NAT Gateway and its AZ fails, private subnets in other AZs routing through it lose internet access. Best practice: deploy **one NAT Gateway per AZ**, with each AZ's private subnet route table pointing to the NAT Gateway in its *own* AZ — this avoids cross-AZ data transfer charges too and gives true HA (one NAT Gateway failing doesn't affect other AZs).

---

### 53. What is the difference between an EBS snapshot and an AMI?
**Answer:** An **EBS snapshot** is a point-in-time backup of a single volume's data (incremental, stored in S3 internally). An **AMI (Amazon Machine Image)** is a full template to launch new EC2 instances — includes one or more EBS snapshots (root + any additional volumes) plus launch permissions and metadata (OS, architecture). You create an AMI *from* an instance (which internally creates snapshots), but a snapshot alone can't launch an instance directly — it needs to be part of/attached via an AMI or a new volume.

---

### 54. How do you set up cross-account IAM role access (Account A user needs to manage resources in Account B)?
**Answer:**
1. In **Account B**, create an IAM Role with a **trust policy** allowing Account A (its account ID, or a specific role ARN in Account A) to assume it.
2. Attach permission policies to that role defining what it can do in Account B.
3. In **Account A**, the user/role needs an `sts:AssumeRole` permission policy pointing to that role's ARN.
4. User runs `aws sts assume-role --role-arn arn:aws:iam::AccountB:role/RoleName` to get temporary credentials, or configures it directly in AWS CLI profile / console "Switch Role."

---

### 55. What's the difference between DynamoDB On-Demand and Provisioned capacity mode?
**Answer:** **Provisioned** – you specify Read/Write Capacity Units upfront (cheaper for predictable, steady traffic), supports Auto Scaling to adjust within a range. **On-Demand** – pay-per-request, no capacity planning needed, automatically scales instantly to any traffic level (great for unpredictable/spiky traffic), but costs more per request than well-utilized provisioned capacity. Scenario: steady predictable traffic → Provisioned (cheaper); new app with unknown/spiky traffic → On-Demand (simpler, no risk of throttling).

---

### 56. How do you troubleshoot "throttling" errors (e.g., DynamoDB `ProvisionedThroughputExceededException` or API Gateway 429s)?
**Answer:**
- Check **CloudWatch metrics** for `ThrottledRequests`/`ConsumedReadCapacityUnits` vs provisioned limits.
- For DynamoDB: check for a **hot partition** (uneven access pattern on partition key) — redesign partition key for better distribution, or switch to On-Demand mode.
- Implement **exponential backoff and retry** in the application (AWS SDKs do this by default, but verify it's not disabled).
- For API Gateway, check **account/API-level throttle limits** and request a limit increase if genuinely needed, or add caching to reduce request volume.

---

### 57. What is the difference between an Availability Zone and a Region? Why does this matter for architecture?
**Answer:** A **Region** is a geographic area (e.g., `ap-south-1` Mumbai) containing multiple, isolated **Availability Zones (AZs)** — each AZ is one or more physically separate data centers with independent power/networking, but low-latency connected to other AZs in the same region. Matters because: **Multi-AZ** = protection against a data-center-level failure (still same region, so still subject to region-wide issues like a regional service outage); **Multi-Region** = protection against an entire region going down, but adds latency/complexity/cost for cross-region replication.

---

### 58. How would you set up centralized billing and cost control for multiple AWS accounts under one company?
**Answer:** Use **AWS Organizations** with **Consolidated Billing** — one management/payer account, member accounts get combined billing (and volume discounts apply across the whole org). Use **AWS Budgets** with alerts per account/team, **Cost Allocation Tags** for granular tracking, and **SCPs** to prevent runaway spend (e.g., block launching expensive instance types without approval) in dev/sandbox accounts.

---

### 59. Your Lambda function needs to access an RDS database inside a VPC. What do you need to configure?
**Answer:**
1. Attach the Lambda function to the **same VPC** as the RDS instance (select subnets + security group in Lambda config).
2. Ensure the **security group** on Lambda's ENI is allowed in RDS's security group inbound rules (port 3306/5432 etc.).
3. If Lambda also needs internet access (e.g., to call an external API) while in a VPC, route through a **NAT Gateway** — VPC-attached Lambdas don't get internet access by default unless the subnet routes through NAT.
4. Consider connection pooling issues — Lambda can spawn many concurrent executions, each potentially opening a new DB connection; use **RDS Proxy** to pool/manage connections and avoid exhausting the DB's max connections.

---

### 60. What is AWS Config, and how is it different from CloudTrail?
**Answer:** **AWS Config** tracks **resource configuration state and changes over time** — "what did this Security Group's rules look like last Tuesday, and who/what changed it" — and can evaluate compliance rules (e.g., "is encryption enabled on all S3 buckets?"). **CloudTrail** tracks **API call history** (who called what action, when). They're complementary: CloudTrail tells you the action taken; Config tells you the resulting configuration state and whether it's compliant.

---

### 61. How do you handle a scenario where an EC2 instance needs different permissions at different times (e.g., normal operation vs a nightly batch job needing extra S3 access)?
**Answer:** Best practice: avoid attaching overly broad permissions permanently. Options: (1) Use a single IAM role with all necessary permissions if the risk is acceptable and scoped tightly (least privilege still applied for both use cases); (2) Have the nightly batch job **assume a separate, more privileged role** temporarily via `sts:AssumeRole` just for that task's duration, rather than the instance's default role always having elevated access; (3) Separate the batch job onto its own instance/Lambda with its own dedicated role instead of overloading one instance's role.

---

### 62. What's the difference between "stopping" and "terminating" an EC2 instance? What happens to attached resources?
**Answer:** **Stop** – instance shuts down but is not deleted; EBS root volume (if not set to delete on termination... actually persists regardless on stop) persists, you keep the instance ID, can start it again later, but you **lose the public IP** (unless using Elastic IP) and stop paying for compute (still pay for EBS storage). **Terminate** – instance is permanently deleted; EBS volumes are deleted too **unless** "Delete on Termination" was unchecked for that volume; instance ID is gone forever.

---

### 63. How do you design an architecture for an application requiring strict data residency (e.g., all data must stay within India)?
**Answer:** Deploy all resources (EC2, RDS, S3) strictly in `ap-south-1` (Mumbai) region — never enable cross-region replication/backup to other regions. Use **SCPs** to restrict which regions the account can even launch resources in (deny all actions except in approved regions). Check that any managed service used (e.g., CloudFront edge caching) doesn't move data outside allowed jurisdictions if compliance is strict (CloudFront by default may cache at edge locations globally — restrict edge locations if needed via price class settings, or avoid CDN for such data).

---

### 64. What is the difference between a Network Load Balancer (NLB) with a static IP vs using an ALB — when do you specifically need static IPs?
**Answer:** NLB supports assigning an **Elastic IP per AZ**, giving a fixed, predictable IP — needed when downstream clients/firewalls **whitelist specific IPs** (common with legacy partner integrations, some corporate firewalls that only allow listed IPs, not DNS-based rules). ALB doesn't support static IPs directly (its IP can change) — so for IP-allowlisting requirements, NLB is the correct choice even if your traffic is HTTP-based (you lose ALB's L7 features like path routing, but you gain static IP compliance).

---

### 65. Your S3 bucket needs to trigger different Lambda functions for different types of file uploads (e.g., .jpg vs .csv). How do you configure this?
**Answer:** S3 Event Notifications support **filtering by prefix and suffix** — configure separate event notification rules on the same bucket, e.g., suffix `.jpg` → triggers Lambda A, suffix `.csv` → triggers Lambda B. You can have multiple notification configurations on one bucket as long as their filters don't overlap for the same event type.

---

### 66. How do you secure API Gateway endpoints — what are the different authorization options?
**Answer:**
- **IAM authorization** – for internal/AWS-to-AWS calls using SigV4 signing.
- **Cognito User Pools** – for end-user authentication (mobile/web apps), API Gateway validates JWT tokens from Cognito.
- **Lambda Authorizer (custom)** – custom logic (e.g., validate a third-party JWT/API key) run before allowing the request through.
- **API Keys + Usage Plans** – simple key-based access control combined with throttling/quota per key, often used for external partner APIs.
- Also apply **resource policies** to restrict which VPCs/IPs can call a private API Gateway.

---

### 67. What is the difference between an EBS gp2 and gp3 volume? Why would you migrate from gp2 to gp3?
**Answer:** **gp2** – IOPS tied to volume size (3 IOPS per GB, burstable), so smaller volumes get poor baseline performance. **gp3** – IOPS and throughput are **independently configurable**, not tied to size — you can get high IOPS (up to 16,000) even on a small volume, and gp3 is generally **~20% cheaper** than gp2 for the same baseline performance. Migrate by simply modifying the volume type (no downtime) via `aws ec2 modify-volume` — an easy, common cost-optimization win.

---

### 68. Your organization wants to enforce that all EC2 instances must have specific tags (e.g., `Owner`, `Environment`) before they can be launched. How do you enforce this?
**Answer:** Use an **SCP** with a condition checking `aws:RequestTag` to deny `ec2:RunInstances` if required tags are missing, or use **AWS Config** with a rule like `required-tags` combined with **auto-remediation** (e.g., auto-terminate or notify if non-compliant), or enforce via **IAM policy conditions** requiring specific tags at resource creation time. Tag Policies (via AWS Organizations) can also standardize allowed tag keys/values across accounts.

---

### 69. How do you troubleshoot "connection timed out" when trying to reach an RDS instance from an EC2 instance in the same VPC?
**Answer:**
1. Check RDS **Security Group** inbound rules — allows the EC2 instance's Security Group (or its IP) on the DB port (3306/5432).
2. Check RDS is not set to a **different VPC/subnet** unreachable from the EC2 instance (rare, but check VPC/subnet placement).
3. Check RDS **"Publicly Accessible"** setting isn't causing routing confusion (should generally be "No" for internal access, but this setting affects public IP assignment, not internal connectivity directly).
4. Test basic network reachability: `telnet <rds-endpoint> 3306` from the EC2 instance — if it hangs (not "connection refused"), it's almost always a Security Group/NACL issue, not an application/credentials issue.
5. Check NACLs on both subnets allow the traffic in both directions (stateless).

---

### 70. What is the difference between "Reserved Instances" and "Savings Plans"? Which offers more flexibility?
**Answer:** **Reserved Instances (RI)** – commit to a specific instance type/family/region for 1 or 3 years for a discount; less flexible (Standard RIs can't change instance family, Convertible RIs allow some changes). **Savings Plans** – commit to a certain **$/hour spend** for 1 or 3 years, and the discount automatically applies to any instance usage (across families, sizes, even Fargate/Lambda for Compute Savings Plans) that fits that spend — much more flexible than RIs since you're not locked to a specific instance type. Most teams now prefer Savings Plans over RIs for this flexibility.

---

### 71. How would you set up monitoring/alerting so your team gets notified within 2 minutes if a production API starts returning 5xx errors?
**Answer:** Set up a **CloudWatch Metric Filter** on ALB access logs (or use the ALB's built-in `HTTPCode_Target_5XX_Count` metric directly) → create a **CloudWatch Alarm** with a short evaluation period (e.g., 1-minute period, 2 consecutive breaches) and a threshold (e.g., >10 errors in 2 min) → alarm triggers an **SNS topic** → SNS delivers to Slack (via webhook/Lambda) and/or PagerDuty for on-call paging. Keep evaluation periods short for fast detection, but not so short that normal noise causes false alarms.

---

### 72. What's the difference between "hard" limits and "soft" limits (quotas) in AWS, e.g. for EC2 instances or VPCs per region?
**Answer:** **Soft limits (Service Quotas)** – default caps that AWS sets, but you can **request an increase** via Service Quotas console/API if you have a legitimate need (e.g., default 5 VPCs per region, can request more). **Hard limits** – fixed AWS-wide constraints that **cannot be increased** regardless of request (e.g., S3 bucket names must be globally unique, or certain per-account absolute maximums for specific resources). Always check Service Quotas proactively before a big scale-up to avoid hitting a soft limit mid-deployment.

---

### 73. How do you design a multi-tenant SaaS application's data isolation on AWS — what are the common approaches?
**Answer:**
- **Silo model** – separate AWS account or fully separate infrastructure per tenant — highest isolation, highest cost/complexity, used for enterprise/compliance-heavy tenants.
- **Pool model** – shared infrastructure, data isolated logically (e.g., a `tenant_id` column in shared DB tables with row-level security, or separate schemas/databases per tenant in a shared RDS instance) — cheaper, more operational overhead to guarantee isolation correctly.
- **Bridge model** – hybrid, e.g., shared compute but separate databases per tenant.
- Choice depends on compliance requirements, tenant size/count, and cost tolerance — large tenants/regulated industries often get siloed, smaller tenants share pooled infrastructure.

---

### 74. What happens during an Availability Zone failure to your Auto Scaling Group spanning multiple AZs?
**Answer:** ASG automatically detects unhealthy instances (via EC2/ELB health checks) in the failed AZ and **launches replacement instances in the healthy AZs** to maintain the desired capacity — as long as the ASG was configured across multiple AZs (a best practice) and subnets in healthy AZs are specified. If the ASG was only in a single AZ, the whole application goes down until that AZ recovers — this is exactly why Multi-AZ ASG configuration is critical for HA.

---

### 75. How do you handle secrets rotation for an RDS database password without downtime?
**Answer:** Use **AWS Secrets Manager's automatic rotation** feature with RDS — it uses a Lambda rotation function (AWS provides pre-built templates for RDS) that: creates a new password, updates it on the RDS instance, tests the new credential, then updates the secret — applications that fetch the password dynamically from Secrets Manager (rather than hardcoding it) automatically pick up the new credential on their next fetch, with no manual coordination or downtime needed.

---

### 76. What's the difference between "Provisioned IOPS SSD (io1/io2)" and "General Purpose SSD (gp3)" EBS volumes? When do you need io1/io2?
**Answer:** **gp3** – good baseline performance for most workloads, cost-effective. **io1/io2** – for **high-performance, consistent, mission-critical workloads** (e.g., large production databases like Oracle/SQL Server) needing very high IOPS (up up to 256,000 for io2 Block Express) with very low latency and consistency guarantees that gp3 can't match at that scale. io2 also offers higher durability (99.999% vs gp3's 99.8-99.9%). Use gp3 by default; upgrade to io1/io2 only when a workload genuinely demonstrates gp3's IOPS/latency isn't sufficient.

---

### 77. How would you migrate a monolithic application running on EC2 to a microservices architecture on AWS — what's your high-level approach?
**Answer:**
- Use the **Strangler Fig pattern** – gradually extract individual features/modules into separate services rather than a risky big-bang rewrite.
- Put an **API Gateway/ALB with path-based routing** in front — route specific paths to new microservices, rest still goes to the monolith, so it's a gradual, low-risk migration.
- Each new microservice gets its own deployment pipeline, own database (avoid shared DB anti-pattern), likely containerized (ECS/EKS).
- Use **SQS/SNS/EventBridge** for async communication between services instead of tight synchronous coupling.
- Migrate the highest-value/most-frequently-changing modules first, leave stable legacy parts in the monolith until there's a real reason to extract them.

---

### 78. What's the difference between "Provisioned Concurrency" and "Reserved Concurrency" in Lambda?
**Answer:** **Reserved Concurrency** – caps the *maximum* number of concurrent executions a function can have (also guarantees that much capacity is available for it, reserved from the account's pool) — used to prevent one function from consuming all account concurrency, or to limit load on a downstream resource (e.g., DB). **Provisioned Concurrency** – keeps a specified number of execution environments **pre-initialized and warm**, eliminating cold starts for that many concurrent invocations — used for latency-sensitive functions, at additional cost since you pay for the warm capacity even when idle.

---

### 79. How do you set up a CI/CD pipeline entirely within AWS native tools (no Jenkins)?
**Answer:** **CodePipeline** as the orchestrator: Source stage (**CodeCommit**/GitHub) → Build stage (**CodeBuild** – compiles, runs tests, builds Docker image, pushes to ECR) → Deploy stage (**CodeDeploy** for EC2/ECS/Lambda deployments, or direct CloudFormation/CDK deploy action). Add manual approval actions before production stage. This is a fully AWS-native alternative to a Jenkins-based pipeline, tightly integrated with IAM for permissions instead of managing separate credentials.

---

### 80. What is the difference between a "hot" partition and normal partitioning issues in DynamoDB, and how do you fix it?
**Answer:** A hot partition occurs when a **disproportionate amount of read/write traffic hits one partition key value** (e.g., using a `date` as partition key means all of "today's" writes hit one partition) — causes throttling even if overall table capacity is sufficient. Fix: use a **more distributed partition key** (e.g., combine `date` + a random suffix or `userId`), or add a **write sharding** technique (append a random number 0-N to the key and query across all shards when reading), or switch to **On-Demand mode** which handles some hot-partition scenarios better (though extreme skew can still throttle).

---

*(Questions 81–100 below — advanced/quick-fire scenario round)*

### 81. Your team accidentally committed AWS access keys to a public GitHub repo. What's your immediate response plan?
**Answer:** 1) **Immediately deactivate/delete the exposed IAM access key** in the AWS console/CLI. 2) Check **CloudTrail** for any unauthorized activity using that key since exposure. 3) Rotate any other credentials that might have been exposed alongside it. 4) Remove the secret from Git history (not just delete in latest commit — use `git filter-repo`/BFG, since old commits still expose it). 5) Going forward, enable **GitHub secret scanning**, use `git-secrets`/pre-commit hooks, and move to IAM roles instead of long-lived access keys wherever possible.

---

### 82. What's the difference between an S3 bucket policy and an IAM policy — when would you use one over the other?
**Answer:** Both can grant S3 permissions, but **bucket policy** is attached to the resource (S3 bucket) and is essential for **cross-account access** or granting access to anonymous/public users. **IAM policy** is attached to a user/role/group and is best for controlling **what your own account's identities** can do across multiple resources. Use bucket policy for "who outside my IAM can access this bucket," IAM policy for "what can my own users/roles do."

---

### 83. How does AWS Global Accelerator differ from CloudFront? When would you use Global Accelerator instead?
**Answer:** **CloudFront** – caches content at edge locations, optimized for **HTTP/HTTPS content delivery** (static/dynamic web content). **Global Accelerator** – routes traffic over AWS's private global network backbone to the nearest healthy endpoint, works for **any TCP/UDP traffic** (not just HTTP), provides **static anycast IPs**, and is better for non-HTTP use cases (gaming, VoIP, IoT) or when you need fast regional failover for non-cacheable, dynamic applications rather than content caching.

---

### 84. Your application needs sub-millisecond latency for a caching layer. What AWS service do you use, and how do you decide between Redis and Memcached on ElastiCache?
**Answer:** Use **Amazon ElastiCache**. Choose **Redis** if you need persistence, replication/Multi-AZ failover, pub/sub, complex data structures (sorted sets, lists), or transactions. Choose **Memcached** if you need pure simplicity, multi-threading (better for simple, high-throughput key-value caching across multiple cores), and don't need persistence or advanced data structures — typically Redis is chosen more often today due to its richer feature set and HA support.

---

### 85. How do you handle a scenario where your application needs to call another AWS service (e.g., S3) but must never leave the AWS network (compliance requirement — no internet path even indirectly)?
**Answer:** Use **VPC Endpoints** (Gateway for S3/DynamoDB, Interface/PrivateLink for other services) so all traffic to AWS services stays within the AWS private network, never traversing the public internet — combine with **removing/blocking the NAT Gateway route** for that specific traffic (or using Security Groups/endpoint policies to enforce only VPC-endpoint paths are used) to guarantee compliance.

---

### 86. What is the difference between "eventual consistency" and "strong consistency" in DynamoDB reads? Why does it matter?
**Answer:** **Eventually consistent read** (default) – might return slightly stale data (usually consistent within a second) but is cheaper and has higher throughput. **Strongly consistent read** – always returns the most up-to-date data, but costs more (2x read capacity) and has slightly higher latency, and isn't available for Global Tables' cross-region reads. Matters for scenarios like "user just updated their profile and immediately views it" — use strongly consistent reads there, but use eventual consistency for high-throughput read-heavy operations where slight staleness is acceptable.

---

### 87. How do you design a system to handle a sudden 10x traffic spike (e.g., a flash sale) without manual intervention?
**Answer:** **Auto Scaling** (target tracking policy reacting to CPU/request count) with a sufficiently high **max capacity** set in advance. Use **SQS as a buffer** in front of processing-heavy operations to smooth out spikes rather than letting them hit the DB/backend directly. Use **ElastiCache** to reduce DB load for repeated reads. Consider **pre-warming** ALB/CloudFront if the spike is a known, scheduled event (AWS recommends notifying them in advance for very large predictable spikes so they can pre-scale their infrastructure).

---

### 88. What's the difference between "vertical scaling" and "horizontal scaling" in AWS, and what are the tradeoffs?
**Answer:** **Vertical scaling** – increase the size/power of a single instance (bigger instance type) — simpler, but has a hard ceiling (max instance size) and usually requires downtime/reboot to resize, and doesn't improve fault tolerance (still a single point of failure). **Horizontal scaling** – add more instances behind a load balancer — better fault tolerance, near-unlimited scale, works well with Auto Scaling, but requires the application to be stateless/support distributed architecture (session handling, shared storage) to work correctly.

---

### 89. How do you troubleshoot an ALB returning 502 Bad Gateway errors intermittently?
**Answer:**
- Check **target group health checks** — are targets flapping between healthy/unhealthy?
- Check **application timeout settings** vs ALB's idle timeout (default 60s) — if the app takes longer to respond than ALB's idle timeout, ALB may close the connection prematurely, causing 502s. Increase ALB idle timeout or optimize app response time.
- Check target instance's **keep-alive settings** — mismatched keep-alive between ALB and backend (e.g., backend keep-alive timeout shorter than ALB's) can cause the ALB to send a request on a connection the backend already closed.
- Check application logs for actual crashes/errors at the exact timestamps 502s occurred.

---

### 90. What is the difference between "AWS Backup" and manually scripting snapshots via Lambda/CloudWatch Events?
**Answer:** **AWS Backup** is a fully managed, centralized backup service — define backup plans/policies once (schedule, retention, cross-region copy) and apply across multiple services (EBS, RDS, DynamoDB, EFS, etc.) with built-in compliance reporting and centralized restore. Manually scripting via Lambda gives more custom control but means building/maintaining your own scheduling, retention logic, cross-service consistency, and monitoring — AWS Backup is generally preferred unless you have very specific custom requirements it doesn't support.

---

### 91. How would you set up alerting for unusual/suspicious account activity (e.g., root account login, API calls from an unusual region)?
**Answer:** Enable **GuardDuty** (ML-based threat detection, flags things like unusual API call patterns, compromised credentials, crypto-mining activity) integrated with **EventBridge → SNS** for real-time alerts. Set up a **CloudWatch Alarm on CloudTrail** for specific sensitive events like root login (`ConsoleLogin` where `userIdentity.type = "Root"`) or `IAM policy changes`. Enable **Security Hub** to aggregate findings from GuardDuty, Config, and other security tools into a single dashboard.

---

### 92. What's the difference between "AWS Organizations SCP" and "IAM Permission Boundary"?
**Answer:** **SCP** – applies at the **AWS Organizations/account level**, sets the maximum permissions for *everyone* in that account (even the root user of that account), managed centrally by the org's management account. **Permission Boundary** – applies to a **specific IAM user/role**, sets the maximum permissions *that identity* can ever have, managed within the account itself (often used to let developers create IAM roles for their apps without risking privilege escalation, since the boundary caps what those created roles can do).

---

### 93. How do you handle graceful degradation when a downstream dependency (e.g., a third-party API) is slow or down?
**Answer:** Implement a **circuit breaker pattern** (open the circuit after repeated failures, stop calling the failing service temporarily, retry after a cooldown) to avoid cascading failures/thread exhaustion. Set appropriate **timeouts** (don't wait indefinitely). Provide a **fallback response** (cached/default data) instead of a hard failure where acceptable. Use **SQS** to queue requests to the dependency if it's an async operation, so the failure is absorbed rather than immediately impacting the user.

---

### 94. What is the difference between "Vertical" and "Horizontal" Pod-style thinking applied to RDS scaling — how do you scale a relational database that can't easily "add more instances" for writes?
**Answer:** For **read scaling**, add **Read Replicas** (horizontal). For **write scaling**, RDS traditionally can only scale **vertically** (bigger instance class) since a single primary handles all writes in most RDBMS engines — this is a key relational DB limitation. For write-heavy scale beyond what vertical scaling can handle, consider **sharding** the data across multiple RDS instances at the application level, or moving to a horizontally-scalable database (Aurora with its distributed storage architecture scales better than standard RDS, or move truly high-write-scale data to DynamoDB).

---

### 95. How do you handle blue-green deployment for a database schema change alongside application code deployment?
**Answer:** This is tricky since a "blue-green" DB swap risks data loss/sync issues. Common approach: keep the **same database** shared between blue and green environments (not two separate DBs), and ensure schema changes are **backward-compatible** with both old and new app code during the transition window (expand-contract pattern) — only fully swap application traffic, not the database itself, unless doing a full DB migration (in which case use DMS/replication to keep both DBs in sync during the transition, then cut over).

---

### 96. What's the difference between "AWS Well-Architected Framework" pillars — can you name them?
**Answer:** Six pillars: **Operational Excellence** (running/monitoring systems, improving processes), **Security**, **Reliability** (recovery from failures, meeting demand), **Performance Efficiency** (using resources efficiently), **Cost Optimization**, and **Sustainability** (minimizing environmental impact). Interviewers sometimes ask this to see if you think about architecture holistically, not just "does it work."

---

### 97. How would you design a system so that a single misconfigured deployment can never take down 100% of production traffic at once?
**Answer:** Use **canary/rolling deployments** (never 100% at once), spread infrastructure across **multiple AZs** (never deploy to all AZs simultaneously — stagger), use **feature flags** to decouple risky changes from deployment, and set up **automated rollback triggers** based on error-rate monitoring so a bad deploy is caught and reverted within minutes rather than requiring a human to notice and act.

---

### 98. What is "idempotency" in the context of API design and why does it matter for retry logic in distributed systems (e.g., SQS message processing)?
**Answer:** An idempotent operation produces the **same result no matter how many times it's executed** (e.g., "set balance to 100" is idempotent; "add 10 to balance" is not). Matters because SQS/distributed systems offer **at-least-once delivery** — a message might be processed twice due to retries/network issues. If your processing logic isn't idempotent (e.g., using an `idempotency key`/`request ID` to detect and skip already-processed messages), duplicate processing can cause real bugs like double-charging a customer.

---

### 99. How do you decide between using AWS Step Functions vs orchestrating a workflow manually with Lambda calling Lambda?
**Answer:** Use **Step Functions** when you have a **multi-step workflow** needing visual state tracking, built-in retry/error handling per step, parallel execution branches, human-approval waits, or long-running workflows (up to a year) — much easier to debug/monitor via its visual execution graph than manually chained Lambda invocations. Manual Lambda-to-Lambda chaining is simpler for very small, linear 2-step workflows but becomes hard to maintain/debug as complexity grows (no built-in state visibility, harder to handle partial failures cleanly).

---

### 100. Final scenario: You're asked to design a cost-effective, secure, and highly available 3-tier web application on AWS from scratch — summarize your architecture in a few points.
**Answer:**
- **Presentation tier:** CloudFront (CDN) + S3 (static assets) or ALB for dynamic content, across multiple AZs.
- **Application tier:** EC2 Auto Scaling Group (or ECS/EKS) behind an ALB, spread across at least 2-3 AZs, in private subnets.
- **Database tier:** RDS Multi-AZ (or Aurora) in private subnets, with Read Replicas if read-heavy.
- **Security:** Security Groups (least privilege between tiers), NACLs as secondary defense, IAM roles (no hardcoded credentials), Secrets Manager for DB credentials, WAF in front of ALB/CloudFront.
- **Networking:** NAT Gateway per AZ for private subnet outbound access, VPC Endpoints for S3/DynamoDB to avoid unnecessary NAT costs.
- **Cost optimization:** Right-sized instances, Savings Plans for steady baseline, Auto Scaling to handle variable load instead of over-provisioning.
- **Monitoring:** CloudWatch dashboards/alarms, CloudTrail for audit, GuardDuty for threat detection.
- **DR:** Automated RDS snapshots + cross-region backup copy for disaster recovery, based on the required RTO/RPO (this is where you'd mention Pilot Light/Warm Standby depending on business needs).
