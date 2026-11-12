# Catálogo de Reglas de Negocio: Módulo Operativo y Control de Asistencia Omnicanal

**Proyecto:** Ecosistema RH / ERP — Módulo Operativo
**Documentos de Referencia:** `00-constitucion.md`, `01-project-brief.md`, `02-alcance.md`, `04-glosario.md`
**Versión:** 2.2.1
**Fecha:** 10 de Noviembre de 2026
**Estado:** Aprobado para Diseño de Software
**Cambios v2.2.2:** Añadida RN-021-A (alerta de saturación al 80 %). Aplicada observación QA-VAL-002.
**Cambios v2.2.1:** Aplicados 18 hallazgos QA v2.2.0. RN-003-A desdoblada en A-1…A-6. RN-033 desdoblada en A-E. RN-036 desdoblada en A-D. Nuevas RN-030-C, RN-030-D, RN-037. RN-004 reformulada con fallbacks. RN-005-H reformulada. RN-019 aclarada. Matriz de trazabilidad añadida.

---

## Catálogo de Reglas de Negocio (Notación EARS)

### Biometría

**RN-001A** | Categoría: Biometría | Prioridad: MUST | EARS: Event-driven

CUANDO un Supervisor capture un rostro en la App Móvil, el motor biométrico DEBE exigir una verificación activa de prueba de vida (Liveness Detection) en <= 5.0 s, requiriendo `liveness_score >= 0.95`. 
El parámetro `tipo_gesto` DEBE tomar obligatoriamente un valor del tipo enumerado estandarizado (`PARPADEO_DOBLE`, `GUINO_IZQUIERDO`, `GUINO_DERECHO`, `INCLINACION_CABEZA`).

✅ Gesto `PARPADEO_DOBLE` en 3 s con `liveness_score = 0.98` → aprobada.
❌ Procesar fotograma de foto impresa sin gesto validado.

---

**RN-001B** | Categoría: Biometría | Prioridad: MUST | EARS: Event-driven

CUANDO el motor evalúe la semejanza vectorial entre captura y foto de referencia, DEBE aprobar solo si la tasa de coincidencia es `>= 92.0%` con una tasa de falsa aceptación (`FAR`) `<= 0.001%`.

✅ `match_score = 0.945` → confirmada.
❌ `match_score = 0.84` aceptado o FAR no verificado.



---

**RN-001C** | Categoría: Biometría | Prioridad: MUST | EARS: Event-driven

CUANDO la App complete la captura facial, `face_embedding` y `liveness_score` DEBEN generarse en el dispositivo. El backend DEBE re-verificar contra `reference_embedding` en ≤ 500 ms y persistir `server_match_score`. SI `|client_match_score − server_match_score| > 0.05`, DEBE registrar `event_type = BIOMETRIC_INCONSISTENCY`. Los fotogramas crudos solo se transmiten si `server_side_reverification = true` (AES-256 + TLS 1.3; purga ≤ 24 h).

✅ cliente=0.94, servidor=0.93 → aprobada.
❌ cliente=0.93, servidor=0.80 → `BIOMETRIC_INCONSISTENCY` + rechazo.

---

**RN-001D** | Categoría: Biometría | Prioridad: MUST | EARS: Event-driven

SI el motor de liveness no inicializa tras 2 reintentos, la App DEBE (a) registrar `event_type = BIOMETRIC_SDK_UNAVAILABLE`, (b) permitir marcación bajo `REQUIRES_MANUAL_APPROVAL` con foto, (c) notificar al Administrador.

✅ Fallo del motor → marcación con foto + alerta.
❌ Error genérico que bloquea sin alternativa.

---

### Geolocalización

**RN-002** | Categoría: Geolocalización | Prioridad: MUST | EARS: State-driven

MIENTRAS el sistema evalúe la ubicación GPS, DEBE calcular la distancia aplicando Haversine (esfera R = 6,371,000 m) con coordenadas WGS84/EPSG:4326 respecto al centroide de la sede. SI `gps_accuracy > 25 m`, NO emitirá veredicto definitivo; asignará `UNCERTAIN_GEOLOCATION` y aplicará RN-002-A/B/C.

✅ distancia 34.2 m, `gps_accuracy = 8 m` → `INSIDE_GEOFENCE`.
❌ `gps_accuracy = 45 m` rechazado como `OUTSIDE_GEOFENCE` sin aplicar RN-002-A/B/C.

---

**RN-002-A** | Categoría: Geolocalización | Prioridad: MUST | EARS: State-driven

MIENTRAS una marcación esté `UNCERTAIN_GEOLOCATION`, la App DEBE permitir máximo 3 re-capturas consecutivas (ventana 5 s cada una). Si el operario se desplaza > 10 m, el contador DEBE reiniciarse.

✅ 3 re-capturas en la misma posición → aplica RN-002-B.
❌ Re-capturas indefinidas.

---

**RN-002-B** | Categoría: Geolocalización | Prioridad: MUST | EARS: Event-driven

SI tras 3 re-capturas `gps_accuracy > 25 m`, la marcación DEBE pasar a `REQUIRES_MANUAL_APPROVAL` con foto + `gps_accuracy` final.

✅ Garita techada con accuracy=30 m → revisión manual.
❌ Rechazo automático sin alternativa.

---

**RN-002-C** | Categoría: Geolocalización | Prioridad: MUST | EARS: Ubiquitous

Toda marcación con `attempts > 1` DEBE registrar en el log de auditoría `attempt_count` y los `gps_accuracy` de cada intento.

✅ `attempts=3`, `accuracies=[28,31,30]` → 3 intentos auditados.
❌ Solo auditar el último intento.

---

**RN-010** | Categoría: Datos Maestros | Prioridad: MUST | EARS: Event-driven

CUANDO un Administrador registre una nueva sede, DEBE exigir `latitude` y `longitude` (WGS84/EPSG:4326) y tolerancia de geocerca por defecto 50 m.

✅ Sede "San Isidro" con `latitude = -12.093100`, `longitude = -77.036500`, radio 50 m.
❌ Solo dirección en texto libre.

---

**RN-017** | Categoría: Evaluador Geodésico | Prioridad: MUST | EARS: Event-driven

CUANDO el backend procese distancia geodésica con `gps_accuracy <= 25 m`, SI `D ≤ 50.00 m` (aplicando RN-017-A), DEBE registrar `INSIDE_GEOFENCE`; SI `D > 50.00 m`, DEBE registrar `OUTSIDE_GEOFENCE`.

✅ `D = 50.00 m` → `INSIDE_GEOFENCE`.
❌ Registrar 52.00 m como `INSIDE_GEOFENCE`.

---

**RN-017-A** | Categoría: Evaluador Geodésico | Prioridad: MUST | EARS: Ubiquitous

Toda comparación de distancia DEBE usar epsilon de `0.01 m`: `D ≤ 50.00 + 0.01` se considera `INSIDE_GEOFENCE`.

✅ `D = 50.005 m` → `INSIDE_GEOFENCE`.
❌ `D = 50.0000001 m` → `OUTSIDE_GEOFENCE` por error de precisión.

---

### Anti-Fraude

**RN-003** | Categoría: Anti-Fraude | Prioridad: MUST | EARS: Unwanted

SI la App detecta que la ubicación proviene de una fuente simulada (validación del proveedor nativo en Android; heurísticas RN-003-A en iOS), DEBE rechazar con HTTP 422 y registrar `event_type = FAKE_LOCATION_DETECTED`.

✅ Proveedor nativo reporta simulación → bloqueo + auditoría.
❌ Backend procesa coordenadas simuladas.

---

**RN-003-A** | Categoría: Anti-Fraude | Prioridad: MUST | EARS: Unwanted

CUANDO la App ejecute en iOS y detecte **3 muestras consecutivas** en ventana `3 s ≤ Δt_total ≤ 30 s` que cumplan al menos una condición de RN-003-A-1…A-4, DEBE clasificar como `FAKE_LOCATION_DETECTED`, rechazar con HTTP 422 y auditar con muestras crudas.

✅ 3 muestras con `Δd=1,500 m` en `Δt=3 s` → bloqueo + HTTP 422.
❌ Lectura anómala aislada → NO bloquea.

---

**RN-003-A-1** | Categoría: Anti-Fraude | Prioridad: MUST | EARS: State-driven

MIENTRAS se evalúe ubicación en iOS, una muestra DEBE ser candidata si `v = Δd/Δt > 138.89 m/s` con `Δt ≥ 1 s` y `Δd ≥ 100 m`.

✅ `Δd=200 m`, `Δt=1 s`, v=200 m/s → candidata.
❌ `Δd=50 m`, `Δt=1 s`, v=50 m/s → no candidata.

---

**RN-003-A-2** | Categoría: Anti-Fraude | Prioridad: MUST | EARS: State-driven

MIENTRAS se evalúe ubicación en iOS, una muestra DEBE ser candidata si `Δd > 1,000 m` en `Δt ≤ 5 s`.

✅ `Δd=1,200 m` en `Δt=4 s` → candidata.
❌ `Δd=800 m` en `Δt=4 s` → no candidata.

---

**RN-003-A-3** | Categoría: Anti-Fraude | Prioridad: MUST | EARS: State-driven

MIENTRAS se evalúe ubicación en iOS, una muestra DEBE ser candidata si la distancia entre BSSID (Wi-Fi conectado) y GPS es > 5 km cuando el BSSID es conocido. En zonas con cobertura de base BSSID < 30 %, el umbral DEBE ser 10 km. Si el BSSID es desconocido, la señal DEBE ignorarse.

✅ BSSID conocido, distancia 6 km → candidata.
❌ BSSID desconocido → señal ignorada.

---

**RN-003-A-4** | Categoría: Anti-Fraude | Prioridad: MUST | EARS: State-driven

MIENTRAS se evalúe ubicación en iOS, una muestra DEBE ser candidata si `horizontalAccuracy > 500 m`.

✅ `horizontalAccuracy = 600 m` → candidata.
❌ `horizontalAccuracy = 100 m` → no candidata.

---

**RN-003-A-5** | Categoría: Anti-Fraude | Prioridad: MUST | EARS: State-driven

MIENTRAS se evalúe ubicación en iOS, el sistema DEBE contabilizar cuántas de las 4 señales (RN-003-A-1…A-4) son testeables en cada muestra.

✅ 4 señales testeables → se aplica RN-003-A.
❌ Menos de 2 señales testeables → ver RN-003-A-6.

---

**RN-003-A-6** | Categoría: Anti-Fraude | Prioridad: MUST | EARS: Event-driven

SI menos de 2 de las 4 señales (RN-003-A-1…A-4) son testeables al momento de la marcación, la App DEBE asignar `REQUIRES_MANUAL_APPROVAL` y notificar al Administrador de Operaciones, sin permitir procesamiento automático.

✅ Zona rural sin BSSID y primera captura → derivación manual.
❌ Procesamiento automático con 0 señales testeables.

---

**RN-003-B** | Categoría: Anti-Fraude | Prioridad: MUST | EARS: Unwanted

SI la App detecta indicadores de jailbreak en iOS, DEBE emitir `event_type = JAILBREAK_DETECTED` y rechazar con HTTP 422, notificando al Administrador.

✅ iOS modificado → rechazo.
❌ iOS íntegro → permitir marcación.

---

**RN-003-C** | Categoría: Anti-Fraude | Prioridad: MUST | EARS: Unwanted

SI la App detecta indicadores de root en Android (binarios `su` en rutas estándar, tags de prueba, `debuggable` activo), DEBE emitir `event_type = ROOT_DETECTED` y rechazar con HTTP 422, notificando al Administrador.

✅ Android con root → rechazo.
❌ Android corporativo íntegro → permitir marcación.

---

**RN-004** | Categoría: Anti-Fraude | Prioridad: MUST | EARS: Ubiquitous

El sistema DEBE registrar la marca de tiempo de cada evento usando el `ntp_timestamp` oficial del servidor NTP en UTC como **fuente primaria**. Los fallbacks autorizados son: (a) RN-020 para captura offline; (b) RN-020-A para captura online con fallo NTP. En todos los casos la hora local del sistema operativo del dispositivo NUNCA se usa sin calibración.

✅ Hora local 07:00 AM; NTP retorna 07:34 AM UTC → `ntp_timestamp = 2026-11-10T07:34:00.000Z`.
❌ Usar `System.currentTimeMillis()` sin calibración NTP.

---

**RN-014** | Categoría: Anti-Fraude | Prioridad: SHOULD | EARS: Event-driven

MIENTRAS esté activada la directiva de bloqueo automático y se detecten 3 intentos fallidos consecutivos de `FAKE_LOCATION_DETECTED` o `LIVENESS_FAILED` desde un mismo dispositivo en 15 min, el sistema DEBE bloquear temporalmente la terminal por 30 minutos y emitir una alerta de seguridad.

✅ 3 fallos → bloqueo + auditoría.
❌ Intentos ilimitados sin bloqueo.

---

**RN-014-A** | Categoría: Anti-Fraude | Prioridad: SHOULD | EARS: State-driven

MIENTRAS el contador de RN-014 esté activo, DEBE reiniciarse con: (a) marcación `SUCCESS`; (b) 60 min sin intentos rechazados; (c) re-autenticación del supervisor con biometría de plataforma.

✅ Supervisor se re-autentica tras 2 fallos → contador vuelve a 0.
❌ Contador persiste indefinidamente.

---

### Offline & Sincronización

**RN-005** | Categoría: Offline & Sincronización | Prioridad: MUST | EARS: State-driven

MIENTRAS el celular no tenga red, la App DEBE cifrar localmente con cifrado respaldado por hardware, con límite 5,000 registros o 200 MB (el menor). Al restablecerse la red, el **proceso nativo en segundo plano** DEBE transmitir en lotes de 50 con backoff (2 s, 4 s, 8 s; máximo 3 reintentos). Tras 3 fallos, la marca queda `PENDING_SYNC` con reintento cada 5 min hasta 72 h (RN-019). Si la plataforma limita background, la App DEBE sincronizar al primer foreground.

✅ 3 marcaciones cifradas localmente → envío 10 s después al salir a superficie.
❌ Datos en texto plano o fallar sin red.

---

**RN-005-G** | Categoría: Offline & Sincronización | Prioridad: MUST | EARS: State-driven

MIENTRAS existan marcaciones `PENDING_SYNC`, la App DEBE mostrar banner con `<count> marcaciones pendientes de sincronizar` (`<count>` en dígitos arábigos), botón "Reintentar ahora", color ámbar si `1 ≤ N ≤ 100`, rojo si `N > 100` o `oldest_pending_age > 24 h`. El banner se actualiza en cambios de red, estado de App y finalización de sincronización.

✅ 3 pendientes → banner ámbar; al reconectar desaparece ≤ 5 s.
❌ No mostrar estado de sincronización.

---

**RN-005-H** | Categoría: Offline & Sincronización | Prioridad: MUST | EARS: Event-driven

CUANDO la App se reinstale y detecte pérdida de claves de cifrado locales, DEBE (a) emitir `event_type = APP_REINSTALL_DATA_LOSS`, (b) contabilizar las marcaciones locales perdidas e informarlas al Administrador, (c) permitir al supervisor **re-marcar** los eventos perdidos si la ventana temporal lo permite (≤ 72 h), (d) generar constancia de pérdida según RN-012-D si aplica. La App NO puede "re-sincronizar" datos cuya clave se perdió.

✅ Reinstalación con 10 marcaciones perdidas → 10 re-marcas + constancia.
❌ Intentar "re-sincronizar" datos indescifrables.

---

**RN-006** | Categoría: Offline & Sincronización | Prioridad: MUST | EARS: Event-driven

CUANDO el backend reciba marcaciones offline, DEBE validar idempotencia con `(worker_id, ntp_timestamp, device_uuid)` y descartar duplicados en ventana de 60 s.

✅ Reenvío 3 veces → se procesa la primera.
❌ 3 registros duplicados.

---

**RN-019** | Categoría: Expiración Offline | Prioridad: MUST | EARS: State-driven

MIENTRAS una marcación offline permanezca sin sincronizar, conservará validez automática por máx. **72 h contadas desde el `capture_timestamp`** (definido en glosario). Al sincronizar, el backend DEBE recalcular la antigüedad como `(ntp_timestamp_actual − capture_timestamp)` usando el `ntp_timestamp` del servidor en el momento de la sincronización. Si la antigüedad excede 72 h, DEBE rechazar la sincronización automática y exigir `REQUIRES_MANUAL_APPROVAL`.

✅ envío a las 48 h → sincronización automática; a las 75 h → `REQUIRES_MANUAL_APPROVAL`.
❌ Sincronizaciones automáticas sin control para registros de semanas.

---

**RN-020** | Categoría: Contingencia NTP | Prioridad: MUST | EARS: Event-driven

CUANDO la App capture una marcación offline y el NTP no responda en 2,000 ms, DEBE calcular la marca con reloj monotónico + `offset_calibrado`. Este método DEBE requerir `|offset_calibrado| ≤ 5 s` y `age(offset_calibrado) ≤ 12 h`; en caso contrario DEBE registrar `event_type = UNTRUSTWORTHY_TIME` y derivar a `REQUIRES_MANUAL_APPROVAL`.

✅ `offset_calibrado = -1,200 ms` vigente → se aplica.
❌ `offset_calibrado = 60 s` sin rechazo.

---

**RN-020-A** | Categoría: Contingencia NTP | Prioridad: MUST | EARS: Event-driven

CUANDO la App capture una marcación online y el backend no obtenga respuesta NTP en 2,000 ms tras 2 intentos a 2 servidores NTP distintos, el sistema DEBE: (a) usar el `offset_calibrado` del cliente si `|offset| ≤ 5 s` y vigencia ≤ 12 h; o (b) rechazar con HTTP 503, registrar `event_type = UNTRUSTWORTHY_TIME` y notificar al Administrador.

✅ 2 servidores NTP fallan → HTTP 503 + auditoría.
❌ Aceptar la hora del backend sin validación NTP.

---

**RN-021** | Categoría: Offline & Saturación | Prioridad: MUST | EARS: State-driven

MIENTRAS el almacenamiento local alcance 5,000 registros o 200 MB (el menor), la App DEBE bloquear nuevas capturas y mostrar alerta requiriendo traslado a zona con cobertura.

✅ Marcación 5,001 sin señal → bloqueo + mensaje.
❌ Sobrescribir registros antiguos.

---

**RN-021-A** | Categoría: Offline & Saturación | Prioridad: MUST | EARS: State-driven

MIENTRAS el almacenamiento local cifrado supere el 80 % de su capacidad (4,000 de 5,000 registros, o 160 de 200 MB, el menor), la App DEBE mostrar un banner rojo parpadeante con el conteo exacto (`registros_pendientes` y `uso_disco_mb`) y forzar reintento de sincronización si hay red disponible. El banner DEBE actualizarse en cada cambio de red, cambio de estado de App y finalización de sincronización. La alerta se mantiene hasta que `registros_pendientes < 4,000` AND `uso_disco_mb < 160`, o hasta el bloqueo total por RN-021.

✅ 4,050 registros pendientes y 162 MB usados → banner rojo parpadeante "⚠️ Almacenamiento local al 81 % (4,050/5,000). Conéctese a red para sincronizar" + reintento de sync si hay red.
❌ Mantener banner ámbar (`N=1…100`) cuando `registros_pendientes = 4,050` (> 80 % de capacidad).

---

### Visualización Mapa GPS

**RN-007** | Categoría: Visualización Mapa GPS | Prioridad: MUST | EARS: Ubiquitous

El Dashboard DEBE permitir acceso mediante token de sesión firmado con vigencia máxima 12 h, sin formulario usuario/contraseña, y renderizar mapa interactivo con geocercas y pines.

✅ Token de entorno → acceso directo.
❌ Formulario clásico usuario/contraseña.

---

**RN-008** | Categoría: Visualización Mapa GPS | Prioridad: MUST | EARS: Event-driven

CUANDO el Administrador seleccione una marcación, DEBE renderizar círculo de radio fijo 50 m centrado en la sede y pin Verde (`INSIDE_GEOFENCE`) o Rojo (`OUTSIDE_GEOFENCE`).

✅ Pin verde a 22 m del centroide + círculo 50 m.
❌ Solo coordenadas en tabla sin geocerca.

---

**RN-013** | Categoría: Visualización Mapa GPS | Prioridad: SHOULD | EARS: Event-driven

CUANDO el usuario filtre por rango de fechas, DEBE retornar marcaciones en p95 ≤ 1.5 s con hasta 10,000 marcaciones y 50 usuarios concurrentes.

✅ 1,200 pines en 850 ms.
❌ Consulta > 5 s por falta de índices.

---

### Seguridad

**RN-009** | Categoría: Seguridad | Prioridad: MUST | EARS: Ubiquitous

Toda información sensible DEBE almacenarse cifrada en reposo con AES-256 y transmitirse con TLS 1.3 obligatorio (TLS 1.2 no aceptado). En el cliente móvil, los datos locales DEBEN cifrarse con mecanismos respaldados por hardware de la plataforma.

✅ Vectores AES-256 + TLS 1.3.
❌ GPS/DNI en texto plano o TLS 1.2.

---

### Integración ZKTeco

**RN-011** | Categoría: Integración ZKTeco | Prioridad: SHOULD | EARS: Event-driven

CUANDO un reloj ZKTeco emita marcación, el backend DEBE confirmar en **p95 ≤ 100 ms y p99 ≤ 250 ms**, y derivar a procesamiento asíncrono.

✅ Confirmación 45 ms + derivación.
❌ Procesamiento síncrono bloqueante.

---

**RN-011-A** | Categoría: Integración ZKTeco | Prioridad: SHOULD | EARS: State-driven

MIENTRAS un `serial_number` exceda 10 RPS, el sistema DEBE responder HTTP 429 con `Retry-After: 5`.

✅ 500 marcas en 20 s → 10 RPS aceptados, resto 429.
❌ Rechazo sin `Retry-After`.

---

**RN-011-B** | Categoría: Integración ZKTeco | Prioridad: SHOULD | EARS: Event-driven

SI el payload excede 10 MB o 5,000 registros, DEBE responder HTTP 413 con `max_records_per_request = 5000`.

✅ 12,000 registros → 413 + chunking.
❌ Aceptar 50 MB y agotar recursos.

---

**RN-018** | Categoría: Contingencia Hardware | Prioridad: MUST | EARS: Event-driven

CUANDO un equipo ZKTeco deje de emitir latido (heartbeat) por más de 10 minutos al servidor ADMS, el sistema DEBE cambiar su estado a `BIOMETRICO_FUERA_DE_LINEA` y habilitar automáticamente la marcación facial móvil en la PWA como contingencia para esa sede.

✅ 10 min de caída → contingencia habilitada.
❌ Caída no habilitar contingente.

**RN-022** | Categoría: Contingencia Hardware | Prioridad: MUST | EARS: Event-driven
CUANDO un equipo ZKTeco en estado `BIOMETRICO_FUERA_DE_LINEA` vuelva a emitir 3 señales consecutivas de latido (heartbeat) al servidor ADMS, el sistema DEBE actualizar el estado de la unidad a `BIOMETRICO_EN_LINEA` y notificar al Administrador de Operaciones el restablecimiento del canal primario.

---

### Seguridad de Dispositivo

**RN-012** | Categoría: Seguridad Dispositivo | Prioridad: MUST | EARS: State-driven

MIENTRAS un Supervisor ejecute la App, el sistema DEBE validar que el `device_uuid` coincida con el registro autorizado antes de permitir la marcación. Este control DEBE considerarse **secundario**; la **autenticación primaria del supervisor (M-10) DEBE completarse antes de cualquier marcación**.

✅ `device_uuid` autorizado + supervisor autenticado → permitir.
❌ Marcación sin autenticación del supervisor (aunque `device_uuid` sea válido).

---

**RN-012-A** | Categoría: Seguridad Dispositivo | Prioridad: MUST | EARS: State-driven

Cada supervisor DEBE tener 1 `primary_device_uuid` y, opcionalmente, 1 `backup_device_uuid`. La activación del backup DEBE requerir OTP al correo (entrega ≤ 2 min) y biometría del supervisor, completándose en ≤ 5 min.

✅ Primario destruido → backup activado ≤ 5 min con OTP + biometría.
❌ Activación sin OTP o sin biometría.

---

**RN-012-B** | Categoría: Seguridad Dispositivo | Prioridad: MUST | EARS: Event-driven

CUANDO un `device_uuid` no enrolado o revocado intente marcar, la App DEBE mostrar "Dispositivo no autorizado" y bloquear, registrando `event_type = DEVICE_UNAUTHORIZED`. Sin red, la marcación se retiene como `PENDING_DEVICE_ACTIVATION` (≤ 72 h; luego `REQUIRES_MANUAL_APPROVAL`).

✅ UUID no enrolado con red → HTTP 403 + auditoría.
❌ Permitir UUID desconocido.

---

**RN-012-C** | Categoría: Seguridad Dispositivo | Prioridad: MUST | EARS: State-driven

Los datos cifrados en un `device_uuid` revocado o destruido NO son recuperables (crypto-shred). La pérdida DEBE registrarse con `event_type = DATA_LOST_DEVICE_LOST` y notificarse al Administrador.

✅ Dispositivo destruido sin backup → evento notificado.
❌ Usar claves maestras del servidor para recuperar.

---

**RN-012-D** | Categoría: Seguridad Dispositivo | Prioridad: MUST | EARS: Event-driven

CUANDO ocurra `DATA_LOST_DEVICE_LOST`, el Administrador DEBE generar constancia de pérdida por cada trabajador afectado en ≤ 48 h y registrarla con `event_type = ATTENDANCE_LOSS_CONSTANCY`. El trabajador puede invocarla para impugnación de tareo.

✅ Pérdida de 50 marcaciones → 50 constancias ≤ 48 h.
❌ Pérdida sin constancia.

---

### Datos Maestros / Testing

**RN-015** | Categoría: Datos Maestros / Pruebas | Prioridad: MUST | EARS: Ubiquitous

Toda suite de pruebas automatizadas DEBE usar exclusivamente datos sintéticos, quedando estrictamente prohibido el uso de datos, fotografías o coordenadas reales de empleados en el entorno de pruebas y QA.

✅ Coordenadas sintéticas `-12.000000, -77.000000`.
❌ Fotos reales y DNIs.

---

### Auditoría

**RN-016** | Categoría: Auditoría | Prioridad: MUST | EARS: Event-driven

CUANDO ocurra cualquier evento de marcación, DEBE insertarse un registro inmutable con: `event_id`, `worker_id`, `event_type`, `gps_coordinates`, `geofence_status`, `ip_address`, `device_info` (objeto con `device_uuid`, `os_version`, `app_version`, `model`) y `utc_timestamp`.

✅ `event_type = OUTSIDE_GEOFENCE` con `device_info` completo.
❌ Descartar rechazos sin rastro.

---

### Cumplimiento Legal (Ley N.° 29733)

**RN-030** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

CUANDO se enrolle a un trabajador, la App DEBE capturar consentimiento con `privacy_policy_version`, `consent_timestamp`, firma digital u OTP, y `consent_hash`. Sin consentimiento, el enrolamiento DEBE rechazarse con HTTP 403.

✅ Consentimiento firmado el 10/11/2026 con `consent_hash`.
❌ Enrolar sin consentimiento.

---

**RN-030-A** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

CUANDO cambie `privacy_policy_version`, el 100 % de trabajadores activos DEBE re-consentir en ≤ 30 días naturales. Los no consentidos quedan `PENDING_CONSENT` y solo pueden marcarse por ZKTeco (o por RN-030-D si no hay ZKTeco).

✅ Política v1 → v2 el 01/12/2026 → 100 % hasta 31/12/2026.
❌ Mantener consentimiento v1 indefinidamente.

---

**RN-030-B** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

CUANDO el módulo se despliegue sobre una base existente, el 100 % de trabajadores activos DEBE capturar consentimiento en ≤ 60 días naturales. Los no consentidos quedan `PENDING_CONSENT` y solo pueden marcarse por ZKTeco (o por RN-030-D si no hay ZKTeco).

✅ Despliegue 01/12/2026 → consentimiento hasta 30/01/2027.
❌ Operar sin consentimiento para heredados.

---

**RN-030-C** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: State-driven

MIENTRAS RN-030-A y RN-030-B se activen simultáneamente, DEBE aplicarse el plazo más restrictivo (30 días de RN-030-A).

✅ Despliegue con política v2 nueva → plazo 30 días.
❌ Aplicar 60 días cuando coexisten ambos eventos.

---

**RN-030-D** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

CUANDO un trabajador `PENDING_CONSENT` opere en una sede sin ZKTeco, el Supervisor DEBE registrar la marcación en modo manual con `event_type = MANUAL_CANCELLATION` y justificación obligatoria; el Administrador DEBE aprobar en ≤ 24 h. El trabajador queda bloqueado de la marcación biométrica hasta que consienta.

✅ Trabajador sin consentimiento en sede sin ZKTeco → marcación manual aprobada ≤ 24 h.
❌ Bloquear la marcación dejando al trabajador sin alternativa.

---

**RN-031** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

CUANDO un titular ejerza Acceso, el sistema DEBE (a) verificar identidad según RN-037, (b) responder en ≤ 10 días hábiles con la totalidad de sus datos en formato estructurado.

✅ Solicitud verificada 01/11/2026 → respuesta 10/11/2026.
❌ Responder sin verificación de identidad o fuera de plazo.

---

**RN-032** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

CUANDO un titular ejerza Rectificación, el sistema DEBE (a) verificar identidad según RN-037, (b) corregir en ≤ 5 días hábiles y registrar `event_type = ARCO_RECTIFICATION`.

✅ Corrección 02/11/2026 → aplicada 05/11/2026 con auditoría.
❌ Corregir sin verificación o sin trazabilidad.

---

**RN-033** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

CUANDO un titular ejerza Cancelación, el sistema DEBE (a) verificar identidad según RN-037, y aplicar los pasos RN-033-A…RN-033-E.

✅ Cancelación verificada 01/11/2026 → flujo RN-033-A…E aplicado.
❌ Cancelar sin verificación de identidad.

---

**RN-033-A** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

A las 24 h de RN-033, DEBE marcarse la biometría como `deleted_at` e inaccesible a toda operación.

✅ Cancelación 01/11/2026 10:00 → `deleted_at` 02/11/2026 10:00.
❌ Biometría accesible después de 24 h.

---

**RN-033-B** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

A las 24 h de RN-033, DEBE revocarse **inmediatamente** la clave AES-256 correspondiente en el servicio de gestión de claves. La revocación es atómica, NO progresiva.

✅ Clave revocada el 02/11/2026 10:00.
❌ Revocación escalonada entre 24 h y 30 días.

---

**RN-033-C** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

A los 30 días de RN-033, DEBEN eliminarse físicamente los datos cifrados.

✅ Datos eliminados 01/12/2026.
❌ Datos cifrados persistidos a los 30 días.

---

**RN-033-D** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Ubiquitous

El log de auditoría DEBE preservarse con `worker_hash = hash_cryptográfico(worker_id + per_tenant_salt)` durante 5 años.

✅ Eventos históricos con `worker_hash` por 5 años.
❌ Eliminar log de auditoría antes de 5 años.

---

**RN-033-E** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Ubiquitous

La identidad legal mínima (`worker_legal_identity`) DEBE conservarse 5 años por obligación legal.

✅ Identidad conservada hasta 01/11/2031.
❌ Eliminar identidad legal junto con la biometría.

---

**RN-034** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

CUANDO un titular ejerza Oposición, el sistema DEBE (a) verificar identidad según RN-037, (b) suspender tratamiento biométrico y derivar al flujo alterno sin penalización.

✅ Oposición verificada → ZKTeco asignado sin impacto.
❌ Penalizar al trabajador por oposición.

---

**RN-035** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: State-driven

Los registros de asistencia y el log de auditoría DEBEN conservarse 5 años. Los vectores y fotos de referencia DEBEN conservarse mientras dure la relación laboral + 30 días. Al vencer, DEBE ejecutarse purga criptográfica con `event_type = DATA_PURGED`.

✅ Cesado 01/10/2026 → biometría purgada 01/11/2026; asistencia hasta 01/10/2031.
❌ Biometría activa 5 años después del cese.

---

**RN-035-A** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Ubiquitous

Los eventos del log de auditoría DEBEN heredar el `tenant_id` del momento del evento y permanecer asociados al tenant original.

✅ Sede reasignada de A a B → eventos históricos con `tenant_id = A`.
❌ Migrar eventos al nuevo tenant sin autorización.

---

**RN-035-B** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

CUANDO ocurra una fusión de tenants, los eventos DEBEN preservar `tenant_id` original y añadir `merged_from_tenant_id`. CUANDO ocurra eliminación, los eventos DEBEN retenerse 5 años en almacenamiento cifrado con clave del tenant eliminado.

✅ Fusión A+B → eventos de A con `merged_from_tenant_id = A`.
❌ Eliminar eventos antes de 5 años.

---

**RN-036** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

CUANDO un trabajador revoque su consentimiento biométrico sin cesar la relación laboral, el sistema DEBE aplicar RN-036-A…RN-036-D.

✅ Trabajador revoca → flujo RN-036-A…D aplicado.
❌ Forzar al trabajador a mantener biometría.

---

**RN-036-A** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

CUANDO ocurra RN-036, DEBE suspenderse el uso de biometría del trabajador.

✅ Biometría suspendida inmediatamente.
❌ Mantener biometría activa tras revocación.

---

**RN-036-B** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

CUANDO ocurra RN-036, el trabajador DEBE ser derivado a ZKTeco o a marcación por supervisor, sin penalización laboral.

✅ Derivado a ZKTeco sin afectar tareo.
❌ Bloquear marcación o penalizar.

---

**RN-036-C** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

CUANDO ocurra RN-036, DEBE registrarse `event_type = CONSENT_REVOKED`.

✅ Evento registrado en log de auditoría.
❌ Revocación sin trazabilidad.

---

**RN-036-D** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Ubiquitous

El log de auditoría del trabajador con consentimiento revocado DEBE conservarse 5 años con `worker_id` real si la biometría se mantiene en el sistema; o con `worker_hash` si la biometría se elimina (RN-033).

✅ Trabajador activo revoca → `worker_id` real 5 años.
❌ Aplicar `worker_hash` a trabajador activo con biometría vigente.

---

**RN-037** | Categoría: Cumplimiento Legal | Prioridad: MUST | EARS: Event-driven

CUANDO se reciba una solicitud ARCO (RN-031…RN-034), el sistema DEBE verificar la identidad del titular mediante: (a) firma digital del titular (Ley N.° 27269) y/o (b) autenticación con DNI + código OTP al correo registrado + verificación biométrica contra el vector de referencia. SIN verificación, la solicitud DEBE rechazarse con HTTP 401 y registrarse como `event_type = ARCO_IDENTITY_UNVERIFIED`.

✅ Solicitud con firma digital válida → verificada y en proceso.
❌ Atender solicitud sin verificación de identidad.

---

## Matriz de Trazabilidad (RN ↔ M ↔ PRINCIPIO)

| Regla | M | PRINCIPIO | Test E2E asociado |
|---|---|---|---|
| RN-001A/B/C/D | M-01 | P-005 | `biometric_flow.e2e` |
| RN-002/A/B/C | M-02 | P-005 | `geofence_retry.e2e` |
| RN-003 | M-03 | P-005 | `fake_location_android.e2e` |
| RN-003-A/A-1…A-6 | M-03 | P-005 | `fake_location_ios.e2e` |
| RN-003-B/C | M-03 | P-005 | `jailbreak_root_detection.e2e` |
| RN-004 | M-03 | P-005 | `ntp_primary.e2e` |
| RN-005/005-G/005-H | M-04 | P-006 | `offline_sync.e2e` |
| RN-006/019/020/020-A | M-04 | P-006 | `offline_expiration.e2e` |
| RN-021 | M-04 | P-006 | `offline_saturation.e2e` |
| RN-021-A | M-04 | P-006 | `offline_saturation_80pct.e2e` |
| RN-007/008/013 | M-05 | P-002 | `dashboard_map.e2e` |
| RN-009 | M-05 | P-003 | `encryption_at_rest.e2e` |
| RN-010 | M-06 | — | `site_creation.e2e` |
| RN-011/A/B/018 | M-07 | P-004 | `zkteco_load.e2e` |
| RN-012/A/B/C/D | M-08 + M-10 | P-005 | `device_binding.e2e` |
| RN-014/014-A | S-05 | P-005 | `temporary_blocking.e2e` |
| RN-015 | — | P-008 | `synthetic_data.spec` |
| RN-016 | M-04 | P-009 | `immutable_audit.e2e` |
| RN-017/017-A | M-02 | P-005 | `haversine_epsilon.e2e` |
| RN-030/030-A/B/C/D | M-09 | P-010 | `consent_lifecycle.e2e` |
| RN-031/032/034/037 | M-09 | P-010 | `arco_operations.e2e` |
| RN-033/A/B/C/D/E | M-09 | P-010 | `arco_cancellation_crypto_shred.e2e` |
| RN-035/A/B | M-09 | P-010 | `retention_policy.e2e` |
| RN-036/A/B/C/D | M-09 | P-010 | `consent_revocation.e2e` |