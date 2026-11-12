# Reporte de Auditoría QA de Backlog — Módulo Operativo y Control de Asistencia Omnicanal

**Versión auditada:** Spec Pack v2.2.1
**Fecha:** 11 de Noviembre de 2026
**Auditor:** QA Architect Senior (Framework BMAD)
**Documentos auditados:** `./00-constitucion.md` a `./08-backlog.md`

---

## Resumen ejecutivo

El Spec Pack v2.2.1 junto con la versión v1.1.0 de Historias de Usuario (`06-user-stories.md`), Arquitectura (`07-arquitecture.md`) y Backlog (`08-backlog.md`) alcanza un nivel de **trazabilidad y completitud del 100 %** de extremo a extremo. Se ha verificado la cobertura bidireccional entre las 67 reglas de negocio EARS (RN-001A…RN-037), las 15 épicas (E-01…E-15), las 26 Historias de Usuario (HU-001…HU-026), los 6 componentes de arquitectura (A…F), los 7 endpoints HTTP/ADMS, los 5 ADRs y las 54 tareas del backlog (T-001…T-054).

Todos los 15 hallazgos de iteraciones previas (QA-C2-001 a QA-C2-015) han sido formalmente cerrados mediante la incorporación de HUs oficiales (HU-022…HU-026), componentes responsables en arquitectura y 20 nuevas tareas específicas en el backlog (T-035…T-054), incluyendo 10 tareas de testing E2E temáticas. **No existen hallazgos críticos ni mayores abiertos.** El Spec Pack se declara **100 % APTO** para iniciar la fase de construcción de software.

| Métrica | Valor |
|---|---|
| Reglas Must cubiertas por HU | 67/67 (100 %) |
| HUs Must cubiertas por componente | 18/18 (100 %) |
| HUs Totales (Must + Should) cubiertas por componente | 26/26 (100 %) |
| Épicas cubiertas por componente + tarea | 15/15 (100 %) |
| Componentes con al menos 1 tarea | 6/6 (100 %) |
| Tareas con trazabilidad completa (épica + HU + regla) | 54/54 (100 %) |
| Endpoints con trazabilidad a HU | 7/7 (100 %) |
| ADRs con trazabilidad a épica + regla | 5/5 (100 %) |
| Reglas Must con test E2E asociado | 67/67 (100 %) |
| Hallazgos críticos | 0 |
| Hallazgos mayores | 0 |
| Sugerencias | 3 |

---

## Hallazgos

### 🔴 Críticos

*No se identificaron hallazgos críticos.* Todos los hallazgos críticos previos (QA-C2-001 sobre RN-030-D, QA-C2-002 sobre RN-035-A/B y QA-C2-003 sobre RN-036) fueron completamente resueltos en `06-user-stories.md` (HU-022, HU-023, HU-024), `07-arquitecture.md` (Componente F y §3.4/§3.7) y `08-backlog.md` (T-035, T-036, T-037, T-038).

---

### 🟡 Mayores

*No se identificaron hallazgos mayores.* Todos los hallazgos mayores previos (QA-C2-004 a QA-C2-011) fueron cerrados mediante la formalización de HUs (HU-025, HU-026), la adición de tareas de infraestructura/features (T-039 a T-044) y la creación de 10 tareas de pruebas E2E automatizadas (T-045 a T-054).

---

### 🟢 Sugerencias

**QA-C2-016: Automatización de Pruebas de Carga RNF-003 en Pipeline CI/CD**
- **Severidad:** Sugerencia
- **Descripción:** Aunque la tarea T-019 define las pruebas de carga k6/Artillery para el endpoint ADMS a $500\text{ RPS}$ con ráfagas de $1,000\text{ RPS}$, no se especifica su ejecución periódica automatizada en el pipeline de integración continua para detectar regresiones de latencia antes de producción.
- **Archivos afectados:** `./08-backlog.md` (T-019), `.github/workflows/ci.yml`.
- **Acción recomendada:** Adicionar un paso programado (ej. *nightly build*) en `.github/workflows/ci.yml` que ejecute el script k6 de T-019 contra un entorno de staging y emita alertas en Slack si $p95 > 100\text{ ms}$ o $p99 > 250\text{ ms}$.

**QA-C2-017: Documentación Explícita de la Política de Rotación de Llaves KMS para Crypto-Shredding**
- **Severidad:** Sugerencia
- **Descripción:** Las tareas T-031 y T-034 implementan la revocación atómica e inmediata de la clave AES-256 en KMS a las 24 h de una solicitud ARCO (RN-033-B). Se sugiere normar formalmente el procedimiento de rotación de la *Master Key* de KMS para datos no revocados de trabajadores activos.
- **Archivos afectados:** `./07-arquitecture.md` (§3.4), `./08-backlog.md` (T-029).
- **Acción recomendada:** Crear el artefacto `docs/kms-key-rotation-policy.md` especificando la rotación anual de la *Master Key* con re-encriptación de sobres (*envelope encryption*) sin alterar la revocación atómica por trabajador.

**QA-C2-018: Matriz de Pruebas Exploratorias Offline en Hardware Móvil de Gama Media/Baja**
- **Severidad:** Sugerencia
- **Descripción:** T-049 y T-012 aseguran la suite E2E de sincronización offline en emuladores. Sin embargo, optimizadores agresivos de batería en dispositivos Android físicos de gama media/baja (OEMs como Xiaomi, Transsion, Samsung) podrían suspender los servicios nativos en segundo plano (`WorkManager`) antes de la ventana de sincronización.
- **Archivos afectados:** `./08-backlog.md` (T-012, T-049).
- **Acción recomendada:** Adicionar una matriz de pruebas exploratorias en dispositivos físicos (`docs/mobile-oem-testing-matrix.md`) para verificar que el banner de pendientes (RN-005-G) fuerce la sincronización en primer plano al abrir la aplicación si el SO suspendió el proceso en segundo plano.

---

## Matriz de trazabilidad

### Regla → Épica → HU → Componente → Tarea → Test E2E

| Regla (RN-XXX) | Épica | HU | Componente (A…F) | Tarea(s) (T-NNN) | Test E2E |
|---|---|---|---|---|---|
| **RN-001A** | E-01 | HU-001 | A — Mobile Application | T-008 | T-047 |
| **RN-001B** | E-01 | HU-001 | A — Mobile Application | T-008 | T-047 |
| **RN-001C** | E-01, E-15 | HU-001, HU-021 | A — Mobile App, C — Async Engine | T-008, T-022 | T-047 |
| **RN-001D** | E-01 | HU-001 | A — Mobile Application | T-008 | T-047 |
| **RN-002** | E-02 | HU-002 | A — Mobile App, D — Attendance Core | T-005, T-009 | T-006 |
| **RN-002-A** | E-02 | HU-003 | A — Mobile Application | T-009 | T-006 |
| **RN-002-B** | E-02 | HU-003 | A — Mobile App, D — Attendance Core | T-009, T-040 | T-006 |
| **RN-002-C** | E-02 | HU-003 | A — Mobile Application | T-009 | T-006 |
| **RN-003** | E-03 | HU-004 | A — Mobile Application | T-010 | T-016 |
| **RN-003-A / A-1…A-5** | E-03 | HU-004 | A — Mobile Application | T-010 | T-016 |
| **RN-003-A-6** | E-03 | HU-026 | A — Mobile Application | T-010 | T-016 |
| **RN-003-B** | E-03 | HU-005 | A — Mobile Application | T-010 | T-016 |
| **RN-003-C** | E-03 | HU-005 | A — Mobile Application | T-010 | T-016 |
| **RN-004** | E-03 | HU-006 | A — Mobile App, C — Async Engine | T-011 | T-048 |
| **RN-005** | E-04 | HU-007 | A — Mobile Application | T-012 | T-049 |
| **RN-005-G** | E-04 | HU-008 | A — Mobile Application | T-012 | T-049 |
| **RN-005-H** | E-04 | HU-025 | A — Mobile Application | T-012 | T-049 |
| **RN-006** | E-04 | HU-007 | C — Async Processing Engine | T-021 | T-024, T-045 |
| **RN-007** | E-05 | HU-009 | D — Attendance Core Service | T-026, T-041 | T-050 |
| **RN-008** | E-05, E-12 | HU-009, HU-018 | E — Monitoring Dashboard | T-026, T-027 | T-050 |
| **RN-009** | transversal | HU-009 | F — Compliance & KMS Service | T-002 | T-049 |
| **RN-010** | E-06 | HU-011 | D — Attendance Core Service | T-003 | T-006 |
| **RN-011** | E-07 | HU-012 | B — ADMS Receiver, C — Async Engine | T-017 | T-019 |
| **RN-011-A** | E-07 | HU-012 | B — ADMS Ingestion Receiver | T-017 | T-019 |
| **RN-011-B** | E-07 | HU-012 | B — ADMS Ingestion Receiver | T-017 | T-019 |
| **RN-012** | E-08, E-10 | HU-013, HU-016 | A — Mobile Application | T-013 | T-051 |
| **RN-012-A** | E-08 | HU-013 | A — Mobile Application | T-014 | T-046, T-051 |
| **RN-012-B** | E-08 | HU-013 | A — Mobile Application | T-013 | T-051 |
| **RN-012-C** | E-08 | HU-013 | F — Compliance & KMS Service | T-032 | T-046 |
| **RN-012-D** | E-08 | HU-014 | F — Compliance & KMS Service | T-032 | T-046 |
| **RN-013** | E-05 | HU-009 | E — Monitoring Dashboard | T-026 | T-050 |
| **RN-014** | E-11 | HU-017 | A — Mobile Application | T-039 | T-016 |
| **RN-014-A** | E-11 | HU-017 | A — Mobile Application | T-039 | T-016 |
| **RN-015** | E-08 | HU-015 | F — Compliance & KMS Service | T-002 | T-047 |
| **RN-016** | transversal | HU-020 | C — Async Processing Engine | T-023 | T-024 |
| **RN-017** | E-02 | HU-002 | C — Async Processing Engine | T-005 | T-006 |
| **RN-017-A** | E-02 | HU-002 | C — Async Processing Engine | T-005 | T-006 |
| **RN-018** | E-07 | HU-012 | B — ADMS Receiver, C — Async Engine | T-018 | T-052 |
| **RN-019** | E-04, E-13 | HU-007, HU-019 | A — Mobile App, D — Attendance Core | T-012, T-040 | T-045, T-049 |
| **RN-020** | E-03 | HU-006 | A — Mobile Application | T-011 | T-048 |
| **RN-020-A** | E-03 | HU-006 | A — Mobile Application | T-011 | T-048 |
| **RN-021** | E-04 | HU-007 | A — Mobile Application | T-012 | T-049 |
| **RN-030** | E-06, E-09 | HU-010 | D — Attendance Core Service | T-004 | T-053 |
| **RN-030-A** | E-06, E-09 | HU-010 | D — Attendance Core Service | T-004 | T-053 |
| **RN-030-B** | E-06, E-09 | HU-010 | D — Attendance Core Service | T-004 | T-053 |
| **RN-030-C** | E-06, E-09 | HU-010 | D — Attendance Core Service | T-004 | T-053 |
| **RN-030-D** | E-06 | HU-022 | F — Compliance & KMS Service | T-035 | T-036 |
| **RN-031** | E-09 | HU-015 | F — Compliance & KMS Service | T-030 | T-054 |
| **RN-032** | E-09 | HU-015 | F — Compliance & KMS Service | T-030 | T-054 |
| **RN-033** | E-09 | HU-015 | F — Compliance & KMS Service | T-031 | T-034 |
| **RN-033-A…E** | E-09 | HU-015 | F — Compliance & KMS Service | T-031 | T-034 |
| **RN-034** | E-09 | HU-015 | F — Compliance & KMS Service | T-030 | T-054 |
| **RN-035** | E-09 | HU-015 | F — Compliance & KMS Service | T-031, T-037 | T-034 |
| **RN-035-A** | E-09 | HU-024 | F — Compliance & KMS Service | T-037 | T-037 |
| **RN-035-B** | E-09 | HU-024 | F — Compliance & KMS Service | T-037 | T-037 |
| **RN-036 / A…D** | E-09 | HU-023 | F — Compliance & KMS Service | T-033 | T-038 |
| **RN-037** | E-09 | HU-015 | F — Compliance & KMS Service | T-030 | T-054 |

---

### Cobertura MoSCoW

| Elemento MoSCoW | HUs que lo cubren | Componente | Tareas | Estado |
|---|---|---|---|---|
| **M-01 Marcación Facial** | HU-001, HU-021 | A, C | T-008, T-015, T-022, T-047 | ✅ Completo |
| **M-02 Geolocalización GPS** | HU-002, HU-003 | A, C, D | T-005, T-006, T-009 | ✅ Completo |
| **M-03 Anti-Fraude & NTP** | HU-004, HU-005, HU-006, HU-026 | A, C | T-010, T-011, T-016, T-048 | ✅ Completo |
| **M-04 Offline-First & Sync** | HU-007, HU-008, HU-025 | A, C | T-012, T-020, T-021, T-024, T-045, T-049 | ✅ Completo |
| **M-05 Dashboard Mapa GPS** | HU-009 | D, E | T-025, T-026, T-041, T-050 | ✅ Completo |
| **M-06 Gestión Personas/Sedes** | HU-010, HU-011, HU-022 | D, F | T-003, T-004, T-035, T-036, T-053 | ✅ Completo |
| **M-07 Integración ZKTeco** | HU-012 | B, C | T-017, T-018, T-019, T-052 | ✅ Completo |
| **M-08 Vinculación Dispositivo** | HU-013, HU-014 | A, F | T-013, T-014, T-032, T-046, T-051 | ✅ Completo |
| **M-09 Cumplimiento ARCO** | HU-015, HU-023, HU-024 | F | T-030, T-031, T-033, T-034, T-037, T-038, T-054 | ✅ Completo |
| **M-10 Autenticación Supervisor** | HU-016 | A | T-013, T-046 | ✅ Completo |
| **S-01 Alertas Tiempo Real** | HU-018 | E | T-027, T-028 | ✅ Completo |
| **S-02 Consolidación Tareo** | HU-019 | D | T-040 | ✅ Completo |
| **S-03 Exportación Reportes** | HU-020 | C, E | T-023 | ✅ Completo |
| **S-04 Server-Side Reverification** | HU-021 | C | T-022 | ✅ Completo |
| **S-05 Bloqueo Temporal** | HU-017 | A | T-039 | ✅ Completo |

---

### Cobertura de Épicas

| Épica | Componente(s) | Tareas | Estado |
|---|---|---|---|
| **E-01 Marcación Facial Móvil** | A — Mobile App, C — Async Engine | T-007, T-008, T-015, T-022, T-047 | ✅ Completo |
| **E-02 Geolocalización y Geocerca** | A — Mobile App, D — Attendance Core | T-005, T-006, T-009 | ✅ Completo |
| **E-03 Anti-Fraude** | A — Mobile App | T-010, T-011, T-016, T-048 | ✅ Completo |
| **E-04 Offline y Sincronización** | A — Mobile App, C — Async Engine | T-012, T-020, T-021, T-024, T-045, T-049 | ✅ Completo |
| **E-05 Dashboard de Monitoreo GPS** | D — Attendance Core, E — Dashboard | T-025, T-026, T-041, T-050 | ✅ Completo |
| **E-06 Gestión Personas y Sedes** | D — Attendance Core, F — Compliance KMS | T-003, T-004, T-035, T-036, T-053 | ✅ Completo |
| **E-07 Integración ZKTeco** | B — ADMS Receiver, C — Async Engine | T-017, T-018, T-019, T-052 | ✅ Completo |
| **E-08 Seguridad de Dispositivo** | A — Mobile App, F — Compliance KMS | T-013, T-014, T-032, T-046, T-051 | ✅ Completo |
| **E-09 Cumplimiento ARCO y Consentimiento** | F — Compliance & KMS Service | T-030, T-031, T-033, T-034, T-037, T-038, T-054 | ✅ Completo |
| **E-10 Autenticación del Supervisor** | A — Mobile Application | T-013, T-046 | ✅ Completo |
| **E-11 Bloqueo Temporal** | A — Mobile Application | T-039 | ✅ Completo |
| **E-12 Alertas Operativas** | E — Monitoring Dashboard | T-027, T-028 | ✅ Completo |
| **E-13 Consolidación de Tareo** | D — Attendance Core Service | T-040 | ✅ Completo |
| **E-14 Exportación de Reportes** | C — Async Engine, E — Dashboard | T-023 | ✅ Completo |
| **E-15 Re-verificación Server-Side** | C — Async Processing Engine | T-022 | ✅ Completo |

---

### Cobertura de Principios Constitucionales

| Principio | Elemento que lo cumple (archivo/sección) | Estado |
|---|---|---|
| **P-001 Idioma y Stack** | `./00` §1; `./07` §1.3 (NestJS, PostgreSQL 15+, Redis/BullMQ, React/Next.js, Leaflet, React Native/Capacitor; identificadores en inglés, docs en español neutro); `./08` T-001. | ✅ Cumplido |
| **P-002 Dashboard GPS sin Login** | `./00` §2; `./02` M-05; `./03` RN-007; `./07` §4.3, ADR-003; `./08` T-026, T-041, T-050 (`POST /api/auth/environment-session`, JWT RS256 TTL 12 h). | ✅ Cumplido |
| **P-003 Cifrado y Protección Ley 29733** | `./00` §3; `./03` RN-009; `./07` §3.3 (AES-256-GCM en reposo, TLS 1.3 obligatorio en tránsito, KeyStore/Keychain nativo); `./08` T-002, T-007, T-029. | ✅ Cumplido |
| **P-004 Integración ZKTeco Desacoplada** | `./00` §4; `./02` M-07; `./03` RN-011/A/B; `./07` §2.2, §4.2, ADR-005 ($p95 \le 100\text{ ms}$ a $500\text{ RPS}$, deduplicación 60 s, rate limit 10 RPS); `./08` T-017, T-019. | ✅ Cumplido |
| **P-005 Marcación Facial Anti-Fraude** | `./00` §5; `./02` M-01, M-03; `./03` RN-001A ($score \ge 0.95$), RN-001B ($score \ge 0.92$), RN-003 (Android native provider), RN-003-A (consenso iOS 3 muestras), RN-003-A-6 (< 2 señales), RN-003-B/C (root/jailbreak), RN-004/RN-020/RN-020-A (NTP primario y fallbacks); `./07` ADR-002; `./08` T-008, T-010, T-011, T-016, T-047, T-048. | ✅ Cumplido |
| **P-006 Operación Offline-First** | `./00` §6; `./02` M-04; `./03` RN-005 (almacenamiento cifrado $\le 5,000$ reg o $200\text{ MB}$), RN-005-G (banner de pendientes), RN-005-H (reinstalación), RN-019 (expiración 72 h); `./07` §6.4; `./08` T-012, T-045, T-049. | ✅ Cumplido |
| **P-007 Restricciones Agentes IA** | `./00` §7; `./08` T-001 (`forbidden-files.yml` bloquea PRs que alteren `migrations/` o desactiven tests), T-044 (Vault/Secrets Manager). | ✅ Cumplido |
| **P-008 Suite E2E y Datos Sintéticos** | `./00` §8; `./03` RN-015; `./07` §6.5 ($\ge 80\%$ backend, $\ge 70\%$ app); `./08` T-006, T-016, T-019, T-024, T-034, T-036…T-038, T-045…T-054. | ✅ Cumplido |
| **P-009 Auditabilidad Inmutable** | `./00` §9; `./03` RN-016; `./07` §3.1 (tabla `audit_logs` con los 8 campos obligatorios y enum cerrado de 22 `event_type`); `./08` T-002, T-023, T-024. | ✅ Cumplido |
| **P-010 ARCO, Consentimiento y Retención** | `./00` §10; `./02` M-09; `./03` RN-030/A/B/C/D, RN-031…RN-035/A/B, RN-036/A…D, RN-037 (verificación de identidad ARCO); `./07` §3.4, §4.4, ADR-004 (Crypto-Shredding atómico a 24 h, borrado físico a 30 d, `worker_hash` y legajo a 5 años); `./08` T-030, T-031, T-033, T-034, T-037, T-038, T-054. | ✅ Cumplido |

---

## Plan de acción priorizado

1. **Críticos (resolver antes de iniciar desarrollo):** 
   - *0 hallazgos abiertos.* El Spec Pack v2.2.1 y la suite de arquitectura/backlog están **100 % listos y autorizados** para dar inicio a la fase de construcción de código (Sprint 1).

2. **Mayores (resolver antes de cerrar Sprint 1):** 
   - *0 hallazgos abiertos.*

3. **Sugerencias (backlog de mejora continua QA/DevOps):** 
   - **QA-C2-016:** Configurar el paso *nightly build* de pruebas de carga k6 para el endpoint ADMS ZKTeco en `.github/workflows/ci.yml`.
   - **QA-C2-017:** Redactar la directiva `docs/kms-key-rotation-policy.md` para normar la rotación anual de la *Master Key* de KMS sin alterar el flujo de revocación atómica por ARCO.
   - **QA-C2-018:** Crear la matriz de pruebas exploratorias en dispositivos físicos de gama media/baja (`docs/mobile-oem-testing-matrix.md`) para verificar el comportamiento de sincronización cuando el SO suspende tareas en segundo plano.