# Documentación del Job de Correos Pendientes

## Descripción General

Sistema automatizado de procesamiento y envío de notificaciones (correos electrónicos y notificaciones push) basado en un modelo de cola de procesos programados. El job principal invoca al procedimiento `Spp_CorreosPendientes` que gestiona la ejecución secuencial de procesos pendientes.

---

## Arquitectura del Sistema

### Flujo Principal

```
┌─────────────────────────────────────────────────────────────┐
│                        SQL Server Job                        │
│                  (Ejecución Programada)                      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Spp_CorreosPendientes                          │
│  • Lee parámetros de configuración                          │
│  • Verifica procesos en ejecución                           │
│  • Recupera procesos pendientes (estado = 1)                │
│  • Controla concurrencia mediante movProcesoPasoTbl         │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Spp_conProcesosTbl                             │
│  • Valida el proceso                                         │
│  • Obtiene procedimiento asociado al motivo                 │
│  • Ejecuta dinámicamente el procedimiento específico        │
│  • Actualiza estado del proceso (3=OK, 4=Error)            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│        Procedimiento Específico del Motivo                  │
│  (Ejemplo: Spp_EnvioCorreoBienvenida,                       │
│            Spp_NotificacionAsistencia, etc.)                │
│  • Genera contenido del correo/notificación                 │
│  • Realiza el envío                                          │
│  • Retorna estatus de ejecución                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Componentes del Sistema

### 1. Procedimiento Principal: `Spp_CorreosPendientes`

**Ubicación:** `/BakNotificacionesGit/Stored Procedures/Spp_CorreosPendientes.sql`

**Propósito:** Orquestar el procesamiento de correos/notificaciones pendientes con control de concurrencia y límites de ejecución.

#### Parámetros

| Parámetro | Tipo | Dirección | Descripción |
|-----------|------|-----------|-------------|
| `@PnIdUsuarioAct` | Integer | Input | ID del usuario que ejecuta el proceso |
| `@PnEstatus` | Integer | Output | Código de estatus de ejecución (0 = Éxito) |
| `@PsMensaje` | Varchar(250) | Output | Mensaje descriptivo del resultado |

#### Configuración del Sistema

Lee dos parámetros críticos de `conParametrosGralesTbl`:

- **ID 14**: `@w_CorreosMaximos` - Número máximo de procesos a ejecutar por ciclo (default: 1)
- **ID 15**: `@w_minutosEspera` - Tiempo de espera en minutos para considerar un proceso como huérfano (default: 1)

#### Lógica de Procesamiento

1. **Control de Procesos Huérfanos**
   ```sql
   -- Si existen procesos en movProcesoPasoTbl con más de @w_minutosEspera minutos
   -- Se trunca la tabla y se resetean procesos en estado 2 (En Proceso) a 1 (Pendiente)
   ```

2. **Verificación de Concurrencia**
   ```sql
   -- Si existe algún registro en movProcesoPasoTbl, sale sin procesar
   -- Esto previene ejecuciones concurrentes
   ```

3. **Selección de Procesos**
   ```sql
   -- Inserta en movProcesoPasoTbl hasta @w_CorreosMaximos procesos
   -- Condiciones:
   --   - idEstatus = 1 (Pendiente)
   --   - fechaProgramada <= fecha actual
   -- Ordenamiento: Por fechaProgramada ASC (primero los más antiguos)
   ```

4. **Procesamiento con Cursor**
   ```sql
   -- Recorre cada proceso en movProcesoPasoTbl (orden DESC por fechaProgramada)
   -- Para cada proceso:
   --   1. Actualiza estado a 2 (En Proceso)
   --   2. Ejecuta Spp_conProcesosTbl
   --   3. El estatus final lo maneja Spp_conProcesosTbl
   ```

5. **Limpieza**
   ```sql
   -- Trunca movProcesoPasoTbl al finalizar el ciclo
   ```

#### Diagrama de Estados de Proceso

```
┌─────────────┐
│ 1: Pendiente│◄─── Inserción inicial o reset de proceso huérfano
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ 2: En Proceso│◄── Spp_CorreosPendientes actualiza antes de ejecutar
└──────┬──────┘
       │
       ├──────┐
       │      │
       ▼      ▼
┌──────────┐ ┌──────────────┐
│3: OK     │ │4: Error      │◄── Spp_conProcesosTbl actualiza según resultado
└──────────┘ └──────────────┘
```

---

### 2. Procedimiento Ejecutor: `Spp_conProcesosTbl`

**Ubicación:** `/BakNotificacionesGit/Stored Procedures/Spp_conProcesosTbl.sql`

**Propósito:** Ejecutar dinámicamente el procedimiento específico asociado a un motivo de correo/notificación.

#### Parámetros

| Parámetro | Tipo | Dirección | Descripción |
|-----------|------|-----------|-------------|
| `@PnIdProceso` | Integer | Input | ID único del proceso en `conProcesosTbl` |
| `@PnIdUsuarioAct` | Int | Input | ID del usuario que ejecuta |
| `@PnEstatus` | Integer | Output | Código de estatus (0 = Éxito, 2007 = Error) |
| `@PsMensaje` | Varchar(250) | Output | Mensaje de resultado |

#### Flujo de Validación y Ejecución

1. **Obtención de Información del Proceso**
   ```sql
   -- Lee de conProcesosTbl:
   --   - idMotivoCorreo (tipo de notificación)
   --   - idEstatus (estado actual)
   --   - parametros (parámetros específicos del proceso)
   --   - idUsuario (usuario asociado al proceso)
   --   - fechaProgramada
   ```

2. **Validación de Procedimiento**
   ```sql
   -- Busca el procedimiento en conProcedimientosCorreoTbl usando idMotivoCorreo
   -- Verifica existencia del procedimiento en sysobjects (Type = 'P', Uid = 1)
   ```

3. **Ejecución Dinámica**
   ```sql
   EXECUTE @w_procedimiento 
      @PnIdProceso    = @PnIdProceso,
      @PnIdUsuarioAct = @w_idUsuario,
      @PnEstatus      = @PnEstatus Output,
      @PsMensaje      = @PsMensaje Output
   ```

4. **Manejo de Errores**
   ```sql
   -- Si @PnEstatus != 0:
   --   - Actualiza conProcesosTbl con idEstatus = 4 (Error)
   --   - Guarda el mensaje de error en mensajeProc
   --   - Actualiza fechaAct
   ```

#### Códigos de Error

| Código | Descripción |
|--------|-------------|
| 0 | Ejecución exitosa |
| 2007 | Error de validación (proceso inválido, procedimiento no existe, etc.) |

---

### 3. Tablas del Sistema

#### 3.1. `conProcesosTbl` - Tabla Principal de Procesos

**Propósito:** Almacena la cola de procesos de notificación programados.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `idProceso` | INT IDENTITY | PK - Identificador único |
| `idMotivoCorreo` | SMALLINT | FK - Tipo de notificación (referencia a conMotivosCorreoTbl) |
| `idTipoNotificacion` | TINYINT | 0=Push, 1=Correo, 2=Ambos |
| `idUsuario` | SMALLINT | Usuario que genera la acción |
| `parametros` | VARCHAR(8000) | Parámetros en formato JSON/XML para el procedimiento |
| `fechaProgramada` | DATETIME | Fecha/hora programada de envío |
| `fechaInicio` | DATETIME | Fecha/hora de inicio de procesamiento |
| `fechaTermino` | DATETIME | Fecha/hora de finalización |
| `tiempoProceso` | NUMERIC(18,6) | Duración en segundos |
| `registrosProcesados` | INT | Cantidad de registros procesados |
| `urlArchivoSalida` | VARCHAR(200) | Ruta del archivo generado (si aplica) |
| `mensajeProc` | VARCHAR(1000) | Mensaje de resultado/error |
| `tablaSalida` | SYSNAME | Nombre de tabla temporal con resultados |
| `idSession` | VARCHAR(60) | ID de sesión para agrupación |
| `idEstatus` | TINYINT | **1**=Pendiente, **2**=En Proceso, **3**=OK, **4**=Error |
| `idUsuarioAct` | SMALLINT | Usuario última actualización |
| `fechaAct` | DATETIME | Fecha última actualización |

**Índices:**
- PK: `idProceso`
- IDX: `idSession`

---

#### 3.2. `conProcedimientosCorreoTbl` - Catálogo de Procedimientos

**Propósito:** Mapea motivos de correo a procedimientos almacenados específicos.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `idMotivoCorreo` | SMALLINT | PK - FK a conMotivosCorreoTbl |
| `idProcedimiento` | SMALLINT | PK - Secuencial por motivo |
| `codigoProcedimiento` | VARCHAR(50) | Código único del procedimiento |
| `procedimiento` | SYSNAME | Nombre del procedimiento almacenado |
| `idUsuarioAct` | SMALLINT | Usuario última actualización |
| `fechaAct` | DATETIME | Fecha última actualización |
| `ipAct` | VARCHAR(30) | IP de última actualización |
| `macAddressAct` | VARCHAR(30) | MAC address de última actualización |

**Trigger:** `TrinsconProcedimientosCorreoTbl`
- Valida unicidad de `codigoProcedimiento`
- Verifica existencia del procedimiento en la base de datos
- Auto-completa `ipAct` y `macAddressAct` si no se proporcionan
- Impide modificación de `idProcedimiento` en updates

**Índices:**
- PK: `(idMotivoCorreo, idProcedimiento)`
- IDX: `codigoProcedimiento`

---

#### 3.3. `movProcesoPasoTbl` - Tabla de Control Temporal

**Propósito:** Tabla de paso para control de concurrencia y selección de procesos.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `idProceso` | INT | PK - ID del proceso en ejecución |
| `fechaProgramada` | DATETIME | Fecha programada (copiada de conProcesosTbl) |
| `fechaAct` | DATETIME | Timestamp de inserción (default: GETDATE()) |

**Uso:**
- Se trunca al inicio de cada ciclo si hay procesos huérfanos
- Se llena con procesos a ejecutar en el ciclo actual
- Se trunca al finalizar el ciclo
- Actúa como semáforo: si tiene registros, no se ejecuta otro ciclo

---

#### 3.4. `conParametrosGralesTbl` - Configuración del Sistema

**Propósito:** Almacena parámetros de configuración global.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `idParametroGral` | SMALLINT | PK - Código de parámetro |
| `descripcion` | VARCHAR(100) | Descripción del parámetro |
| `parametroChar` | VARCHAR(150) | Valor tipo texto |
| `parametroNumber` | NUMERIC(24,6) | Valor tipo numérico |
| `parametroFecha` | DATETIME | Valor tipo fecha |
| `idUsuarioAct` | SMALLINT | Usuario última actualización |
| `fechaAct` | DATETIME | Fecha última actualización |
| `ipAct` | VARCHAR(30) | IP de última actualización |
| `macAddressAct` | VARCHAR(30) | MAC de última actualización |

**Parámetros Clave para el Job:**

| ID | Descripción | Campo Usado |
|----|-------------|-------------|
| 14 | Número máximo de correos por ciclo | `parametroNumber` |
| 15 | Minutos de espera para proceso huérfano | `parametroNumber` |

---

## Ejemplo de Ejecución

### 1. Inserción de un Proceso

```sql
-- Programar envío de correo de bienvenida
INSERT INTO conProcesosTbl (
    idMotivoCorreo,
    idTipoNotificacion,
    idUsuario,
    parametros,
    fechaProgramada,
    idEstatus,
    idUsuarioAct,
    fechaAct
)
VALUES (
    5,                                    -- Motivo: Correo de Bienvenida
    1,                                    -- Tipo: Solo Correo
    12345,                                -- Usuario solicitante
    '{"idEmpleado": 98765, "nombre": "Juan Pérez"}',
    '2025-11-13 14:30:00',               -- Envío programado
    1,                                    -- Estado: Pendiente
    1,                                    -- Usuario sistema
    GETDATE()
);
```

### 2. Ejecución del Job

```sql
-- Llamada desde SQL Server Agent Job
DECLARE @PnEstatus INT;
DECLARE @PsMensaje VARCHAR(250);

EXEC Spp_CorreosPendientes 
    @PnIdUsuarioAct = 1,
    @PnEstatus = @PnEstatus OUTPUT,
    @PsMensaje = @PsMensaje OUTPUT;

SELECT @PnEstatus AS Estatus, @PsMensaje AS Mensaje;
```

### 3. Flujo de Procesamiento

```sql
-- 1. Spp_CorreosPendientes lee configuración
SELECT parametroNumber FROM conParametrosGralesTbl 
WHERE idParametroGral IN (14, 15);
-- Resultado: @w_CorreosMaximos = 5, @w_minutosEspera = 2

-- 2. Verifica procesos huérfanos
SELECT * FROM movProcesoPasoTbl 
WHERE DATEDIFF(mi, fechaAct, GETDATE()) >= 2;
-- Si existen: TRUNCATE y reset de estados

-- 3. Selección de procesos (máximo 5)
INSERT INTO movProcesoPasoTbl (idProceso, fechaProgramada)
SELECT TOP 5 idProceso, fechaProgramada
FROM conProcesosTbl
WHERE idEstatus = 1 
  AND fechaProgramada <= GETDATE()
ORDER BY fechaProgramada;

-- 4. Actualiza estado a "En Proceso"
UPDATE conProcesosTbl
SET idEstatus = 2, fechaInicio = GETDATE(), mensajeProc = 'En Proceso'
WHERE idProceso = @w_idProceso;

-- 5. Obtiene procedimiento a ejecutar
SELECT p.procedimiento
FROM conProcesosTbl c
JOIN conProcedimientosCorreoTbl p ON c.idMotivoCorreo = p.idMotivoCorreo
WHERE c.idProceso = @w_idProceso;
-- Resultado: 'Spp_EnvioCorreoBienvenida'

-- 6. Ejecución dinámica
EXECUTE Spp_EnvioCorreoBienvenida 
    @PnIdProceso = @w_idProceso,
    @PnIdUsuarioAct = 12345,
    @PnEstatus = @PnEstatus OUTPUT,
    @PsMensaje = @PsMensaje OUTPUT;

-- 7. Actualiza resultado final
UPDATE conProcesosTbl
SET idEstatus = CASE WHEN @PnEstatus = 0 THEN 3 ELSE 4 END,
    mensajeProc = @PsMensaje,
    fechaAct = GETDATE()
WHERE idProceso = @w_idProceso;
```

---

## Configuración del SQL Server Agent Job

### Configuración Recomendada

```sql
-- Nombre del Job
USE msdb;
GO

EXEC sp_add_job
    @job_name = N'NOTIFICACIONES - Procesamiento de Correos Pendientes',
    @enabled = 1,
    @description = N'Procesa cola de correos y notificaciones programadas';

-- Paso de ejecución
EXEC sp_add_jobstep
    @job_name = N'NOTIFICACIONES - Procesamiento de Correos Pendientes',
    @step_name = N'Ejecutar Spp_CorreosPendientes',
    @subsystem = N'TSQL',
    @database_name = N'NCTI',
    @command = N'
        DECLARE @PnEstatus INT;
        DECLARE @PsMensaje VARCHAR(250);

        EXEC Spp_CorreosPendientes 
            @PnIdUsuarioAct = 1,
            @PnEstatus = @PnEstatus OUTPUT,
            @PsMensaje = @PsMensaje OUTPUT;

        IF @PnEstatus != 0
        BEGIN
            RAISERROR(@PsMensaje, 16, 1);
        END',
    @on_success_action = 1,  -- Quit with success
    @on_fail_action = 2;     -- Quit with failure

-- Programación (cada 5 minutos)
EXEC sp_add_jobschedule
    @job_name = N'NOTIFICACIONES - Procesamiento de Correos Pendientes',
    @name = N'Cada_5_Minutos',
    @freq_type = 4,           -- Daily
    @freq_interval = 1,       -- Every day
    @freq_subday_type = 4,    -- Minutes
    @freq_subday_interval = 5, -- Every 5 minutes
    @active_start_time = 0,   -- 00:00:00
    @active_end_time = 235959; -- 23:59:59

-- Asignar a servidor local
EXEC sp_add_jobserver
    @job_name = N'NOTIFICACIONES - Procesamiento de Correos Pendientes',
    @server_name = N'(local)';
GO
```

---

## Manejo de Errores y Recuperación

### Procesos Huérfanos

**Problema:** Un proceso queda en estado 2 (En Proceso) indefinidamente por:
- Falla del servidor
- Timeout de conexión
- Cancelación manual del job

**Solución Automática:**
```sql
-- Ejecutado al inicio de cada ciclo en Spp_CorreosPendientes
IF EXISTS (
    SELECT TOP 1 1
    FROM movProcesoPasoTbl
    WHERE DATEDIFF(mi, fechaAct, GETDATE()) >= @w_minutosEspera
)
BEGIN
    TRUNCATE TABLE movProcesoPasoTbl;
    
    UPDATE conProcesosTbl
    SET idEstatus = 1,      -- Reset a Pendiente
        fechaInicio = NULL,
        fechaAct = GETDATE()
    WHERE idEstatus = 2;    -- Todos los "En Proceso"
END
```

### Reintentos

Para reintentar un proceso fallido:

```sql
-- Opción 1: Reprogramar manualmente
UPDATE conProcesosTbl
SET idEstatus = 1,              -- Pendiente
    fechaProgramada = GETDATE(), -- Procesar inmediatamente
    mensajeProc = 'Reintento manual',
    fechaAct = GETDATE()
WHERE idProceso = 12345;

-- Opción 2: Clonar el proceso
INSERT INTO conProcesosTbl (
    idMotivoCorreo, idTipoNotificacion, idUsuario,
    parametros, fechaProgramada, idEstatus, 
    idUsuarioAct, fechaAct
)
SELECT 
    idMotivoCorreo, idTipoNotificacion, idUsuario,
    parametros, GETDATE(), 1, -- Nuevo proceso pendiente
    idUsuarioAct, GETDATE()
FROM conProcesosTbl
WHERE idProceso = 12345;
```

---

## Monitoreo y Diagnóstico

### Queries de Monitoreo

#### 1. Procesos Pendientes
```sql
SELECT 
    p.idProceso,
    p.idMotivoCorreo,
    m.descripcion AS Motivo,
    p.fechaProgramada,
    DATEDIFF(mi, p.fechaProgramada, GETDATE()) AS MinutosRetrasados,
    p.parametros
FROM conProcesosTbl p
JOIN conMotivosCorreoTbl m ON p.idMotivoCorreo = m.idMotivo
WHERE p.idEstatus = 1
  AND p.fechaProgramada <= GETDATE()
ORDER BY p.fechaProgramada;
```

#### 2. Procesos En Ejecución
```sql
SELECT 
    p.idProceso,
    m.descripcion AS Motivo,
    p.fechaInicio,
    DATEDIFF(s, p.fechaInicio, GETDATE()) AS SegundosEjecucion,
    p.mensajeProc
FROM conProcesosTbl p
JOIN conMotivosCorreoTbl m ON p.idMotivoCorreo = m.idMotivo
WHERE p.idEstatus = 2;
```

#### 3. Procesos Fallidos (últimas 24 horas)
```sql
SELECT 
    p.idProceso,
    m.descripcion AS Motivo,
    p.fechaInicio,
    p.fechaTermino,
    p.mensajeProc,
    p.parametros
FROM conProcesosTbl p
JOIN conMotivosCorreoTbl m ON p.idMotivoCorreo = m.idMotivo
WHERE p.idEstatus = 4
  AND p.fechaAct >= DATEADD(hh, -24, GETDATE())
ORDER BY p.fechaAct DESC;
```

#### 4. Estadísticas de Rendimiento
```sql
SELECT 
    m.descripcion AS Motivo,
    COUNT(*) AS TotalProcesos,
    SUM(CASE WHEN p.idEstatus = 3 THEN 1 ELSE 0 END) AS Exitosos,
    SUM(CASE WHEN p.idEstatus = 4 THEN 1 ELSE 0 END) AS Fallidos,
    AVG(p.tiempoProceso) AS TiempoPromedioSeg,
    MAX(p.tiempoProceso) AS TiempoMaximoSeg,
    SUM(p.registrosProcesados) AS TotalRegistros
FROM conProcesosTbl p
JOIN conMotivosCorreoTbl m ON p.idMotivoCorreo = m.idMotivo
WHERE p.fechaAct >= DATEADD(dd, -7, GETDATE())
GROUP BY m.descripcion
ORDER BY TotalProcesos DESC;
```

#### 5. Estado del Job
```sql
-- Verificar última ejecución del job
SELECT 
    j.name AS JobName,
    h.run_date,
    h.run_time,
    h.run_status,  -- 0=Failed, 1=Succeeded, 2=Retry, 3=Canceled
    h.message
FROM msdb.dbo.sysjobs j
JOIN msdb.dbo.sysjobhistory h ON j.job_id = h.job_id
WHERE j.name LIKE '%Correos Pendientes%'
  AND h.step_id = 0  -- 0 = Estado final del job
ORDER BY h.run_date DESC, h.run_time DESC;
```

---

## Mejores Prácticas

### 1. Configuración de Parámetros

- **@w_CorreosMaximos (ID 14):**
  - Entornos de bajo volumen: 1-5
  - Entornos de alto volumen: 10-20
  - Considerar capacidad del servidor de correo

- **@w_minutosEspera (ID 15):**
  - Procesos rápidos (< 1 min): 2-3 minutos
  - Procesos lentos (> 5 min): 10-15 minutos

### 2. Frecuencia del Job

- Alta prioridad: 1-5 minutos
- Prioridad normal: 5-15 minutos
- Baja prioridad: 30-60 minutos

### 3. Gestión de Parámetros

```sql
-- Almacenar parámetros complejos como JSON
INSERT INTO conProcesosTbl (parametros, ...)
VALUES (
    '{
        "idEmpleado": 12345,
        "tipoReporte": "mensual",
        "fechaInicio": "2025-11-01",
        "fechaFin": "2025-11-30",
        "correoAdicional": ["jefe@empresa.com"]
    }',
    ...
);
```

### 4. Logging Adicional

Considerar crear tabla de log detallado:

```sql
CREATE TABLE logProcesoDetalladoTbl (
    idLog INT IDENTITY PRIMARY KEY,
    idProceso INT NOT NULL,
    fechaLog DATETIME DEFAULT GETDATE(),
    nivel VARCHAR(20),  -- INFO, WARNING, ERROR
    mensaje VARCHAR(MAX),
    datosAdicionales VARCHAR(MAX)
);
```

---

## Diagramas de Secuencia

### Secuencia Completa de Ejecución

```
Job              Spp_CorreosPendientes    Spp_conProcesosTbl    Proc. Específico    Base de Datos
 │                        │                        │                     │                 │
 │──Ejecutar cada 5 min──>│                        │                     │                 │
 │                        │                        │                     │                 │
 │                        │──Leer Config (14,15)──>│                     │                 │
 │                        │<──────────────────────│                     │                 │
 │                        │                        │                     │                 │
 │                        │──Verificar Huérfanos──>│                     │                 │
 │                        │<──────────────────────│                     │                 │
 │                        │                        │                     │                 │
 │                        │──Seleccionar Procesos─>│                     │                 │
 │                        │<──────────────────────│                     │                 │
 │                        │                        │                     │                 │
 │                        │──Foreach Proceso       │                     │                 │
 │                        │  │                     │                     │                 │
 │                        │  │─Update Estado=2────>│                     │                 │
 │                        │  │<───────────────────│                     │                 │
 │                        │  │                     │                     │                 │
 │                        │  │──Ejecutar──────────>│                     │                 │
 │                        │  │                     │──Obtener Motivo────>│                 │
 │                        │  │                     │<───────────────────│                 │
 │                        │  │                     │                     │                 │
 │                        │  │                     │──Validar Procedimiento────────────────>│
 │                        │  │                     │<───────────────────────────────────────│
 │                        │  │                     │                     │                 │
 │                        │  │                     │──EXEC Dinámico─────>│                 │
 │                        │  │                     │                     │──Generar Correo>│
 │                        │  │                     │                     │<────────────────│
 │                        │  │                     │                     │                 │
 │                        │  │                     │                     │──Enviar Correo─>│
 │                        │  │                     │                     │<────────────────│
 │                        │  │                     │                     │                 │
 │                        │  │                     │<──Return @PnEstatus──│                 │
 │                        │  │                     │                     │                 │
 │                        │  │                     │──Update Final──────────────────────────>│
 │                        │  │                     │  (Estado 3 o 4)     │                 │
 │                        │  │                     │<───────────────────────────────────────│
 │                        │  │<────Return──────────│                     │                 │
 │                        │  │                     │                     │                 │
 │                        │──End Foreach           │                     │                 │
 │                        │                        │                     │                 │
 │                        │──Truncate Paso Tbl────>│                     │                 │
 │                        │<──────────────────────│                     │                 │
 │<────Return Estatus─────│                        │                     │                 │
 │                        │                        │                     │                 │
```

---

## Anexos

### A. Estructura de Parámetros Comunes

```json
{
    "idEmpleado": 12345,
    "nombre": "Juan Pérez",
    "correo": "juan.perez@empresa.com",
    "parametrosAdicionales": {
        "asunto": "Bienvenido al Sistema",
        "plantilla": "PLANTILLA_BIENVENIDA",
        "adjuntos": ["manual_usuario.pdf"]
    }
}
```

### B. Códigos de Tipo de Notificación

| Código | Descripción |
|--------|-------------|
| 0 | Solo Notificación Push |
| 1 | Solo Correo Electrónico |
| 2 | Ambos (Push + Correo) |

### C. Procedimientos Específicos Típicos

- `Spp_EnvioCorreoBienvenida` - Correo de bienvenida a nuevos usuarios
- `Spp_NotificacionAsistencia` - Notificación de asistencia/falta
- `Spp_ReporteMensual` - Generación y envío de reportes mensuales
- `Spp_AlertaVencimiento` - Alertas de documentos o credenciales próximos a vencer
- `Spp_ConfirmacionPago` - Confirmación de transacciones de pago

---

## Historial de Versiones

| Versión | Fecha | Autor | Cambios |
|---------|-------|-------|---------|
| 2 | 2024-01-30 | Edgar R Rodriguez | Cambio en orden de recuperación de registros |
| 2 | 2024-11-13 | Elizabeth Fernandez | Actualización de tipo de dato @PnIdUsuarioAct de SMALLINT a INT |

---

## Contacto y Soporte

Para dudas o soporte sobre este sistema, contactar al equipo de Base de Datos.

**Ubicación de Scripts:**
- Base: `/BakNotificacionesGit/`
- Procedimientos: `/Stored Procedures/`
- Tablas: `/Tables/`
- Jobs: `/Jobs/`

---

**Documento generado:** 13 de noviembre de 2025  
**Sistema:** NCTI - Notificaciones Corporativas  
**Base de Datos:** NCTI (SQL Server)
