# 💀 Post Mortem de Incidencia: Usuario no cuenta con el ambiente asignado 

## 1. Información General del Evento

| Campo | Valor |
| :--- | :--- |
| **Título de la Incidencia** | Usuario no cuenta con el ambiente asignado |
| **ID de Incidencia/Ticket** | INC-2025-12-1-US-Impacto|
| **Fecha y Hora de Detección** | 1/12/2025 12:28 p.m UTC |
| **Fecha y Hora de Inicio del Impacto** | 1/12/2025 12:28 p.m UTC |
| **Fecha y Hora de Resolución** | 1/12/2025  1:22 p.m. UTC |
| **Duración del Impacto** | 54 minutos |
| **Servicio(s) / Sistema(s) Afectado(s)** | Smart Teams Android y IOS|
| **Impacto para el Usuario/Negocio** | Imposibilidad total de iniciar sesión, en Smart Teams Android y IOS si no tenian la aplicacion de control de asistencia asignada |

---

## 2. Cronología Detallada

| Hora (UTC) | Evento | Persona(s) Involucrada(s) |
| :--- | :--- | :--- |
| Viernes - 15:00 | Se publican las versiones correspondientes a la migracion de API. | Play Store - App Store |
| Lunes - 12:28 | **Inicio del Impacto**: Llegaron algunos reportes de los usuarios de la marca danone. | PO Smart Teams |
| Lunes - 12:40 | **Detección**: Se debuggeo con el usuario proporcionado para encontrar el error. | Mobile Developer Sr (Lider Tech) |
| Lunes -  12:45 | **Inicio de la Respuesta**: Mobile Developer Sr confirma que en ambas aplicaciones se esta enviando de manera incorrecta el id aplicacion en el servicio de login de Smart Teams. | Mobile Developer Sr |
|  Lunes - 12:50 | **Identificación de la Causa Raíz (Preliminar)**: Se revisa los controladores donde viene el idAplicacion estatico | Equipo de Desarrollo / Mobile Developer Sr |
| Lunes - 12:50 | **Mitigación / Solución Aplicada**: Se genera una nueva version y se comparte via chat de inmediato se envia a tiendas para su publicacion automatica. | Play Store & AppStore / Mobile Developer Sr |
| Lunes - 12:55 | **Resolución / Verificación de Estabilidad**: Se hace un test general con el usuario y se verifica que tenga el acceso correcto en ambas plataformas | Mobile Developer Sr |
| Lunes - 13:22 | **Resolución / Verificación de Estabilidad**: Se compartio una version via chat al equipo que reporto el error para su distribucio | Mobile Developer Sr |
| Lunes - 13:57 | **Resolución / Verificación de Estabilidad**: Se monitorean las tiendas para ver el estatus de la aplicacion y ya puede ser publicada (Play Store) | Mobile Developer Sr |
| Lunes - 06:33 | **Resolución / Verificación de Estabilidad**: Se monitorean las tiendas para ver el estatus de la aplicacion y ya puede ser publicada (App Store) | Mobile Developer Sr |



---

## 3. Análisis de la Causa Raíz

### 3.1. ¿Cuál fue la Causa Raíz?
En la migracion de api se dejo estatico el idapp que se envia en el servicio de login estatico en 2, lo que ocasionaba que los usuarios que no tuvieran asiganda esa app en el ambiente de produccion no pudieran acceder a Smart Teams debido a la configuracion

### 3.2. Detalle del Problema
En el refactor de peticiones se codifico de manera incorrecta el id aplicacion 2 que se envia en el servicio de validacion de correo y contraseña, el equipo de desarrollo, testing.

### 3.3. Factores Contribuyentes
* **Fallo en el *Health Check***: El *Health Check* actual solo revisa que el app estuviera funcionando de manera correcta con usuarios con una configuracion similar. 
* **Brecha en la Revisión**: No se hizo un correcto code-review al revisar los cambios en las peticiones.
* **Ausencia de Pruebas de Despliegue**: Las pruebas de *preproduccion* no simularon correctamente el entorno de producción.

### 3.4. Aplicación de los 5 Porqués
1. **El servicio falló...** ¿Por qué? Porque no pudo iniciar sesion(Error 500).
2. **No pudo inicializar el token...** ¿Por qué? Porque el idAplicacion que se envia en el servicio estaba incorrecto.
3. **La variable estaba mal configurada...** ¿Por qué? se tiene de manera estatica en el codigo al implentar estos cambios no se verifico que se respetara el id correcto .
4. **El cambio no se reviso...** ¿Por qué? al hacer la prueba general no se hizo un code review a conciencia.
5. **Este error de configuración no fue capturado...** ¿Por qué? **Porque el *Health Check* fue  demasiado superficial y no se valido con un usuario con configuracion diferente de perfil.** (Causa Raíz Final)

---

## 4. Acciones y Plan de Seguimiento

| Acción Correctiva | Prioridad | Responsable | Fecha Límite | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **Corto Plazo**: Se genero una variable de entorno que controla el idAplicacion general de Smart Teams | Alta | Mobile Developer Sr (TechLead) | 01/12/2025 | Completado |
| **Largo Plazo**: **Implementar un *Deep Health Check*** se realizara un test que revise estas variables estaticas para evitar estos errores en caso de ser distinto no dejara complicar  | Alta | Mobile Developer Team | 05/12/2025 | Pendiente |
| **Proceso**: Tener al menos 2 usuarios con configuraciones distintas para generar flujo de pruebas general | Media | Mobile Developer Team | 15/12/2025 | Pendiente |
| **Documentación**: Hacer un script que valide estos parametros para evitar errores asi. | Baja | Mobile Developer Team | 15/12/2025 | Pendiente |

---

## 5. Lecciones Aprendidas

### Lo que se hizo bien (Keep)
* **Rapidez del Rollback:** El proceso de rollback se puede generar desde la tienda agilizando una respuesta para futuras ocaciones sin necesidad de publicar nueva version.
* **Alertas Claras:** Mayor contro en la configuracion y ambientacion de las aplicaciones moviles.
* **Comunicación Interna:** El estado del incidente se comunicó a los stakeholders en los primeros 4 dias se nos dificulto rastrear el error y que no lo reportaran.

### Áreas de mejora (Improve)
* **Validación de Configuración:** Generar un test unitario que nos indiquen este tipo de configuraciones con el objetivo de correr dichos test antes de generar nuevas publicaciones .
* **Procedimiento de Despliegue:** Implementacion de test unitarios para revisar estos puntos criticos que pueden generar falsos positivos.