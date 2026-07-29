# FARMY — Modelo del sistema y propuesta de reestructuración

Documentación generada a partir del análisis de los dos repositorios del sistema:

| Proyecto | Tecnología | Descripción |
|---|---|---|
| **PharmacyAPI** | ASP.NET Core 8 + EF Core 9 + SQL Server | API REST/OData con 21 módulos de negocio |
| **PharmacyFrontEnd** | Angular 20 + DevExtreme 25 + AG Grid | SPA con 26 áreas funcionales |

## Contenido

0. ⭐ [**Plan maestro de implementación**](docs/00-plan-maestro-implementacion.md) — el paso a paso consolidado (7 entregas) de costeo real, código PRE y OCR de facturas, listo para ejecutar.
1. [**Modelo actual del sistema**](docs/01-modelo-actual-farmy.md) — cómo está organizado Farmy hoy: arquitectura, módulos, procesos de negocio y seguridad, con diagramas.
2. [**Propuesta de reestructuración**](docs/02-propuesta-reestructuracion.md) — hallazgos estructurales detectados y plan de cambios por fases.
3. [**Mejora comercial: costeo real de cotizaciones**](docs/03-costeo-real-cotizaciones.md) — registrar el costo real de compra, adjuntar facturas y calcular la rentabilidad real de cotizaciones aprobadas.
4. [**Producto nuevo de punta a punta: código PRE y OCR de facturas**](docs/04-codigo-pre-y-ocr-facturas.md) — código provisional para productos sin inventario, su ciclo de vida (sin dejar basura), lectura OCR de facturas y semaforización.

> Los diagramas están en formato Mermaid y se renderizan directamente en GitHub.
