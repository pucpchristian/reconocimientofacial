# Documento de Alcance: Módulo Operativo y Control de Asistencia Omnicanal

**Proyecto:** Módulo Operativo y Control de Asistencia Omnicanal (Ecosistema RH / ERP)
**Marco Normativo:** Ley N.° 29733, Ley N.° 27269, Normativa MTPE/SUNAT
**Documentos de Referencia:** `00-constitucion.md`, `01-project-brief.md`, `03-reglas-de-negocio.md`, `04-glosario.md`
**Versión:** 2.2.2
**Fecha:** 10 de Noviembre de 2026
**Estado:** Aprobado para Diseño de Software
**Cambios v2.2.2:** M-04 referencia RN-021-A; añadida RNF-008-A (alerta al 80 %). Aplicadas observaciones QA-VAL-002.
**Cambios v2.2.1:** S-04 elevada a M-10. M-10 previa degradada a S-05. Añadidas RN-037 (identidad ARCO), RN-030-C (prioridad plazos), RN-030-D (fallback sin ZKTeco) al scope. Aplicados 18 hallazgos QA v2.2.0.
**Cambios v2.2.0:** Arquitectura removida.
**Cambios v2.0.0:** App Móvil Nativa/Híbrida; identificadores en inglés.

---

## 1. IN-SCOPE (MoSCoW)

### Must-have (M) — Máximo 10

* **M-01: Marcación Facial Móvil con Prueba de Vida:** `liveness_score ≥ 0.95`, gesto ≤ 5 s, `match_score ≥ 0.92` (1:1), `ntp_timestamp` de servidor.
* **M-02: Geolocalización GPS y Validación de Geocerca (50 m):** Haversine; re-captura 3 intentos; fallback `REQUIRES_MANUAL_APPROVAL` (RN-002-A/B/C, RN-017-A).
* **M-03: Detección y Bloqueo Anti-Fraude:** proveedor nativo (Android); heurísticas con consenso y **mínimo 2 señales testeables** (iOS, RN-003-A-6); detección root/jailbreak; NTP como fuente primaria con fallbacks (RN-003, RN-003-A-1…A-6, RN-003-B, RN-003-C, RN-004, RN-020, RN-020-A).
* **M-04: Recepción Backend y Sincronización Offline-First:** cifrado local respaldado por hardware; sincronización en segundo plano; banner de pendientes (RN-005, RN-005-G); **alerta de saturación al 80 % (RN-021-A)**; reinstalación de app (RN-005-H).
* **M-05: Dashboard de Monitoreo en Mapa GPS (Sin Login Tradicional):** token de sesión firmado; filtros; radio 50 m; pines verde/rojo (RN-007, RN-008, RN-013).
* **M-06: Gestión Básica de Personas y Sedes:** CRUD trabajadores (foto, vector) y sedes (WGS84).
* **M-07: Integración con Hardware ZKTeco:** recepción asíncrona; p95 ≤ 100 ms, p99 ≤ 250 ms, 500 RPS, ráfagas 1,000 RPS ≤ 10 s; rate-limit; payload size (RN-011, RN-011-A, RN-011-B, RN-018).
* **M-08: Vinculación Estricta de Dispositivo:** `device_uuid`; primario + backup (RN-012-A); UUID desconocido (RN-012-B); pérdida (RN-012-C, RN-012-D).
* **M-09: Cumplimiento ARCO y Consentimiento:** consentimiento versionado (RN-030, 030-A, 030-B, 030-C); **verificación de identidad ARCO (RN-037)**; operaciones ARCO (RN-031…RN-034); retención y crypto-shred con revocación inmediata (RN-033-A…E, RN-035); revocación sin cese (RN-036-A…D); fallback sin ZKTeco (RN-030-D).
* **M-10: Autenticación Primaria del Supervisor (elevada desde S-04):** PIN + biometría de plataforma al inicio de jornada, como **prerrequisito obligatorio** de RN-012. Sin esta autenticación, el `device_uuid` no autoriza marcaciones.

### Should-have (S) — Máximo 5

* **S-01: Alertas Operativas en Tiempo Real** (fuera de geocerca, fraude detectado).
* **S-02: Consolidación Automática de Tareos para Pre-Planilla.**
* **S-03: Exportación de Reportes Auditables (CSV/PDF).**
* **S-04: Feature Flag `server_side_reverification`** (re-verificación biométrica en servidor cuando el tenant lo active, RN-001C).
* **S-05: Bloqueo Temporal de Terminal** (30 min tras 3 intentos fallidos en 15 min, RN-014, RN-014-A). *Degradado desde M-10 en v2.2.1 porque RN-014 es SHOULD y no es requisito operativo del flujo principal.*

### Could-have (C) — Máximo 4

* **C-01: Programación de Mallas Operativas Rotativas.**
* **C-02: Gestión de Retenes y Cubrimientos al Instante.**
* **C-03: Reconocimiento Facial Multi-Persona en Cuadrilla.**
* **C-04: Firma Digital del Consentimiento Informado (Ley N.° 27269).**

---

## 2. OUT-OF-SCOPE (Exclusiones Explícitas)

1. **Cálculo de Planilla Final y Tributación:** Módulo Core RRHH.
2. **Firma Digital de Contratos y Boletas:** Módulo de Gestión Documental.
3. **Gestión de Inventario de Hardware Biométrico y EPPs:** Módulo de Inventario.
4. **Publicación en App Stores:** MVP distribuido como build interno o vía MDM.
5. **Autenticación con Usuario/Contraseña para el Mapa GPS:** solo token de sesión.
6. **Soporte BYOD:** solo celular corporativo del Supervisor.
7. **Motor de reconocimiento facial propietario:** SDKs de plataforma o proveedor NIST FRVT.

---

## 3. MVP — Flujo Funcional End-to-End

El MVP digitaliza la asistencia del personal en campo mediante celular corporativo con App Móvil. El **Supervisor de Ronda** se autentica con PIN + biometría de plataforma (M-10) y registra la marcación capturando foto con liveness en vivo y validando GPS dentro de los 50 m de la geocerca; si `gps_accuracy > 25 m`, la App permite máximo 3 re-capturas y, agotadas, deriva a `REQUIRES_MANUAL_APPROVAL` (RN-002-A/B/C). La App almacena cifrado localmente cuando no hay red (Offline-First), fuerza `ntp_timestamp` del servidor con fallbacks explícitos (RN-020, RN-020-A), bloquea intentos de ubicación simulada (proveedor nativo en Android; heurísticas con consenso en iOS, RN-003-A, con **mínimo 2 señales testeables**, RN-003-A-6), y vincula el dispositivo por `device_uuid` con primario + backup (RN-012-A). El **Administrador de Operaciones** abre el mapa GPS vía token de sesión y filtra por fechas y trabajador. En paralelo, el sistema captura consentimiento versionado, expone ARCO con **verificación de identidad (RN-037)** y SLA legal, y aplica retención diferenciada con crypto-shred de **revocación inmediata de clave (RN-033-B)**. La integración ZKTeco procesa biométricos de forma asíncrona con capacidad p95 ≤ 100 ms a 500 RPS y conmutación a App Móvil ante caída de heartbeat (RN-018).

---

## 4. Supuestos (Assumptions)

1. Supervisor con smartphone Android 8.0+ / iOS 14+.
2. Sedes con coordenadas WGS84 registradas.
3. Cámaras con resolución mínima para liveness y extracción de vector.
4. Red permite consultas NTP sin bloqueos.
5. App distribuida vía MDM o build interno.
6. DPO designado y banco de datos registrado ante ANPD.
7. Parque máximo simultáneo de 500 relojes ZKTeco (revisable anualmente).

---

## 5. Restricciones (Constraints)

1. **Ley N.° 29733:** AES-256 en reposo; TLS 1.3 en tránsito (TLS 1.2 no aceptado); almacenamiento local móvil cifrado; consentimiento/ARCO/retención (PRINCIPIO-010); verificación de identidad ARCO (RN-037).
2. **Idioma:** código en inglés; docs/tests/UI en Español Neutro (PRINCIPIO-001).
3. **Tolerancia Geográfica:** 50 m (Haversine, R = 6,371,000 m) con epsilon 0.01 m (RN-017-A).
4. **Datos Sintéticos:** prohibido usar fotos o coordenadas reales en pruebas.
5. **Capacidad ZKTeco:** p95 ≤ 100 ms, p99 ≤ 250 ms a 500 RPS (RNF-003).

---

## 6. Dependencias (Dependencies)

1. Servidor NTP oficial (o pool).
2. Proveedor de mapas web.
3. Motor de visión por computadora para liveness (en dispositivo o servidor).
4. Base de datos relacional y mecanismo de procesamiento asíncrono (en arquitectura).
5. Gestión empresarial de dispositivos (MDM).
6. Servicio de gestión de claves (KMS) para crypto-shred.
7. Certificación NIST FRVT del proveedor de motor facial (si aplica).
8. Proveedor de OTP al correo corporativo (para activación de backup y ARCO).

---

## 7. Requisitos No Funcionales (RNF)

| ID | Categoría | Requisito | Verificación |
|---|---|---|---|
| **RNF-001** | Rendimiento API marcación | p95 ≤ 500 ms (excluye `server_side_reverification`) | Prueba de carga a 200 RPS |
| **RNF-002** | Rendimiento Dashboard | Render de pines p95 ≤ 1.5 s con 10,000 marcaciones y 50 usuarios | Prueba de carga |
| **RNF-003** | Capacidad ZKTeco | p95 ≤ 100 ms; p99 ≤ 250 ms a 500 RPS; ráfagas 1,000 RPS ≤ 10 s; 500 `serial_number` concurrentes | Prueba de carga (500 SN × 1,000 registros); 0 errores 5xx |
| **RNF-004** | Throughput de cola | ≥ 1,000 marcaciones/s sin backlog | Prueba de carga + métricas |
| **RNF-005** | Disponibilidad | 99.9 % mensual | Monitoreo de uptime |
| **RNF-006** | Latencia `server_side_reverification` | p95 ≤ 500 ms con flag activo | Prueba de carga |
| **RNF-007** | Cobertura de pruebas | ≥ 80 % backend, ≥ 70 % App Móvil | Herramienta de cobertura |
| **RNF-008** | Almacenamiento local | 5,000 registros o 200 MB (el menor) | Test de saturación E2E |
| **RNF-008-A** | Alerta de saturación local | La App Móvil DEBE mostrar alerta visual roja parpadeante cuando el almacenamiento local supere el 80 % de su capacidad (4,000 de 5,000 registros, o 160 de 200 MB, el menor) | Test E2E de saturación + revisión UI |
| **RNF-009** | Seguridad HTTP | TLS 1.3 obligatorio; HSTS ≥ 6 meses; CSP estricto | Escaneo de seguridad |
| **RNF-010** | Auditoría inmutable | Solo inserción; sin modificación ni borrado; 5 años | Test de integridad + revisión DPO |