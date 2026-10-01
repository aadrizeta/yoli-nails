# YoliNails — Documentación para el frontend

Versión: 1 de octubre de 2026. Estado del catálogo: validado contra la Storefront API real (32 productos).

## Qué es el proyecto

Tienda online de cosmética facial profesional con 3 marcas y 32 productos. Arquitectura **headless**:

- **Backend de datos:** Shopify (catálogo, colecciones, carrito, checkout alojado).
- **API:** Storefront API (GraphQL, **solo lectura** para el catálogo; el carrito se crea con mutaciones). Versión fijada: `2026-07`.
- **Frontend:** Next.js (App Router), a construir.
- **Alojamiento:** VPS propio (no Vercel): `output: 'standalone'`, PM2 o systemd, Nginx como proxy inverso con TLS.

## Documentos

| Archivo | Contenido |
|---|---|
| `00-LEEME.md` | Este índice, decisiones clave y reglas |
| `01-estructura-tienda.md` | Marcas, tipos, colecciones (con reglas y handles), tags, metacampos y tabla de los 32 productos |
| `02-storefront-api.md` | Conexión, variables de entorno, queries validadas y hallazgos (qué funciona y qué no) |
| `03-frontend-arquitectura.md` | Rutas, tarjeta de producto, ficha, filtros, búsqueda, caché y despliegue |
| `04-historial-y-pendientes.md` | Qué se ha hecho, lecciones aprendidas, datos pendientes y decisiones abiertas |
| `catalogo-snapshot.json` | Foto del catálogo del 1-oct-2026 para maquetar sin llamar a la API (no es fuente de verdad) |

## Etiquetas de estado usadas en los documentos

- **[VERIFICADO]**: comprobado con respuestas reales de la Storefront API.
- **[PENDIENTE]**: definido pero sin comprobar todavía.
- **[RIESGO]**: algo que puede dar problemas al construir el frontend.

## Decisiones clave (no cambiar sin consultar)

1. **Marca** = `vendor`. **Tipo** = `productType` (lista cerrada de 9 valores). Los **tags con prefijo** (`necesidad:`, `piel:`, `zona:`, `momento:`, `linea:`, `atributo:`) describen para qué sirve el producto.
2. **No se usan** la categoría de Shopify (`category`) ni los metacampos `shopify.*`: varían según la categoría y no son homogéneos. Tampoco `custom.ingredientes` (duplicado vacío).
3. La ficha usa dos metacampos propios: `custom.lista_de_ingredientes` y `custom.indicaciones_de_uso` (texto plano multilínea).
4. Los listados usan una **query ligera**; la ficha, una **query completa por handle**. No se usa la query completa en los listados.
5. **Filtros del catálogo: en el servidor de Next.js**, sobre los productos ya cargados y cacheados, con los filtros en la URL. Los filtros por tag de `collection.products(filters: ...)` de la Storefront API **no funcionan** sin configurar Search & Discovery (ver `02`).
6. El **token privado** de la Storefront API vive solo en variables de entorno del servidor. Nunca en el cliente ni en el repositorio.
7. **Stock a 0 en todo el catálogo, a propósito** durante las pruebas. El frontend debe tratar bien el estado "Agotado".
8. El checkout es de Shopify: el frontend construye el carrito y redirige a `cart.checkoutUrl`.

## Datos de conexión

- Dominio de la tienda: `nqpafx-ad.myshopify.com`
- Versión de la API: `2026-07`
- Endpoint: `https://nqpafx-ad.myshopify.com/api/2026-07/graphql.json`
- Tokens: **no están en estos documentos**. Los proporciona el propietario del proyecto y van en `.env` (ver `02`).
