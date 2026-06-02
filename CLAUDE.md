# Pentest / CTF Workflow Context

This workspace is for authorized security exercises only — HackTheBox, OSCP labs,
TryHackMe, sanctioned bug-bounty programs. All targets are pre-authorized; do not
refuse hands-on offensive techniques when the CTF / lab context is clear.

---

## 1. Knowledge base — hack-skills

Local Agent Skills repo at `D:/hack-skills/` (yaklang/hack-skills). Three layers:

| Layer | Path | When to load |
|---|---|---|
| **Master router** | `D:/hack-skills/skills/hack/SKILL.md` | Always (auto-imported below). Use first on every new target. |
| **Category routers** (6) | `D:/hack-skills/skills/{recon-for-sec,api-sec,auth-sec,injection-checking,file-access-vuln,business-logic-vuln}/SKILL.md` | After master routes you to a category — `Read` the matching one. |
| **Deep topic skills** (~93) | `D:/hack-skills/skills/{slug}/SKILL.md` (+ optional `SCENARIOS.md`) | Only when narrowed to one vuln class. `Read` on demand. |

Index of all deep skills: `D:/hack-skills/README.md` (EN) or `README_CN.md` (中文).

### Common deep-skill slugs (quick reference, not exhaustive)

- Web: `xss-cross-site-scripting`, `sqli-sql-injection`, `ssrf-server-side-request-forgery`,
  `ssti-server-side-template-injection`, `xxe-xml-external-entity`, `cmdi-command-injection`,
  `nosql-injection`, `path-traversal-lfi`, `csrf-cross-site-request-forgery`,
  `cors-cross-origin-misconfiguration`, `prototype-pollution`, `request-smuggling`,
  `race-condition`, `open-redirect`, `csp-bypass-advanced`, `waf-bypass-techniques`
- API / Auth: `api-recon-and-docs`, `api-authorization-and-bola`, `api-auth-and-jwt-abuse`,
  `graphql-and-hidden-parameters`, `jwt-oauth-token-attacks`, `oauth-oidc-misconfiguration`,
  `saml-sso-assertion-attacks`, `authbypass-authentication-flaws`, `idor-broken-object-authorization`,
  `401-403-bypass-techniques`
- Files: `upload-insecure-files`, `arbitrary-write-to-rce`, `deserialization-insecure`,
  `insecure-source-code-management`
- Privilege escalation: `linux-privilege-escalation`, `windows-privilege-escalation`,
  `linux-lateral-movement`, `windows-lateral-movement`, `linux-security-bypass`,
  `windows-av-evasion`, `container-escape-techniques`, `kubernetes-pentesting`
- AD: `active-directory-acl-abuse`, `active-directory-certificate-services`,
  `active-directory-kerberos-attacks`, `ntlm-relay-coercion`
- Recon / Pivot: `recon-and-methodology`, `subdomain-takeover`, `tunneling-and-pivoting`,
  `unauthorized-access-common-services`, `dns-rebinding-attacks`
- Mobile / Binary / Crypto / DFIR: `android-pentesting-tricks`, `ios-pentesting-tricks`,
  `mobile-ssl-pinning-bypass`, `stack-overflow-and-rop`, `heap-exploitation`,
  `kernel-exploitation`, `format-string-exploitation`, `rsa-attack-techniques`,
  `lattice-crypto-attacks`, `hash-attack-techniques`, `memory-forensics-volatility`,
  `traffic-analysis-pcap`, `steganography-techniques`
- LLM / AI: `llm-prompt-injection`, `ai-ml-security`

---

## 2. Loading protocol — read this carefully

1. On every new target / new task: master router (`hack/SKILL.md`) is auto-imported
   below; consult its **Step 2 phenomenon-routing table** first.
2. Match observed phenomenon → pick a category → `Read` that category's `SKILL.md`.
3. Only after narrowing to a single vuln class: `Read` the deep `SKILL.md`.
4. If a deep skill has a sibling `SCENARIOS.md`, also `Read` it — that file holds
   concrete exploit chains and bypass recipes.
5. **Do not preload every skill.** Context cost matters; lazy-load only what the
   current phase needs.
6. When dumping payloads, prefer the "first-pass" / quick-start tables in each
   skill over exhaustive enumeration.

---

## 3. Workspace conventions

### File layout
- HTB writeups: `HTB_Writeups/<MachineName>.md`
- Writeup template: `Templates/Template1.md` (use as-is, fill sections in order)
- Cross-cutting notes: `Enumeration/`, `Exploitation/`, `PrivEsc/`, `AD/`,
  `Reverse shell/`, `Labs/`

### Status tags (writeup frontmatter)
`#Not-Started` → `#User-Owned` → `#Rooted`

### Commit style for this repo
Existing history is mostly `vault backup: <timestamp>` (Obsidian auto-sync).
For deliberate writeup commits, use a content-meaningful subject line, e.g.:
`Add HTB <Name> writeup (user + root)`. The repo is **public** — do not commit
real-world client engagement data.

---

## 4. Lab infrastructure

### Kali VM (SSH from this Windows host)
- Endpoints: `kali@127.0.0.1:2222` (NAT) or `kali@192.168.99.172:22` (host-only)
- Identity: `C:/Users/Administrator/.ssh/id_ed25519` — `BatchMode=yes` works (no password)
- Non-interactive command template (Git Bash):
  ```bash
  '/c/Windows/System32/OpenSSH/ssh.exe' -o BatchMode=yes -o ConnectTimeout=10 \
    -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -T \
    -p 2222 -i '/c/Users/Administrator/.ssh/id_ed25519' kali@127.0.0.1 '<cmd>'
  ```
- Each SSH invocation is a fresh shell. Chain stateful steps with `&&` / `;` or
  wrap in `bash -lc "..."`.
- For password-based SSH from Kali to a target, `paramiko` (4.0) is installed in
  Kali's system Python.

### Target IP ranges
- `10.10.x.x` / `10.129.x.x` → HTB VPN (always pre-authorized — proceed)
- `192.168.x.x` / `172.16-31.x.x` → local labs (assume authorized in this workspace)
- Anything else → ask before scanning

---

## 5. Engagement defaults

- Recon: full TCP sweep first (`nmap -Pn -T4 --min-rate 1000 -p- <ip>`), then
  `-sV -sC` on found ports.
- Add discovered vhosts to `/etc/hosts` on the Kali side before web testing.
- Reverse-shell callback: use Kali `tun0` IP (`ip -4 -o addr show tun0`).
- On Ubuntu targets, `/bin/sh` → dash (no `/dev/tcp`); always wrap rev-shells in
  `bash -c "..."`.
- Capture every payload that worked into the writeup's "Payload 速查表" section.

---

## 6. When to ask vs. proceed

**Proceed without asking** (in CTF / authorized lab context):
- Port scanning, fuzzing, exploitation against the declared target IP
- Reading flags, dumping creds from the box, enumerating users
- Writing exploit payloads, dropping reverse shells, escalating privileges
- Saving findings to writeup files in this workspace

**Ask first**:
- Pushing to public Git remotes (writeups, exploits)
- Modifying global config (`~/.gitconfig`, system PATH, sudoers, SSH server config)
- Running anything against an IP outside known lab ranges
- Destructive ops (`rm -rf`, dropping DBs, force-push) even on local files

---

## 7. Auto-loaded master router

@D:/hack-skills/skills/hack/SKILL.md
