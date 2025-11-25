# 💀 Post Mortem de Incidencia: [Título de la Incidencia]

## 1. Información General del Evento

| Campo | Valor |
| :--- | :--- |
| **Título de la Incidencia** | [Descripción concisa del problema] |
| **ID de Incidencia/Ticket** | [Ej: INC-2025-01-18-001] |
| **Fecha y Hora de Detección** | [DD/MM/AAAA HH:MM UTC] |
| **Fecha y Hora de Inicio del Impacto** | [DD/MM/AAAA HH:MM UTC] |
| **Fecha y Hora de Resolución** | [DD/MM/AAAA HH:MM UTC] |
| **Duración del Impacto** | [Ej: 1 hora y 15 minutos] |
| **Servicio(s) / Sistema(s) Afectado(s)** | [Ej: API de Pagos, Portal de Clientes, Base de Datos Principal] |
| **Impacto para el Usuario/Negocio** | [Breve descripción del efecto. Ej: 10% de transacciones fallidas en EMEA.] |

---

## 2. Cronología Detallada

| Hora (UTC / Local) | Evento | Persona(s) Involucrada(s) |
| :--- | :--- | :--- |
| HH:MM | **Detección**: [Describe cómo se identificó, Ej: Alerta en PagerDuty por alto uso de CPU.] | [Nombre/Sistema] |
| HH:MM | **Inicio de la Respuesta**: [Ej: Se convocó al equipo de guardia y se inició la investigación.] | [Nombre/Equipo] |
| HH:MM | **Identificación de la Causa Raíz** | [Nombre/Equipo] |
| HH:MM | **Mitigación / Solución Aplicada**: [Ej: Se revirtió el despliegue a la versión anterior (v1.2.5).] | [Nombre/Equipo] |
| HH:MM | **Resolución / Verificación de Estabilidad** | [Nombre/Equipo] |

---

## 3. Análisis de la Causa Raíz

### 3.1. ¿Cuál fue la Causa Raíz?
[La razón fundamental, no el síntoma. Ej: Un cambio de configuración reciente en el balanceador de carga no manejó correctamente el tráfico en la región de Asia.]

### 3.2. Detalle del Problema
[Descripción técnica y detallada de lo que falló. Incluye la secuencia de eventos que llevó al fallo.]

### 3.3. Factores Contribuyentes
* [Ej: La alerta de latencia estaba configurada con un umbral demasiado alto.]
* [Ej: La revisión de código no detectó la posible condición de carrera.]
* [Ej: La documentación del proceso de 'rollback' estaba desactualizada.]

### 3.4. Aplicación de los 5 Porqués
1. **El impacto ocurrió...** ¿Por qué? [Respuesta]
2. **[Respuesta 1] ocurrió...** ¿Por qué? [Respuesta]
3. **[Respuesta 2] ocurrió...** ¿Por qué? [Respuesta]
4. **[Respuesta 3] ocurrió...** ¿Por qué? [Respuesta]
5. **[Respuesta 4] ocurrió...** ¿Por qué? [Respuesta - Causa Raíz Final]

---

## 4. Acciones y Plan de Seguimiento

| Acción Correctiva | Prioridad | Responsable | Fecha Límite | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **Corto Plazo**: [Ej: Agregar una prueba de estrés para el nuevo servicio de autenticación.] | Alta | [Nombre/Equipo] | DD/MM/AAAA | Pendiente |
| **Largo Plazo**: [Ej: Implementar un sistema de 'canary release' para todos los despliegues de la API.] | Media | [Nombre/Equipo] | DD/MM/AAAA | Pendiente |
| **Documentación**: [Ej: Actualizar la documentación del runbook de DR (Disaster Recovery) para el servicio afectado.] | Baja | [Nombre/Equipo] | DD/MM/AAAA | Pendiente |

---

## 5. Lecciones Aprendidas

### Lo que se hizo bien (Keep)
* [Ej: La comunicación inicial del impacto a los stakeholders fue rápida y precisa.]
* [Ej: El proceso de escalado funcionó correctamente.]

### Áreas de mejora (Improve)
* [Ej: Se tardó demasiado en correlacionar los logs de diferentes servicios. Necesitamos una herramienta de centralización de logs más eficiente.]
* [Ej: Falta de automatización para el diagnóstico inicial.]