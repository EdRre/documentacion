# 💀 Post Mortem de Incidencia: Usuario no cuenta con el ambiente asignado 

## 1. Información General del Evento

| Campo | Valor |
| :--- | :--- |
| **Título de la Incidencia** | Usuario no visualiza solicitud para autorizar |
| **ID de Incidencia/Ticket** | N/A |
| **Fecha y Hora de Detección** | 27/01/2026 18:00 |
| **Fecha y Hora de Inicio del Impacto** | 27/01/2026 18:00 |
| **Fecha y Hora de Resolución** | 29/01/2026 21:00  |
| **Duración del Impacto** | 2 días y 3 horas |
| **Servicio(s) / Sistema(s) Afectado(s)** | Control de asistencia |
| **Impacto para el Usuario/Negocio** | Los usuarios no visualizan solicitudes para autorizar y estas se cancelaron por vigencia |

---

## 2. Cronología Detallada

| Hora (UTC) | Evento | Persona(s) Involucrada(s) |
| :--- | :--- | :--- |
| 2026-01-23 | Se libera desarrollo para migrar login de CA a tokens JWT | Desarrollo CA |
| 2026-01-23 a 2026-01-26 | **Inicio del Impacto**: Se realizan autorizaciones que llegan sin ID del autorizador | Usuarios |
| 2026-01-27 - 18:00 | **Detección**: Se reporta error al equipo de desarrollo | Operaciones |
| 2026-01-27 - 19:00 | **Inicio de la Respuesta**: Desarrollo valida que el problema se debe a que las autorizaciones se realizaban sin enviar el id del autorizador | Equipo de desarrollo |
| 2026-01-27 - 20:00 | **Proceso de resolución de incidencia**: Se trabaja en el mantenimiento de las solicitudes que no tienen correctamente a su autorizador | Equipo de Desarrollo |
| 2026-01-28 - 09:00 | **Mitigación**: Se aplica mantenimiento a contestaciones, se regresan a un paso anterior para que se puedan autorizar nuevamente y se actualiza el autorizador de las solicitudes que ya estaban autorizadas, pero que su autorizador no se había registrado | Equipo de desarrollo / PO |
| 2026-01-28 - 21:00 | **Mitigación**: Se autorizan correctamente las solicitudes canceladas por vigencia, pero se identifica un nuevo problema, los flujos de autorización no se establecen correctamente aún cuando el id del autorizador se carga correctamente | Equipo de desarrollo / PO |
| 2026-01-29 - 09:00 | **Mitigación**: Se realiza análisis del problema en los flujos de las solicitudes y se identifica que la configuración de los autorizadores no esta completa (falta configurar roles y flujos  y configurar el uso de jerarquía) | Equipo de desarrollo |
| 2026-01-29 - 13:00 | **Mitigación**: Se inicia proceso de mantenimiento a los usuarios para que pueda generarse correctamente el flujo de autorizaciones | Equipo de desarrollo |
| 2026-01-27 - 13:55 | **Resolución / Verificación de Estabilidad**: Se confirma que los usuarios ya pueden ver correctamente las solicitudes | PO |



---

## 3. Análisis de la Causa Raíz

### 3.1. ¿Cuál fue la Causa Raíz?
Durante la migración del proceso de login se dejó un bug en el proceso de autorizaciones masivas que ocasionaba que el id del usuario que autoriza una solicitud no se registre en la bitácora, esto ocasiona que en el detalle de las solicitudes los autorizadores no se muestren correctamente.
Así mismo, se identificó que el problema de que los autorizadores no vean las solicitudes se deben a que no estaban configurados correctamente. En este sentido, algunos no tenían el rol o no tenían el flag aplicaJerarquía por lo que el proceso de buscar al autorizador no los tomaba en cuenta y no les presentaba las solicitudes.

### 3.2. Detalle del Problema
En la migración del login se eliminó un parámetro por default que se utilizaba para autorizaciones desde el detalle de solicitud como desde las autorizaciones masivas, al quitarlo se agregó el parámetro desde la función de las autorizaciones individuales, pero no se agregó en las masivas.
Referente a la configuración de usuarios, se asume que estos e debió a un error durante la carga de los mismos.

### 3.3. Factores Contribuyentes
* **Errores en la configuración de usuarios**: El proceso de configuración de usuarios es un proceso manual en el que se pueden generar errores de configuración ya que se pueden omitir datos que son necesarios para el sistema. 
* **Brecha en la Revisión**: No se hizo un correcto code-review que permitiera identificar que el parámetro que se quita afectaba en más de un módulo.
* **Errores en validacion de api**: El proceso que realiza las autorizaciones de los permisos/vacaciones no valida que el id del usuario autorizador no venga en null o 0.

### 3.4. Aplicación de los 5 Porqués
1. **El servicio falló...** ¿Por qué? Porque los usuarios no podían autorizar solicitudes ya que no las veían en sus lista de solicitudes a autorizar. Además la bitácora no registraba quien autoriza.
2. **Porque no veían las solicitudes** ¿Por qué? Porque los autorizadores no tienen configurado correctamente los datos que el proceso que busca al autorizador necesita (aplicaJerarquia, rol).
3. **La bitácora no mostraba al autorizador** ¿Por qué? porque el id del autorizador no se registraba debido a un bug de un desarrollo previo .
4. **El flujo no se encontraba** ¿Por qué? porque el proceso de buscar autorizador se realiza hasta que el usuario consulta sus solicitudes a autorizar, y hasta ese momento se puede observar que no esta bien configurado.
5. **La bitácora permite almacenar autorizador en 0** ¿Por qué? porque no se realizaba ninguna validación que nos permitiera identificar este problema

---

## 4. Acciones y Plan de Seguimiento

| Acción Correctiva | Prioridad | Responsable | Fecha Límite | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **Corto Plazo**: Se corrigen las configuraciones de autorizadores y se asignan los autorizadores a las solicitudes que ya estan autorizadas. Se despliegan validación para no permitir autorizar si no se envía el id del autorizador | Alta | Teach Lead / Sql Developer | 29/01/2025 | Completado |
| **Largo Plazo**: **Mejorar proceso de autorizaciones** se realizará un análisis exhaustivo para identificar de manera temprana cuando un flujo de autorizaciones no se genera de manera correcta  | Alta | Teach Lead | 12/02/2026 | Pendiente |
| **Proceso**: Se analizarán y validarán propuestas de solución en conjunto con PO | Media | Teach Lead / PO | 15/12/2025 | Pendiente |
| **Documentación**: Hacer un script que valide estos parametros para evitar errores asi. | Baja | Mobile Developer Team | 18/02/2026 | Pendiente |

---

## 5. Lecciones Aprendidas

### Lo que se hizo bien (Keep)
* **Reacción del equipo:** Posibilidad de dar mantenimiento a los registros afectados.
* **Comunicación Interna:** Comunicación correcta ente los implicados para dar solución al problema con afectación de un mes.

### Áreas de mejora (Improve)
* **Mejorar proceso de code review:** Mejorar el proceso de code review para identificar afectaciones de negocio y no enfocarse únicamente en aspectos técnicos.
* **Capacitación en la configuración de usuarios:** Capacitar a los usuarios que configuran a los autorizadores para reforzar los puntos claves en la configuración
* **Mejorar proceso de autorizaciones:** Buscar opciones para identificar de manera temprana problemas de configuración de autorizadores y así impedir que una solicitud se genere con un flujo incorrecto