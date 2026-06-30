# AWS FedRAMP-Aligned Multi-Tier VPC Lab

> A fully functional, security-hardened, three-tier cloud infrastructure deployed via modular AWS CloudFormation templates, engineered to align with **FedRAMP Rev. 5 Low Baseline** controls and documented with a full System Security Plan (SSP), control narratives, and open POA&M items. End-to-end traffic flow was verified live — WAF blocking, ALB proxy routing, and private backend responses all confirmed.

---

## System Identification

| Field | Value |
|---|---|
| **Author** | Chris George Paikayil |
| **Status** | Implemented (CloudFormation Baseline) |
| **Service Model** | Infrastructure as Code (IaC) — network baseline |
| **Region** | US East (Ohio) — `us-east-2` |
| **VPC CIDR** | `10.0.0.0/16` |
| **FedRAMP Baseline** | Rev. 5 Low |

**Authorization Boundary:** The environment boundary is restricted entirely to the AWS Virtual Private Cloud (VPC) using the `10.0.0.0/16` CIDR block. All compute resources are isolated inside private subnets with no direct internet exposure.

---

## Overview

This lab is a **fully deployed and verified** AWS network architecture built from scratch using infrastructure-as-code. It is not a static config exercise — the ALB resolved to a live DNS endpoint, the WAF actively inspected and blocked real HTTP requests, and end-to-end round-trip traffic was confirmed from the public internet through to a private backend EC2 instance with no public IP.

Every design decision maps to a specific NIST 800-53 / FedRAMP control, and the deployment is documented with an SSP, control narratives, and two open POA&M items reflecting real residual risk.

**What "dormant" means here:** The Internet Gateway is pre-provisioned but not wired into any subnet route table — meaning no instance is directly reachable via a public IP. This is intentional. All live traffic flows exclusively through the ALB perimeter. The IGW exists as a dormant infrastructure artifact, not because nothing works.

---

## Architecture

```
Internet
    │
    ▼ HTTPS :443 only
[IGW] ──► Public Subnet (10.0.1.0/28)
    │         ALB (HTTPS :443, TLS 1.2/1.3 enforced)
    │         WAF Web ACL (AWSManagedRulesCommonRuleSet)
    │         NACL-1: inbound 443 / outbound ephemeral 1024–65535
    │
    ▼ :8080 (web-to-app link)
Private App Subnet (10.0.2.0/28)
    │         EC2 t2.micro — no public IP
    │         Python HTTP daemon (UserData bootstrap)
    │         NACL-2: inbound from 10.0.1.0/28 only
    │
    ▼ :3306 (app-to-DB link)
Private DB Subnet (10.0.3.0/28)
              Air-gapped — SG egress locked to []
              NACL-3: inbound from 10.0.2.0/28 only

All other traffic: DENY (explicit)

Public IP auto-assign: DISABLED on all subnets
IGW routing: PRE-PROVISIONED / DORMANT — no instance is directly reachable; all live traffic routes through ALB only
ALB DNS: LIVE — resolves to a real endpoint and served verified HTTP responses during testing
```

---

## Authorized Services, Ports & Protocols

| Traffic Path | Port | Protocol | Justification |
|---|---|---|---|
| Inbound HTTPS | 443 | TCP | Allows public encrypted web traffic from the internet to the web tier |
| NACL-1 Outbound Egress | 1024–65535 | TCP | Standard ephemeral port range for return traffic; does not authorize web→app forwarding |
| Web-to-App Link | 8080 | TCP | Allows public web tier to forward filtered requests across the boundary to private app tier |
| App-to-DB Link | 3306 | TCP | Allows private app tier to send DB queries across the isolated boundary |
| All other traffic | ANY | ANY | **Explicitly denied** |

---

## Stack Components

| Template | Resource | Notes |
|---|---|---|
| `vpc.json` | VPC | `10.0.0.0/16`, DNS enabled |
| `subnet.json` | 3 subnets | Public web, private app, private DB |
| `igw.json` | Internet Gateway | Pre-provisioned; dormant — not attached to any subnet route table |
| `routetable.json` | Route Tables | Explicit `DependsOn` constraints to prevent race conditions |
| `nacl1-pubweb.json` | NACL – Public Web | Inbound 443 only; outbound ephemeral ports (1024–65535); explicit deny-all |
| `nacl2-privapp.json` | NACL – Private App | Inbound from `10.0.1.0/28` only |
| `nacl3-db.json` | NACL – DB | Inbound from `10.0.2.0/28` only; fully air-gapped |
| `sgec2.json` | Security Groups | SG-to-SG ID references on DB tier — no hardcoded CIDRs |
| `alb.json` | Application Load Balancer | HTTPS :443, `ELBSecurityPolicy-TLS13-1-2-Resilient-2024-03`, ACM cert |
| `waf.json` | WAF Web ACL | `AWSManagedRulesCommonRuleSet`; associated to ALB; XSS/SQLi blocking verified |
| `ec2.json` | EC2 Instance | Private subnet, UserData HTTP daemon, no public IP |

---

## NIST SP 800-53 Control Implementation Statements

### SC-7 — Boundary Protection
**Status:** Implemented via IaC | **Role:** Cloud Security Engineer

The system is strictly bound within the AWS VPC structure. All subnets are isolated from the public internet. A dual-layer stateless firewall (NACLs) is attached to each network segment, filtering all inbound and outbound traffic before it reaches any compute resource.

---

### SC-7(3) — Access Points
**Status:** Pre-provisioned / Dormant | **Role:** Cloud Security Engineer

An Internet Gateway has been defined in the configuration baseline (`igw.json`) but is currently dormant — it is not attached to any internal subnet route table and has zero active external exposure. The public routing path exists as infrastructure-as-code but carries no live traffic.

---

### AC-3 — Access Enforcement
**Status:** Implemented via IaC | **Role:** Cloud Security Engineer

The system enforces approved authorizations for logical access to network enclaves and resources. NACLs are enforced at the network layer (Layer 3) with explicit rules governing communication between all network subjects, preventing any unauthorized lateral movement.

---

### AC-4 — Information Flow Enforcement
**Status:** Implemented via IaC | **Role:** Cloud Security Engineer

Information flow is strictly regulated across tiered subnets. Routing policies enforce a unidirectional pathway: Web → App → DB. No reverse or cross-tier flows are authorized. This is directly aligned with SC-7 boundary protection controls.

---

### AC-6 — Least Privilege
**Status:** Implemented via IaC | **Role:** Cloud Security Engineer / Identity Admin

Least privilege is enforced across both the network and identity layers. At the network layer, NACLs and security groups use granular CIDR blocks scoped to specific subnet ranges rather than VPC-wide ranges. At the identity layer, all engineering and deployment operations are restricted to a scoped, non-root sandbox admin account — eliminating unnecessary system privileges.

---

### IA-2 — Identification and Authentication
**Status:** Implemented | **Role:** Identity Admin

The system uniquely identifies and authenticates all users performing engineering or deployment tasks. All CLI operations and CloudFormation deployments are tied to a specific non-root IAM user (`sandbox-admin`), ensuring every action is explicitly associated with a valid, tracked identity.

---

### IA-2(1) — MFA to Privileged Accounts
**Status:** Implemented | **Role:** Identity Admin

MFA is enforced on all privileged accounts. Any user with escalated privileges — including the root user — must successfully authenticate with an authenticator app before being authorized to access the account.

---

### CM-2 — Baseline Configuration
**Status:** Implemented via IaC | **Role:** Cloud Security Engineer / DevOps Lead

The system's baseline configuration is formally established, documented, and maintained via infrastructure-as-code. All components — VPC, IGW, three subnets, and per-subnet NACLs — are explicitly defined in modular JSON CloudFormation templates. These serve as immutable, repeatable configuration artifacts across development, test, and production environments.

---

### SI-4 — Information System Monitoring
**Status:** Implemented | **Role:** Cloud Security Engineer / DevSecOps Engineer

An AWS WAF Web ACL is positioned in front of the public-facing ALB to monitor and filter all inbound HTTP/HTTPS requests. The Web ACL is configured with the vendor-managed `AWSManagedRulesCommonRuleSet`, which inspects request headers, query strings, and body content for patterns associated with web application attacks including XSS and SQLi. Enforcement was validated by submitting a `<script>` test payload through the public URL, which returned a **403 Forbidden** response at the perimeter.

---

## Live Verification

This infrastructure was fully deployed and tested. The following outcomes were confirmed against a live environment:

**1. End-to-end proxy routing**

A real HTTP request was sent to the ALB's public DNS endpoint and successfully routed through the full stack — ALB → private app subnet → EC2 Python daemon — returning a valid response. The EC2 instance has no public IP and is unreachable by any path other than the ALB.

**2. WAF XSS blocking**

```
GET /?q=<script>alert(1)</script> HTTP/1.1
Host: [ALB DNS]

→ 403 Forbidden  ✓  (dropped at perimeter by AWSManagedRulesCommonRuleSet)
```

**3. Backend HTTP daemon**

The EC2 instance bootstrapped a Python HTTP server via UserData on startup, confirmed live by ALB health checks passing and a valid HTTP response returned through the proxy chain.

**4. ALB health checks**

ALB target group health checks passed against the private EC2 backend, confirming the SG-to-SG rules, NACL flows, and routing tables were all correctly configured end-to-end.

---

## Plan of Action & Milestones (POA&M)

### Item 1 — Outbound Patch Management & Dependency Updates
**Controls:** SC-7 (Boundary Protection) / SI-2 (Flaw Remediation)  
**Scheduled Completion:** 30 days from baseline authorization

**Deficiency:** The private application and database tiers are configured with restrictive network boundaries that block outbound access to external update repositories. OS and application patching cannot be performed through standard internet-based mechanisms.

**Remediation:** Provision a managed AWS NAT Gateway in the public tier and update private route tables to allow controlled, one-way outbound internet access for patching and dependency retrieval — while preserving full inbound isolation.

---

### Item 2 — Secure Remote Command-Line Access
**Controls:** AC-17 (Remote Access) / AC-6 (Least Privilege)  
**Scheduled Completion:** 60 days from baseline authorization

**Deficiency:** The environment currently lacks a secure remote administration channel for command-line access to compute instances. Traditional inbound SSH/RDP is not enabled, which limits troubleshooting, log review, and operational maintenance.

**Remediation:** Implement AWS Systems Manager Session Manager for all remote administration. This provides IAM-authorized, encrypted CLI access through the AWS Console or API — with full audit logging and no requirement for public IP addresses or bastion host infrastructure.

---

## Key Engineering Challenges Resolved

**NACL stateless logic** — Initial NACL only defined inbound rules with no outbound, rendering it completely ineffective. Added explicit outbound ephemeral port range (`1024–65535`) and deny-all rules to enforce true stateless filtering.

**CloudFormation race condition** — Route table deployment hit an unrecoverable race condition against VPC and IGW initialization. Resolved with explicit `DependsOn` constraints and a staged subnet deployment split.

**SG egress schema error** — App server egress rule incorrectly used `SourceSecurityGroupId` instead of `DestinationSecurityGroupId`. CloudFormation would have rejected and rolled back the entire stack. Caught and corrected pre-deployment.

**ALB/NACL port mismatch** — ALB listener was on port 80 while NACLs only permitted 443 inbound. Aligned listener to HTTPS 443 with ACM cert and enforced `ELBSecurityPolicy-TLS13-1-2-Resilient-2024-03` to prevent legacy TLS 1.0/1.1 ciphers.

**FedRAMP metadata tag mismatch** — Route table tags referenced "FedRAMP-Moderate" while the SSP documents a Low baseline. Corrected tags so IaC metadata exactly matches SSP documentation for a clean audit trail.

**EC2 UserData pathing** — Python web daemon was failing silently due to unscoped working directory. Added `mkdir -p /var/www/html && cd /var/www/html` before daemon startup to anchor the server to the correct path.

**Ghost stack clearances** — Manually tracked and executed `delete-stack` commands via CLI to clear `ROLLBACK_COMPLETE` resource names from account state before re-runs.

---

## Cost

**$0** — All resources are within AWS Free Tier or kept dormant. No NAT Gateway deployed. Public IP auto-assignment disabled on all subnets.

---

## Tools & Services

`AWS CloudFormation` · `EC2` · `VPC` · `ALB` · `WAF` · `ACM` · `IAM` · `AWS CLI`

---

## Author

**Chris George Paikayil** · Cybersecurity Student, DePaul University (CAE-CD)  
CompTIA Security+ (SY0-701) · Targeting GRC / Cloud Security roles  
[LinkedIn](#) · [GitHub](#)
