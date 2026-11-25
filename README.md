# Vulnerability Management Report

A real vulnerability scan of my own Active Directory lab (the same DC01 + soc-victim environment from the SOC Detection Lab project), risk-ranked by actual exploitability and business impact rather than raw scanner severity, with a prioritized remediation plan a manager could act on directly.

This is the 5th project in my SOC/security portfolio. Anyone can run a scanner and hand over a list of findings — companies pay analysts for the judgment to look at that list and say which five need fixing before Monday, which can wait, and why. That judgment call is the actual point of this project, not the scan itself.

## What was scanned

- **DC01** (192.168.56.10) — Windows Server 2022, Domain Controller, soclab.local
- **soc-victim** (192.168.56.12) — Windows Server, domain-joined workstation

Both are the same hosts built for the SOC Detection Lab (Project 1), reused here for a different kind of assessment.

## Tools and approach

1. **Nmap** service/version detection, OS fingerprinting, and built-in vulnerability scripts (`-sV -O --script vuln`)
2. A targeted follow-up scan (`smb-protocols`, `smb2-security-mode`) after the built-in SMB vulnerability scripts returned an inconclusive result — rather than treating silence as a clean bill of health
3. **Nmap's `vulners` NSE script**, to attempt automated CVE cross-referencing directly against detected service versions — originally planned as a second scanner (Nessus Essentials), which was dropped after it fought a multi-hour installation and backend-initialization problem that never resolved (documented in full in the project walkthrough)
4. Manual exploitability/impact analysis once automated CVE matching came back empty — explained below

## Why there's no CVE list

`vulners` needs a precise product-and-version string to match against the CVE database. The Windows AD services this lab exposes (Kerberos, LDAP, netbios, generic `microsoft-ds?`/`kpasswd5?` banners) were correctly identified by Nmap but don't expose that level of version detail — itself a small positive finding, not a dead end. The report moves past automated tooling at that point and reasons directly about what's exposed, how exploitable it realistically is, and what it would cost if exploited. That reasoning — not a CVE feed — is the actual deliverable.

## Risk-ranked findings (summary — full detail in [`REPORT.md`](REPORT.md))

| # | Finding | Priority |
|---|---|---|
| 1 | Kerberos ticket encryption downgradable to RC4 on a service account (Kerberoastable) | P1 — fix immediately |
| 2 | RDP exposed with no confirmed NLA/MFA | P1 — fix this week |
| 3 | WinRM reachable over HTTP, not HTTPS | P2 — fix this week |
| 4 | Full AD/DC service set reachable | P2 — scope/monitor, not a patch |
| 5 | SMBv1 absent, SMB signing required | Verified control — no action needed |

## Evidence (screenshots)

| File | What it shows |
|---|---|
| [`00-connectivity-diagnosis.png`](screenshots/00-connectivity-diagnosis.png) | ICMP-blocked-by-firewall diagnosis (ping failure traced to Windows Firewall, not a real network issue) |
| [`01a-nmap-service-vuln-scan-part1.png`](screenshots/01a-nmap-service-vuln-scan-part1.png) / [`01b-...-part2.png`](screenshots/01b-nmap-service-vuln-scan-part2.png) | Initial `-sV -O --script vuln` scan against both hosts, including the inconclusive SMB vulnerability-script results |
| [`02-smb-signing-followup.png`](screenshots/02-smb-signing-followup.png) | Targeted SMB-protocol follow-up scan confirming no SMBv1, signing required |
| [`03-vulners-scan.png`](screenshots/03-vulners-scan.png) | `vulners` NSE scan output |

## Skills demonstrated

- Vulnerability scanning methodology: service/version detection, OS fingerprinting, targeted follow-up on inconclusive results
- Risk-ranking by real exploitability and business impact, not raw scanner severity
- Writing a remediation plan a non-technical stakeholder can act on, including explicitly documenting what's already secure rather than padding the findings list
- Troubleshooting under constraints and adapting the approach (Nessus install/backend failure after hours of attempts, pivoting to an NSE-script-based alternative) rather than stalling the project
- Recognizing the limits of automated tooling (CPE-based CVE matching) and reasoning manually where it falls short

## Running it yourself

```powershell
nmap -Pn -sV -O --script vuln -oN nmap-scan-results.txt <target1> <target2>
nmap -p445 --script smb-protocols,smb2-security-mode <dc-target>
nmap -Pn -sV --script vulners -oN vulners-scan-results.txt <target1> <target2>
```

`vulners.nse` is a community NSE script, not bundled with Nmap — download it from [vulnersCom/nmap-vulners](https://github.com/vulnersCom/nmap-vulners), place it in Nmap's `scripts/` folder, and run `nmap --script-updatedb` before use.
