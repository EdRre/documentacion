# PRD - Endpoints de Desregistro de Usuarios

## 1. Resumen Ejecutivo

### 1.1 Descripción General
Este documento describe los requisitos del producto para tres endpoints especializados del sistema de seguridad JWT que gestionan el proceso completo de baja de usuarios corporativos, desde el registro inicial de la baja hasta su eliminación definitiva del sistema.

### 1.2 Objetivos del Producto
- Automatizar el proceso de baja de usuarios en el sistema de seguridad corporativo
- Garantizar el procesamiento gradual y controlado de las bajas de usuarios
- Mantener trazabilidad completa de todo el proceso de desregistro
- Permitir la desactivación parcial de operaciones específicas antes de la baja total
- Procesar bajas programadas de manera automatizada mediante tareas diarias
- Eliminar definitivamente usuarios después de un período de retención de 30 días

### 1.3 Alcance
**En Alcance:**
- Endpoint de baja inicial (`/deregistration`)
- Endpoint de procesamiento diario (`/daily-deregistration`)
- Endpoint de baja definitiva (`/definitive-deregistration`)
- Integración con sistema Pub/Sub para mensajería asíncrona
- Gestión de estados del proceso de baja
- Desactivación parcial y total de usuarios

**Fuera de Alcance:**
- Interfaz de usuario para gestión manual
- Notificaciones por email a usuarios
- Reversión automática de bajas procesadas

---

## 2. Contexto del Negocio

### 2.1 Problema a Resolver
El sistema necesita gestionar el proceso completo de baja de usuarios corporativos de manera controlada, permitiendo:
- Bajas programadas con fecha futura
- Procesamiento parcial por tipo de operación y marca
- Retención temporal de datos antes de eliminación definitiva
- Procesamiento automatizado mediante tareas programadas

### 2.2 Usuarios y Stakeholders
- **Sistemas RHIN**: Envían notificaciones de baja mediante Pub/Sub
- **Administradores del sistema**: Supervisan el proceso de bajas
- **Departamento encargado de realizar las bajas**: Solicitan bajas de empleados

---

## 3. Especificaciones Técnicas

### 3.1 Arquitectura General

```mermaid
graph TB
    A[Sistema Externo] -->|Pub/Sub Message| B[UserSubscriptor Controller]
    B --> C{Tipo de Operación}
    C -->|deregistration| D[UserDeregistrationCommand]
    C -->|daily-deregistration| E[DailyUserDeregistrationCommand]
    C -->|definitive-deregistration| F[DefinitiveDeregistrationCommand]
    
    D --> G[UserDeregistrationHandler]
    E --> H[DailyUserDeregistrationHandler]
    F --> I[DefinitiveDeregistrationHandler]
    
    G --> J[UserDeregistrationService]
    H --> J
    
    J --> K[(Base de Datos)]
    G --> K
    H --> K
    I --> K
    
    K --> L[MovBajasProgramadasTbl]
    K --> M[MovMensajesPubSubTbl]
    K --> N[CatUsuariosTbl]
    K --> O[RelUsuariosTokenTbl]
    K --> P[RelUsuariosAplicacionesTbl]
```

### 3.2 Estados del Proceso de Baja

```mermaid
stateDiagram-v2
    [*] --> RECEIVED: Solicitud de baja recibida
    RECEIVED --> INPROCESS: Procesamiento iniciado
    INPROCESS --> PATIALLY_PROCESSED: Operaciones específicas desactivadas
    INPROCESS --> ERROR: Fallo en procesamiento
    PATIALLY_PROCESSED --> DEFINITIVE_DEREGISTRATION: Después de 30 días
    DEFINITIVE_DEREGISTRATION --> [*]: Usuario eliminado definitivamente
    RECEIVED --> RE_ENTRY: Reingreso del usuario
    PATIALLY_PROCESSED --> RE_ENTRY: Reingreso del usuario
    ERROR --> [*]: Requiere intervención manual
```

**Estados disponibles:**
- `RECEIVED (1)`: Baja registrada, pendiente de procesar
- `INPROCESS (2)`: Baja en proceso de ejecución
- `PATIALLY_PROCESSED (3)`: Baja parcial completada (operaciones específicas desactivadas)
- `DEFINITIVE_DEREGISTRATION (4)`: Baja definitiva completada (después de 30 días)
- `ERROR (5)`: Error durante el procesamiento
- `RE_ENTRY (6)`: Usuario reingresado al sistema

---

## 4. Endpoints Detallados

### 4.1 POST /api/v1/UserSubscriptor/deregistration

#### 4.1.1 Descripción
Endpoint que recibe solicitudes de baja de usuarios desde sistemas externos mediante Pub/Sub. Registra la baja programada y la procesa inmediatamente si la fecha de baja es anterior a la fecha actual. Durante una baja parcial, se mantienen activas únicamente las siguientes aplicaciones:
- **Notificaciones**: Para mantener comunicación con el usuario
- **Recibos de Nómina**: Para acceso a información histórica de pagos

#### 4.1.2 Request

**Headers:**
```
Content-Type: application/json
```

**Body (Envelope de Pub/Sub):**
```json
{
  "message": {
    "messageId": "12345678-1234-1234-1234-123456789abc",
    "data": "base64_encoded_data",
    "publishTime": "2026-01-15T10:30:00Z",
    "attributes": {
      "seguridad":"baja"
    }
  },
  "subscription": "projects/my-project/subscriptions/seguridad-usuario-baja"
}
```

**Estructura del Data decodificado:**
```json
{
    "idEmpleado":242755,
    "idRelLab":551920,
    "fechaBaja":"2026-01-15",
    "fechaEnvio":"2026-01-15",
    "idUsuarioAct":141,
    "idCausaBaja":32,
    "accion":3
}
```

#### 4.1.3 Response

**Success (204 No Content)**
```
No body
```

**Error (400 Bad Request)**
```json
{
  "error": "Invalid Pub/Sub message format"
}
```

#### 4.1.4 Lógica de Negocio

```mermaid
flowchart TD
    A[Recibir Envelope] --> B{¿Envelope válido?}
    B -->|No| C[Return 400 Bad Request]
    B -->|Sí| D[Guardar mensaje en MovMensajesPubSubTbl]
    D --> E[Mapear a MovBajasProgramadasTbl]
    E --> F[Obtener segDatosAdicionalesTbl del empleado]
    F --> H[Guardar registro de baja programada]
    H --> I{¿Fecha baja < Fecha actual?}
    I -->|Sí| J[Procesar baja inmediatamente]
    I -->|No| K[Return 204 - Baja programada]
    J --> L[UserDeregistrationService.ProcessDeregistration]
    L --> K
```

#### 4.1.5 Proceso de Baja (UserDeregistrationService)

```mermaid
flowchart TD
    A[ProcessDeregistrationAsync] --> B[Actualizar estado a INPROCESS]
    B --> C[Obtener datos del usuario]
    C --> D{¿Usuario existe?}
    D -->|No| E[Actualizar a ERROR]
    D -->|Sí| F[Desactivar tipo de operación específica]
    F --> G{¿Operación desactivada?}
    G -->|No| H[Actualizar a ERROR]
    G -->|Sí| I{¿Usuario tiene operaciones activas?}
    I -->|Sí| J[Solo operación desactivada]
    I -->|No| K[Desactivar usuario parcialmente]
    K --> L[Desactivar tokens]
    L --> M[Desactivar email]
    M --> N[Desactivar aplicaciones excepto: Notificaciones y Recibos de Nómina]
    N --> O[Actualizar a PATIALLY_PROCESSED]
    J --> P[Publicar mensaje de éxito]
    O --> P
    P --> Q[Return Success]
    E --> R[Return Error]
    H --> R
```

**Nota importante**: Durante la desactivación de aplicaciones, el sistema mantiene activas únicamente las aplicaciones de **Notificaciones** y **Recibos de Nómina**, permitiendo al usuario continuar recibiendo notificaciones y consultando su historial de pagos incluso después de la baja.

---

### 4.2 POST /api/v1/UserSubscriptor/daily-deregistration

#### 4.2.1 Descripción
Endpoint diseñado para ser invocado diariamente por un scheduler (Cloud Scheduler **seguridad-usuario-baja**). Procesa todas las bajas programadas cuya fecha de baja coincide con la fecha actual y que están en estado `RECEIVED`.

#### 4.2.2 Request

**Headers:**
```
Content-Type: application/json
```

**Body:**
```json
{
  "message": {
    "messageId": "12345678",
    "data": "",
    "publishTime": "2026-01-15T06:00:00Z",
    "attributes": {
      "baja-diaria": "ok"
    }
  },
  "subscription": "projects/my-project/subscriptions/seguridad-usuario-baja-diaria"
}
```

#### 4.2.3 Response

**Success (204 No Content)**
```
No body
```

**Error (400 Bad Request)**
```json
{
  "error": "Invalid Pub/Sub message format"
}
```

#### 4.2.4 Lógica de Negocio

```mermaid
flowchart TD
    A[Recibir trigger diario] --> B{¿Envelope válido?}
    B -->|No| C[Return 400 Bad Request]
    B -->|Sí| D[Guardar mensaje en MovMensajesPubSubTbl]
    D --> E[Consultar bajas programadas]
    E --> F{Filtros aplicados}
    F --> G[FechaBaja.Date == Hoy]
    G --> H[IdEstatus == RECEIVED]
    H --> I{¿Hay usuarios?}
    I -->|No| J[Return 204 - No hay bajas hoy]
    I -->|Sí| K[Iterar usuarios]
    K --> L[ProcessDeregistrationAsync para cada usuario]
    L --> M{¿Más usuarios?}
    M -->|Sí| K
    M -->|No| N[Return 204 - Procesamiento completo]
```

#### 4.2.5 Casos de Uso
1. **Procesamiento automático diario**: Cloud Scheduler dispara este endpoint cada día a las 20 hrs
2. **Procesamiento batch**: Procesa múltiples usuarios en un solo ciclo
3. **Resiliencia**: Cada usuario se procesa individualmente, un error no afecta a los demás

---

### 4.3 POST /api/v1/UserSubscriptor/definitive-deregistration

#### 4.3.1 Descripción
Endpoint para eliminar definitivamente usuarios que llevan más de 30 días en estado `PATIALLY_PROCESSED`. Ejecuta la eliminación completa de tokens, relaciones con aplicaciones y desactivación definitiva del usuario.

#### 4.3.2 Request

**Headers:**
```
Content-Type: application/json
```

**Body:**
```json
{
  "message": {
    "messageId": "1234567890",
    "data": "",
    "publishTime": "2026-01-15T02:00:00Z",
    "attributes": {
      "baja-final": "ok"
    }
  },
  "subscription": "projects/my-project/subscriptions/seguridad-baja-definitiva"
}
```

#### 4.3.3 Response

**Success (204 No Content)**
```
No body
```

**Error (400 Bad Request)**
```json
{
  "error": "Invalid Pub/Sub message format"
}
```

#### 4.3.4 Lógica de Negocio

```mermaid
flowchart TD
    A[Recibir trigger mensual] --> B{¿Envelope válido?}
    B -->|No| C[Return 400 Bad Request]
    B -->|Sí| D[Guardar mensaje en MovMensajesPubSubTbl]
    D --> E[Consultar usuarios PATIALLY_PROCESSED]
    E --> F{Filtrar por fecha}
    F --> G[Fecha actual - FechaBaja > 30 días]
    G --> H{¿Hay usuarios?}
    H -->|No| I[Return 204 - No hay usuarios para eliminar]
    H -->|Sí| J[Iterar usuarios]
    J --> K[Obtener datos adicionales]
    K --> L[Desactivar todos los tokens]
    L --> M[Actualizar estado a DEFINITIVE_DEREGISTRATION]
    M --> N[Desactivar todas las aplicaciones]
    N --> O[Desactivar usuario]
    O --> P{¿Más usuarios?}
    P -->|Sí| J
    P -->|No| Q[Return 204 - Eliminación completa]
```

#### 4.3.5 Criterios de Eliminación
- Estado actual: `PATIALLY_PROCESSED (3)`
- Tiempo transcurrido: > 30 días desde `FechaBaja`
- Proceso irreversible: No existe reingreso después de este punto

---

## 5. Modelos de Datos

### 5.1 MovBajasProgramadasTbl

```mermaid
erDiagram
    MovBajasProgramadasTbl {
        int idMovimiento PK
        int idTipoProceso
        int idMarca
        int idTipoOperacion
        int idProyecto
        int idRelLab
        int idEmpleado
        date fechaProceso
        date FechaBaja
        tinyint idEstatus
        varchar(max) observaciones
        int idUsuarioAct
    }
    
    segDatosAdicionalesTbl {
        idUsuario       int PK
        rfc             varchar(15)
        curp            varchar(20)
        nss             varchar(18)
        claveElector    varchar(20)
        idEmpleado      int
        idRelLab        int
        fechaNacimiento date
        idUsuarioAct    int         
        fechaAct        datetime
        ipAct           varchar(30)
        macAddressAct   varchar(30)
        idMarca         smallint
        idTipoOperacion smallint
        idGrupoPago     smallint
        codigoGrupoPago varchar(100)
        idCompania      smallint
        modoChecado     varchar(100)
        codigoEmpleado  varchar(20)
    }
    
    segUsuariosTbl {
        idUsuario       int
        claveUsuario    varchar(100)
        apellidoPat     varchar(100)
        apellidoMat     varchar(100)
        nombre          varchar(100)
        iniciales       varchar(5)
        password        varchar(250)
        fechaVigencia   datetime    
        idEstatus       bit
        idTipoUsuario   bit
        correo          varchar(100)
        fechaAlta       datetime    
        fechaAct        datetime
        idUsuarioAct    int         
        tokenGoogle     varchar(500)
        correoValidado  bit
        conteoEnvio     tinyint
        ultFechaEnvio   datetime
        cambiarPassword bit    
        permiteRF       bit    
        tieneRegistroRF bit    
    }

    segDatosAdicionalesTbl ||--|| segUsuariosTbl : "relaciona"
```

### 5.2 MovMensajesPubSubTbl

```csharp
{
    int IdMensaje PK,
    int IdTipoMensaje,
    string IdMensaje, // MessageId de Pub/Sub
    string DatosMensaje, // Data decodificado
    DateTime FechaPublicacion,
    string Subscripcion,
    string AtributosMensaje, // JSON serializado
    int IdUsuarioAct
}
```

### 5.3 Envelope (Value Object)

```csharp
public class Envelope
{
    public Message Message { get; set; }
    public string Subscription { get; set; }
}

public class Message
{
    public Dictionary<string, string>? Attributes { get; set; }
    public string MessageId { get; set; }
    public string Data { get; set; } // Base64 encoded
    public DateTime PublishTime { get; set; }
    
    public string EncodingData { get; } // Decoded data
    public T? To<T>() // Deserializa a tipo específico
}
```

---

## 6. Flujo Completo del Sistema

```mermaid
sequenceDiagram
    participant Ext as Sistema Externo RHIN
    participant PS as Pub/Sub
    participant API as UserSubscriptor API
    participant Handler as Command Handlers
    participant Service as UserDeregistrationService
    participant DB as Base de Datos
    participant Scheduler as Cloud Scheduler
    
    Note over Ext,DB: Fase 1: Solicitud de Baja
    Ext->>PS: Publica mensaje de baja
    PS->>API: POST /deregistration
    API->>Handler: UserDeregistrationCommand
    Handler->>DB: Guardar mensaje Pub/Sub
    Handler->>DB: Crear registro en MovBajasProgramadasTbl (RECEIVED)
    alt Fecha baja <= Hoy
        Handler->>Service: ProcessDeregistrationAsync
        Service->>DB: Actualizar a INPROCESS
        Service->>DB: Desactivar operación específica
        alt Usuario sin operaciones activas
            Service->>DB: Desactivar usuario
            Service->>DB: Desactivar tokens
            Service->>DB: Desactivar aplicaciones
            Service->>DB: Actualizar a PATIALLY_PROCESSED
        end
        Service->>PS: Publicar mensaje de éxito
    end
    API-->>PS: 204 No Content
    
    Note over Ext,DB: Fase 2: Procesamiento Diario (Automático)
    Scheduler->>PS: Trigger diario (06:00 AM)
    PS->>API: POST /daily-deregistration
    API->>Handler: DailyUserDeregistrationCommand
    Handler->>DB: Guardar mensaje Pub/Sub
    Handler->>DB: Consultar bajas programadas (RECEIVED, FechaBaja = Hoy)
    loop Para cada usuario
        Handler->>Service: ProcessDeregistrationAsync
        Service->>DB: Procesar baja
    end
    API-->>PS: 204 No Content
    
    Note over Ext,DB: Fase 3: Eliminación Definitiva (Mensual)
    Scheduler->>PS: Trigger mensual (1er día del mes)
    PS->>API: POST /definitive-deregistration
    API->>Handler: DefinitiveDeregistrationCommand
    Handler->>DB: Guardar mensaje Pub/Sub
    Handler->>DB: Consultar usuarios (PATIALLY_PROCESSED, >30 días)
    loop Para cada usuario
        Handler->>DB: Desactivar todos los tokens
        Handler->>DB: Actualizar a DEFINITIVE_DEREGISTRATION
        Handler->>DB: Desactivar todas las aplicaciones
        Handler->>DB: Desactivar usuario
    end
    API-->>PS: 204 No Content
```

---

## 7. Casos de Uso

### 7.1 CU-01: Baja Inmediata de Usuario

**Actor Principal**: Sistema de RHIN  
**Precondiciones**: 
- Usuario existe en el sistema
- Fecha de baja es hoy o anterior

**Flujo Principal**:
1. Sistema externo envía solicitud de baja mediante Pub/Sub
2. Endpoint `/deregistration` recibe el mensaje
3. Sistema registra la baja en `MovBajasProgramadasTbl` con estado `RECEIVED`
4. Sistema detecta que fecha de baja <= fecha actual
5. Sistema procesa la baja inmediatamente
6. Sistema desactiva operación específica (marca + tipo de operación)
7. Si no hay más operaciones activas, desactiva usuario completamente
8. Sistema actualiza estado a `PATIALLY_PROCESSED`
9. Sistema envía confirmación a Pub/Sub

**Postcondiciones**:
- Usuario parcial o totalmente desactivado
- Baja registrada con estado `PATIALLY_PROCESSED`
- Aplicaciones desactivadas excepto Notificaciones y Recibos de Nómina
- Mensaje de confirmación publicado

---

### 7.2 CU-02: Baja Programada Futura

**Actor Principal**: Sistema de RHIN  
**Precondiciones**: 
- Usuario existe en el sistema
- Fecha de baja es futura

**Flujo Principal**:
1. Sistema externo envía solicitud de baja mediante Pub/Sub
2. Endpoint `/deregistration` recibe el mensaje
3. Sistema registra la baja en `MovBajasProgramadasTbl` con estado `RECEIVED`
4. Sistema detecta que fecha de baja > fecha actual
5. Sistema NO procesa la baja inmediatamente
6. Sistema programa la baja para procesamiento futuro
7. Sistema retorna éxito

**Postcondiciones**:
- Baja registrada con estado `RECEIVED`
- Usuario permanece activo hasta la fecha programada
- Baja será procesada por tarea diaria

---

### 7.3 CU-03: Procesamiento Diario Automático

**Actor Principal**: Cloud Scheduler  
**Precondiciones**: 
- Existen bajas programadas con fecha actual
- Bajas están en estado `RECEIVED`

**Flujo Principal**:
1. Cloud Scheduler dispara endpoint `/daily-deregistration` a las 20:00 Hrs
2. Sistema consulta todas las bajas con `FechaBaja = Hoy` y estado `RECEIVED`
3. Para cada usuario encontrado:
   - Procesa la baja mediante `UserDeregistrationService`
   - Desactiva operaciones específicas
   - Si corresponde, desactiva usuario completamente
   - Actualiza estado a `PATIALLY_PROCESSED`
4. Sistema retorna éxito

**Postcondiciones**:
- Todas las bajas del día procesadas
- Usuarios desactivados según corresponda
- Estados actualizados

---

### 7.4 CU-04: Eliminación Definitiva (30 días)

**Actor Principal**: Cloud Scheduler  
**Precondiciones**: 
- Existen usuarios en estado `PATIALLY_PROCESSED`
- Han transcurrido más de 30 días desde la fecha de baja

**Flujo Principal**:
1. Cloud Scheduler dispara endpoint `/definitive-deregistration` (mensual)
2. Sistema consulta usuarios con estado `PATIALLY_PROCESSED` y más de 30 días
3. Para cada usuario encontrado:
   - Desactiva todos los tokens del usuario
   - Actualiza estado a `DEFINITIVE_DEREGISTRATION`
   - Desactiva todas las relaciones con aplicaciones
   - Desactiva usuario definitivamente
4. Sistema retorna éxito

**Postcondiciones**:
- Usuarios eliminados definitivamente del sistema
- Estados actualizados a `DEFINITIVE_DEREGISTRATION`
- Proceso irreversible completado

---

### 7.5 CU-05: Error en Procesamiento

**Actor Principal**: Sistema  
**Precondiciones**: 
- Solicitud de baja recibida

**Flujo Principal**:
1. Sistema inicia procesamiento de baja
2. Ocurre un error:
   - Usuario no encontrado
   - Operación específica no existe
   - Error de base de datos
3. Sistema actualiza registro a estado `ERROR`
4. Sistema registra detalles del error en campo `Observaciones`
5. Sistema continúa con siguiente registro (si aplica)

**Postcondiciones**:
- Baja marcada con estado `ERROR`
- Detalles del error registrados
- Requiere intervención manual

---

## 8. Consideraciones de Seguridad

### 8.1 Validación de Datos
- Validar estructura del Envelope completo
- Validar decodificación de data Base64
- Validar existencia de campos requeridos
- Validar tipos de datos

### 8.2 Trazabilidad
- Todos los mensajes Pub/Sub se registran en `MovMensajesPubSubTbl`
- Estados del proceso registrados con timestamps
- Campo `Observaciones` para información adicional
- Auditoría completa del proceso de baja

### 8.3 Aplicaciones que Permanecen Activas

Durante el proceso de baja parcial, el sistema mantiene activas específicamente las siguientes aplicaciones:

| Aplicación | Razón de Permanencia |
|------------|----------------------|
| **Notificaciones** | Permite al usuario recibir comunicaciones importantes del sistema, incluyendo notificaciones sobre el estado de su baja y otros avisos relevantes |
| **Recibos de Nómina** | Garantiza el acceso continuo al historial de recibos de pago, cumpliendo con obligaciones legales de disponibilidad de información laboral |

**Implementación Técnica:**
- El parámetro global `APPS_TO_KEEP` (ID: 14) contiene los IDs de estas aplicaciones
- Durante el método `UpdateUserApplicationsAsync`, se consultan estos IDs desde `CatParametrosGralesTbl`
- Se desactivan todas las aplicaciones del usuario **excepto** las especificadas en este parámetro
- Esta configuración es flexible y puede modificarse ajustando el parámetro global sin cambios en el código

---

## 9. Configuración y Deployment

### 9.1 Cloud Scheduler Configuration

**Tarea Programada:**
```yaml
description: Tarea encargada de realizar las bajas programadas de los usuarios
name: projects/plowserve/locations/us-central1/jobs/seguridad-usuario-baja
pubsubTarget:
  attributes:
    baja-diaria: ok
    baja-final: ok
  topicName: projects/plowserve/topics/seg-usuario-alta
retryConfig:
  maxBackoffDuration: 3600s
  maxDoublings: 5
  maxRetryDuration: 0s
  minBackoffDuration: 5s
schedule: 0 20 * * 1-5
timeZone: America/Mexico_City
```
---
## 10. Testing

### 10.1 Casos de Prueba

#### Test Case 1: Baja inmediata exitosa
```
Given: Usuario existe con IdEmpleado = 12345
  And: Fecha baja = Hoy
  And: Usuario tiene solo una operación activa
  And: Usuario tiene 5 aplicaciones activas
When: POST /deregistration
Then: Response 204
  And: Estado = PATIALLY_PROCESSED
  And: Usuario desactivado
  And: Tokens desactivados
  And: Aplicaciones desactivadas = 3
  And: Aplicaciones activas = 2 (Notificaciones y Recibos de Nómina)
```

#### Test Case 2: Baja programada futura
```
Given: Usuario existe con IdEmpleado = 12345
  And: Fecha baja = +7 días
When: POST /deregistration
Then: Response 204
  And: Estado = RECEIVED
  And: Usuario permanece activo
```

#### Test Case 3: Procesamiento diario batch
```
Given: Existen 5 bajas programadas para hoy
  And: Todas en estado RECEIVED
When: POST /daily-deregistration
Then: Response 204
  And: Todas las bajas procesadas
  And: Todos los estados = PATIALLY_PROCESSED o ERROR
```

#### Test Case 4: Eliminación definitiva
```
Given: Usuario en estado PATIALLY_PROCESSED
  And: Fecha baja hace 35 días
When: POST /definitive-deregistration
Then: Response 204
  And: Estado = DEFINITIVE_DEREGISTRATION
  And: Todos los tokens desactivados
  And: Todas las aplicaciones desactivadas
```

#### Test Case 5: Error - Usuario no existe
```
Given: IdEmpleado = 99999 no existe
When: POST /deregistration
Then: Response 204 (procesado)
  And: Estado = ERROR
  And: Observaciones = "No se encontró la información del usuario."
```
---

## 11. Dependencias

### 11.1 Paquetes NuGet
- MediatR (para CQRS)
- AutoMapper (para mapeo de DTOs)
- Google.Cloud.PubSub.V1 (integración Pub/Sub)
- Microsoft.EntityFrameworkCore (acceso a datos)
- Newtonsoft.Json (serialización)

### 11.2 Servicios Externos
- Google Cloud Pub/Sub
- Google Cloud Scheduler
- Base de datos SQL Server

### 11.3 Repositorios Requeridos
- `IUserDeregistrationService`
- `IScheduledDeregistrationRepository`
- `IAdditionalDataRepository`
- `IRelUsuariosTokenRepository`
- `IUserRepository`
- `IApplicationRepository`
- `IPubSubRepository`
- `IUserOperationService`

---

## 12. Criterios de Aceptación

### 12.1 Funcionales
- ✅ Endpoint `/deregistration` registra y procesa bajas correctamente
- ✅ Endpoint `/daily-deregistration` procesa todas las bajas del día
- ✅ Endpoint `/definitive-deregistration` elimina usuarios después de 30 días
- ✅ Estados del proceso se actualizan correctamente
- ✅ Mensajes Pub/Sub se registran para auditoría
- ✅ Errores se manejan y registran apropiadamente
- ✅ Desactivación parcial funciona por operación específica
- ✅ Desactivación total funciona cuando no hay operaciones activas

### 12.2 No Funcionales
- ✅ Tiempo de respuesta < 5 segundos por usuario
- ✅ Procesamiento batch de hasta 1000 usuarios
- ✅ Tasa de error < 5%
- ✅ Disponibilidad 99.9%
- ✅ Logs completos de todas las operaciones

---

**Documento Versión**: 1.0  
**Fecha**: 15 de enero de 2026  
**Autor**: Equipo de Desarrollo - Seguridad Corporativa  
**Aprobado por**: [Engineering Manager]
