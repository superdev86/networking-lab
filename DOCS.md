# AWS Networking Lab — Incident Runbook & Resolution Notes

This document consolidates the resolution steps, reasoning, and key learnings from multiple incidents involving AWS VPC networking, DNS, security groups, and service-to-service communication.

---

# INC-4521 — Private Subnet Egress Failure (API Cannot Reach Internet)

## Summary

API service in a private subnet could not reach external services. Internal service-to-service communication worked, but all outbound internet requests (e.g., HTTPS) timed out.

## Symptoms


```curl localhost:8080``` ✔ works  
```dig google.com``` ✔ works (DNS OK)  
```curl google.com``` ❌ hangs / times out


## Investigation

### Security Group Check


```aws ec2 describe-security-groups```


- Outbound HTTP/HTTPS allowed
- SG ruled out

### Route Table Check


```aws ec2 describe-route-tables```


Missing route:


0.0.0.0/0 → NAT Gateway


## Root Cause

Private subnet had no route to NAT Gateway, so no internet egress path existed.

## Fix

```
aws ec2 create-route
--route-table-id rtb-xxxx
--destination-cidr-block 0.0.0.0/0
--nat-gateway-id nat-xxxx
```

## Key Learnings

- SG ≠ routing
- NAT Gateway required for private subnet internet access
- DNS working does NOT imply internet access
- Timeouts typically indicate routing/NAT issues

---

# INC-4522 — Internal Service Discovery Failure (Route 53)

## Summary

Internal hostnames (`web.internal.local`, `api.internal.local`, `db.internal.local`) stopped resolving.

## Symptoms


dig web.internal.local → SERVFAIL
dig google.com → OK


## Investigation

### DNS Resolver Check


resolvectl status


- VPC resolver (10.0.0.2) working
- OS missing internal domain association

### Route 53 Verification

Missing or incorrect A records initially identified.

## Fix

Added correct A records:

- web.internal.local → 10.0.2.x
- api.internal.local → 10.0.2.x
- db.internal.local  → 10.0.3.x

Validation:


dig api.internal.local → OK


## Key Learnings

- Route 53 misconfig can mimic network outage
- DNS resolution ≠ correct mapping
- Always validate against real instance IPs

---

# INC-4523 — Web → API → DB Connectivity Failure

## Summary

Service-to-service communication failed across tiers:

- Web → API (8080)
- API → DB (5432)

## Symptoms

- API works locally
- DB works locally
- Cross-service calls fail/time out

## Investigation

### Security Groups

- Web → API allowed (8080)
- API → DB allowed (5432)
- SG-to-SG references correct

✔ SGs ruled out

---

### NACL Issue (Root Cause)

Database subnet NACL contained:


DENY TCP 5432 inbound


## Fix

```
aws ec2 create-network-acl-entry
--network-acl-id acl-xxxx
--ingress
--rule-number 50
--protocol tcp
--port-range From=5432,To=5432
--cidr-block 0.0.0.0/0
--rule-action allow
```


---

### DNS Issue (systemd-resolved)


```curl api.internal.local``` → could not resolve host


Fix:


```sudo resolvectl domain ens5 internal.local```


This registered the domain with systemd-resolved.

## Validation


```resolvectl query api.internal.local``` → OK
```nc -zv db.internal.local 5432``` → OK
```curl api.internal.local:8080``` → OK


## Key Learnings

- NACLs are stateless → rule order matters
- SGs are stateful → easier to reason about
- systemd-resolved controls OS-level DNS behavior
- Route 53 alone is not enough for resolution

---

# INC-4524 — Security Group Hardening (Compliance Fix)

## Summary

Security audit identified overly permissive network access.

## Issues

- SSH open to internet on all hosts
- DB accessible too broadly on 5432
- ICMP open on web tier from anywhere

## Fixes

### SSH Hardening

- Bastion: SSH allowed only from trusted IP
- Web/API/DB: SSH allowed only from bastion SG

### DB Restriction

- Only API SG allowed on 5432

### ICMP Restriction

- Web ICMP restricted to bastion SG only

## Key Learnings

- Prefer SG-to-SG rules over CIDR-based trust
- Bastion should be sole entry point
- Least privilege applies at network layer too

---

# Core Networking Concepts Learned

## DNS is layered

- Route 53 = authoritative mapping
- VPC DNS (10.0.0.2) = resolver
- systemd-resolved = OS glue layer

## SG vs NACL vs Route Tables

- Security Group = permission
- NACL = subnet filtering (stateless)
- Route Table = traffic path

## Tooling

- dig → DNS resolution (queries Route 53 / VPC DNS directly)
- resolvectl → OS-level DNS resolution (systemd-resolved state + domains)
- resolvectl query → direct DNS query through systemd-resolved path
- curl → HTTP/HTTPS application-layer connectivity testing
- nc (netcat) → raw TCP port connectivity testing (L4 handshake validation)
- ss → socket inspection (what processes are listening on which ports)
- ip route → local routing table inspection (Linux kernel routing view)
- ping → ICMP reachability test (basic network path validation)
- aws ec2 → AWS infrastructure inspection (SGs, NACLs, routes, VPC config)
- aws route53 → DNS hosted zone and record management

