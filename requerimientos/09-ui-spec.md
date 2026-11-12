# Especificación de Interfaz de Usuario (UI Specification) — Módulo Operativo y Control de Asistencia Omnicanal

**Versión:** 1.0.1
**Fecha:** 11 de Noviembre de 2026
**Estado:** Aprobado para Desarrollo Frontend (React Native Móvil / Next.js Web)
**Referencias del Spec Pack:**
- `./00-constitucion.md` (v2.2.2)
- `./03-reglas-de-negocio.md` (v2.2.2)
- `./06-user-stories.md` (v1.1.1)
- `./07-architecture.md` (v1.0.1)
- `./08-backlog.md` (v1.2.0)
- `./10-validation-log.md` (v1.0.0)
**Cambios v1.0.1:** QA-VAL-001 (temporizador `MM:SS` P-001), QA-VAL-002 (alerta 80 % P-003), QA-VAL-003 (modal remediación P-002), QA-VAL-004 (indicador "Muestra X/3" P-002), QA-VAL-005 (badge Ley 29733 P-005), QA-VAL-006 (badge SHA-256 P-007).


---

## 1. Inventario de Pantallas

| ID Pantalla | Nombre de la Pantalla | Actor Principal | HUs Implementadas | Tareas del Backlog | Prioridad MoSCoW |
|-------------|-----------------------|-----------------|-------------------|--------------------|------------------|
| **P-001** | Autenticación Primaria de Supervisor Móvil | Supervisor de Ronda | HU-016, HU-017 | T-013, T-039, T-046, **T-055** | **Must Have** |
| **P-002** | Captura Facial Biométrica y Prueba de Vida Móvil | Supervisor de Ronda | HU-001, HU-004, HU-005, HU-006, HU-026 | T-008, T-010, T-011, T-016, T-047, T-048, **T-057**, **T-058** | **Must Have** |
| **P-003** | Confirmación de Marcación y Resultado de Geocerca Móvil | Supervisor de Ronda | HU-002, HU-003, HU-007, HU-008, HU-025 | T-005, T-006, T-009, T-012, T-045, T-049 | **Must Have** |
| **P-004** | Dashboard Web de Monitoreo GPS Operativo | Administrador de Operaciones | HU-009, HU-018 | T-025, T-026, T-027, T-028, T-041, T-050 | **Must Have** |
| **P-005** | Módulo de Enrolamiento, Consentimiento y Solicitud ARCO | Admin de RRHH / Compliance Officer | HU-010, HU-015, HU-022, HU-023, HU-024 | T-003, T-030, T-031, T-033…T-038, T-053, T-054, **T-059** | **Must Have** |
| **P-006** | Activación de Dispositivo de Respaldo / Backup | Supervisor de Ronda / Admin TI | HU-013, HU-014 | T-013, T-014, T-032, T-046, T-051 | **Must Have** |
| **P-007** | Consolidado de Tareo y Exportación Pre-Planilla | Analista de Operaciones / RRHH | HU-019, HU-020 | T-023, T-040, **T-060** | **Should Have** |

---

## 2. User Flows (Flujos de Usuario)

### 2.1 Flujo F-01: Autenticación de Supervisor y Marcación Facial en Campo
**Objetivo:** Permitir al Supervisor de Ronda autenticarse en el celular corporativo y registrar la asistencia biométrica de un trabajador en una sede de cliente.

```
 [Inicio] ──► P-001: Auth Supervisor (Biometría SO / PIN 4 Dígitos)
                   │
                   ▼ (Autenticado correctamente)
              P-002: Captura Facial & Liveness
                   │ ├── (Liveness OK >= 0.95 & Anti-Fake GPS OK)
                   │ └── (Foto capturada + Coordenadas GPS + NTP)
                   ▼
              P-003: Confirmación & Resultado Geocerca (INSIDE / OUTSIDE)
                   │ ├── SI Online ──► Envío directo API REST (/api/attendance/mark)
                   │ └── SI Offline ─► Cifrado SubtleCrypto AES-GCM ──► Storage IndexedDB/SQLite
                   ▼
               [Fin de Marcación - Retorno automático a P-002 en 3s]
```

---

### 2.2 Flujo F-02: Marcación Offline y Sincronización Diferida (RN-005-G)
**Objetivo:** Operar sin señal de red (4G/5G) y sincronizar marcaciones acumuladas de manera transparente al recuperar conectividad.

```
 [Marcación en Sótano/Sin Red] ──► P-003: Marcación Almacenada Localmente
                                        │
                                        ▼
                   [Banner Persistente Ámbar en Top Bar (RN-005-G)]
                   "3 marcaciones pendientes de sincronizar"
                                        │
             ┌──────────────────────────┴──────────────────────────┐
             ▼ (Detecta red 4G/Wi-Fi en segundo plano)            ▼ (Transcurren > 48h sin red)
   Service Worker / WorkManager Sync                      Banner cambia a Rojo Alerta
   Reintento Exponencial Jitter (T-021)                   "Sincronización Urgente (48h+)"
             │                                                     │
             ▼ (HTTP 201 Created deduplicado)                      ▼
   Banner cambia a Verde 3s ──► Limpieza IndexedDB        Límite 5,000 registros (RN-021) 
   "Todas las marcaciones sincronizadas"                  Bloqueo Preventivo de Captura
```

---

### 2.3 Flujo F-03: Monitoreo Operativo en Mapa GPS sin Login Tradicional (P-002 / ADR-003)
**Objetivo:** Permitir al Administrador de Operaciones inspeccionar el mapa de asistencias en tiempo real mediante un enlace/token firmado.

```
 [Acceso por Enlace Seguro / JWT Token TTL 12h] ──► P-004: Dashboard Web Mapa GPS
                                                         │
           ┌─────────────────────────────────────────────┼─────────────────────────────────────────────┐
           ▼                                             ▼                                             ▼
  Filtro por Rango Fechas                      Filtro por Estado Geocerca                    Visualización de Alertas
  (Fecha Inicio / Fecha Fin)                   [Verdes: 50m / Rojos: >50m]                   (Side Panel Fuera de Geocerca)
           │                                             │                                             │
           └─────────────────────────────────────────────┼─────────────────────────────────────────────┘
                                                         ▼
                                       Click en Pin del Mapa ──► Modal Detalle
                                       - Foto enmascarada
                                       - Distancia exacta (ej. 22.4 m)
                                       - Dispositivo UUID & Sello NTP
```

---

### 2.4 Flujo F-04: Enrolamiento Biométrico, Consentimiento y Solicitud ARCO
**Objetivo:** Registrar a un colaborador con consentimiento expreso Ley 29733 y procesar solicitudes ARCO de revocación/eliminación.

```
 [P-005: Panel Compliance / Enrolamiento]
           │
           ├──► Caso A: Enrolamiento Físico (ZKTeco ADMS) ──► Asignación `worker_id` + `enroll_number`
           │
           ├──► Caso B: Fallback Manual Sin ZKTeco ─────────► Estado `PENDING_CONSENT` (HU-022)
           │                                                   │
           │                                                   ▼
           │                                              Firma Digital / Huella Dactilar
           │                                              en Formato Físico/Digital
           │
           └──► Caso C: Solicitud ARCO (HU-015 / RN-037)
                      │
                      ▼
               Verificación de Identidad por DNI + Foto En Vivo
                      │
                      ├─► Aprobado ─► Revocación Clave AES-256 en KMS a las 24h (Crypto-Shredding)
                      └─► Rechazado ─► Audit Log con causa de denegación
```

---

## 3. Especificación Detallada de Pantallas y Wireframes ASCII

### 3.1 P-001 — Autenticación Primaria del Supervisor Móvil (HU-016, HU-017)

- **Propósito:** Validar la identidad del Supervisor de Ronda mediante la biometría nativa del sistema operativo o PIN de 4 dígitos antes de habilitar la consola de marcación en campo.
- **Actor Principal:** Supervisor de Ronda.
- **Historias de Usuario:** HU-016, HU-017.
- **Tareas del Backlog:** T-013, T-039, T-046.
- **Datos Mostrados:** Alias de la terminal (`device_alias`), Estado de binding del dispositivo (`PRIMARY_BOUND`), Contador de intentos fallidos (0/3).
- **Acciones Disponibles:** Desbloquear con Biometría SO (FaceID / Fingerprint), Ingresar PIN de 4 dígitos, Solicitar Desbloqueo de Emergencia por Admin TI.
- **Reglas de Negocio:** RN-012, RN-014, RN-014-A (Bloqueo temporal 15 min tras 3 intentos fallidos en < 5 min).

#### Layout Wireframe ASCII (P-001)

```
+-------------------------------------------------------+
|  [FiaYa Ops]  Terminal: MOB-LIM-042      [Net: 4G OK] |
+-------------------------------------------------------+
|                                                       |
|                     [ SHIELD ICON ]                   |
|              ACCESO A SUPERVISOR DE RONDA             |
|                                                       |
|  Dispositivo Vinculado: Samsung Galaxy Tab A8         |
|  Estado: DISPOSITIVO PRIMARIO AUTORIZADO              |
|                                                       |
|  +-------------------------------------------------+  |
|  |  [ TOUCH ID / FACE ID ]                         |  |
|  |  Presione el sensor de huella o mire a cámara   |  |
|  +-------------------------------------------------+  |
|                                                       |
|                 -- O INGRESE PIN --                   |
|                                                       |
|                   [ * ] [ * ] [ * ] [ * ]             |
|                                                       |
|           +-----------+-----------+-----------+       |
|           |     1     |     2     |     3     |       |
|           +-----------+-----------+-----------+       |
|           |     4     |     5     |     6     |       |
|           +-----------+-----------+-----------+       |
|           |     7     |     8     |     9     |       |
|           +-----------+-----------+-----------+       |
|           |   CANCEL  |     0     |   BORRAR  |       |
|           +-----------+-----------+-----------+       |
|                                                       |
|  [!] Intentos fallidos: 0 / 3                         |
|  [ Botón Desbloqueo de Emergencia TI (>= 48px) ]      |
|                                                       |
+-------------------------------------------------------+
```

#### Layout Wireframe ASCII (P-001) — Estado Bloqueado con Temporizador (QA-VAL-001)

+-------------------------------------------------------+
|  [FiaYa Ops]  Terminal: MOB-LIM-042      [Net: 4G OK] |
+-------------------------------------------------------+
|                                                       |
|                     [ LOCK ICON ]                     |
|           TERMINAL TEMPORALMENTE BLOQUEADA            |
|                                                       |
|  Ha superado el máximo de 3 intentos fallidos.        |
|  El dispositivo se desbloqueará automáticamente.      |
|                                                       |
|  +-------------------------------------------------+  |
|  |                                                 |  |
|  |        INTENTE DE NUEVO EN  29:45               |  |
|  |        [██████████████░░░░░░░░░] 58%            |  |
|  |                                                 |  |
|  +-------------------------------------------------+  |
|                                                       |
|  Evento registrado: DEVICE_TEMPORARILY_BLOCKED        |
|  (RN-014, RN-014-A)                                   |
|                                                       |
|  +-------------------------------------------------+  |
|  | [ BIOMETRÍA SO (Deshabilitado) ]   (opacity 0.4)|  |
|  +-------------------------------------------------+  |
|  | [ INGRESAR PIN (Deshabilitado) ]  (opacity 0.4)|  |
|  +-------------------------------------------------+  |
|                                                       |
|  [!] El contador se actualiza cada segundo.           |
|  [ Botón Desbloqueo de Emergencia TI (>= 48px) ]      |
|                                                       |
+-------------------------------------------------------+

- **Estados UI (actualizado QA-VAL-001):**
  - *Normal:* Botón biométrico activo, teclado numérico habilitado.
  - *Error Credencial (RN-014):* Borde rojo en casilla de PIN, mensaje "PIN Incorrecto (Intento 2/3)".
  - *Bloqueado (HU-017):* Pantalla grisada. **Contador regresivo `MM:SS` visible** con el tiempo restante de bloqueo de 30 min (ej. "Intente de nuevo en 29:45"). Los botones de biometría y PIN están deshabilitados (`opacity: 0.4`) hasta que el contador llegue a `00:00`. El contador se actualiza cada segundo. Al expirar el contador, los botones se rehabilitan sin re-bloqueo automático (RN-014-A).
  - *Micro-interacciones:* El contador inicia en `30:00` al tercer intento fallido y decrece hasta `00:00`; al expirar, se emite un `light haptic feedback` (vibración 50 ms) y se rehabilita el botón biométrico.

---

### 3.2 P-002 — Captura Facial Biométrica y Prueba de Vida Móvil (HU-001, HU-004, HU-005, HU-006, HU-026)

- **Propósito:** Capturar la fotografía del colaborador en campo con prueba de vida activa/pasiva y recopilar coordenadas GPS inmutables con sello NTP.
- **Actor Principal:** Supervisor de Ronda.
- **Historias de Usuario:** HU-001, HU-004, HU-005, HU-006, HU-026.
- **Tareas del Backlog:** T-008, T-010, T-011, T-016, T-047, T-048.
- **Datos Mostrados:** Vista previa de cámara en tiempo real, Guía elíptica de posicionamiento facial, Indicador de gesto solicitado (ej. "Parpadee 2 veces"), Indicador NTP y estado GPS (`Accuracy: 12m`), Banner de pendientes (`PENDING_SYNC`).
- **Acciones Disponibles:** Capturar Marcación (Automática por consenso de liveness / Manual), Alternar Cámara (Frontal/Trasera), Cancelar Marcación.
- **Reglas de Negocio:** RN-001A ($score \ge 0.95$, $time \le 5.0\text{ s}$), RN-003 (Android Native Provider), RN-003-A (Consenso iOS 3 muestras), RN-003-A-6 (< 2 señales), RN-003-B/C (Bloqueo Root/Jailbreak), RN-004 / RN-020 (Sello NTP servidor).

#### Layout Wireframe ASCII (P-002) Estado Normal

```
+-------------------------------------------------------+
| [<=] Sede: BCP San Isidro   [SYNC PENDING: 0] [GPS OK]|
+-------------------------------------------------------+
|                                                       |
|  +-------------------------------------------------+  |
|  |                 /-------------\                 |  |
|  |                /   ENQUADRE    \                |  |
|  |               |   EL ROSTRO     |               |  |
|  |               |     AQUI        |               |  |
|  |                \               /                |  |
|  |                 \-------------/                 |  |
|  |                                                 |  |
|  |  [ GESTO SOLICITADO: PARPADEE DOS VECES ]       |  |
|  |  Progreso Liveness: [████████████░░░] 82%       |  |
|  +-------------------------------------------------+  |
|                                                       |
|  Colaborador: PÉREZ GÓMEZ, JUAN CARLOS                |
|  DNI: *******842                                      |
|  Sello Tiempo NTP: 11/11/2026 14:32:05 UTC-5          |
|  Precisión GPS: +/- 8.5 metros (Excelente)             |
|                                                       |
|  +-------------------------------------------------+  |
|  | [ PROCESAR MARCACIÓN BIOMÉTRICA (>= 48px) ]      |  |
|  +-------------------------------------------------+  |
|  | [ CANCELAR / SELECCIONAR OTRO TRABAJADOR ]      |  |
|  +-------------------------------------------------+  |
|                                                       |
+-------------------------------------------------------+
```

- **Estados UI:**
  - *Analizando Liveness:* Círculo de progreso verde giratorio alrededor de la elipse.
  - *Prueba Fallida (RN-001A):* Banner rojo superior "Prueba de Vida no superada. Intente de nuevo".
  - *Detectado Fake GPS (HU-004):* Modal rojo bloqueante "Ubicación Simulada Detectada. Evento reportado a Seguridad".
  - *Detectado Root/Jailbreak (HU-005):* Cierre de aplicación e inhabilitación del dispositivo.
- **Micro-interacciones:** Captura fotográfica automática al alcanzar $0.95$ en score de liveness sin necesidad de toque en pantalla.


#### Layout Wireframe ASCII (P-002) — Estado Bloqueado por Root/Jailbreak con Modal de Remediación (QA-VAL-003)

+-------------------------------------------------------+
| [<=] Sede: BCP San Isidro   [SYNC PENDING: 0] [GPS OK]|
+-------------------------------------------------------+
|                                                       |
|  +-------------------------------------------------+  |
|  |                                                 |  |
|  |            [ SHIELD / LOCK ICON ]               |  |
|  |                                                 |  |
|  |        DISPOSITIVO NO ÍNTEGRO                   |  |
|  |        (ROOT / JAILBREAK DETECTADO)             |  |
|  |                                                 |  |
|  |  Código de diagnóstico:                         |  |
|  |  ERR_INTEGRITY_FAIL_a3f9c1e2                    |  |
|  |                                                 |  |
|  |  Contacte a su administrador para restablecer   |  |
|  |  la integridad del dispositivo.                 |  |
|  |                                                 |  |
|  |  +-------------------------------------------+  |  |
|  |  | [ COPIAR CÓDIGO Y CONTACTAR SOPORTE TI ]  |  |  |
|  |  +-------------------------------------------+  |  |
|  |  | [ CERRAR SESIÓN ]                         |  |  |
|  |  +-------------------------------------------+  |  |
|  |                                                 |  |
|  +-------------------------------------------------+  |
|                                                       |
|  Evento registrado: ROOT_DETECTED / JAILBREAK_DETECTED|
|  (RN-003-B, RN-003-C)                                 |
|                                                       |
+-------------------------------------------------------+

- **Comportamiento (QA-VAL-003):**
  - Al detectar root/jailbreak, se superpone el modal bloqueante con el `integrity_diagnostic_code = "ERR_INTEGRITY_FAIL_" + device_uuid.slice(0,8)`.
  - El botón "COPIAR CÓDIGO Y CONTACTAR SOPORTE TI" ejecuta `Clipboard.setString(integrity_diagnostic_code)` y abre el canal de soporte (mailto o deep link).
  - "CERRAR SESIÓN" cierra la sesión activa del supervisor y deshabilita P-002 hasta que el dispositivo sea remediado.
  - El código se registra en `audit_logs` con `event_type = ROOT_DETECTED | JAILBREAK_DETECTED`.

#### Layout Wireframe ASCII (P-002) — Estado iOS con Barra de Progreso "Muestra X/3" (QA-VAL-004)

+-------------------------------------------------------+
| [<=] Sede: BCP San Isidro   [SYNC PENDING: 0] [GPS OK]|
+-------------------------------------------------------+
|                                                       |
|  +-------------------------------------------------+  |
|  |                 /-------------\                 |  |
|  |                /   ENQUADRE    \                |  |
|  |               |   EL ROSTRO     |               |  |
|  |               |     AQUI        |               |  |
|  |                \               /                |  |
|  |                 \-------------/                 |  |
|  |                                                 |  |
|  |  [ GESTO SOLICITADO: PARPADEE DOS VECES ]       |  |
|  |  Progreso Liveness: [████████████░░░] 82%       |  |
|  +-------------------------------------------------+  |
|                                                       |
|  Verificación de geolocalización iOS (RN-003-A):      |
|  +-------------------------------------------------+  |
|  |  Muestra 1/3 · Muestra 2/3 · Muestra 3/3        |  |
|  |  [██████████░░░░░░░░░░] 1 / 3                   |  |
|  |  Capturando lectura GPS consecutiva...           |  |
|  +-------------------------------------------------+  |
|                                                       |
|  Colaborador: PÉREZ GÓMEZ, JUAN CARLOS                |
|  DNI: *******842                                      |
|  Sello Tiempo NTP: 11/11/2026 14:32:05 UTC-5          |
|  Precisión GPS: +/- 8.5 metros (Excelente)             |
|                                                       |
|  +-------------------------------------------------+  |
|  | [ PROCESAR MARCACIÓN BIOMÉTRICA (>= 48px) ]      |  |
|  +-------------------------------------------------+  |
|  | [ CANCELAR / SELECCIONAR OTRO TRABAJADOR ]      |  |
|  +-------------------------------------------------+  |
|                                                       |
+-------------------------------------------------------+

- **Comportamiento (QA-VAL-004):**
  - En iOS, la App captura 3 muestras GPS consecutivas (ventana `3 s ≤ Δt_total ≤ 30 s`). La barra muestra "Muestra 1/3", "Muestra 2/3", "Muestra 3/3" con el progreso visual y el contador `sample_progress`.
  - Si las 3 muestras alcanzan consenso (≥ 2 señales testeables activadas), se procede con la evaluación RN-003-A normal.
  - Si tras 3 muestras `signals_count < 2`, se muestra el estado `REQUIRES_MANUAL_APPROVAL` con botón "Derivar a Aprobación Manual" que navega a P-005 con `event_type = UNCERTAIN_GEOLOCATION`.
  - El indicador se oculta en Android (que usa el proveedor nativo `isFromMockProvider`).


---

### 3.3 P-003 — Confirmación de Marcación y Resultado de Geocerca Móvil (HU-002, HU-003, HU-007, HU-008, HU-025)

- **Propósito:** Mostrar al supervisor y trabajador la confirmación inmediata del resultado del marcaje (Dentro de Geocerca / Fuera de Geocerca) y el estado de sincronización.
- **Actor Principal:** Supervisor de Ronda.
- **Historias de Usuario:** HU-002, HU-003, HU-007, HU-008, HU-025.
- **Tareas del Backlog:** T-005, T-006, T-009, T-012, T-045, T-049.
- **Datos Mostrados:** Badge de resultado (`INSIDE_GEOFENCE` / `OUTSIDE_GEOFENCE`), Distancia geodésica calculada ($D\text{ metros}$), Foto thumbnail enmascarada, Timestamp inmutable, Badge de conectividad (`ONLINE` / `STORED_OFFLINE`).
- **Acciones Disponibles:** Confirmar y Volver a Lista, Re-capturar Coordenadas por GPS Incierto (HU-003), Ver Cola de Sincronización Local.
- **Reglas de Negocio:** RN-002 ($D \le 50.00\text{m}$), RN-002-A ($accuracy > 25\text{m} \rightarrow \text{UNCERTAIN}$), RN-005 (IndexedDB Cifrado), RN-005-G (Banner Pendientes).

#### Layout Wireframe ASCII (P-003) estado normal

```
+-------------------------------------------------------+
|  MARCACIÓN REGISTRADA           [ESTADO: GUARDADO]    |
+-------------------------------------------------------+
|                                                       |
|            +-----------------------------+            |
|            |   [ CHECKMARK VERDE ]       |            |
|            |   DENTRO DE GEOCERCA        |            |
|            |   Distancia: 14.2 metros    |            |
|            +-----------------------------+            |
|                                                       |
|  [Foto Enmascarada]  Trabajador: PÉREZ GÓMEZ, JUAN    |
|                      DNI: *******842                  |
|                      Sede: BCP - San Isidro           |
|                      Hora: 11/11/2026 14:32:05        |
|                                                       |
|  ---------------------------------------------------  |
|  Estado de Transmisión:                               |
|  [OK] Guardado Cifrado en Dispositivo (Offline-First) |
|  [!] Se sincronizará automáticamente al detectar red |
|  ---------------------------------------------------  |
|                                                       |
|  +-------------------------------------------------+  |
|  | [ ENVIAR Y RETORNAR A INICIO (3s) ]             |  |
|  +-------------------------------------------------+  |
|  | [ RE-CAPTURAR UBICACIÓN (GPS Incierto > 25m) ]  |  |
|  +-------------------------------------------------+  |
|                                                       |
+-------------------------------------------------------+
```

#### Layout Wireframe ASCII (P-003) — Banner de Sincronización con Alerta al 80 % (QA-VAL-002)

+-------------------------------------------------------+
|  MARCACIÓN REGISTRADA           [ESTADO: GUARDADO]    |
+-------------------------------------------------------+
|  ⚠️ ALMACENAMIENTO LOCAL AL 81 % (4,050/5,000)        |
|  [██████████████████░░░░░░░] 4,050 / 5,000 registros  |
|  Uso de disco: 162 / 200 MB                           |
|  ⚠️ Conéctese a red para sincronizar antes del bloqueo |
|  (parpadeo 1 Hz · RN-021-A)                           |
+-------------------------------------------------------+
|                                                       |
|            +-----------------------------+            |
|            |   [ CHECKMARK VERDE ]       |            |
|            |   DENTRO DE GEOCERCA        |            |
|            |   Distancia: 14.2 metros    |            |
|            +-----------------------------+            |
|                                                       |
|  [Foto Enmascarada]  Trabajador: PÉREZ GÓMEZ, JUAN    |
|                      DNI: *******842                  |
|                      Sede: BCP - San Isidro           |
|                      Hora: 11/11/2026 14:32:05        |
|                                                       |
|  ---------------------------------------------------  |
|  Estado de Transmisión:                               |
|  [OK] Guardado Cifrado en Dispositivo (Offline-First) |
|  [!] Se sincronizará automáticamente al detectar red  |
|  [!] Reintento de sincronización forzado si hay red   |
|  ---------------------------------------------------  |
|                                                       |
|  +-------------------------------------------------+  |
|  | [ ENVIAR Y RETORNAR A INICIO (3s) ]             |  |
|  +-------------------------------------------------+  |
|  | [ RE-CAPTURAR UBICACIÓN (GPS Incierto > 25m) ]  |  |
|  +-------------------------------------------------+  |
|                                                       |
+-------------------------------------------------------+

- **Estados UI (actualizado QA-VAL-002):**
  - *Fuera de Geocerca (RN-002-C):* Badge rojo `OUTSIDE_GEOFENCE`, texto "Marcación registrada a 145m de la sede (Requiere justificación)".
  - *GPS Incierto (HU-003):* Badge amarillo `GEOLOCATION_UNCERTAIN` con botón parpadeante "Re-capturar GPS".
  - *Sincronizado Online:* Icono de nube verde con checkmark.
  - *Saturación ≥ 80 % (RN-021-A):* **Banner superior rojo parpadeante** (frecuencia 1 Hz) con el conteo exacto (`registros_pendientes`, `uso_disco_mb`) y el mensaje "⚠️ Almacenamiento local al 81 % (4,050/5,000). Conéctese a red para sincronizar". Se mantiene hasta que `registros_pendientes < 4,000` AND `uso_disco_mb < 160` o hasta el bloqueo total por RN-021 (5,000/200 MB).
  - *Saturación 100 % (RN-021):* Bloqueo total de captura; mensaje "Almacenamiento local lleno (5,000/200 MB). Trasládese a zona con cobertura para sincronizar".

---

### 3.4 P-004 — Dashboard Web de Monitoreo GPS Operativo (HU-009, HU-018, P-002)

- **Propósito:** Permitir al Administrador de Operaciones monitorear visualmente en un mapa interactivo la ubicación exacta de las asistencias, validar radios de geocercas y gestionar alertas en tiempo real sin requerir login convencional (acceso por token de entorno JWT TTL 12 h).
- **Actor Principal:** Administrador de Operaciones.
- **Historias de Usuario:** HU-009, HU-018.
- **Tareas del Backlog:** T-025, T-026, T-027, T-028, T-041, T-050.
- **Datos Mostrados:** Mapa Leaflet / OpenStreetMap en pantalla completa, Pines de marcación (Verde: DENTRO, Rojo: FUERA, Ámbar: RE-VERIFICACIÓN), Círculo azul overlay de Geocerca 50m, Feed de alertas operativas en panel lateral, Filtros por Fecha, Sede y Trabajador.
- **Acciones Disponibles:** Filtrar Registros, Hacer Clic en Pin para Ver Detalle, Exportar Mapa en PDF/PNG, Abrir Panel de Alertas en Tiempo Real.
- **Reglas de Negocio:** RN-007 (Visor sin login), RN-008 (Alertas en mapa), RN-013 (Sello temporal UTC-5).

#### Layout Wireframe ASCII (P-004)

```
+-----------------------------------------------------------------------------------+
| [FiaYa Ops] DASHBOARD DE MONITOREO GPS EN TIEMPO REAL    [Token Expira: 11h 45m]  |
+-----------------------------------------------------------------------------------+
| Filtros: [ Sede: Todas v ] [ Fecha: 11/11/2026 v ] [ Estado: Todos v ] [Buscar...] |
+---------------------------------------------------+-------------------------------+
|                                                   | PANEL DE ALERTAS EN VIVO      |
|                                                   | ----------------------------- |
|         +-------------------------------+         | [!] 14:31:02 - FUERA GEOCERCA |
|         |           MAPA LEAFLET        |         |     Trabajador: RAMOS, M.    |
|         |                               |         |     Sede: Rímac (Dist: 180m)   |
|         |    (o) Sede BCP San Isidro    |         |     [ Ver en Mapa ]           |
|         |     |                         |         |                               |
|         |   .-'-. Radio 50m             |         | [!] 14:28:44 - RE-VERIF FAIL  |
|         |  /  .  \                      |         |     Trabajador: LOPEZ, C.    |
|         | |  (*)  | Pin Verde (14.2m)   |         |     Score Server: 0.74        |
|         |  \     /                      |         |     [ Revisar Auditoría ]     |
|         |   '-.-'                       |         |                               |
|         |                               |         | [OK] 14:25:10 - MARCACION OK  |
|         |         (*) Pin Rojo (145m)   |         |     Trabajador: PEREZ, J.     |
|         +-------------------------------+         |     Sede: San Isidro (12m)    |
|                                                   |                               |
| Total Marcaciones: 142 | Dentro: 135 | Fuera: 7   | [ Exportar Vista CSV/PDF ]    |
+---------------------------------------------------+-------------------------------+
```

- **Estados UI:**
  - *Carga Inicial:* Skeleton Loader en mapa y tabla de alertas.
  - *Nueva Alerta WebSocket (HU-018):* Destello rojo en el panel lateral y sonido discreto de notificación.
  - *Token Expirado (P-002):* Modal superpuesto "Sesión de Entorno Expirada. Solicite un nuevo enlace de acceso".

---

### 3.5 P-005 — Gestión de Enrolamiento, Consentimiento y Solicitud ARCO (HU-010, HU-015, HU-022, HU-023, RN-037)

- **Propósito:** Registrar nuevos colaboradores con vectores biométricos, administrar firmas de consentimiento Ley 29733, gestionar fallbacks manuales (`PENDING_CONSENT`) y procesar solicitudes ARCO de revocación con verificación de identidad en dos pasos.
- **Actor Principal:** Administrador de RRHH / Compliance Officer.
- **Historias de Usuario:** HU-010, HU-015, HU-022, HU-023, HU-024.
- **Tareas del Backlog:** T-003, T-030, T-031, T-033, T-034, T-037, T-038, T-053, T-054.
- **Datos Mostrados:** Lista de trabajadores, Estado de consentimiento (`ACTIVE_CONSENT` / `PENDING_CONSENT` / `REVOKED`), Formulario de datos personales cifrados, Panel de verificación de identidad ARCO (DNI + Foto En Vivo), Bitácora de solicitudes ARCO.
- **Acciones Disponibles:** Registrar Trabajador, Enrolar Vector Biométrico, Cargar Formato Físico de Consentimiento, Ejecutar Revocación ARCO (Crypto-Shredding), Ver Log de Auditoría.
- **Reglas de Negocio:** RN-030/A/B/C/D, RN-031…RN-035/A/B, RN-036/A…D, RN-037 (Verificación Identidad ARCO obligatoria).

#### Layout Wireframe ASCII (P-005) — Estado Normal (con Badge QA-VAL-005)

```
+-----------------------------------------------------------------------------------+
| [FiaYa Compliance] CUMPLIMIENTO LEY 29733 & SOLICITUDES ARCO                      |
+-----------------------------------------------------------------------------------+
| TABLA DE TRABAJADORES Y ESTADO DE CONSENTIMIENTO                                  |
| +-------------------+------------+-------------------+--------------------------+ |
| | NOMBRE            | DNI        | ESTADO BIOMÉTRICO | CONSENTIMIENTO LEY 29733 | |
| +-------------------+------------+-------------------+--------------------------+ |
| | PÉREZ G., JUAN    | *******842 | ENROLADO (ZKTeco) | [VALIDADO: 15/01/2026]   | |
| | RAMOS K., MARIA   | *******119 | ENROLADO (App)    | [VALIDADO: 02/03/2026]   | |
| | GÓMEZ T., CARLOS  | *******554 | SIN BIOMETRÍA     | [PENDING_CONSENT (H-22)] | |
| +-------------------+------------+-------------------+--------------------------+ |
|                                                                                   |
| PANEL DE ENROLAMIENTO BIOMÉTRICO                                                  |
| +-------------------------------------------------------------------------------+ |
| | Captura de Vector Facial                                                      | |
| | [ 📷 Iniciar Captura Biométrica ]                                             | |
| |                                                                               | |
| | 🔒 Datos biométricos cifrados en reposo AES-256 (Ley N.° 29733)  [ⓘ]         | |
| |    Tooltip: "Los vectores y fotos faciales se cifran con AES-256 y se        | |
| |    conservan durante la relación laboral + 30 días naturales. Al vencer,       | |
| |    se ejecuta crypto-shredding con revocación inmediata de clave."           | |
| +-------------------------------------------------------------------------------+ |
|                                                                                   |
| PANEL DE PROCESAMIENTO DE SOLICITUD ARCO (RN-037)                                 |
| +-------------------------------------------------------------------------------+ |
| | Solicitar: REVOCACIÓN DE CONSENTIMIENTO Y SHREDDING BIOMÉTRICO                | |
| | Trabajador Seleccionado: PÉREZ GÓMEZ, JUAN CARLOS (ID: W-84920)               | |
| |                                                                               | |
| | Pasos de Verificación de Identidad Obligatoria (RN-037):                       | |
| |  [X] Step 1: Cotejo de DNI y Formulario Firmado ARCO                         | |
| |  [X] Step 2: Captura en Vivo de Identidad del Solicitante                     | |
| |                                                                               | |
| | [ EJECUTAR REVOCACIÓN Y SHREDDING AES-256 A LAS 24H (T-031/T-037) (>= 48px) ] | |
| +-------------------------------------------------------------------------------+ |
+-----------------------------------------------------------------------------------+

```

- **Badge de cumplimiento (QA-VAL-005):** Componente `ComplianceBadge` persistente junto al campo de captura de vector facial, con texto "🔒 Datos biométricos cifrados en reposo AES-256 (Ley N.° 29733)" y tooltip explicativo de retención (duración laboral + 30 días). Verificado por T-059 y T-053.

- **Estados UI:**
  - *Procesando Shredding:* Progress bar "Revocando Clave KMS AES-256... Destrucción criptográfica completa".
  - *Identidad ARCO Rechazada (RN-037):* Alerta en pantalla "No se pudo verificar la identidad del solicitante. Proceso abortado y registrado en Audit Log".


---

### 3.6 P-006 — Activación de Dispositivo de Respaldo / Backup (HU-013, HU-014, RN-012-A)

- **Propósito:** Permitir la activación inmediata de un smartphone de contingencia/backup ante la pérdida, robo o avería del celular corporativo primario.
- **Actor Principal:** Supervisor de Ronda / Administrador de TI.
- **Historias de Usuario:** HU-013, HU-014.
- **Tareas del Backlog:** T-013, T-014, T-032, T-046, T-051.
- **Datos Mostrados:** UUID de terminal primaria inhabilitada, Formulario de ingreso de código de activación de emergencia (8 caracteres alfanuméricos), Constancia de pérdida en PDF cargada, UUID de terminal de respaldo.
- **Acciones Disponibles:** Cargar Constancia Policial/Interna de Pérdida, Solicitar Token de Respaldo, Activar Dispositivo Backup, Desvincular Dispositivo Primario.
- **Reglas de Negocio:** RN-012-A (Binding de respaldo), RN-012-C/D (Autorización TI).

#### Layout Wireframe ASCII (P-006)

```
+-------------------------------------------------------+
| ACTIVACIÓN DE DISPOSITIVO DE RESPALDO (BACKUP)        |
+-------------------------------------------------------+
|                                                       |
|  Dispositivo Actual (Sin Activar):                    |
|  UUID: f47ac10b-58cc-4372-a567-0e02b2c3d479           |
|  Modelo: Xiaomi Redmi Note 12 (BACKUP-02)             |
|                                                       |
|  ---------------------------------------------------  |
|  PASO 1: Adjuntar Constancia de Pérdida / Falla       |
|  [ Archivo: constancia_robos_11112026.pdf (Cargado) ] |
|                                                       |
|  PASO 2: Ingrese Código de Desbloqueo TI (8 dígitos) |
|  +-------------------------------------------------+  |
|  |  [ B A K - 8 9 2 4 - X ]                        |  |
|  +-------------------------------------------------+  |
|                                                       |
|  +-------------------------------------------------+  |
|  | [ ACTIVAR DISPOSITIVO Y DESVINCULAR ANTERIOR ]  |  |
|  +-------------------------------------------------+  |
|                                                       |
|  [!] Esta acción inhabilitará la terminal primaria    |
|      y registrará un evento inmutable de seguridad.   |
|                                                       |
+-------------------------------------------------------+


```

### 3.7 P-007 — Consolidado de Tareo y Exportación Pre-Planilla (HU-019, HU-020, T-040, T-060)

- **Propósito:** Permitir al Analista de Planillas generar y descargar el reporte de tareo del periodo cerrado, con sellado criptográfico SHA-256 verificable para auditoría MTPE/SUNAT.
- **Actor Principal:** Analista de Operaciones / RRHH.
- **Historias de Usuario:** HU-019, HU-020.
- **Tareas del Backlog:** T-040, T-060, T-023, T-024.
- **Datos Mostrados:** Periodo seleccionado, total de trabajadores, total de marcaciones, consolidado de asistencia/tardanzas/sobretiempos, marca de faltas explícitas, exclusiones por `REQUIRES_MANUAL_APPROVAL`, `signature_hash` (truncado a 16 caracteres), `tenant_id`, `utc_timestamp`.
- **Acciones Disponibles:** Seleccionar periodo cerrado, Generar reporte (≤ 15 min), Descargar CSV/PDF, Verificar integridad (descarga manifiesto `signature_manifest.json`).
- **Reglas de Negocio:** RN-016, RN-019, RN-020-A.

#### Layout Wireframe ASCII (P-007)

+-----------------------------------------------------------------------------------+
| [FiaYa Ops] CONSOLIDADO DE TAREO Y PRE-PLANILLA                    [Token: 11h 45m]|
+-----------------------------------------------------------------------------------+
| Periodo cerrado: 01/11/2026 — 15/11/2026        [ Cambiar periodo v ]             |
| +-------------------------------------------------------------------------------+ |
| | Resumen:                                                                       | |
| |  · Trabajadores con asistencia: 142 / 150                                       | |
| |  · Marcaciones válidas: 2,840                                                    | |
| |  · Tardanzas: 12  · Faltas: 8  · Sobretiempos: 46 h                            | |
| |  · Marcaciones REQUIRES_MANUAL_APPROVAL (excluidas): 3                         | |
| +-------------------------------------------------------------------------------+ |
|                                                                                   |
| +-------------------------------------------------------------------------------+ |
| | ✅ Reporte firmado digitalmente (SHA-256)                                       | |
| | Hash: a3f9c1e2b7d84f01…                                                         | |
| | Tenant: TEN-001  ·  Generado: 11/11/2026 14:32:05 UTC-5                         | |
| | [ ⬇ Descargar CSV ]   [ ⬇ Descargar PDF ]   [ 🔍 Verificar integridad ]        | |
| +-------------------------------------------------------------------------------+ |
|                                                                                   |
| [ GENERAR REPORTE (≤ 15 min) ]                                                    |
+-----------------------------------------------------------------------------------+

- **Comportamiento (QA-VAL-006):** El componente `SignedReportBadge` muestra el `signature_hash` truncado (primeros 16 caracteres) junto al botón de exportación. "Verificar integridad" descarga `signature_manifest.json` con `{ signature_hash, tenant_id, utc_timestamp }`. Verificado por T-060 y T-024.
- **Estados UI:** *Generando* → progress bar con ETA; *Listo* → badge verde "✅ Reporte firmado"; *Excluidas* → contador ámbar con enlace a la lista de marcaciones pendientes de aprobación.

---


## 4. Sistema de Diseño y Tokens (Design System)

### 4.1 Paleta de Colores y Tokens WCAG 2.1 AA

Todos los colores han sido seleccionados para garantizar un ratio de contraste de texto/fondo $\ge 4.5:1$ (WCAG 2.1 AA) para operabilidad bajo luz solar directa en campo.

```
┌─────────────────────────┬──────────────┬──────────────┬──────────────────┬─────────────────┐
│ Token UI                │ Hex Code     │ Uso Principal│ Elemento UI      │ Contrast Ratio  │
├─────────────────────────┼──────────────┼──────────────┼──────────────────┼─────────────────┤
│ `--color-success-bg`    │ `#DCFCE7`    │ Fondo Estado │ Badge INSIDE     │ 7.2:1 (v. Dark) │
│ `--color-success-fg`    │ `#15803D`    │ Texto/Icono  │ Pin Verde / Check│ 5.1:1 (vs Light)│
│ `--color-danger-bg`     │ `#FEE2E2`    │ Fondo Estado │ Badge OUTSIDE    │ 8.4:1 (v. Dark) │
│ `--color-danger-fg`     │ `#B91C1C`    │ Texto/Icono  │ Pin Rojo / Alert │ 6.3:1 (vs Light)│
│ `--color-warning-bg`    │ `#FEF3C7`    │ Fondo Estado │ Badge SYNC PEND  │ 6.8:1 (v. Dark) │
│ `--color-warning-fg`    │ `#B45309`    │ Texto/Icono  │ Banner Ámbar     │ 4.9:1 (vs Light)│
│ `--color-brand-primary` │ `#1E40AF`    │ Botones/Bars │ Botón Primario   │ 7.1:1 (vs White)│
│ `--color-neutral-dark`  │ `#0F172A`    │ Títulos/Text │ Texto Principal  │ 15.4:1 (v. White│
│ `--color-neutral-light` │ `#F8FAFC`    │ Fondo App    │ Canvas Principal │ 1.05:1 (Ref)    │
└─────────────────────────┴──────────────┴──────────────┴──────────────────┴─────────────────┘
```

---

### 4.2 Tipografía y Escala Visual Móvil/Web

- **Fuente Primaria:** `Inter`, `Roboto`, system-ui.
- **Escala Tipográfica:**
  - **Título Pantalla (H1):** `24px` / `Line-height: 32px` / Bold (700)
  - **Subtítulo (H2):** `18px` / `Line-height: 24px` / Semibold (600)
  - **Cuerpo Texto (Body):** `16px` / `Line-height: 22px` / Regular (400)
  - **Texto Secundario / Labels:** `14px` / `Line-height: 20px` / Medium (500)
  - **Badges / Micro-copy:** `12px` / `Line-height: 16px` / Bold (700) Upper

---

### 4.3 Botones y Áreas Táctiles (Touch Targets $\ge 48\text{px}$)

Para garantizar la operación a una sola mano por los supervisores de ronda en campo:
- **Área Táctil Mínima:** $48\text{px} \times 48\text{px}$ en todos los elementos interactivos (botones, casillas de selección, iconos de menú, pines de mapa).
- **Separación Mínima entre Botones:** $12\text{px}$ de margen de seguridad para evitar toques accidentales.
- **Botón Primario Móvil:** Altura fija de $56\text{px}$, ancho $100\%$ del contenedor móvil, bordes redondeados $8\text{px}$.

---

### 4.4 Reglas de Formato de Localización Peruana en UI

1. **Moneda:** Formato en Soles `S/ 1,250.50` (Separador de miles `,`, decimal `.`).
2. **Fechas:** Formato peruano `DD/MM/AAAA` (ej. `11/11/2026`).
3. **Horas:** Formato 24 horas con zona horaria oficial `America/Lima` (UTC-5) (ej. `14:32:05 UTC-5`).
4. **Coordenadas Geográficas:** Formato decimal WGS84 / EPSG:4326 con 6 decimales de precisión (ej. Lat: `-12.093100`, Lng: `-77.036500`).
5. **Ventana Horaria WhatsApp:** Botón "Notificar por WhatsApp" habilitado estrictamente de 09:00 a 20:00 UTC-5 (PRINCIPIO-006). Fuera de horario, el botón muestra el estado `DISABLED` con el tooltip *"Notificaciones suspendidas fuera del horario legal (09:00 - 20:00)"*.

---

## 5. Navigation Map (Mapa de Navegación Árbol Textual)

```
[ APLICACIÓN MÓVIL SUPERVISOR (React Native / Capacitor) ]
 │
 ├── P-001: Autenticación Supervisor (PIN / Biometría SO)
 │    │
 │    ├── [Autenticado] ──► P-002: Captura Facial Biométrica & Liveness
 │    │                      │
 │    │                      ├── [Captura Exitosa] ──► P-003: Confirmación & Geocerca
 │    │                      │                          │
 │    │                      │                          └── [3s Auto-Return] ──► (Regresa a P-002)
 │    │                      │
 │    │                      └── [GPS Incierto > 25m] ──► Modal Re-captura GPS
 │    │
 │    └── [Falla Pérdida Dispositivo] ──► P-006: Activación Dispositivo Backup
 │
 └── [Banner Persistente Top Bar (RN-005-G)] ──► Modal Visualizador Cola Offline


[ DASHBOARD WEB ADMINISTRATIVO (Next.js 14 App Router) ]
 │
 ├── P-004: Dashboard Monitoreo GPS (Acceso por Token JWT TTL 12h sin Login Tradicional)
 │    │
 │    ├── Modal Detalle de Marcación (Click en Pin / Fila Alerta)
 │    ├── Panel Lateral de Alertas en Tiempo Real (WebSockets)
 │    └── Exportador de Reporte Visual (PDF / CSV)
 │
 ├── P-005: Módulo Compliance, Enrolamiento & ARCO (Acceso Admin Autenticado)
 │    │
 │    ├── Formulario Registro & Enrolamiento Biométrico
 │    ├── Modal Firma / Carga Consentimiento Ley 29733 (HU-022)
 │    └── Panel Verificación Identidad & Crypto-Shredding ARCO (RN-037)
 │
 └── P-007: Consolidado de Tareo & Pre-Planilla (Acceso Analista Operaciones)
      │
      └── Exportador de Reporte Auditado de Asistencia & Horas Extra
```

---

## 6. Matriz de Trazabilidad UI ↔ HU ↔ RN ↔ Backlog (T-XXX)

| ID Pantalla / Componente UI | Historia de Usuario | Regla de Negocio (RN) | Tarea de Backlog | Componente de Arquitectura |
|---|---|---|---|---|
| **P-001: Keypad Auth & Bio** | HU-016, HU-017 | RN-012, RN-014, RN-014-A | T-013, T-039, T-046, **T-055** | Componente A — Mobile Application |
| **P-001: Timer MM:SS bloqueo** | HU-017 | RN-014, RN-014-A | **T-055** | Componente A — Mobile Application |
| **P-002: Frame Elipse Liveness** | HU-001, HU-026 | RN-001A, RN-003-A-6 | T-008, T-010, T-016, T-047 | Componente A — Mobile Application |
| **P-002: Anti-Fake GPS & NTP** | HU-004, HU-005, HU-006 | RN-003-B/C, RN-004, RN-020 | T-010, T-011, T-048 | Componente A — Mobile Application |
| **P-002: Modal Remediation Root/JB** | HU-005 | RN-003-B, RN-003-C | **T-057** | Componente A — Mobile Application |
| **P-002: iOS Consensus "X/3"** | HU-003, HU-026 | RN-003-A-1…A-6 | **T-058** | Componente A — Mobile Application |
| **P-003: Badge Geocerca 50m** | HU-002, HU-003 | RN-002, RN-002-A/B/C | T-005, T-006, T-009, T-040 | Componente A & D — Attendance Core |
| **P-003: Banner Sync Offline** | HU-007, HU-008, HU-025 | RN-005, RN-005-G, RN-005-H | T-012, T-021, T-045, T-049 | Componente A & C — Async Engine |
| **P-003: Alert 80% Saturation** | HU-007 | RN-021, **RN-021-A** | T-012, T-049 | Componente A — Mobile Application |
| **P-004: Mapa Leaflet & Pines** | HU-009 | RN-007, RN-013 | T-025, T-026, T-041, T-050 | Componente E — Monitoring Dashboard |
| **P-004: Feed Alertas Real-time** | HU-018 | RN-008 | T-027, T-028 | Componente E — Monitoring Dashboard |
| **P-005: Form Consentimiento** | HU-010, HU-022 | RN-030, RN-030-A/B/C/D | T-003, T-004, T-035, T-036 | Componente D & F — Compliance KMS |
| **P-005: Badge Ley 29733** | HU-010, HU-015 | RN-030, RN-037 | **T-059** | Componente F — Compliance KMS |
| **P-005: Panel ARCO Shredding** | HU-015, HU-023, HU-024 | RN-031…RN-037 | T-030, T-031, T-033, T-034, T-037, T-038, T-054 | Componente F — Compliance KMS |
| **P-006: Form Backup Binding** | HU-013, HU-014 | RN-012-A/B/C/D | T-013, T-014, T-032, T-046, T-051 | Componente A & F — Compliance KMS |
| **P-007: Consolidado Tareo** | HU-019, HU-020 | RN-016, RN-019 | T-023, T-040 | Componente C & D — Attendance Core |
| **P-007: Badge SHA-256** | HU-020 | RN-016 | **T-060** | Componente C & D — Attendance Core |