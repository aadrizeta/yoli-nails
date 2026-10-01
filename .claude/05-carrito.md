# 05 · Carrito (Shopify Storefront API + Next.js)

Estado: [VERIFICADO en docs de Shopify] = leído en documentación oficial; [PENDIENTE de probar] = no ejecutado contra la tienda (la sandbox no llega a myshopify.com; hace falta que Adrian lo pruebe en Postman o en local). **Nunca escribir tokens en este documento.**

## 1. Cómo funciona el carrito en Shopify

- El carrito vive **en Shopify**, no en nuestro servidor. Lo creamos y modificamos con mutations de la Storefront API; el frontend solo guarda el **ID del carrito**. [VERIFICADO]
- El ID tiene la forma `gid://shopify/Cart/<token>?key=<secreto>`. Hay que guardarlo **completo, con el `?key=`**. Quien tenga ese ID puede leer y modificar el carrito: se trata como una contraseña. [VERIFICADO]
- Caducidad: los carritos sin usar caducan a los 30 días de su creación; se borran al completar el checkout. Máximo 500 líneas por carrito, y se pueden crear carritos sin límite. [VERIFICADO]
- El **pago no se hace en nuestro frontend**: el carrito devuelve `checkoutUrl` y se redirige al cliente a esa URL, que es el checkout alojado por Shopify. [VERIFICADO]
- El carrito solo se puede comprar si las variantes tienen stock disponible (o permiten vender sin stock). Ahora todo está a stock 0 a propósito, así que para probar hay que darle stock a 1–2 variantes. [PENDIENTE de probar]

## 2. Mutations y queries (resumen)

| Operación | Para qué | Notas |
|---|---|---|
| `cartCreate(input)` | Crear carrito (con o sin líneas) | Devuelve `cart`, `userErrors`, `warnings` |
| `cart(id)` | Leer el carrito | Query, no mutation |
| `cartLinesAdd(cartId, lines)` | Añadir productos | `CartLineInput`: `merchandiseId` (obligatorio, ID de **variante**), `quantity` (por defecto 1), `sellingPlanId`, `attributes`. Hasta 250 líneas por llamada |
| `cartLinesUpdate(cartId, lines)` | Cambiar cantidad | Lleva el `id` de la **línea** (no de la variante) y `quantity` |
| `cartLinesRemove(cartId, lineIds)` | Quitar líneas | Recibe los IDs de línea |
| `cartDiscountCodesUpdate` | Código de descuento | Sustituye la lista de códigos |
| `cartNoteUpdate` / `cartAttributesUpdate` | Nota del pedido / atributos personalizados | Opcionales |
| `cartBuyerIdentityUpdate` | Email, país, y `customerAccessToken` para checkout con cuenta | Solo si hay login de clientes |
| `cartMetafieldsSet`, `cartDeliveryAddresses*` | Metacampos y direcciones | No los necesitamos de momento |

Los nombres de las mutations `cartLinesRemove`, `cartDiscountCodesUpdate`, `cartNoteUpdate` y `cartAttributesUpdate` los he confirmado por el patrón de la API y la referencia de Shopify, pero su firma exacta conviene validarla en el primer test. [PENDIENTE de probar]

### Errores y avisos

- `userErrors { field message code }`: la operación **falló** (variante inválida, cantidad inválida…). Siempre hay que revisarlos. [VERIFICADO]
- `warnings { code message target }`: la operación **funcionó pero Shopify ajustó algo**. Ejemplo documentado: `MERCHANDISE_OUT_OF_STOCK` (un artículo ya no tiene stock); el `target` es el ID de la línea afectada, que se puede usar para llamar a `cartLinesRemove`. La lista completa de códigos no la he podido ver (la página la oculta). [VERIFICADO parcialmente]

## 3. Fragmento reutilizable

```graphql
fragment CartFields on Cart {
  id
  checkoutUrl
  totalQuantity
  note
  cost {
    subtotalAmount { amount currencyCode }
    totalAmount { amount currencyCode }
  }
  lines(first: 100) {
    nodes {
      id
      quantity
      cost { totalAmount { amount currencyCode } }
      merchandise {
        ... on ProductVariant {
          id
          title
          availableForSale
          price { amount currencyCode }
          image { url altText width height }
          product { handle title vendor }
        }
      }
    }
  }
}
```

## 4. Operaciones

```graphql
mutation CartCreate($lines: [CartLineInput!]) {
  cartCreate(input: { lines: $lines }) {
    cart { ...CartFields }
    userErrors { field message code }
    warnings { code message target }
  }
}

mutation CartLinesAdd($cartId: ID!, $lines: [CartLineInput!]!) {
  cartLinesAdd(cartId: $cartId, lines: $lines) {
    cart { ...CartFields }
    userErrors { field message code }
    warnings { code message target }
  }
}

mutation CartLinesUpdate($cartId: ID!, $lines: [CartLineUpdateInput!]!) {
  cartLinesUpdate(cartId: $cartId, lines: $lines) {
    cart { ...CartFields }
    userErrors { field message code }
    warnings { code message target }
  }
}

mutation CartLinesRemove($cartId: ID!, $lineIds: [ID!]!) {
  cartLinesRemove(cartId: $cartId, lineIds: $lineIds) {
    cart { ...CartFields }
    userErrors { field message code }
    warnings { code message target }
  }
}

query GetCart($cartId: ID!) {
  cart(id: $cartId) { ...CartFields }
}
```

Variables de ejemplo para probar en Postman (el `merchandiseId` sale de `variants.nodes[].id` de cualquier producto):

```json
{ "lines": [{ "merchandiseId": "gid://shopify/ProductVariant/XXXX", "quantity": 1 }] }
```

## 5. Implementación en Next.js (App Router, VPS)

Principios:

1. **Todo el carrito se maneja en el servidor** (Server Actions). El token de la Storefront API no sale al navegador.
2. El `cartId` se guarda en una **cookie httpOnly**, `secure`, `sameSite: 'lax'`, caducidad ~30 días (igual que el carrito). El navegador nunca lo lee por JavaScript.
3. Las lecturas del carrito **no se cachean** (`cache: 'no-store'`): es dato por usuario. Al contrario que el catálogo, que sí usa `revalidate` y tags.
4. Tras cada acción se llama a `revalidatePath('/', 'layout')` o se devuelve el carrito actualizado para refrescar la interfaz.
5. Pago: botón "Finalizar compra" que redirige a `cart.checkoutUrl`.
6. Al hacer peticiones desde el servidor, Shopify recomienda enviar la cabecera `Shopify-Storefront-Buyer-IP` con la IP del cliente (importante para el checkout autenticado y para evitar bloqueos por rate-limit). En Nginx hay que reenviar `X-Forwarded-For`. [VERIFICADO que existe; su necesidad sin login de cliente es PENDIENTE de confirmar]

Estructura sugerida:

```
lib/shopify/client.ts      // shopifyFetch (ya existente para catálogo)
lib/shopify/cart.ts        // createCart, addLines, updateLine, removeLines, getCart
lib/shopify/queries/cart.ts
app/actions/cart.ts        // 'use server': addToCart, updateQty, removeItem, goToCheckout
components/cart/AddToCartButton.tsx   // client, useActionState / useOptimistic
components/cart/CartDrawer.tsx        // lee el carrito en servidor
```

`lib/shopify/cart.ts` (esquema):

```ts
import { cookies } from 'next/headers';
const COOKIE = 'cartId';

export async function getCartId() {
  return (await cookies()).get(COOKIE)?.value;
}

export async function setCartId(id: string) {
  (await cookies()).set(COOKIE, id, {
    httpOnly: true, secure: true, sameSite: 'lax',
    path: '/', maxAge: 60 * 60 * 24 * 30,
  });
}

export async function getCart() {
  const id = await getCartId();
  if (!id) return null;
  const data = await shopifyFetch({ query: GET_CART, variables: { cartId: id }, cache: 'no-store' });
  return data.cart ?? null;   // null si el carrito caducó o se compró
}

export async function addToCart(variantId: string, quantity = 1) {
  const id = await getCartId();
  const lines = [{ merchandiseId: variantId, quantity }];
  if (!id || !(await getCart())) {
    const { cartCreate } = await shopifyFetch({ query: CART_CREATE, variables: { lines }, cache: 'no-store' });
    check(cartCreate);
    await setCartId(cartCreate.cart.id);
    return cartCreate.cart;
  }
  const { cartLinesAdd } = await shopifyFetch({ query: CART_LINES_ADD, variables: { cartId: id, lines }, cache: 'no-store' });
  check(cartLinesAdd);
  return cartLinesAdd.cart;
}
// check(): si userErrors.length -> throw; si warnings -> devolverlos a la UI
```

Casos que hay que cubrir:

- **Carrito inexistente o caducado**: `cart(id)` devuelve `null` → crear uno nuevo y sobrescribir la cookie.
- **Compra completada**: Shopify borra el carrito → mismo tratamiento (`null`).
- **Sin stock**: ahora mismo todo está a 0. Mostrar "Agotado" con `availableForSale` y desactivar el botón; además gestionar `userErrors`/`warnings` por si falla una acción.
- **Cantidad**: en `cartLinesUpdate` con `quantity: 0` la línea se elimina; aun así se usa `cartLinesRemove` para ser explícitos.
- **Contador del header**: `totalQuantity`, leído en el layout del servidor.
- **Moneda y precios**: siempre `amount` + `currencyCode` del carrito; no calcular totales en el frontend.
- **País/idioma**: la directiva `@inContext(country: ES, language: ES)` en queries y mutations fija la localización del precio y del checkout. Solo hace falta si se activan Markets. [PENDIENTE de decidir]
- **Seguridad**: el `cartId` nunca en la URL, en logs ni en localStorage.

## 6. Orden de pruebas recomendado

1. Dar stock (p. ej. 5) a 1–2 variantes en el admin.
2. En Postman: `CartCreate` con una variante → copiar el `cart.id` y el `checkoutUrl`.
3. `CartLinesAdd` con otra variante, `CartLinesUpdate` (cantidad 2) y `CartLinesRemove`.
4. Abrir el `checkoutUrl` en el navegador y confirmar que el checkout muestra los artículos (sin pagar).
5. Probar una variante sin stock para ver el `userError` o `warning` que devuelve y fijar el mensaje de la interfaz.
6. Pegar las respuestas en la conversación para marcar este documento como [VERIFICADO].

## Fuentes

- https://shopify.dev/docs/storefronts/headless/building-with-the-storefront-api/cart
- https://shopify.dev/docs/storefronts/headless/building-with-the-storefront-api/cart/manage
- https://shopify.dev/docs/storefronts/headless/building-with-the-storefront-api/cart/cart-warnings
- https://shopify.dev/docs/api/storefront/latest/mutations/cartLinesAdd
- https://shopify.dev/docs/api/storefront/latest/objects/CartWarning
