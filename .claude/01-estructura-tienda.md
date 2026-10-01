# 01 — Estructura de la tienda

Fecha: 1 de octubre de 2026. Fuente: respuestas reales de la Storefront API salvo donde se indica **[PENDIENTE]**.

## Marcas (`vendor`)

| Marca (valor exacto de `vendor`) | Productos | Colección (handle) |
|---|---|---|
| `D'LUCANI` | 16 | `dlucanni` |
| `Integra` | 10 | `integra-cosmetics` |
| `EME PROFESSIONAL SKINCARE` | 6 | `eme-professional-skincare` |

Notas: el `vendor` lleva apóstrofo (`D'LUCANI`). Hay una errata conocida de la marca (la forma correcta parece ser "D'Lucanni"); se corregirá en Shopify más adelante, incluido el handle `dlucanni`. **No codificar nombres de marca a mano**: tomarlos de `vendor` o de la colección. Títulos visibles de las colecciones: "D'Lucanni", "EME Professional Skincare", "Integra Cosmetics".

## Tipos de producto (`productType`) [VERIFICADO]

| Tipo | Nº |
|---|---|
| Crema facial | 10 |
| Sérum facial | 7 |
| Limpiador facial | 3 |
| Contorno de ojos | 3 |
| Booster facial | 2 |
| Loción tónica | 2 |
| Fotoprotector | 2 |
| Cuidado corporal | 2 |
| Bruma facial | 1 |

Son valores en singular. Cada producto tiene exactamente uno.

## Colecciones

Todas son automáticas (por reglas) excepto *Destacados*. La condición de las reglas por tipo es "Tipo de producto", **nunca** "Categoría de producto".

### Por tipo [VERIFICADO]

| Título | Handle | Regla | Nº |
|---|---|---|---|
| Limpiadores faciales | `limpiadores-faciales` | Tipo = Limpiador facial | 3 |
| Tónicos, lociones y brumas | `tonicos-lociones-y-brumas` | Tipo = Loción tónica O Bruma facial | 3 |
| Sérums y boosters | `serums-y-boosters` | Tipo = Sérum facial O Booster facial | 9 |
| Cremas faciales | `cremas-faciales` | Tipo = Crema facial | 10 |
| Contorno de ojos | `contorno-de-ojos` | Tipo = Contorno de ojos | 3 |
| Protección solar | `proteccion-solar` | Tipo = Fotoprotector (ver cambio previsto abajo) | 2 |
| Corporal | `corporal` | Tipo = Cuidado corporal | 2 |

### Por marca [VERIFICADO]

`dlucanni` (16), `eme-professional-skincare` (6), `integra-cosmetics` (10). Regla: Proveedor es igual a la marca.

### Por necesidad, piel y línea [PENDIENTE de verificar con la API]

Definidas con la condición "Etiqueta de producto es igual a":

| Título | Handle | Regla | Nº esperado |
|---|---|---|---|
| Antiedad | `antiedad` | `necesidad:antiedad` | 14 |
| Hidratación | `hidratacion` | `necesidad:hidratacion` | 9 |
| Acné y piel grasa | `acne-y-piel-grasa` | `necesidad:acne-piel-grasa` | 8 |
| Luminosidad y antimanchas | `luminosidad-y-antimanchas` | `necesidad:luminosidad-antimanchas` | 6 |
| Pieles sensibles | `pieles-sensibles` | `piel:sensible` O `necesidad:calmante` | 4 |
| Matipur | `matipur` | `linea:matipur` | 4 |

Cambio previsto: *Protección solar* (`proteccion-solar`) pasa a la regla `necesidad:solar`, con 3 productos (entra el After Sun).

### Destacados [PENDIENTE]

Colección **manual** para la home. Aún no existe; la clienta decidirá qué productos incluir. **No usar `frontpage`** ("Home page"): es la colección por defecto de Shopify y no se publica en la API.

### Navegación sugerida

Tres grupos: por **tipo** (7 colecciones), por **necesidad** (5: antiedad, hidratación, acné y piel grasa, luminosidad y antimanchas, pieles sensibles) y por **marca** (3), más Matipur como enlace de línea.

## Tags (taxonomía) [VERIFICADO en los 32 productos]

Cada producto tiene entre 2 y 6 tags. Solo se etiqueta lo que la descripción o el modo de uso respaldan; si un dato no consta, el tag no existe (no inferir). Valores en minúscula, sin acentos, con guiones. Orden alfabético (así los devuelve la API).

| Faceta | Valores (nº de productos) |
|---|---|
| `necesidad:` | `antiedad` (14), `hidratacion` (9), `acne-piel-grasa` (8), `luminosidad-antimanchas` (6), `calmante` (3), `solar` (3), `ojeras-bolsas` (2), `piernas-cansadas` (1) |
| `piel:` | `mixta-grasa` (7), `madura` (4), `sensible` (4), `seca-deshidratada` (1), `todo-tipo` (1) |
| `zona:` | `rostro` (27), `ojos` (3), `cuerpo` (2) |
| `momento:` | `dia-noche` (6), `dia` (4), `noche` (2) |
| `linea:` | `matipur` (4), `active` (2), `essential` (2), `purifying` (2), `special-c` (2), `vita-c` (2) |
| `atributo:` | `vegano` (2) |

Los tags son identificadores, no textos para mostrar: el frontend necesita un diccionario slug → etiqueta (ver `03`).

## Metacampos

| Uso | Namespace.key | Tipo | Notas |
|---|---|---|---|
| Ingredientes | `custom.lista_de_ingredientes` | `multi_line_text_field` | Acceso Storefront activo. Formato: ingredientes separados por `, `, sin punto final. Vacío (`null`) en 17 productos; ver `04`. |
| Modo de uso | `custom.indicaciones_de_uso` | `multi_line_text_field` | Puede contener saltos de línea (párrafos). `null` en 1 producto. |

No usar: `custom.ingredientes` (duplicado, vacío en los 32), `shopify.*` (atributos de categoría, no homogéneos).

## Variantes, imágenes y SEO

- Los 32 productos tienen **una sola variante** (opción "Title" / "Default Title"). No hace falta selector de variantes por ahora, pero el código debe tolerar varias.
- Cada producto tiene una imagen principal. El `altText` está vacío en muchos: usar el título como alternativa.
- Los campos SEO (`seo.title`, `seo.description`) están vacíos: usar título y descripción como alternativa.
- **[RIESGO]** `descriptionHtml` contiene restos de pegado: etiquetas `<meta charset>`, `<div class="rte-content ...">` y clases ajenas (`font-claude-response-body`, etc.). Sanear el HTML (quitar `meta`, `class`, `id`, `dir`) o usar `description` en texto plano.

## Precios y stock

- Moneda: EUR. El precio se lee de `priceRange.minVariantPrice`.
- **[RIESGO]** `Matipur Sérum Equilibrante y Seborregulador Anti-brillos` tiene precio `0.0` (error de datos pendiente de corregir). El frontend puede ocultar el precio si es 0.
- Stock 0 en todos los productos: `availableForSale: false`, `quantityAvailable: 0`, inventario sin "seguir vendiendo sin stock". Es intencional durante las pruebas.

## Catálogo completo (32 productos) [VERIFICADO]

| # | Handle | Título | Marca | Tipo | Precio € | Tags |
|---|---|---|---|---|---|---|
| 1 | `serum-antiedad-coenzima-q10` | Sérum Antiedad Coenzima Q10 | D'LUCANI | Sérum facial | 55.00 | `necesidad:antiedad`, `zona:rostro`, `atributo:vegano` |
| 2 | `gel-contorno-de-ojos-active` | Gel contorno de ojos Active | Integra | Contorno de ojos | 23.90 | `necesidad:hidratacion`, `necesidad:antiedad`, `zona:ojos`, `momento:dia`, `linea:active` |
| 3 | `crema-antimanchas-y-luminosidad-vita-c-booster` | Crema antimanchas y luminosidad Vita C Booster | D'LUCANI | Crema facial | 46.00 | `necesidad:luminosidad-antimanchas`, `piel:madura`, `zona:rostro`, `momento:noche`, `linea:vita-c` |
| 4 | `matipur-serum-sebo-regulador-anti-imperfecciones` | Matipur Sérum sebo-regulador anti-imperfecciones | D'LUCANI | Sérum facial | 46.00 | `necesidad:acne-piel-grasa`, `piel:mixta-grasa`, `zona:rostro`, `momento:dia-noche`, `linea:matipur` |
| 5 | `bruma-hidratante-magic-glow-mist` | Bruma hidratante Magic Glow Mist | Integra | Bruma facial | 14.90 | `necesidad:hidratacion`, `necesidad:luminosidad-antimanchas`, `zona:rostro`, `atributo:vegano` |
| 6 | `crema-multifuncion-special-c` | Crema multifunción Special C | Integra | Crema facial | 41.90 | `necesidad:antiedad`, `necesidad:luminosidad-antimanchas`, `zona:rostro`, `linea:special-c` |
| 7 | `serum-antioxidante-special-c` | Sérum Antioxidante Special C | Integra | Sérum facial | 29.90 | `necesidad:luminosidad-antimanchas`, `necesidad:antiedad`, `zona:rostro`, `linea:special-c` |
| 8 | `locion-equilibrante-triple-a` | Loción equilibrante triple A | D'LUCANI | Loción tónica | 23.90 | `necesidad:acne-piel-grasa`, `zona:rostro` |
| 9 | `sun-defense-locion-nutritiva-after-sun` | Sun defense Loción nutritiva After Sun | Integra | Cuidado corporal | 24.90 | `necesidad:solar`, `necesidad:calmante`, `piel:sensible`, `zona:cuerpo` |
| 10 | `serum-antiedad-con-retinol` | Sérum antiedad con retinol | D'LUCANI | Sérum facial | 59.90 | `necesidad:antiedad`, `zona:rostro` |
| 11 | `crema-relajante-felbo-relax-plus` | Crema relajante Felbo Relax Plus | D'LUCANI | Cuidado corporal | 39.90 | `necesidad:piernas-cansadas`, `zona:cuerpo` |
| 12 | `purifying-cleaner-limpiador-para-pieles-mixtas-y-grasas` | Purifying cleaner limpiador para pieles mixtas y grasas | EME PROFESSIONAL SKINCARE | Limpiador facial | 24.90 | `necesidad:acne-piel-grasa`, `piel:mixta-grasa`, `zona:rostro`, `momento:dia-noche`, `linea:purifying` |
| 13 | `contorno-de-ojos-miracle-eye-contour` | Contorno de Ojos Miracle Eye Contour | EME PROFESSIONAL SKINCARE | Contorno de ojos | 49.90 | `necesidad:ojeras-bolsas`, `necesidad:antiedad`, `zona:ojos` |
| 14 | `crema-essential-care` | Crema facial hidratante Essential Care | EME PROFESSIONAL SKINCARE | Crema facial | 39.90 | `necesidad:antiedad`, `necesidad:hidratacion`, `zona:rostro`, `momento:dia-noche`, `linea:essential` |
| 15 | `crema-purifying-care` | Crema facial hidratante purifying care | EME PROFESSIONAL SKINCARE | Crema facial | 30.90 | `necesidad:acne-piel-grasa`, `necesidad:hidratacion`, `piel:mixta-grasa`, `zona:rostro`, `momento:dia-noche`, `linea:purifying` |
| 16 | `serum-hialuronico-intensivo` | Sérum contorno de ojos con ácido hialurónico | EME PROFESSIONAL SKINCARE | Sérum facial | 59.90 | `necesidad:antiedad`, `necesidad:hidratacion`, `piel:madura`, `zona:rostro` |
| 17 | `leche-desmaquillante-hidratante-para-piel-adulta-y-deshidratada` | Leche Desmaquillante Hidratante para Piel Adulta y Deshidratada | Integra | Limpiador facial | 30.90 | `necesidad:hidratacion`, `piel:madura`, `piel:seca-deshidratada`, `zona:rostro` |
| 18 | `crema-de-dia-calmante-con-azuleno-para-piel-sensible` | Crema de Día Calmante con Azuleno para Piel Sensible | Integra | Crema facial | 29.90 | `necesidad:calmante`, `piel:sensible`, `zona:rostro`, `momento:dia` |
| 19 | `fotoprotector-solar-anti-edad-extreme-spf-50` | Fotoprotector Solar Anti-edad Extreme SPF 50 | Integra | Fotoprotector | 29.90 | `necesidad:solar`, `necesidad:antiedad`, `zona:rostro` |
| 20 | `glass-booster-alisador-anti-imperfecciones-y-minimizador-de-poros` | Glass Booster Alisador Anti-imperfecciones y Minimizador de Poros | D'LUCANI | Booster facial | 29.90 | `necesidad:acne-piel-grasa`, `necesidad:antiedad`, `piel:mixta-grasa`, `zona:rostro` |
| 21 | `booster-revitalizante-con-vitamina-c` | Booster Revitalizante con Vitamina C | D'LUCANI | Booster facial | 75.90 | `necesidad:luminosidad-antimanchas`, `zona:rostro`, `linea:vita-c` |
| 22 | `matipur-serum-equilibrante-y-seborregulador-anti-brillos` | Matipur Sérum Equilibrante y Seborregulador Anti-brillos | D'LUCANI | Sérum facial | 0.00 | `necesidad:acne-piel-grasa`, `piel:mixta-grasa`, `piel:sensible`, `zona:rostro`, `linea:matipur` |
| 23 | `protector-solar-coloreado-muy-alta-proteccion-spf-50` | Protector Solar Coloreado Muy Alta Protección SPF 50+ | D'LUCANI | Fotoprotector | 49.90 | `necesidad:solar`, `zona:rostro` |
| 24 | `crema-antiarrugas-y-reafirmante-active-cream` | Crema Antiarrugas y Reafirmante Active Cream | Integra | Crema facial | 28.00 | `necesidad:antiedad`, `zona:rostro`, `linea:active` |
| 25 | `crema-de-dia-hidratante-y-antioxidante-con-jalea-real` | Crema de Día Hidratante y Antioxidante con Jalea Real | Integra | Crema facial | 36.00 | `necesidad:hidratacion`, `necesidad:antiedad`, `zona:rostro`, `momento:dia` |
| 26 | `serum-facial-antioxidante-antipolucion-y-unificador-de-tono` | Sérum Facial Antioxidante, Antipolución y Unificador de Tono | D'LUCANI | Sérum facial | 49.90 | `necesidad:luminosidad-antimanchas`, `necesidad:antiedad`, `zona:rostro` |
| 27 | `locion-refrescante-excell-con-extracto-de-rosas-hamamelis-y-aloe-vera` | Loción refrescante Excell con Extracto de Rosas, Hamamelis y Aloe Vera | D'LUCANI | Loción tónica | 29.90 | `necesidad:hidratacion`, `piel:todo-tipo`, `zona:rostro` |
| 28 | `matipur-night-crema-de-noche-reparadora-y-regeneradora-anti-imperfecciones` | Matipur Night Crema de Noche Reparadora y Regeneradora Anti-imperfecciones | D'LUCANI | Crema facial | 21.90 | `necesidad:acne-piel-grasa`, `piel:mixta-grasa`, `zona:rostro`, `momento:noche`, `linea:matipur` |
| 29 | `matipur-day-crema-de-dia-equilibrante-y-matificante-anti-imperfecciones` | Matipur Day Crema de Día Equilibrante y Matificante Anti-imperfecciones | D'LUCANI | Crema facial | 21.90 | `necesidad:acne-piel-grasa`, `piel:mixta-grasa`, `zona:rostro`, `momento:dia`, `linea:matipur` |
| 30 | `contorno-de-ojos-k-lift-anti-ojeras-y-anti-bolsas` | Contorno de Ojos K-Lift Anti-ojeras y Anti-bolsas | D'LUCANI | Contorno de ojos | 44.50 | `necesidad:ojeras-bolsas`, `zona:ojos` |
| 31 | `essential-cleanser-limpiador-hidratante-para-pieles-sensibles` | Essential Cleanser Limpiador Hidratante para Pieles Sensibles | EME PROFESSIONAL SKINCARE | Limpiador facial | 24.90 | `necesidad:calmante`, `piel:sensible`, `zona:rostro`, `momento:dia-noche`, `linea:essential` |
| 32 | `exo-ionic-cream-revitalizador-mensajero-celular-triexosoma` | Exo IONIC Cream Revitalizador Mensajero Celular Triexosoma | D'LUCANI | Crema facial | 75.90 | `necesidad:antiedad`, `necesidad:hidratacion`, `piel:madura`, `zona:rostro`, `momento:dia-noche` |
