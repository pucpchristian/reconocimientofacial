# Constitución del Proyecto: Módulo Operativo y Control de Asistencia Omnicanal

**Versión:** 2.2.2
**Fecha:** 11 de Noviembre de 2026
**Cambios v2.2.2:** PRINCIPIO-006.4 añade sub-punto de alerta de saturación al 80 % (QA-VAL-002).
**Cambios v2.2.1:** PRINCIPIO-009 separa "Estados" de "Tipos de evento" y ordena alfabéticamente. PRINCIPIO-010 añade verificación de identidad ARCO (RN-037). PRINCIPIO-005 alinea fallbacks NTP. Aplicados 18 hallazgos QA v2.2.0.
**Cambios v2.2.0:** Arquitectura removida.
**Cambios v2.1.0:** PRINCIPIO-010 (ARCO, Consentimiento y Retención).
**Cambios v2.0.0:** App Móvil Nativa/Híbrida; identificadores en inglés, docs/UI en Español Neutro.

## Visión y Alcance
Reglas no negociables de negocio, seguridad y gobernanza para el **Módulo Operativo y Control de Asistencia Omnicanal** de la plataforma RH/ERP en Perú. **Las decisiones de arquitectura (stack, librerías, protocolos concretos, esquemas y APIs) se documentan en un artefacto separado posterior.**

---

### PRINCIPIO-001: Idioma Estándar

Todo el código fuente (backend, dashboard web, app móvil, scripts, infraestructura como código) DEBE cumplir:

1. **Identificadores en inglés:** variables, funciones, métodos, clases, tipos, interfaces, campos de datos, nombres de archivo, commits y ramas Git.
2. **Documentación, comentarios, descripciones de tests y UI en Español Latinoamericano Neutro.**
3. **Excepciones:** APIs y marcas de terceros se referencian en su idioma original. Siglas estándar (`NTP`, `HTTP`, `TLS`, `AES`, `GPS`, `UUID`) no se traducen.

* **Cumplimiento:** entidad `Employee` con campos `full_name`, `hire_date`, `document_number`; test describe `"debe calcular la antigüedad del empleado"`.
* **Violación:** clase `Empleado` con método `calcularAntiguedad()`.

---

### PRINCIPIO-002: Frontend de Monitoreo GPS sin Login Tradicional

El Dashboard de Monitoreo Operativo DEBE funcionar mediante token de sesión firmado de vigencia limitada vinculado al entorno autorizado, sin login convencional con contraseña. DEBE renderizar mapa interactivo con: (a) posicionamiento de marcaciones; (b) filtro por rango de fechas e identificador de trabajador; (c) superposición explícita del radio de geocerca (50 m) respecto al punto de marcación.

* **Cumplimiento:** al seleccionar una marcación, el mapa dibuja un círculo de 50 m y ubica el pin en las coordenadas capturadas.
* **Violación:** mostrar listado plano sin coordenadas o requerir usuario/contraseña en kiosco.

---

### PRINCIPIO-003: Cifrado y Protección de Datos Biométricos y Ubicación (Ley 29733)

1. **Cifrado en Reposo:** AES-256 para fotos y vectores faciales. Cifrado transparente de volumen y base de datos.
2. **Cifrado en Tránsito:** TLS 1.3 obligatorio en toda comunicación. TLS 1.2 NO es aceptado.
3. **Búsqueda:** índices de trigramas sobre datos legibles ya protegidos por cifrado de volumen.
4. **Almacenamiento Local Móvil:** datos persistidos DEBEN cifrarse mediante mecanismos criptográficos respaldados por hardware de la plataforma.

* **Cumplimiento:** vectores cifrados AES-256 + TLS 1.3 forzado.
* **Violación:** fotos en carpeta pública o GPS por HTTP plano; TLS 1.2.

---

### PRINCIPIO-004: Integración Desacoplada e Idempotente con Biométricos ZKTeco

1. **Idempotencia:** deduplicar marcaciones repetidas en ventana de 60 s por combinación `(worker_id, ntp_timestamp, device_uuid)`.
2. **Resiliencia Offline:** si cae la red local, las marcaciones se acumulan en el dispositivo y se procesan en lote sin perder el orden.
3. **Capacidad:** endpoint receptor DEBE sostener **p95 ≤ 100 ms y p99 ≤ 250 ms a 500 RPS**, con ráfagas de 1,000 RPS durante ≤ 10 s.

* **Cumplimiento:** el procesador descarta 2 duplicados de 3 paquetes idénticos.
* **Violación:** procesamiento síncrono bloqueante.

---

### PRINCIPIO-005: Marcación Facial Móvil Anti-Fraude

1. **Prueba de Vida:** DEBE exigir gesto activo aleatorio con `liveness_score >= 0.95` en ≤ 5.0 s.
2. **Bloqueo de Ubicación Falsa:**
   - **Android:** DEBE validar el proveedor nativo de la plataforma.
   - **iOS:** DEBE aplicar heurísticas con consenso de **3 muestras consecutivas** en ventana `3 s ≤ Δt_total ≤ 30 s`, según RN-003-A.
   - **Ambas plataformas:** DEBE detectar jailbreak (iOS) y root (Android).
3. **Sello de Tiempo NTP:** la marca primaria DEBE ser el `ntp_timestamp` oficial del servidor. Los fallbacks autorizados son: (a) RN-020 para captura offline; (b) RN-020-A para captura online con fallo NTP. En ambos casos se requiere `|offset_calibrado| ≤ 5 s` y vigencia ≤ 12 h. La hora local del sistema operativo NUNCA se usa sin calibración.

* **Cumplimiento:** rechazar marcación si el proveedor nativo reporta ubicación simulada o si se detectan 3 muestras anómalas en iOS.
* **Violación:** usar la hora del sistema operativo sin calibración NTP.

---

### PRINCIPIO-006: Operación Offline-First en Marcaciones Móviles

1. Datos locales DEBEN cifrarse con mecanismos respaldados por hardware.
2. La marca de tiempo del evento se deriva de reloj monotónico + `offset_calibrado` y se reconcilia con `ntp_timestamp` al sincronizar.
3. Al recuperar red, la App DEBE sincronizar en segundo plano con idempotencia.
4. Si la plataforma limita la ejecución en segundo plano, la App DEBE sincronizar al primer evento de primer plano y **mostrar banner persistente con el número exacto de marcaciones pendientes** (RN-005-G).
5. **Alerta de saturación:** MIENTRAS el almacenamiento local cifrado supere el 80 % de su capacidad (4,000 de 5,000 registros, o 160 de 200 MB, el menor), la App DEBE mostrar alerta visual roja parpadeante con el conteo exacto (`registros_pendientes`, `uso_disco_mb`) y, si hay red disponible, forzar reintento de sincronización antes de alcanzar el bloqueo total (RN-021, RN-021-A). El banner DEBE actualizarse en cada cambio de red, cambio de estado de App y finalización de sincronización.

* **Cumplimiento:** marcación en sótano cifrada localmente y sincronizada 10 min después; al alcanzar 4,050 registros locales, la App muestra banner rojo parpadeante "⚠️ Almacenamiento local al 81 % (4,050/5,000). Conéctese a red para sincronizar".
* **Violación:** datos en texto plano, bloquear la marcación sin conexión, o mantener el banner ámbar (`N=1…100`) cuando el almacenamiento local ya superó el 80 %.

---

### PRINCIPIO-007: Restricciones Estrictas para Agentes de IA

Ningún agente de IA puede:
1. Modificar scripts de base de datos o migraciones sin revisión explícita por Pull Request.
2. Desactivar, comentar o alterar pruebas para forzar la aprobación de una compilación.
3. Introducir claves secretas, tokens, llaves de cifrado o credenciales en el código fuente.
4. Generar o responder con datos reales de empleados en pruebas o prompts.

* **Cumplimiento:** agente genera scripts E2E con datos sintéticos y abre PR con revisión humana.
* **Violación:** agente deshabilita el test de detección de ubicación falsa para pasar CI.

---

### PRINCIPIO-008: Entorno de Pruebas de Flujo (Suite E2E y Datos Simulados)

1. Scripts deterministas que simulen marcaciones, fraudes, fallos de liveness y marcaciones ZKTeco offline.
2. Datos de renderizado: exclusivamente personas sintéticas.
3. Pruebas móviles: emuladores con proveedor de ubicación simulada controlado.
4. Cobertura mínima: **≥ 80 % backend, ≥ 70 % App Móvil**.

* **Cumplimiento:** suite E2E valida pin rojo para marcación fuera de geocerca con datos sintéticos.
* **Violación:** fotos y DNIs reales en pruebas.

---

### PRINCIPIO-009: Auditabilidad Inmutable de Intentos y Modificaciones

Cualquier intento de marcación y cualquier modificación manual DEBE registrarse en un **log de auditoría inmutable** (solo inserción).

**Campos obligatorios:** `event_id`, `worker_id`, `event_type`, `gps_coordinates`, `geofence_status`, `ip_address`, `device_info` (`{device_uuid, os_version, app_version, model}`), `utc_timestamp`.

**Tipos de evento válidos** (orden alfabético):
`APP_REINSTALL_DATA_LOSS`, `ARCO_CANCELLATION`, `ARCO_IDENTITY_UNVERIFIED`, `ARCO_OPPOSITION`, `ARCO_RECTIFICATION`, `ATTENDANCE_LOSS_CONSTANCY`, `BIOMETRIC_INCONSISTENCY`, `BIOMETRIC_SDK_UNAVAILABLE`, `CONSENT_REVOKED`, `DATA_LOST_DEVICE_LOST`, `DATA_PURGED`, `DEVICE_UNAUTHORIZED`, `FAKE_LOCATION_DETECTED`, `JAILBREAK_DETECTED`, `LIVENESS_FAILED`, `MANUAL_CANCELLATION`, `OUTSIDE_GEOFENCE`, `ROOT_DETECTED`, `SUCCESS`, `UNCERTAIN_GEOLOCATION`, `UNTRUSTWORTHY_TIME`, `WORKER_NOT_ENROLLED`.

**Estados de entidad** (no son tipos de evento; se persisten como campos de estado):
- **Estado de trabajador:** `PENDING_CONSENT`.
- **Estado de marcación:** `PENDING_SYNC`, `PENDING_TEMPORAL_VALIDATION`, `PENDING_DEVICE_ACTIVATION`, `REQUIRES_MANUAL_APPROVAL`.
- **Estado de dispositivo:** `AUTHORIZED`, `UNAUTHORIZED`, `TEMPORARILY_BLOCKED`, `BIOMETRIC_DEVICE_OFFLINE`, `LOST_OR_DESTROYED`.

* **Cumplimiento:** intento de ubicación falsa registrado con `event_type = FAKE_LOCATION_DETECTED` y coordenadas declaradas vs reales.
* **Violación:** descartar una marcación fallida sin dejar rastro.

---

### PRINCIPIO-010: Cumplimiento ARCO, Consentimiento y Retención (Ley N.° 29733)

1. **Consentimiento:** versionado, firmado digitalmente u OTP; `consent_hash = hash_cryptográfico(policy_version + worker_id + consent_timestamp)`.
2. **Derechos ARCO:** operaciones con SLA legal. **Toda solicitud ARCO DEBE verificar la identidad del titular** mediante firma digital y/o DNI + OTP + verificación biométrica contra el vector de referencia (RN-037). Sin verificación, la solicitud DEBE rechazarse con HTTP 401.
3. **Retención:** biometría = duración laboral + 30 días naturales; asistencia/auditoría = 5 años.
4. **Cancelación ARCO:** (a) a las 24 h, marcar biometría `deleted_at` e inaccesible; (b) revocar inmediatamente la clave AES-256; (c) eliminar físicamente los datos cifrados a los 30 días; (d) preservar el log de auditoría con `worker_hash = hash_cryptográfico(worker_id + per_tenant_salt)`; (e) conservar identidad legal mínima 5 años.
5. **Gobernanza multi-tenant:** los eventos de auditoría heredan el `tenant_id` original.

* **Cumplimiento:** trabajador cesado el 01/10/2026 → biometría inaccesible 02/10/2026, clave revocada inmediatamente, datos eliminados 31/10/2026; auditoría con `worker_hash` por 5 años.
* **Violación:** biometría activa 5 años después del cese; atender ARCO sin verificar identidad.