# 💀 Post Mortem de Incidencia: Error 500 en la API de Autenticación tras Despliegue

## 1. Información General del Evento

| Campo | Valor |
| :--- | :--- |
| **Título de la Incidencia** | Error 500 generalizado en la API de Autenticación (Auth API) |
| **ID de Incidencia/Ticket** | INC-2025-11-18-042 |
| **Fecha y Hora de Detección** | 18/11/2025 10:05 UTC |
| **Fecha y Hora de Inicio del Impacto** | 18/11/2025 10:02 UTC |
| **Fecha y Hora de Resolución** | 18/11/2025 10:25 UTC |
| **Duración del Impacto** | 23 minutos |
| **Servicio(s) / Sistema(s) Afectado(s)** | Auth API (Microservicio de autenticación), API Gateway |
| **Impacto para el Usuario/Negocio** | Imposibilidad total de iniciar sesión, registrarse, o acceder a cualquier servicio que requiriera autenticación. Impacto en el 100% de los usuarios. |

---

## 2. Cronología Detallada

| Hora (UTC) | Evento | Persona(s) Involucrada(s) |
| :--- | :--- | :--- |
| 10:00 | Despliegue de la versión `v2.4.0` de Auth API iniciado por el pipeline de CI/CD. | Sistema Automático |
| 10:02 | **Inicio del Impacto**: La primera instancia de `v2.4.0` pasa las comprobaciones de salud (`Health Check`) y comienza a recibir tráfico. Los logs muestran errores 500 masivos. | Sistema de Monitoreo |
| 10:05 | **Detección**: Alerta de PagerDuty activada por `Error Rate > 5%` en el servicio Auth API. | Juan (DevOps On-Call) |
| 10:08 | **Inicio de la Respuesta**: Juan confirma el error 500 en todas las rutas de la Auth API. Se inicia el `rollback`. | Juan (DevOps) |
| 10:15 | **Identificación de la Causa Raíz (Preliminar)**: Se detecta que el error ocurrió inmediatamente después del despliegue. Se sospecha de la nueva versión. | Equipo de Desarrollo / Juan |
| 10:20 | **Mitigación / Solución Aplicada**: El `rollback` a la versión `v2.3.9` finaliza. Las instancias antiguas comienzan a servir tráfico. | Sistema Automático / Juan |
| 10:25 | **Resolución / Verificación de Estabilidad**: Tasa de error de la Auth API vuelve a 0%. Se verifica la funcionalidad de inicio de sesión. | Juan |

---

## 3. Análisis de la Causa Raíz

### 3.1. ¿Cuál fue la Causa Raíz?
Una variable de entorno crítica (`JWT_SECRET_KEY`) se renombró en la versión `v2.4.0` de la aplicación, pero la configuración en el sistema de despliegue (Kubernetes/ECS) no se actualizó, lo que provocó que el servicio no pudiera inicializar la clave de cifrado y fallara al arrancar.

### 3.2. Detalle del Problema
La versión `v2.4.0` introdujo un refactor donde la variable `AUTH_JWT_SECRET` fue renombrada a `JWT_SECRET_KEY`. Este cambio se documentó en la tarea de desarrollo, pero se olvidó incluir la actualización del archivo de configuración de variables de entorno (`.env.prod` o ConfigMap/Secret). La aplicación falló silenciosamente al inicializar la clave, lanzando una excepción no manejada durante el *startup*, lo que resultó en errores HTTP 500 generalizados.

### 3.3. Factores Contribuyentes
* **Fallo en el *Health Check***: El *Health Check* actual solo revisa si el puerto está abierto, no si la aplicación está completamente operativa y lista para manejar solicitudes.
* **Brecha en la Revisión**: La revisión de código (`Code Review`) se centró en la lógica de negocio y no verificó el impacto del cambio de nombre de la variable en la configuración del despliegue.
* **Ausencia de Pruebas de Despliegue**: Las pruebas de *staging* no simularon correctamente el entorno de producción, donde la variable se obtiene de un Secret y no de un valor codificado.

### 3.4. Aplicación de los 5 Porqués
1. **El servicio falló...** ¿Por qué? Porque no pudo inicializar el token JWT (Error 500).
2. **No pudo inicializar el token...** ¿Por qué? Porque la variable de entorno `JWT_SECRET_KEY` estaba vacía.
3. **La variable estaba vacía...** ¿Por qué? Porque su nombre había cambiado en el código, pero el sistema de despliegue seguía usando el nombre antiguo (`AUTH_JWT_SECRET`).
4. **El sistema de despliegue no se actualizó...** ¿Por qué? Porque el ingeniero olvidó actualizar el `ConfigMap` al crear la Pull Request y no se incluyó en la lista de revisión de despliegue.
5. **Este error de configuración no fue capturado...** ¿Por qué? **Porque el *Health Check* es demasiado superficial y no valida que todos los componentes críticos (como la clave secreta) estén cargados correctamente.** (Causa Raíz Final)

---

## 4. Acciones y Plan de Seguimiento

| Acción Correctiva | Prioridad | Responsable | Fecha Límite | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **Corto Plazo**: Actualizar la configuración de variables de entorno para `v2.4.0` y hacer un despliegue manual. | Alta | Juan (DevOps) | 18/11/2025 | Completado |
| **Largo Plazo**: **Implementar un *Deep Health Check*** que valide la carga de secretos/variables críticas durante el arranque del servicio. | Alta | Equipo de Plataforma | 05/12/2025 | Pendiente |
| **Proceso**: Agregar la revisión de *ConfigMaps* y *Secrets* como un paso obligatorio en la lista de chequeo de la plantilla de Pull Request (PR). | Media | Equipo de Desarrollo | 28/11/2025 | Pendiente |
| **Documentación**: Crear un *runbook* específico para "Fallo de Carga de Variables de Entorno en Despliegue". | Baja | María (Dev) | 01/12/2025 | Pendiente |

---

## 5. Lecciones Aprendidas

### Lo que se hizo bien (Keep)
* **Rapidez del Rollback:** El proceso de `rollback` fue rápido y automatizado, lo que limitó la duración del impacto a 23 minutos.
* **Alertas Claras:** La alerta de PagerDuty fue precisa y activó al equipo adecuado inmediatamente.
* **Comunicación Interna:** El estado del incidente se comunicó a los stakeholders en los primeros 10 minutos.

### Áreas de mejora (Improve)
* **Validación de Configuración:** Nunca asumir que un servicio es saludable solo porque su puerto está abierto. Se necesita una validación más profunda del estado de la aplicación.
* **Procedimiento de Despliegue:** Mejorar la etapa de pruebas del *pipeline* para verificar que los *ConfigMaps* y *Secrets* necesarios existen y se cargan correctamente antes de que el tráfico sea dirigido a las nuevas instancias.