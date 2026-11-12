# Project Brief: Módulo Operativo y Control de Asistencia Omnicanal

**Proyecto:** Módulo Operativo y Control de Asistencia Omnicanal (Ecosistema RH / ERP)
**Ámbito:** Empresas de servicios corporativos, seguridad física, outsourcing y tareos en campo en Perú.
**Versión:** 2.2.1
**Fecha:** 10 de Noviembre de 2026
**Estado:** Aprobado para Diseño de Software
**Cambios v2.2.1:** Aplicados 18 hallazgos QA v2.2.0. S-04 elevada a M-10 (autenticación del supervisor). M-10 anterior degradada a S-05. Añadida verificación de identidad ARCO (RN-037). Fallback sin ZKTeco (RN-030-D).
**Cambios v2.2.0:** Arquitectura removida.
**Cambios v2.0.0:** App Móvil Nativa/Híbrida; identificadores en inglés.

---

## 1. Problema y Oportunidad Operativa

### Problema Real
* **Alteración y Manipulación Manual:** cuadernos y partes físicos son falsificables (firmas, marcaciones "de favor", suplantación).
* **Demora en Tareo:** consolidación manual toma días al cierre de mes y retrasa planilla.
* **Vulnerabilidad en Garitas o Unidades Remotas:** sedes itinerantes dependían de llamadas o cuadernos.

### Oportunidad
**Control de Asistencia Omnicanal (Biométricos ZKTeco + Reconocimiento Facial Móvil)** que centralice marcaciones en tiempo real:
* Sedes fijas → hardware ZKTeco.
* Campo → App Móvil en celular corporativo del Supervisor con liveness, geocerca GPS, detección nativa de ubicación simulada e identificador de hardware.
* Dashboard GPS sin login convencional.

---

## 2. Usuarios y Segmentos Objetivo

1. **Supervisor de Ronda:** marca por los operarios; necesita modo offline, autenticación biométrica de plataforma y rapidez.
2. **Administrador de Operaciones:** monitorea mapa GPS con geocercas de 50 m, filtros por fecha y trabajador.
3. **Agente de Campo / Vigilante / Operario:** sujeto de marcación; requiere transparencia y protección de derechos (constancia de pérdida, revocación sin penalización).
4. **Analista de Planillas / RRHH:** cierra tareo y calcula sobretiempos.
5. **DPO / Oficial de Privacidad:** gobierna consentimiento, ARCO (con verificación de identidad), retención y crypto-shred.

---

## 3. Propuesta de Valor

* **Centralización Omnicanal Inviolable.**
* **Cero Carga Tecnológica para el Trabajador de Campo.**
* **Visibilidad Geográfica sin Fricción.**
* **Tolerancia y Resiliencia** (conmutación a App Móvil ante caída de ZKTeco).
* **Anti-Fraude con Capacidades Nativas Reales:** validación del proveedor nativo (Android); heurísticas con consenso y mínimo de señales testeables (iOS, RN-003-A-6); detección de jailbreak/root; identificador de hardware; sincronización en segundo plano.
* **Autenticación Primaria del Supervisor (M-10):** biometría de plataforma + PIN antes de cualquier marcación. Sin esta autenticación, el `device_uuid` por sí solo no autoriza la marcación.
* **Cumplimiento Legal Reforzado (Ley 29733):** consentimiento versionado; ARCO con **verificación de identidad (RN-037)**; retención diferenciada; crypto-shred con **revocación inmediata de clave**; fallback operativo para trabajadores sin ZKTeco (RN-030-D).

---

## 4. Modelo de Impacto / Retorno de Inversión (ROI)

* **Eliminación del 100 % del tiempo de digitación de cuadernos.**
* **Reducción del sobretiempo ficticio o suplantado.**
* **Continuidad de servicio.**
* **Reducción de riesgo legal** (multas hasta 100 UIT).

---

## 5. Métricas de Éxito (KPIs Medibles)

| KPI | Baseline | Meta |
|---|---|---|
| Consolidación de Tareo Mensual | 3-5 días | **< 15 min** |
| Marcaciones Fuera de Geocerca (> 50 m) | ~12 % | **< 1 %** |
| Registros manuales por pérdida de red o fallas ZKTeco | Papel | **100 % digitalizadas** |
| Efectividad Anti-Fraude | Nula | **≥ 99 % bloqueados y auditados**; FRR ≤ 1 % |
| Cumplimiento ARCO (con verificación de identidad) | No aplica | **100 % dentro del SLA legal** |
| Disponibilidad del endpoint receptor ZKTeco | No aplica | **p95 ≤ 100 ms, p99 ≤ 250 ms a 500 RPS** |

---

## 6. Riesgos Identificados, Supuestos y Estrategia Anti-Fraude

### Riesgos y Mitigación
1. **Robo/Pérdida del Celular Corporativo:** vinculación primaria + backup (RN-012-A); revocación por gestión de dispositivos; crypto-shred local (RN-012-C).
2. **Sin Cobertura en Sótanos:** Offline-First con sincronización en segundo plano nativa y banner de pendientes (RN-005-G).
3. **Fraude por Foto o Ubicación Falsa:** proveedor nativo (Android); heurísticas con consenso (iOS, RN-003-A); **mínimo 2 señales testeables (RN-003-A-6)**; detección root/jailbreak; NTP; liveness.
4. **Falsos Positivos de Anti-Fraude:** consenso de 3 muestras en ventana acotada + auditoría con muestras crudas.
5. **Incumplimiento de Ley 29733:** consentimiento versionado (RN-030, 030-A, 030-B); ARCO con verificación de identidad (RN-037); retención y crypto-shred progresivo (RN-033); revocación sin cese (RN-036).
6. **Trabajadores sin ZKTeco y sin consentimiento:** fallback manual con aprobación administrativa (RN-030-D).
7. **Pérdida de marcaciones por reinstalación de App:** auditoría y re-marcación manual (RN-005-H).

### Supuestos Clave
* Supervisor con smartphone Android 8.0+ / iOS 14+.
* Sedes con coordenadas WGS84 registradas.
* Parque máximo simultáneo de 500 relojes ZKTeco (revisable anualmente).
* DPO designado y banco de datos registrado ante ANPD.

---

## 7. Restricciones Conocidas y Cumplimiento

1. **Ley N.° 29733:** AES-256 en reposo; TLS 1.3 en tránsito (TLS 1.2 no aceptado); almacenamiento local móvil cifrado; consentimiento/ARCO/retención (PRINCIPIO-010); **verificación de identidad ARCO (RN-037)**.
2. **Idioma:** código en inglés; docs/UI/tests en Español Neutro (PRINCIPIO-001).
3. **Datos Sintéticos:** prohibido usar fotos o coordenadas reales en pruebas.
4. **Los 10 principios de `00-constitucion.md` son no negociables.**