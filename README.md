# Active Directory Home Lab: Building AD to Understand Attacks

A home lab built from the ground up to learn Active Directory by deploying it myself, then using that environment to see firsthand how a real attack against it looks — from reconnaissance through exploitation to detection.

## Why I Built This

Reading about Active Directory attacks is one thing — actually building a domain, joining a machine to it, attacking it, and watching the logs light up is what made the concepts click. This lab was built to understand:
- How Active Directory actually works under the hood (domains, DNS dependency, authentication)
- How an attacker would realistically approach an AD environment (recon before exploitation)
- What that attack actually looks like from the defender's side in the logs

## Architecture

| Machine | Role | IP | OS |
|---|---|---|---|
| DC01 | Domain Controller | 192.168.10.7 | Windows Server 2022 |
| TARGET | Domain-joined client | 192.168.10.100 | Windows 11 |
| Splunk Server | SIEM / log aggregation | 192.168.10.10 | Ubuntu (Splunk) |
| Kali Linux | Attacker machine | 192.168.10.250 | Kali 2026.2 |

Domain: `Owner.local`
Network: isolated internal network (VirtualBox), subnet `192.168.10.0/24`

## Part 1: Building the Active Directory Environment

- Deployed Windows Server 2022 and promoted it to a Domain Controller, standing up the `Owner.local` domain
- Configured static IPs and DNS so the domain-joined client could actually locate the DC (AD depends entirely on DNS working correctly)
- Joined a Windows 11 client to the domain
- Created test user accounts, including the account later used as the attack target
- Deployed a Splunk server to centralize logging from the domain-joined machines, so authentication activity could actually be observed and analyzed

## Part 2: Attacking It to Understand How It Breaks

Once the domain was working, I attacked it the way a real adversary would — starting with no assumed knowledge of which IP was which machine.

**Reconnaissance:** Used `nmap` from Kali to enumerate the subnet and identify every machine purely through service fingerprinting (Kerberos/LDAP for the DC, RDP for the target, Splunk's own web service for the SIEM).

**Exploitation:** Ran an RDP brute-force attack against the domain-joined client using Hydra, successfully cracking a test account's password.

**Verification:** Manually logged into the target with the cracked credentials via RDP to confirm real interactive access was achieved — not just a validated credential.

Full technical breakdown: [RDP Brute-Force Investigation](write-ups/rdp-bruteforce-investigation.md)

## Part 3: Watching It in Splunk

With logging in place from Part 1, the attack in Part 2 was fully visible and traceable in Splunk:
- Failed login flood (Event ID 4625) from the attacker's IP
- Successful credential validation (Event ID 4624, Logon_Type 3)
- Confirmed interactive session (Event ID 4624, Logon_Type 10) tying the attacker's IP directly to a full domain logon

## What This Taught Me

- Active Directory's moving parts (DNS, domain join, authentication flow) make a lot more sense after actually building and breaking one
- Attackers don't need prior knowledge of a network — service fingerprinting alone reveals what's what
- Logs don't always tell the whole story on the first read — an initial theory about a missing log event turned out to be wrong once tested directly, which was a good lesson in verifying rather than assuming

## Write-ups

- [RDP Brute-Force Investigation](write-ups/rdp-bruteforce-investigation.md)

## Screenshots

See `/screenshots` for the full evidence trail: network recon, attack execution, and Splunk log correlation.
