# 💀 Post Mortem de Incidencia: Procesamiento duplicado de notificaciones y parámetros con valores vacíos

## 1. Información General del Evento

| Campo | Valor |
| :--- | :--- |
| **Título de la Incidencia** | Envío masivo de notificaciones por parámetro vacío y procesamiento concurrente duplicado |
| **ID de Incidencia/Ticket** | N/A |
| **Fecha y Hora de Detección** | 09/02/2026 21:45 |
| **Fecha y Hora de Inicio del Impacto** | 09/02/2026 18:30 |
| **Fecha y Hora de Resolución** | 11/02/2026 09:10 (En monitoreo) |
| **Duración del Impacto** | ~39 horas |
| **Servicio(s) / Sistema(s) Afectado(s)** | Control de asistencia - Módulo de notificaciones de solicitudes (endpoint `/api/v2/notifications`) |
| **Impacto para el Usuario/Negocio** | 1) Notificaciones masivas enviadas a TODOS los usuarios debido a parámetro `idUsuarioSeg` vacío en conProcesos NCTI. 2) Procesamiento duplicado: mismo `idProcesoEspecialSolicitud` ejecutado múltiples veces generando notificaciones duplicadas |

---

## 2. Cronología Detallada

| Hora (UTC-6) | Evento | Persona(s) Involucrada(s) |
| :--- | :--- | :--- |
| 09/02/2026 - 18:30 | **Inicio del Cambio**: Se desactiva job anterior de notificaciones y se activa nuevo schedule que consume endpoint `/api/v2/notifications` (producción: http://ca-api-prod-355837773731.us-central1.run.app/api/v2/notifications) | Infraestructura |
| 09/02/2026 - 18:30 | **Inicio del Impacto**: Proceso comienza a enviar notificaciones con parámetro `idUsuarioSeg` vacío en string separado por pipes | Usuarios |
| 09/02/2026 - 21:45 | **Detección**: Se reporta primer caso de notificaciones enviadas de manera masiva a múltiples usuarios | Operaciones |
| 09/02/2026 - 22:00 | **Inicio de la Respuesta**: Equipo revisa logs y confirma que string de parámetros no contiene `idUsuarioSeg` del superior, causando que conProcesos NCTI envíe a todos los usuarios | Equipo de desarrollo |
| 10/02/2026 - 09:00 | **Mitigación Inmediata**: Se detiene endpoint de notificaciones para contener el daño | Equipo de desarrollo |
| 10/02/2026 - 09:00 | **Prioridad Alta**: Se inicia depuración de notificaciones enviadas masivamente durante la noche | Equipo de desarrollo |
| 10/02/2026 - 10:00 | **Análisis de Causa**: Se confirma que `authorizerData` (idUsuarioSeg) llegaba vacío porque no se encontró autorizador de la solicitud, causando envío broadcast | Equipo de desarrollo |
| 10/02/2026 - 11:00 | **Desarrollo de Fix**: Se agregan validaciones para el parametro (authorizerData) | Equipo de desarrollo |
| 10/02/2026 - 11:30 | **Modificación de SP**: Se actualiza `Spc_listaDatosNotificacion0` para validar destinatarios antes de enviar a conProcesos NCTI. Si no hay destinatarios válidos, se marca proceso con estatus Error(5) | Equipo de desarrollo |
| 10/02/2026 - 12:00 | **Testing**: Se replican escenarios para validar comportamiento con datos vacíos | Equipo de desarrollo |
| 10/02/2026 - 13:00 | **Mejora Adicional**: Se agregan logs de información para auditoría y se revisa lógica de estados (FinalizeProcessStepAsync) | Equipo de desarrollo |
| 10/02/2026 - 20:00 | **Primer Despliegue**: Se libera cambio con validaciones y corrección de flujo de estados | Equipo de desarrollo |
| 10/02/2026 - 20:50 | **Detección Secundaria**: Durante monitoreo se detecta que múltiples ejecuciones procesan mismo `idProcesoEspecialSolicitud` (problema de concurrencia) | Equipo de desarrollo |
| 10/02/2026 - 21:00 | **Mitigación**: Se detiene proceso nuevamente para corregir problema de concurrencia | Equipo de desarrollo |
| 10/02/2026 - 21:10 | **Análisis de Concurrencia**: Se confirma que múltiples instancias del schedule consultan simultáneamente registros con `estatusProceso = NULL` | Equipo de desarrollo |
| 10/02/2026 - 22:20 | **Desarrollo de Fix Concurrencia**: Se crea enum `ProcessStepStatus` y se implementa actualización masiva a `estatusProceso = 2 (Processing)` inmediatamente después de recuperar registros usando `ExecuteUpdateAsync` | Equipo de desarrollo |
| 10/02/2026 - 22:20 | **Segundo Despliegue**: Se libera cambio con state locking para prevenir procesamiento concurrente | Equipo de desarrollo |
| 10/02/2026 - 22:35 | **Verificación Parcial**: Se confirma que registros se marcan correctamente como Processing(2) antes de procesarse. | Equipo de desarrollo |
| 11/02/2026 - 12:30 | **Aclaración de Negocio**: Se valida con el equipo de desarrollo que `idProcesoEspecial = 2` no envía notificaciones (regla de negocio: vacaciones ya autorizadas con estatus=5) | Equipo de desarrollo |
| 11/02/2026 - En curso | **Monitoreo Activo**: Continúa seguimiento de logs y comportamiento del sistema | Equipo de desarrollo  / PO |


---

## 3. Análisis de la Causa Raíz

### 3.1. ¿Cuál fue la Causa Raíz?

**Problema 1 - Envío masivo de notificaciones:**
Cuando el proceso de búsqueda de autorizador (`FindRequestAuthorizerAsync`) no encontraba el `idUsuarioSeg`, el parámetro `authorizerData` se enviaba vacío en el string de parámetros separados por pipes. El sistema externo **conProcesos de NCTI** interpreta un destinatario vacío como "enviar a TODOS los usuarios", causando notificaciones broadcast masivas no deseadas.

**Problema 2 - Procesamiento duplicado:**
El nuevo schedule ejecuta el endpoint `/api/v2/notifications` cada 5 minutos programado. Múltiples ejecuciones simultáneas consultaban registros con `estatusProceso = NULL` y los procesaban concurrentemente, resultando en el mismo `idProcesoEspecialSolicitud` ejecutándose múltiples veces.

**Problema 3 - Sobrescritura de errores:**
`FinalizeProcessStepAsync` se ejecutaba incondicionalmente después del procesamiento, marcando registros fallidos como `Completed(0)` y ocultando errores críticos.

### 3.2. Detalle del Problema

**Envío masivo:**
- En `GenerateRequestNotificationService`, cuando `FindRequestAuthorizerAsync` no encontraba autorizador, `authorizerData` quedaba vacío
- El string de parámetros se construía: `"Descripcion|message||ValorAdicional|env|NotifierId,team"` (campo 3 vacio)
- **conProcesos NCTI** interpreta destinatario vacío como broadcast → notificación enviada a todos los usuarios registrados
- Durante la noche del 09/02 se enviaron notificaciones masivas que debieron depurarse manualmente

**Procesamiento duplicado:**
- Schedule configurado ejecuta endpoint cada 5 minutos
- Ejecución 1 (09:06:00): `GetPendingProcessStepsAsync()` recupera 3 o mas registros (NULL)
- Ejecución 2 (09:07:00): Mientras Ejecución 1 procesa, recupera los mismos 3 registros (aún NULL)
- Resultado: 3 registros procesados múltiples veces (reportado "3 registros procesados 8 veces")

**Migración de job a schedule:**
- Job anterior: Ejecutaba lógica directamente con control interno
- Schedule nuevo: Llama endpoint REST `/api/v2/notifications` sin considerar ejecuciones concurrentes
- No se implementó mecanismo de bloqueo optimista/pesimista

### 3.3. Factores Contribuyentes

* **Cambio de arquitectura sin validar concurrencia**: Migración de job a schedule HTTP no consideró múltiples llamadas simultáneas
* **Falta de validaciones defensivas**: No se validaba que `authorizerData` tuviera valor antes de enviar a sistema externo
* **Comportamiento no documentado de conProcesos NCTI**: No era evidente que parámetro vacío causaría broadcast masivo
* **Testing insuficiente**: No se probó escenario de autorizador no encontrado ni concurrencia del schedule
* **Ausencia de feature flag**: No había forma rápida de desactivar el proceso sin detener el schedule completo
* **Logging insuficiente**: No había logs de auditoría que mostraran qué parámetros se enviaban a conProcesos
* **Brecha en Code Review**: No se identificó riesgo de concurrencia en nueva arquitectura / hizo falta revisión mas a detalle

### 3.4. Aplicación de los 5 Porqués

**Para envío masivo:**
1. **Se enviaron notificaciones a todos los usuarios** ¿Por qué? Porque conProcesos NCTI recibió parámetro de destinatario vacío
2. **El parámetro estaba vacío** ¿Por qué? Porque `authorizerData` (idUsuarioSeg) no tenía valor al construir el string
3. **authorizerData no tenía valor** ¿Por qué? Porque `FindRequestAuthorizerAsync` no encontró autorizador para esa solicitud debido a que los autorizadores no tienen configurado correctamente los datos que el proceso necesita (aplicaJerarquia, rol)
4. **No se encontró autorizador** ¿Por qué? Porque puede haber solicitudes con configuración incompleta o flujo especial
5. **No se validó si authorizerData estaba vacío** ¿Por qué? Porque se asumió que siempre vendría con valor y no se conocía el comportamiento broadcast de conProcesos NCTI

**Para procesamiento duplicado:**
1. **Los registros se procesaban múltiples veces** ¿Por qué? Porque el schedule ejecutaba el endpoint cada X minutos sin esperar a que termine la ejecución anterior
2. **Múltiples ejecuciones concurrentes** ¿Por qué? Porque el endpoint no tenía control de concurrencia ni state locking
3. **No había state locking** ¿Por qué? Porque el diseño no marcaba registros como "en proceso" inmediatamente
4. **No se marcaban como Processing** ¿Por qué? Porque la migración de job a schedule no consideró este escenario
5. **No se consideró en la migración** ¿Por qué? Porque no se hicieron pruebas de carga ni concurrencia antes de liberar

---


## 4. Acciones y Plan de Seguimiento

| Acción Correctiva | Prioridad | Responsable | Fecha Límite | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **Corto Plazo**: Agregar validaciones para `authorizerData` y todos componentes de parámetros en `GenerateRequestNotificationService` | Alta | Programador | 10/02/2026 | Completado |
| **Corto Plazo**: Crear enum `ProcessStepStatus` (Completed=0, Processing=2, Error=3, null=Pending) | Alta | Programador | 10/02/2026 | Completado |
| **Corto Plazo**: Implementar actualización masiva con `ExecuteUpdateAsync` a `estatusProceso=2` inmediatamente después de recuperar registros | Alta | Programador | 11/02/2026 | Completado |
| **Corto Plazo**: Corregir lógica de `FinalizeProcessStepAsync` para ejecutar solo cuando `processSuccessful == true` | Alta | Programador | 10/02/2026 | Completado |
| **Corto Plazo**: Agregar logging de auditoría de parámetros antes de enviar a conProcesos NCTI | Alta | Programador | 10/02/2026 | Completado |
| **Corto Plazo**: Documentar comportamiento de `idProcesoEspecial = 2` (vacaciones autorizadas, no envía notificación) confirmado con César | Media | Programador | 11/02/2026 | Completado |
| **Mediano Plazo**: Agregar try-catch con logging en `UpdateStepsToProcessingAsync` y detener proceso si marca masiva falla | Alta | Programador | 13/02/2026 | Pendiente |
| **Mediano Plazo**: Implementar monitoreo de registros en estado Processing(2) por más de X minutos (detectar registros huérfanos) | Alta | Programador | 18/02/2026 | Pendiente |
| **Largo Plazo**: Implementar validación automática de configuración de autorizadores (aplicaJerarquia, rol) al momento de alta/modificación | Media | programador / DBA | 28/02/2026 | Pendiente |
| **Mediano Plazo**: Implementar sesión de explicación obligatoria (meet interno) antes de code review para cambios de riesgo medio/alto, donde el desarrollador explica contexto y funcionalidad al revisor | Alta | Programador | 20/02/2026 | Pendiente |
| **Mediano Plazo**: Establecer estándar mínimo de code review: revisor debe bajar código, probarlo localmente, validar escenarios edge cases y proponer "¿qué pasa si...?". Code reviews no deben ser aprobaciones de 5 minutos | Alta | Programador | 20/02/2026 | Pendiente |

---

## 5. Lecciones Aprendidas

### Lo que se hizo bien (Keep)
* **Reacción rápida del equipo**: Se reportó el incidente a las 21:45 y se analizó la causa raíz esa misma noche. A primera hora del día siguiente (09:00) se detuvo el endpoint y se inició la depuración de notificaciones enviadas masivamente
* **Priorización y dedicación**: Se dio máxima prioridad al día siguiente con equipo completo trabajando en la solución
* **Análisis sistemático**: Se identificaron y corrigieron 3 problemas distintos de manera incremental (validaciones, estado de errores, concurrencia)
* **Comunicación efectiva**: Coordinación entre desarrollo, operaciones y PO para aclarar comportamiento de negocio (`idProcesoEspecial = 2`)

### Áreas de mejora (Improve)
* **Mejorar proceso de code review**: Enfocarse no solo en aspectos técnicos sino también en riesgos de concurrencia, validaciones defensivas y comportamiento de sistemas externos
* **Testing pre-producción exhaustivo**: Implementar pruebas de carga y concurrencia antes de liberar cambios arquitectónicos (migración job → schedule)
* **Validación de configuración de autorizadores**: Implementar validación automática al momento de alta/modificación para prevenir datos incompletos (aplicaJerarquia, rol) que causen problemas en producción
* **Documentación de dependencias externas**: Documentar comportamiento crítico de APIs externas (ej: conProcesos NCTI interpreta parámetro vacío como broadcast) antes de integrar

---

**Notas adicionales para monitoreo:**
- ✅ Validar en próximas 48 horas que no hay más procesamiento duplicado
- ✅ Confirmar que todas las notificaciones lleguen solo a destinatarios correctos (no broadcast)
- 📊 Analizar logs para determinar frecuencia óptima del schedule vs tiempo promedio de procesamiento