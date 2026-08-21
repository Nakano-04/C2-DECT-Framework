# C2-DECT Commercial Readiness Roadmap

## Executive Summary
Transform C2-DECT from a lab/educational framework into a commercial-grade C2 platform suitable for authorized professional red team engagements.

---

## Phase 0: Critical Security Fixes (Week 1-2) - MUST DO FIRST

### 0.1 Authentication & Secrets
- [ ] Remove hardcoded `admin` / `c2-dect` credentials
- [ ] Implement secure initial setup wizard (forced password change on first run)
- [ ] JWT secret: generate cryptographically random 256-bit key on first run, store in config file with restricted permissions
- [ ] Add password complexity requirements + bcrypt cost factor ≥ 12
- [ ] Implement account lockout after failed attempts
- [ ] Add 2FA support (TOTP) for operator accounts

### 0.2 Cryptography
- [ ] Replace XOR-SHA256 in delivery/stage with AES-256-GCM
- [ ] Implement proper key derivation (HKDF-SHA256) from master key
- [ ] Use unique nonce per encryption operation
- [ ] Add authenticated encryption for all C2 traffic (not just delivery)
- [ ] Rotate encryption keys per session

### 0.3 Binary Hygiene
- [ ] Remove all static strings from stager/agent binaries (URLs, keys, paths)
- [ ] Implement build-time string encryption with runtime decryption
- [ ] Strip debug symbols (`-trimpath -ldflags="-s -w -buildid="`)
- [ ] Enable Go build hardening: `-buildmode=pie`, `-ldflags="-extldflags=-Wl,-z,relro,-z,now"`
- [ ] Add control flow integrity where possible

---

## Phase 1: Core Platform Hardening (Week 3-6)

### 1.1 Server Hardening
- [ ] TLS 1.3 only with modern cipher suites
- [ ] Certificate pinning for agent-to-server
- [ ] Mutual TLS (mTLS) for agent authentication
- [ ] Rate limiting on all API endpoints
- [ ] Input validation/sanitization on all handlers
- [ ] Audit logging to immutable store (append-only)
- [ ] Database encryption at rest (SQLCipher or encrypted filesystem)
- [ ] Secure session management with short-lived tokens + refresh tokens

### 1.2 Agent Hardening
- [ ] Reflective DLL injection (no disk writes)
- [ ] Process hollowing with proper cleanup
- [ ] Memory encryption for sensitive data (keys, tokens)
- [ ] Anti-debug/anti-analysis: timing checks, hardware breakpoint detection
- [ ] Secure beacon protocol: jitter, jittered intervals, domain fronting support
- [ ] Kill switch: cryptographic verification of kill command

### 1.3 Communications
- [ ] Malleable C2 profiles: full Cobalt Strike profile compatibility
- [ ] Domain fronting / CDN support
- [ ] HTTP/2 and gRPC listeners
- [ ] DNS-over-HTTPS (DoH) channel
- [ ] Traffic shaping to mimic legitimate applications

---

## Phase 2: Operational Security (Week 7-10)

### 2.1 Evasion
- [ ] Syscall unhooking (full ntdll refresh from disk)
- [ ] ETW/AMSI patching with multiple fallback techniques
- [ ] Sleep obfuscation (RWX→RW→RX transitions)
- [ ] Module stomping / DLL hollowing
- [ ] PPL/Protected Process Light bypass research
- [ ] Kernel callback removal (for drivers)

### 2.2 Persistence
- [ ] Multiple persistence methods with rotation
- [ ] WMI event subscriptions
- [ ] COM hijacking
- [ ] Scheduled tasks with random names/timing
- [ ] Service installation with custom security descriptors
- [ ] Boot persistence (UEFI variables, bootkit research)

### 2.3 Lateral Movement
- [ ] Pass-the-Hash / Pass-the-Ticket with OpSec-safe implementations
- [ ] Kerberos delegation abuse (constrained/unconstrained)
- [ ] DCOM/WMI/WinRM/SMB with proper cleanup
- [ ] SOCKS5 proxy with authentication
- [ ] SSH agent forwarding abuse

---

## Phase 3: Enterprise Features (Week 11-16)

### 3.1 Multi-Tenancy
- [ ] Isolated workspaces per engagement/client
- [ ] Role-based access control (RBAC): Admin, Operator, Analyst, Viewer
- [ ] SSO integration (SAML/OIDC)
- [ ] Audit trail per workspace
- [ ] Resource quotas (agents, listeners, storage)

### 3.2 Team Collaboration
- [ ] Real-time session sharing
- [ ] Shared command history
- [ ] Annotations on sessions/hosts
- [ ] Integrated chat/notes

### 3.3 Reporting & Compliance
- [ ] Executive summaries (PDF)
- [ ] Technical findings with MITRE ATT&CK mapping
- [ ] Evidence chains (hashes, timestamps, screenshots)
- [ ] Compliance frameworks: NIST 800-53, ISO 27001, SOC 2
- [ ] Custom report templates

### 3.4 Integration Ecosystem
- [ ] SIEM connectors (Splunk, Elastic, Sentinel)
- [ ] SOAR playbooks
- [ ] Ticketing integration (Jira, ServiceNow)
- [ ] Threat intel platform integration (MISP, OpenCTI)
- [ ] API-first design with OpenAPI 3.0 spec

---

## Phase 4: Delivery & Payload Generation (Week 17-20)

### 4.1 Payload Factory
- [ ] Supported formats: EXE, DLL, Service EXE, Shellcode (raw, base64, C, Python, PowerShell, VBA, CSHARP)
- [ ] Architecture: x86, x64, ARM64
- [ ] Staged / stageless options
- [ ] Custom prepend/append shellcode
- [ ] Resource embedding (icons, manifests, version info)

### 4.2 Obfuscation Pipeline
- [ ] Control flow flattening
- [ ] Instruction substitution
- [ ] Dead code insertion
- [ ] String encryption with per-build keys
- [ ] Import table obfuscation
- [ ] Resource encryption

### 4.3 Delivery Vectors
- [ ] HTA, LNK, ISO, VHD, MSI, MSIX
- [ ] Macro documents (Office, PDF)
- [ ] Shortcut/URL files with icon spoofing
- [ ] Browser exploit kits (integration with BeEF)
- [ ] Supply chain: trojanized installers

---

## Phase 5: Post-Exploitation Modules (Week 21-26)

### 5.1 Credential Access
- [ ] LSASS dumping: multiple techniques (comsvcs.dll, werfault, direct syscall)
- [ ] DPAPI: user/master key extraction
- [ ] Kerberos: AS-REQ/AS-REP/TGS-REQ/TGS-REP
- [ ] Credential managers: Chrome, Firefox, Edge, Windows Vault
- [ ] SSH keys, GPG keys, AWS/GCP/Azure credentials

### 5.2 Discovery
- [ ] BloodHound/SharpHound integration
- [ ] AD enumeration: trusts, GPOs, ACLs
- [ ] Cloud enumeration: Azure AD, AWS IAM, GCP IAM
- [ ] Network mapping: ARP, NDP, SNMP, NetBIOS

### 5.3 Collection & Exfiltration
- [ ] File staging with compression/encryption
- [ ] Chunked exfiltration over multiple channels
- [ ] Clipboard monitoring
- [ ] Screen/audio capture with throttling
- [ ] Browser history/cookies/passwords

---

## Phase 6: Quality Assurance (Week 27-30)

### 6.1 Testing
- [ ] Unit tests: ≥80% coverage on core packages
- [ ] Integration tests: full C2 lifecycle
- [ ] Evasion testing: against Defender, CrowdStrike, SentinelOne, Cortex XDR
- [ ] Stress testing: 1000+ concurrent agents
- [ ] Chaos engineering: network partitions, server restarts

### 6.2 Security Audit
- [ ] Internal code review
- [ ] External penetration test of C2 server
- [ ] Dependency scanning (Snyk, Dependabot)
- [ ] SAST/DAST pipeline

### 6.3 Documentation
- [ ] Operator manual (PDF + web)
- [ ] API reference (auto-generated from OpenAPI)
- [ ] Deployment guides (Docker, Kubernetes, bare metal)
- [ ] Hardening guide
- [ ] Incident response playbook for C2 compromise

---

## Phase 7: Release & Support (Week 31+)

### 7.1 Release Engineering
- [ ] Semantic versioning
- [ ] Signed releases (cosign/sigstore)
- [ ] SBOM generation (CycloneDX)
- [ ] Release notes with migration guides
- [ ] LTS branches with security backports

### 7.2 Licensing
- [ ] License key validation (offline-capable)
- [ ] Feature gates: Core / Professional / Enterprise
- [ ] Trial mode with time/features limits
- [ ] Air-gapped license activation

### 7.3 Support Infrastructure
- [ ] Customer portal
- [ ] Knowledge base
- [ ] Ticketing system
- [ ] Security advisory mailing list
- [ ] Professional services offering

---

## Resource Estimate

| Phase | Duration | Engineers | Notes |
|-------|----------|-----------|-------|
| 0 | 2 weeks | 2 | Critical - blocker for any commercial use |
| 1 | 4 weeks | 3 | Core platform |
| 2 | 4 weeks | 2-3 | OpSec specialist needed |
| 3 | 6 weeks | 3 | Frontend + backend |
| 4 | 4 weeks | 2 | Reverse engineering skills |
| 5 | 6 weeks | 3 | Windows internals expertise |
| 6 | 4 weeks | 2 | QA + security |
| 7 | Ongoing | 1-2 | Release + support |

**Total: ~30 weeks (7.5 months) with 3-4 engineers**

---

## Immediate Next Steps (This Week)

1. **Run the security fixes in this repo** (see `SECURITY_FIXES.md`)
2. **Set up CI/CD** with GoReleaser, automated testing
3. **Create threat model** document (STRIDE for each component)
4. **Define MVP scope** for first commercial release (phases 0-1 minimum)
5. **Legal review**: export controls (Wassenaar), EULA, liability

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| EDR detection of current binaries | High | Critical | Phase 0-2 priority |
| Legal/export control issues | Medium | High | Early legal review |
| Single maintainer bus factor | High | High | Hire/contract immediately |
| Technical debt in legacy phases | High | Medium | Refactor incrementally |
| Competitor feature parity | Medium | Medium | Focus on unique value (worm, delivery kit) |

---

*Document version: 1.0 | Created: 2026-08-17 | Classification: INTERNAL - CONFIDENTIAL*