# 04 — Historial, lecciones y pendientes

Fecha: 1 de octubre de 2026.

## Qué se ha hecho (cronológico)

1. **Investigación inicial** de la Storefront API para Next.js en VPS (queries de colección, producto, búsqueda, carrito; arquitectura y despliegue).
2. **Validación de la API** con pruebas en Postman (el entorno de IA no pudo llamar a Shopify por restricciones de red): conexión, paginación, 32 productos, campos de producto, variantes y metacampos.
3. **Revisión de metacampos**: `custom.lista_de_ingredientes` e `custom.indicaciones_de_uso` accesibles desde la Storefront API (tipo texto multilínea).
4. **Unificación de tipos** (`productType`): de 6 valores incoherentes a 9 en singular (Crema facial, Sérum facial, Booster facial…). Aplicado por importación de CSV.
5. **Colecciones**: se detectó que dos fallaban porque su regla usaba "Categoría de producto" en lugar de "Tipo de producto". Corregido. Las 10 colecciones (7 por tipo, 3 por marca) devuelven los recuentos esperados.
6. **Tags con prefijo** (`necesidad:`, `piel:`, `zona:`, `momento:`, `linea:`, `atributo:`) aplicados a los 32 productos por importación de CSV y verificados.
7. **Formato de ingredientes y modo de uso**: separadores unificados, sin asteriscos ni puntos finales, textos de relleno vaciados, espacios y erratas de puntuación corregidos. Aplicado por CSV y verificado.
8. **Pruebas de filtrado**: la búsqueda por tag funciona; el filtro de tag dentro de colección no (requiere Search & Discovery).
9. **Diseño de colecciones por tags** (6 nuevas + cambio de *Protección solar*), pendiente de verificar.

## Cómo se modifica el catálogo (lecciones aprendidas)

La Storefront API es de solo lectura. Los cambios masivos se han hecho así:

1. En Shopify: Productos → Exportar → "Todos los productos" → CSV para Excel/Numbers.
2. Modificar **solo las columnas necesarias** con un script (no a mano) sobre ese export.
3. Importar con "Overwrite products with matching handles": primero un CSV de 2 productos de prueba, comprobar, y luego el completo.

Lecciones:

- Un CSV mínimo (`Handle`, `Title`, `Type`) **falla**: Shopify exige el formato completo del export ("Debes especificar el título"; después, "Se requiere la entrada de opciones de producto cuando actualizas variantes").
- La columna `Tags` del import **sustituye** todos los tags del producto. Exportar de nuevo antes de reimportar si hubo cambios manuales.
- Las columnas de metacampos se importan con cabeceras `Nombre (product.metafields.namespace.key)`.
- El tipo de una colección (manual/automática) no se puede cambiar después de crearla.
- Para escribir por API haría falta la Admin API: desde el 1 de enero de 2026 ya no se crean apps personalizadas con token permanente desde el admin; se usa el Dev Dashboard con *client credentials* (token de 24 h). No está montado.

## Estado de los datos de producto

### Ingredientes (`custom.lista_de_ingredientes`)

- Lista INCI completa y formateada: **12 de 32** productos.
- Solo un resumen, no una lista INCI (3): 

- `crema-antimanchas-y-luminosidad-vita-c-booster`
- `matipur-serum-sebo-regulador-anti-imperfecciones`
- `serum-antiedad-con-retinol`

- Sin datos (17; incluye 3 que tenían texto de relleno):

- `gel-contorno-de-ojos-active`
- `bruma-hidratante-magic-glow-mist`
- `crema-multifuncion-special-c`
- `serum-antioxidante-special-c`
- `sun-defense-locion-nutritiva-after-sun`
- `crema-relajante-felbo-relax-plus`
- `leche-desmaquillante-hidratante-para-piel-adulta-y-deshidratada`
- `crema-de-dia-calmante-con-azuleno-para-piel-sensible`
- `fotoprotector-solar-anti-edad-extreme-spf-50`
- `glass-booster-alisador-anti-imperfecciones-y-minimizador-de-poros`
- `booster-revitalizante-con-vitamina-c`
- `matipur-serum-equilibrante-y-seborregulador-anti-brillos`
- `protector-solar-coloreado-muy-alta-proteccion-spf-50`
- `crema-antiarrugas-y-reafirmante-active-cream`
- `crema-de-dia-hidratante-y-antioxidante-con-jalea-real`
- `matipur-night-crema-de-noche-reparadora-y-regeneradora-anti-imperfecciones`
- `contorno-de-ojos-k-lift-anti-ojeras-y-anti-bolsas`

- Las listas de `crema-essential-care` y `purifying-cleaner-limpiador-para-pieles-mixtas-y-grasas` son **idénticas**; una de las dos es incorrecta (verificar con el envase).
- En los tres resúmenes, el formato aplicó mayúscula a cada elemento, lo que queda raro en un texto que no es una lista; corregir al cargar los datos reales.
- No se ha traducido ni alterado ningún nombre de ingrediente (conviven formas como `Propanediol` y `Propanodiol`).

### Modo de uso (`custom.indicaciones_de_uso`)

- Falta en `serum-hialuronico-intensivo`.
- El tono de las instrucciones es mixto (infinitivo, "aplicamos", "repita", "aplícala"). Unificarlo es una reescritura, no hecha.

### Otros datos

- **Precio 0,00 €** en `matipur-serum-equilibrante-y-seborregulador-anti-brillos`.
- **Posible duplicado**: `matipur-serum-equilibrante-y-seborregulador-anti-brillos` y `matipur-serum-sebo-regulador-anti-imperfecciones` parecen el mismo producto.
- **Título incorrecto probable**: `serum-hialuronico-intensivo` se titula "Sérum contorno de ojos con ácido hialurónico", pero su descripción es de sérum facial y otro producto lo cita como "Serum Hyaluronic Intensive". También carece de modo de uso.
- `crema-relajante-felbo-relax-plus` se titula "Crema" pero su descripción dice "Gel corporal".
- Erratas en descripciones (ejemplos): "anti-envegecimiento", "hidratante hidratante", "apacidad", "considerablementedesde", "lineas", "Exo~".
- Los títulos no siguen una convención (mayúsculas, orden de nombre de línea y tipo).
- `altText` de imágenes vacío en muchos productos; SEO vacío.
- Errata de marca `D'LUCANI` / `DLUCANNI` / `D'Lucanni` y handle `dlucanni`: se corregirá en Shopify.

## Decisiones abiertas

1. **Filtros del frontend**: servidor de Next.js (recomendado hoy) o Search & Discovery (si el catálogo crece o se quieren recuentos nativos).
2. **Selección de *Destacados*** para la home (la decide la clienta).
3. **Verificar** las colecciones por tags y el cambio de *Protección solar* con esta query:

```graphql
query TagCollectionsCheck($first: Int!) {
  collections(first: 50) {
    nodes { handle products(first: $first) { nodes { handle } } }
  }
}
```

   Recuentos esperados: `antiedad` 14, `hidratacion` 9, `acne-y-piel-grasa` 8, `luminosidad-y-antimanchas` 6, `pieles-sensibles` 4, `matipur` 4, `proteccion-solar` 3; las 10 anteriores sin cambios.
4. **Stock** para probar el carrito y el checkout.
5. **Regenerar el token privado** antes de producción.
6. **Ejecutar y validar** las queries `ProductsList` y `ProductByHandle` de `02` (no se ha pegado aún su respuesta).
7. Probar combinaciones de búsqueda (`tag:... AND product_type:...`).
