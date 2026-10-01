# 03 — Arquitectura del frontend (propuesta)

Fecha: 1 de octubre de 2026. Next.js App Router, TypeScript, Server Components por defecto.

## Rutas

| Ruta | Contenido | Datos |
|---|---|---|
| `/` | Home: Destacados, accesos por tipo y necesidad | `getCollection("destacados")`, colecciones |
| `/tienda` | Todo el catálogo con filtros y orden | `getProducts()` (ligera, cacheada) |
| `/coleccion/[handle]` | Productos de una colección (tipo, necesidad o marca) con filtros | `getCollectionProducts(handle)` |
| `/producto/[handle]` | Ficha completa | `getProduct(handle)` |
| `/buscar?q=` | Búsqueda de texto | `getProducts()` filtrada |
| `/carrito` | Carrito | `getCart()` |

Las marcas son colecciones (`/coleccion/dlucanni`, `/coleccion/integra-cosmetics`, `/coleccion/eme-professional-skincare`). Generar las rutas de producto con `generateStaticParams` a partir de los handles.

## Tarjeta de producto

Muestra: **nombre** (`title`), **marca** (`vendor`), **imagen** (`featuredImage`) y **precio** (`priceRange.minVariantPrice`). Además:

- Insignia "Agotado" si `availableForSale` es `false` (hoy será así en todos).
- Insignia "Vegano" si el tag `atributo:vegano` está presente.
- Si el precio es `0` o falta, no mostrar precio.
- `alt` de la imagen: `altText` o, si está vacío, el título.

Enlace a `/producto/[handle]`.

## Ficha de producto

Título, marca, galería (`images`), precio, estado de stock, botón de añadir al carrito (deshabilitado si no está disponible), descripción, **ingredientes**, **modo de uso**, chips de tags (necesidad, piel) enlazados a `/tienda?...`, productos relacionados (misma `linea:` o mismo tipo) y metadatos SEO.

Reglas de presentación:

- **Descripción**: sanear `descriptionHtml` (quitar `meta`, `class`, `id`, `dir`, `div` ajenos) o usar `description` en texto plano. Ver riesgo en `01`.
- **Ingredientes** (`ingredientes.value`): si es `null`, no pintar la sección. Si existe, es una lista separada por `, `: partir con `value.split(", ")` (no por coma sola: hay nombres con coma interna, como `1,2-Hexanodiol`) y mostrar como lista o como párrafo. Tres productos tienen solo un resumen, no una lista INCI completa (ver `04`).
- **Modo de uso** (`uso.value`): si es `null`, no pintar la sección. Respetar saltos de línea (`white-space: pre-line`); los párrafos se separan con línea en blanco.

## Datos: dos queries, no una

- **Listado ligero** (`ProductsList` en `02`): lo justo para tarjeta y filtros (incluye `tags`, `vendor`, `productType`). Se carga una vez, se cachea y alimenta `/tienda`, colecciones y búsqueda.
- **Ficha completa** (`ProductByHandle` en `02`): una llamada por producto, cacheada por handle.

No hace falta una query personalizada por filtro: los filtros operan sobre los datos ya cargados.

Estructura sugerida:

```
lib/shopify/client.ts        // shopifyFetch(query, variables, { revalidate, tags })
lib/shopify/queries.ts       // strings GraphQL
lib/shopify/products.ts      // getProducts, getProduct, getCollections, getCollectionProducts
lib/shopify/cart.ts          // cartCreate, cartLinesAdd, getCart
lib/catalog/labels.ts        // diccionario de tags y marcas
lib/catalog/filters.ts       // applyFilters, facetCounts, sortProducts
app/api/revalidate/route.ts  // webhook de Shopify
```

## Filtros (en el servidor de Next.js)

Con 32 productos, filtrar en memoria es lo más simple, rápido y robusto, y funciona con renderizado en servidor (SEO) sin depender de Search & Discovery.

**Estado en la URL** (compartible): `/tienda?necesidad=antiedad,hidratacion&piel=sensible&marca=integra-cosmetics&tipo=crema-facial&orden=precio-asc&q=vitamina`

- `necesidad`, `piel`, `zona`, `momento`: valores de los tags sin prefijo.
- `marca`: handle de la colección de marca (se traduce al `vendor`).
- `tipo`: slug del `productType`.
- `orden`: `precio-asc`, `precio-desc`, `nombre`.
- `q`: texto libre (título, marca y tags; opcional Fuse.js para tolerar erratas).

**Lógica**: dentro de una misma faceta, los valores se suman (OR); entre facetas, se restringe (AND).

```ts
type Filters = { necesidad?: string[]; piel?: string[]; zona?: string[]; momento?: string[]; marca?: string[]; tipo?: string[] };

export function applyFilters(products: Product[], f: Filters) {
  const hasTag = (p: Product, prefix: string, values?: string[]) =>
    !values?.length || values.some(v => p.tags.includes(`${prefix}:${v}`));
  return products.filter(p =>
    hasTag(p, "necesidad", f.necesidad) &&
    hasTag(p, "piel", f.piel) &&
    hasTag(p, "zona", f.zona) &&
    hasTag(p, "momento", f.momento) &&
    (!f.marca?.length || f.marca.includes(p.vendor)) &&
    (!f.tipo?.length || f.tipo.includes(p.productType))
  );
}
```

**Recuentos del panel**: para cada valor de cada faceta, contar los productos que cumplirían los filtros activos de las *otras* facetas (así el usuario ve cuántos resultados obtendría).

**Comportamiento de producto sin tag**: un producto sin tag de una faceta (por ejemplo, sin `piel:`) simplemente no aparece al filtrar por esa faceta. Es intencional: solo se etiqueta lo que consta.

**Cuándo cambiar de estrategia**: con cientos de productos, o si se quieren recuentos y filtros nativos, instalar Search & Discovery, activar los filtros y usar `collection.products(filters: ...)` (hoy no funciona sin eso; ver `02`). Los datos ya están estructurados para ese cambio.

## Diccionario de etiquetas (slug → texto)

```ts
export const TAG_LABELS: Record<string, string> = {
  "necesidad:antiedad": "Antiedad",
  "necesidad:hidratacion": "Hidratación",
  "necesidad:acne-piel-grasa": "Acné y piel grasa",
  "necesidad:luminosidad-antimanchas": "Luminosidad y antimanchas",
  "necesidad:calmante": "Calmante",
  "necesidad:solar": "Protección solar",
  "necesidad:ojeras-bolsas": "Ojeras y bolsas",
  "necesidad:piernas-cansadas": "Piernas cansadas",
  "piel:mixta-grasa": "Piel mixta o grasa",
  "piel:madura": "Piel madura",
  "piel:sensible": "Piel sensible",
  "piel:seca-deshidratada": "Piel seca o deshidratada",
  "piel:todo-tipo": "Todo tipo de piel",
  "zona:rostro": "Rostro",
  "zona:ojos": "Contorno de ojos",
  "zona:cuerpo": "Cuerpo",
  "momento:dia": "Día",
  "momento:noche": "Noche",
  "momento:dia-noche": "Día y noche",
  "linea:matipur": "Línea Matipur",
  "linea:active": "Línea Active",
  "linea:essential": "Línea Essential",
  "linea:purifying": "Línea Purifying",
  "linea:special-c": "Línea Special C",
  "linea:vita-c": "Línea Vita C",
  "atributo:vegano": "Vegano",
};
```

Los tipos (`productType`) y las marcas (`vendor`) ya vienen como texto legible desde la API.

Si en el futuro se filtra por `momento:`, recordar que `dia-noche` significa que el producto vale para ambos momentos: al filtrar "Día" conviene incluir también `dia-noche`.

## Stock y carrito

- Hoy todo está "Agotado" por decisión del proyecto. Diseñar el estado agotado como estado normal (no como error).
- Carrito: `cartId` en cookie httpOnly; Server Actions o Route Handlers para las mutaciones; redirigir a `cart.checkoutUrl` para pagar.
- Para probar el flujo completo, dar stock a 1–2 productos de prueba en Shopify.

## Despliegue en VPS

- `next.config.js` con `output: 'standalone'`; ejecutar con `node server.js` bajo PM2 o systemd.
- Nginx como proxy inverso con TLS (Let's Encrypt).
- Variables de entorno solo en el servidor (`.env` fuera del repo).
- Webhooks de Shopify apuntando al dominio público con HTTPS.
- Los ejemplos tipo "Next.js Commerce" están pensados para Vercel: sirven como referencia de queries y estructura, pero hay que adaptar la caché y la revalidación al modelo de servidor único.

## Qué no hacer

- No exponer el token privado en el cliente.
- No usar `category` ni los metacampos `shopify.*`.
- No depender de `collection.products(filters: [{ tag: ... }])` hasta activar Search & Discovery.
- No pintar `descriptionHtml` sin sanear.
- No pintar secciones de ingredientes o uso vacías.
- No codificar nombres de marca ni handles de colección de marca a mano (la marca D'Lucanni cambiará de handle).
