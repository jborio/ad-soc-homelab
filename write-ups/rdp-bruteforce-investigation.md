# RDP Brute-Force Attack: Reconnaissance, Exploitation, and Investigation

## Scenario

**Attacker:** Kali Linux VM (192.168.10.250)
**Target:** Windows 11 domain-joined client, hostname `TARGET` (192.168.10.100), domain `Owner.local`
**Attack tool:** Hydra
**Detection tool:** Splunk

## 1. Reconnaissance

Rather than assuming the target's IP, I used nmap from Kali to identify every machine on the subnet through service fingerprinting alone.

```
sudo nmap -A 192.168.10.0/24
```

This single scan identified every machine's role without needing to check them manually:

| IP | Identified As | Evidence |
|---|---|---|
| 192.168.10.7 | Domain Controller (DC01) | Kerberos (88), LDAP (389/3268), NetBIOS name `DC01`, domain `Owner.local` |
| 192.168.10.10 | Splunk Server | Splunkd httpd (8000/8089), SSL cert `SplunkServerDefaultCert` |
| 192.168.10.100 | Windows 11 target | Port 3389 open, NetBIOS name `TARGET`, domain `Owner.local` |

## 2. Attack Execution

With the target confirmed, I ran a brute-force attack against RDP using Hydra:

```
hydra -l klojano -P passwords.txt rdp://192.168.10.100/OWNER -t 1 -V
```

**Result:** Password found — `Password@123`

### Troubleshooting note: why the first attempt failed

My initial attempt used the default multi-threaded settings without specifying the domain:
```
hydra -l klojano -P passwords.txt rdp://192.168.10.100
```
This returned an inconclusive result:
```
account might be valid but account not active for remote desktop: login: klojano password: Password@123
```
alongside several `[ERROR] freerdp: The connection failed to establish` messages.

**Root cause:** Two issues compounded:
1. No domain was specified in the target URL, so Windows NLA could not fully validate the domain account context.
2. Running 4 parallel connections (`-t 4`, the default) overwhelmed the RDP service, which handles concurrent handshake negotiations poorly — a limitation Hydra itself warns about.

**Fix:** Specifying the domain explicitly (`rdp://192.168.10.100/OWNER`) and reducing to a single thread (`-t 1`) allowed the RDP handshake to complete cleanly, confirming the password on the next run.

## 3. Detection and Investigation in Splunk

**Step 1 — Failed logon attempts:**
```
index=* EventCode=4625
| table _time, src_ip, Account_Name
```
Showed a flood of failed attempts against `klojano` originating from `192.168.10.250` (the Kali VM), matching the Hydra attack window.

**Step 2 — Successful credential validation:**
```
index=* EventCode=4624
| table _time, src_ip, Account_Name, Logon_Type
```
Found a successful logon event for `klojano` at `Logon_Type=3` (Network logon), Source Network Address `192.168.10.250` — confirming the brute-forced credential was valid and accepted by the domain.

**Step 3 — Checking for a true interactive session:**

Initially, no `Logon_Type=10` (RemoteInteractive) events appeared anywhere in the environment. This raised a hypothesis: that `klojano` might lack Remote Desktop logon rights, and that the credential was valid but couldn't be used for an actual RDP session.

**This hypothesis was tested directly rather than assumed.** Hydra's brute-force module only performs the NLA credential-validation handshake — it does not open a full interactive desktop session by design. So the absence of a `Logon_Type=10` event was not evidence of a permissions restriction; it simply reflected how the attack tool operates.

To test this properly, I manually connected to the target using the cracked credentials:
```
xfreerdp /v:192.168.10.100 /u:klojano /p:'Password@123' /d:OWNER
```
**Result:** The RDP session connected successfully.

**Step 4 — Confirming in Splunk:**
```
index=* EventCode=4624 Logon_Type=10
```
This returned a matching event, with the `New Logon` section confirming:
```
Account Name:    klojano
Account Domain:  OWNER
```
and Source Network Address `192.168.10.250` — completing the evidence chain from credential compromise to full interactive access.

## 4. Findings Summary

1. Every machine on the network was identified purely through service fingerprinting, without prior knowledge of IP assignments.
2. The `klojano` account's password was compromised via brute-force in ~2.5 minutes using a small custom wordlist.
3. An initial brute-force attempt returned an inconclusive result due to a missing domain specification and RDP's poor tolerance for parallel connections — both were diagnosed and corrected.
4. The compromised credential was confirmed valid at the network authentication layer (`Logon_Type=3`) before a full interactive session was established.
5. Rather than accepting an initial (incorrect) theory that the account lacked RDP permissions, the theory was tested directly — a manual RDP login succeeded, correcting the conclusion and confirming full interactive compromise (`Logon_Type=10`).

## 5. Recommended Response (as a SOC analyst would report it)

- Disable or force a password reset on the `klojano` account
- Block the source IP (`192.168.10.250`) at the network boundary
- Enforce an account lockout policy after a defined number of failed attempts
- Enforce MFA for RDP access, or restrict RDP exposure to a VPN/jump host rather than direct network access
- Review Group Policy for accounts with Remote Desktop rights to ensure least-privilege access

## Splunk Queries Used

```
index=* EventCode=4625 | table _time, src_ip, Account_Name
index=* EventCode=4624 | table _time, src_ip, Account_Name, Logon_Type
index=* EventCode=4624 Logon_Type=10
```
