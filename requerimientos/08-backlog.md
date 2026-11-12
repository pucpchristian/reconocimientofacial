# Backlog — Módulo Operativo y Control de Asistencia Omnicanal

**Versión:** 1.2.0
**Fecha:** 11 de Noviembre de 2026
**Referencias:** Spec Pack v2.2.2 (`00` a `07`).
**Cambios v1.2.0:** +6 tareas (T-055 a T-060) para cerrar los hallazgos QA-VAL-001 a QA-VAL-006 del `10-validation-log.md`. Se extienden T-010, T-012, T-016, T-039, T-049. Total: **60 tareas**.
**Cambios v1.1.0:** +20 tareas (T-035 a T-054) para cerrar los hallazgos QA-C2-001 a QA-C2-015 del `09-agente-qa-backlog.md`. +Anexo C (matriz de 5 columnas). Total: **54 tareas**.

---

## Resumen Ejecutivo

| Métrica | Valor |
|---|---|
| **Total de tareas** | **60** |
| Must | 56 |
| Should | 4 (T-039, T-040, T-028, T-055) |
| Could | 0 |
| Setup | 13 |
| Features | 25 |
| Integración | 4 |
| Testing | 18 |

### Por componente

| Componente | Tareas | Must | Should |
|---|---|---|---|
| A — Mobile Application | 20 | 18 | 2 |
| B — ADMS Ingestion Receiver | 4 | 4 | 0 |
| C — Async Processing Engine | 6 | 6 | 0 |
| D — Attendance Core Service | 13 | 13 | 0 |
| E — Monitoring Dashboard | 4 | 2 | 2 |
| F — Compliance & KMS Service | 11 | 11 | 0 |
| Transversal / UI (P-005, P-007) | 2 | 2 | 0 |

### Por tamaño

| Tamaño | Cantidad | Esfuerzo |
|---|---|---|
| S | 8 | ~4 h |
| M | 36 | ~30 h |
| L | 16 | ~24 h |
| **Total** | **60** | **~58 h** |

### Ruta crítica

`T-001 → T-002 → T-007 → T-010 → T-015 → T-020 → T-021 → T-023 → T-016`

Duración estimada: **~6.5 h**. Profundidad: **9 niveles**. T-055…T-060 no alargan la ruta crítica (paralelizables tras T-010/T-012/T-039).

---

## Componente D — Attendance Core Service

### Setup

**T-001: Configurar monorepo, CI/CD y policy de restricciones para agentes de IA**
- Épica: Infraestructura | HU: Infraestructura | Prioridad: Must | Tamaño: M | Dependencias: Ninguna
- **Descripción:** Monorepo con workspaces. `.github/workflows/ci.yml` con lint/test/build. `forbidden-files.yml` bloquea merges si modifican `migrations/` o desactivan tests (P-007).
- **Criterio:** `npm run ci` verde; PR que modifica `migrations/` sin aprobación es rechazado.

**T-002: Crear esquema PostgreSQL 15+ con 6 entidades, trigger inmutable e índices**
- Épica: transversal | HU: Infraestructura | Prioridad: Must | Tamaño: L | Dependencias: T-001
- **Descripción:** Migración `001_init_schema.sql` con `tenants`, `sites`, `workers`, `devices`, `attendance_events`, `audit_logs`. FK, CHECK, BYTEA AES-256. Trigger inmutable. Índices para RNF-002/RNF-003.
- **Criterio:** Script ejecuta sin errores; UPDATE/DELETE en `audit_logs` falla; EXPLAIN usa `idx_attendance_tenant_timestamp`.

**T-042: Configurar observabilidad (Prometheus + Grafana + alertas RNF-003/005)**
- Épica: Infraestructura | HU: Infraestructura | Prioridad: Must | Tamaño: M | Dependencias: T-001
- **Descripción:** Prometheus + Grafana con métricas de latencia p50/p95/p99, tasa 5xx, throughput ADMS, backlog BullMQ. Alertas: p95 > 100 ms por 5 min en `/api/adms/push`, 5xx > 1 % por 10 min, backlog > 1,000 jobs por 5 min. Documentar en `docs/observability.md`.
- **Criterio:** Dashboards muestran métricas; alertas notifican por Slack/email al simular condiciones con k6.

**T-043: Configurar deployment a producción con IaC (Terraform/Pulumi)**
- Épica: Infraestructura | HU: Infraestructura | Prioridad: Must | Tamaño: L | Dependencias: T-001
- **Descripción:** IaC para provisioning de PostgreSQL, Redis, KMS, ALB, pods. Pipeline `deploy-prod.yml` con aprobación manual. Documentar en `docs/deployment.md`.
- **Criterio:** `terraform apply` levanta el entorno completo; pipeline despliega a staging y producción tras aprobación.

**T-044: Configurar HashiCorp Vault (o AWS Secrets Manager) para KMS/OTP/MDM**
- Épica: Infraestructura | HU: Infraestructura | Prioridad: Must | Tamaño: M | Dependencias: T-001
- **Descripción:** Vault para almacenar credenciales de KMS, OTP, MDM. Backend lee secrets en runtime (no en código). `P-007` prohíbe secretos en código.
- **Criterio:** Backend obtiene secrets de Vault; no hay secretos en `.env` ni en código; rotación de credenciales funciona.

### Features

**T-003: Implementar gestión de sedes con coordenadas WGS84**
- Épica: E-06 | HU: HU-011 | Prioridad: Must | Tamaño: M | Dependencias: T-002
- **Descripción:** `SiteController.create()` exige `latitude` y `longitude` WGS84. Fija `geofence_radius_m = 50.00` (RN-010). Advierte "Sede duplicada" si coordenadas coinciden.
- **Criterio:** Sede válida → 201; `latitude = 91` → 422; dirección en texto libre → 422; duplicado → advertencia.

**T-004: Implementar enrollment de trabajador con consentimiento versionado**
- Épica: E-06 | HU: HU-010 | Prioridad: Must | Tamaño: M | Dependencias: T-002
- **Descripción:** `WorkerController.enroll()` exige `privacy_policy_version`, `consent_timestamp`, `signature_method`, `consent_hash` (RN-030). Sin consentimiento → 403. Cambio de política fuerza re-consentimiento (RN-030-A, RN-030-C).
- **Criterio:** Sin consentimiento → 403; con consentimiento → 201; cambio de política fuerza re-consentimiento ≤ 30 días.

**T-005: Implementar evaluación geodésica Haversine con epsilon 0.01 m**
- Épica: E-02 | HU: HU-002, HU-003 | Prioridad: Must | Tamaño: M | Dependencias: T-002
- **Descripción:** `GeofenceService.evaluate()` con Haversine (R = 6,371,000 m) + epsilon 0.01 m (RN-017-A).
- **Criterio:** `D = 50.00` → INSIDE; `D = 50.01` → OUTSIDE; `D = 50.005` → INSIDE.

**T-041: Implementar POST /api/auth/environment-session con JWT RS256 TTL 12 h (RN-007, P-002)**
- Épica: E-05 | HU: HU-009 | Prioridad: Must | Tamaño: M | Dependencias: T-008, T-002
- **Descripción:** `AuthController.createEnvironmentSession()` recibe `tenant_id`, `user_agent_hash`, `ip_hash`, `kiosk_uuid`. Emite `session_token` JWS RS256 TTL 12 h con claims `tenant_id`, `kiosk_uuid`, `environment_fingerprint_hash`. Sin formulario usuario/contraseña (P-002). Rechaza kioscos no registrados con HTTP 403.
- **Criterio:** POST con kiosco autorizado → 200 con `session_token`; kiosco no registrado → 403; token expira a las 12 h; request con token expirado → 401.

**T-040: Implementar generación de reporte de tareo para pre-planilla (RN-002-B, RN-019, RN-020-A)**
- Épica: E-13 | HU: HU-019 | Prioridad: Should | Tamaño: L | Dependencias: T-005
- **Descripción:** `TareoService.generateReport()` para periodo cerrado. Consolidado de asistencia, tardanzas y sobretiempos en ≤ 15 min con formato S/ y fecha DD/MM/AAAA. Excluye marcaciones `REQUIRES_MANUAL_APPROVAL` hasta aprobación. Marca faltas (días sin marcación).
- **Criterio:** Reporte con periodo cerrado → ≤ 15 min; marcaciones no aprobadas excluidas; faltas marcadas explícitamente.

### Testing

**T-006: E2E de evaluación geodésica con epsilon (RN-017-A)**
- Épica: E-02 | HU: HU-002 | Prioridad: Must | Tamaño: S | Dependencias: T-005
- **Descripción:** `geofence-epsilon.e2e-spec.ts` con 4 casos de borde.
- **Criterio:** 4 casos pasan en CI.

**T-050: E2E de POST /api/auth/environment-session (RN-007, P-002)**
- Épica: E-05 | HU: HU-009 | Prioridad: Must | Tamaño: M | Dependencias: T-041
- **Descripción:** `auth-env-session.e2e-spec.ts`. Verifica: (a) emisión de token con kiosco autorizado; (b) rechazo de kiosco no registrado; (c) expiración a las 12 h; (d) rechazo de token expirado.
- **Criterio:** 4 escenarios pasan; token válido permite acceso al Dashboard sin login tradicional.

**T-053: E2E de consentimiento versionado (RN-030, RN-030-A, RN-030-B, RN-030-C)**
- Épica: E-06 | HU: HU-010 | Prioridad: Must | Tamaño: M | Dependencias: T-004
- **Descripción:** `consent-lifecycle.e2e-spec.ts`. Verifica: (a) enrolamiento sin consentimiento → 403; (b) con consentimiento → 201; (c) cambio de política fuerza re-consentimiento ≤ 30 días; (d) solapamiento RN-030-A/B aplica 30 días (RN-030-C).
- **Criterio:** 4 escenarios pasan; `consent_hash` reproducible; `PENDING_CONSENT` bloquea biometría.

---

## Componente A — Mobile Application

### Setup

**T-007: Configurar React Native/Capacitor con cifrado local respaldado por hardware**
- Épica: E-01, E-04, E-08 | HU: HU-001, HU-007 | Prioridad: Must | Tamaño: M | Dependencias: T-001
- **Descripción:** React Native 0.74+ con `Android Keystore` (AES-GCM 256) e `iOS Keychain`. Plugins: camera, geolocation, netinfo, notifee. CI para APK/IPA firmados.
- **Criterio:** Compila Android + iOS; `encryptLocal`/`decryptLocal` funcionales.

### Features

**T-008: Implementar captura facial con liveness y match on-device**
- Épica: E-01 | HU: HU-001 | Prioridad: Must | Tamaño: L | Dependencias: T-007
- **Descripción:** `BiometricCaptureScreen` con `CameraX`/`AVFoundation` + ML Kit/Vision. Gesto ≤ 5.0 s, `liveness_score >= 0.95`, `match_score >= 0.92` (RN-001A, RN-001B).
- **Criterio:** Flujo ≤ 5 s; gesto correcto + match 0.94 → SUCCESS; fallo → LIVENESS_FAILED.

**T-009: Implementar geolocalización y geocerca cliente con re-captura**
- Épica: E-02 | HU: HU-002, HU-003 | Prioridad: Must | Tamaño: M | Dependencias: T-007
- **Descripción:** `LocationService` con Haversine + epsilon 0.01 m. `gps_accuracy > 25 m` → 3 re-capturas (RN-002-A/B).
- **Criterio:** `D = 50.00` → INSIDE; `gps_accuracy = 45` → UNCERTAIN; 3 re-capturas fallidas → REQUIRES_MANUAL_APPROVAL.

**T-010: Implementar anti-fraude Android (mock) e iOS (consenso 3 muestras)**
- Épica: E-03 | HU: HU-004, HU-005, HU-026 | Prioridad: Must | Tamaño: L | Dependencias: T-007
- **Descripción:** `AntiFraudService`. Android: `isFromMockProvider()` + root. iOS: 4 señales con consenso de 3 muestras en ventana 3-30 s (RN-003-A-1…A-5). < 2 señales testeables → REQUIRES_MANUAL_APPROVAL (RN-003-A-6). Jailbreak (RN-003-B). **Exponer `sample_progress` (`1/3` → `2/3` → `3/3`) al componente UI `IosConsensusProgressBar` (QA-VAL-004) y `integrity_diagnostic_code` = `ERR_INTEGRITY_FAIL_<uuid8>` al modal de remediación (QA-VAL-003).**
- **Criterio:** Mock Android → FAKE_LOCATION_DETECTED; iOS 3 muestras → FAKE_LOCATION_DETECTED; < 2 señales → REQUIRES_MANUAL_APPROVAL; `sample_progress` y `integrity_diagnostic_code` disponibles en el estado.
**T-011: Implementar fallback NTP en cliente (reloj monotónico + offset calibrado)**
- Épica: E-03 | HU: HU-006 | Prioridad: Must | Tamaño: M | Dependencias: T-007
- **Descripción:** `NtpService` con timeout 2,000 ms. Offline → reloj monotónico + `offset_calibrado` con `|offset| ≤ 5 s` y vigencia ≤ 12 h (RN-020).
- **Criterio:** NTP válido → `ntp_timestamp`; offline con offset válido → derivado; offset > 5 s → UNTRUSTWORTHY_TIME.

**T-012: Implementar almacenamiento offline cifrado y banner persistente**
- Épica: E-04 | HU: HU-007, HU-008, HU-025 | Prioridad: Must | Tamaño: L | Dependencias: T-007, T-011
- **Descripción:** `OfflineStorageService` con Keystore/Keychain (5,000 reg o 200 MB, RN-021). `WorkManager`/`BGTaskScheduler` lotes de 50 con backoff (RN-005). `SyncBanner` con umbrales ámbar/rojo (RN-005-G) **y nuevo estado rojo parpadeante al 80 % de capacidad (4,000 reg / 160 MB) con mensaje "⚠️ Almacenamiento local al 81 % (4,050/5,000). Conéctese a red para sincronizar" (RN-021-A / QA-VAL-002)**. Manejo de reinstalación con `APP_REINSTALL_DATA_LOSS` (RN-005-H). **Exponer `saturation_state` (`NORMAL` / `WARNING_80` / `BLOCKED_100`) al componente `SyncBanner`.**
- **Criterio:** 3 pendientes → banner ámbar; 4,050 pendientes → banner rojo parpadeante; 5,000 → bloqueo; reinstalación → APP_REINSTALL_DATA_LOSS; `saturation_state` cambia en los umbrales exactos.

**T-013: Implementar autenticación del supervisor y device binding**
- Épica: E-08, E-10 | HU: HU-013, HU-016 | Prioridad: Must | Tamaño: M | Dependencias: T-007
- **Descripción:** `AuthScreen` con PIN 4 dígitos + biometría de plataforma. `session_token` TTL ≤ 12 h. Valida `device_uuid` (RN-012). Vinculación primaria + backup con OTP ≤ 2 min (RN-012-A).
- **Criterio:** PIN + biometría → acceso; PIN incorrecto 5x → bloqueo; `device_uuid` no enrolado → DEVICE_UNAUTHORIZED.

**T-014: Implementar activación de dispositivo de respaldo (OTP + biometría)**
- Épica: E-08 | HU: HU-013 | Prioridad: Must | Tamaño: M | Dependencias: T-013
- **Descripción:** `BackupActivationScreen` invoca `POST /api/devices/{uuid}/activate-backup` con OTP + biometría. SLA: OTP ≤ 2 min, activación ≤ 5 min (RN-012-A).
- **Criterio:** Activación válida → 200 con `sla_met = true`; OTP expirado → 401.

**T-039: Implementar bloqueo temporal de terminal (RN-014/RN-014-A)**
- Épica: E-11 | HU: HU-017 | Prioridad: Should | Tamaño: M | Dependencias: T-010
- **Descripción:** `TemporaryBlockService`. Cuenta intentos rechazados por `FAKE_LOCATION_DETECTED` o `LIVENESS_FAILED`. 3 intentos en 15 min → bloqueo 30 min + `event_type = DEVICE_TEMPORARILY_BLOCKED`. Reinicio del contador con `SUCCESS`, 60 min sin rechazos, o re-autenticación (RN-014-A). **La App DEBE exponer el `block_expires_at` al componente UI `BlockCountdownTimer` para renderizar el temporizador `MM:SS` (QA-VAL-001).**
- **Criterio:** 3 fallos → bloqueo 30 min; re-autenticación → contador a 0; bloqueo expira → permite nuevo intento; `block_expires_at` disponible en el estado de la App.

**T-055: Implementar temporizador regresivo `MM:SS` en P-001 durante bloqueo PIN (QA-VAL-001)**
- Épica: E-11 | HU: HU-017 | Prioridad: Should | Tamaño: S | Dependencias: T-039, T-013
- **Descripción:** Componente React Native `BlockCountdownTimer` que consume `block_expires_at` del `TemporaryBlockService` y renderiza un contador `MM:SS` (ej. "Intente de nuevo en 29:45"). Deshabilita los botones de biometría y PIN hasta que el contador llegue a `00:00`. Al expirar, rehabilita los controles sin re-bloqueo automático (RN-014-A).
- **Criterio:** Al abrir P-001 con bloqueo activo, se renderiza el contador regresivo `MM:SS`; a `00:00` los botones se rehabilitan; el test E2E T-016 verifica el texto exacto "Intente de nuevo en 29:45".

**T-058: Implementar indicador visual paso a paso "Muestra X/3" para consenso iOS (QA-VAL-004)**
- Épica: E-03 | HU: HU-003, HU-026 | Prioridad: Must | Tamaño: S | Dependencias: T-010, T-016
- **Descripción:** Componente React Native `IosConsensusProgressBar` en P-002 que renderiza una barra de progreso con el texto "Muestra 1/3 · Muestra 2/3 · Muestra 3/3" durante la captura de las 3 lecturas GPS consecutivas en iOS (RN-003-A). Si tras 3 muestras no se alcanza el consenso mínimo, muestra el estado `REQUIRES_MANUAL_APPROVAL` con botón "Derivar a Aprobación Manual" hacia P-005.
- **Criterio:** El texto "Muestra 1/3" aparece en la primera lectura; "Muestra 3/3" en la tercera; al fallar el consenso, el botón "Derivar a Aprobación Manual" está habilitado.

**T-057: Implementar modal de remediación root/jailbreak en P-002 (QA-VAL-003)**
- Épica: E-03 | HU: HU-005 | Prioridad: Must | Tamaño: S | Dependencias: T-010
- **Descripción:** Modal de seguridad en P-002 tras detección de root/jailbreak con: (a) código de diagnóstico único `ERR_INTEGRITY_FAIL_<uuid8>`, (b) botón "Contactar Soporte TI" que copia el código al portapapeles (`Clipboard.setString()`), (c) instrucción "Contacte a su administrador para restablecer la integridad del dispositivo". Cierra la sesión activa y bloquea P-002 hasta remediación.
- **Criterio:** Detección de root/jailbreak → modal con `ERR_INTEGRITY_FAIL_<uuid8>` visible; al pulsar "Contactar Soporte TI" el código se copia al portapapeles; P-002 queda bloqueada.

**T-059: Implementar badge "🔒 Datos cifrados AES-256 (Ley N.° 29733)" en P-005 (QA-VAL-005)**
- Épica: E-06, E-09 | HU: HU-010, HU-015 | Prioridad: Must | Tamaño: S | Dependencias: T-004
- **Descripción:** Componente `ComplianceBadge` en el formulario de enrolamiento biométrico (P-005), junto al campo de captura de vector facial, con texto persistente "🔒 Datos biométricos cifrados en reposo AES-256 (Ley N.° 29733)" y tooltip explicativo sobre retención (duración laboral + 30 días). Visible en la pantalla de enrolamiento y en el panel de detalle del trabajador.
- **Criterio:** El badge aparece junto al campo de captura de vector; el tooltip muestra el texto de retención; el test E2E T-053 verifica la presencia del badge.

**T-060: Implementar badge "✅ Reporte firmado SHA-256" en P-007 (QA-VAL-006)**
- Épica: E-13, E-14 | HU: HU-019, HU-020 | Prioridad: Should | Tamaño: S | Dependencias: T-023, T-040
- **Descripción:** Componente `SignedReportBadge` en P-007 junto al botón de exportación de pre-planilla. Muestra "✅ Reporte firmado digitalmente (SHA-256)" y el `signature_hash` truncado (primeros 16 caracteres). Añade enlace "Verificar integridad" que descarga el manifiesto `signature_manifest.json` (`signature_hash` + `tenant_id` + `utc_timestamp`).
- **Criterio:** Al generar el reporte, el badge muestra el `signature_hash` truncado; el enlace descarga el manifiesto con los 3 campos; T-024 verifica la reproducibilidad del hash.


### Integración

**T-015: Integrar Mobile App con POST /api/attendance/mark**
- Épica: E-01, E-02, E-03 | HU: HU-001 | Prioridad: Must | Tamaño: M | Dependencias: T-008…T-013, T-020
- **Descripción:** `AttendanceApiService.mark()` envía POST con schema de §4.1. Maneja 200/401/403/422/429/503.
- **Criterio:** POST válido → 200; 422 → re-muestra error; 503 → REQUIRES_MANUAL_APPROVAL.

### Testing

**T-016: E2E de anti-fraude iOS con consenso de 3 muestras (RN-003-A-6)**
- Épica: E-03 | HU: HU-004, HU-026 | Prioridad: Must | Tamaño: M | Dependencias: T-010
- **Descripción:** 3 escenarios: 3 muestras → FAKE_LOCATION_DETECTED; 2 muestras → no bloquea; `signals_count = 1` → REQUIRES_MANUAL_APPROVAL. **Añadir verificación del indicador paso a paso "Muestra 1/3 · Muestra 2/3 · Muestra 3/3" en P-002 (QA-VAL-004) y del modal de remediación root/jailbreak con código `ERR_INTEGRITY_FAIL_<uuid8>` y botón "Contactar Soporte TI" (QA-VAL-003).** **Añadir verificación del temporizador regresivo `MM:SS` en P-001 al 3.er intento fallido (QA-VAL-001).**
- **Criterio:** 5 escenarios verdes en CI (3 anti-fraude + 1 UI indicador muestras + 1 UI temporizador PIN).

**T-046: E2E de activación de backup con SLA y rechazo (RN-012-A)**
- Épica: E-08 | HU: HU-013 | Prioridad: Must | Tamaño: M | Dependencias: T-014
- **Descripción:** `backup-activation.e2e-spec.ts`. Verifica: (a) activación exitosa con SLA ≤ 5 min; (b) OTP expirado → 401; (c) backup no enrolado → 403; (d) `sla_met = true` con OTP ≤ 2 min.
- **Criterio:** 4 escenarios pasan; SLA verificado con timestamps.

**T-047: E2E de biometría (RN-001A, RN-001B, RN-001C, RN-001D)**
- Épica: E-01, E-15 | HU: HU-001, HU-021 | Prioridad: Must | Tamaño: M | Dependencias: T-008
- **Descripción:** `biometric-flow.e2e-spec.ts`. Verifica: (a) liveness ≥ 0.95 y match ≥ 0.92 → SUCCESS; (b) liveness < 0.95 → LIVENESS_FAILED; (c) match < 0.92 → rechazo; (d) SDK no inicializa → BIOMETRIC_SDK_UNAVAILABLE; (e) `|client − server| > 0.05` → BIOMETRIC_INCONSISTENCY.
- **Criterio:** 5 escenarios pasan; `event_type` correcto en `audit_logs`.

**T-048: E2E de NTP y fallback online/offline (RN-004, RN-020, RN-020-A)**
- Épica: E-03 | HU: HU-006 | Prioridad: Must | Tamaño: M | Dependencias: T-011
- **Descripción:** `ntp-fallback.e2e-spec.ts`. Verifica: (a) NTP online válido → `ntp_timestamp`; (b) offline con offset válido → derivado; (c) offline sin offset → UNTRUSTWORTHY_TIME; (d) online con 2 servidores NTP caídos → HTTP 503 + UNTRUSTWORTHY_TIME.
- **Criterio:** 4 escenarios pasan; `|offset| ≤ 5 s` y vigencia ≤ 12 h validados.

**T-049: E2E de offline y banner (RN-005, RN-005-G, RN-019, RN-021)**
- Épica: E-04 | HU: HU-007, HU-008 | Prioridad: Must | Tamaño: M | Dependencias: T-012
- **Descripción:** `offline-sync.e2e-spec.ts`. Verifica: (a) 3 marcaciones offline cifradas + sincronización al recuperar red; (b) banner ámbar con 3 pendientes; (c) banner rojo con 101 pendientes; (d) 72 h → REQUIRES_MANUAL_APPROVAL; (e) 5,000 registros → bloqueo. **(f) Con 4,050 registros pendientes o 162 MB usados, el banner muestra el estado rojo parpadeante "⚠️ Almacenamiento local al 81 %" y fuerza reintento de sync si hay red (RN-021-A / QA-VAL-002).**
- **Criterio:** 6 escenarios pasan; `sync_status` correcto en cada caso; estado de saturación al 80 % verificado.

**T-051: E2E de device binding (RN-012, RN-012-A, RN-012-B)**
- Épica: E-08 | HU: HU-013 | Prioridad: Must | Tamaño: M | Dependencias: T-013
- **Descripción:** `device-binding.e2e-spec.ts`. Verifica: (a) `device_uuid` autorizado → permite; (b) `device_uuid` no enrolado → 403 + DEVICE_UNAUTHORIZED; (c) backup activación con OTP + biometría ≤ 5 min; (d) sin backup sin red → PENDING_DEVICE_ACTIVATION.
- **Criterio:** 4 escenarios pasan; SLA de activación verificado.

---

## Componente B — ADMS Ingestion Receiver

### Setup

**T-017: Configurar AdmsReceiver Fastify e implementar POST /api/adms/push**
- Épica: E-07 | HU: HU-012 | Prioridad: Must | Tamaño: M | Dependencias: T-002
- **Descripción:** `AdmsController.push()` valida `serial_number`. Rate limit 10 RPS (RN-011-A). Payload > 10 MB → 413 (RN-011-B). Responde 200 `OK` en p95 ≤ 100 ms.
- **Criterio:** POST válido → 200 `OK`; SN no enrolado → 400; 12,000 registros → 413; SN > 10 RPS → 429.

### Features

**T-018: Implementar heartbeat y detección BIOMETRIC_DEVICE_OFFLINE**
- Épica: E-07 | HU: HU-012 | Prioridad: Must | Tamaño: M | Dependencias: T-017
- **Descripción:** Job cada 60 s. `now − last_heartbeat > 10 × intervalo` → `BIOMETRIC_DEVICE_OFFLINE` (RN-018).
- **Criterio:** Sin heartbeat 11 min → BIOMETRIC_DEVICE_OFFLINE; evento en `audit_logs`.

### Testing

**T-019: Load test ADMS 500 RPS y ráfagas 1,000 RPS (RNF-003)**
- Épica: E-07 | HU: HU-012 | Prioridad: Must | Tamaño: M | Dependencias: T-017, T-020
- **Descripción:** k6/Artillery. Escenario 1: 500 SN × 1,000 registros. Escenario 2: ráfaga 1,000 RPS × 10 s.
- **Criterio:** p95 ≤ 100 ms, p99 ≤ 250 ms, 0 errores 5xx.

**T-052: E2E de heartbeat y detección offline (RN-018)**
- Épica: E-07 | HU: HU-012 | Prioridad: Must | Tamaño: M | Dependencias: T-018
- **Descripción:** `heartbeat.e2e-spec.ts`. Verifica: (a) heartbeat cada 60 s → AUTHORIZED; (b) sin heartbeat > 10 min → BIOMETRIC_DEVICE_OFFLINE; (c) recuperación de heartbeat → AUTHORIZED; (d) `event_type = BIOMETRIC_DEVICE_OFFLINE` en audit.
- **Criterio:** 4 escenarios pasan; timers simulados con `jest.useFakeTimers()`.

---

## Componente C — Async Processing Engine

### Setup

**T-020: Configurar workers BullMQ para ingesta asíncrona**
- Épica: E-04, E-07 | HU: HU-007, HU-012 | Prioridad: Must | Tamaño: M | Dependencias: T-002
- **Descripción:** Colas `adms-ingestion-queue` y `mobile-sync-queue`. Concurrencia 20, reintentos 3. Pool Redis 50/200.
- **Criterio:** Job encolado y consumido ≤ 500 ms; pool mínimo 50 conexiones.

### Features

**T-021: Implementar idempotencia y deduplicación de marcaciones**
- Épica: E-04 | HU: HU-007 | Prioridad: Must | Tamaño: M | Dependencias: T-020
- **Descripción:** Valida terna `(worker_id, ntp_timestamp, device_uuid)` contra `idempotency_keys` en 60 s (RN-006).
- **Criterio:** Duplicado → no inserta; nuevo → inserta + clave.

**T-022: Implementar re-verificación biométrica server-side condicional**
- Épica: E-15 | HU: HU-021 | Prioridad: Must | Tamaño: M | Dependencias: T-020
- **Descripción:** Si `server_side_reverification = true`: descifra `reference_embedding`, calcula `server_match_score`. `|client − server| > 0.05` → BIOMETRIC_INCONSISTENCY (RN-001C).
- **Criterio:** Discrepancia 0.01 → SUCCESS; 0.13 → BIOMETRIC_INCONSISTENCY.

**T-023: Implementar log de auditoría inmutable con firma hash**
- Épica: transversal | HU: HU-020 | Prioridad: Must | Tamaño: M | Dependencias: T-020, T-002
- **Descripción:** Worker `audit.write` inserta con 8 campos (P-009) + `signature_hash = SHA-256(row + tenant_secret)`.
- **Criterio:** INSERT con 8 campos + firma; UPDATE/DELETE falla por trigger.

### Testing

**T-024: E2E de idempotencia y auditoría inmutable (RN-006, RN-016, P-009)**
- Épica: E-04 | HU: HU-007, HU-020 | Prioridad: Must | Tamaño: M | Dependencias: T-021, T-023
- **Descripción:** 3 jobs idénticos → 1 registro; UPDATE/DELETE fallan; `signature_hash` reproducible.
- **Criterio:** 1 registro tras 3 jobs; fallos de UPDATE/DELETE; hash coincidente.

**T-045: E2E de POST /api/attendance/sync con lotes y expiración (RN-006, RN-019)**
- Épica: E-04 | HU: HU-007 | Prioridad: Must | Tamaño: M | Dependencias: T-021
- **Descripción:** `attendance-sync.e2e-spec.ts`. Verifica: (a) lote de 50 → 207 con 50 resultados; (b) `batch_id` repetido → 409; (c) antigüedad > 72 h → REQUIRES_MANUAL_APPROVAL; (d) idempotencia por terna.
- **Criterio:** 4 escenarios pasan; expiración validada con timers.

---

## Componente E — Monitoring Dashboard

### Setup

**T-025: Configurar Next.js + Leaflet + Tailwind con WCAG 2.1 AA**
- Épica: E-05 | HU: HU-009 | Prioridad: Must | Tamaño: M | Dependencias: T-001
- **Descripción:** Next.js 14+ con Leaflet y Tailwind. `@axe-core/react` para accesibilidad. Contraste ≥ 4.5:1.
- **Criterio:** `axe` reporta 0 violaciones; navegación por teclado funciona.

### Features

**T-026: Implementar GET /api/dashboard/attendance y mapa interactivo**
- Épica: E-05 | HU: HU-009 | Prioridad: Must | Tamaño: L | Dependencias: T-025, T-005, T-041
- **Descripción:** `DashboardController.getAttendance()` con filtros. `MapView` con geocercas de 50 m y pines verde/rojo. Acceso vía `session_token` (P-002). p95 ≤ 1.5 s (RNF-002).
- **Criterio:** Círculo 50 m + pin color; filtros paginados; p95 ≤ 1.5 s.

**T-027: Implementar alertas operativas en tiempo real**
- Épica: E-12 | HU: HU-018 | Prioridad: Should | Tamaño: M | Dependencias: T-026
- **Descripción:** `AlertPanel` consume WebSocket/SSE. Alertas por `OUTSIDE_GEOFENCE` y `FAKE_LOCATION_DETECTED`.
- **Criterio:** Alerta ≤ 2 s; contador actualizado.

### Integración

**T-028: Integrar Dashboard con API SSE para alertas**
- Épica: E-12 | HU: HU-018 | Prioridad: Should | Tamaño: S | Dependencias: T-026, T-027
- **Descripción:** Cliente SSE con reconexión y backoff.
- **Criterio:** Conexión activa; reconexión automática.

---

## Componente F — Compliance & KMS Service

### Setup

**T-029: Configurar ComplianceKMSService con cliente KMS**
- Épica: E-08, E-09 | HU: HU-015 | Prioridad: Must | Tamaño: M | Dependencias: T-002
- **Descripción:** Cliente KMS con `encrypt`, `decrypt`, `revokeKey`. Jobs diferidos `SOFT_DELETE_BIOMETRIC` (24 h) y `PHYSICAL_DELETE` (30 d).
- **Criterio:** KMS funcional; jobs persistidos.

### Features

**T-030: Implementar POST /api/arco/operations con verificación de identidad**
- Épica: E-09 | HU: HU-015 | Prioridad: Must | Tamaño: L | Dependencias: T-029
- **Descripción:** `ArcoController` con verificación RN-037. Sin verificación → 401 + ARCO_IDENTITY_UNVERIFIED. Programa `scheduled_actions[]` (RN-031…RN-034).
- **Criterio:** Sin verificación → 401; con verificación → 200; los 4 tipos ARCO aceptados.

**T-031: Implementar cancelación ARCO con crypto-shred progresivo**
- Épica: E-09 | HU: HU-015 | Prioridad: Must | Tamaño: L | Dependencias: T-030
- **Descripción:** 24 h `deleted_at` + revocación KMS (RN-033-A/B). 30 d borrado físico (RN-033-C). Preservar `worker_hash` (RN-033-D) y `worker_legal_identity` (RN-033-E) 5 años.
- **Criterio:** 24 h `deleted_at` + revocación; 30 d BYTEA vacío; `worker_hash` preservado.

**T-032: Implementar constancia de pérdida de asistencia (RN-012-D)**
- Épica: E-08 | HU: HU-014 | Prioridad: Must | Tamaño: M | Dependencias: T-029
- **Descripción:** `LossConstancyService` genera `ATTENDANCE_LOSS_CONSTANCY` ≤ 48 h por trabajador afectado.
- **Criterio:** Constancia ≤ 48 h; PDF/CSV descargable.

**T-033: Implementar revocación de consentimiento sin cese laboral (RN-036)**
- Épica: E-09 | HU: HU-023 | Prioridad: Must | Tamaño: M | Dependencias: T-030
- **Descripción:** `ConsentRevocationService` (RN-036-A…D): suspende biometría, deriva a ZKTeco/manual, registra CONSENT_REVOKED, preserva logs.
- **Criterio:** Biometría suspendida; derivación sin afectar tareo; evento en audit.

**T-035: Implementar fallback manual MANUAL_CANCELLATION con aprobación ≤ 24 h (RN-030-D)**
- Épica: E-06 | HU: HU-022 | Prioridad: Must | Tamaño: M | Dependencias: T-004
- **Descripción:** `ManualMarkingService.register()` para trabajadores `PENDING_CONSENT` en sedes sin ZKTeco. Registra `event_type = MANUAL_CANCELLATION` con justificación (mín. 30 caracteres). Administrador aprueba/rechaza en ≤ 24 h; si no aprueba, deriva a REQUIRES_MANUAL_APPROVAL con alerta crítica.
- **Criterio:** Trabajador `PENDING_CONSENT` sin ZKTeco → MANUAL_CANCELLATION; aprobación ≤ 24 h; trabajador con consentimiento regularizado → ACTIVE.

**T-037: Implementar gobernanza multi-tenant de audit_logs (RN-035-A/B)**
- Épica: E-09 | HU: HU-024 | Prioridad: Must | Tamaño: L | Dependencias: T-023
- **Descripción:** `AuditGovernanceService`. (a) Reasignación de sede: eventos históricos conservan `tenant_id` original (RN-035-A); (b) Fusión de tenants: añade `merged_from_tenant_id` (RN-035-B); (c) Eliminación de tenant: retiene 5 años en almacenamiento cifrado con clave del tenant eliminado.
- **Criterio:** Reasignación preserva `tenant_id` original; fusión añade `merged_from_tenant_id`; eliminación retiene 5 años con clave del tenant.

### Testing

**T-034: E2E de crypto-shred progresivo (RN-033-A…E)**
- Épica: E-09 | HU: HU-015 | Prioridad: Must | Tamaño: L | Dependencias: T-031
- **Descripción:** Simula cancelación con fake timers. Verifica: (a) 24 h `deleted_at`; (b) 24 h clave revocada en KMS mock; (c) 30 d BYTEA vacío; (d) `worker_hash` 5 a; (e) `worker_legal_identity` intacto.
- **Criterio:** 5 sub-flujos pasan; KMS mock recibe RevokeKey.

**T-036: E2E de fallback manual sin ZKTeco (RN-030-D)**
- Épica: E-06 | HU: HU-022 | Prioridad: Must | Tamaño: M | Dependencias: T-035
- **Descripción:** `manual-marking.e2e-spec.ts`. Verifica: (a) trabajador `PENDING_CONSENT` sin ZKTeco → MANUAL_CANCELLATION con justificación; (b) aprobación del Administrador ≤ 24 h; (c) aprobación no en plazo → REQUIRES_MANUAL_APPROVAL; (d) trabajador con consentimiento regularizado → ACTIVE.
- **Criterio:** 4 escenarios pasan; `event_type` y plazos correctos.

**T-038: E2E de revocación de consentimiento sin cese (RN-036-A…D)**
- Épica: E-09 | HU: HU-023 | Prioridad: Must | Tamaño: M | Dependencias: T-033
- **Descripción:** `consent-revocation.e2e-spec.ts`. Verifica: (a) suspensión de biometría; (b) derivación a ZKTeco/manual; (c) `CONSENT_REVOKED` en audit; (d) `worker_id` real 5 a si biometría se mantiene; (e) sin penalización laboral.
- **Criterio:** 5 escenarios pasan; `worker_id` preservado 5 a.

**T-054: E2E de verificación de identidad ARCO (RN-037)**
- Épica: E-09 | HU: HU-015 | Prioridad: Must | Tamaño: M | Dependencias: T-030
- **Descripción:** `arco-identity.e2e-spec.ts`. Verifica: (a) firma digital válida → 200; (b) OTP correcto + biometría ≥ 0.92 → 200; (c) sin verificación → 401 + ARCO_IDENTITY_UNVERIFIED; (d) OTP expirado (> 5 min) → 401; (e) biometría < 0.92 → 401.
- **Criterio:** 5 escenarios pasan; `ARCO_IDENTITY_UNVERIFIED` registrado en audit.

---

## Trazabilidad RN ↔ Épica ↔ T-NNN

| Regla | Épica | Tareas que la implementan | Tarea de testing |
|---|---|---|---|
| RN-001A/B/C/D | E-01, E-15 | T-008, T-022 | T-047 |
| RN-002 / A/B/C | E-02 | T-005, T-009 | T-006 |
| RN-003 / A-1…A-6 / B / C | E-03 | T-010 | T-016 |
| RN-004 | E-03 | T-011 | T-048 |
| RN-005 / 005-G | E-04 | T-012 | T-049 |
| RN-005-H | E-04 | T-012 | T-049 |
| RN-006 | E-04 | T-021 | T-024, T-045 |
| RN-007 | E-05 | T-026, T-041 | T-050 |
| RN-008 / 013 | E-05, E-12 | T-026 | — |
| RN-009 | transversal | T-002 | — |
| RN-010 | E-06 | T-003 | — |
| RN-011 / A / B | E-07 | T-017 | T-019 |
| RN-012 / A / B | E-08 | T-013, T-014 | T-046, T-051 |
| RN-012-C/D | E-08 | T-032 | — |
| RN-014 / A | E-11 | T-039 | — |
| RN-016 | transversal | T-023 | T-024 |
| RN-017 / A | E-02 | T-005 | T-006 |
| RN-018 | E-07 | T-018 | T-052 |
| RN-019 | E-04 | T-012, T-040 | T-045, T-049 |
| RN-020 / A | E-03 | T-011 | T-048 |
| RN-021 | E-04 | T-012 | T-049 |
| RN-030 / A / B / C | E-06, E-09 | T-004 | T-053 |
| RN-030-D | E-06 | T-035 | T-036 |
| RN-031 / 032 / 034 | E-09 | T-030 | T-054 |
| RN-033 / A…E | E-09 | T-031 | T-034 |
| RN-035 / A / B | E-09 | T-031, T-037 | T-037 (e2e) |
| RN-036 / A…D | E-09 | T-033 | T-038 |
| RN-037 | E-09 | T-030 | T-054 |

---

## Cobertura de Épicas

| Épica | Estado | Tareas |
|---|---|---|
| E-01 Marcación Facial | ✅ Completa | T-007, T-008, T-015, T-022, T-047 |
| E-02 Geolocalización | ✅ Completa | T-005, T-006, T-009 |
| E-03 Anti-Fraude | ✅ Completa | T-010, T-011, T-016, T-048, T-057, T-058 |
| E-04 Offline y Sync | ✅ Completa | T-012, T-021, T-024, T-045, T-049 |
| E-05 Dashboard | ✅ Completa | T-025, T-026, T-041, T-050 |
| E-06 Personas y Sedes | ✅ Completa | T-003, T-004, T-035, T-036, T-053, T-059 |
| E-07 Integración ZKTeco | ✅ Completa | T-017, T-018, T-019, T-052 |
| E-08 Seguridad Dispositivo | ✅ Completa | T-013, T-014, T-032, T-046, T-051 |
| E-09 Cumplimiento ARCO | ✅ Completa | T-030, T-031, T-033, T-034, T-037, T-038, T-054, T-059 |
| E-10 Auth Supervisor | ✅ Completa | T-013 |
| E-11 Bloqueo Temporal | ✅ Completa | T-039, T-055, T-016 (ext) |
| E-12 Alertas Operativas | ✅ Completa | T-027, T-028 |
| E-13 Consolidación Tareo | ✅ Completa | T-040, T-060 |
| E-14 Exportación Reportes | ✅ Completa | T-023, T-060 |
| E-15 Re-verificación Server-Side | ✅ Completa | T-022 |

---

## Anexo C — Matriz de Trazabilidad Completa (RN ↔ Épica ↔ HU ↔ Componente ↔ Tarea ↔ Test)

| RN | Épica | HU | Componente | Tarea(s) | Test |
|---|---|---|---|---|---|
| RN-001A/B/C/D | E-01, E-15 | HU-001, HU-021 | A, C | T-008, T-022 | T-047 |
| RN-002 | E-02 | HU-002 | A, D | T-005, T-009 | T-006 |
| RN-002-A/B/C | E-02 | HU-003 | A | T-009 | T-006 |
| RN-003 / A-1…A-6 | E-03 | HU-004, HU-026 | A | T-010 | T-016 |
| RN-003-B/C | E-03 | HU-005 | A | T-010 | T-016 |
| RN-004 | E-03 | HU-006 | A, C | T-011 | T-048 |
| RN-005 | E-04 | HU-007 | A | T-012 | T-049 |
| RN-005-G | E-04 | HU-008 | A | T-012 | T-049 |
| RN-005-H | E-04 | HU-025 | A | T-012 | T-049 |
| RN-006 | E-04 | HU-007 | C | T-021 | T-024, T-045 |
| RN-007 | E-05 | HU-009 | D | T-026, T-041 | T-050 |
| RN-008 | E-05, E-12 | HU-009, HU-018 | E | T-026, T-027 | — |
| RN-009 | transversal | transversal | H | T-002 | — |
| RN-010 | E-06 | HU-011 | D | T-003 | — |
| RN-011 / A / B | E-07 | HU-012 | B | T-017 | T-019 |
| RN-012 / A / B | E-08 | HU-013 | A, F | T-013, T-014 | T-046, T-051 |
| RN-012-C/D | E-08 | HU-013, HU-014 | F | T-032 | — |
| RN-014 / A | E-11 | HU-017 | A | T-039 | — |
| RN-016 | transversal | HU-020 | C | T-023 | T-024 |
| RN-017 / A | E-02 | HU-002 | A, D | T-005 | T-006 |
| RN-018 | E-07 | HU-012 | B, C | T-018 | T-052 |
| RN-019 | E-04 | HU-007, HU-019 | A, D | T-012, T-040 | T-045, T-049 |
| RN-020 / A | E-03 | HU-006 | A | T-011 | T-048 |
| RN-021 | E-04 | HU-007 | A | T-012 | T-049 |
| RN-030 / A / B / C | E-06, E-09 | HU-010 | D | T-004 | T-053 |
| RN-030-D | E-06 | HU-022 | F | T-035 | T-036 |
| RN-031 / 032 / 034 | E-09 | HU-015 | F | T-030 | T-054 |
| RN-033 / A…E | E-09 | HU-015 | F | T-031 | T-034 |
| RN-035 / A / B | E-09 | HU-015, HU-024 | F | T-031, T-037 | T-037 |
| RN-036 / A…D | E-09 | HU-023 | F | T-033 | T-038 |
| RN-037 | E-09 | HU-015 | F | T-030 | T-054 |

---

## Validación del DAG

- **Sin ciclos:** verificado por recorrido topológico.
- **Tareas sin dependencias:** solo `T-001`.
- **Paralelizables tras T-001 + T-002:** T-007, T-017, T-020, T-025, T-029, T-003, T-004, T-005, T-041, T-042, T-043, T-044.
- **Profundidad máxima:** 9 niveles (`T-001 → T-002 → T-007 → T-010 → T-015 → T-020 → T-021 → T-023 → T-016`).
- **Esfuerzo total:** ~54 h (16 L + 34 M + 4 S).
- **Equipo mínimo:** 3 agentes en paralelo.

---

**Fin del documento `08-backlog.md` v1.1.0**

**Cierre de hallazgos QA-VAL (10-validation-log.md v1.0.0):**
- 🔴 QA-VAL-001 → cerrado con T-039 (extendida), T-055 (nueva), T-016 (extendida)
- 🔴 QA-VAL-002 → cerrado con RN-021-A, RNF-008-A, HU-007 (extendida), T-012 (extendida), T-049 (extendida), T-056 (nueva)
- 🔴 QA-VAL-004 → cerrado con T-010 (extendida), T-058 (nueva), T-016 (extendida)
- 🟡 QA-VAL-003 → cerrado con T-010 (extendida), T-057 (nueva)
- 🟢 QA-VAL-005 → cerrado con T-059 (nueva)
- 🟢 QA-VAL-006 → cerrado con T-060 (nueva)