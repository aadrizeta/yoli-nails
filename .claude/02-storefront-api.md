# 02 — Storefront API

Fecha: 1 de octubre de 2026.

## Conexión

- Endpoint: `POST https://nqpafx-ad.myshopify.com/api/2026-07/graphql.json`
- Cabecera `Content-Type: application/json`
- Autenticación (una de las dos):
  - Token **público**: `X-Shopify-Storefront-Access-Token: <token público>` (puede usarse en el navegador, pero se recomienda llamar desde el servidor).
  - Token **privado**: `Shopify-Storefront-Private-Token: <token privado>` (solo servidor).
- Cuerpo: `{ "query": "...", "variables": {...} }`

### Variables de entorno propuestas

```
SHOPIFY_STORE_DOMAIN=nqpafx-ad.myshopify.com
SHOPIFY_STOREFRONT_API_VERSION=2026-07
SHOPIFY_STOREFRONT_PRIVATE_TOKEN=   # servidor, nunca al cliente ni al repo
# Solo si algún día hay llamadas desde el navegador:
# NEXT_PUBLIC_SHOPIFY_STOREFRONT_PUBLIC_TOKEN=
```

Los valores de los tokens los proporciona el propietario del proyecto por un canal seguro. **No incluirlos en el repositorio.** El token privado se compartió en un documento de proyecto durante las pruebas y debe regenerarse en Shopify (canal Headless) antes de producción.

### Permisos y límites

- La Storefront API es de **solo lectura** para catálogo. No sirve para modificar productos (eso se hace por CSV o Admin API, ver `04`).
- `quantityAvailable` y `totalInventory` funcionan con el token actual (permiso de inventario concedido).
- Los metacampos solo se devuelven si su definición tiene **acceso Storefront**. Ya lo tienen `custom.lista_de_ingredientes` y `custom.indicaciones_de_uso`.
- Las colecciones solo se devuelven si están **publicadas en el canal Headless**.
- Coste observado: 17–122 puntos por query (`extensions.cost.requestedQueryCost`), muy por debajo de cualquier límite.
- Paginación: `first` máximo 250; con `pageInfo { hasNextPage endCursor }`. El catálogo cabe hoy en una página (`hasNextPage: false`).

## Queries

### Listado ligero (tarjetas) [PENDIENTE de ejecutar completa; todos los campos fueron verificados por separado]

```graphql
query ProductsList($first: Int!, $after: String) {
  products(first: $first, after: $after) {
    pageInfo { hasNextPage endCursor }
    nodes {
      id
      handle
      title
      vendor
      productType
      tags
      availableForSale
      featuredImage { url altText width height }
      priceRange { minVariantPrice { amount currencyCode } }
      compareAtPriceRange { minVariantPrice { amount currencyCode } }
    }
  }
}
```

Variables: `{ "first": 250, "after": null }`.

### Ficha de producto (completa) [PENDIENTE de ejecutar; la versión con `products(...)` está lista]

```graphql
query ProductByHandle($handle: String!) {
  product(handle: $handle) {
    id handle title description descriptionHtml vendor productType tags
    availableForSale totalInventory createdAt updatedAt publishedAt
    seo { title description }
    featuredImage { url altText width height }
    images(first: 10) { nodes { url altText width height } }
    priceRange {
      minVariantPrice { amount currencyCode }
      maxVariantPrice { amount currencyCode }
    }
    compareAtPriceRange {
      minVariantPrice { amount currencyCode }
      maxVariantPrice { amount currencyCode }
    }
    options { name values }
    variants(first: 10) {
      nodes {
        id title sku barcode availableForSale quantityAvailable currentlyNotInStock
        price { amount currencyCode }
        compareAtPrice { amount currencyCode }
        selectedOptions { name value }
        image { url altText }
      }
    }
    collections(first: 10) { nodes { handle title } }
    ingredientes: metafield(namespace: "custom", key: "lista_de_ingredientes") { type value }
    uso: metafield(namespace: "custom", key: "indicaciones_de_uso") { type value }
  }
}
```

Variables: `{ "handle": "crema-purifying-care" }`. Para recorrer todos los productos con el mismo conjunto de campos, usar `products(first: 50, after: $after) { pageInfo { hasNextPage endCursor } nodes { ...mismos campos... } }`.

### Colecciones y su contenido [VERIFICADO]

```graphql
query Collections {
  collections(first: 50) {
    nodes {
      handle
      title
      image { url altText }
      products(first: 50) { nodes { handle productType } }
    }
  }
}
```

Para una página de colección: `collection(handle: $handle) { title description products(first: 50) { nodes { ...campos de tarjeta... } } }`.

### Búsqueda por tag [VERIFICADO]

```graphql
query {
  products(first: 50, query: "tag:'momento:noche'") { nodes { handle } }
}
```

Devuelve exactamente los productos con ese tag (con las comillas simples alrededor del valor, necesarias por los dos puntos). Combinar con `AND` y `product_type:'Crema facial'` o `vendor:'Integra'` **no se ha probado todavía**.

### Carrito (de la investigación inicial) [PENDIENTE de probar; hace falta stock > 0]

```graphql
mutation CartCreate($lines: [CartLineInput!]) {
  cartCreate(input: { lines: $lines }) {
    cart { id checkoutUrl totalQuantity cost { totalAmount { amount currencyCode } } }
    userErrors { field message }
  }
}

mutation CartLinesAdd($cartId: ID!, $lines: [CartLineInput!]!) {
  cartLinesAdd(cartId: $cartId, lines: $lines) {
    cart { id totalQuantity cost { totalAmount { amount currencyCode } } }
    userErrors { field message }
  }
}

query GetCart($cartId: ID!) {
  cart(id: $cartId) {
    id checkoutUrl totalQuantity
    lines(first: 50) {
      nodes {
        id quantity
        merchandise { ... on ProductVariant {
          id title price { amount currencyCode }
          product { title handle featuredImage { url } }
        } }
      }
    }
    cost { totalAmount { amount currencyCode } subtotalAmount { amount currencyCode } }
  }
}
```

Análogas: `cartLinesUpdate`, `cartLinesRemove`. El `cartId` va en una cookie httpOnly. El pago lo gestiona Shopify en `cart.checkoutUrl`. **Con stock 0 el carrito rechazará las líneas**: para probarlo, dar stock a uno o dos productos de prueba.

## Hallazgos importantes

1. **[RIESGO] Filtrar por tag dentro de una colección no funciona** de entrada. `collection(handle: "cremas-faciales") { products(filters: [{ tag: "necesidad:antiedad" }]) }` devolvió las 10 cremas en lugar de las 5 con ese tag. Según la documentación y los foros de Shopify, los filtros de etiqueta, marca y tipo requieren activarse en la app **Shopify Search & Discovery** (Filters → Edit filters). No se ha instalado ni probado. Mientras tanto, filtrar en el servidor de Next.js (ver `03`).
2. `products(query: "tag:'x'")` **sí** filtra correctamente.
3. Dentro de una misma consulta, varios filtros de distinto tipo se combinan con AND y varios del mismo tipo con OR (según la documentación de Shopify), si se activan en Search & Discovery.
4. La categoría (`category { id name ancestors { name } }`) existe en la API y devuelve la taxonomía estándar de Shopify en inglés, pero **no se usa**.
5. Las respuestas incluyen `extensions.cost` útil para vigilar el coste.

## Caché y revalidación (resumen de la investigación)

- `fetch` con `next: { revalidate: 3600, tags: ['products'] }` para catálogo y colecciones.
- Webhooks de Shopify (`products/create`, `products/update`, `inventory_levels/update`, y `collections/update`) hacia un Route Handler propio que ejecute `revalidateTag('products')` (y `collections`). Requiere HTTPS público en el VPS.
- En un VPS la caché de datos de Next.js vive en disco/proceso: asegurar que persiste entre despliegues.
