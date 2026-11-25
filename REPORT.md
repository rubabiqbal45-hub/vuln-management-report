# Vulnerability Management Report — SOC Lab (DC01 & soc-victim)

**Scope:** 192.168.56.10 (DC01, Windows Server 2022, Domain Controller) and 192.168.56.12 (soc-victim, Windows Server, domain-joined workstation)
**Date:** 2026-11-25
**Tools used:** Nmap (service/version detection, `--script vuln`, targeted SMB scripts, `vulners` CVE cross-reference)

## 1. Why this report is structured this way

A scanner can produce a long list of findings. What actually matters to a business is which of those findings are realistically exploitable, what the impact would be if they were exploited, and therefore which ones need fixing first. This report ranks findings by that logic — exploitability and business impact — rather than by raw scanner severity alone, and ends with a remediation plan a non-technical manager could act on directly.

## 2. Methodology

1. **Service and vulnerability discovery:** `nmap -Pn -sV -O --script vuln` against both hosts. `-Pn` was required because Windows Firewall silently drops inbound ICMP by default — this was verified as a non-issue (both hosts could reach each other and the host machine fine) before scanning, not assumed.
2. **Follow-up on inconclusive results:** the built-in SMB vulnerability scripts (`ms10-054`, `ms10-061`, `samba-cve-2012-1182`) returned "could not negotiate a connection" against DC01 — inconclusive, not a clean result. A second, targeted scan (`smb-protocols`, `smb2-security-mode`) was run instead of treating silence as safety, and returned a clear, positive result.
3. **CVE cross-referencing attempt:** Nmap's `vulners` NSE script was run to match detected service versions against the CVE database directly. It returned no matches — not a failure, but a real finding in itself: the Windows AD services exposed here (Kerberos, LDAP, netbios, generic `microsoft-ds?`/`kpasswd5?` banners) don't expose a specific enough version string for automated CPE-based CVE matching. This is where the report moves from automated tooling to direct exploitability/impact reasoning — the actual point of this exercise.
4. **Context from prior lab work:** DC01's Kerberos service here is the same domain controller used in the SOC Detection Lab (Project 1), where a live Kerberoasting attack against a deliberately weak service account was demonstrated and confirmed via weak RC4 ticket encryption in Event ID 4769. That confirmed exploitability is folded into the risk ranking below rather than ignored.

## 3. Risk-ranked findings

| # | Finding | Host(s) | Exploitability | Impact | Priority |
|---|---|---|---|---|---|
| 1 | Kerberos ticket encryption downgradable to RC4 on a service account (Kerberoastable) | DC01 | **High** — already demonstrated exploitable in this lab: any domain foothold can request a ticket and crack it offline | **High** — recovers a service account credential, a direct path to privilege escalation | **P1 — fix immediately** |
| 2 | RDP (3389) exposed, no confirmed NLA/MFA | soc-victim | **High** — one of the most common real-world initial-access vectors (brute force, credential stuffing, credential replay) | **High** — full interactive host compromise | **P1 — fix this week** |
| 3 | WinRM reachable over HTTP, not HTTPS (5985) | Both hosts | **Medium** — requires an on-path attacker to intercept traffic | **High** — a successful capture yields remote command execution | **P2 — fix this week** |
| 4 | Full AD/DC service set reachable (LDAP, Kerberos, DNS, RPC, netbios) | DC01 | **Context-dependent** — necessary for domain function; risk is in *where* it's reachable from, not that it exists | **High if internet-facing, expected if internal-only** | **P2 — scope and monitor, not a patch** |
| 5 | SMBv1 absent; SMB signing enabled and required | DC01 | N/A — confirmed control, not a vulnerability | N/A | **Verified control — no action needed** |

## 4. Remediation plan

**Fix immediately (this is the priority):**
- Enforce AES-only Kerberos ticket encryption on the affected service account, or migrate it to a Group Managed Service Account (gMSA), closing the credential-theft path already proven exploitable in this environment.

**Fix this week:**
- Require Network Level Authentication (NLA) on RDP and restrict access to VPN-only or add MFA before any production exposure of soc-victim.
- Reconfigure WinRM on both hosts to use HTTPS (5986) with a valid certificate instead of HTTP (5985).

**Ongoing control, not a one-time fix:**
- Confirm DC01's AD/DC service set (LDAP/Kerberos/DNS/RPC) is firewalled from anything outside the internal network or VPN boundary. This is a network segmentation review, to be re-checked periodically, not a patch to apply once.

**No action required:**
- SMB configuration (SMBv1 disabled, signing required) is already correctly hardened. Re-verify on future scans rather than re-remediate.

## 5. What was deliberately left out of the "fix now" list, and why

The scan also confirmed a clean SMB posture and a correctly functioning AD service set. Neither is listed as something to "fix" — including working controls in a remediation list as if they were problems would waste a manager's time and dilute the findings that actually need action. A vulnerability management report earns trust by being as clear about what's already fine as it is about what isn't.
