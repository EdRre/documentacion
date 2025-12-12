# Documentación de Jobs SQL Server

## Actualiza Datos E-learning

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | Actualiza Datos E-learning |
| **Estado** | ❌ Deshabilitado |
| **Owner** | sa |
| **Base de Datos** | NCTI |
| **Descripción** | No description available. |

### Pasos del Job

#### Step 1: Actualiza Datos E-learnig

- **Subsistema:** T-SQL
- **Base de Datos:** NCTI
- **Comando:**
```sql
dbo.Spu_MasivoUsuariosElearning
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 9 minutos 12 segundos

---

## AnalisisSemanalBD

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | AnalisisSemanalBD |
| **Estado** | ✅ Activo |
| **Owner** | sa |
| **Base de Datos** | SMBDTI |
| **Descripción** | Proceso semanal de Mantenimiento de Bases de Datos |

### Propósito

Proceso automatizado semanal que ejecuta tareas de mantenimiento, análisis y optimización de bases de datos, incluyendo:
- Ejecución de mantenimiento de índices
- Análisis de ejecución de procedimientos almacenados
- Cálculo de espacio libre en discos
- Generación de notificaciones sobre el estado del sistema

### Pasos del Job

#### Step 1: Paso 1

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PnEstatus             Integer      = 0,
    @PsMensaje             Varchar(250) = Null;

Begin
    Execute Spp_EjecutaMantenBD_1 @PnEstatus = @PnEstatus Output,
                                  @PsMensaje = @PsMensaje Output
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 7 segundos

**Acción en éxito:** Ir al paso 3
**Acción en fallo:** Ir al paso 3

---

#### Step 2: Paso 2

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PsDbName              Sysname      = Null,
    @PnEstatus             Integer      = 0,
    @PsMensaje             Varchar(250) = Null

Begin
    Execute dbo.Spp_EjecutaMantenIndices @PsDbName  = @PsDbName,
                                         @PnEstatus = @PnEstatus Output,
                                         @PsMensaje = @PsMensaje Output
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 minuto 43 segundos

**Acción en éxito:** Ir al paso 3
**Acción en fallo:** Ir al paso 3

---

#### Step 3: Solicitud de Notificaciones

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PdFechaProceso        Date         = Null,
    @PsIdProceso           Varchar(250) = '1, 2',
    @PnEstatus             Integer      = 0,
    @PsMensaje             Varchar(250) = Null;

Begin
    Execute Spp_SolicitaCorreoMasivoMantBD @PdFechaProceso = @PdFechaProceso,
                                           @PsIdProceso    = @PsIdProceso,
                                           @PnEstatus      = @PnEstatus Output,
                                           @PsMensaje      = @PsMensaje Output
    
    Select @PnEstatus, @PsMensaje
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ✅ Éxito
- Duración: 1 segundo

**Acción en éxito:** Ir al paso 3
**Acción en fallo:** Ir al paso 3

---

#### Step 4: Análiza Ejecución Procedimientos

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PsBaseDatos             Sysname      = 'RSTI',
    @PnEstatus               Integer      = 0,
    @PsMensaje               Varchar(250);

Begin
    Execute Spp_analizaEjecucionProcedimientos @PsBaseDatos = @PsBaseDatos,
                                               @PnEstatus   = @PnEstatus Output,
                                               @PsMensaje   = @PsMensaje Output;
    
    If @PnEstatus != 0
    Begin
        Select @PnEstatus As Estatus, @PsMensaje As Mensaje
    End
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

**Acción en éxito:** Ir al paso 3
**Acción en fallo:** Ir al paso 3

---

#### Step 5: Calcula Espacio Libre DD

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PnEstatus               Integer      = 0,
    @PsMensaje               Varchar(250) = Null

Begin
    Execute dbo.Spp_calculaEspacioDD @PnEstatus = @PnEstatus Output,
                                     @PsMensaje = @PsMensaje Output;
    
    Select @PnEstatus, @PsMensaje
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

**Acción en éxito:** Ir al paso 3
**Acción en fallo:** Ir al paso 3

---

#### Step 6: NotificacionAnalisiProc

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PdFechaProc             Date         = Null,
    @PnEstatus               Integer      = 0,
    @PsMensaje               Varchar(250) = Null;

Begin
    Execute dbo.Spp_solicitaNotificacionAnalisiProc @PdFechaProc = @PdFechaProc,
                                                    @PnEstatus   = @PnEstatus Output,
                                                    @PsMensaje   = @PsMensaje Output;
    
    If @PnEstatus != 0
    Begin
        Select @PnEstatus As Estatus, @PsMensaje As Mensaje
    End
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

**Acción en éxito:** Salir reportando éxito
**Acción en fallo:** Salir reportando fallo

---

## Asistencia_Staff

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | Asistencia_Staff |
| **Estado** | ✅ Activo |
| **Owner** | SEGURIDAD-CORPO\Administrator |
| **Base de Datos** | CTRLAsistencia |
| **Descripción** | No description available. |

### Pasos del Job

#### Step 1: Proceso Actualización Asistencia

- **Subsistema:** T-SQL
- **Base de Datos:** CTRLAsistencia
- **Comando:**
```sql
Declare 
    @PnidMarca                        Smallint     = 5,
    @PnEstatus                        Integer      = 0,
    @PsMensaje                        Varchar(850) = ' '

Begin
    Execute prp_ProcAutomaticoControlAsistencia 
        @PnidMarca = @PnidMarca,
        @PnEstatus = @PnEstatus Output,
        @PsMensaje = @PsMensaje Output
End
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 54 segundos

---

## bkp_SIAN_GASTOS_Seguridad

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | bkp_SIAN_GASTOS_Seguridad |
| **Estado** | ❌ Deshabilitado |
| **Owner** | sa |
| **Base de Datos** | SIAN_GASTOS_Seguridad |
| **Descripción** | No description available. |

### Propósito

Job de respaldo automático de la base de datos SIAN_GASTOS_Seguridad.

### Pasos del Job

#### Step 1: dump_bkp_sian_gastos_seguridad

- **Subsistema:** T-SQL
- **Base de Datos:** SIAN_GASTOS_Seguridad
- **Comando:**
```sql
DECLARE @path VARCHAR(500)
DECLARE @name VARCHAR(500)
DECLARE @pathwithname VARCHAR(500)
DECLARE @time DATETIME
DECLARE @year VARCHAR(4)
DECLARE @month VARCHAR(2)
DECLARE @day VARCHAR(2)
DECLARE @hour VARCHAR(2)
DECLARE @minute VARCHAR(2)
DECLARE @second VARCHAR(2)

SET @path = 'C:\sysadmin\local\sian_gastos_seguridad\'

SELECT @time   = GETDATE()
SELECT @year   = (SELECT CONVERT(VARCHAR(4), DATEPART(yy, @time)))
SELECT @month  = (SELECT CONVERT(VARCHAR(2), FORMAT(DATEPART(mm,@time),'00')))
SELECT @day    = (SELECT CONVERT(VARCHAR(2), FORMAT(DATEPART(dd,@time),'00')))

SELECT @name ='SIAN_GASTOS_Seguridad' + '_' + @day + @month + @year
SET @pathwithname = @path + @namE + '.bak'

BACKUP DATABASE [SIAN_GASTOS_seguridad] 
TO DISK = @pathwithname 
WITH NOFORMAT, NOINIT, SKIP, REWIND, NOUNLOAD, STATS = 10
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 2 segundos

---

## Campañas Leidas

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | Campañas Leidas |
| **Estado** | ❌ Deshabilitado |
| **Owner** | sa |
| **Base de Datos** | SCCTI |
| **Descripción** | Acualizacion de Estatus para push de notificaciones de Campañas Elearning |

### Pasos del Job

#### Step 1: Estatus campañas

- **Subsistema:** T-SQL
- **Base de Datos:** SCCTI
- **Comando:**
```sql
Execute dbo.Spp_NotificacionesEviadas
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 3 segundos

---

## Campañas Reporteador

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | Campañas Reporteador |
| **Estado** | ❌ Deshabilitado |
| **Owner** | sa |
| **Base de Datos** | SCCTI |
| **Descripción** | monitorea tablas de repoteador apra generar campañas automaticas |

### Pasos del Job

#### Step 1: Campañas Reporteador

- **Subsistema:** T-SQL
- **Base de Datos:** SCCTI
- **Comando:**
```sql
Execute dbo.Spp_NotificacionesReporteador
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 4 segundos

---

## CANCELACION_SOLICITUDES_VIGENCIA

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | CANCELACION_SOLICITUDES_VIGENCIA |
| **Estado** | ✅ Activo |
| **Owner** | SEGURIDAD-CORPO\Administrator |
| **Base de Datos** | CTRLAsistencia |
| **Descripción** | No description available. |

### Propósito

Proceso automático de cancelación de solicitudes que han excedido su período de vigencia.

### Pasos del Job

#### Step 1: CANCELACION_SOLICITUDES_VIGENCIA

- **Subsistema:** T-SQL
- **Base de Datos:** CTRLAsistencia
- **Comando:**
```sql
Declare 
    @PdFechaProceso        Datetime     = Getdate()-1,
    @PnEstatus             Integer      = 0,
    @PsMensaje             Varchar(850) = ' '

Begin
    Execute prp_CancelacionSolicitudPorVigencia 
        @PdFechaProceso = @PdFechaProceso,
        @PnEstatus      = @PnEstatus Output,
        @PsMensaje      = @PsMensaje Output
    
    -- Select @PnEstatus, @PsMensaje
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

---

## Confirmación Correos Mars

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | Confirmación Correos Mars |
| **Estado** | ❌ Deshabilitado |
| **Owner** | sa |
| **Base de Datos** | NCTI |
| **Descripción** | Proceso Diario de Envio de Confirmación de Correos Mars |

### Propósito

Proceso diario automatizado que envía correos de confirmación a empleados de Mars que no han confirmado su dirección de correo electrónico.

### Pasos del Job

#### Step 1: Envio Correo

- **Subsistema:** T-SQL
- **Base de Datos:** NCTI
- **Comando:**
```sql
Declare 
    @PnEstatus            Integer       = 0,
    @PsMensaje            Varchar(max)  = ''

Begin
    Execute dbo.Spp_CorreoMarsDiario @PnEstatus = @PnEstatus Output,
                                     @PsMensaje = @PsMensaje Output
    
    Select @PnEstatus, @PsMensaje
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

---

## Confirmacion Correos RHIN

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | Confirmacion Correos RHIN |
| **Estado** | ❌ Deshabilitado |
| **Owner** | sa |
| **Base de Datos** | NCTI |
| **Descripción** | Proceso que Busca los Empleados de RHIN que no Han Confirmado su Correo y Se les Envia Correo Para que Confirmen su Correo |

### Pasos del Job

#### Step 1: Genera confirmación de correo

- **Subsistema:** T-SQL
- **Base de Datos:** NCTI
- **Comando:**
```sql
Execute dbo.Spp_CorreoRhinDiario
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 50 segundos

---

## CONTROLASISTENCIA_OPERACION

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | CONTROLASISTENCIA_OPERACION |
| **Estado** | ✅ Activo |
| **Owner** | SEGURIDAD-CORPO\Administrator |
| **Base de Datos** | CTRLAsistencia |
| **Descripción** | No description available. |

### Propósito

Proceso automatizado para el control de asistencia de operaciones, incluyendo procesamiento de marcas y actualización de nombres de usuario.

### Pasos del Job

#### Step 1: Paso 2

- **Subsistema:** T-SQL
- **Base de Datos:** CTRLAsistencia
- **Comando:**
```sql
Declare 
    @PnidMarca              Smallint     = Null,
    @PnEstatus              Integer      = 0,
    @PsMensaje              Varchar(850) = ' '

Begin
    Execute prp_ProcAutomaticoControlAsistenciaMarcaOperacion 
        @PnidMarca = @PnidMarca,
        @PnEstatus = @PnEstatus Output,
        @PsMensaje = @PsMensaje Output
    
    Select @PnEstatus, @PsMensaje
End
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 12 minutos 1 segundo

**Acción en éxito:** Ir al paso 3
**Acción en fallo:** Salir reportando fallo

---

#### Step 2: Paso 3

- **Subsistema:** T-SQL
- **Base de Datos:** CTRLAsistencia
- **Comando:**
```sql
Declare 
    @PnEstatus      Int          = Null,
    @PsMensaje      Varchar(250) = Null

Begin
    Exec Spa_NombreUsuario_CA 
        @PnEstatus = @PnEstatus Output,
        @PsMensaje = @PsMensaje Output
    
    Select @PnEstatus As Estatus, @PsMensaje As Mensaje
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

---

## Depuración Solicitudes Cambio Correo

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | Depuración Solicitudes Cambio Correo |
| **Estado** | ❌ Deshabilitado |
| **Owner** | usr_NotifSeg_app |
| **Base de Datos** | SCSTI |
| **Descripción** | No description available. |

### Pasos del Job

#### Step 1: Depuración Solicitud Cambio de Correo

- **Subsistema:** T-SQL
- **Base de Datos:** SCSTI
- **Comando:**
```sql
Execute dbo.Spp_DepuraSolicitudCambios
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

---

## EnviaNotificacionesProgramadas

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | EnviaNotificacionesProgramadas |
| **Estado** | ✅ Activo |
| **Owner** | SEGURIDAD-CORPO\Administrator |
| **Base de Datos** | CTRLAsistencia |
| **Descripción** | No description available. |

### Propósito

Job que ejecuta el envío de notificaciones que han sido programadas previamente.

### Pasos del Job

#### Step 1: EnviaNotProg

- **Subsistema:** T-SQL
- **Base de Datos:** CTRLAsistencia
- **Comando:**
```sql
Exec spp_EnviaNotificacionProgramada_Temp
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

---

## Envío Póliza Rhin

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | Envío Póliza Rhin |
| **Estado** | ❌ Deshabilitado |
| **Owner** | sa |
| **Base de Datos** | NCTI |
| **Descripción** | Proceso de Envío de Polizas de Seguro a Colaboradores Rhin |

### Pasos del Job

#### Step 1: Envío Póliza Rhin

- **Subsistema:** T-SQL
- **Base de Datos:** NCTI
- **Comando:**
```sql
Declare 
    @PnIdMotivo            Smallint      = 39,
    @PsParametros          Varchar(8000) = '21|1|||',
    @PnIdTipoNotificacion  Tinyint       = 1,
    @PsURL                 Varchar(2000) = '',
    @PdFechaProgramada     Datetime      = Getdate(),
    @PnIdUsuarioAct        Smallint      = 268,
    @PnEstatus             Integer       = 0,
    @PsMensaje             Varchar(250)  = Null

Begin
    Execute NCTI.NCTI.dbo.Spa_conProcesosTbl 
        @PnIdMotivo           = @PnIdMotivo,
        @PnIdTipoNotificacion = @PnIdTipoNotificacion,
        @PsParametros         = @PsParametros,
        @PsURL                = @PsURL,
        @PdFechaProgramada    = @PdFechaProgramada,
        @PnIdUsuarioAct       = @PnIdUsuarioAct,
        @PnEstatus            = @PnEstatus Output,
        @PsMensaje            = @PsMensaje Output
    Return
End
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

---

## ENVIO_REPORTEASISTENCIA

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | ENVIO_REPORTEASISTENCIA |
| **Estado** | ✅ Activo |
| **Owner** | SEGURIDAD-CORPO\Administrator |
| **Base de Datos** | CTRLAsistencia |
| **Descripción** | No description available. |

### Pasos del Job

#### Step 1: ENVIO_REPORTEASISTENCIA

- **Subsistema:** T-SQL
- **Base de Datos:** CTRLAsistencia
- **Comando:**
```sql
Exec Spp_envioReporteAsistencia
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 2 segundos

---

## JobDiario

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | JobDiario |
| **Estado** | ✅ Activo |
| **Owner** | sa |
| **Base de Datos** | SMBDTI |
| **Descripción** | No description available. |

### Propósito

Job diario que analiza la ejecución de procedimientos almacenados en el servidor.

### Pasos del Job

#### Step 1: JobDiario

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PnEstatus        Integer      = 0,
    @PsMensaje        Varchar(250) = Null;

Begin
    Execute Spp_analizaEjecucionProcServ 
        @PnEstatus = @PnEstatus Output,
        @PsMensaje = @PsMensaje Output;
    
    If @PnEstatus != 0
    Begin
        Select @PnEstatus As Estatus, @PsMensaje As Mensaje
    End
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 3 segundos

---

## MonitoreoSQL

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | MonitoreoSQL |
| **Estado** | ✅ Activo |
| **Owner** | sa |
| **Base de Datos** | SMBDTI |
| **Descripción** | Monitorea errores, timeouts, bloqueos, CPU y transacciones abiertas. Guarda todo en dbo.MonitoreoSQL. |

### Propósito

Sistema de monitoreo integral de SQL Server que:
- Captura eventos de Extended Events (XEL)
- Monitorea errores y timeouts
- Detecta bloqueos y uso de CPU
- Registra transacciones abiertas
- Limpia registros antiguos automáticamente

### Pasos del Job

#### Step 1: Importar eventos de XEL

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
EXEC dbo.sp_ImportaEventosSQL;
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 2
- Intervalo de reintento: 5 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 32 minutos 8 segundos

---

#### Step 2: Limpiar registros antiguos

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
DELETE FROM dbo.MonitoreoSQL
WHERE FechaHora < DATEADD(DAY, -7, GETDATE());
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 1
- Intervalo de reintento: 10 minutos

**Última Ejecución:**
- Resultado: ✅ Éxito
- Duración: 0 segundos

---

#### Step 3: Registrar transacciones abiertas

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
EXEC dbo.sp_RegistraTransaccionesAbiertas;
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 2
- Intervalo de reintento: 5 minutos

**Última Ejecución:**
- Resultado: ✅ Éxito
- Duración: 0 segundos

---

## Movimientos Colaboradores E-Learning

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | Movimientos Colaboradores E-Learning |
| **Estado** | ❌ Deshabilitado |
| **Owner** | sa |
| **Base de Datos** | NCTI |
| **Descripción** | Emisión de Correos por Movimientos de Colaboradores en Plataformas E-Learning |

### Pasos del Job

#### Step 1: Movimientos Colaboradores E-Learnig

- **Subsistema:** T-SQL
- **Base de Datos:** NCTI
- **Comando:**
```sql
Execute Spa_EnviaNotificacionMovEmpleadosEL
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 2 segundos

---

## Notificaciones Campañas

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | Notificaciones Campañas |
| **Estado** | ❌ Deshabilitado |
| **Owner** | sa |
| **Base de Datos** | NCTI |
| **Descripción** | Notificaciones de Campañas Campañas |

### Pasos del Job

#### Step 1: Notificaciones Campañas

- **Subsistema:** T-SQL
- **Base de Datos:** NCTI
- **Comando:**
```sql
Execute dbo.Spp_catCampanaTbl
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

---

## Notificaciones Cursos Campañas

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | Notificaciones Cursos Campañas |
| **Estado** | ❌ Deshabilitado |
| **Owner** | sa |
| **Base de Datos** | SCCTI |
| **Descripción** | Notificaciones de los Cursos Elearning atravez de Campañas |

### Pasos del Job

#### Step 1: Notificaciones Cursos Campañas

- **Subsistema:** T-SQL
- **Base de Datos:** SCCTI
- **Comando:**
```sql
Execute dbo.Spp_catCursosTbl
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

---

## Notificaciones de Maniobras

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | Notificaciones de Maniobras |
| **Estado** | ❌ Deshabilitado |
| **Owner** | SEGURIDAD-CORPO\Administrator |
| **Base de Datos** | CTRLAsistencia |
| **Descripción** | No description available. |

### Pasos del Job

#### Step 1: Nottificaciones de Maniobras

- **Subsistema:** T-SQL
- **Base de Datos:** CTRLAsistencia
- **Comando:**
```sql
Begin
    Execute dbo.Spp_procesaNotificaBath
End
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ✅ Éxito
- Duración: 1 segundo

---

## Polizas Danone

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | Polizas Danone |
| **Estado** | ❌ Deshabilitado |
| **Owner** | sa |
| **Base de Datos** | NCTI |
| **Descripción** | Envio de Plizas a Danone |

### Pasos del Job

#### Step 1: Envío correo

- **Subsistema:** T-SQL
- **Base de Datos:** NCTI
- **Comando:**
```sql
Declare 
    @PnIdMotivo            Integer       = 31,
    @PnIdTipoNotificacion  Tinyint       = 1,
    @PsParametros          Varchar(8000) = '||',
    @PsURL                 Varchar(2000) = '',
    @PdFechaProgramada     Datetime      = Getdate(),
    @PnIdUsuarioAct        Smallint      = 1,
    @PnEstatus             Integer       = 0,
    @PsMensaje             Varchar(250)  = Null

Begin
    Execute dbo.Spa_conProcesosTbl 
        @PnIdMotivo           = @PnIdMotivo,
        @PnIdTipoNotificacion = @PnIdTipoNotificacion,
        @PsParametros         = @PsParametros,
        @PsURL                = @PsURL,
        @PdFechaProgramada    = @PdFechaProgramada,
        @PnIdUsuarioAct       = @PnIdUsuarioAct,
        @PnEstatus            = @PnEstatus Output,
        @PsMensaje            = @PsMensaje Output
    
    Select @PnEstatus, @PsMensaje
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

---

## Proceso Envia Correo

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | Proceso Envia Correo |
| **Estado** | ✅ Activo |
| **Owner** | sa |
| **Base de Datos** | NCTI |
| **Descripción** | No hay ninguna descripción. |

### Propósito

Job principal para el procesamiento y envío de correos pendientes en cola. Ver documentación detallada en [DOCUMENTACION_JOB_CORREOS_PENDIENTES.md](DOCUMENTACION_JOB_CORREOS_PENDIENTES.md).

### Pasos del Job

#### Step 1: Envia Correo

- **Subsistema:** T-SQL
- **Base de Datos:** NCTI
- **Comando:**
```sql
Declare 
    @PnIdUsuarioAct        Smallint     = 1,
    @PnEstatus             Integer      = 0,
    @PsMensaje             Varchar(250) = Null

Execute dbo.Spp_CorreosPendientes 
    @PnIdUsuarioAct,
    @PnEstatus Output,
    @PsMensaje Output
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ✅ Éxito
- Duración: 1 segundo

---

## Proceso Usuario Baja Reingreso

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | Proceso Usuario Baja Reingreso |
| **Estado** | ❌ Deshabilitado |
| **Owner** | sa |
| **Base de Datos** | NCTI |
| **Descripción** | Proceso donde se dan de baja o reingresa usuarios |

### Pasos del Job

#### Step 1: ProcesoBajaReingreso

- **Subsistema:** T-SQL
- **Base de Datos:** NCTI
- **Comando:**
```sql
Execute dbo.Spp_procesaBajaReingreso
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

---

## ProcesoDiarioMBD

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | ProcesoDiarioMBD |
| **Estado** | ✅ Activo |
| **Owner** | sa |
| **Base de Datos** | SMBDTI |
| **Descripción** | Proceso Diario de Validación y Mantenimiento de BD |

### Propósito

Proceso diario automatizado que ejecuta validaciones críticas del servidor SQL Server:
- Validación de tiempos de procesos
- Análisis de Link Servers
- Verificación de Jobs no ejecutados
- Validación de servicios (RHIN, Correos, Notificaciones Push)
- Detección de bloqueos
- Generación de notificaciones de estado

### Pasos del Job

#### Step 1: Paso 1

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PnEstatus               Integer      = 0,
    @PsMensaje               Varchar(250) = Null

Begin
    Execute dbo.Spp_validaTiempoProcesos 
        @PnEstatus = @PnEstatus Output,
        @PsMensaje = @PsMensaje Output;
    
    Select @PnEstatus, @PsMensaje
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

**Acción en éxito:** Ir al paso 3
**Acción en fallo:** Ir al paso 3

---

#### Step 2: Paso 2

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PnEstatus      Integer      = Null,
    @PsMensaje      Varchar(250) = Null,
    @w_idProceso    Integer      = 0;

Begin
    Execute dbo.Spp_AnalisisLinkServer 
        @PnEstatus = @PnEstatus Output,
        @PsMensaje = @PsMensaje Output;
    
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 3 segundos

**Acción en éxito:** Ir al paso 3
**Acción en fallo:** Ir al paso 3

---

#### Step 3: Paso 3

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PnEstatus       Integer      = Null,
    @PsMensaje      Varchar(250) = Null,
    @w_idProceso    Integer      = 0;

Begin
    Execute dbo.Spp_AnalisisJobsNoEjecutados 
        @PnEstatus = @PnEstatus Output,
        @PsMensaje = @PsMensaje Output;
    
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

**Acción en éxito:** Ir al paso 3
**Acción en fallo:** Ir al paso 3

---

#### Step 4: Paso 4

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PnEstatus      Integer      = Null,
    @PsMensaje      Varchar(250) = Null

Begin
    Execute dbo.Spp_ValidaServicioProcesosRhin 
        @PnEstatus = @PnEstatus Output,
        @PsMensaje = @PsMensaje Output;
    
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

**Acción en éxito:** Ir al paso 3
**Acción en fallo:** Ir al paso 3

---

#### Step 5: Paso 5

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PsMensaje           Varchar(250) = Null,
    @w_idProceso         Integer      = 0,
    @PnEstatus           Integer      = 0

Begin
    Execute Spp_ValidaServicioCorreos 
        @PnEstatus = @PnEstatus Output,
        @PsMensaje = @PsMensaje Output
    
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

**Acción en éxito:** Ir al paso 3
**Acción en fallo:** Ir al paso 3

---

#### Step 6: Paso 6

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PnEstatus             Integer      = 0,
    @PsMensaje             Varchar(250) = Null;

Begin
    Execute Spp_ValidaServicioNotPush 
        @PnEstatus = @PnEstatus Output,
        @PsMensaje = @PsMensaje Output
    
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

**Acción en éxito:** Ir al paso 3
**Acción en fallo:** Ir al paso 3

---

#### Step 7: Paso 7

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PnEstatus               Integer      = 0,
    @PsMensaje               Varchar(250) = Null

Begin
    Execute dbo.Spp_validaBloqueos 
        @PnEstatus = @PnEstatus Output,
        @PsMensaje = @PsMensaje Output;
    
    Return
End
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

**Acción en éxito:** Ir al paso 3
**Acción en fallo:** Ir al paso 3

---

#### Step 8: Solicita Correos

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PdFechaProceso        Date          = Getdate(),
    @PsIdProceso           Varchar(1000) = '11, 12, 14, 15, 16',
    @PnEstatus             Integer       = 0,
    @PsMensaje             Varchar(250)  = Null;

Begin
    Execute Spp_SolicitaCorreoMantBDV2 
        @PdFechaProceso = @PdFechaProceso,
        @PsIdProceso    = @PsIdProceso,
        @PnEstatus      = @PnEstatus Output,
        @PsMensaje      = @PsMensaje Output
    
    Select @PnEstatus, @PsMensaje
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 1 segundo

---

## ProcesoFinDiaMantBaseDatos

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | ProcesoFinDiaMantBaseDatos |
| **Estado** | ✅ Activo |
| **Owner** | sa |
| **Base de Datos** | SMBDTI |
| **Descripción** | Proceso Fin de Día de Mantenimiento de Base de Datos |

### Propósito

Proceso que se ejecuta al final del día para realizar tareas de mantenimiento y depuración de tablas.

### Pasos del Job

#### Step 1: Paso 1

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PnEstatus             Integer      = 0,
    @PsMensaje             Varchar(250) = Null;

Begin
    Execute Spp_MantenimientoDepuraTablas 
        @PnEstatus = @PnEstatus Output,
        @PsMensaje = @PsMensaje Output
    
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 2 segundos

**Acción en éxito:** Ir al paso 3
**Acción en fallo:** Ir al paso 3

---

#### Step 2: Solicita Correo

- **Subsistema:** T-SQL
- **Base de Datos:** SMBDTI
- **Comando:**
```sql
Declare 
    @PdFechaProceso        Date         = Null,
    @PsIdProceso           Varchar(250) = '13',
    @PnEstatus             Integer      = 0,
    @PsMensaje             Varchar(250) = Null;

Begin
    Execute Spp_SolicitaCorreoMasivoMantBD 
        @PdFechaProceso = @PdFechaProceso,
        @PsIdProceso    = @PsIdProceso,
        @PnEstatus      = @PnEstatus Output,
        @PsMensaje      = @PsMensaje Output
    
    Select @PnEstatus, @PsMensaje
    Return
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 2 segundos

---

## PROCESOS ESPECIALES

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | PROCESOS ESPECIALES |
| **Estado** | ✅ Activo |
| **Owner** | SEGURIDAD-CORPO\Administrator |
| **Base de Datos** | CTRLAsistencia |
| **Descripción** | No description available. |

### Pasos del Job

#### Step 1: PROCESOS ESPECIALES

- **Subsistema:** T-SQL
- **Base de Datos:** CTRLAsistencia
- **Comando:**
```sql
Declare 
    @PnEstatus                Integer      = 0,
    @PsMensaje                Varchar(850) = ' '

Begin
    Execute prp_ProcesoEspecialSolicitud 
        @PnEstatus = @PnEstatus Output,
        @PsMensaje = @PsMensaje Output
    
    Select @PnEstatus, @PsMensaje
End
Go
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 27 segundos

---

## VerificaConfirmacionCorreoKaizen

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Job Name** | VerificaConfirmacionCorreoKaizen |
| **Estado** | ❌ Deshabilitado |
| **Owner** | sa |
| **Base de Datos** | NCTI |
| **Descripción** | Proceso de Verificacio de Correo Kaizen |

### Pasos del Job

#### Step 1: Valida correos Kaizen

- **Subsistema:** T-SQL
- **Base de Datos:** NCTI
- **Comando:**
```sql
Declare 
    @PnEstatus            Integer       = 0,
    @PsMensaje            Varchar(max)  = ''

Execute Spp_CorreoKaizenDiario 
    @PnEstatus = @PnEstatus Output,
    @PsMensaje = @PsMensaje Output
```

**Configuración de Reintentos:**
- Reintentos en caso de fallo: 0
- Intervalo de reintento: 0 minutos

**Última Ejecución:**
- Resultado: ❌ Error
- Duración: 3 minutos 48 segundos

---

## Resumen de Estados de Jobs

### Jobs Activos (10)

1. AnalisisSemanalBD
2. Asistencia_Staff
3. CANCELACION_SOLICITUDES_VIGENCIA
4. CONTROLASISTENCIA_OPERACION
5. EnviaNotificacionesProgramadas
6. ENVIO_REPORTEASISTENCIA
7. JobDiario
8. MonitoreoSQL
9. Proceso Envia Correo
10. ProcesoDiarioMBD
11. ProcesoFinDiaMantBaseDatos
12. PROCESOS ESPECIALES

### Jobs Deshabilitados (17)

1. Actualiza Datos E-learning
2. bkp_SIAN_GASTOS_Seguridad
3. Campañas Leidas
4. Campañas Reporteador
5. Confirmación Correos Mars
6. Confirmacion Correos RHIN
7. Depuración Solicitudes Cambio Correo
8. Envío Póliza Rhin
9. Movimientos Colaboradores E-Learning
10. Notificaciones Campañas
11. Notificaciones Cursos Campañas
12. Notificaciones de Maniobras
13. Polizas Danone
14. Proceso Usuario Baja Reingreso
15. VerificaConfirmacionCorreoKaizen

---

## Notas Importantes

- **Última actualización:** 12 de diciembre de 2025
- Los datos se obtuvieron del análisis de los jobs configurados en SQL Server
- Se recomienda revisar periódicamente el estado de los jobs deshabilitados
- Muchos jobs muestran último resultado de error, se sugiere investigación adicional
- Los jobs con múltiples pasos utilizan lógica de salto condicional basada en éxito/fallo

---

## Mantenimiento y Recomendaciones

1. **Jobs Críticos:** Los jobs activos deben monitorearse diariamente
2. **Jobs en Error:** Investigar y resolver los jobs que reportan último resultado en error
3. **Jobs Deshabilitados:** Evaluar si deben reactivarse o pueden eliminarse
4. **Documentación:** Agregar descripciones faltantes a los jobs sin documentación
5. **Notificaciones:** Configurar alertas para jobs críticos que fallen
6. **Respaldos:** Asegurar que los procesos de backup estén funcionando correctamente
