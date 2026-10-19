# C2-DECT Threat Model

Documento de seguridad que define activos, adversarios, fronteras de confianza y
medidas de mitigación para el despliegue de C2-DECT. Es el punto de partida de
cualquier revisión de seguridad o auditoría.

## 1. Assets

| # | Asset | Confidencialidad | Integridad | Disponibilidad | Notas |
|---|-------|------------------|-----------|----------------|-------|
| A1 | Credenciales de operador | Alta | Alta | Alta | Hasheadas (bcrypt/scrypt), SetupWizard fuerza min 12 chars |
| A2 | JWT signing secret | Alta | Alta | Alta | Generado aleatorio (256-bit) en primer arranque, `c2-dect.conf` permisos 0600 |
| A3 | Registro de agentes / claves de sesión | Alta | Alta | Media | Intercambio de claves ECDH + AES-GCM por sesión |
| A4 | Base de tareas (SQLite) | Alta | Alta | Media | `c2-dect.db` - contiene loot, tareas, telemetría |
| A5 | Workstation del operador | Alta | Alta | Media | GUI WPF + token JWT en memoria |
| A6 | Delivery tokens | Alta | Alta | Media | Tokens de campaña (funnel) |
| A7 | Telemetría / actividad | Media | Alta | Media | Logs de actividad y check-ins |
| A8 | Contexto del módulo IA | Media | Media | Baja | Conversaciones/contexto enviado a LLM externo |
| A9 | Certificado TLS autofirmado | Media | Alta | Media | server.crt/server.key |
| A10 | Servidor de delivery (HTTP) | Alta | Alta | Media | Landing + staging |
| A11 | Payloads generados / artefactos | Alta | Alta | Baja | stagers, obfuscators, resultados de build |

## 2. Adversaries

| ID | Adversario | Motivación | Acceso inicial | Impacto potencial |
|----|-----------|-----------|----------------|-------------------|
| ADV-1 | Usuario no autorizado externo | Abuso, explotación | Redes públicas, fuerza bruta login | Tomar control del C2 completo |
| ADV-2 | Agente comprometido (host enemigo) | Pivot, inyección de tareas | Agente capturado / clave de sesión robada | Falsificar tareas, envenenar telemetría |
| ADV-3 | Operador malicioso / interno | Sabotaje, exfiltración | Credenciales legítimas | Exfiltración de loot, destrucción de datos |
| ADV-4 | Atacante contra el servidor | RCE, DoS | Vulnerabilidades HTTP/API/WebSocket | Compromiso total del servidor |
| ADV-5 | Atacante contra la GUI/operador | Phishing, malware de operador | Workstation del operador | Robo de JWT, control indirecto |
| ADV-6 | Fuga accidental de artefactos | - | Commits, backups, delivery mal configurado | Pérdida de OpSec, implicaciones legales |

## 3. Trust boundaries

```
              +---------------------+
              | Browser / GUI WPF   |
              |  (operador)         |
              +----------+----------+
                         | JWT sobre TLS
                         v
              +---------------------+
              |   C2 Server (REST)  |
              +----+--------+---+---+
                   |        |       |
                   | TLS    | local | TLS
                   v        v       v
        +----------+  +---------+  +-------------+
        |  SQLite  |  |  Delivery|  |   Agents    |
        |  (DB)    |  |  server |  |  (beacons)  |
        +----------+  +---------+  +------+------+
                                          |
                                          | (claves de sesión)
                                          v
                                    +-----------+
                                    | IA / LLM  |
                                    | (contexto)|
                                    +-----------+
```

Fronteras críticas:
- **F1 - GUI → C2**: validada por JWT (firmado HS256, pin de algoritmo, issuer `c2-dect`, expiración). CORS restringido a orígenes configurados (`C2_ALLOWED_ORIGIN`).
- **F2 - C2 → DB**: sola, sin autenticación adicional (protección por permisos del sistema de archivos; recomendado cifrado de disco).
- **F3 - C2 → Agents**: autenticación por sesión (clave compartida ECDH), payloads cifrados AES-GCM.
- **F4 - C2 → Delivery**: servidor HTTP separado; sólo sirve artefactos estáticos y staging.
- **F5 - C2 → IA**: contexto limitado y opcional; tratar la salida del LLM como no confiable (no ejecutar jamás directamente).

## 4. Mitigaciones implementadas

| Amenaza | Control |
|---------|---------|
| Ataque de fuerza bruta al login | Rate limiting por IP (login: 3 req/s burst 10; API: 20/s burst 100; agents: 50/s burst 200) |
| Registro de operador no autorizado | `/api/auth/register` restringido a rol `admin` (antes era público) |
| Falsificación JWT / confusión de algoritmo | Pin HS256 + issuer + `WithExpirationRequired` + leeway 30s |
| Robo de token por CORS abusivo | CORS por allow-list de orígenes (`C2_ALLOWED_ORIGIN`), default localhost |
| Exfiltración de datos no auditada | Auditoría de acciones administrativas en `activity_log` (login, tareas, listeners, profiles, payloads) |
| Memoria agotada por subida de archivos | `http.MaxBytesReader` (cap 100 MB en upload) |
| Panic por type assertion en handlers | Cast seguro de `operator_id` |
| Secretos en el repo | `.gitignore` (`.conf`, `*.exe`, bin/obj, payloads, DBs) + `c2-dect.conf` 0600 |
| Tráfico en claro | TLS por defecto (cert autofirmado; en producción usar CA real o proxy) |

## 5. Deficiencias conocidas / trabajo pendiente

- JWT sin rotación automática (el operador re-logea cada 8h; no hay token refresh).
- SQLite sin cifrado a nivel de archivo ni backups automáticos.
- Sin aislamiento de procesos entre listener y API (mismo binario).
- El módulo de entrega (delivery) incluye componentes ofensivos reales: sólo usar en
  laboratorios aislados con autorización; para demostraciones públicas usar el
  modo simulación (ver `delivery/README.md`).
- El módulo worm (`modules/lateral-rust`) ejecuta propagación real: para entornos
  educativos usar el simulador (`modules/worm-sim/`).

## 6. Operación segura (checklist)

- [ ] Ejecutar el servidor con usuario sin privilegios y `--addr 127.0.0.1:8443` si la GUI es local.
- [ ] TLS real (Let's Encrypt / CA privada) o proxy inverso; nunca `--http` en producción.
- [ ] `c2-dect.conf` y `c2-dect.db` con permisos 0600 y fuera de repositorios.
- [ ] `C2_ALLOWED_ORIGIN` con la lista exacta de orígenes de la GUI/web UI.
- [ ] Monitorear `activity_log` (login fallidos, `warning`/`error` levels).
- [ ] Backups de `c2-dect.db` con cifrado; rotación mensual.
- [ ] No exponer el servidor de delivery port 8899 a Internet sin autorización.