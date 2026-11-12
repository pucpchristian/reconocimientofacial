# Historias de Usuario — Módulo Operativo y Control de Asistencia Omnicanal

**Versión:** 1.1.1
**Ámbito:** Must-have (M-01…M-10) y Should-have (S-01…S-05) del MVP.
**Documentos de referencia:** `00-constitucion.md` (v2.2.2), `01-project-brief.md`, `02-alcance.md` (v2.2.2), `03-reglas-de-negocio.md` (v2.2.2), `04-glosario.md`, `05-preguntas-abiertas.md`.
**Cambios v1.1.1:** HU-007 extendida con criterio de alerta al 80 % (RN-021-A / QA-VAL-002); HU-017 extendida con criterio de temporizador regresivo (QA-VAL-001).
**Cambios v1.1.0:** 5 HUs sugeridas (HU-022…HU-026) formalizadas como HUs oficiales en sus épicas correspondientes. Se elimina la sección "HUs Sugeridas". Cobertura: 26 HUs (21 + 5).

---

## Épica 1: Marcación Facial Móvil (M-01)

### HU-001: Marcación Facial con Prueba de Vida
**Prioridad:** Must

Como Supervisor de Ronda, quiero registrar la asistencia de un trabajador capturando su rostro con prueba de vida desde la App Móvil, para que la marcación quede asociada a la identidad verificada del operario.

**Criterios de aceptación:**
- DADO QUE el Supervisor tiene la App Móvil abierta con un trabajador enrolado seleccionado, CUANDO captura el rostro y ejecuta el gesto aleatorio solicitado en ≤ 5.0 s, ENTONCES el sistema valida `liveness_score >= 0.95` y `match_score >= 0.92`, registra la marcación con `ntp_timestamp` del servidor y actualiza la asistencia con `event_type = SUCCESS`.
- DADO QUE el rostro capturado no supera los umbrales, CUANDO el motor biométrico evalúa la imagen, ENTONCES rechaza la marcación con HTTP 422 y registra `event_type = LIVENESS_FAILED`.
- DADO QUE el gesto aleatorio no se completa dentro de 5.0 s, CUANDO se agota el tiempo, ENTONCES cancela la captura sin registrar marcación y muestra "Tiempo de gesto excedido".

**Reglas de negocio relacionadas:** RN-001A, RN-001B, RN-004
**Notas:** Cumple PRINCIPIO-005.

---

## Épica 2: Geolocalización y Geocerca (M-02)

### HU-002: Validación de Geocerca de 50 m
**Prioridad:** Must

Como Supervisor de Ronda, quiero que la marcación se valide contra un radio de 50 m de la sede cliente, para garantizar que el operario se encuentra físicamente en el puesto de trabajo al marcar.

**Criterios de aceptación:**
- DADO QUE el GPS reporta coordenadas dentro del radio de 50 m y `gps_accuracy <= 25 m`, CUANDO el Supervisor confirma la marcación, ENTONCES registra `geofence_status = INSIDE_GEOFENCE` y aprueba la asistencia.
- DADO QUE el GPS reporta coordenadas fuera del radio de 50 m con `gps_accuracy <= 25 m`, CUANDO el Supervisor confirma la marcación, ENTONCES registra `geofence_status = OUTSIDE_GEOFENCE` y muestra alerta crítica en el Dashboard.
- DADO QUE la distancia calculada es `D = 50.005 m`, CUANDO el backend procesa con epsilon 0.01 m, ENTONCES registra `INSIDE_GEOFENCE`.

**Reglas de negocio relacionadas:** RN-002, RN-017, RN-017-A
**Notas:** Haversine esfera R = 6,371,000 m. Cumple PRINCIPIO-005.

---

### HU-003: Re-captura por Geolocalización Incierta
**Prioridad:** Must

Como Supervisor de Ronda, quiero reintentar la captura de GPS cuando la precisión es insuficiente, para no quedar bloqueado por una lectura imprecisa en zonas techadas.

**Criterios de aceptación:**
- DADO QUE el GPS reporta `gps_accuracy > 25 m`, CUANDO el Supervisor confirma la captura, ENTONCES asigna `UNCERTAIN_GEOLOCATION` y solicita re-captura con ventana de 5 s.
- DADO QUE tras 3 re-capturas consecutivas `gps_accuracy` sigue siendo > 25 m, CUANDO el Supervisor confirma la última, ENTONCES deriva a `REQUIRES_MANUAL_APPROVAL` adjuntando foto y `gps_accuracy` final.
- DADO QUE el operario se desplaza > 10 m entre dos intentos, CUANDO el sistema detecta el desplazamiento, ENTONCES reinicia el contador de re-capturas a 0.

**Reglas de negocio relacionadas:** RN-002-A, RN-002-B, RN-002-C
**Notas:** Cumple PRINCIPIO-009 (auditoría de cada intento).

---

## Épica 3: Anti-Fraude (M-03)

### HU-004: Bloqueo de Ubicación Simulada (Fake GPS)
**Prioridad:** Must

Como Administrador de Operaciones, quiero que el sistema rechace marcaciones originadas en ubicaciones simuladas, para evitar que se registren asistencias desde lugares distintos al puesto de trabajo.

**Criterios de aceptación:**
- DADO QUE la App ejecuta en Android y el proveedor nativo reporta ubicación simulada, CUANDO el Supervisor intenta marcar, ENTONCES rechaza con HTTP 422 y registra `event_type = FAKE_LOCATION_DETECTED`.
- DADO QUE la App ejecuta en iOS y detecta 3 muestras consecutivas en ventana `3 s ≤ Δt_total ≤ 30 s` con alguna señal anómala (`v > 138.89 m/s`, `Δd > 1000 m` en `Δt ≤ 5 s`, inconsistencia BSSID↔GPS > 5 km, `horizontalAccuracy > 500 m`), CUANDO el Supervisor intenta marcar, ENTONCES rechaza con HTTP 422 y audita con muestras crudas.

**Reglas de negocio relacionadas:** RN-003, RN-003-A-1…A-6
**Notas:** Cumple PRINCIPIO-005.

---

### HU-005: Detección de Root y Jailbreak
**Prioridad:** Must

Como Administrador de Operaciones, quiero que el sistema bloquee marcaciones desde dispositivos con root o jailbreak, para evitar manipulación de las APIs de ubicación y del identificador de hardware.

**Criterios de aceptación:**
- DADO QUE la App ejecuta en Android y detecta indicadores de root, CUANDO el Supervisor intenta marcar, ENTONCES rechaza con HTTP 422 y registra `event_type = ROOT_DETECTED`.
- DADO QUE la App ejecuta en iOS y detecta indicadores de jailbreak, CUANDO el Supervisor intenta marcar, ENTONCES rechaza con HTTP 422 y registra `event_type = JAILBREAK_DETECTED`.

**Reglas de negocio relacionadas:** RN-003-B, RN-003-C
**Notas:** Cumple PRINCIPIO-005.

---

### HU-006: Sello de Tiempo NTP con Fallback
**Prioridad:** Must

Como Administrador de Operaciones, quiero que la marca de tiempo de cada marcación provenga del servidor NTP oficial, para invalidar cualquier hora alterada del dispositivo.

**Criterios de aceptación:**
- DADO QUE la App captura una marcación online y el servidor NTP responde en ≤ 2,000 ms, CUANDO se registra el evento, ENTONCES la marca oficial es `ntp_timestamp` en UTC (ISO 8601 con ms).
- DADO QUE la App captura offline y no hay respuesta NTP, CUANDO se registra, ENTONCES la marca se deriva de reloj monotónico + `offset_calibrado` con `|offset| ≤ 5 s` y vigencia ≤ 12 h; si excede, registra `event_type = UNTRUSTWORTHY_TIME` y deriva a `REQUIRES_MANUAL_APPROVAL`.

**Reglas de negocio relacionadas:** RN-004, RN-020, RN-020-A
**Notas:** Cumple PRINCIPIO-005.3.

---

### HU-026: Fallback Manual cuando < 2 Señales Anti-Fraude son Testeables en iOS
**Prioridad:** Must

Como Administrador de Operaciones, quiero que el sistema derive a aprobación manual cuando no hay suficientes señales anti-fraude testeables en iOS, para no rechazar automáticamente marcaciones en zonas rurales o en la primera captura.

**Criterios de aceptación:**
- DADO QUE la App ejecuta en iOS y solo 1 de las 4 señales (RN-003-A-1…A-4) es testeable al momento de la marcación, CUANDO el Supervisor intenta marcar, ENTONCES asigna `REQUIRES_MANUAL_APPROVAL` y notifica al Administrador de Operaciones, sin procesamiento automático.
- DADO QUE exactamente 2 de las 4 señales son testeables, CUANDO el sistema evalúa, ENTONCES aplica RN-003-A normalmente (umbral mínimo inclusivo).

**Reglas de negocio relacionadas:** RN-003-A-5, RN-003-A-6
**Notas:** Cubre el escenario "zona rural sin BSSID + primera captura" que HU-004 solo menciona tangencialmente.

---

## Épica 4: Offline y Sincronización (M-04)

### HU-007: Marcación en Modo Offline
**Prioridad:** Must

Como Supervisor de Ronda, quiero registrar marcaciones sin conexión a internet, para no perder la asistencia cuando estoy en sótanos o zonas remotas sin cobertura.

**Criterios de aceptación:**
- DADO QUE el dispositivo no tiene conexión de red, CUANDO el Supervisor confirma la marcación, ENTONCES la App cifra localmente con mecanismo respaldado por hardware, actualiza `sync_status = PENDING_SYNC` y mantiene el registro con límite de 5,000 registros o 200 MB (el menor).
- DADO QUE la red se restablece, CUANDO el proceso nativo en segundo plano detecta conectividad, ENTONCES transmite en lotes de 50 con backoff exponencial (2 s, 4 s, 8 s; máximo 3 reintentos) y descarta duplicados por terna `(worker_id, ntp_timestamp, device_uuid)` en ventana de 60 s.
- DADO QUE una marcación offline cumple 72 h sin sincronizar, CUANDO el backend la recibe, ENTONCES la clasifica como `REQUIRES_MANUAL_APPROVAL`.
- **DADO QUE el almacenamiento local supera 4,000 registros (80 % de 5,000) o 160 MB (80 % de 200 MB), CUANDO la App detecta la proximidad de saturación, ENTONCES muestra banner rojo parpadeante con el conteo exacto (ej. "⚠️ Almacenamiento local al 81 % (4,050/5,000). Conéctese a red para sincronizar") y, si hay red disponible, fuerza reintento de sincronización (RN-021-A).** *(Nuevo — QA-VAL-002)*

**Reglas de negocio relacionadas:** RN-005, RN-006, RN-019, RN-021, **RN-021-A**
**Notas:** Cumple PRINCIPIO-006 y PRINCIPIO-003.4.


---

### HU-008: Banner de Sincronización Pendiente
**Prioridad:** Must

Como Supervisor de Ronda, quiero ver cuántas marcaciones están pendientes de sincronizar, para saber si debo acercarme a una zona con cobertura.

**Criterios de aceptación:**
- DADO QUE existen `1 ≤ N ≤ 100` marcaciones `PENDING_SYNC`, CUANDO el Supervisor abre la App, ENTONCES ve un banner ámbar con `<count> marcaciones pendientes de sincronizar` y botón "Reintentar ahora".
- DADO QUE existen `N > 100` o `oldest_pending_age > 24 h`, CUANDO el Supervisor abre la App, ENTONCES el banner se muestra en rojo y se actualiza en cada cambio de red, cambio de estado de la App y finalización de sincronización.

**Reglas de negocio relacionadas:** RN-005-G, RN-019
**Notas:** Cumple PRINCIPIO-006.4.

---

### HU-025: Manejo de Reinstalación de App con Pérdida de Claves
**Prioridad:** Must

Como Supervisor de Ronda, quiero que la App detecte la pérdida de claves de cifrado tras una reinstalación, para reportar las marcaciones perdidas al Administrador y re-marcarlas si es posible.

**Criterios de aceptación:**
- DADO QUE la App se reinstala y detecta pérdida de claves de cifrado locales, CUANDO el Supervisor abre la App, ENTONCES emite `event_type = APP_REINSTALL_DATA_LOSS`, contabiliza las marcaciones perdidas y las reporta al Administrador de Operaciones.
- DADO QUE las marcaciones perdidas tienen antigüedad ≤ 72 h, CUANDO el Supervisor las re-marca, ENTONCES el sistema las acepta como marcaciones nuevas con `capture_timestamp` actualizado.
- DADO QUE las marcaciones perdidas tienen antigüedad > 72 h, CUANDO el Supervisor intenta re-marcarlas, ENTONCES el sistema las clasifica como `REQUIRES_MANUAL_APPROVAL` y genera constancia de pérdida según RN-012-D.

**Reglas de negocio relacionadas:** RN-005-H, RN-012-D
**Notas:** La App NO puede "re-sincronizar" datos cuya clave se perdió; solo puede re-marcar o generar constancia.

---

## Épica 5: Dashboard de Monitoreo GPS (M-05)

### HU-009: Visualización en Mapa con Geocerca
**Prioridad:** Must

Como Administrador de Operaciones, quiero ver en un mapa interactivo dónde se realizaron las marcaciones y el radio de 50 m de la sede, para detectar desviaciones del puesto de trabajo sin fricción de credenciales.

**Criterios de aceptación:**
- DADO QUE el Administrador accede con `session_token` firmado (TTL ≤ 12 h, sin usuario/contraseña), CUANDO selecciona una marcación del 15/03, ENTONCES el mapa dibuja un círculo de 50 m centrado en la sede y un pin verde (`INSIDE_GEOFENCE`) o rojo (`OUTSIDE_GEOFENCE`).
- DADO QUE el Administrador aplica filtro por rango de fechas, CUANDO consulta, ENTONCES la respuesta llega en p95 ≤ 1.5 s con hasta 10,000 marcaciones y 50 usuarios concurrentes.
- DADO QUE el token de sesión expira (> 12 h), CUANDO el Administrador intenta acceder, ENTONCES responde HTTP 401 y solicita renovación por canal privilegiado (no login tradicional).

**Reglas de negocio relacionadas:** RN-007, RN-008, RN-013
**Notas:** Cumple PRINCIPIO-002 y RNF-002.

---

## Épica 6: Gestión de Personas y Sedes (M-06)

### HU-010: Enrolamiento con Consentimiento Informado
**Prioridad:** Must

Como Administrador de Operaciones, quiero enrolar trabajadores capturando su foto y consentimiento versionado, para cumplir con la Ley N.° 29733 desde el primer día.

**Criterios de aceptación:**
- DADO QUE el Administrador inicia el enrolamiento, CUANDO captura la foto y el vector del trabajador, ENTONCES el sistema exige `privacy_policy_version`, `consent_timestamp` (UTC), firma digital u OTP y `consent_hash`; sin consentimiento, rechaza con HTTP 403.
- DADO QUE la `privacy_policy_version` cambia, CUANDO el sistema detecta la nueva versión, ENTONCES exige re-consentimiento al 100 % de trabajadores activos en ≤ 30 días naturales; los no consentidos quedan `PENDING_CONSENT`.

**Reglas de negocio relacionadas:** RN-030, RN-030-A, RN-030-B, RN-030-C
**Notas:** Cumple PRINCIPIO-010.

---

### HU-011: Parametrización de Sede con Coordenadas
**Prioridad:** Must

Como Administrador de Operaciones, quiero registrar sedes con coordenadas WGS84, para que la validación de geocerca funcione correctamente.

**Criterios de aceptación:**
- DADO QUE el Administrador registra una sede, CUANDO ingresa `latitude` y `longitude` en WGS84/EPSG:4326, ENTONCES el sistema fija tolerancia de geocerca por defecto en 50 m.
- DADO QUE el Administrador intenta guardar una sede solo con dirección en texto libre o con `latitude = 91`, CUANDO confirma, ENTONCES rechaza la operación.
- DADO QUE las coordenadas coinciden con una sede existente en el mismo tenant, CUANDO confirma, ENTONCES advierte "Sede duplicada" y solicita confirmación.

**Reglas de negocio relacionadas:** RN-010
**Notas:** Cumple PRINCIPIO-005.

---

### HU-022: Fallback Manual para Trabajadores PENDING_CONSENT sin ZKTeco
**Prioridad:** Must

Como Administrador de Operaciones, quiero permitir el registro manual de marcaciones para trabajadores `PENDING_CONSENT` en sedes sin ZKTeco, para no bloquear operativamente al trabajador ni penalizar su tareo mientras se regulariza su consentimiento.

**Criterios de aceptación:**
- DADO QUE un trabajador `PENDING_CONSENT` opera en una sede sin ZKTeco, CUANDO el Supervisor intenta marcar, ENTONCES el sistema permite el registro manual con `event_type = MANUAL_CANCELLATION` y justificación obligatoria (mínimo 30 caracteres).
- DADO QUE el Supervisor registró una marcación manual, CUANDO el Administrador de Operaciones revisa, ENTONCES debe aprobar o rechazar en ≤ 24 h; si no aprueba en ese plazo, la marcación pasa a `REQUIRES_MANUAL_APPROVAL` con alerta crítica.
- DADO QUE el trabajador regulariza su consentimiento, CUANDO el Administrador lo confirma, ENTONCES el trabajador pasa a `ACTIVE` y puede usar la marcación biométrica.

**Reglas de negocio relacionadas:** RN-030-D, RN-030-A, RN-030-B
**Notas:** Cubre el gap operativo de trabajadores sin ZKTeco. Cumple PRINCIPIO-010 (revocación sin penalización).

---

## Épica 7: Integración ZKTeco (M-07)

### HU-012: Recepción Asíncrona de Marcaciones ZKTeco
**Prioridad:** Must

Como Administrador de Operaciones, quiero que el backend reciba marcaciones de relojes ZKTeco de forma asíncrona, para no degradar el rendimiento del servidor bajo carga.

**Criterios de aceptación:**
- DADO QUE un reloj ZKTeco envía una marcación vía ADMS Push, CUANDO el backend la recibe, ENTONCES responde HTTP 200 OK con cuerpo "OK" en p95 ≤ 100 ms y p99 ≤ 250 ms, y deriva a procesamiento asíncrono.
- DADO QUE un `serial_number` excede 10 RPS, CUANDO llega un nuevo payload, ENTONCES responde HTTP 429 con `Retry-After: 5`; si el payload excede 10 MB o 5,000 registros, responde HTTP 413 con `max_records_per_request = 5000`.
- DADO QUE un reloj deja de emitir heartbeat durante > 10 min, CUANDO se cumple el umbral, ENTONCES el sistema cambia a `BIOMETRIC_DEVICE_OFFLINE` y habilita la marcación facial móvil como contingencia.

**Reglas de negocio relacionadas:** RN-011, RN-011-A, RN-011-B, RN-018
**Notas:** Cumple PRINCIPIO-004.

---

## Épica 8: Seguridad de Dispositivo (M-08)

### HU-013: Vinculación Primaria + Backup del Dispositivo
**Prioridad:** Must

Como Administrador de Operaciones, quiero que cada Supervisor tenga un dispositivo primario y uno de respaldo pre-enrolado, para garantizar continuidad operativa si el primario falla.

**Criterios de aceptación:**
- DADO QUE un Supervisor tiene primario y backup enrolados, CUANDO el primario se destruye y el Supervisor activa el backup, ENTONCES el sistema exige OTP al correo (entrega ≤ 2 min) y biometría del supervisor, completando en ≤ 5 min.
- DADO QUE un `device_uuid` no enrolado o revocado intenta marcar, CUANDO el Supervisor confirma, ENTONCES muestra "Dispositivo no autorizado" y registra `event_type = DEVICE_UNAUTHORIZED`; sin red, retiene como `PENDING_DEVICE_ACTIVATION` hasta 72 h.

**Reglas de negocio relacionadas:** RN-012, RN-012-A, RN-012-B
**Notas:** Cumple PRINCIPIO-005 y PRINCIPIO-010.

---

### HU-014: Constancia de Pérdida de Asistencia
**Prioridad:** Must

Como Administrador de Operaciones, quiero generar una constancia por cada trabajador afectado cuando ocurre una pérdida de datos locales, para proteger los derechos laborales del operario.

**Criterios de aceptación:**
- DADO QUE ocurrió `DATA_LOST_DEVICE_LOST`, CUANDO el Administrador procesa el evento, ENTONCES el sistema genera una constancia de pérdida por cada trabajador afectado en ≤ 48 h y registra `event_type = ATTENDANCE_LOSS_CONSTANCY`.
- DADO QUE el trabajador invoca la constancia, CUANDO la presenta al Analista de Planillas, ENTONCES puede impugnar el tareo del periodo afectado sin penalización.

**Reglas de negocio relacionadas:** RN-012-C, RN-012-D
**Notas:** Cumple PRINCIPIO-010.

---

## Épica 9: Cumplimiento ARCO y Consentimiento (M-09)

### HU-015: Atención de Solicitud ARCO con Verificación de Identidad
**Prioridad:** Must

Como DPO, quiero atender solicitudes ARCO con verificación de identidad del titular, para cumplir con la Ley N.° 29733 sin exponer datos a terceros.

**Criterios de aceptación:**
- DADO QUE se recibe una solicitud ARCO, CUANDO el sistema procesa, ENTONCES verifica la identidad del titular mediante firma digital y/o DNI + OTP + biometría contra `reference_embedding`; sin verificación, rechaza con HTTP 401 y registra `event_type = ARCO_IDENTITY_UNVERIFIED`.
- DADO QUE la solicitud es de Acceso, CUANDO se verifica la identidad, ENTONCES responde en ≤ 10 días hábiles con la totalidad de los datos del titular en formato estructurado.
- DADO QUE la solicitud es de Cancelación, CUANDO se verifica, ENTONCES ejecuta: (a) a las 24 h `deleted_at` e inaccesible; (b) revocación inmediata y atómica de la clave AES-256 en KMS; (c) eliminación física a los 30 días; (d) `audit_records` conserva `worker_hash` 5 años; (e) `worker_legal_identity` 5 años.
- DADO QUE la solicitud es de Rectificación, CUANDO se verifica, ENTONCES corrige en ≤ 5 días hábiles y registra `event_type = ARCO_RECTIFICATION`.

**Reglas de negocio relacionadas:** RN-031, RN-032, RN-033, RN-033-A…E, RN-034, RN-035, RN-035-A, RN-035-B, RN-037
**Notas:** Cumple PRINCIPIO-010.

---

### HU-023: Revocación de Consentimiento Biométrico sin Cese Laboral
**Prioridad:** Must

Como trabajador, quiero revocar mi consentimiento biométrico sin cesar la relación laboral, para ejercer mi derecho según la Ley N.° 29733 sin penalización laboral.

**Criterios de aceptación:**
- DADO QUE un trabajador activo revoca su consentimiento biométrico sin cesar la relación laboral, CUANDO el sistema procesa la revocación, ENTONCES (a) suspende el uso de biometría, (b) deriva al trabajador a ZKTeco o marcación por supervisor, (c) registra `event_type = CONSENT_REVOKED`, y (d) preserva `audit_logs` con `worker_id` real 5 años si la biometría se mantiene o `worker_hash` si se elimina.
- DADO QUE la revocación se completó, CUANDO el trabajador intenta marcar por biometría, ENTONCES el sistema bloquea la marcación biométrica y sugiere los canales alternos (ZKTeco o supervisor).
- DADO QUE la revocación ocurrió, CUANDO el Analista de Planillas procesa el tareo, ENTONCES el trabajador NO recibe penalización ni descuento por la revocación.

**Reglas de negocio relacionadas:** RN-036-A, RN-036-B, RN-036-C, RN-036-D
**Notas:** Cumple PRINCIPIO-010.

---

### HU-024: Gobernanza Multi-Tenant del Log de Auditoría
**Prioridad:** Must

Como DPO, quiero que el log de auditoría preserve la trazabilidad del tenant original tras reasignación de sede, fusión o eliminación de tenants, para cumplir con la retención legal de 5 años sin exponer datos entre tenants.

**Criterios de aceptación:**
- DADO QUE una sede se reasigna del tenant A al tenant B, CUANDO se consulta el `audit_logs` de la sede, ENTONCES los eventos históricos conservan `tenant_id = A` y son accesibles en solo lectura por los DPOs de ambos tenants.
- DADO QUE ocurre una fusión de tenants A+B, CUANDO se consulta el `audit_logs` del tenant fusionado, ENTONCES los eventos históricos de A conservan `tenant_id = A` y añaden `merged_from_tenant_id = A`.
- DADO QUE un tenant se elimina, CUANDO se ejecuta la retención, ENTONCES los eventos se conservan 5 años en almacenamiento cifrado con la clave del tenant eliminado, accesibles solo por el DPO de ese tenant.

**Reglas de negocio relacionadas:** RN-035-A, RN-035-B
**Notas:** Cumple PRINCIPIO-010 (gobernanza multi-tenant).

---

## Épica 10: Autenticación del Supervisor (M-10)

### HU-016: Autenticación Primaria del Supervisor
**Prioridad:** Must

Como Administrador de Operaciones, quiero que el Supervisor se autentique con biometría de plataforma + PIN al inicio de jornada, para que el `device_uuid` no sea la única barrera de seguridad.

**Criterios de aceptación:**
- DADO QUE el Supervisor inicia jornada, CUANDO se autentica con PIN + biometría de plataforma, ENTONCES el sistema habilita la marcación y emite un `session_token` con TTL ≤ 12 h.
- DADO QUE el dispositivo está autorizado pero el Supervisor no se ha autenticado, CUANDO intenta marcar, ENTONCES el sistema bloquea la operación (el `device_uuid` es control secundario).
- DADO QUE el Supervisor ingresa un PIN incorrecto 5 veces, CUANDO se cumple el intento, ENTONCES bloquea el acceso hasta re-autenticación con biometría de plataforma.

**Reglas de negocio relacionadas:** RN-012, RN-014, RN-014-A
**Notas:** Cumple PRINCIPIO-005 y PRINCIPIO-010.

---

## Épica 11: Bloqueo Temporal (S-05)

### HU-017: Bloqueo Temporal tras Intentos Fallidos
**Prioridad:** Should

Como Administrador de Operaciones, quiero que el sistema bloquee temporalmente una terminal tras múltiples intentos de fraude, para mitigar ataques de fuerza bruta o manipulación.

**Criterios de aceptación:**
- DADO QUE un `device_uuid` acumula 3 intentos rechazados por `FAKE_LOCATION_DETECTED` o `LIVENESS_FAILED` en 15 min, CUANDO ocurre el tercero, ENTONCES el sistema bloquea la terminal 30 min y registra `event_type = DEVICE_TEMPORARILY_BLOCKED`.
- DADO QUE el contador de RN-014 está activo, CUANDO ocurre una marcación `SUCCESS`, pasan 60 min sin intentos rechazados, o el Supervisor se re-autentica, ENTONCES el contador se reinicia a 0.
- DADO QUE el bloqueo de 30 min expira, CUANDO el Supervisor intenta marcar, ENTONCES el sistema permite el intento sin re-bloqueo automático.
- **DADO QUE la terminal está bloqueada por 3 intentos fallidos, CUANDO el Supervisor abre la App, ENTONCES ve un temporizador regresivo `MM:SS` (ej. "Intente de nuevo en 29:45") con el tiempo restante de bloqueo, con los botones de autenticación deshabilitados hasta que expire el contador.** *(Nuevo — QA-VAL-001)*

**Reglas de negocio relacionadas:** RN-014, RN-014-A
**Notas:** Cumple PRINCIPIO-005.

---

## Épica 12: Alertas Operativas (S-01)

### HU-018: Alerta en el Dashboard por Marcación Fuera de Geocerca
**Prioridad:** Should

Como Administrador de Operaciones, quiero recibir una alerta visual en el Dashboard cuando una marcación cae fuera de la geocerca, para priorizar la revisión operativa.

**Criterios de aceptación:**
- DADO QUE una marcación se registra con `geofence_status = OUTSIDE_GEOFENCE`, CUANDO el Administrador abre el Dashboard, ENTONCES la marcación aparece como alerta visual con pin rojo y distancia calculada.
- DADO QUE no hay marcaciones `OUTSIDE_GEOFENCE` en el rango consultado, CUANDO el Administrador abre el Dashboard, ENTONCES muestra el estado vacío "Sin marcaciones fuera de geocerca".

**Reglas de negocio relacionadas:** RN-008, RN-013
**Notas:** Cumple RNF-002.

---

## Épica 13: Consolidación de Tareo (S-02)

### HU-019: Generación de Reporte de Tareo para Pre-Planilla
**Prioridad:** Should

Como Analista de Planillas, quiero descargar el reporte de tareo del periodo cerrado, para cargarlo en el módulo de planilla sin re-digitar.

**Criterios de aceptación:**
- DADO QUE el Administrador cierra un periodo de tareo, CUANDO solicita la generación del reporte, ENTONCES el sistema produce el consolidado de asistencia, tardanzas y sobretiempos en ≤ 15 min con formato S/ y fecha DD/MM/AAAA.
- DADO QUE existen marcaciones `REQUIRES_MANUAL_APPROVAL` en el periodo, CUANDO el Administrador genera el reporte, ENTONCES el sistema no incluye esas marcaciones en el consolidado hasta que sean aprobadas.
- DADO QUE un trabajador tiene 30 días de antigüedad y solo 15 marcaciones registradas, CUANDO se genera el reporte, ENTONCES marca las faltas (días sin marcación) explícitamente.

**Reglas de negocio relacionadas:** RN-002-B, RN-019, RN-020-A
**Notas:** Cumple PRINCIPIO-008.

---

## Épica 14: Exportación de Reportes (S-03)

### HU-020: Exportación de Reportes Auditables
**Prioridad:** Should

Como Administrador de Operaciones, quiero exportar los registros de auditoría en CSV/PDF, para evidenciar el cumplimiento ante inspecciones del MTPE o auditorías del cliente.

**Criterios de aceptación:**
- DADO QUE el Administrador solicita exportación de marcaciones, CUANDO confirma el periodo y filtros, ENTONCES el sistema genera CSV/PDF con los 8 campos obligatorios de PRINCIPIO-009.
- DADO QUE el Administrador solicita exportación del log de auditoría, CUANDO confirma el periodo, ENTONCES incluye los eventos rechazados, `event_type` y `device_info`.
- DADO QUE el Administrador solicita exportación de un periodo de otro tenant, CUANDO lo intenta, ENTONCES rechaza con HTTP 403 sin filtrar datos.

**Reglas de negocio relacionadas:** RN-016
**Notas:** Cumple PRINCIPIO-009.

---

## Épica 15: Re-verificación Server-Side (S-04)

### HU-021: Activación de Re-verificación Biométrica Server-Side
**Prioridad:** Should

Como Administrador de Operaciones, quiero activar la re-verificación biométrica en backend cuando mi tenant lo requiera, para tener una segunda capa de control en sectores críticos.

**Criterios de aceptación:**
- DADO QUE el feature flag `server_side_reverification` está activo, CUANDO la App completa una captura facial, ENTONCES el backend re-verifica el vector contra el maestro en ≤ 500 ms, persiste `server_match_score` y, si `|client_match_score − server_match_score| > 0.05`, registra `event_type = BIOMETRIC_INCONSISTENCY` y rechaza.
- DADO QUE el flag está inactivo, CUANDO la App completa una captura, ENTONCES el backend no transmite fotogramas crudos y opera solo con el vector del dispositivo.
- DADO QUE el flag está activo y se transmiten fotogramas crudos, CUANDO se persisten, ENTONCES se cifran AES-256 + TLS 1.3 y se purgan ≤ 24 h.

**Reglas de negocio relacionadas:** RN-001C
**Notas:** Cumple PRINCIPIO-005 y RNF-006.

---

## Verificación INVEST del Catálogo de HUs

- **Independent:** Cada HU corresponde a un módulo funcional distinto. La única dependencia técnica es que HU-001 requiere que el worker biométrico esté operativo, pero se puede desarrollar y probar con mocks.
- **Negotiable:** El detalle de UI/UX es flexible; los criterios fijan el comportamiento observable (valores, códigos HTTP, `event_type`).
- **Valuable:** Cada HU aporta valor operativo o de cumplimiento.
- **Estimable:** Los criterios tienen valores cuantitativos que permiten estimación por puntos de historia.
- **Small:** Cada HU cabe en un sprint.
- **Testable:** 100% verificable con los criterios y valores concretos.

---

## Resumen de Cobertura

- Total de HUs generadas: **26**
- Must cubiertas: **10/10** (M-01…M-10)
- Should cubiertas: **5/5** (S-01…S-05)
- Reglas del `03-reglas-de-negocio.md` referenciadas: **67/67**
- HUs con al menos 2 criterios: **26/26**
- HUs con trazabilidad a ≥ 1 regla: **26/26**
- HUs con valores cuantitativos coherentes: **26/26**
- Cumplimiento constitucional: P-001, P-002, P-003, P-004, P-005, P-006, P-008, P-009, P-010

---

**Cambios v1.1.0:**
- HU-022 (fallback sin ZKTeco) movida a Épica E-06
- HU-023 (revocación sin cese) movida a Épica E-09
- HU-024 (multi-tenant audit) movida a Épica E-09
- HU-025 (reinstalación) movida a Épica E-04
- HU-026 (fallback iOS) movida a Épica E-03
- Se elimina la sección "HUs Sugeridas"
- Total: 26 HUs oficiales