# PRD - Tipado de Módulos Móviles

## 1. Resumen Ejecutivo

### 1.1 Descripción General
Este documento describe los requisitos del producto para la implementación de un sistema de tipado robusto para módulos móviles en el catálogo de la Suite. La modificación reemplaza la columna booleana `isWebView` por un campo tipado que permita clasificar los módulos en cuatro categorías distintas: nativo, webview, informativo y redireccionamiento.

### 1.2 Objetivos del Producto
- Implementar un sistema de tipado flexible y escalable para módulos móviles
- Facilitar el renderizado y comportamiento correcto de cada tipo de módulo en la aplicación móvil
- Permitir al equipo móvil tomar decisiones de UI/UX basadas en el tipo de módulo

### 1.3 Alcance
**En Alcance:**
- Modificación del esquema de base de datos (columna `isWebView` → `tipoModulo`)
- Actualización del endpoint `/v1/catalogs/mobile-modules/user`
- Migración de datos existentes al nuevo esquema
- Documentación de los tipos de módulos y sus comportamientos
- Actualización de DTOs y modelos en el backend
- Agregar 3 nuevos módulos informativos:
  - **idAplicacion 4**: Reclutamiento y Selección
  - **idAplicacion 36**: Informate
  - **idAplicacion 43**: Comunicación Interna

**Fuera de Alcance:**
- Cambios en la implementación móvil (responsabilidad del equipo móvil)
- Modificaciones en los módulos existentes más allá del tipado (excepto los 3 nuevos módulos informativos)
- Cambios en la lógica de permisos o autorizaciones

---

## 2. Contexto del Negocio

### 2.1 Problema a Resolver
Actualmente el sistema utiliza un campo booleano `isWebView` que solo permite distinguir entre módulos que son WebView (TRUE) y los que no lo son (FALSE/NULL). Esta limitación presenta los siguientes problemas:

- **Falta de flexibilidad**: No permite categorizar módulos informativos o de redireccionamiento
- **Experiencia de usuario**: El equipo móvil no puede determinar el comportamiento esperado del módulo sin lógica adicional
- **Escalabilidad**: Agregar nuevos tipos de módulos requeriría columnas adicionales

### 2.2 Usuarios y Stakeholders
- **Equipo de Desarrollo Móvil**: Consumidores principales del endpoint, necesitan saber cómo renderizar cada módulo
- **Usuarios finales de la App**: Se benefician de una experiencia más consistente y apropiada según el tipo de módulo
- **Equipo de Desarrollo Suite**: Responsables de implementar y mantener el cambio
- **Product Owners**: Definen nuevos módulos y sus comportamientos

### 2.3 Casos de Uso
1. **Usuario accede a módulo nativo**: La app móvil abre una pantalla nativa desarrollada específicamente
2. **Usuario accede a módulo webview**: La app móvil abre un navegador interno con la URL configurada
3. **Usuario accede a módulo informativo**: La app móvil muestra información estática sin navegación
4. **Usuario accede a módulo de redireccionamiento**: La app móvil redirige a otra sección o módulo

---

## 3. Especificaciones Técnicas
### 3.2 Tipos de Módulos
#### 3.2.1 Definición de Tipos

| Tipo | Valor | Descripción | Requiere URL | Comportamiento Esperado | Visible |
|------|-------|-------------|--------------|------------------------|---------|
| **Nativo(1)** | `nativo` | Módulo implementado nativamente en la app móvil | No | Navega a pantalla nativa específica del módulo | Menú y Notificaciones |
| **WebView(2)** | `webview` | Módulo que se muestra en un navegador interno | Sí | Abre WebView con la URL configurada | Menú y Notificaciones |
| **Informativo(3)** | `informativo` | Módulo que solo se muestra en el apartado de notificaciones | No | Visible solo en recepción de notificaciones | Notificaciones |
| **Redireccionamiento(4)** | `redireccionamiento` | Módulo que redirige a otra sección o URL externa | Sí (opcional) | Redirige según configuración(solo Android) | Menú |
#### 3.2.2 Nuevos Módulos Informativos a Crear

Como parte de este desarrollo, se crearán 3 nuevos módulos de tipo informativo que solo serán visibles en el apartado de notificaciones:

| idAplicacion | Nombre del Módulo | Tipo | Descripción |
|--------------|-------------------|------|-------------|
| 4 | Reclutamiento y Selección | informativo | Información sobre procesos de reclutamiento y vacantes internas |
| 36 | Informate | informativo | Comunicados y noticias corporativas |
| 43 | Comunicación Interna | informativo | Boletines y anuncios importantes de la empresa | 

**Características de estos módulos:**
- `tipoModulo`: 3 (informativo)
- `webViewUrl`: NULL
- Visibles únicamente en sección de notificaciones push
- No aparecen en el menú principal de la aplicación
- Configuración de icono requerida para identificación en notificaciones(me falta el icono)
#### 3.3.2 Response Actual (Antes del Cambio)
```json
[
    {
        "idModulo": 1,
        "idAplicacion": 9,
        "modulo": "E-Learning",
        "webViewUrl": "https://www.portal3i.mx/2546-idmkt/course/view.php",
        "idModuloPadre": 0,
        "orden": 1,
        "icono": "https://storage.googleapis.com/servicios_storage/suite/850cd588-c8c7-43f8-b97e-80b402174dc5.svg",
        "isWebView": true,
        "children": []
    },
    {
        "idModulo": 7,
        "idAplicacion": 66,
        "modulo": "Crédito Hipotecario",
        "webViewUrl": "https://boomfinance.mx",
        "idModuloPadre": 0,
        "orden": 3,
        "icono": "https://storage.googleapis.com/servicios_storage/suite/e9eaa1d2-09c4-4c3d-8f70-cac720e0594f.svg",
        "isWebView": true,
        "children": []
    },
]
```

#### 3.3.3 Response Propuesto (Después del Cambio)
```json
[
    {
        "idModulo": 1,
        "idAplicacion": 9,
        "modulo": "E-Learning",
        "webViewUrl": "https://www.portal3i.mx/2546-idmkt/course/view.php",
        "idModuloPadre": 0,
        "orden": 1,
        "icono": "https://storage.googleapis.com/servicios_storage/suite/850cd588-c8c7-43f8-b97e-80b402174dc5.svg",
        "tipoModulo": 1,
        "children": []
    },
    {
        "idModulo": 7,
        "idAplicacion": 66,
        "modulo": "Crédito Hipotecario",
        "webViewUrl": "https://boomfinance.mx",
        "idModuloPadre": 0,
        "orden": 3,
        "icono": "https://storage.googleapis.com/servicios_storage/suite/e9eaa1d2-09c4-4c3d-8f70-cac720e0594f.svg",
        "tipoModulo": 2,
        "children": []
    },
]
```
---

## 4. Consideraciones Técnicas

### 4.1 Retrocompatibilidad
No se requiere retrocompatibilidad dado que la aplicación es nueva y todavia no tiene operación.

---

## 5. Criterios de Aceptación (Gherkin)

### 5.1 Feature: Tipado de Módulos Móviles

```gherkin
Feature: Tipado de módulos móviles en el catálogo
  Como equipo de desarrollo móvil
  Quiero recibir un campo tipoModulo en el endpoint de catálogo
  Para poder renderizar correctamente cada módulo según su tipo

  Background:
    Given el endpoint "/v1/catalogs/mobile-modules/user" está disponible
    And existen módulos configurados en la base de datos

  Scenario: Obtener módulos con campo tipoModulo
    Given un usuario autenticado hace una petición al endpoint
    When se solicita el catálogo de módulos móviles
    Then la respuesta debe incluir el campo "tipoModulo" para cada módulo
    And el campo "tipoModulo" debe tener uno de los valores: "nativo(1)", "webview(2)", "informativo(3)", "redireccionamiento(4)"

  Scenario: Validar módulo de tipo nativo
    Given un módulo con tipoModulo "nativo(1)"
    When se obtiene el catálogo de módulos
    Then el módulo debe tener tipoModulo = "nativo(1)"
    And el campo webViewUrl puede ser null

  Scenario: Validar módulo de tipo webview
    Given un módulo con tipoModulo "webview(2)"
    When se obtiene el catálogo de módulos
    Then el módulo debe tener tipoModulo = "webview(2)"
    And el campo webViewUrl debe contener una URL válida
    And el campo webViewUrl no debe ser null

  Scenario: Validar módulo de tipo informativo
    Given un módulo con tipoModulo "informativo(3)"
    When se obtiene el catálogo de módulos
    Then el módulo debe tener tipoModulo = "informativo(3)"
    And el campo webViewUrl puede ser null

  Scenario: Validar módulo de tipo redireccionamiento
    Given un módulo con tipoModulo "redireccionamiento(4)"
    When se obtiene el catálogo de módulos
    Then el módulo debe tener tipoModulo = "redireccionamiento(4)"
    And el campo webViewUrl puede contener una URL de destino

  Scenario Outline: Migración de datos existentes
    Given un módulo con isWebView = <isWebView_valor>
    And el módulo es identificado como <tipo_identificado>
    When se ejecuta el script de migración
    Then el campo tipoModulo debe ser <tipo_esperado>
    And los datos del módulo deben permanecer intactos

    Examples:
      | isWebView_valor | tipo_identificado      | tipo_esperado        |
      | true            | E-Learning             | webview              |
      | false           | Control asistencia     | nativo               |
      | null            | Solicitudes            | nativo               |
      | false           | Personal               | informativo          |
      | true            | AppIntegraSalud        | redireccionamiento   |

  Scenario: Retrocompatibilidad con campo isWebView
    Given módulos migrados con el nuevo campo tipoModulo
    When el equipo móvil solicita el catálogo
    Then todos los módulos deben incluir tanto tipoModulo como isWebView
    And el valor de isWebView debe ser consistente con tipoModulo

  Scenario: Equipo móvil renderiza módulo nativo
    Given un módulo con tipoModulo "nativo(1)" en el catálogo
    When la aplicación móvil procesa el módulo
    Then debe navegar a la pantalla nativa correspondiente
    And no debe intentar abrir un webview

  Scenario: Equipo móvil renderiza módulo webview
    Given un módulo con tipoModulo "webview(2)" y webViewUrl válida
    When la aplicación móvil procesa el módulo
    Then debe abrir un webview interno
    And debe cargar la URL configurada en webViewUrl

  Scenario: Equipo móvil renderiza módulo informativo
    Given un módulo con tipoModulo "informativo(3)"
    When la aplicación móvil procesa el módulo
    Then debe mostrar la aplicación solo en notificaciones
    And no debe realizar navegación adicional

  Scenario: Equipo móvil renderiza módulo de redireccionamiento solo Android
    Given un módulo con tipoModulo "redireccionamiento(4)"
    When la aplicación móvil procesa el módulo
    Then debe redirigir según la configuración
    And puede abrir una app externa o sección específica(play store)
```

### 6.2 Criterios de Aceptación Checklist

#### Funcionales
- ✅ El endpoint `/v1/catalogs/mobile-modules/user` incluye el campo `tipoModulo`
- ✅ Todos los módulos existentes tienen un tipo válido asignado
- ✅ El campo `tipoModulo` acepta únicamente los 4 valores definidos
- ✅ Módulos de tipo 'webview' tienen una URL válida configurada

#### De Negocio
- ✅ Equipo móvil valida que recibe correctamente el campo `tipoModulo`
- ✅ Equipo móvil puede renderizar correctamente cada tipo de módulo

---

## 7. Historial de Cambios

| Versión | Fecha | Autor | Cambios |
|---------|-------|-------|---------|
| 1.0 | 2026-01-21 |  | Documento inicial |

---
