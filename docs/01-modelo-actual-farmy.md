# FARMY — Modelo actual del sistema (a hoy)

Este documento describe **cómo está organizado Farmy hoy**, de forma didáctica: la arquitectura general, los módulos que existen, los procesos de negocio y cómo funciona la seguridad.

---

## 1. Vista de pájaro: ¿qué es Farmy?

Farmy es un **ERP farmacéutico** compuesto por dos aplicaciones que conversan entre sí:

```mermaid
flowchart LR
    subgraph Usuarios
        U1["👩‍💼 Personal interno<br/>(compras, comercial, bodega, GH)"]
        U2["🏥 Usuarios institucionales<br/>(puntos de venta)"]
        U3["📱 Clientes por WhatsApp"]
    end

    subgraph Front["PharmacyFrontEnd (Angular 20)"]
        SPA["SPA con DevExtreme + AG Grid<br/>26 áreas funcionales"]
    end

    subgraph API["PharmacyAPI (ASP.NET Core 8)"]
        C["Controllers<br/>(REST + OData)"]
        S["Services<br/>(lógica de negocio)"]
        P["EF Core 9<br/>(Persistence)"]
    end

    DB[("SQL Server")]

    subgraph Externos["Servicios externos"]
        WA["WhatsApp Cloud API / WATI"]
        CLAUDE["Claude API<br/>(bot de atención)"]
        MAIL["Ingesta de correos"]
        ETL["ETL de precios<br/>(TB_UltimosPreciosProductos)"]
    end

    U1 --> SPA
    U2 --> SPA
    U3 --> WA
    SPA -->|"HTTP + cookie JWT"| C
    C --> S --> P --> DB
    WA <--> API
    API <--> CLAUDE
    MAIL --> API
    ETL --> DB
```

**En una frase:** el front Angular consume la API .NET, que concentra toda la lógica y persiste en SQL Server; además la API habla con WhatsApp (aprobaciones y atención al cliente con bot de Claude), recibe correos y se apoya en una ETL externa que alimenta la tabla de precios/productos.

---

## 2. Stack tecnológico

| Capa | Tecnología | Notas |
|---|---|---|
| Frontend | Angular 20 (standalone components) | Sin NgModules |
| UI | DevExtreme 25 + AG Grid 35 + Angular Material + Bootstrap 5 | **4 librerías de UI conviviendo** |
| Exportaciones | ExcelJS, xlsx, jsPDF + autotable | Excel y PDF desde el front |
| Backend | ASP.NET Core 8, C# | API REST + OData (`$filter`, `$expand`…) |
| ORM | Entity Framework Core 9 | Fluent API en `Persistence/Configurations` (60+ archivos) |
| BD | SQL Server | Única base de datos |
| Auth | JWT en **cookie** + BCrypt | Sliding token + versionado de configuración de token |
| Mensajería | WhatsApp Cloud API o WATI (conmutables por config) | `IWaGateway` decide el canal |
| IA | Claude API (`WaBotClaudeService`) | Bot de primera línea en WhatsApp |
| Docs API | Swagger | Solo en desarrollo |

---

## 3. Mapa de módulos del backend (PharmacyAPI)

La API se organiza en **módulos por dominio**, cada uno con `Controllers / Services / Models / DTO`:

```mermaid
mindmap
  root((PharmacyAPI))
    Auth
      Usuarios
      Security["Security (JWT, permisos, acciones)"]
      Menus
    Comercial
      Clientes
      Vendedores
      Regulados
      Cotizaciones
      Remisiones
      Financiero
    Compras
      OrdenCompra["Órdenes de compra"]
      PorAprobar
      ProductoPendiente
      Catalogos["Estados, causas devolución, métodos pago"]
    Institucional
      PedidosInst["Pedidos institucionales"]
      PuntosVenta["Puntos de venta"]
      Rotacion["Rotación + logs"]
      UsuariosInst["Usuarios/menús/permisos propios"]
    Operacion
      Domicilios
      Codificaciones["Codificaciones (inventario)"]
      Precios["Precios (lotes, homólogos, ETL)"]
      Correos["Correos (ingesta + casos)"]
    Personas
      GestionHumana["Gestión Humana (permisos, incapacidades, vacaciones, llamados, desprendibles)"]
      Evaluaciones["Evaluaciones / Escuela"]
    Canales
      AtencionWhatsApp["Atención WhatsApp (bot + asesores)"]
      NotificacionesWA["Notificaciones WhatsApp (aprobaciones OC)"]
    Catalogos2["Catálogos"]
      Sedes
      Convenios
      Distribuidores
      Homologos["Homólogos / tarifario"]
      LineasNegocio
      CamposBD
    Transversal
      Roles
      Menus2["Menús"]
      Configuraciones["Configuraciones (token)"]
      Resumen["Resumen / Dashboard"]
```

**Números de hoy:** 21 carpetas de módulos + `Auth`, ~70 servicios registrados en `DependencyInjection.cs`, 60+ entidades configuradas en el `AppDbContext`.

> ⚠️ El README de la API dice "14 módulos": está desactualizado, hoy hay 21.

---

## 4. Mapa del frontend (PharmacyFrontEnd)

```mermaid
flowchart TB
    subgraph Shell["Shell de la aplicación"]
        LAY["Layouts (side-nav + toolbar)"]
        NAV["app-navigation.ts<br/>(menú hardcodeado)"]
        GUARDS["Guards: AuthGuard + PermissionGuard"]
        INT["jwt.interceptor"]
    end

    subgraph Features["pages/modules (26 áreas)"]
        F1["compras: gestor-pedido, orden-compra,<br/>por-aprobar, estado-orden, faltantes-correos"]
        F2["comercial: cliente, vendedor, regulado,<br/>cotizador, cotizaciones"]
        F3["bodega: recepción, despacho,<br/>despacho-institucional, consulta-pedidos"]
        F4["institucional: pedido, rotación, punto-venta,<br/>usuarios, permisos, menú"]
        F5["gestion-humana: permisos, autorización,<br/>incapacidades, llamados, vacaciones,<br/>desprendibles, certificados"]
        F6["evaluaciones: escuelas, tablero,<br/>mis-evaluaciones, reportes, buzón"]
        F7["domicilios: crear, consultar, logística,<br/>rápido, entidades"]
        F8["otros: whatsapp-atención, financiera,<br/>ETL, precios, inventario, catálogos,<br/>roles/usuarios/menús/acciones"]
    end

    subgraph Shared["shared/"]
        GRID["farmy-grid (wrapper AG Grid)"]
        SRV["services: auth, generic-api,<br/>export excel/pdf, notification,<br/>session heartbeat/keepalive"]
        CMP["componentes: login, header,<br/>fy-confirm, fy-page-header…"]
    end

    Shell --> Features
    Features --> Shared
```

**Cómo llega el usuario a una pantalla:** `app.routes.ts` (todas las rutas **importadas de forma eager**) → guards de sesión y permisos → componente de la feature → servicios compartidos → API.

---

## 5. Procesos de negocio principales

### 5.1 Compras: del pedido a la recepción en bodega

```mermaid
flowchart LR
    A["🛒 Gestor de Pedidos<br/>(buscador sobre<br/>TB_UltimosPreciosProductos<br/>alimentada por ETL)"] --> B["📋 Por Aprobar<br/>(carrito consolidado)"]
    B --> C{"Aprobación<br/>por WhatsApp"}
    C -->|PENDIENTE_1 → PENDIENTE_2| C
    C -->|APROBADA| D["📄 Orden de Compra<br/>(encabezado + detalle + bitácora)"]
    C -->|RECHAZADA| X["Fin / ajuste"]
    D --> E["📦 Recepción en Bodega<br/>(validación por detalle)"]
    E --> F["✅ Cierre OC"]
    E --> G["⏳ Producto Pendiente /<br/>Causa de devolución"]
```

La aprobación usa **plantillas de WhatsApp** con dos niveles de verificadores (`PENDIENTE_1`, `PENDIENTE_2`, `APROBADA`, `RECHAZADA` en `VerificacionEstados`), gestionadas por el módulo `Notificaciones/WhatsApp`.

### 5.2 Comercial: de la cotización a la remisión

```mermaid
flowchart LR
    A["💰 Cotizador<br/>(gestor-cotizacion)"] --> B["📑 Cotización<br/>(encabezado + detalle)"]
    B --> C{"Estado del<br/>cotizador"}
    C -->|Aprobar| D["Remisión"]
    C -->|Rechazar / Anular| X["Fin<br/>(reversible: revert-reject)"]
    D --> E["🚚 Despacho en Bodega"]
    B -.-> F["Cliente + condición financiera<br/>(módulo Financiero: cupos,<br/>autorizaciones, historial)"]
```

### 5.3 Institucional: pedidos de puntos de venta

```mermaid
flowchart LR
    A["🏥 Punto de venta<br/>(usuario institucional)"] --> B["📝 Pedido Institucional<br/>(correlativo + detalle)"]
    B --> C["🔎 Revisión<br/>(acciones de revisión)"]
    C --> D["🛍️ Marcar comprado"]
    D --> E["🚚 Despacho institucional<br/>(bodega)"]
    E --> F["📦 Confirmar recepción<br/>(punto de venta)"]
    B -.-> G["Bitácoras de encabezado<br/>y detalle"]
    H["📊 Rotación institucional<br/>(importación masiva + logs)"] -.-> A
```

Este módulo tiene su **propio subsistema de usuarios, menús y permisos** (`UsuarioInstitucional`, `MenuInstitucional`, `PermisoUsuarioInstitucional`), paralelo al general.

### 5.4 Atención por WhatsApp (bot + asesores)

```mermaid
flowchart LR
    A["📱 Cliente escribe<br/>por WhatsApp"] --> B["Webhook<br/>(Meta o WATI)"]
    B --> C["WaService"]
    C --> D{"¿Quién atiende?"}
    D -->|Primera línea| E["🤖 Bot Claude<br/>(WaBotClaudeService)"]
    D -->|Escalado| F["👨‍💻 Asesor humano<br/>(wa-atencion en el front)"]
    F -->|"Sin respuesta en 10 min"| E
    E -.-> G["WaReasignacionService<br/>(HostedService que devuelve<br/>conversaciones al bot)"]
```

### 5.5 Gestión Humana

```mermaid
flowchart LR
    A["👤 Empleado"] --> B["Solicitud de permiso"]
    B --> C{"Autorización"}
    C -->|Aprueba / Rechaza| D["Resuelto"]
    A --> E["Incapacidades<br/>(tipos + resolución + reportes)"]
    A --> F["Vacaciones"]
    A --> G["Desprendibles de nómina"]
    H["👔 Líder"] --> I["Llamados de atención<br/>(tipos)"]
    J["Recursos administrativos:<br/>empleados, cargos, procesos"] -.-> A
```

### 5.6 Otros procesos activos

- **Evaluaciones / Escuela Farmy:** escuelas → módulos → contenidos → evaluaciones → progreso e intentos → reportes (avance, semestral) + buzón de convivencia.
- **Domicilios:** creación (normal y rápida), logística, consulta, entidades y clientes de domicilio.
- **Correos:** ingesta por API key → casos de correo → clasificación en el front (`clasificacion-correos`, `faltantes-correos`).
- **Precios:** carga master ETL, lotes de precios, homólogos, comparación de estados (`Up/Down/Equal`), stock.
- **Dashboard (Resumen):** compras, evolución, rotación, tiempos de entrega.

---

## 6. Seguridad y permisos (cómo funciona hoy)

```mermaid
flowchart LR
    A["Login<br/>(usuario + contraseña BCrypt)"] --> B["JwtService emite token"]
    B --> C["🍪 Cookie 'token'<br/>(no header Authorization)"]
    C --> D["Middlewares:<br/>SlidingToken (renueva)<br/>TokenConfigVersion (invalida)"]
    D --> E["Autorización por:<br/>• Menú (PermisoMenu)<br/>• Acción (RequiereAccionAttribute)<br/>• Rol (PermisoRol, RolAccion)"]
    E --> F["Front: AuthGuard +<br/>PermissionGuard + directiva hasAction"]
```

**Dato clave:** existen **tres sistemas de permisos** conviviendo:

1. **General:** `Usuario` + `Rol` + `PermisoUsuario/PermisoAccion/PermisoMenu` (+ `PermisoRol`, `RolAccion`).
2. **Institucional:** `UsuarioInstitucional` + `MenuInstitucional` + `PermisoUsuarioInstitucional`.
3. **Menús duplicados:** hay un modelo `Menu` en `Auth/Menus` y otro en `Modules/Menus`.

---

## 7. Radiografía en números

| Métrica | Valor |
|---|---|
| Módulos backend | 21 + Auth |
| Servicios en DI | ~70 (registro manual, sin interfaces) |
| Entidades EF | 60+ |
| Áreas del front | 26 |
| Componente más grande | `gestor-cotizacion` → **3.990 líneas** |
| Otros componentes >1.800 líneas | orden-compra (2.606), gestor-pedido (2.586), cotizaciones (2.140), despacho (2.050), recepción (1.881) |
| `Program.cs` | **2.108 líneas** (incluye seeds y utilidades dev) |
| Migraciones EF | 1 (+ `EnsureCreated` para desarrollo) |
| Tests | 0 en API; solo esqueleto en el front |

➡️ Continúa en: [**Propuesta de reestructuración**](02-propuesta-reestructuracion.md)
