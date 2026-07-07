# Security Audit — EscaleiraP/Anthropic-Cybersecurity-Skills (fork)

Target: `~/cybersecurity-skills-src` = clone of `EscaleiraP/Anthropic-Cybersecurity-Skills`
Upstream: `mukul975/Anthropic-Cybersecurity-Skills` @ HEAD `673da1f3` (v1.3.0)
Plugin: `cybersecurity-skills` v1.3.0 — author **mukul975**, Apache-2.0 — **817 skills**
Date: 2026-07-07
Mode: Repo scan (mechanical + LLM deep-dive)
Threat model: does this attack YOU — your host, credentials, processes? (NOT a code-quality review)
Scope note: baseline `04450304` was audited SAFE on 2026-06-11. This audit re-covers the full
tree and gives priority to the **26 new commits / +44,866 lines / 55+ new skills** added since,
including third-party PRs (andrewibrah, DevRedious, nyxst4ck, ioxoi, Homan Ansari, Shanujan Suresh).

## Phase 1 — Mechanical Scan

| Scan | Result |
|------|--------|
| 01 Dependencies | clean — no vulnerable deps |
| 02 Build scripts | clean — no dangerous shell patterns |
| 03 Code patterns | CRITICAL/HIGH hits = **all false positives** (see below) |
| 04 Binaries | clean — no compiled/hidden executables |
| 05 CI/CD & Docker | clean — no workflow threats |

The 03 "CRITICAL exfiltration/reverse-shell" hits (`/etc/shadow`, `Login Data`, `meterpreter`,
`cobaltstrike`, `xmrig`, `stratum+tcp`, `pool.minexmr`…) are **attack signatures, forensic target
paths, and named-tool references living inside security tooling** that hunts or analyzes them —
not operations against this host. Same class the baseline audit cleared.

## Phase 2 — LLM Deep-Dive (structural threat vectors)

- **[SAFE] No auto-execution.** `.claude-plugin/plugin.json` declares **zero hooks**. No
  `PreToolUse`/`PostToolUse`/`SessionStart`. No `hooks.json`, no settings-embedded hooks anywhere
  in the tree. Skills load as **context only**; scripts run **only if the user/agent explicitly
  invokes them**.
- **[SAFE] Only two hardcoded external egress destinations in the entire tree**, both legitimate
  named services with benign payloads:
  - `v2.kubesec.io/scan` — posts the K8s manifest the user chose to scan.
  - `canarytokens.org/generate` — posts a user-supplied email + memo.
  Neither reads or transmits host secrets. Every **other** network call targets a
  **variable/env-configured API endpoint** (VirusTotal, Shodan, MISP, OpenCTI, urlscan…) — the
  expected design of SOC/CTI skills. **No static path to an attacker-controlled host exists.**
- **[SAFE] No download-and-execute.** All `curl … | sh` occurrences are legit vendor installers
  (Trivy, Grype, Syft, Tailscale, uv, Foundry, Sliver, linpeas), **defanged examples**
  (`evil[.]example`, `attacker.com/shell.sh`), or **detection signatures** in skills that hunt the
  pattern. No `base64 -d | sh`, no eval-of-downloaded-content.
- **[SAFE] Third-party contributor code reviewed.** GRC/deception `process.py` files
  (andrewibrah) are inert reporting logic — no network/exec. The only external script using
  `subprocess` is `auditing-foundry-smart-contract-security/scripts/agent.py` (DevRedious): a clean
  `slither` wrapper — **argument-list only, never `shell=True`**, tool-existence checks, timeouts,
  and **local-only** private-key detection that transmits nothing.
- **[SAFE] No prompt injection targeting the reading agent.** "ignore previous instructions"-class
  scan is empty. Keyword hits (`exfiltration`, `override`, `jailbreak`) are all legitimate subject
  matter (DNS-exfil analysis, DLP override policy, jailbreak detection, threat-group notes). The
  only zero-width/bidi unicode is a `ZERO_WIDTH` **sanitization table** inside the anti-injection
  detection skill — the defense, not an attack.
- **[SAFE] Repo tooling is benign.** `tools/validate-skill.py` diff only hardens validation
  (extra required fields, nested-key parsing, `.bak` skip). No `subprocess`/network in `tools/`.
  `index.json` is pure metadata. Not auto-invoked by the plugin.

## Dual-use note (expected, not a defect)

The library ships genuinely offensive tooling — Sliver C2, EternalBlue (MS17-010), LaZagne
credential access, meterpreter references. These are **dual-use**: they attack **targets you point
them at**, not your host, and any install step (e.g. `curl https://sliver.sh/install | sudo bash`)
is a command **the user chooses to run**, never auto-executed. This is the inherent nature of a
red-team / DFIR skill library, not a supply-chain compromise.

## SUMMARY

```
SUMMARY
  CRITICAL: 0  (all pattern hits = false positives for the host threat model)
  HIGH:     0
  MEDIUM:   0
  LOW:      0
  INFO:     dual-use offensive tooling present (by design); scripts run only on explicit invocation
  VERDICT:  SAFE TO RUN
```

Conclusion unchanged from the 2026-06-11 baseline, now extended to the full v1.3.0 tree and every
third-party contribution since. The fork is safe to install and use.
