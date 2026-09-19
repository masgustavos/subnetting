# AWS Network Design Reference

> **App-knowledge SoT is `/llms.txt`.** This file is for _network design_
> — what CIDR base to pick, how to size subnets, what to ask the user.
> The app's AWS-mode hierarchy (account → region → VPC → AZ → subnet),
> facade signatures, and per-method recipes live in
> `https://subnetting.dev/llms.txt`. When the two disagree, the app rules
> bind; this file gives you the _content_ to fit into them.

> **Design-gate location:** the gate that triggers this reference is
> Gate 1 in [SKILL.md](../SKILL.md). Read that first for _when_
> and _how_ to ask. This file is _what_ to ask and _what defaults to
> propose_.

## AWS hard rules — non-negotiable

These are facts about AWS VPCs, not opinions. Any plan that violates them
is wrong on its face — refuse to propose them.

| Rule                      | Value                                                                                                                                                             |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| VPC IPv4 CIDR range       | `/16` (max, 65,536 IPs) to `/28` (min, 16 IPs). Initial CIDR cannot be changed after VPC creation.                                                                |
| Subnet IPv4 CIDR range    | `/28` (min) to `/16` (max). Cannot overlap other subnets in the same VPC.                                                                                         |
| Reserved IPs per subnet   | **5**: network address, VPC router (`.1`), DNS (`.2`), future (`.3`), broadcast (`.last`). A `/28` has 11 usable IPs, not 16.                                     |
| Routable address space    | RFC 1918: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. AMS landing zones additionally accept `100.64.0.0/10` (CGNAT) for non-routable workload-adjacent uses. |
| CIDR blocks per VPC       | 5 IPv4 (1 primary + 4 secondary). Secondary CIDRs cannot overlap primary or each other.                                                                           |
| Subnets per VPC           | 200 (default).                                                                                                                                                    |
| VPCs per region           | 5 (default, raisable).                                                                                                                                            |
| AZ floor for HA workloads | 2 minimum; 3 is the production default. RDS / Aurora require ≥ 2 AZs in the DB subnet group.                                                                      |
| Non-resizable             | The VPC's primary CIDR. Plan headroom **at design time** — adding a secondary CIDR later does not give you contiguous space.                                      |

Citations: see "Sources" at the bottom.

## AWS discovery questions

These are the decisions behind a design, not a questionnaire to read
out. When Gate 1 fires, first draft the full plan with every default
below applied, then ask only what is still open, in **one turn**, as
choices: the default first and marked recommended with its one-line
reason, the realistic alternatives after it, each as a concrete value
("3 AZs", "2 AZs", "4 AZs"; `10.0.0.0/16`, `10.1.0.0/16`). The quoted
wording under each question is the intent, not a script. Skip anything
the request or the canvas already answers.

Q0 (naming) and Q8 (VPC differentiation) are presentation/structure
questions; Q1–Q7 are sizing/topology questions. When the question tool
takes fewer questions than are open, ask in this order and apply the
defaults for the rest, listing them in the draft: CIDR base (Q2 and
the CIDR base rules), AZ count (Q3), sizing profile and tier mix (Q4,
Q5), naming (Q0).

### 0. Naming convention

> "AZ names — `us-east-1a/b/c` (AWS standard) or `az-a/b/c`
> (region-agnostic)? Subnet names — `<tier>-<letter>` (e.g.,
> `private-a`) or `<tier>-<region-shortcode><letter>` (e.g.,
> `private-use1a`)? VPC names — env-prefix for workload VPCs
> (`dev-eks-vpc`, `qa-app-vpc`) or flat for function VPCs
> (`inspection-vpc`, `shared-services-vpc`)?"

**Why it matters:** Auto-names (`AZ-1`, `Subnet-1`, `VPC-1`) are
useless once the network leaves the canvas into IaC. The names are
the design contract — Terraform export uses them as identifiers,
operators read them in logs, route-table rules reference them.
Picking the convention up-front saves a full rebuild later when the
user discovers their canvas is a soup of `Subnet-1`s.

**Default:** `us-east-1a/b/c` for AZs, `<tier>-<letter>` for subnets,
env-prefix for workload VPCs, flat for function VPCs (inspection,
shared-services, ingress, egress).

**How to ask:** one option per naming variant, each with a preview
ASCII tree of the result where the question tool supports previews —
it lets the user pick by seeing the result, not parsing the rules.

**Block-creation note:** plan the names into the build recipe, not as
an afterthought. `/llms.txt` shows which creation methods take a name
up front (`addChild`, `applyTemplate`); `renameBlock` fixes the rest.

### 1. Connectivity scope

> "Is this a single VPC, multiple VPCs in the same account, or part of a
> multi-account landing zone with Transit Gateway?"

**Why it matters:** Determines whether CIDRs need to be globally
non-overlapping across accounts (TGW requirement). Single VPC can be
opportunistic; multi-account requires an IPAM-style allocation plan.

**Default:** Single VPC unless the user mentions accounts, regions,
peering, TGW, on-prem, or Direct Connect.

### 2. Adjacent CIDRs to avoid

> "Are there existing VPCs, on-prem networks, partner CIDRs, or VPN
> ranges this network must not overlap with?"

**Why it matters:** TGW and VPC peering reject overlapping CIDRs. On-prem
overlap blocks Direct Connect / VPN routing. The user's current canvas
is visible via `listBlocks()`, but adjacent ranges _outside_ the canvas
are not.

**Default:** Read `getNetwork()` + `listBlocks()` first. Surface every
CIDR already on the canvas. Ask the user only about ranges _outside_ the
canvas.

### 3. AZ count

> "How many Availability Zones — 2 (cost-optimized), 3 (HA default), or
> 4+ (rare, region-specific)?"

**Why it matters:** Drives child count under the VPC and child sizing
math. AZs are powers-of-two-friendly (2, 3 with one slot held, 4) for
the app's `split` requirement. Two-AZ VPCs save NAT-gateway cost; three
is the AWS reference architecture.

**Default:** **3 AZs** for any production-shaped request. 2 only if the
user said "lab", "dev", or "cheap".

### 4. Workload sizing profile

> "What scale — small (< 256 hosts/AZ, no EKS), medium (1–4k hosts/AZ,
> ALB-fronted apps), or large (EKS / Fargate, > 4k IPs/AZ)?"

**Why it matters:** EKS pods, Fargate tasks, and ALB scale events
consume IPs faster than EC2 hosts. ALB alone can take ≥ 8 IPs per AZ
during scale events; EKS with VPC-CNI consumes one IP per pod.
Under-sizing private subnets is the most common AWS network failure.

**Default:** Medium. If the user mentions Kubernetes, EKS, Fargate, or
"microservices", upgrade to large.

### 5. Subnet tier mix per AZ

> "Which tiers per AZ — public (ALB / NAT), private (apps), database
> (isolated), intra (TGW / VPC endpoints)?"

**Why it matters:** Database tier rarely needs the same size as private
(RDS multi-AZ wants two small subnets, not two large ones). Intra
subnets are best carved from a non-routable secondary CIDR, not the
primary (see Q7).

**Default:** Public + private + database (3-tier — the AWS web+DB
reference architecture). Add intra only if Q1 indicated TGW or the user
named VPC endpoints.

### 6. Expansion headroom

> "Should I leave roughly 50% of the VPC CIDR unallocated so you can add
> AZs, EKS pod CIDRs, or new tiers later without touching root?"

**Why it matters:** The VPC primary CIDR cannot be resized. Reserve
_now_ or live with secondary CIDRs (which break contiguity and add
route-table complexity) later. AWS Well-Architected REL02-BP03 calls
this out explicitly: leave unused CIDR space within the VPC.

**Default:** Yes — reserve 50%. Override only if the user says
"compact" / "minimum" / "tight".

### 7. TGW attachment subnets (only if Q1 = multi-account / TGW)

> "Want intra subnets for TGW attachments out of a non-routable secondary
> CIDR (e.g., `100.64.0.0/10`) so they don't consume routable space?"

**Why it matters:** TGW attachment subnets must exist (≥ `/28`) but
carry no workload traffic. Carving them from the primary CIDR wastes
routable IPs across every AZ × every VPC. The AWS Prescriptive Guidance
pattern uses a non-routable secondary CIDR with TGW blackhole routes.

**Default:** Yes — `/28` per AZ from `100.64.0.0/16` (or another agreed
non-routable secondary range).

### 8. VPC differentiation

> "Will this network contain VPCs with different purposes — workload
> (3-tier or EKS), inspection (centralized firewall), shared services
> (AD/DNS/CI), ingress, egress? If so, each typed VPC gets its own
> per-AZ shape; one Medium-style template across all of them is the
> wrong default."

**Why it matters:** A landing zone with workload + inspection +
shared-services VPCs has at least three distinct per-VPC shapes.
Inspection is transit-only (small `/28` subnets, no app/DB tiers).
Shared services has no public tier. EKS has tiny control-plane
subnets (≥ 6 IPs) plus large worker subnets, with the pod CIDR in a
secondary range. A 3-tier app VPC has app/public/database/intra at
Medium-profile sizes. These are not the same shape — pretending they
are wastes routable space (intra in primary CIDR), under-sizes EKS
workers, and oversizes the inspection VPC.

**Default:** If Q1 = single VPC, skip — the user has one VPC. If Q1
indicates multi-account / TGW, **propose typed VPC layouts per the
Typed VPC catalog** (see "Large — multi-account / TGW landing zone"
below) before any build. Get explicit confirmation per VPC type.

### Service-specific research (before proposing per-VPC shape)

If any VPC in the plan hosts a specific AWS service with documented
subnet requirements, fetch the service's `network-reqs` doc _before_
proposing the VPC's layout. Generic Medium-profile sizing does not
encode service constraints; service docs do. Common cases:

- **EKS** — https://docs.aws.amazon.com/eks/latest/userguide/network-reqs.html
  (control plane needs ≥ 6 IPs, `/28` recommended; worker subnets
  size with VPC-CNI assumptions; pod CIDR via `100.64.0.0/10`
  secondary).
- **RDS / Aurora** — https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_VPC.WorkingWithRDSInstanceinaVPC.html
  (DB subnet group requires ≥ 2 AZs in different subnets).
- **AWS Network Firewall** — https://docs.aws.amazon.com/network-firewall/latest/developerguide/architectures.html
  (centralized inspection deployment models — distributed,
  centralized, combined).
- **MSK / OpenSearch / ElastiCache** — per-AZ subnet sizing and
  cluster-placement rules vary per service. Fetch the service's
  cluster-creation doc.
- **Transit Gateway** — https://docs.aws.amazon.com/vpc/latest/tgw/tgw-best-design-practices.html
  (dedicated `/28` attachment subnets per AZ; non-routable secondary
  CIDR pattern).
- **Direct Connect / VPN** — DX gateway routing rules and the BGP
  CIDR advertisement patterns.

For services not in this list, search the official AWS docs for
`<service> network requirements` before proposing. If you cannot
find an authoritative source, **say so** in the proposal and ask the
user whether to proceed with a generic shape or wait for research.

## Sensible-defaults catalog

Three profiles. **Use these to _propose_, never to _execute_ without
confirmation.** Each row is a complete plan including reserved-for-
expansion remainder.

### Small — single VPC, 3 AZs, 2-tier

For labs, dev, single-team accounts.

| Layer    | CIDR                                      | Per AZ × 3                                     | Notes                     |
| -------- | ----------------------------------------- | ---------------------------------------------- | ------------------------- |
| VPC      | `10.0.0.0/20` (4096 IPs)                  | —                                              | Whole network.            |
| Public   | `10.0.0.0/24` × 3 (251 usable each)       | `10.0.0.0/24`, `10.0.1.0/24`, `10.0.2.0/24`    | NAT gateways, ALBs.       |
| Private  | `10.0.16.0/22` × 3 (1019 usable each)     | `10.0.16.0/22`, `10.0.20.0/22`, `10.0.24.0/22` | Apps.                     |
| Reserved | `10.0.4.0/22` + `10.0.28.0/22` (8192 IPs) | —                                              | Future tier or expansion. |

Headroom: ~50% of the VPC unallocated. Room for one secondary CIDR if
needed.

### Medium — single VPC, 3 AZs, 4-tier (recommended default)

The AWS web+DB reference architecture, plus an intra tier for TGW
attachments / VPC endpoints. Use this when the user gives no sizing
signal.

**Rendered AZ-grouped** (matches the canvas hierarchy: VPC → AZ →
subnets nested under each AZ). The size-descending sequence
(`/19 → /22 → /24 → /28`) is the order to issue `addChild` calls per
AZ — largest prefix-range first → smallest last — so each subnet
lands at a clean boundary.

```
VPC: 10.0.0.0/16 (65,536 IPs)

Per AZ (e.g., us-east-1a /18 inside the /16 VPC):
  private   /19  (8190 usable)  — 10.0.0.0/19
  public    /22  (1019 usable)  — 10.0.32.0/22
  database  /24  (251 usable)   — 10.0.36.0/24
  intra     /28  (11 usable)    — 10.0.37.0/28
  reserved  ~6 KB per AZ        — 10.0.37.16 .. 10.0.63.255

Repeat for AZs b, c at /18 boundaries 10.0.64.0 and 10.0.128.0.
Network-wide reserved: 10.0.192.0/18 (16 KB) for a 4th AZ or new tier.
```

Headroom: ~50% total — split between per-AZ slack (~6 KB inside each
AZ's `/18`) and network-wide tail (`10.0.192.0/18`).

**Drop the `intra` row** when Q1 = single VPC (no TGW, no VPC
endpoints). Per-AZ packing then becomes private `/19` + public `/22`

- database `/24` = ~9.25 KB used, ~7 KB per-AZ reserved. Total
  headroom rises to ~52%.

**Build sequence** (per AZ, after the AZ container exists):

```
subnet /19  private-a   categories: private
subnet /22  public-a    categories: public
subnet /24  database-a  categories: private, database
subnet /28  intra-a     categories: private, intra
```

Size-descending order matters: each new child is allocated from the
next free slot at the requested prefix, and starting from the largest
range guarantees each tier lands at a clean address boundary. Land the
four as one `applyTemplate` per AZ (or per VPC) where the `/llms.txt`
recipes show how; the category ids and the mandatory-group rule are in
the AWS mode section there.

### Large — multi-account / TGW landing zone

When Q1 indicated multi-account or TGW. The shape of the **org-wide
allocation** is consistent (every account gets a `/16` from a planned
`/8`); the shape of **each VPC inside an account** is _not_ one-size-
fits-all — see the typed VPC catalog below.

| Layer                    | CIDR                                         | Per AZ                                              | Notes                                                       |
| ------------------------ | -------------------------------------------- | --------------------------------------------------- | ----------------------------------------------------------- |
| Org `/8` allocation plan | `10.0.0.0/8` segmented per environment       | —                                                   | Recommend AWS IPAM.                                         |
| Per-account VPC          | `/16` from the account's reserved slot       | —                                                   | Never reuse the same `/16` across accounts.                 |
| Per-VPC shape            | See **Typed VPC catalog** below              | —                                                   | Each VPC type gets its own per-AZ shape — propose per type. |
| Intra (TGW attach)       | `100.64.0.0/16` secondary CIDR, `/28` per AZ | `100.64.0.0/28`, `100.64.0.16/28`, `100.64.0.32/28` | Non-routable; TGW blackhole route.                          |
| Reserved                 | 50% of the account `/16`                     | —                                                   | For new VPCs, secondary CIDRs, EKS pod CIDRs.               |

#### Typed VPC catalog

Different VPC types have different purposes, so they get different
per-AZ shapes. When Q8 (VPC differentiation) indicates a landing
zone with workload + inspection + shared-services VPCs, propose
each VPC's layout from this catalog. Don't collapse them into one
template — wasted address space and under-sized service subnets are
the result.

| VPC type                | Per-AZ subnets                                                             | Notes                                                                      |
| ----------------------- | -------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Workload — 3-tier       | `private /19`, `public /22`, `database /24`, `intra /28` (size-descending) | Web+DB reference architecture (== Medium profile shape).                   |
| Workload — EKS          | `workers /19`, `public /24`, `control /28`                                 | Pod CIDR in `100.64.0.0/16` secondary; control-plane subnets need ≥ 6 IPs. |
| Networking — inspection | `public /28` (NAT), `firewall /28` (NF endpoints), `tgw /28` (attachment)  | All transit. AWS Network Firewall central inspection deployment.           |
| Shared services         | `shared-services /22`, `database /24`, `intra /28`                         | AD/DNS/CI runners + interface endpoints. No public tier.                   |

**Naming follow-through.** Workload VPCs need an environment prefix
(`dev-eks-vpc`, `qa-app-vpc`, `prod-eks-vpc`) when multiple
environments share the plan; function VPCs are flat
(`inspection-vpc`, `shared-services-vpc`). Confirm in Q0.

**Build follow-through.** The build sequence in Medium (size-
descending per AZ, named and categorized as agreed) applies per VPC
type. Build one VPC of each type fully, verify it matches its catalog
row, then `copy` + `pasteAsSibling` to replicate within its
account/region — _but only after the template is verified_ (see
Gate 3 in [SKILL.md](../SKILL.md); replication amplifies template bugs
N times).

## CIDR base recommendation rules

When the user gives no CIDR base, ask before picking — never assume.
Offer the first fitting candidate below as the recommended option and
the next ones as alternatives, each with the assumption it rests on.
Recommended starting points, in order:

1. `**10.0.0.0/16` from `10.0.0.0/8`\*\* — only if the user has confirmed
   no other AWS estate uses `10.0.0.0/16` and there is no on-prem
   conflict. Cite this assumption in the proposal.
2. **Next free `/16` in their `10.0.0.0/8` plan** — if the user
   indicated existing AWS VPCs. Ask which `/16` slots are taken; pick
   the next free one (`10.1.0.0/16`, `10.2.0.0/16`, etc.).
3. `**172.16.0.0/12`\*\* — when the user has on-prem in `10.0.0.0/8` or
   has run out of `10.x.0.0/16` slots. Less common; equally valid.
4. `**192.168.0.0/16**` — labs / single-VPC experiments only. Cramped
   for production estates that may grow.
5. `**100.64.0.0/10**` — never the primary VPC CIDR. Reserved for
   non-routable secondary CIDRs (TGW attachments, EKS pod CIDR, GWLBe
   subnets) where blackhole routing keeps it from leaking.

When in doubt: ask. The cost of an extra clarification is one round-trip;
the cost of `10.0.0.0/16` colliding with a sister VPC is permanent for
this VPC's lifetime.

## Common AWS anti-patterns

Refuse to silently produce these. If the user explicitly insists, name
the trade-off and confirm again.

| Anti-pattern                                       | Why it bites                                                                                                                                 |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Production VPC at `/24` or smaller                 | 251 usable IPs across every tier and AZ — exhausts within months once ALB / EKS / Fargate enter the picture.                                 |
| Equal subnet sizes across all tiers                | Database tier almost never needs `/19`. Sizing every tier the same wastes routable space.                                                    |
| Zero expansion headroom                            | Primary VPC CIDR is non-resizable. Filling 100% on day one means every future change requires a secondary CIDR + route-table churn.          |
| `/27` or smaller for ALB-fronted private subnets   | ALB scale events can claim ≥ 8 IPs/AZ; with reserved-5 there are only 27 IPs total in a `/27`. One scale event in a busy AZ exhausts it.     |
| Reusing `10.0.0.0/16` across accounts              | Blocks future TGW peering. The moment two accounts overlap, peering between them is impossible without re-IPing one.                         |
| TGW attachment subnets in primary CIDR             | Wastes routable space across every AZ × every VPC × every TGW-attached account. Use a non-routable secondary CIDR (`100.64.0.0/10`) instead. |
| `192.168.0.0/16` for production multi-VPC estate   | 65k addresses total across the _entire_ RFC 1918 `192.168` range — too cramped for an estate that will grow past one VPC.                    |
| Single-AZ "production" VPC                         | Two AZs minimum for any HA workload. RDS / Aurora _require_ ≥ 2 AZs in the DB subnet group.                                                  |
| Picking sizes before knowing workload              | Medium-profile defaults exist for a reason. If the user says "EKS" or "Fargate", upgrade to large _before_ sizing private subnets.           |
| One subnet shape across all VPCs in a landing zone | An inspection VPC, a shared-services VPC, an EKS VPC, and a 3-tier app VPC do not share a per-AZ shape. Use the typed VPC catalog.           |
| Auto-named blocks (`AZ-1`, `Subnet-1`, `VPC-1`)    | Auto-names break Terraform export, log readability, and route-table mapping. Confirm naming in Q0 before any build.                          |

## How this reference plugs into Gate 1

Gate 1 (in [SKILL.md](../SKILL.md)) requires you to:

1. Read `getNetwork()` + `listBlocks()`.
2. Identify mode = `aws`.
3. Ask Q0–Q8 above (skip what's answered). Q0 (naming) and Q8 (VPC
   differentiation) determine _what to build_; Q1–Q7 determine _how
   to size it_.
4. For Q8 = differentiated landing zone, propose layouts per VPC
   type from the **Typed VPC catalog** in the Large profile.
   Otherwise, choose the small / medium / large profile that
   matches the Q1–Q7 answers.
5. Apply CIDR-base recommendation rules to pick the parent CIDR.
6. Surface the full plan (parent CIDR + per-VPC-type per-AZ shape +
   per-tier names + reserved-for-expansion remainder + the facade
   calls you intend to issue) as a preview.
7. **Wait for explicit go-ahead** before any `addChild` / `split` /
   `createSubnets` / `applyTemplate` / `changeCIDR` call.
8. After the build, run Gate 3 (in [SKILL.md](../SKILL.md)) against
   every block in scope before declaring complete. For `copy` +
   `pasteAsSibling` replication, run Gate 3 against the _template_
   before issuing any paste.

## Sources

- [VPC CIDR blocks](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html)
  — `/16`–`/28` range, RFC 1918, secondary CIDR rules.
- [Subnet CIDR blocks](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-sizing.html)
  — `/28`–`/16` subnet range, 5 reserved IPs (`.0`, `.1`, `.2`, `.3`,
  `.last`).
- [REL02-BP03: IP subnet allocation for expansion and availability](https://docs.aws.amazon.com/wellarchitected/latest/framework/rel_planning_network_topology_ip_subnet_allocation.html)
  — Well-Architected guidance on headroom, transient fleets, ELB IP
  consumption.
- [Preserve routable IP space in multi-account VPC designs](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/preserve-routable-ip-space-in-multi-account-vpc-designs-for-non-workload-subnets.html)
  — Non-routable secondary CIDR for TGW attachments and GWLBe subnets.
- [Network connectivity for a multi-account architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/transitioning-to-multiple-aws-accounts/network-connectivity.html)
  — TGW vs VPC peering, IPAM, centralized egress.
- [IPAM for robust network design with Control Tower](https://docs.aws.amazon.com/prescriptive-guidance/latest/robust-network-design-control-tower/ipam.html)
  — Hierarchical pool design (org → region → environment).
- [Amazon VPC quotas](https://docs.aws.amazon.com/vpc/latest/userguide/amazon-vpc-limits.html)
  — 5 VPCs/region, 5 CIDRs/VPC, 200 subnets/VPC.
- [TGW best design practices](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-best-design-practices.html)
  — Dedicated TGW attachment subnets, blackhole routing.
- [Amazon EKS network requirements](https://docs.aws.amazon.com/eks/latest/userguide/network-reqs.html)
  — Control-plane subnet sizing (≥ 6 IPs, `/28` recommended), worker
  subnet sizing for VPC-CNI, secondary pod CIDR via `100.64.0.0/10`.
- [AWS Network Firewall deployment architectures](https://docs.aws.amazon.com/network-firewall/latest/developerguide/architectures.html)
  — Centralized inspection deployment models: distributed,
  centralized, combined; per-AZ firewall-endpoint subnet sizing.
- [Working with a DB instance in a VPC](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_VPC.WorkingWithRDSInstanceinaVPC.html)
  — RDS / Aurora subnet group requirements (≥ 2 AZs in different
  subnets per DB instance).
- [Shared services VPC patterns (AWS Prescriptive Guidance)](https://docs.aws.amazon.com/prescriptive-guidance/latest/transitioning-to-multiple-aws-accounts/network-connectivity.html#shared-services-vpc)
  — AD/DNS/CI runners + interface endpoints; no public tier; per-AZ
  sizing recommendations for shared workloads.
