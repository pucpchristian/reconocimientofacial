# Glosario del Dominio Técnico y Operativo

**Proyecto:** Módulo Operativo y Control de Asistencia Omnicanal (Ecosistema RH / ERP)
**Ámbito:** Definiciones del dominio funcional, normativo y técnico para Perú.
**Versión:** 2.2.1
**Fecha:** 10 de Noviembre de 2026
**Cambios v2.2.1:** Aplicados hallazgos QA v2.2.0: término 10 reformulado (sin "arquitectura"); "Cola Asíncrona" fusionado con BullMQ; añadidos PENDING_CONSENT, ARCO_IDENTITY_UNVERIFIED, capture_timestamp; PRINCIPIO-009 alinea vocabulario.
**Cambios v2.2.0:** Arquitectura removida.

---

## Términos del Dominio

1. **Tareo:** proceso operativo y administrativo mediante el cual se registra, consolida y verifica la consolidación de horas en la asistencia diaria, horas trabajadas, tardanzas, faltas, sobretiempos y cubrimientos de turnos del personal operario desplegado en campo.

2. **Geocerca (Geofencing):** perímetro geográfico virtual definido alrededor de una ubicación física real (coordenadas centrales de la sede). En el módulo, radio de tolerancia de exactamente 50 metros.

3. **Prueba de Vida (Liveness Detection):** técnica de visión por computadora para determinar si el rostro presentado corresponde a un ser humano real presente en el momento y no a una reproducción estática. Se ejecuta en la App Móvil y exige gesto aleatorio con `liveness_score >= 0.95` (RN-001A). La profundidad 3D es complementaria, no sustituto.

4. **Ubicación Simulada (Mock Location / Fake GPS):** herramienta o función del SO móvil que permite alterar las coordenadas GPS. Detección vía validación del proveedor nativo (Android) y heurísticas con consenso de 3 muestras (iOS, RN-003-A, con mínimo de 2 señales testeables, RN-003-A-6).

5. **Marca de Tiempo UTC (UTC Timestamp):** marca oficial e inmutable provista por servidor NTP, en ISO 8601 con milisegundos, siempre en UTC. En código: `ntp_timestamp` o `utc_timestamp`. Es la **fuente primaria** de tiempo (RN-004); los fallbacks autorizados son RN-020 (offline) y RN-020-A (online con fallo NTP).

6. **Marca de Tiempo de Captura (Capture Timestamp):** marca de tiempo del evento en el dispositivo, definida como (a) el `ntp_timestamp` real si fue posible; o (b) en su defecto, la derivación por reloj monotónico + `offset_calibrado` (RN-020). Es la referencia para calcular la validez de sincronización de 72 h (RN-019).

7. **Vinculación de Dispositivo (Device Binding):** mecanismo de seguridad que vincula el identificador de hardware (`device_uuid`) de un celular corporativo autorizado con la cuenta del Supervisor. Soporta vinculación primaria + backup (RN-012-A). Es un control **secundario**; la autenticación primaria es biometría de plataforma + PIN (M-10).

8. **Offline-First:** patrón que permite a la App Móvil operar sin conexión, cifrando datos localmente y sincronizando al detectar red, salvo eventos con antigüedad > 72 h que requieren aprobación manual (RN-019).

9. **ADMS (Automatic Data Master Server):** protocolo propietario del hardware ZKTeco para envío de marcaciones al backend. Capacidad exigida: p95 ≤ 100 ms, p99 ≤ 250 ms a 500 RPS (RNF-003).

10. **Ley N.° 29733:** Ley de Protección de Datos Personales del Perú. Establece el principio de seguridad (art. 9) y su Reglamento (D.S. 003-2013-JUS) exige medidas técnicas y organizativas, consentimiento, derechos ARCO y finalidad determinada. **Cifrado reforzado del proyecto** (decisión de negocio, no exigencia literal de la ley): AES-256 en reposo, cifrado local respaldado por hardware y **TLS 1.3 obligatorio en tránsito (TLS 1.2 no aceptado)**.

11. **Supervisor de Ronda:** usuario operativo itinerante responsable de inspeccionar puestos en sedes del cliente y registrar la asistencia del personal de campo mediante la App Móvil.

12. **Agente de Retén:** personal operario de reserva disponible para cubrir de inmediato un puesto ante inasistencia o tardanza del titular.

13. **Idempotencia:** propiedad por la cual el procesamiento repetido de la misma marcación (terna `worker_id` + `ntp_timestamp` + `device_uuid`) en ventana de 60 s produce el mismo resultado sin duplicar registros.


14. **Almacenamiento Local Cifrado:** persistencia cifrada provista por la plataforma móvil, respaldada por hardware de seguridad del dispositivo.

15. **Cola Asíncrona (Asynchronous Queue):** mecanismo de procesamiento desacoplado que permite recibir marcaciones rápidamente y procesarlas sin bloquear el endpoint receptor. Se implementa mediante un gestor de colas sobre un almacén de datos en memoria (típicamente Redis/BullMQ); los detalles de implementación se definen en el documento de arquitectura.

16. **Datos Sintéticos (Mocks):** datos ficticios generados algorítmicamente para pruebas unitarias y E2E; prohíben datos reales.

17. **Haversine (Fórmula):** ecuación trigonométrica esférica para calcular distancia en metros sobre esfera de radio medio `R = 6,371,000 m` entre pares de coordenadas en WGS84 (EPSG:4326). Aproximación esférica, no elipsoidal; error < 0.25 % a 50 m (±12.5 cm).

18. **App Móvil Nativa/Híbrida:** aplicación instalable en Android/iOS con acceso a APIs nativas de la plataforma: cámara, GPS, validación de proveedor de ubicación, identificador de hardware, almacenamiento seguro respaldado por hardware y ejecución en segundo plano.

19. **Proveedor Nativo de Ubicación (Native Location Provider):** API del SO Android que permite a la App determinar si la ubicación proviene del GPS real o de una fuente simulada. En iOS no existe API equivalente; se sustituye por heurísticas con consenso (RN-003-A).

20. **Proceso Nativo en Segundo Plano (Native Background Process):** servicio del SO móvil que ejecuta tareas sin interacción del usuario (sincronización, reintentos, banner de pendientes). **No confundir con "trabajador"** (empleado) del dominio laboral.

21. **ARCO (Acceso, Rectificación, Cancelación, Oposición):** derechos del titular reconocidos por la Ley N.° 29733. Toda solicitud DEBE verificar identidad del titular (RN-037) antes de procesarse.

22. **Consentimiento Informado:** manifestación de voluntad libre, previa, expresa e informada (Ley N.° 29733 art. 5). Versionado (`privacy_policy_version`), firmado u OTP, con `consent_hash = hash_cryptográfico(policy_version + worker_id + consent_timestamp)` (RN-030).

23. **Crypto-Shredding:** técnica de borrado seguro que destruye la clave criptográfica en lugar de los datos cifrados, dejándolos inaccesibles. Aplica al cese laboral y a la cancelación ARCO (RN-033, RN-035, PRINCIPIO-010). La revocación de la clave es **inmediata** (RN-033-B); la eliminación física de los datos se produce a los 30 días (RN-033-C).

24. **Worker Hash:** `worker_hash = hash_cryptográfico(worker_id + per_tenant_salt)` conservado en el log de auditoría tras cancelación ARCO de biometría, permitiendo retención legal (5 años) sin exponer datos biométricos.

25. **Jailbreak:** modificación no autorizada de iOS que elimina restricciones del sandbox. Detección dispara `JAILBREAK_DETECTED` (RN-003-B).

26. **Root (Android):** modificación no autorizada de Android que otorga privilegios elevados. Detección dispara `ROOT_DETECTED` (RN-003-C).

27. **PENDING_SYNC:** estado de una marcación cifrada localmente sin sincronizar. Reintento cada 5 min hasta 72 h (RN-005, RN-019). Visible vía banner (RN-005-G).

28. **REQUIRES_MANUAL_APPROVAL:** estado de una marcación que no pudo resolverse automáticamente (fallo persistente de precisión GPS tras 3 re-capturas, antigüedad > 72 h, fallo de NTP online, o < 2 señales testeables en iOS) y requiere liberación del Administrador (RN-002-B, RN-019, RN-020-A, RN-003-A-6).

29. **PENDING_DEVICE_ACTIVATION:** estado de una marcación en un `device_uuid` no enrolado, retenida hasta 72 h para validación administrativa (RN-012-B).

30. **BIOMETRIC_INCONSISTENCY:** evento emitido cuando `|client_match_score − server_match_score| > 0.05` (RN-001C), indicando posible ataque de sustitución o degradación.

31. **JAILBREAK_DETECTED:** evento emitido al detectar jailbreak en iOS (RN-003-B). Bloquea con HTTP 422.

32. **ROOT_DETECTED:** evento emitido al detectar root en Android (RN-003-C). Bloquea con HTTP 422.

33. **DATA_PURGED:** evento de auditoría al ejecutar crypto-shred de biometría por vencimiento de retención o cancelación ARCO (RN-035).

34. **DATA_LOST_DEVICE_LOST:** evento emitido cuando un `device_uuid` revocado o destruido no puede recuperar marcaciones pendientes (RN-012-C).

35. **ATTENDANCE_LOSS_CONSTANCY:** constancia generada por el Administrador en ≤ 48 h tras `DATA_LOST_DEVICE_LOST` para proteger derechos laborales (RN-012-D).

36. **CONSENT_REVOKED:** evento emitido cuando un trabajador revoca su consentimiento biométrico sin cesar la relación laboral (RN-036-C).

37. **APP_REINSTALL_DATA_LOSS:** evento emitido cuando la App Móvil se reinstala y las claves de cifrado previas se pierden (RN-005-H).

38. **Server-Side Reverification:** feature flag (`server_side_reverification = true`) que activa re-verificación biométrica en backend con transmisión cifrada de fotogramas crudos (TLS 1.3 + AES-256) y purga ≤ 24 h (RN-001C).

39. **PENDING_CONSENT:** **estado** de un trabajador activo cuya versión de consentimiento no está vigente. Solo permite marcación por ZKTeco (o por RN-030-D si no hay ZKTeco) hasta que consienta.

40. **ARCO_IDENTITY_UNVERIFIED:** evento emitido cuando una solicitud ARCO no supera la verificación de identidad del titular (RN-037). Bloquea el proceso con HTTP 401.