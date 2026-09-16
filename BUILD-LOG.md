# BUILD-LOG — Zero Trust Remote Access Gateway on AWS

**A complete technical write-up of what was built, how it was built, and — more importantly — *why every single piece exists at all*.**

Region: `us-east-1` · Built entirely through the AWS Console · Domain: `sinosomtechnology.qzz.io`

---

## How to read this document

This is not a checklist. If you want the checklist, it's the plan document — this is the *explanation*.

The order here is deliberate and it is **not** the order things were clicked in. It goes:

1. **The problem** — what's actually wrong with normal remote access, in concrete terms
2. **The journey of a single request** — the whole system explained as one story, end to end, before any AWS service is named in detail
3. **The building blocks** — every component explained from zero, assuming no prior knowledge. What a load balancer *is*. What a subnet *is*. Why a certificate exists.
4. **The build** — each piece, in the order it was created, with the reasoning behind every non-obvious choice, including the dropdown selections that looked arbitrary at the time
5. **The bug** — one genuine, subtle failure that took real debugging, documented in full because it's the most instructive part of the entire project
6. **Proof** — the evidence, what each screenshot actually proves, and why that specific thing needed proving
7. **Cost, teardown, and what this would look like in production**

If you build something by following steps, you end up able to rebuild it and unable to explain it. This document exists to close that gap. Read Part 1 and Part 2 in full before anything else — those two parts are the ones that make everything after them make sense instead of feeling arbitrary.

---

## Repository layout

```
.
├── README.md                          ← short intro, start there
├── BUILD-LOG.md                       ← this file
├── diagrams/
│   └── architecture.svg
├── VPC/
│   └── VPC available.png
├── Route table/
│   └── route table.png
├── EC2/
│   ├── apps running and available.png
│   ├── security group APP-SG details page.png
│   ├── security group APP-SG inbound rules.png
│   ├── security group APP-SG outbound rules.png
│   ├── security group APP-ALB details page.png
│   ├── security group APP-ALB inboud rule.png
│   └── security group APP-ALB outbound rule.png
├── load balancer/
│   ├── active load balancer.png
│   ├── General ALB details page.png
│   └── target groups Healthy.png
├── certification verification/
│   ├── cert issued.png
│   ├── digitalplat record added .png
│   └── two endpoints subdomains.png
├── IAM/
│   ├── users.png
│   └── groups.png
├── Verified Access/
│   ├── 1 Verified Access trust providers tab.png
│   ├── 2 Verified Access group.png
│   ├── 3 Verified Access endpoints.png
│   ├── 4 General endpoint details page.png
│   ├── 5 General endpoint policy page.png
│   ├── 6 admin endpoint details page.png
│   └── 7 admin endpoint policy page.png
├── general app url/
│   ├── check to general url.png
│   ├── redirected to login.png
│   ├── login as said.png
│   ├── forced to MFA.png
│   ├── mfa success.png
│   ├── am redirected to application.png
│   └── try admin application for said got unauthorized access.png
├── admin app url/
│   ├── login as admin.png
│   ├── register MFA asked.png
│   ├── MFA success.png
│   ├── admin login to admin application.png
│   └── he can also access staff application.png
└── cloudwatch/
    ├── cloudwatch is watching.png
    ├── access allow.png
    └── access deny.png
```

---

# PART 1 — The problem this project exists to solve

## 1.1 How remote access normally works

A company has internal systems: an admin dashboard, an internal wiki, a database console, a finance tool. None of them should be on the public internet. So they're put on a private network — a network with no direct door to the outside world.

Staff still need to reach them from home. So the company installs a **VPN** (Virtual Private Network). The employee opens a VPN client, enters a username and password, and the client builds an encrypted tunnel from their laptop into the company network. From the operating system's point of view, that laptop is now *inside* the network. It gets an internal IP address. It can talk to internal machines as though it were physically plugged into a desk in the office.

That's the model. It's been the default for roughly thirty years. And it has two structural problems that are not configuration mistakes — they're baked into the design.

## 1.2 Problem one: authentication happens once, at the edge

The VPN checks who you are **at the moment you connect**. After that, it doesn't check again. You authenticated at 9:00 AM; at 3:00 PM your traffic is still trusted, based on a decision made six hours ago.

Nothing re-verifies that:

- the person at the keyboard is still the person who logged in
- the laptop hasn't been compromised since
- the account hasn't been disabled by HR in the meantime
- this particular request — to this particular system — is something that person should be allowed to do at all

The authentication event and the access events are separated by hours. Everything after the first check runs on trust inherited from that check.

## 1.3 Problem two: the unit of access is *the network*, not *the application*

This is the bigger one.

When the VPN lets you in, it doesn't let you into "the finance tool." It lets you into **the network segment the finance tool happens to live on**. And on that segment there is also the admin dashboard, the database, the backup server, the printer with default credentials from 2016, and the forgotten test box nobody has patched since it was built.

Access control then has to be re-implemented *inside* the network, by firewall rules between subnets, by each application's own login page, by host-based rules — dozens of separate mechanisms maintained by different people at different times. In practice these are incomplete. There is almost always a path from "authenticated VPN user" to "something they were never meant to touch."

This is what the security industry calls **lateral movement**, and it is the single most valuable thing an attacker gets from a stolen VPN credential. The credential doesn't give them one system. It gives them a *position* — a foothold inside the perimeter, from which everything else becomes reachable. The initial compromise is rarely the crown jewels. It's a low-value account that happens to sit on the same network as the crown jewels.

The shorthand for this model is **castle and moat**: a hard outer wall, and a soft, flat, trusting interior. Get over the wall once and the interior offers almost no resistance.

## 1.4 Why this keeps mattering

Credentials leak constantly. Phishing works. Password reuse across a breached site and a corporate VPN works. MFA fatigue attacks work. The realistic assumption for any serious organisation is not "our credentials will never be stolen" — it's **"some credential will eventually be stolen, and the architecture has to survive that."**

A castle-and-moat network does not survive it. One valid credential collapses the entire security model, because the entire security model was one check at one wall.

## 1.5 What Zero Trust actually means

"Zero Trust" is a badly named idea, because it sounds like "trust nothing," which is not actionable. The accurate version is:

> **Never trust something because of where it is on the network. Verify identity and authorisation on every request, for every resource, individually.**

The key phrase is *because of where it is on the network*. In the old model, network position **is** the credential — being inside means being trusted. Zero Trust removes network position from the trust calculation entirely. Being on the network proves nothing. It's just plumbing — a way for packets to travel, carrying zero authority on its own.

The formal version of this is **NIST SP 800-207**, the US standards document that defines the model. A few of its tenets, in plain language:

- **All resources are accessed the same way, whether you're in the office or in a café.** There's no "trusted internal" path with weaker checks.
- **Access is granted per-session, per-resource.** Access to one system grants nothing toward another.
- **Access decisions use identity, device state, and other context** — not IP address.
- **Everything is monitored and logged.** Every decision, allow or deny.

It also gives us two pieces of vocabulary that make the rest of this project legible:

- **PDP — Policy Decision Point.** The brain. The thing that evaluates "should this identity be allowed to do this?" and returns yes or no.
- **PEP — Policy Enforcement Point.** The muscle. The thing that sits physically in the traffic path, asks the PDP, and either forwards the request or blocks it.

**This entire project is the construction of a PEP and a PDP in front of two applications.** That sentence is the whole architecture. Everything that follows is the details of making that real on AWS.

## 1.6 What this project actually proves

It would be easy to build something that *looks* like access control and proves nothing. The bar set here was deliberately higher — three specific pieces of evidence:

1. A user who **should** have access gets in. (Proves the system isn't just blocking everything.)
2. A user who **should not** have access is **denied** — not by a suggestion, not by a hidden menu item, but by a hard `403 Unauthorized` before a single byte of the application is reachable. (Proves the system isn't just allowing everything.)
3. **Both events appear in a log**, with the identity, the timestamp, and the decision. (Proves the system is accountable, not silent.)

Anything less than all three is a demo, not a control. A control you can't prove denies is not a control — it's a hope.

---

# PART 2 — The journey of one request, start to finish

Before naming a single AWS service in detail, here's the entire system as one continuous story. This is the section to reread if any later part feels disconnected. **Everything in Part 3 and Part 4 exists to make one of these steps possible.**

A staff member named **said** opens a laptop at home and types `https://general.ztlab.sinosomtechnology.qzz.io` into a browser.

**Step 1 — DNS: "where does this name live?"**
The browser has a name, not an address. It asks the global DNS system to translate it. The answer does **not** point at the application server. It points at an AWS-managed Verified Access endpoint — a checkpoint. The application's real server has no public address at all and is not reachable by name, by IP, or by any other means from the internet. The only publicly resolvable thing in this entire architecture is the checkpoint.

**Step 2 — TLS: "prove you're really that name."**
The browser connects and demands a certificate. AWS presents one issued for `*.ztlab.sinosomtechnology.qzz.io`, cryptographically signed by an authority the browser already trusts. This proves the browser is talking to the genuine endpoint and not an impostor, and it encrypts everything from here on. Without this step, an attacker on the same café Wi-Fi could sit in the middle and read or alter the traffic.

**Step 3 — The checkpoint asks: "who are you?"**
The request arrives at the Verified Access endpoint. There is no session yet, so the endpoint has no identity to evaluate. It does not guess, and it does not default to allowing. It redirects the browser to the identity provider — AWS IAM Identity Center.
*Evidence: `general app url/redirected to login.png`*

**Step 4 — Identity: username and password.**
said authenticates against Identity Center. This is a separate system from Verified Access, deliberately: the thing that *verifies identity* and the thing that *enforces access* are different components with different jobs, and either can be replaced without rebuilding the other.
*Evidence: `general app url/login as said.png`*

**Step 5 — MFA: "prove it's really you, not just someone holding your password."**
A password is a thing you know, and things you know get phished, reused, and leaked. Identity Center demands a second factor from a device said physically holds. This is the specific control that makes a stolen password insufficient on its own.
*Evidence: `general app url/forced to MFA.png`, `general app url/mfa success.png`*

**Step 6 — Identity Center issues a verdict about *identity*, and nothing more.**
It confirms: this is said, authentication succeeded, and here is the set of groups said belongs to — `all-staff`. Notice what it does **not** say: it says nothing about whether said may enter this application. Identity Center answers "who is this." It does not answer "what may they do." Those are separate questions answered by separate systems, and keeping them separate is what makes the design scale.

**Step 7 — The Policy Decision Point evaluates.**
Verified Access now holds a verified identity and its group memberships. It evaluates the policies attached to this specific path, written in a language called Cedar. Two policies apply, and — this is the detail that produced the one real bug in this project — **both must independently say yes**:

- the **group-level** policy, which applies to every endpoint in the Verified Access group
- the **endpoint-level** policy, which applies only to this one application

For said reaching the general app: group policy requires `all-staff` → said is in `all-staff` → pass. Endpoint policy requires `all-staff` → pass. Both yes. Access granted.

**Step 8 — Only now does traffic move inward.**
Having decided yes, Verified Access forwards the request into the private network — to an internal load balancer, which passes it to the EC2 instance actually running the application. The user finally sees the page.
*Evidence: `general app url/am redirected to application.png`*

Note the ordering precisely: **the decision happens before the connection.** The application server never sees an unauthorised request. It isn't rejecting anyone — it never hears from them. This is meaningfully stronger than an application that receives requests and checks a session cookie itself, because it means a vulnerability *in the application's own login code* is not reachable by an unauthenticated attacker.

**Step 9 — The decision is written to a log.**
Allowed or denied, an entry goes to CloudWatch Logs: who, when, which application, what verdict. Enforcement without logging is a control nobody can audit, investigate, or prove after the fact.
*Evidence: `cloudwatch/access allow.png`*

**Step 10 — Now the interesting half. said tries the admin app.**
Same browser, same already-authenticated session, no new login. `https://admin.ztlab.sinosomtechnology.qzz.io`.

Identity is already established — Step 3 through 6 are skipped. But the authorisation decision in Step 7 runs again, **from scratch, against this application's own policy**, which requires membership in `admins-only`. said is not in `admins-only`. Denied. `403 Unauthorized`.
*Evidence: `general app url/try admin application for said got unauthorized access.png`*

**This single screenshot is the thesis of the entire project.** said is fully authenticated. said passed MFA. said is, by any traditional network definition, "inside." And it buys nothing. Access to one application conferred zero access to the other, because the two applications never shared a trust boundary — they only shared plumbing.

In a VPN world, said would have reached both, because both live on the same private network and the VPN's only question was "are you on the network."

**Step 11 — The denial is logged too.**
*Evidence: `cloudwatch/access deny.png`*

A denial that isn't recorded is invisible. The denials are frequently the more valuable log data — repeated denials against a sensitive application from a valid account is one of the clearest signals of either a compromised credential or an insider probing for reach.

---

That's the whole system. Ten steps. Everything below is the detail of how each step was made real, and why each component is shaped the way it is.

---

# PART 3 — Every building block, explained from zero

This part assumes nothing. If a term appeared in the build and felt like something to click past, it's explained here. The order is bottom-up: the network first, then what runs on it, then what protects it.

## 3.1 What a VPC actually is

**VPC = Virtual Private Cloud.** It is a private network that exists only for you, inside AWS's physical infrastructure.

The mental model that works: AWS owns an enormous number of physical machines in enormous buildings. Your resources and thousands of other customers' resources run on that same shared hardware. A VPC is the software boundary that makes your slice behave like a network you own alone — your own IP address range, your own routing rules, your own firewall policy, with no ability for another customer's machine to address yours at all.

Without a VPC, "private subnet" would be meaningless, because there'd be no defined space for something to be private *within*.

In this build: **`ZeroTrustLab-VPC`, address range `10.1.0.0/16`.**

*Evidence: `VPC/VPC available.png` — showing state Available.*

### Why `10.1.0.0/16`, and what that notation means

`10.1.0.0/16` is **CIDR notation**. It describes a block of IP addresses in two parts: a starting address, and a number saying how many of the leading bits are fixed.

An IPv4 address is 32 bits, written as four numbers (`10.1.0.0` = `00001010.00000001.00000000.00000000`). The `/16` means "the first 16 bits are fixed; the remaining 16 bits are free to vary."

- First 16 bits fixed → `10.1.` never changes
- Remaining 16 bits free → 2¹⁶ = **65,536 addresses**, from `10.1.0.0` to `10.1.255.255`

A smaller number after the slash means a bigger block. `/16` is big, `/24` is small (256 addresses), `/32` is exactly one address. This trips people up constantly: **smaller prefix number = larger network.**

Two reasons for `10.1.0.0/16` specifically:

**First, `10.x.x.x` is a private range.** RFC 1918 reserves three ranges that are not routable on the public internet: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. Anyone can use them internally; no internet router will forward traffic to them. Using a private range means the addresses inside this VPC are structurally incapable of being reached from the internet, before a single firewall rule exists.

**Second, `10.1` rather than `10.0` was a deliberate anti-collision choice.** The previous project (SecureWebLab) used `10.0.0.0/16`. If these two VPCs are ever peered, connected via Transit Gateway, or linked to the same on-premises network, overlapping ranges make routing ambiguous and the connection simply cannot be built. Address planning done casually at the start is one of the most common causes of painful, expensive re-architecture years later in real organisations. Two minutes of thought here avoids it.

## 3.2 What a subnet is, and why there are several

A subnet is a subdivision of the VPC's address space. `10.1.0.0/16` was split into smaller `/24` blocks, each holding 256 addresses.

Two independent reasons subnets exist:

**Reason 1 — Availability Zones.** An AWS region like `us-east-1` is not one building. It's several physically separate data centres, kilometres apart, with independent power and cooling and network feeds. Each is an **Availability Zone** (`us-east-1a`, `us-east-1b`, and so on). A subnet lives in exactly one AZ, always. So the *only* way to place resources in two different buildings is to create subnets in two different AZs. Subnets are how physical location gets expressed.

**Reason 2 — routing and policy boundaries.** Route tables attach to subnets, not to individual machines. So "these machines can reach the internet, those cannot" is expressed by putting them in different subnets with different route tables. The subnet is the unit at which network reachability is decided.

In this build:

| Subnet | CIDR | AZ | Purpose |
|---|---|---|---|
| `ZeroTrustLab-App-Subnet` | `10.1.1.0/24` | `us-east-1a` | The EC2 instances running the two applications |
| `ZeroTrustLab-LB-Subnet` | `10.1.2.0/24` | `us-east-1a` | Load balancer, AZ 1 |
| `ZeroTrustLab-LB-Subnet-2` | `10.1.3.0/24` | `us-east-1b` | Load balancer, AZ 2 |

### Why a second load balancer subnet existed at all

This was not planned upfront — it was forced by AWS partway through, and the reason is worth understanding rather than treating as a form to fill in.

**An Application Load Balancer must be given at least two subnets in two different Availability Zones.** AWS refuses to create one otherwise. This is a hard platform rule, not a recommendation.

The reason is what a load balancer *is*: a thing whose entire purpose is to keep an application reachable. A load balancer that exists in exactly one data centre becomes a single point of failure for availability — if that building has a problem, the load balancer disappears and every application behind it becomes unreachable, even if the application servers themselves are fine elsewhere. AWS refuses to let you build something whose stated job is availability but whose construction guarantees it can't deliver it.

So `ZeroTrustLab-LB-Subnet-2` was created in `us-east-1b` purely to satisfy this. In this lab it's mostly ceremonial — the actual application instances are still single-AZ, so real resilience isn't achieved. But the requirement taught something real about why the rule exists, which is worth more than working around it.

## 3.3 Route tables — the map the network reads

A route table is a lookup list. Every packet leaving a resource gets its destination compared against this list, and the most specific matching entry decides where it's sent.

A route table entry is `destination → target`:

```
10.1.0.0/16  →  local
```

That's the entry AWS creates automatically and will not let you delete. It says: anything addressed inside this VPC stays inside this VPC, delivered directly. It's the reason the load balancer can reach the application instances without any configuration at all — they're all inside `10.1.0.0/16`, so the `local` route handles it.

In a typical internet-facing setup you'd also see:

```
0.0.0.0/0  →  igw-xxxxxxxx    (an Internet Gateway)
```

`0.0.0.0/0` means "literally any address" — the default route, the catch-all for anything not matched by a more specific entry. Pointing it at an Internet Gateway is what makes a subnet "public."

**What matters in this project is what's absent.** The application subnet's route table has no `0.0.0.0/0` entry to an Internet Gateway. There is no path out, and structurally therefore no path in. The instances are not "firewalled off from the internet" — they have *no route to it at all*. That's a stronger property than a blocking rule, because a blocking rule can be misconfigured, reordered, or removed by someone who doesn't understand it. A missing route is not a rule that can be weakened; it's an absent capability.

*Evidence: `Route table/route table.png`*

**Matching is most-specific-wins.** If a table contains both `0.0.0.0/0 → A` and `10.1.1.0/24 → B`, traffic to `10.1.1.50` uses B, because `/24` is more specific than `/0`. This is the same longest-prefix-match logic every router on the internet uses.

## 3.4 Security groups — the firewall that travels with the resource

A traditional network firewall sits at a chokepoint and inspects traffic crossing it. An AWS **security group** is different: it attaches to the network interface of the resource itself. Every resource carries its own firewall.

Three properties that matter:

**It's default-deny.** A security group with no inbound rules permits no inbound traffic. You never write "deny" rules — there is no deny. You only write what's permitted, and everything unlisted is refused. This is the safe default direction: mistakes produce broken connectivity you notice immediately, rather than silent exposure you notice in a breach report.

**It's stateful.** If an inbound request is allowed, the response is automatically allowed back out, even with no matching outbound rule. The security group remembers the connection. This is why you don't have to write mirror-image rules for every conversation, and it's the main practical difference from Network ACLs, which are stateless and operate at the subnet level rather than the resource level.

**The source can be another security group, not just an IP range.** This is the underrated feature. You can write "allow HTTPS from anything carrying `ZeroTrustLab-ALB-SG`" — an identity-based rather than address-based rule. If the load balancer's IP changes, the rule still holds, because the rule was never about addresses.

### The two security groups in this build

**`ZeroTrustLab-App-SG`** — attached to the application instances:

- **Inbound: HTTP (80) from `10.1.0.0/16`** — the whole VPC range, not the internet. This lets the load balancer reach the web server. `10.1.0.0/16` is an internal range with no internet route, so this rule is structurally incapable of allowing an internet host, no matter how it's read.
- **Inbound: HTTPS (443) from `10.1.0.0/16`** — added mid-build, and the reason is instructive. The SSM VPC endpoints (see 3.6) communicate over 443, and the original rule only permitted 80. Nothing was broken; a required protocol was simply missing. This is the ordinary texture of real network work: connectivity failures are far more often "a port nobody thought about" than anything conceptually wrong.
- **Outbound: all** — default. The instance needs to reach AWS services and package repositories.

*Evidence: `EC2/security group APP-SG details page.png`, `EC2/security group APP-SG inbound rules.png`, `EC2/security group APP-SG outbound rules.png`*

**`ZeroTrustLab-ALB-SG`** — attached to the load balancers:

- **Inbound: HTTPS (443) from `10.1.0.0/16`** — the load balancer is *internal*, so the only thing that ever legitimately talks to it is the Verified Access endpoint's network interface, which lives inside the VPC.

*Evidence: `EC2/security group APP-ALB details page.png`, `EC2/security group APP-ALB inboud rule.png`, `EC2/security group APP-ALB outbound rule.png`*

**What you should see when reading these screenshots:** the string `0.0.0.0/0` appears in no inbound rule anywhere in this project. Not on the instances, not on the load balancers. `0.0.0.0/0` inbound — especially on port 22 — is among the most common findings in real cloud security audits, and it's usually there because someone needed to test something quickly and never came back. The habit being demonstrated is scoping correctly the first time, because "I'll tighten it later" is a promise that is empirically almost never kept.

## 3.5 EC2 — the applications themselves

**EC2 = Elastic Compute Cloud.** A virtual machine running on AWS hardware. Two were launched:

- `ZeroTrustLab-GeneralApp` — the low-sensitivity application ("Meridian Ops", access level: All Staff)
- `ZeroTrustLab-AdminApp` — the high-sensitivity application

Both run a trivial web server (`httpd`) serving a single page that states which application it is and what access level it represents.

**Why two, and why deliberately different sensitivity levels?** Because with one application, this project would prove nothing. "Authenticated users can reach the app" is just a login page — every website on earth does that. The entire thesis of Zero Trust is that **access to one resource confers nothing toward another**, and demonstrating that requires at minimum two resources with genuinely different access rules, plus a user who legitimately holds one and not the other. The second application isn't extra work; it's the experimental control.

**Why the applications are deliberately trivial.** Nothing about this project's claims depends on what the applications do. A real internal dashboard would add hundreds of lines of code, a database, a session layer — and would prove *less*, because the security boundary being tested lives entirely outside the application. A one-line HTML page makes the boundary unmistakable: if the page loads, the request passed every check; if it doesn't, it didn't. There's no application logic to confuse the result with.

**Launched with no public IPv4 address.** This is the single most important configuration on these instances. With no public IP:

- no host on the internet can address them
- no port scan can find them
- there is no login prompt exposed to be brute-forced
- there is no SSH daemon reachable from outside to have a vulnerability in

*Evidence: `EC2/apps running and available.png` — both instances Running, Public IPv4 column empty.*

That empty column looks like something went wrong. It's the opposite. "Running, healthy, and unreachable from the internet" is the intended and correct end state for a backend system. The rest of this architecture is about providing exactly one narrow, verified, logged path to something in that state — rather than opening it up and trying to control the opening afterwards.

## 3.6 A detour: how do you administer a server you can't reach?

This is a genuine practical problem created by the previous decision, and it's worth documenting because the solution is itself a Zero Trust pattern.

The instances have no public IP and no inbound SSH. So how was `httpd` installed and verified?

**AWS Systems Manager Session Manager.** Instead of you connecting *inward* to the server, an agent on the server connects *outward* to the AWS Systems Manager service and waits. When you open a session in the console, it goes to Systems Manager, which relays it down the connection the agent already established.

What this changes:
- **No inbound port.** Zero. The security group needs no SSH rule, because nothing connects inward.
- **No SSH key.** No key file to lose, leak, commit to a repository, or leave on a former employee's laptop.
- **Authentication is IAM.** Access is your AWS identity, with all of IAM's policy machinery, MFA, and revocation — not a shared secret in a file.
- **Every session is logged in CloudTrail**, automatically, as a `StartSession` event with the calling identity.

That last point is worth sitting with. Traditional SSH with a shared key frequently leaves *no record of who actually connected*, because the key doesn't identify a person. "Who accessed this server on Tuesday?" is a question a lot of organisations genuinely cannot answer. This model answers it by default, at no extra cost and with no extra configuration.

**The VPC endpoints.** For the agent to reach Systems Manager, it needs a network path to it. Normally that's a NAT Gateway providing outbound internet. Instead, three **interface VPC endpoints** were created:

- `com.amazonaws.us-east-1.ssm`
- `com.amazonaws.us-east-1.ssmmessages`
- `com.amazonaws.us-east-1.ec2messages`

An interface VPC endpoint places a network interface for an AWS service *inside your VPC*, with a private IP from your own range. The instance then reaches Systems Manager at `10.1.1.x` — traffic that never leaves AWS's network and never touches the public internet at all.

This is strictly better than a NAT Gateway for this purpose: narrower (only the specific services you choose, not all outbound internet), cheaper at this scale, and it keeps the property that the instances have no general internet path.

**These endpoints bill hourly (~$0.01/hr each) and were deleted after verification.** They're plumbing for the build, not part of the running architecture.

## 3.7 What a load balancer actually is — and why one is here at all

This is the component that most deserves a real explanation, because its name describes its least important function in this project.

### The original purpose

The name comes from the obvious case: one application, more traffic than one server can handle, so you run several identical servers and put something in front to distribute requests across them. That thing *balances the load*. Hence the name.

That is not why a load balancer is in this project — there's exactly one instance behind each one. So why is it here?

### What a load balancer really is: a stable front door

The genuinely important property is **indirection**. The load balancer is a fixed, named, stable thing that clients talk to, decoupled from the actual servers behind it, which can change freely.

That decoupling buys a lot:

- **Servers can be replaced without anything in front knowing.** Terminate an instance, launch a new one, register it — the front door never moved.
- **Health checking.** The load balancer continuously probes each server. If one stops responding correctly, it's removed from rotation automatically and traffic goes elsewhere. Failure becomes a non-event instead of an outage.
- **TLS termination.** The load balancer holds the certificate and does the encryption work, so individual application servers don't each need a copy of the private key or the CPU cost of handling TLS.
- **A single place to attach policy** — routing rules, headers, logging, WAF — instead of configuring every server identically and hoping they stay identical.

### Why Verified Access specifically requires one

This is the concrete answer for this build: **an AWS Verified Access endpoint of type "load balancer" attaches to a load balancer. It does not attach to an EC2 instance.**

And the reason is architectural, not arbitrary. Verified Access is designed to protect *an application*, which is a stable, addressable thing that outlives any particular server. If endpoints attached directly to instances, every instance replacement would break the access configuration, and every application with more than one server would need multiple endpoints with no coherent way to keep their policies in sync. Attaching to a load balancer means the security control is bound to the application as a concept, which is the correct granularity for an access policy.

So in this build the load balancer serves as a required, stable attachment point that turns "this specific virtual machine" into "this application" — a thing an access policy can sensibly be written about.

### ALB vs NLB, and why ALB

AWS offers several load balancer types. The relevant distinction:

- **Network Load Balancer (NLB)** operates at **Layer 4** — TCP/UDP. It sees addresses and ports. It has no idea what HTTP is. Extremely fast, protocol-agnostic, but it can't make decisions based on anything inside the request.
- **Application Load Balancer (ALB)** operates at **Layer 7** — HTTP/HTTPS. It parses the actual request: the path, the headers, the hostname, the cookies. It can route `/api` to one set of servers and `/admin` to another, terminate TLS, and rewrite headers.

("Layer" refers to the OSI model — the standard way of describing network stacks in layers, where Layer 3 is IP addressing, Layer 4 is TCP ports, and Layer 7 is the application protocol like HTTP.)

**ALB was correct here** because Verified Access's HTTP/HTTPS endpoint type is fundamentally an HTTP-aware control. It works with browser redirects to a login page, session cookies, and per-request HTTP-level evaluation. All of that is Layer 7. An NLB, which cannot see HTTP at all, has no ability to participate in that flow.

### Scheme: Internal — the most important dropdown in the build

When creating an ALB, AWS asks for a **scheme**, with two options:

- **Internet-facing** — gets public IP addresses, resolvable and reachable from the internet
- **Internal** — gets only private IPs from your subnets, reachable only from within the VPC

**Internal was selected, and choosing wrong here would have silently destroyed the entire project.**

If these ALBs were internet-facing, anyone on the internet who discovered their DNS names could connect directly to them, and the ALB would forward straight to the applications — **completely bypassing Verified Access**. Every policy, every group, every MFA prompt, every log entry: skipped. The applications would be publicly accessible to anyone who found the bypass address, while the front door continued to look rigorously secured.

This is worth generalising, because it's a class of failure rather than a one-off gotcha: **a security control that can be gone around is not a security control.** The only thing that makes Verified Access meaningful is that there is *no other path* to the application. The internal scheme is what guarantees that. Every other component in this project can be understood as either creating that single path or proving nothing else exists.

*Evidence: `load balancer/active load balancer.png` — Scheme column reading `internal` for both. `load balancer/General ALB details page.png` — full configuration including the HTTPS:443 listener and attached certificate.*

### Target groups and health checks

A **target group** is the list of servers the load balancer forwards to, plus the rules for checking whether they're alive.

The ALB doesn't hold a list of servers directly. It has listeners, and listeners have rules, and rules forward to target groups, and target groups contain targets. The extra layer exists so one ALB can serve many applications by routing different hostnames or paths to different target groups.

The **health check** is a request the ALB sends to each target on a schedule — by default `GET /`, expecting HTTP 200. Targets that respond correctly are `healthy` and receive traffic; targets that don't are `unhealthy` and are silently pulled out.

*Evidence: `load balancer/target groups Healthy.png`*

That "healthy" status is doing real evidentiary work in this project. It proves the applications were genuinely running and serving content over the network, independently of the access control layer. Without it, a failed access test would be ambiguous: was access denied, or was the app simply broken? A healthy target group removes that ambiguity — any subsequent failure is an access decision, not a dead service.

### The listener

A **listener** is the port and protocol the load balancer accepts on. Here: **HTTPS on 443**, with the ACM certificate attached.

Note what happens across the hop: the connection into the ALB is HTTPS:443, and the ALB then forwards to the instance over HTTP:80. TLS is *terminated* at the load balancer. The final hop is unencrypted — but it's unencrypted **inside a private VPC subnet with no internet route**, between two resources whose security groups permit only each other's traffic. This is a normal and accepted pattern. In a stricter environment (regulated data, zero-trust-to-the-host), you'd re-encrypt that last hop too; the honest note here is that this build didn't, and the reason is that it's a lab, not that the last hop doesn't matter.

## 3.8 DNS — how a name becomes an address

Every device on the internet is reachable by an IP address. Humans don't use them. **DNS (Domain Name System)** is the global distributed database that translates names into addresses.

The resolution of `general.ztlab.sinosomtechnology.qzz.io` is read **right to left**, most general to most specific:

- `io` — a top-level domain
- `qzz.io` — a domain under it
- `sinosomtechnology.qzz.io` — the domain actually controlled here
- `ztlab.sinosomtechnology.qzz.io` — a subdomain, created purely to group this project's names
- `general.ztlab.sinosomtechnology.qzz.io` — the specific application name

Owning a domain means controlling its **DNS zone** — the authoritative list of records saying what each name under it points to. In this build, that zone is hosted at **DigitalPlat**, not Route 53. That's a perfectly valid arrangement: AWS doesn't need to host your DNS, it only needs you to be able to create records. It does mean every DNS change in this project was manual copy-paste rather than an automated AWS-side update, which matters mainly for how validation worked (below).

**Record types used here:**

- **CNAME** — "this name is an alias for that other name." `general.ztlab.sinosomtechnology.qzz.io` is a CNAME pointing at the AWS-generated Verified Access endpoint domain. When AWS changes the underlying addresses, nothing here needs updating, because the record points at a *name*, not an address.
- **A** — a name pointing directly at an IPv4 address. Not used for the endpoints, precisely because AWS-managed addresses change.

**Why the extra `ztlab` layer?** Nothing technical requires it. It's organisational hygiene: every name belonging to this project sits under one branch of the domain, so it's immediately obvious what a name is for, and the whole project can be cleaned up by removing one subtree. In a real organisation with dozens of services under one domain, this kind of structure is the difference between a navigable DNS zone and a landfill.

*Evidence: `certification verification/two endpoints subdomains.png` — both subdomain records pointing at their Verified Access endpoints.*

## 3.9 TLS and certificates — why the padlock exists

**TLS (Transport Layer Security)**, the successor to SSL, is what makes the `s` in `https`. It provides three things:

1. **Encryption** — nobody between the browser and the server can read the traffic
2. **Integrity** — nobody can modify it undetected in transit
3. **Authentication** — the browser can verify the server really is the name it claims

The third is the one people skip, and it's the one that matters most here. Encryption alone is worthless if you're encrypting to an attacker. Without authentication, someone on the same Wi-Fi could intercept the connection, present themselves as the login page, encrypt everything beautifully, and harvest credentials — with a padlock icon showing the whole time.

**A certificate is how the third property is achieved.** It's a signed document stating "the holder of this private key is genuinely `general.ztlab.sinosomtechnology.qzz.io`," signed by a **Certificate Authority** whose signing key browsers already trust, out of the box. The browser checks the signature chain, checks the name matches, checks the dates, and only then proceeds.

### Why the certificate needed DNS validation

A CA won't sign a certificate for a name you don't control — otherwise anyone could get a valid certificate for any bank on earth and the entire system would be meaningless. So the CA demands proof of control.

**AWS Certificate Manager (ACM)** issued this certificate, using **DNS validation**: ACM generated a unique random CNAME record and required it to appear in the domain's DNS zone. Only someone who genuinely controls the zone can create that record. Creating it proves control, and ACM issues the certificate.

Those records were added manually in DigitalPlat. This is the step where friction usually appears, and it did here: DNS dashboards differ in whether the "name" field expects the full record name or just the portion before the root domain, and getting it wrong produces a record that looks right and validates nothing.

*Evidence: `certification verification/digitalplat record added .png` — the validation CNAMEs in the DigitalPlat zone. `certification verification/cert issued.png` — ACM showing status **Issued**, with both subdomains as Subject Alternative Names on one certificate.*

**Why one certificate with two names rather than two certificates?** A certificate can carry multiple **Subject Alternative Names** (SANs). One certificate covering both `general.` and `admin.` means one validation process, one renewal to track, one expiry date, and one object attached to both load balancers. Fewer moving parts, fewer things to forget. Certificate expiry is one of the most reliably recurring causes of self-inflicted production outages; halving the number of certificates halves the surface for that mistake.

**Why ACM specifically:** ACM certificates are free, and they **renew automatically** as long as the DNS validation record stays in place. Manual certificate renewal is a calendar event that eventually gets missed. Automating it away is the correct engineering decision, not a convenience.

**One constraint worth knowing:** ACM certificates are **regional**. A certificate in `us-east-1` cannot be attached to a load balancer in `eu-west-1`. This is why holding the region constant at `us-east-1` across the whole build mattered — a mismatch produces a certificate that simply doesn't appear in the dropdown, with no explanation offered.

## 3.10 Identity — IAM Identity Center

**AWS IAM Identity Center** (formerly AWS SSO) is the identity provider in this architecture. Its job, and its only job: establish **who someone is**.

It holds:

- **Users** — `said` and the admin user, each with credentials and a registered MFA device
- **Groups** — `all-staff` and `admins-only`
- **Memberships** — which users are in which groups
- **MFA enforcement** — the requirement for a second factor at sign-in

*Evidence: `IAM/users.png`, `IAM/groups.png`*

### Why groups, not direct user permissions

This is one of those decisions that looks like extra work at two users and becomes load-bearing at two hundred.

Policies in this project reference **groups**, never individual users. So:

- Onboarding is one action: add to the right groups. No policies are touched.
- Offboarding is one action: disable the user. Every policy everywhere stops applying instantly.
- "Who can reach the admin app?" has a single authoritative answer — the membership list of `admins-only` — instead of requiring an audit of every policy in the account.
- Access changes are **membership** changes, not **policy** changes. Policies stay stable, reviewed, and rarely edited. Memberships change daily. Keeping the volatile thing separate from the reviewed thing is the entire point.

The failure mode this avoids is real and extremely common: policies that name individual users, accumulated over years, so that nobody can answer "what can this person actually reach" without reading everything, and revoking access reliably becomes practically impossible.

### Why MFA is not optional here

A password is a shared secret. It can be phished, reused from a breached site, guessed, keylogged, or shoulder-surfed — and when it's stolen, **nothing observable changes**. The legitimate user notices nothing. The attacker becomes indistinguishable from them.

MFA requires a second factor from a device physically in the user's possession, generating a time-based code that can't be reused. Stealing the password is no longer sufficient; the attacker also needs the device.

*Evidence: `general app url/forced to MFA.png`, `general app url/mfa success.png`, `admin app url/register MFA asked.png`, `admin app url/MFA success.png`*

The word **forced** in that first filename matters. MFA was not offered as an option the user could dismiss. It was required before any access decision could be made at all. Optional MFA has adoption rates that make it approximately decorative; the only meaningful MFA is enforced MFA.

**The important boundary:** Identity Center answers "who is this," and stops. It does **not** answer "may they enter the admin application." That question belongs to Verified Access. This separation — **authentication** (who you are) versus **authorisation** (what you may do) — is one of the foundational distinctions in security architecture, and this build keeps them in genuinely separate systems. The practical benefit: the identity provider could be swapped for Okta, Entra ID, or any OIDC provider without touching a single access policy, and access policies can be rewritten without touching identity.

## 3.11 AWS Verified Access — the four components

This is the actual Zero Trust engine. It has four parts, and the relationships between them are the part worth understanding.

### 1. The Trust Provider

The trust provider tells Verified Access **where to get verified identity from**. Type: IAM Identity Center.

This is what makes Verified Access a *policy* system rather than an *identity* system. It doesn't store users or check passwords. It delegates that entirely, and consumes the result. Verified Access also supports device-based trust providers (Jamf, CrowdStrike, Intune) that report on the *device's* posture — patch level, disk encryption, EDR agent present — and policies can require both. That's the full Zero Trust picture: identity plus device context. This build uses identity only, which is the honest scope.

*Evidence: `Verified Access/1 Verified Access trust providers tab.png`*

### 2. The Verified Access Instance

The instance is the running service itself — the thing that actually sits in the request path, terminates the user's connection, redirects to login, evaluates policy, and forwards or refuses. It is the **Policy Enforcement Point** from Part 1. It is also where logging is configured, which is why every decision across every endpoint lands in one place.

### 3. The Verified Access Group

A logical container that endpoints belong to, so that **policy shared by multiple applications can be written once**. 

Both endpoints in this build belong to `ZeroTrustLab-Group`. The intent was for the group to carry a baseline rule — something like "must be an authenticated employee" — applying to everything, with each endpoint adding its own specific requirement on top.

*Evidence: `Verified Access/2 Verified Access group.png`*

**This component is where the project's one real bug lived.** Full account in Part 5.

### 4. The Verified Access Endpoints

An endpoint is the checkpoint in front of **one** application. Two were built: one for the general app, one for the admin app.

Each endpoint holds:

- **Attachment** — which VPC and Verified Access group it belongs to
- **Endpoint type** — `load-balancer`, attached to the corresponding internal ALB
- **Application domain** — the name users type (`general.ztlab.sinosomtechnology.qzz.io`)
- **Endpoint domain prefix** — a string AWS combines with generated values to produce the AWS-side domain name that DNS points at
- **Certificate** — the ACM certificate covering that name
- **Policy** — the Cedar rules specific to this application

*Evidence: `Verified Access/3 Verified Access endpoints.png`, `Verified Access/4 General endpoint details page.png`, `Verified Access/6 admin endpoint details page.png`*

**One endpoint per application is the entire design, not an implementation detail.** It's the structural expression of "access is granted per resource, never to a network." Two applications on the same subnet, behind the same group, sharing the same VPC — and completely independent access decisions, because the checkpoint is bound to the application rather than to the network the application happens to sit on.

**This is also the thing that bills at ~$0.27 per endpoint per hour.** Two endpoints ≈ $0.54/hr, from the moment they exist, with no free tier. That cost shaped the entire schedule of the build (see Part 8).

## 3.12 Cedar — the policy language

**Cedar** is a policy language AWS built and open-sourced, used by Verified Access and Amazon Verified Permissions. It was designed for one purpose: expressing authorisation rules that can be *analysed*, not just executed.

The basic shape:

```cedar
permit(principal, action, resource)
when {
    <condition>
};
```

The general app's policy, in effect:

```cedar
permit(principal, action, resource)
when {
    context.<trust-provider-reference>.groups has "<all-staff group ID>"
};
```

The admin app's policy is identical in shape, referencing the `admins-only` group ID instead.

*Evidence: `Verified Access/5 General endpoint policy page.png`, `Verified Access/7 admin endpoint policy page.png`*

### Reading it properly

- `permit(principal, action, resource)` — the three unconstrained positions mean "for any caller, any action, any resource." All the real work is in the `when` block.
- `context` — everything Verified Access knows about *this specific request*: the identity claims from the trust provider, and optionally device signals, time, and request metadata.
- `.groups has "..."` — group membership is delivered as a record keyed by group **ID**, and `has` tests for the presence of that key.

### Why group IDs instead of names

The policies reference opaque identifiers like `b4080488-d071-706a-cea5-1ae81ac7065b` rather than the readable string `admins-only`.

This is deliberate on AWS's part, and correct. Group **names** are mutable — someone renames `admins-only` to `platform-admins` in a tidy-up, and every policy referencing the old name silently stops matching. Depending on how the policy is written, that failure can be either a lockout (annoying, immediately visible) or an unintended grant (catastrophic, invisible). Group **IDs** are immutable for the life of the group, so the binding survives renames.

The cost is human readability, and that cost is real — it is precisely what made the bug in Part 5 hard to see, because two structurally identical policies differing only in a 36-character hex string look approximately the same to a human eye. The correct mitigation is a comment in the policy naming the group in plain language. Worth adopting.

### The critical semantic: policies are default-deny, and multiple policies are ANDed

Two rules that govern everything about how these policies behave:

1. **Cedar is default-deny.** If no `permit` matches, the answer is deny. Access is never granted by the absence of a rule. Writing a policy that grants nothing produces a locked door, not an open one — the safe direction to fail in.

2. **If both a group policy and an endpoint policy exist, both must independently permit.** They are not alternatives and they are not merged — they are evaluated separately and ANDed. Passing one and failing the other is a denial.

That second rule is not obvious from the console, which presents the two policy editors on different pages with no visual indication that they combine. It is the direct cause of the bug in Part 5, and understanding it is the single most transferable thing in this document.

## 3.13 CloudWatch Logs — the accountability layer

**CloudWatch Logs** is AWS's log storage and search service. Verified Access was configured to deliver every access decision there.

Each entry records:

- **who** — the authenticated identity from the trust provider
- **when** — timestamp
- **what** — which endpoint / application was requested
- **verdict** — allowed or denied

*Evidence: `cloudwatch/cloudwatch is watching.png`, `cloudwatch/access allow.png`, `cloudwatch/access deny.png`*

**Why this is not optional.** A control that enforces but doesn't record can't be investigated, can't be audited, and can't be proven to have worked. "Would you know if someone tried to access the admin system?" is a question a security team asks constantly, and the answer for most systems is honestly no.

The **denials** are the higher-value data. A successful access is normal. A pattern of denials — one valid account repeatedly probing a system it has no rights to — is one of the cleanest signals of a compromised credential or an insider mapping out reach. Logging only successes captures the boring half and discards the interesting half.

---

# PART 4 — The build, in order, and why that order

Build order was not arbitrary. It was driven by two constraints that pulled in the same direction:

**Constraint 1 — dependency.** Some things cannot exist before others. A certificate can't be attached to a load balancer that doesn't exist. An endpoint can't reference a policy that hasn't been written. Dependencies fix part of the order for you.

**Constraint 2 — cost.** Verified Access bills ~$0.27 per application per hour from creation, with no free tier. Two endpoints is roughly $0.54/hr — small in absolute terms, but it accrues whether or not anyone is using it, including overnight while you sleep.

Constraint 2 produces a rule that shaped everything: **build every free thing completely first, verify it works, and only then create the billing resources — then move immediately to testing and immediately to teardown.**

The billing resources (load balancers and Verified Access endpoints) were the *last* things created and the *first* things destroyed. The expensive window was measured in hours, not days. This is cost discipline as an engineering constraint, and it's a genuinely relevant professional skill — cloud cost incidents are overwhelmingly caused by resources nobody remembered were running, not by deliberate overspend.

## Phase 0 — Safety net

Confirmed the zero-spend AWS Budgets alert from the previous project was still active, and confirmed the region selector read `us-east-1`.

**Why a budget alert first, before anything:** because the failure mode being defended against is *forgetting*. A resource left running doesn't announce itself; it shows up on a bill weeks later. A budget alert is an independent tripwire that doesn't depend on remembering to check. Establishing it before creating anything billable is the same instinct as turning on the smoke detector before lighting the stove.

**Why region matters enough to check:** the AWS console remembers a region per browser session and silently switches on you. Resources in different regions cannot see each other. A certificate in the wrong region simply won't appear in the load balancer's dropdown, and the console won't explain why — it just shows an empty list. Holding `us-east-1` constant eliminated an entire class of confusing, hard-to-diagnose failures.

## Phase 1 — The network

Created `ZeroTrustLab-VPC` (`10.1.0.0/16`) and the subnets.

**Why a new VPC instead of reusing the previous project's:** three reasons, all real.

1. **Blast radius.** A mistake in this project can't affect the other one. Independent VPCs are independent failure domains.
2. **Clean teardown.** Deleting this project means deleting one VPC, with no risk of removing something another project needed.
3. **Honest demonstration.** The project stands alone. Nobody reading the repository needs to understand a different build to understand this one.

*Evidence: `VPC/VPC available.png`, `Route table/route table.png`*

## Phase 2 — The applications

Security group first, then IAM role, then the SSM VPC endpoints, then the instances.

**Why the security group before the instances:** security groups are scoped to a specific VPC and can't be created during instance launch with full rule sets cleanly. More importantly, creating it first means the instance is protected from the instant it boots, rather than existing briefly in a default state while you go and configure its firewall. Small window, but the habit of "protection exists before the thing that needs protecting" is the right one.

**Why an IAM role rather than credentials on the instance:** an IAM **role** is an identity a *resource* wears, as opposed to an IAM **user**, which is an identity a *person* logs in as. Attaching `ZeroTrustLab-SSM-Role` with the AWS-managed `AmazonSSMManagedInstanceCore` policy means the instance obtains short-lived, automatically-rotated credentials from the instance metadata service. No access key is ever stored on disk, in an environment variable, or in a file that can be read by a compromised process.

**Why that specific policy and nothing broader:** `AmazonSSMManagedInstanceCore` grants exactly what the SSM agent needs — register with Systems Manager, accept a session — and nothing else. Not `AdministratorAccess`. Not broad EC2 rights. An over-permissioned instance role is one of the most common real audit findings, and it matters because instance credentials are exactly what an attacker harvests first after compromising a web application. The blast radius of that compromise is defined entirely by what the role can do.

The instances were then launched into the private subnet with **auto-assign public IP disabled** and no key pair — "proceed without a key pair" chosen deliberately, because SSH is not part of this architecture at all.

*Evidence: `EC2/apps running and available.png`*

## Phase 3 — Identity

Enabled IAM Identity Center, created the two users, created `all-staff` and `admins-only`, assigned memberships, enforced MFA.

**Why identity was built before policies:** policies reference group IDs, and group IDs don't exist until the groups do. More practically: getting the identity model right first forces the question "what are the actual access tiers here?" to be answered deliberately, before any policy is written. Policies written before the identity model is settled tend to encode accidents.

**The membership design at this stage:**

| User | Groups | Intended reach |
|---|---|---|
| said | `all-staff` | General app only |
| admin | `admins-only` | Both apps |

That table contains the seed of the bug. Look at it and hold the question: *if a policy anywhere in the system requires `all-staff`, what happens to admin?* This was not visible at the time and became obvious only in hindsight — which is exactly why it's worth documenting rather than quietly fixing.

*Evidence: `IAM/users.png`, `IAM/groups.png`*

## Phase 4 — Domain and certificate

Requested one ACM public certificate covering both subdomains, validated by DNS through records added manually in DigitalPlat, and confirmed **Issued**.

**Why before the load balancers:** the ALB creation form asks for a certificate at listener configuration time. An unissued certificate doesn't appear. And DNS validation takes an unpredictable amount of time — minutes usually, sometimes much longer, depending on propagation. Waiting for a certificate is free. Waiting for a certificate with two ALBs already billing is not.

*Evidence: `certification verification/digitalplat record added .png`, `certification verification/cert issued.png`*

## Phase 5 — The load balancers

Second-AZ subnet, then `ZeroTrustLab-ALB-SG`, then target groups, then the ALBs themselves.

**Target group before ALB:** the ALB creation form requires a default action, and the default action is "forward to a target group." The target group must already exist. This ordering is forced by the console, but it reflects something real: the ALB is defined by where it sends traffic, so the destination has to be defined first.

**The dropdown choices that actually mattered, and what each one was:**

| Field | Chosen | Why |
|---|---|---|
| Load balancer type | Application | Layer 7 / HTTP-aware; required by Verified Access's HTTP endpoint type |
| **Scheme** | **Internal** | **The single most important choice in the build — see 3.7. Internet-facing would have created a complete bypass of every access control.** |
| Mappings | `us-east-1a` + `us-east-1b` | Two-AZ minimum enforced by AWS |
| Security group | `ZeroTrustLab-ALB-SG` (default removed) | The auto-selected default SG is permissive and belongs to nothing in particular; leaving it attached alongside a scoped one means the permissive rules still apply |
| Listener | HTTPS : 443 | Encrypted, matching what Verified Access forwards |
| Certificate | The ACM cert covering both names | Same cert serves both ALBs |
| Default action | Forward to the matching target group | Where traffic actually goes |

**Note on removing the default security group.** Security groups are additive — attaching two means the union of their rules applies. Adding a tightly-scoped group while leaving a permissive default attached achieves nothing, because the permissive rules are still in force. Removing it is not cosmetic; it's the difference between the scoped rules being the policy and being decoration.

**Billing starts here.** From the moment the first ALB became Active, the clock was running.

*Evidence: `load balancer/active load balancer.png`, `load balancer/General ALB details page.png`, `load balancer/target groups Healthy.png`*

The healthy target groups were the checkpoint before proceeding. Building Verified Access on top of an application that isn't actually serving traffic means every subsequent failure is ambiguous. Confirming health first meant that from that point on, any failure could be attributed to the access layer with confidence.

## Phase 6 — Verified Access trust provider, instance, group

Created in that order, because each depends on the previous: the instance references the trust provider, and the group belongs to the instance.

**Creating these is free.** Billing begins at endpoint association in Phase 8. This is why the maximum possible amount of Verified Access configuration was completed here, before the expensive phase.

A **policy was attached at the group level** during this phase, intended as a baseline "must be an authenticated employee" gate.

That policy is the bug. It was written correctly by intention and wrongly by reference, and nothing in the console suggested anything was off.

*Evidence: `Verified Access/1 Verified Access trust providers tab.png`, `Verified Access/2 Verified Access group.png`*

## Phase 7 — Policies, written before going live

Both endpoint policies were written and syntax-checked before any endpoint existed.

**Why in advance:** policy authoring is the slowest, most iterative part of this kind of work — syntax errors, wrong references, uncertainty about the exact context structure. Doing it while two endpoints bill at $0.54/hr means paying for thinking time. Doing it first means Phase 8 and 9 are mechanical and fast.

This is the general pattern worth taking away: **when a resource bills per hour, move every decision you can *before* the resource exists.** The goal is for the expensive window to contain only actions, never deliberation.

## Phase 8 — Endpoints created, DNS pointed, system live

Two Verified Access endpoints created, attached to their respective ALBs, each with its policy, each with the certificate. Then the DNS CNAMEs in DigitalPlat pointed at the AWS-generated endpoint domains.

**This is the moment the architecture became real** — and the moment the expensive clock started properly.

*Evidence: `Verified Access/3 Verified Access endpoints.png`, `Verified Access/4 General endpoint details page.png`, `Verified Access/5 General endpoint policy page.png`, `Verified Access/6 admin endpoint details page.png`, `Verified Access/7 admin endpoint policy page.png`, `certification verification/two endpoints subdomains.png`*

## Phase 9 — Proof

### Test 1 — said, general app: should succeed

1. `check to general url.png` — the initial request
2. `redirected to login.png` — no session, so redirect to Identity Center. **The redirect itself is the first piece of evidence in the whole test**: an unauthenticated request did not reach the application, was not silently allowed, and was not served a default page. It was intercepted before the application existed as far as the browser was concerned.
3. `login as said.png` — authentication
4. `forced to MFA.png` — second factor demanded, not offered
5. `mfa success.png` — second factor accepted
6. `am redirected to application.png` — **the application loads.** "Meridian Ops", Access level: All Staff.

**What this sequence proves:** the full chain — DNS → TLS → interception → identity → MFA → policy evaluation → forwarding through the ALB → the real application — works end to end. Every layer functioned, in order.

### Test 2 — said, admin app: should fail

Same browser. Same live session. No re-authentication. Straight to the admin URL.

`try admin application for said got unauthorized access.png` — **403 Unauthorized.**

**This is the single most important artefact in the project.** Everything else demonstrates that a correctly-built system lets the right people in, which is the easy half. This demonstrates that it *keeps the wrong people out* — and specifically, that it keeps out someone who is fully authenticated, MFA-verified, and by every traditional network measure already "inside."

Under a VPN model, this request succeeds. said is on the network; the admin app is on the network; there is no second checkpoint. Here, network position bought exactly nothing, because authorisation was re-evaluated independently for this resource, against this resource's own policy.

**Note also what the denial is not.** It's not a hidden link, not a greyed-out button, not an application-level "you don't have permission" page. It's a hard refusal at the gateway — the request never reached the admin application at all. The admin application has no idea this happened. That's a categorically stronger boundary than in-application permission checks, because it means bugs in the application's own authorisation logic are unreachable by an unauthorised user.

### Test 3 — admin, both apps: should succeed

This is the test that failed, and it produced the most instructive part of the whole project.

---

# PART 5 — The bug: two locks on one door

## What was expected

The admin user is in `admins-only`. The admin endpoint's policy requires `admins-only`. Therefore: access granted.

## What happened

**403 Unauthorized.** The same denial said received — but from a user who was in the correct group, against a policy that named exactly that group.

## The debugging, in order

### Step 1 — Establish what the evidence actually shows

The first round of screenshots presented as "admin was denied" turned out, on inspection, to be said's test session. The 403 in them was correct behaviour, not the bug.

**This is worth recording honestly, because it's the most common failure in real incident work and it costs more time than any technical cause.** Debugging the wrong event produces confident conclusions about a problem that isn't there, and every minute spent on it is worse than wasted, because it builds a false model. The discipline: before theorising about a cause, confirm the artefact in front of you is the event you think it is. Check the identity in the screenshot, not the filename.

Once a genuine admin-session denial was confirmed, debugging could actually start.

### Step 2 — Check group membership first, before touching policy

This ordering was pre-committed in the plan's troubleshooting table: *"Access allowed when it should be denied (or vice versa) → check group membership in IAM Identity Center first — policy logic errors are the second most likely cause, not the first."*

**Why membership before policy:** membership is a fact with a single, unambiguous answer that takes ten seconds to verify. Policy logic is an interpretation problem that can absorb an hour. Checking the cheap, decisive thing first is basic triage. Beyond that, membership is empirically the more common cause — policies tend to be written once and carefully, memberships get changed casually by many people over time.

There's a subtlety in *how* it was checked: **the group's member list, not the user's profile page.** These normally agree, but they're rendered by different views, and the authoritative answer for "does this group contain this user" is the group's own membership list. Verified: admin was genuinely a member of `admins-only`.

Membership was correct. Move on.

### Step 3 — The endpoint policy

Read the admin endpoint's Cedar policy directly. It required the `admins-only` group ID. Correct.

At this point the state of knowledge was: user is in the right group, endpoint policy names the right group, and access is denied anyway. Every obvious cause was eliminated. **Which is the actual signal that the model of the system is wrong — not that some detail is wrong.**

When all the pieces you know about are correct and the outcome is still wrong, the error is in the set of pieces you know about. Something is participating in the decision that isn't on your list.

### Step 4 — The missing piece

The relevant behaviour, confirmed against AWS documentation:

> **If a Verified Access group has a policy AND the endpoint has a policy, both must independently allow the request.**

Not "either." Not "the more specific one wins." Both. Evaluated separately, ANDed together.

So the decision path for any request has two gates in series, and passing the second one means nothing if you failed the first.

### Step 5 — Reading the group policy

The group policy existed, and it required group ID:

```
c4f86418-6041-705d-7caa-913073fab34d
```

The `admins-only` group ID is:

```
b4080488-d071-706a-cea5-1ae81ac7065b
```

Different. The group-level policy was requiring **`all-staff`** — written during Phase 6 as a general "must be a real employee" baseline, without appreciating that it would apply to the admin endpoint as a hard prerequisite as well.

## The full explanation

Both endpoints belong to `ZeroTrustLab-Group`. Every request to either application passes through the group policy first.

**said reaching the general app:**
- Group policy: requires `all-staff`. said is in `all-staff`. **Pass.**
- Endpoint policy: requires `all-staff`. **Pass.**
- Both permit → **allowed.** ✅

**said reaching the admin app:**
- Group policy: requires `all-staff`. **Pass.**
- Endpoint policy: requires `admins-only`. said is not. **Fail.**
- Denied. ✅ *(correct, but for only one of the two reasons it might have been)*

**admin reaching the admin app:**
- Group policy: requires `all-staff`. admin is in `admins-only` **only**. **Fail.**
- Endpoint policy: never reaches a permit result that matters — the decision is already no.
- **Denied.** ❌

The admin user was failing at a gate nobody was looking at, one layer before the gate everyone was inspecting. The endpoint policy — the thing under scrutiny for the entire investigation — was correct and was never the problem.

## Why this was genuinely hard to see

Five things conspired:

1. **The console gives no hint.** Group policy and endpoint policy live on separate pages with separate editors. Nothing on the endpoint's policy page indicates another policy also gates this endpoint.
2. **The failure mode is identical.** A group-policy failure and an endpoint-policy failure produce the same `403 Unauthorized`. The response carries no information about which gate refused.
3. **The evidence pointed elsewhere.** said worked perfectly. A system that works for one user and not another looks like a user problem, and attention goes to the user's configuration — which was correct.
4. **Group IDs are opaque.** `c4f86418-...` and `b4080488-...` are not distinguishable at a glance. Had the policies read `all-staff` and `admins-only`, the mismatch would have been visible in one second.
5. **The intention was reasonable.** "Everyone must be an authenticated employee" is a sensible baseline. The mistake wasn't the idea — it was implementing "authenticated employee" as membership in a specific group that not all employees belonged to.

## The fix, and the choice behind it

Two valid options:

**Option A — Add admin to `all-staff` as well.** The group policy becomes a genuine "any authenticated employee" gate. The per-application decision stays where it belongs, in the endpoint policies.

**Option B — Remove the group policy entirely.** Each endpoint's policy becomes the sole gate.

**Option A was chosen.** The reasoning:

- It matches the original intent. `all-staff` was always meant to mean *all staff*. Admins are staff. The group should have contained them from the start; the policy was fine, the membership was incomplete.
- It preserves a defence-in-depth layer. Two independent gates means a mistake in one endpoint policy — say, an over-broad `permit` written during a future change — is still caught by the group-level requirement that the caller be a known employee at all. Removing the group policy removes that backstop.
- It's the cheaper operation. Adding a user to a group is one action with an unambiguous result. Rewriting policy live, on a billing system, invites new mistakes at the worst moment.

**The semantic correction is worth stating explicitly, because it's the real lesson:** `all-staff` should be read as *"is a legitimate employee at all"*, and `admins-only` as *"additionally holds elevated rights."* These are **layered**, not **parallel**. The original membership model treated them as two alternative tiers, which is exactly the assumption the group policy quietly invalidated.

## The second gotcha: stale sessions

Adding admin to `all-staff` was not sufficient on its own.

**An already-issued session carries the group claims it was issued with.** Verified Access received admin's groups from Identity Center at sign-in time and cached them for the session. Changing group membership in Identity Center does nothing to a session that already exists — the session is still carrying the old claim set.

So the fix requires terminating the session: sign out completely, close the browser windows, open a **fresh** window (incognito is cleanest — it guarantees no cookie survives), and authenticate again. Only then does the new membership appear in the context Cedar evaluates.

**Without knowing this, the fix appears to have failed.** You add the group, retest, get the same 403, and conclude the diagnosis was wrong — when it was right and the test was invalid.

This generalises well beyond AWS: **identity changes take effect at the next authentication, not immediately.** It's the same reason revoking someone's access doesn't always kill their active session, which is a genuine and widely-underappreciated operational security concern. "We removed their access" and "they can no longer do anything" are different statements, separated by however long an existing session remains valid.

## Result after the fix

- `admin app url/login as admin.png`, `register MFA asked.png`, `MFA success.png` — fresh authentication with fresh claims
- `admin app url/admin login to admin application.png` — **admin app loads** ✅
- `admin app url/he can also access staff application.png` — **general app loads too** ✅

Both gates now pass for admin on both applications, for the right reasons:

| | Group policy (`all-staff`) | Endpoint policy | Result |
|---|---|---|---|
| said → general | pass | pass (`all-staff`) | ✅ allowed |
| said → admin | pass | **fail** (`admins-only`) | ✅ **denied** |
| admin → general | pass | pass (`all-staff`) | ✅ allowed |
| admin → admin | pass | pass (`admins-only`) | ✅ allowed |

Every cell in that table is now correct **and correct for the intended reason** — which is the distinction between a system that works and a system that happens to produce the right answers.

## What this bug is worth

More than the rest of the build, honestly.

The happy path demonstrates that documentation was followed. This demonstrates: reading a denial correctly, verifying the evidence was the right evidence before theorising, triaging cheap checks before expensive ones, recognising that "everything I can see is correct" means the model is incomplete rather than the details are wrong, finding the actual behaviour in vendor documentation rather than guessing, understanding *why* the console made it invisible, choosing between two valid fixes on articulable grounds, and knowing that the fix wouldn't appear to work without a session reset.

It is also, specifically, a **quiet misconfiguration** — the dangerous kind. It failed closed, which is the safe direction. The mirror image of this mistake — a group policy that's more permissive than intended, silently widening access past what each endpoint policy appears to allow — would have produced no error at all, no failed login, no symptom. It would simply have been wrong, indefinitely, while every individual endpoint policy read correctly to anyone auditing them.

That asymmetry is the reason this is written up at length rather than fixed quietly.

---

# PART 6 — The evidence, and what each piece actually proves

A screenshot with no stated claim is decoration. Each one here exists to prove a specific thing that would otherwise be an assertion.

## Network foundation

| Evidence | Proves |
|---|---|
| `VPC/VPC available.png` | An isolated private network exists with a deliberately chosen, non-colliding address range |
| `Route table/route table.png` | The application subnet has **no** `0.0.0.0/0` route to an Internet Gateway — internet unreachability is structural, not a rule |

## Compute and firewalling

| Evidence | Proves |
|---|---|
| `EC2/apps running and available.png` | Two applications Running with an **empty Public IPv4 column** — no internet-addressable surface |
| `EC2/security group APP-SG inbound rules.png` | Inbound scoped to `10.1.0.0/16` only; no `0.0.0.0/0`, no SSH |
| `EC2/security group APP-SG outbound rules.png` | Outbound documented rather than assumed |
| `EC2/security group APP-SG details page.png` | The SG is bound to the correct VPC |
| `EC2/security group APP-ALB inboud rule.png` | The load balancer accepts HTTPS from inside the VPC only |
| `EC2/security group APP-ALB outbound rule.png` | Complete ALB firewall picture |
| `EC2/security group APP-ALB details page.png` | Correct VPC binding for the ALB security group |

## Load balancing

| Evidence | Proves |
|---|---|
| `load balancer/active load balancer.png` | Both ALBs Active with **Scheme: internal** — no bypass path exists around Verified Access |
| `load balancer/General ALB details page.png` | HTTPS:443 listener with the ACM certificate attached |
| `load balancer/target groups Healthy.png` | The applications were genuinely serving traffic — so any access failure is an access decision, not a broken app |

## Certificate and DNS

| Evidence | Proves |
|---|---|
| `certification verification/digitalplat record added .png` | Domain control was proven via DNS validation records |
| `certification verification/cert issued.png` | ACM status **Issued**, one certificate covering both subdomain SANs |
| `certification verification/two endpoints subdomains.png` | Public DNS points at Verified Access endpoints — **not** at the applications |

## Identity

| Evidence | Proves |
|---|---|
| `IAM/users.png` | Distinct identities with MFA registered |
| `IAM/groups.png` | Group-based access model — the basis every policy references |

## Verified Access

| Evidence | Proves |
|---|---|
| `Verified Access/1 ... trust providers tab.png` | Identity is delegated to IAM Identity Center; authentication and authorisation are separate systems |
| `Verified Access/2 ... group.png` | The group and its baseline policy — the container the bug lived in |
| `Verified Access/3 ... endpoints.png` | Two independent checkpoints, one per application |
| `Verified Access/4 General endpoint details page.png` | Endpoint bound to the internal ALB, domain, and certificate |
| `Verified Access/5 General endpoint policy page.png` | The Cedar policy gating the general app |
| `Verified Access/6 admin endpoint details page.png` | Same construction for the admin app, independently configured |
| `Verified Access/7 admin endpoint policy page.png` | The stricter Cedar policy gating the admin app |

## The live proof

| Evidence | Proves |
|---|---|
| `general app url/check to general url.png` | Starting state |
| `general app url/redirected to login.png` | Unauthenticated requests never reach the application |
| `general app url/login as said.png` | Real identity provider, real authentication |
| `general app url/forced to MFA.png` | MFA **enforced**, not offered |
| `general app url/mfa success.png` | Second factor accepted |
| `general app url/am redirected to application.png` | Full chain works end to end; correct user reaches correct app |
| `general app url/try admin application for said got unauthorized access.png` | **The thesis.** A fully authenticated, MFA-verified user is hard-denied a different application on the same network |
| `admin app url/login as admin.png` | Second identity, independently authenticated |
| `admin app url/register MFA asked.png` | MFA enforced universally, not per-user |
| `admin app url/MFA success.png` | Second factor accepted |
| `admin app url/admin login to admin application.png` | Elevated access works — the system isn't just denying everything |
| `admin app url/he can also access staff application.png` | Layered group model working: admin holds both tiers |

## Logging

| Evidence | Proves |
|---|---|
| `cloudwatch/cloudwatch is watching.png` | Log delivery is configured, not assumed |
| `cloudwatch/access allow.png` | Successful decisions are recorded with identity and timestamp |
| `cloudwatch/access deny.png` | **Denials are recorded** — the higher-value half of the data |

## Why the pairing matters

The four decisive artefacts are:

- said **allowed** to general
- said **denied** to admin
- admin **allowed** to admin
- admin **allowed** to general

An allow on its own proves nothing — a system that allows everything produces it. A deny on its own proves nothing — a broken system produces it. **Only the matrix proves discrimination**: the same system reaching different verdicts for different identities against different resources, each correct. That's the difference between demonstrating a control and demonstrating a coincidence.

---

# PART 7 — Logging

Verified Access was configured to deliver decision logs to CloudWatch Logs during Phase 6, before anything went live.

**Configured before, not after, deliberately.** Logging enabled after testing captures nothing that already happened. Access logs cannot be retroactively generated — the events either were recorded when they occurred or they're gone. In a real incident this is the difference between an investigation and a shrug.

Both test outcomes were located and captured, showing identity, timestamp, target endpoint, and verdict.

**What this closes.** Without logs, every claim in this document is an assertion about a system that no longer exists. With them, the claims are checkable: the denial happened, at a specific time, to a specific identity, against a specific application. That's the difference between "the control works" and "the control worked, here's the record."

**What a production version would add:**

- **Metric filters and alarms** on denial patterns — alert when one identity accumulates denials against a sensitive application in a short window
- **Log retention and archival** to S3 with object lock, so logs survive both accident and deliberate deletion by someone with console access
- **Shipping to a SIEM** for correlation with other sources — a Verified Access denial is much more meaningful next to a simultaneous impossible-travel signal from the identity provider
- **Longer retention**, since CloudWatch Logs defaults to never expiring but is expensive as long-term storage

---

# PART 8 — Cost, and why it shaped the entire build

## What actually bills

| Resource | Rate | Status |
|---|---|---|
| VPC, subnets, route tables, security groups | $0 | Free indefinitely |
| IAM, IAM Identity Center | $0 | Free |
| ACM certificate | $0 | Free, auto-renewing |
| EC2 `t2.micro`/`t3.micro` | Free tier | Free within monthly limits |
| CloudWatch Logs | ~$0 at this volume | Effectively free here |
| **Interface VPC endpoints (×3)** | **~$0.01/hr each** | **Deleted after use** |
| **Application Load Balancers (×2)** | **~$0.0225/hr each** | **Deleted after testing** |
| **Verified Access endpoints (×2)** | **~$0.27/hr each + per-GB** | **Deleted immediately after testing** |

## Why this was treated as an engineering constraint, not an inconvenience

$0.54/hr sounds trivial. Left running for a month it's roughly **$390**. Left running because a project ended and nobody deleted anything, it's $390 every month, indefinitely, for a system nobody is using.

This is the actual shape of real cloud cost incidents. Not deliberate overspend — forgotten resources. A lab built in an afternoon and abandoned. A load balancer left behind after the instances it served were terminated. A NAT Gateway in a VPC nobody opens anymore.

**The discipline that followed from it:**

1. Every free resource built and verified completely first
2. Policies written and syntax-checked **before** the resources they'd be attached to existed
3. Billing resources created last, in one sitting, as a single batch
4. Testing performed immediately, not the next day
5. Teardown executed immediately after evidence capture — not "later today"
6. An independent budget alert running the whole time as a backstop against the discipline failing

**Point 2 is the transferable one.** The goal was for the expensive window to contain only *actions*, never *thinking*. Every decision that could be made in advance was made in advance. When the meter is running, you should be executing a plan, not forming one.

## Teardown order, and why

1. **Verified Access endpoints** — most expensive, first
2. Verified Access instance, group, trust provider
3. **Both ALBs** — second most expensive
4. **The three interface VPC endpoints** — small, and the easiest to forget precisely because they're small
5. EC2 instances — stop or terminate
6. VPC and dependencies, if fully finished

**Ordered by burn rate, not by dependency.** Dependencies would suggest working inward from endpoints anyway, but the principle is worth stating separately: when shutting down a billing system, kill the expensive things first and let the cheap cleanup take as long as it takes.

**Point 4 is the one that catches people.** At $0.01/hr, an interface endpoint is beneath notice — which is exactly why it survives teardown, and why three of them running for a year is $260 nobody can account for. The threshold for "worth deleting" should be "is it still needed," not "is it expensive."

---

# PART 9 — Limitations, and what production would require

Stating these is not a weakness in the project. Claiming a lab is production-grade is the weakness.

## What this build genuinely demonstrates

- Per-application access control with no network-level trust
- Enforced MFA before any authorisation decision
- Group-based authorisation using an immutable-identifier policy language
- Complete absence of any bypass path around the control
- Logged allow and deny decisions
- Correct, provable denial of an authenticated user

## What it does not

**No device trust.** Real Zero Trust evaluates the *device* as well as the identity — is the disk encrypted, is the OS patched, is the EDR agent running, is this a managed device at all. Verified Access supports this through device trust providers (Jamf, CrowdStrike, Intune), and this build used identity only. That's the largest gap between this and a full implementation. A valid identity on a fully compromised laptop passes every check here.

**No continuous re-evaluation within a session.** Policy is evaluated per request, which is already far better than a VPN's once-per-connection. But a session established with valid credentials remains valid for its lifetime — the stale-claims behaviour in Part 5 is the same property viewed from the other side. Production wants short session lifetimes and, ideally, revocation that takes effect immediately rather than at next authentication.

**Single AZ for the applications.** The ALBs span two AZs because AWS forced it; the EC2 instances don't. One AZ failure takes the applications down. Resilience was explicitly out of scope, but the limitation is real.

**No WAF.** Verified Access decides *who may reach* the application. It doesn't inspect *what they send*. An authorised user can still send SQL injection, XSS, or a malicious file upload. AWS WAF on the ALB would address the content layer. Authorisation and content inspection are different problems and need different controls.

**Unencrypted final hop.** TLS terminates at the ALB; ALB→instance is HTTP. Acceptable inside a private VPC with scoped security groups, not acceptable for regulated data or a strict zero-trust-to-the-host posture.

**Manual everything.** Entirely console-built, which was the point — you should understand a system by constructing it before automating it. But a production version belongs in Terraform or CloudFormation, for reviewability, repeatability, and a git history of who changed which policy when. **Notably, the Part 5 bug would have been far more visible in code review**, where the group policy and both endpoint policies sit in one file, next to each other, with the group ID mismatch visible on adjacent lines — rather than spread across three console pages that never appear on screen simultaneously.

**Trivial applications.** Correct for isolating the security boundary under test; not representative of real application behaviour, session handling, or performance.

**Two users, two groups.** Real environments have hundreds of groups with overlapping membership, inherited access, and years of accumulated exceptions. The model here is clean in a way real ones never are.

## What would come next

1. **Infrastructure as Code** — the whole build in Terraform
2. **Device trust** — add a device trust provider and require both identity and posture
3. **AWS WAF** on the ALBs
4. **Alerting** — CloudWatch metric filters on denial patterns feeding real notifications
5. **Multi-AZ applications** — Auto Scaling group across both AZs
6. **End-to-end TLS** — re-encrypt the ALB→instance hop
7. **More granular policy** — time-of-day conditions, request-path conditions, step-up authentication for the most sensitive routes
8. **Break-glass access** — a documented, heavily-logged emergency path for when the identity provider itself fails, because a Zero Trust architecture with a single identity dependency has a single point of total lockout

---

# PART 10 — Glossary

**ACM (AWS Certificate Manager)** — AWS service that issues and auto-renews free TLS certificates.

**ALB (Application Load Balancer)** — Layer 7, HTTP-aware load balancer. Can route on path, host, and headers.

**Availability Zone (AZ)** — A physically separate data centre within an AWS region, with independent power and networking.

**Cedar** — AWS's open-source policy language, used by Verified Access to express authorisation rules.

**CIDR** — Notation like `10.1.0.0/16` describing a block of IP addresses; the number after the slash is how many leading bits are fixed. Smaller number = bigger block.

**CloudWatch Logs** — AWS log storage and search service.

**CNAME** — A DNS record making one name an alias for another name.

**Default-deny** — A design where anything not explicitly permitted is refused. The safe failure direction.

**DNS** — The global system translating names into IP addresses.

**EC2** — AWS virtual machines.

**Health check** — A periodic probe a load balancer sends to each target to decide whether it should receive traffic.

**IAM Identity Center** — AWS's identity provider: users, groups, MFA, and sign-in.

**IAM Role** — An identity a *resource* assumes, as opposed to an IAM User, which a *person* signs in as. Provides temporary, auto-rotating credentials with no stored secret.

**Interface VPC Endpoint** — A private network interface inside your VPC for an AWS service, so traffic to it never crosses the internet.

**Internal (load balancer scheme)** — Only reachable from inside the VPC. The alternative, internet-facing, would have created a bypass around Verified Access.

**Lateral movement** — An attacker expanding from an initial foothold to other systems on the same network. The primary thing Zero Trust is designed to prevent.

**Listener** — The port and protocol a load balancer accepts connections on.

**MFA** — Requiring a second factor beyond a password, typically from a device the user holds.

**NLB (Network Load Balancer)** — Layer 4 load balancer. Fast, protocol-agnostic, cannot see HTTP.

**PDP (Policy Decision Point)** — The component that decides whether access is permitted.

**PEP (Policy Enforcement Point)** — The component sitting in the traffic path that enforces the PDP's decision.

**Private IP range (RFC 1918)** — `10.x`, `172.16–31.x`, `192.168.x`. Not routable on the public internet.

**Route table** — The list of destination→target rules deciding where packets from a subnet are sent.

**SAN (Subject Alternative Name)** — Additional names covered by a single certificate.

**Security group** — A stateful, default-deny firewall attached to a resource's network interface.

**Session Manager** — AWS Systems Manager feature providing IAM-authenticated shell access with no inbound port and no SSH key.

**Stateful firewall** — One that automatically permits return traffic for connections it allowed.

**Subnet** — A subdivision of a VPC's address range, living in exactly one Availability Zone.

**Target group** — The set of servers a load balancer forwards to, plus their health check configuration.

**TLS** — Encryption, integrity, and server authentication for network connections. The `s` in `https`.

**Trust provider** — The identity or device source Verified Access consults for context.

**Verified Access endpoint** — The checkpoint in front of one application.

**Verified Access group** — A container for endpoints that can carry shared policy. **Its policy ANDs with each endpoint's policy.**

**VPC** — Your own isolated virtual network inside AWS.

**Zero Trust** — Never granting trust based on network position; verifying identity and authorisation for every request to every resource independently.

---

# PART 11 — Questions this project should make answerable

If the project was understood rather than followed, these should be answerable without notes.

**Why is there a load balancer when there's only one server behind it?**
Because Verified Access endpoints attach to load balancers, not instances — by design, so that the access control is bound to *the application* as a stable concept rather than to a specific, replaceable virtual machine. Load distribution is irrelevant here; indirection and a stable attachment point are the point.

**Why must the load balancer be internal?**
Because an internet-facing load balancer would be directly reachable from the internet, bypassing Verified Access entirely. Every policy, MFA prompt, and log entry would be skipped by anyone who found its DNS name. The internal scheme is what guarantees Verified Access is the *only* path, and a control that can be gone around is not a control.

**What does the certificate actually do?**
Three things: encrypts the connection, protects it from tampering, and — most importantly — proves to the browser that the server genuinely is the name it claims. Without the third, encryption is worthless because you might be encrypting to an attacker.

**Why groups instead of naming users in policies?**
So access changes are membership changes, not policy changes. Onboarding and offboarding become single actions; "who can reach this?" has one authoritative answer; policies stay stable and reviewable while the volatile part lives separately.

**Why do policies use ugly group IDs instead of readable names?**
Because names are mutable and IDs are not. A rename would silently break every policy referencing the old name — either locking people out or, worse, silently granting access. The cost is readability, and that cost caused the bug in Part 5.

**Why is the denial screenshot more important than the success?**
Because a broken system produces successes too — anything that fails open lets the right user in. Only a correct denial, of a fully authenticated and MFA-verified user, proves the system is actually discriminating rather than just working.

**A user is in the correct group and the endpoint policy names that group, and they're still denied. Where do you look?**
At the Verified Access group's own policy. Group policy and endpoint policy are evaluated independently and ANDed — passing one means nothing if you fail the other, and the console gives no indication on the endpoint page that a second gate exists.

**You fixed a group membership and the user is still denied. Why?**
Their existing session carries the group claims it was issued with. Membership changes take effect at next authentication. Full sign-out and a fresh session are required. This is also why revoking access doesn't necessarily terminate someone's current session — an underappreciated operational security problem in its own right.

**How is this different from a VPN?**
A VPN authenticates once, at the edge, and grants access to a *network*. Everything on that network becomes reachable, and internal access control has to be re-implemented separately by each system. This authenticates identity, then authorises *per application, per request*, with no trust conferred by network position. The proof is the same user being allowed into one application and hard-denied another on the same subnet, in the same session.

**What's the weakest part of this build?**
No device trust. A valid identity on a fully compromised laptop passes every check here. Real Zero Trust evaluates device posture alongside identity, and this build evaluates identity only.

---

## CV bullet

> Designed and deployed a Zero Trust remote access architecture on AWS, replacing VPN-style network-level trust with per-application, per-request authorisation. Built two private applications behind internal Application Load Balancers with no internet-routable path, fronted by AWS Verified Access endpoints enforcing Cedar policies against MFA-verified identities from IAM Identity Center. Proved the control by demonstrating a fully authenticated user being hard-denied a higher-sensitivity application, with both allow and deny decisions verified in CloudWatch Logs. Diagnosed and documented a non-obvious policy-evaluation failure caused by Verified Access group and endpoint policies being independently ANDed. Managed the build under strict hourly-cost discipline, sequencing all free configuration ahead of billed resources and tearing down within the same session.