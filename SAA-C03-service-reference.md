# AWS SAA-C03 Service Reference

Grouped by course-section topic (mirrors the Udemy course's own section/quiz
structure where confirmed — IAM & CLI, Data & Analytics, Machine Learning,
Monitoring & Auditing, IAM Advanced, Security & Encryption — filled out with
standard AWS service-category groupings for the rest). Full extracts per
service: what it's for, the concrete numbers/limits worth memorizing, and the
decision criteria the exam actually tests. No quiz questions — pure
reference.

---

## 0. Exam Overview & Test-Taking Strategy

**Exam format**: 65 questions (50 scored + 15 unscored trial, indistinguishable
from each other), 130 minutes, multiple choice (1 correct) and multiple
response (2+ correct, exact match required), scaled score 100-1000, pass mark
**720**, **no penalty for wrong answers** — never leave a question blank.

**Domain weighting** (current SAA-C03 exam guide): Domain 1 Secure
Architectures 30%, Domain 2 Resilient Architectures 26%, Domain 3
High-Performing Architectures 24%, Domain 4 Cost-Optimized Architectures 20%.
Security carries the most weight — budget prep time accordingly.

**Logistics**: $150 USD, delivered via Pearson VUE (test center or online
proctored), results typically within 5 business days, certification valid
3 years (recertify by retaking or earning a higher-level cert). No formal
prerequisite; AWS recommends roughly 1 year of hands-on AWS experience.

**Shared Responsibility Model**: AWS is responsible for security **OF** the
cloud — physical infrastructure, hardware, the global network, and the host
OS/virtualization layer underneath managed services. The customer is
responsible for security **IN** the cloud — their data, IAM configuration,
network/firewall configuration, client-side encryption, and (for IaaS-style
services) guest OS patching. The split shifts by service type: on EC2 the
customer patches the guest OS; on a managed service like RDS or Lambda, AWS
patches the OS/runtime and the customer still owns access control and data.

**Global Infrastructure**: a **Region** is an isolated geographic area
containing multiple Availability Zones. An **Availability Zone** is one or
more discrete data centers with redundant power/networking, isolated as a
failure domain but low-latency connected to other AZs in the same Region. An
**Edge Location** (CloudFront/Route 53 endpoints) is far more numerous than
Regions, used for content delivery/DNS. Local Zones/Wavelength/Outposts are
covered in Section 20.

**AWS Support Plans**: **Basic** (free — docs, forums, core Trusted Advisor
checks only), **Developer** (business-hours email support, general
guidance), **Business** (24/7 phone/chat/email, <1hr response for
production-system-down, full Trusted Advisor checks), **Enterprise** (24/7,
dedicated Technical Account Manager, <15min response for
business-critical-system-down). The exam tests matching a described
urgency/criticality to the right support tier.

**Test-taking strategy**: budget ~2 minutes per question, flag-and-return
anything uncertain rather than stalling. Read the **last sentence** of a
scenario first — that's the actual question — then read the scenario knowing
what to look for. Eliminate the 1-2 obviously-wrong options first, then
decide between the remaining plausible ones by finding the requirement
adjective (fastest / most secure / least cost / least operational overhead)
— it usually decides between the last two options. Always guess rather than
skip; there's no penalty, and an educated guess beats a guaranteed zero.

---

## 1. IAM & AWS CLI

**IAM core model**: Users / Groups / Roles / Policies (JSON documents:
Effect, Action, Resource, Condition). Groups can't be nested and can't be
a principal in a trust policy — only users, roles, and AWS services can
assume/be granted access.

**Policy types**:
- **Identity-based**: attached to a user/group/role.
- **Resource-based**: attached directly to a resource (S3 bucket policy,
  SQS queue policy, KMS key policy) — grants access without the caller
  needing to assume anything.
- **Permissions boundary**: caps the *maximum* permissions an identity can
  ever have, regardless of what identity-based policies grant it —
  commonly used to let a team self-service create roles without being able
  to escalate beyond the boundary.
- **SCP** (Organizations-level): see Section 18 — same "maximum, not
  grant" logic, but scoped to accounts/OUs, not individual identities.
- **Session policies**: passed when assuming a role via STS, further
  restrict (never expand) what the assumed role's own policy allows.

**Evaluation logic**: explicit **Deny always wins**, over any Allow, at
every layer combined. For same-account access, an identity-based policy
allowing an action is generally sufficient. For cross-account access, or
for certain resource types (S3, KMS, SQS/SNS), **both** the identity
policy on the caller's side AND the resource policy on the resource's
side must allow the action.

**Instance profile**: the container object that actually lets an EC2
instance assume a role — when you "attach a role to EC2" in the console,
it's silently creating/using an instance profile behind the scenes.

**IMDS (Instance Metadata Service) v1 vs v2**: v1 is a simple GET request
directly to `169.254.169.254`; **v2 requires a session token via a PUT
request first** (protects against SSRF-style attacks that trick a proxy
into fetching credentials from IMDS). Exam signal: "harden against
credential theft via a misconfigured reverse proxy" → enforce IMDSv2 and
optionally increase the hop limit restriction.

**MFA**: virtual (authenticator app TOTP), FIDO security key (hardware),
or hardware TOTP token — required practice at minimum for the root user
and any highly privileged identity.

**IAM Credentials Report**: account-wide CSV of every user's credential
status (password age, MFA enabled, access key age/last used) —
account-wide audit answer.

**IAM Access Analyzer**: finds resources (S3, IAM roles, KMS keys, Lambda,
SQS, Secrets Manager) shared with an entity outside your account/
organization; can also generate a least-privilege policy draft from
observed CloudTrail activity.

**CLI/SDK credential resolution order** (roughly): explicit
CLI flags → environment variables → shared credentials/config file
(`~/.aws/credentials`, supports named profiles) → container credentials →
instance profile (EC2 IMDS). Prefer roles over long-lived access keys
wherever the compute context supports it — this ordering matters for
"why is my CLI using the wrong credentials" troubleshooting questions.

---

## 2. EC2 Fundamentals

**Instance family letters**: **C**ompute-optimized, **R**AM/memory-
optimized, **M**general purpose, **I/D**storage-optimized (high local
NVMe IOPS), **G/P**GPU/accelerated computing, **T**burstable (CPU
credits — cheap baseline, burst above it using accumulated credits,
throttled to baseline once credits are exhausted unless "Unlimited" mode
is enabled, which allows sustained high CPU at extra cost).

**Nitro System**: the underlying hypervisor/hardware platform for modern
instance types — offloads networking/storage virtualization to dedicated
hardware, is why newer instance types get near-bare-metal performance and
support features like EBS encryption-by-default with no performance hit.

**User data**: script executed **once**, at first boot, runs as root —
for bootstrapping only, not ongoing configuration management (that's
Systems Manager / config management tooling's job).

**Launch Templates vs. Launch Configurations**: Launch Templates are the
current/recommended mechanism (versioned, support newer features like
mixed instance types and Spot), Launch Configurations are the legacy ASG
mechanism (immutable once created, no versioning) — the exam favors
Launch Templates whenever both are offered as options.

**Security Groups**: stateful (return traffic automatically allowed
regardless of outbound rules), evaluated at the instance/ENI level, allow
rules only, implicit deny-all inbound by default, all outbound allowed by
default. Can reference *other security groups* as a source (not just
CIDR), useful for tiered architectures (e.g., "allow inbound 3306 only
from the app-tier SG").

**Elastic IP**: static public IPv4; free while attached to a running
instance, **billed while allocated but idle/unattached** or attached to a
stopped instance — this asymmetric billing detail is a common cost-
optimization exam trap ("we have unused EIPs sitting around costing
money").

**Tenancy**: default (shared underlying hardware) vs. **Dedicated
Instances** (your instances get dedicated hardware, but hardware can
still be shared with other instances of the same account over time) vs.
**Dedicated Hosts** (a specific physical server allocated to you, visible/
addressable — needed when licensing is tied to physical socket/core count,
e.g. some BYOL Windows/SQL Server scenarios).

**Placement groups**: **Cluster** (single AZ, instances physically close,
lowest network latency/highest throughput — HPC, tightly-coupled batch);
**Spread** (each instance on distinct underlying hardware, max
7 instances per AZ, use for a small number of critical instances needing
max isolation from each other); **Partition** (instances grouped into
partitions on distinct hardware, up to 7 partitions per AZ, used by
distributed systems like HDFS/Cassandra/Kafka that already understand
rack-awareness).

**Hibernate**: preserves in-memory RAM state to the root EBS volume on
stop, enabling much faster resume than a cold boot — for stateful
workloads with expensive warm-up/initialization.

---

## 3. EC2 Instance Storage (EBS, EFS, AMI)

**EBS**: network-attached block storage, bound to a single AZ (must
match the instance's AZ) but persists independently of the instance's own
lifecycle (stop/start/terminate, depending on the `DeleteOnTermination`
flag per volume).
- **gp3**: current-generation general purpose — IOPS (baseline 3,000) and
  throughput (baseline 125 MiB/s) are provisioned *independently of
  volume size*, unlike gp2 where IOPS scaled with (and was capped by)
  size — gp3 is usually both the cheaper AND higher-performing answer
  over gp2 once IOPS/throughput needs go up.
- **io2 / io2 Block Express**: highest durability (99.999%) and IOPS
  (up to 256,000 on io2 Block Express), sub-millisecond latency — for
  mission-critical, IOPS-intensive databases.
- **st1**: throughput-optimized HDD, large sequential workloads (big
  data, log processing, data warehousing) — cannot be used as a boot
  volume.
- **sc1**: cold HDD, cheapest per-GB, infrequent large sequential access.
- **EBS Multi-Attach**: io1/io2 only, attach a single volume to multiple
  instances in the same AZ simultaneously — requires a cluster-aware
  filesystem, not a general-purpose solution.
- **Encryption**: can enable "encryption by default" account/region-wide;
  an *existing unencrypted* volume can't be encrypted in place — you
  snapshot it, copy the snapshot with encryption enabled, then create a
  new volume from that snapshot.
- **Snapshots**: incremental (only changed blocks since the last
  snapshot), stored in S3 behind the scenes (not directly visible/
  billed as an S3 bucket), can copy cross-region and cross-account, can
  create an AMI directly from a snapshot.

**EFS**: managed NFSv4 filesystem, multi-AZ by design, mountable
concurrently from many instances/containers, Linux-only, pay-per-use
(no pre-provisioning required in Bursting Throughput mode) — the shared,
concurrently-writable filesystem answer whenever multiple compute
instances need the *same live* data.
- **Performance modes**: General Purpose (default, lowest latency) vs.
  Max I/O (higher aggregate throughput/IOPS, higher per-operation
  latency — many concurrent clients, e.g. big data workloads).
- **Throughput modes**: Bursting (scales with storage size, credit-based
  like T-instances), Provisioned (fixed throughput regardless of storage
  size), Elastic (auto-scales throughput up/down with workload, no
  planning needed — good default for spiky/unpredictable access).
- **EFS storage classes + lifecycle management**: Standard, Standard-IA,
  One Zone, One Zone-IA — lifecycle policies auto-transition files based
  on last-access time, same cost-optimization idea as S3 lifecycle rules.

**AMI**: a region-scoped bootable image (root volume snapshot + launch
permissions + block device mapping); must be explicitly copied to another
region to launch instances there.

**Instance store**: physically attached, ephemeral, highest possible IOPS
(no network hop) but **data is lost on stop/terminate/hardware failure**
— only for data that's fine to lose (cache, temp/scratch, or data
already durably replicated elsewhere).

---

## 4. Elastic Load Balancing & Auto Scaling

**ALB**: layer 7 (HTTP/HTTPS/WebSocket), supports path-based and
host-based routing to different target groups, supports multiple TLS
certs on one listener via SNI, native fit for microservices/container
target groups. **Sticky sessions** (via a load-balancer-generated cookie)
bind a client to one target — use when the app isn't stateless and you
haven't yet externalized session state.

**NLB**: layer 4 (TCP/UDP/TLS), extreme low latency and very high
throughput (millions of req/s), supports static IP and Elastic IP per AZ
— pick over ALB specifically when you need a fixed IP address or
raw non-HTTP protocol performance.

**GWLB**: transparent in-line insertion of third-party virtual
appliances (firewalls, IDS/IPS) in front of a fleet, using GENEVE
encapsulation — traffic flows through the appliance without the client/
server needing to know it's there.

**Cross-zone load balancing**: evens out request distribution across all
targets regardless of how unevenly they're spread across AZs — ALB has it
on by default (no extra charge); NLB has it off by default (enabling it
can incur cross-AZ data transfer charges).

**Health checks**: ELB-level (protocol/path/interval/threshold config) —
an ASG can use EC2 status checks alone or ELB health checks (recommended
when behind a load balancer, since it also factors in application-level
health, not just instance-level).

**Auto Scaling Groups**: min/max/desired capacity, spans multiple AZs for
HA. Scaling policies: **Target Tracking** (simplest — "keep average CPU
at 50%"), **Step Scaling** (different scaling amounts per CloudWatch
alarm breach severity), **Scheduled** (known traffic patterns, e.g.
scale up before a daily peak), **Predictive Scaling** (ML-based, forecasts
and pre-scales ahead of recurring patterns).

**Warm pools**: pool of pre-initialized (stopped or running) instances
ready to attach — cuts scale-out latency for slow-booting applications
without paying full running cost for a stopped warm pool.

**Lifecycle hooks**: pause an instance in the `Pending` or `Terminating`
state to run custom actions (e.g., drain in-flight connections, run
final data upload) before it fully enters/exits service.

**Termination policies**: control which instance ASG picks to terminate
first during scale-in (default balances across AZs, then picks oldest
launch config/template, then closest to next billing hour) — can be
customized when specific instances should be protected/preferred for
removal.

---

## 5. Databases: RDS, Aurora, ElastiCache

**RDS Multi-AZ**: synchronous standby replica in a different AZ,
automatic failover (DNS endpoint flips to the standby, app doesn't need
reconfiguration), standby is **not readable** — pure HA/DR, not a
read-scaling mechanism.

**RDS Read Replicas**: asynchronous replication, can be same-region or
cross-region, **are readable** — for offloading read traffic, not
automatic failover (manual promotion to a standalone instance is
possible but breaks replication).

**RDS backups**: automated backups (daily snapshot + transaction logs,
enable point-in-time recovery within the retention window, default
retention up to 35 days) vs. manual snapshots (retained until you delete
them, independent of the instance's lifecycle — survive even instance
deletion if you choose to keep them).

**RDS encryption**: enabled at creation time only — you **cannot encrypt
an existing unencrypted RDS instance in place**; the pattern is snapshot →
copy the snapshot with encryption enabled → restore a new instance from
the encrypted copy.

**Aurora**: MySQL/PostgreSQL-compatible, storage layer auto-replicates
6-way across 3 AZs, self-healing storage, up to 15 read replicas with
typically single-digit-millisecond replica lag (much lower than standard
RDS async replication). **Aurora Serverless v2** scales compute capacity
to actual load continuously and can scale to near-zero for intermittent
workloads — pick over provisioned Aurora for spiky/unpredictable or
dev-test database workloads where cost tracking usage matters.
**Aurora Global Database**: primary region + up to 5 secondary regions,
typically <1 second replication lag, secondary regions support fast
promotion for cross-region DR.

**RDS Proxy**: managed connection pooling in front of RDS/Aurora —
protects the database from connection exhaustion, particularly the
classic "Lambda opens a new DB connection per concurrent invocation"
problem; also enables faster failover for the application since the proxy
absorbs the reconnect.

**ElastiCache**: in-memory cache layered in front of a database to
reduce read latency and DB load.
- **Redis**: supports persistence (snapshotting/AOF), replication with
  automatic failover (cluster mode), pub/sub, richer data structures —
  pick when the cache itself needs durability/HA or you need pub/sub.
- **Memcached**: simpler, multi-threaded (can use multiple cores per
  node), pure cache with no persistence/replication — pick when you just
  need raw caching horsepower and losing the cache entirely is fine.
- **Eviction policies** (e.g. LRU variants) control what happens when the
  cache is full — relevant when a question describes stale/evicted-data
  symptoms.

**DynamoDB DAX**: a *separate* microsecond-latency in-memory cache
specific to DynamoDB (not the same product as ElastiCache) — don't
conflate the two on the exam.

---

## 6. Route 53 (DNS & Traffic Routing)

**Hosted zones**: public (internet-resolvable) vs. **private hosted
zones** (resolvable only within one or more associated VPCs — internal
service discovery / internal DNS without exposing records publicly).

**Routing policies**:
- **Simple**: one record (or multiple values returned together, client
  picks), no health checks.
- **Weighted**: split traffic by assigned % weight — canary releases,
  A/B testing, gradual migration between environments.
- **Latency-based**: routes to whichever region gives the requester the
  lowest measured latency, not necessarily the geographically closest.
- **Failover**: active-passive pair, backed by health checks — automatic
  DR cutover at the DNS layer.
- **Geolocation**: routes based on the *requester's* geographic location
  — compliance/content-licensing restrictions (e.g. "EU users must hit
  the EU endpoint").
- **Geoproximity**: routes by geographic distance with an adjustable
  "bias" to shift traffic toward/away from a region — requires Route 53
  Traffic Flow.
- **Multi-value answer**: returns several healthy record values for the
  client to pick from — lightweight client-side load distribution, not a
  substitute for a real load balancer.

**Alias records**: Route-53-specific record type that can point at AWS
resources (ALB, CloudFront distribution, S3 static website endpoint,
another Route 53 record) **at the zone apex** (bare domain, e.g.
`example.com`) — a plain CNAME cannot be used at the zone apex per DNS
spec, so Alias records are the standard workaround, and they're free (no
extra query charge) plus automatically track the target's IP changes.

**Health checks**: can monitor an endpoint directly, monitor another
health check (calculated health checks combining several), or monitor a
CloudWatch alarm — drive Failover routing and can also remove unhealthy
records from Multi-value/Weighted responses.

**Route 53 Resolver**: hybrid DNS resolution between on-prem and a VPC
(inbound and outbound endpoints) — for environments where on-prem DNS
needs to resolve AWS private records and vice versa.

**Domain registration vs. hosted zone**: registering a domain through
Route 53 is a separate concern from hosting its DNS records — you can
register a domain elsewhere and still host its zone in Route 53 (and
vice versa).

---

## 7. VPC Fundamentals

**Subnets** are AZ-scoped; **VPCs** are regional. A subnet is "public"
purely because its route table sends `0.0.0.0/0` to an Internet Gateway —
there's no other flag that makes it public.

**CIDR sizing**: VPC CIDR can range from /16 (65,536 addresses) to /28
(16 addresses); AWS reserves the **first 4 and last 1 IP address in every
subnet** (network address, VPC router, DNS, future use, broadcast) — so a
/28 subnet effectively gives you only 11 usable addresses, a common
"why did I run out of IPs" exam trap.

**NAT Gateway**: managed, AZ-scoped (deploy one per AZ for HA — a single
NAT Gateway is a single point of failure and a cross-AZ cost driver),
scales automatically, billed per-hour **plus per-GB processed** — that
per-GB charge is why routing S3/DynamoDB traffic through a Gateway VPC
Endpoint instead (free) is a recurring cost-optimization answer.

**NAT instance**: self-managed EC2 acting as a NAT — cheaper at low
volume, but you own patching, HA (no built-in failover), and it must have
source/destination check disabled.

**Egress-only Internet Gateway**: IPv6 equivalent of a NAT Gateway
(outbound-only internet access for IPv6, since IPv6 addresses are
public by design and NAT isn't used the same way).

**Security Groups vs. NACLs** (the classic pairing question): SGs are
stateful, instance-level, allow-only, evaluate ALL rules before deciding.
NACLs are stateless (return traffic must be explicitly allowed both
directions), subnet-level, support explicit **Deny**, evaluated in
ascending rule-number order with first match winning — NACLs are the only
layer that can outright block a specific IP/CIDR.

**VPC Peering**: direct 1:1 connection between two VPCs, **not
transitive** (A↔B and B↔C does not give A↔C — you'd need a direct A↔C
peering or a Transit Gateway instead), CIDR ranges must not overlap.

**Transit Gateway**: regional hub-and-spoke for connecting many
VPCs/on-prem networks through one managed gateway — replaces an
unmanageable full-mesh of peering connections once you're past a
handful of VPCs; supports transitive routing (unlike peering) and can
peer across regions. Also supports **multicast** routing between
attached VPCs — the only one of the VPC-connectivity options (peering,
Direct Connect, VPN) that does, so a scenario needing one-to-many
multicast distribution across VPCs points straight at Transit Gateway.

**VPC Endpoints**:
- **Gateway endpoints**: S3 and DynamoDB only, **free**, implemented via
  route table entries (not an ENI).
- **Interface endpoints (PrivateLink)**: most other AWS services (and
  third-party/custom services you expose via PrivateLink), backed by an
  ENI with a private IP in your subnet, small hourly + per-GB cost, also
  reachable over Direct Connect/VPN — both endpoint types keep traffic off
  the public internet, and for S3/DynamoDB specifically a Gateway
  endpoint also avoids routing that traffic through (and paying for) a
  NAT Gateway.

**VPC Flow Logs**: captures IP traffic metadata (not payload) at ENI,
subnet, or VPC scope — feeds GuardDuty, troubleshooting connectivity/
security issues, and compliance evidence.

**Reachability Analyzer**: static-analysis tool that traces whether a
path between two resources *would* be reachable given current
routes/SGs/NACLs — troubleshooting without generating live traffic.

**Site-to-Site VPN**: IPsec tunnel over the public internet, fast to
provision, variable/unpredictable latency and bandwidth. **Direct
Connect**: dedicated physical link into an AWS Direct Connect location,
consistent low latency and higher bandwidth, **not encrypted by default**
(layer a VPN over DX — "Direct Connect plus VPN" — when both performance
and encryption are required) — pick DX over VPN for large, steady,
performance-sensitive on-prem↔AWS data flows; pick VPN for quick setup or
lower/variable volume.

---

## 8. S3 Fundamentals

Buckets are globally-uniquely-named, region-scoped, effectively unlimited
storage, 11 nines (99.999999999%) durability across every storage class
— **durability is not availability**; availability SLA varies by class
(e.g., Standard-IA has a lower availability SLA than Standard, One Zone-
IA lower still since it's single-AZ).

**Storage classes** (organize your thinking by *access pattern*, not
raw price):
- **Standard**: frequent access, millisecond first-byte latency.
- **Intelligent-Tiering**: unknown or changing access pattern — auto-
  moves objects between access tiers based on observed usage, no
  retrieval fee, small monthly monitoring fee per object — the default
  "safe" answer whenever the question doesn't specify a clear, predictable
  access pattern.
- **Standard-IA / One Zone-IA**: infrequent access but still millisecond
  retrieval, per-GB retrieval fee; One Zone-IA is single-AZ (cheaper,
  acceptable only for data that's easily reproducible/non-critical if
  that AZ is lost).
- **Glacier Instant Retrieval**: archive tier with millisecond access —
  quarterly-ish access patterns that still need instant reads.
- **Glacier Flexible Retrieval**: minutes (expedited) to ~12 hours
  (bulk) retrieval time, cheaper than Instant Retrieval.
- **Glacier Deep Archive**: cheapest class, 12+ hour retrieval, built
  for 7-10 year regulatory/compliance retention.
- **Lifecycle policies**: automate transitions between classes and/or
  expiration purely by object age — the standard cost-optimization
  mechanism whenever a question describes a known aging-access pattern.

**Versioning**: must be enabled before either CRR or SRR can be
configured; also the mechanism behind protecting against accidental
overwrite/delete (a delete just adds a delete marker, prior versions
remain recoverable).

**Replication**: **Cross-Region Replication (CRR)** for DR/latency/
compliance across regions; **Same-Region Replication (SRR)** for
compliance/log-aggregation within a region — both are asynchronous, and
replicating SSE-KMS-encrypted objects requires explicitly specifying the
destination-region KMS key (the source key can't operate cross-region).

**Encryption**: **SSE-S3** (AWS owns and rotates the key, simplest,
default for new buckets), **SSE-KMS** (AWS-managed or customer-managed
CMK — gives an audit trail via CloudTrail and fine-grained key-policy
access control), **SSE-C** (caller supplies the key with every request,
AWS never stores it — the caller is fully responsible for key
management).

**Access control layering**: bucket policies (resource-based, can grant
cross-account access directly), IAM policies (identity-based), Access
Control Lists (legacy, object/bucket-level, generally discouraged now in
favor of policies + **Object Ownership** settings that can disable ACLs
entirely), and **S3 Block Public Access** (account/bucket-level override
that can forcibly prevent public access regardless of what a policy/ACL
says — the standard "prevent accidental public bucket" answer).

**S3 Object Lock / Glacier Vault Lock**: WORM (write-once-read-many)
enforcement — Compliance mode (nobody, including root, can override
before retention expires) vs. Governance mode (can be overridden with
special permissions) — for regulatory data-immutability requirements.

**Other S3 features worth knowing by name**: **Access Points** (named
network endpoints with their own policy, simplify access management for
shared buckets with many different access patterns/teams), **Event
Notifications** (trigger Lambda/SQS/SNS on object create/delete — the
usual glue for event-driven pipelines fed by S3), **S3 Select** (retrieve
just the needed subset of an object's data via SQL-like expressions
instead of downloading the whole object), **Static website hosting**
(serve HTML directly from a bucket), **Batch Operations** (run an action
across billions of objects at once), **Storage Lens** (org-wide usage/
activity visibility and cost-optimization recommendations), **MFA
Delete** (requires MFA to permanently delete a version or change
versioning state — extra protection for critical buckets), **Requester
Pays** (the requester, not the bucket owner, pays data transfer costs —
for widely-shared public datasets).

**Performance features**: **Transfer Acceleration** (routes uploads
through CloudFront edge locations for faster long-distance transfer),
**Multipart upload** (required above 5GB, recommended above ~100MB, for
resilience/performance on large object uploads — a failed part can be
retried without re-uploading the whole object).

---

## 9. Content Delivery: CloudFront & Global Accelerator

**CloudFront**: CDN — edge-caches content at AWS edge locations close to
users, reducing both latency for the user and load on the origin
(S3, ALB, or any custom HTTP origin, including outside AWS) — the default
answer for cacheable HTTP(S) content.
- **Cache behaviors**: per-path-pattern rules controlling caching TTL,
  allowed methods, which origin to use — lets one distribution serve
  multiple origins/behaviors.
- **Price classes**: restrict which edge locations are used (e.g., only
  North America/Europe) to trade a bit of global performance for lower
  cost when the audience is regionally concentrated.
- **Origin Access Control (OAC)**: locks an S3 origin down so it's only
  reachable through CloudFront, never directly — the standard "private S3
  content served only via CDN" pattern (OAC is the current recommendation,
  superseding the older Origin Access Identity/OAI).
- **Signed URLs / Signed Cookies**: restrict access to specific
  authorized users/sessions — signed URLs for single-file access, signed
  cookies when a user needs access to multiple restricted files at once.
- **Lambda@Edge vs. CloudFront Functions**: Lambda@Edge runs full Lambda
  functions (Node/Python) at a subset of edge locations for heavier
  logic (can call other AWS services); CloudFront Functions run
  lightweight JavaScript at *every* edge location with much lower
  latency/cost — for simple tasks (header manipulation, redirects, auth
  token validation), prefer CloudFront Functions; for anything needing
  more compute or AWS SDK access, use Lambda@Edge.
- **Field-Level Encryption**: encrypts specific sensitive fields
  (e.g., credit card numbers) all the way through to the origin, so even
  CloudFront/the origin's web server never sees the plaintext.

**Global Accelerator**: routes client traffic over AWS's private global
network backbone using two static anycast IP addresses — for
**non-cacheable/dynamic TCP or UDP traffic** (gaming servers, VoIP,
IoT, non-HTTP protocols) where CloudFront's HTTP-caching model doesn't
apply; also gives fast regional failover since it's an anycast-based
health-checked routing layer, not DNS-based (avoids DNS TTL/caching
propagation delay during failover).

---

## 10. Storage Extras: Gateway, Snow Family, DataSync, FSx, Backup, Transfer Family

**Storage Gateway modes**:
- **File Gateway**: NFS/SMB access, backed by S3, with local caching of
  frequently-accessed files.
- **Volume Gateway — Cached mode**: primary data lives in S3, frequently
  used data cached locally.
- **Volume Gateway — Stored mode**: primary data lives locally, async
  backed up to S3 as EBS snapshots.
- **Tape Gateway**: virtual tape library interface for existing backup
  software that expects physical tapes — backs onto S3/Glacier.

**DataSync**: online, automated, scheduled transfer of large datasets
between on-prem storage and AWS (or between AWS storage services
directly), encrypted in transit, handles retries/validation — the answer
for *repeated/ongoing* large transfers over an existing network link
(distinct from Snow Family, which is for one-off/offline transfer when
network transfer would take too long).

**Snow Family** (physical data transfer): **Snowcone** (small, rugged,
edge locations with limited space/power), **Snowball Edge** (tens of TB
per device, has onboard compute — can run Lambda functions or a mini EC2
instance at the edge before/during transfer, e.g. for pre-processing),
**Snowmobile** (exabyte-scale, an actual shipping container on a truck)
— the exam tests this as a bandwidth-vs-time tradeoff calculation:
if transferring your dataset over your available network link would take
longer than shipping a physical device, use Snow Family instead.

**FSx for Windows File Server**: SMB protocol, Active Directory
integration, for Windows-based workloads needing native Windows
filesystem features.

**FSx for Lustre**: high-performance, scale-out filesystem for HPC/ML
training workloads, can link directly to an S3 bucket as its backing
data repository (transparently presenting S3 objects as files, and can
write results back to S3).

**AWS Backup**: centralized backup service across many AWS resource
types at once (EBS, RDS, DynamoDB, EFS, FSx, Storage Gateway volumes,
EC2) with a single policy-driven backup plan (schedule, retention,
lifecycle to cold storage, cross-region/cross-account copy) — the answer
whenever a question wants "one consistent backup policy across multiple
different AWS services" instead of configuring each service's native
backup separately.

**AWS Transfer Family**: fully managed SFTP/FTPS/FTP endpoints backed by
S3 or EFS — for migrating or exposing legacy file-transfer workflows
(that specifically require the SFTP/FTP protocol) without running your
own FTP servers.

---

## 11. Application Integration / Decoupling

**SQS Standard**: at-least-once delivery, best-effort ordering (messages
can arrive out of order or, rarely, duplicated), near-unlimited
throughput. **Visibility timeout** (default 30s) hides a message from
other consumers while one consumer processes it — if processing takes
longer than the timeout, another consumer can pick up the same message
(a classic "duplicate processing" root cause to know for troubleshooting
questions). **Long polling** (`WaitTimeSeconds` up to 20s) reduces empty
responses and API cost compared to short polling. Max message size
256KB — for larger payloads, use the **SQS Extended Client Library**,
which stores the payload in S3 and passes a reference through the queue.

**SQS FIFO**: exactly-once processing, strict ordering (within a message
group), throughput capped (300 msg/s base, up to 3,000/s with batching)
— use only when true ordering/deduplication is required, since it's
slower and more constrained than Standard.

**SNS**: pub/sub fan-out to multiple subscriber types at once (SQS,
Lambda, HTTP/S, email, SMS, mobile push) — SNS itself does **not**
durably retain a message if a subscriber is unavailable; pair it with an
SQS queue per subscriber (the fan-out pattern) whenever durability/
guaranteed-delivery matters.

**EventBridge** (formerly CloudWatch Events): event bus with
content-based routing/filtering rules across AWS service events, SaaS
partner events, and custom application events — richer routing logic
than SNS, plus a built-in scheduler (replacing the need for cron-on-EC2
for many use cases) and a **Schema Registry** that can infer/version
event schemas for producer/consumer contract management.

**Step Functions**: orchestrates multi-step workflows as a visible state
machine with built-in retry/catch/error-handling per step — use when a
process needs explicit branching/state visibility, not just a
fire-and-forget queue. **Standard workflows** (up to 1 year duration,
exactly-once execution, full execution history) vs. **Express workflows**
(up to 5 minutes, higher throughput, at-least-once execution, cheaper —
for high-volume, short-duration event-processing workloads).

**SES**: outbound (and inbound) email sending — for transactional/
marketing email specifically, not a general message queue.

**Dead-letter queues**: capture messages that repeatedly fail
processing after a configured number of receives — usable with SQS
consumers and with Lambda/SNS failure destinations, preventing a
poison-pill message from blocking a queue indefinitely.

**Amazon MQ**: managed ActiveMQ/RabbitMQ — for *migrating* an existing
application that already speaks a standard broker protocol (JMS, AMQP,
MQTT, STOMP). Unlike SQS/SNS (AWS-native APIs), MQ is the answer whenever
the requirement explicitly needs broker-protocol compatibility rather
than a cloud-native rebuild.

---

## 12. Containers on AWS

**ECS**: AWS-native container orchestrator.
- **EC2 launch type**: you manage the underlying EC2 instances (cluster
  capacity) — more control, cheaper at sustained high scale, more
  operational overhead (patching, capacity planning).
- **Fargate launch type**: serverless — no instances to manage, pay per
  task's vCPU/memory — less operational overhead, typically pricier per
  unit of compute at large sustained scale.
- **Capacity Providers**: let a service span both EC2 and Fargate, or
  use Spot capacity for EC2-backed tasks, with automatic capacity
  management.
- **Task role vs. task execution role**: the **task role** is what the
  application code inside the container assumes to call other AWS
  services (S3, DynamoDB, etc.); the **task execution role** is what ECS
  itself needs (pulling the image from ECR, writing logs to CloudWatch) —
  same "role attaches to the thing doing the calling" logic as Lambda's
  execution role, just split into two roles here because two different
  actors (ECS agent vs. your app) need different permissions.
- **Service Auto Scaling**: scales the number of running tasks based on
  CloudWatch metrics (target tracking, step scaling) — separate concern
  from cluster-capacity scaling (EC2 Auto Scaling / Capacity Providers).

**EKS**: managed Kubernetes control plane — pick specifically when the
requirement is Kubernetes/multi-cloud portability, not just "I need
containers." Node groups can be **Managed** (AWS handles node
provisioning/lifecycle), **Self-managed** (you own the EC2 Auto Scaling
Group), or **Fargate profiles** (serverless pods, no nodes to manage at
all, matched by namespace/labels).

**ECR**: private container image registry, integrates with ECS/EKS via
IAM permissions on the task execution role; supports image scanning for
vulnerabilities.

**App Runner**: fully managed service specifically for deploying web
apps/APIs straight from source code or a container image, with
automatic build/deploy/scaling and an integrated load balancer — the
"I just want my container running as a web service with essentially zero
infra configuration" answer, sitting above Fargate in abstraction level.

---

## 13. Serverless & Application Deployment

**Lambda**: event-driven compute, max 15-minute execution timeout, pay
per invocation + duration (billed per millisecond of execution), scales
out automatically per concurrent invocation. Always has exactly **one
execution role** attached (the role direction principle: attaches to the
compute doing the calling, never to the resource being called).
- **Cold starts**: the delay incurred when Lambda has to initialize a new
  execution environment — **Provisioned Concurrency** keeps a specified
  number of environments pre-initialized and warm to eliminate this for
  latency-sensitive workloads; **Reserved Concurrency** instead caps
  (and guarantees) the maximum concurrent executions for a function,
  which is about capacity control, not warm-start latency — don't
  confuse the two.
- **VPC access**: attaching a Lambda to a VPC (to reach an RDS instance,
  for example) requires ENI attachment; modern Lambda uses a shared
  "Hyperplane" ENI model that made this dramatically faster than it used
  to be, but it's still a relevant "why is my Lambda slow to start when
  it's in a VPC" exam-adjacent fact.
- **Layers**: share common code/dependencies across multiple functions
  without bundling them into every deployment package.
- **Destinations**: route the async invocation's success/failure result
  to another service (SQS, SNS, EventBridge, another Lambda) without
  writing custom retry-handling code.

**API Gateway**: front door for APIs. **REST API** (full feature set —
request/response transformation, usage plans, API keys, caching);
**HTTP API** (lighter-weight subset, lower latency, cheaper — pick when
you don't need REST API's extra features); **WebSocket API** (persistent
bidirectional connections). Built-in throttling, request validation, and
native Lambda/HTTP backend integration.

**X-Ray**: distributed tracing across microservices/Lambda/API Gateway —
visualizes request flow and latency across a service graph, the answer
whenever a question is about diagnosing *where* latency/errors are
introduced across a chain of services (as distinct from CloudWatch,
which is per-service metrics/logs, not a cross-service trace).

**SAM (Serverless Application Model)**: a CloudFormation extension +
CLI tooling purpose-built for Lambda-based serverless apps (simplified
syntax for functions/APIs/tables, local testing/invoke support).

**Elastic Beanstalk**: PaaS — you supply application code, it provisions
and manages the underlying resources (EC2, ELB, ASG, RDS if configured)
for you — less low-level control than raw EC2, far less setup effort
than building the stack yourself by hand; you can still access/tune the
underlying resources if needed (unlike Lambda, where there's no server to
touch at all) — the "low operational overhead, but I still want a
conventional server-based app" answer.

**CloudFormation**: infrastructure as code — declarative templates,
grouped into **stacks**. **Change sets** preview what a template update
would actually change before applying it. **Drift detection** flags
resources that were modified outside CloudFormation. **Nested stacks**
compose reusable template modules. **StackSets** deploy the same stack
across many accounts/regions from one operation — the answer for
"deploy this baseline consistently across our whole AWS Organization."

**CI/CD pipeline services**: **CodeCommit** (managed Git source, largely
legacy now that GitHub/GitLab integration is common), **CodeBuild**
(compile/test, produces build artifacts), **CodeDeploy** (automates
deployment to EC2/on-prem/Lambda/ECS, supports blue/green and
in-place strategies), **CodePipeline** (orchestrates the end-to-end
release workflow across the above stages).

**Systems Manager (SSM)**: **Session Manager** (browser/CLI shell access
to instances without opening inbound SSH or running a bastion host —
a common "reduce attack surface" exam answer), **Run Command** (ad-hoc
remote command execution at scale across a fleet), **Patch Manager**
(automated OS patch compliance/scheduling), **Parameter Store** (see
Security section).

---

## 14. Databases Extras: DynamoDB

Managed key-value/NoSQL, single-digit-millisecond latency regardless of
scale, schemaless beyond the key structure.

**Key design**: every table has a **partition key** (hash key) alone, or
a **partition key + sort key** (composite key). Item distribution across
partitions is driven by the partition key's hash — a poorly chosen
partition key (e.g. a low-cardinality value most writes share) creates a
**hot partition**, throttling throughput even if overall table capacity
looks sufficient — a very common exam scenario ("why are we getting
throttled even though we provisioned plenty of capacity").

**Capacity modes**: **On-demand** (pay per request, no capacity planning,
best for unpredictable/spiky traffic) vs. **Provisioned** (set read/write
capacity units ahead of time, cheaper at steady sustained volume,
combine with **Auto Scaling** to adjust within a configured range as load
changes).

**Indexes**: **Local Secondary Index (LSI)** — same partition key as the
base table, different sort key, must be created at table-creation time,
shares the table's provisioned throughput. **Global Secondary Index
(GSI)** — different partition key (and optionally sort key) entirely, can
be added/removed any time, has its own separate provisioned throughput —
GSIs are the flexible, generally-preferred option unless you specifically
need LSI's strong-consistency read option.

**Consistency**: reads are eventually consistent by default (cheaper,
lower latency); strongly consistent reads are available (base table and
LSI, not GSI) at roughly double the read cost.

**DynamoDB Transactions**: ACID transactions across multiple items/tables
in a single all-or-nothing operation — for workloads needing true
multi-item atomicity, at extra cost/latency versus normal reads/writes.

**Global Tables**: multi-region, multi-active replication with
last-writer-wins automatic conflict resolution — for globally
distributed, low-latency, highly-available applications, not just a DR
backup copy.

**DAX**: a separate, purpose-built microsecond-latency in-memory read
cache sitting in front of DynamoDB specifically (distinct product from
ElastiCache).

**TTL**: automatic item expiration/deletion by timestamp attribute,
doesn't consume write capacity.

**Streams**: an ordered, near-real-time change-data-capture feed of
item-level table modifications — commonly triggers a Lambda function for
downstream processing (replication, aggregation, notifications).

**Limits worth remembering**: max item size 400KB.

---

## 15. Data & Analytics (course Quiz 19 section)

**Kinesis Data Streams**: raw real-time ingestion pipe you manage
yourself — data is organized into shards, each shard supports 1MB/s or
1,000 records/s **in**, and 2MB/s **out**; default retention 24 hours,
extendable up to 365 days — the base building block for custom real-time
processing applications you write consumers for.

**Amazon Managed Service for Apache Flink** (formerly Kinesis Data
Analytics): runs continuous SQL or Flink applications **directly against
a live stream** — this is the "near-real-time query" answer whenever
speed is the explicit requirement, because it queries data in motion
rather than waiting for it to be persisted somewhere first.

**Data Firehose**: fully managed **delivery** pipeline — no shard
management, unlike Kinesis Data Streams. Supported destinations: S3,
Redshift, OpenSearch Service, Splunk, Datadog, New Relic, Dynatrace, Sumo
Logic, LogicMonitor, MongoDB, or a generic HTTP endpoint. It buffers
incoming records on a size/time interval before writing — this is the
durability/persistence leg of a pipeline, never the fast-query leg
(there's an inherent minimum latency from its buffering behavior that a
stream-query approach like Flink doesn't have).

**MSK (Managed Streaming for Kafka)**: managed Apache Kafka — pick this
specifically when the requirement explicitly calls for Kafka or
Kafka-ecosystem tooling/compatibility, not as a general streaming
default.

**Glue**: serverless ETL (Spark-based jobs, auto-generated or custom
scripts) + the **Glue Data Catalog** (a shared metadata catalog —
table/schema definitions — consumed by Athena, Redshift Spectrum, and
EMR alike, so cataloging data once in Glue makes it queryable from all
three).

**Lake Formation**: centralized permissions/governance layer on top of an
S3-based data lake, built on the Glue Data Catalog — fine-grained
(column/row-level) access control across the services that read the lake.

**Athena**: serverless SQL directly against data sitting in S3 (via a
Glue Catalog table definition) — ad-hoc querying of data already at
rest, pay-per-query-scanned, not a real-time/streaming query engine.

**Redshift**: OLAP data warehouse, columnar storage, optimized for
complex analytical queries over very large datasets — not intended for
transactional (OLTP) workloads. **Redshift Spectrum** queries data
directly in S3 without first loading it into Redshift's own storage.

**OpenSearch** (formerly Elasticsearch): search engine + log analytics/
visualization (via OpenSearch Dashboards) — the go-to for full-text
search and operational log analysis at scale.

**EMR**: managed Hadoop/Spark/Presto/Hive clusters — for big-batch
processing at scale when Glue's serverless job model doesn't fit
(long-running clusters, specific framework/tooling requirements, more
manual tuning control).

**QuickSight**: BI dashboards/visualization layer that sits on top of
the above data sources.

**Kinesis Video Streams**: purpose-built ingestion for streaming video/
audio/time-encoded data, distinct from Kinesis Data Streams' generic
record-based model.

---

## 16. Machine Learning (course Quiz 20 section — "what is X for" recall)

- **Rekognition**: image/video analysis — object, scene, and face
  detection/comparison.
- **Transcribe**: speech-to-text. **Polly**: text-to-speech.
- **Translate**: language translation.
- **Lex**: conversational chatbot engine (powers Amazon Connect's voice/
  chat interaction flows).
- **Comprehend**: NLP — sentiment analysis, entity recognition, key-phrase
  extraction from unstructured text. **Comprehend Medical**: same
  capability, tuned for clinical/medical text.
- **SageMaker**: build/train/tune/deploy *your own custom* ML models — the
  "bring your own model" service, distinct from the rest of this list,
  which are pre-built, ready-to-call APIs.
- **Kendra**: managed enterprise search with natural-language query
  understanding across internal documents.
- **Personalize**: managed recommendation-engine service.
- **Textract**: extracts text, forms, and table data from scanned
  documents/images.
- **Fraud Detector**: pre-built, fraud-specific ML models that score a
  transaction or signup for fraud risk — you set the business
  rules/thresholds against the returned score, no ML expertise or custom
  model training needed.

Exam depth here is shallow — mostly single-fact "what does this service
do" recall, not multi-service architecture scenarios.

---

## 17. Monitoring & Auditing (course Quiz 21 section)

**CloudWatch Metrics**: numeric time-series data — many AWS services
publish metrics automatically (e.g. EC2 CPU) at 5-minute intervals by
default, or 1-minute with "detailed monitoring" enabled; custom
application metrics can be published via the API/agent.

**CloudWatch Logs**: centralized log aggregation. **Logs Insights**
provides an interactive query language over log data; **Live Tail**
streams matching log lines in real time.

**CloudWatch Agent**: required to collect **OS-level metrics** (memory,
disk usage — not just CPU/network, which come for free from the
hypervisor level) and to ship custom application log files off an
instance — these are *not* collected automatically without it, a common
"why don't I see memory metrics" exam trap.

**CloudWatch Alarms**: trigger actions (SNS notification, Auto Scaling
policy, EC2 action like reboot/stop) when a metric crosses a defined
threshold over a defined number of evaluation periods. **Composite
alarms** combine multiple alarms with AND/OR logic to reduce noise from
individually-flapping metrics.

**CloudWatch Synthetics**: scripted "canaries" that proactively simulate
user traffic against endpoints/APIs on a schedule — catches outages/
regressions even with no real user traffic hitting the endpoint at that
moment.

**EventBridge** (see Section 11): the event-bus/scheduler successor to
"CloudWatch Events" — same underlying service, current branding.

**CloudTrail**: records **API activity** — who called what action, on
what resource, when, from where — governance/compliance/security
forensics. Enabled account-wide by default for the last 90 days of
management events (a "trail" must be explicitly created for indefinite
retention/S3 delivery); can be **organization-wide** to cover every
account under an AWS Organization from one trail. Supports **log file
integrity validation** (cryptographically verify logs haven't been
tampered with) and can route events into **EventBridge** for automated
reaction to specific API calls.

**AWS Config**: tracks **resource configuration state** over time and
evaluates it against rules (managed or custom) for compliance —
answers "what did this resource's configuration look like at time X" and
"is it currently compliant," a different axis entirely from CloudTrail's
"who did what."

**CloudTrail vs. CloudWatch vs. Config** (the classic three-way exam
disambiguation): **CloudTrail** = API call audit trail (who/what/when);
**CloudWatch** = operational metrics, logs, and alarms (how is it
performing right now); **Config** = configuration state history and
compliance/drift detection (what does it look like, and should it).

---

## 18. IAM Advanced (course Quiz 22 section)

**Organizations**: multi-account management, consolidated billing,
volume discounts pooled across accounts. Organized into **OUs**
(Organizational Units) which can nest, letting policies apply broadly by
business unit/environment.

**SCPs**: set the *maximum* permissions for every identity in an
account/OU — purely restrictive, never grants anything by itself; an SCP
allowing an action still requires the account's own IAM policies to
separately allow it too (SCPs and identity policies are both a gate the
request must pass, not either/or). SCPs are **inherited down the OU
tree** — a Deny at a parent OU can't be overridden by a more permissive
SCP lower in the tree. **SCPs never affect the management (root) account
of the Organization** — a common exam trap is assuming an org-wide "deny
X" SCP locks down every account, when the management account is always
exempt by design and can still perform the denied action.

**Tag Policies**: enforce consistent resource tagging (allowed
keys/values) across accounts in an Organization — governance/cost-
allocation hygiene at scale.

**Resource-based policies vs. IAM roles** (two different mechanisms for
the same goal — cross-account access): a resource-based policy (S3
bucket policy, SQS queue policy, KMS key policy) grants access **directly
on the resource** to a specified principal, with no `AssumeRole` step
needed by the caller; an IAM role requires the caller to explicitly
`sts:AssumeRole` first and receive temporary credentials. Which one a
question wants often comes down to whether the caller needs to switch
identity (role) or just needs the resource itself to vouch for them
(resource policy).

**IAM Policy Evaluation Logic** (deeper than the simple "Deny wins"
rule): for same-account access, the identity-based policy on the caller
is generally sufficient by itself. For cross-account access — or for
resource types where AWS specifically requires it (S3, KMS, SQS, SNS,
Lambda resource policies, and a few others) — **both** the caller's
identity policy AND the resource's own policy must independently allow
the action; an explicit Deny anywhere in the whole chain (identity
policy, resource policy, SCP, permissions boundary, session policy)
overrides every Allow.

**AWS RAM (Resource Access Manager)**: shares specific resources
(a Transit Gateway, a subnet, a License Manager configuration, a Route
53 Resolver rule, and others) directly across accounts/Organizational
Units **without** duplicating the resource or requiring full
cross-account IAM role setup for each consumer — the answer whenever a
scenario wants "let other accounts use this one shared subnet/Transit
Gateway attachment" cleanly.

**Cognito**: distinct from IAM Identity Center — Cognito is for **your
application's end users**, not AWS workforce access.
- **User Pools**: a user directory with built-in sign-up/sign-in, MFA,
  and federation with external IdPs (Google, Facebook, SAML/OIDC
  corporate IdPs) — issues its own JWTs after authentication.
- **Identity Pools**: exchange an authenticated identity (from a User
  Pool, or an external IdP directly, or even unauthenticated "guest"
  access) for **temporary AWS credentials**, letting a client app call
  AWS services (e.g., upload directly to S3) without a backend
  intermediary.
- User Pools and Identity Pools are commonly used **together**: User Pool
  authenticates the user, Identity Pool then hands out scoped temporary
  AWS credentials based on that authentication.

**IAM Identity Center** (successor to AWS SSO): centralized **workforce**
access across multiple AWS accounts plus SAML/OIDC business
applications — permission sets mapped to accounts/groups, one login for
humans managing many accounts (contrast with Cognito, which is for
end-user-facing applications, not internal staff account access).

**AWS Directory Services**: **AWS Managed Microsoft AD** (full,
managed AD you can extend/trust with on-prem AD), **AD Connector**
(a proxy redirecting auth requests to an existing on-prem AD, no user
data stored in AWS), **Simple AD** (a standalone, Samba-based, basic
AD-compatible directory, no on-prem trust support) — pick based on
whether you need real trust with on-prem AD (Managed AD/AD Connector) or
just basic AD-compatible directory features standalone (Simple AD).

**Control Tower**: sets up and governs a multi-account **landing
zone** on top of Organizations — automated account provisioning (Account
Factory), pre-built guardrails (built on SCPs and Config rules), and a
dashboard for org-wide compliance — the answer for "set up a
well-governed multi-account environment from scratch with best practices
baked in," rather than assembling Organizations/SCPs/Config manually.
**Guardrails come in two kinds**: **preventive** guardrails are
implemented as SCPs and actually **block** an API call before it happens
(hard stop, e.g. "disallow disabling CloudTrail"); **detective**
guardrails are implemented as Config rules and only **detect and flag**
non-compliant resources after the fact, they don't stop the action. A
scenario asking to "prevent" something needs a preventive guardrail; one
asking to "identify/alert on" something already done needs a detective
one.

---

## 19. Security & Encryption (course Quiz 23 section)

**Encryption 101**: symmetric (single shared key, faster, KMS's default
and typical exam answer for encrypting data at rest) vs. asymmetric
(public/private key pair — used for things like digital signatures or
when the encrypting party must not have decrypt access).

**KMS key types**:
- **AWS-owned key**: fully invisible/unmanaged by you (e.g. default
  SSE-S3 encryption uses this) — zero configuration, zero visibility,
  zero cost.
- **AWS-managed key**: visible in your account (e.g. `aws/s3`), AWS
  controls its rotation automatically, you can view but not edit its
  key policy.
- **Customer-managed key (CMK)**: full control — you set the key policy
  (who can use/manage it), can enable/disable it, schedule it for
  deletion, control rotation, and every single use is logged in
  CloudTrail — pick a CMK whenever the requirement is an audit trail of
  key usage or fine-grained access control over the encryption key
  itself, distinct from access control over the *data*. **Key deletion
  is never immediate** — you schedule it with a mandatory **waiting
  period of 7 to 30 days** (your choice within that range), during which
  the key can still be canceled/recovered; this exists specifically to
  prevent an accidental or malicious deletion from instantly and
  irreversibly destroying every piece of data encrypted under that key.
- **Envelope encryption**: KMS encrypts a locally-generated **data key**
  rather than the payload directly; that data key then encrypts the
  actual data on the client side — this is why KMS can handle
  arbitrarily large objects efficiently despite the KMS API itself only
  accepting small payloads directly (a 4KB request-size limit on the
  `Encrypt` API is exactly why this indirection exists).
- **Multi-Region Keys**: the same key material (a primary key + one or
  more explicitly-created replica keys) exists in multiple regions
  without needing to decrypt-and-re-encrypt data during a regional
  failover — the answer whenever cross-region DR needs KMS-encrypted
  data to remain readable in the failover region without an extra
  re-encryption pass; note replica keys are independently manageable,
  not automatically synced beyond the key material itself.

**S3 replication with SSE-KMS encryption**: replicating cross-region
requires explicitly specifying the **destination region's own KMS key**
in the replication configuration — a source-region key has no authority
to operate in the destination region.

**Encrypted AMI cross-account sharing**: sharing an AMI encrypted under a
customer-managed CMK across accounts requires **explicitly granting the
target account permission in the CMK's key policy** — sharing the AMI's
launch permissions alone is not sufficient, since the target account
also needs to be able to use the key to decrypt the underlying snapshot.

**SSM Parameter Store**: free at the standard tier, hierarchical
key-value config/secret storage, supports encryption via KMS for
`SecureString` parameters, **no built-in automatic rotation** — good for
general application configuration and less-sensitive secrets/large
config hierarchies.

**Secrets Manager**: built-in **automatic rotation** via a Lambda
function on a schedule, charges per secret stored — the default answer
whenever the requirement explicitly calls for rotating credentials
(database passwords, API keys) rather than just storing static config.

**ACM (Certificate Manager)**: free public TLS certificates for
AWS-integrated resources (ALB/NLB, CloudFront, API Gateway), automatic
renewal — certificates **cannot be exported** for use outside these
native integrations (e.g., you can't pull the private key out to install
on a self-managed EC2 web server directly with a public ACM cert).
**ACM Private CA** is a separate product — it issues *private*
certificates for internal-only TLS (service-to-service traffic, IoT
device identities) — pick it whenever the requirement is an internal
PKI/CA hierarchy, not a public-facing certificate.

**CloudHSM**: dedicated, single-tenant hardware security module — pick
over KMS specifically when a workload requires full administrative
control over the HSM, FIPS 140-2 **Level 3** validation (vs. KMS's
underlying Level 2), or needs to manage keys in a way that's entirely
outside AWS's own KMS service boundary (e.g., certain compliance regimes
or specific cryptographic operations KMS doesn't expose directly).

**WAF**: layer-7 rule engine (SQL injection, XSS, rate-based rules,
geo-blocking, managed rule groups) attached to CloudFront, ALB,
API Gateway, or AppSync.

**Shield Standard**: automatic, free, basic layer 3/4 DDoS protection
included on every AWS account by default.

**Shield Advanced**: paid tier — adds DDoS cost protection (credits
against scaling charges incurred during an attack), 24/7 access to the
AWS DDoS Response Team (DRT), and richer attack diagnostics/detection —
for high-value or high-risk-profile workloads where Standard's baseline
protection isn't enough. **Nuance with Global Accelerator**: Shield
Advanced does protect a Global Accelerator resource, but it protects it
at layer 3/4 (network-level DDoS) — it does **not** automatically attach
WAF's layer-7 rules to the accelerator itself, because WAF is not a
supported resource type for Global Accelerator. Layer-7 filtering still
has to happen at the actual HTTP-terminating resource behind the
accelerator (an ALB or CloudFront distribution), where WAF *can* be
attached — don't assume "Shield Advanced + Global Accelerator" alone
gives you WAF-style request filtering.

**Firewall Manager**: centrally manages and enforces WAF rules, Shield
Advanced protections, and Security Group policies across every account
in an AWS Organization from one place — the "consistent security
policy at scale across accounts" answer, distinct from WAF/Shield
themselves which operate per-resource. Requires **AWS Organizations**
to be in use and **AWS Config** enabled (recording resources) in every
account/region it manages — it's an org-wide governance layer, not a
standalone single-account tool.

**GuardDuty**: continuous, agentless threat detection built from VPC
Flow Logs, CloudTrail management/data events, and DNS query logs —
flags things like known-malicious IP communication, unusual API call
patterns, or crypto-mining behavior, without deploying anything to your
resources.

**Inspector**: automated vulnerability scanning for EC2 instances,
container images in ECR, and Lambda functions — continuously assesses
against known CVEs and network reachability.

**Macie**: uses ML to discover and classify sensitive data (PII,
credentials, financial data) stored in S3 — the answer whenever a
question is about finding/flagging sensitive data at rest in S3
specifically, not general threat detection.

---

## 20. Migration & Cost Management (cross-cutting, typically late-course)

**The 7 Rs of Migration** (the standard vocabulary for "how should this
workload move to AWS" scenario questions, roughly least-to-most
transformation effort): **Retire** (decommission — no longer needed);
**Retain** (leave it where it is, revisit later); **Rehost**
("lift-and-shift" — move as-is, no code changes, fastest path, typically
via Application Migration Service); **Relocate** (move a whole VMware
environment or similar into AWS with no changes, at the hypervisor
level); **Repurchase** ("drop and shop" — replace with a different
product, e.g. a SaaS alternative); **Replatform** ("lift-tinker-and-
shift" — a few targeted optimizations during the move, e.g. moving a
self-managed database onto RDS without a full rewrite); **Refactor/
Re-architect** (rebuild the application to be cloud-native, most effort,
biggest long-term payoff — e.g. breaking a monolith into microservices/
serverless). A scenario emphasizing speed/minimal risk points toward
Rehost; one emphasizing long-term cost/agility payoff points toward
Refactor.

**DMS (Database Migration Service)**: homogeneous (same engine) or
heterogeneous migration (different engine, paired with the **Schema
Conversion Tool** to translate schema/code) — supports continuous
replication/change-data-capture for a near-zero-downtime cutover
(replicate continuously, then do a brief final cutover rather than one
big offline migration window).

**Application Migration Service (MGN)**: lift-and-shift of entire
servers into AWS via continuous block-level replication to a staging
area, with a low-downtime cutover when ready — for whole-server
migration, as opposed to DMS's database-specific scope.

**Application Discovery Service**: inventories on-prem servers and their
dependencies/utilization to inform migration planning — feeds into
Migration Hub for tracking migration progress across many servers/apps.

**AWS Elastic Disaster Recovery (DRS)**: continuous, low-cost block-level
replication of servers (on-prem or other clouds) into a low-cost staging
area in AWS, with fast full-environment launch on actual failover — the
current DR-specific answer (successor to CloudEndure) when the
requirement is explicitly disaster recovery (standby-and-launch-on-
failover) rather than a one-time migration.

**AWS Batch**: fully managed batch computing — schedules and scales
compute (via ECS/Fargate/EC2/Spot under the hood) to run large numbers
of batch jobs, handling job queues/dependencies/retries — for compute-
heavy batch workloads, distinct from Lambda (15-minute cap) or Step
Functions (orchestration, not raw batch compute scheduling).

**License Manager**: tracks and enforces software license usage/limits
(e.g., per-core licensing) across an organization's AWS (and on-prem)
resources — relevant when a scenario mentions BYOL (bring-your-own-
license) constraints.

**Outposts / Local Zones / Wavelength** (hybrid & edge infrastructure —
know the distinction at a glance): **Outposts** = AWS-managed hardware
physically installed in your own data center for workloads needing
extremely low latency to on-prem systems or local data-residency
requirements. **Local Zones** = AWS infrastructure deployed in additional
metro areas (closer to end users than the parent region) for latency-
sensitive applications without owning hardware yourself. **Wavelength** =
AWS compute/storage embedded within telecom providers' 5G networks, for
ultra-low-latency mobile/edge applications.

**Compute cost layering**: On-Demand (unpredictable usage) → Savings
Plans / Reserved Instances (steady baseline, 1 or 3-year commitment,
up to ~72% discount) → Spot (interruptible, up to ~90% discount,
reclaimed with a 2-minute warning — fault-tolerant/stateless
batch/CI workloads only, never anything needing guaranteed
availability). **Compute Savings Plans** (most flexible — any instance
family/region, and covers Lambda/Fargate too) vs. **EC2 Instance
Savings Plans** (locked to one instance family + region, bigger discount
in exchange) vs. **Reserved Instances** (least flexible, but resellable
on the RI Marketplace if plans change).

**Cost visibility/governance tools**: **Cost Explorer** (visualize/
analyze historical spend, forecast future cost, RI/Savings Plan
purchase recommendations based on usage history), **AWS Budgets**
(proactive threshold alerts before/when spend or usage crosses a
limit), **Trusted Advisor** (automated checks across cost, performance,
security, fault tolerance, and service limits — flags idle/
underutilized resources like low-CPU EC2 instances or unattached
Elastic IPs), **Compute Optimizer** (ML-based rightsizing
recommendations for EC2/EBS/Lambda/ASG derived from actual utilization
history, deeper than Trusted Advisor's simpler checks), **Cost and
Usage Report (CUR)** (the most granular billing data available, meant
to be queried via Athena/fed into custom BI rather than read directly).

**Well-Architected Framework pillars**: Operational Excellence,
Security, Reliability, Performance Efficiency, Cost Optimization,
Sustainability — scenario questions sometimes frame a design tradeoff
explicitly around which pillar a given choice is optimizing for, so
recognizing which pillar a described concern belongs to can help
eliminate answer options that optimize for the wrong one.

**Well-Architected Tool & Service Catalog**: the **Well-Architected
Tool** is a free service that reviews an actual workload against the 6
pillars via a guided questionnaire and flags risks — distinct from the
Framework itself (the set of principles) versus this Tool (the thing
that evaluates you against them). **Service Catalog** lets admins
publish a curated, pre-approved list of AWS products/templates for
self-service — the governance answer whenever a requirement is "let
teams self-serve, but only from an approved catalog."

---

## Cross-cutting exam patterns (apply to every question, regardless of section)

- **"Near-real-time" / "fastest"** → in-motion processing (Flink,
  Lambda-on-stream) beats at-rest querying (Athena, a DB SELECT) whenever
  speed is the explicit ask.
- **"Access AWS resource X from compute Y"** → IAM role attached to Y,
  never a role "on" X, never static credentials.
- **"No data loss" / "decouple"** → SQS/SNS/EventBridge/Kinesis; SNS alone
  isn't durable without an SQS subscriber behind it.
- **"Least operational overhead"** → serverless/managed beats anything you
  provision/patch yourself.
- **"MOST cost-effective" + unpredictable traffic** → on-demand pricing
  models beat Reserved/Savings-Plans answers, which assume steady usage.
- **Hot partition / uneven throttling despite adequate provisioned
  capacity** → look at partition/shard key cardinality (DynamoDB,
  Kinesis), not raw capacity numbers.
- **"Encrypt with an audit trail / control over the key"** → KMS
  customer-managed key, not AWS-managed or SSE-S3.
- **Cross-account access** → check whether the scenario wants the caller
  to switch identity (IAM role + AssumeRole) or wants the resource itself
  to vouch for the caller (resource-based policy) — and remember both
  sides (identity + resource policy) generally need to allow it for
  cross-account S3/KMS/SQS/SNS/Lambda scenarios.
- The "obviously reasonable, very common architecture" option is often a
  distractor if it doesn't hit the *specific* adjective the question asks
  for (fastest / most secure / least cost / least overhead) — check that
  word first, it usually eliminates half the options immediately.
