---
name: "picsil-amazon"
description: "Skill de optimización de Amazon para PICSIL (equipamiento cross training, marketplaces EU). Usar SIEMPRE que la tarea toque: campañas/keywords/bids/negativas de Amazon Ads, SEO de listings (títulos, bullets, backend keywords), generación de propuestas para ads_proposals o listing_proposals, análisis de search terms o SQP, consolidación de ASINs, o validación multi-idioma de listings. Contiene el motor de decisión V3 (jerarquía de reglas, clasificación de targeting, evidencia cuantificada, break-even con fees reales) y las reglas innegociables del sistema."
---

# PICSIL Amazon — Skill de optimización (motor V3, act. 24-ago-2026)

Frameworks adaptados de nexscope-ai/Amazon-Skills (MIT) + reglas propias + AJUSTES_INPUT_V3 validado con el equipo de Amazon. Complementa a CLAUDE.md y PRD.md del repo — si algo aquí los contradice, mandan CLAUDE.md y PRD.md.

## 0. JERARQUÍA GLOBAL DE REGLAS (V3)

Cuando dos reglas choquen, gana la de nivel superior:
1. Protección/Compliance (marca, términos protegidos) → 2. Integridad estructural y dependencias → 3. Evidencia y suficiencia de datos → 4. Protección estratégica de marca (campañas BRAND/MARCA/DEFENSE) → 5. Rentabilidad (objetivo por categoría) → 6. Crecimiento → 7. Formato.
Ejemplos: crecimiento dice subir puja pero hay datos insuficientes → gana datos insuficientes. Rentabilidad dice negativizar pero el término es PICSIL → gana protección.

## 1. Principios que gobiernan toda decisión

1. **TACOS por producto/marketplace, nunca ACOS de campaña.** El ACOS objetivo se deriva del margen del producto (`product_costs`, con coste real por producto cargado del Excel CALCULO COSTES desde 17-ago-2026). Recortar ads que sostienen rank orgánico "para mejorar ACOS" está prohibido.
2. **Toda salida es una propuesta, no una acción.** Filas compatibles con `ads_proposals`/`listing_proposals`, status='pending', Human Approval siempre. El PVP no se toca jamás.
3. **Datos completos o nada.** Paginación agotada, reconciliación campaña↔targets ±5%, estructura+rendimiento juntos. Ante datos truncados: alerta, no propuesta.
4. **Los reports NO son el inventario de entidades.** Un report solo devuelve lo que tuvo impresiones: una keyword ENABLED con 0 impresiones no aparece. Toda propuesta que **cree** algo (graduate_term, negativas) o que dependa del estado de una entidad debe cruzarse contra el endpoint de estructura (`/sp/keywords/list`, `/sp/targets/list`, `/sb/keywords`…) ANTES de insertarse. Ignorar esto produjo la tanda de duplicados del 20-ago y del 24-ago.
5. **EU multi-idioma.** Nunca traducir keywords: investigar términos locales reales (IT: "magnesite" vs "magnesio", "paracalli" = calleras; NL: "springtouw"; SE: "hopprep"; PL: "skakanka", "skórki").

## 2. Clasificación del targeting (4 categorías, ANTES de cualquier recomendación)

- **MARCA/MODELO** — según tabla `brand_protected_terms` (Supabase, 26 términos por niveles).
- **COMPETIDOR** — velites, wodies, rx smart gear (combas), gornation, bear grips, versa gripps...
- **GENÉRICO** — categoría/uso/material sin marca.
- **INDETERMINADO** — fallback de TODA duda. Sin acciones automáticas: ni negativizar, ni pausar, ni bajar puja. Solo revisión manual. (Genérico NUNCA es fallback.)

### Niveles de protección de marca (brand_protected_terms)
- **brand**: picsil + typos (picsl, picsill, pic sil, piksil, pcsl...) → protegido SIEMPRE.
- **distinctive**: neoboost, lockpro, insigne, azor, hex tech, nova01, ingot, heron, grain phantom → protegidos solos o combinados.
- **model_context**: falcon, hawk, condor, phoenix, golden, rx, grain, phantom, sphinx, rook, bee, maverick, horizon → protegidos SOLO junto a palabras de SU familia (en el idioma del mercado) o junto a la marca. "falcon grips" → protegido; "falcon" solo → INDETERMINADO.
- **brand_only**: abs, lumbar, tactical → protegidos SOLO pegados a picsil. "abs jump rope" → GENÉRICO.
- **Cruce de familias**: modelo + familia equivocada NO se protege ("rx grips" = Picsil; "rx jump rope" = RX Smart Gear → COMPETIDOR).

### Qué se puede hacer con términos protegidos
Orden: 1) mantener; 2) pausa de targeting o ajuste de segmentación; 3) revisión manual; 4) negativizar JAMÁS de forma automática. Campañas BRAND/MARCA/DEFENSE con bajo rendimiento: solo revisión manual y/o test gradual de puja.
**Sobrepuja de marca**: si puja ≥3× CPC realizado con Top-of-Search ~100% → señalar "posible sobrepuja en defensa" y recomendar −5% gradual (nunca pausar/negativizar por esto). Trigger = puja vs CPC realizado + ToS, NO el ROAS.
**Recorte encubierto (prohibido)**: nunca proponer crear/graduar una keyword de marca con una puja INFERIOR a la que ya está viva para ese mismo texto y match type. Si la entidad ya existe, la única lectura válida es mantener o subir; bajar exige un `bid_down` explícito con su evidencia.

## 3. Evidencia cuantificada (dos niveles)

| Acción | Mínima (Confianza Baja/Media) | Robusta (acción aplicable) | Guardarraíl fijo |
|---|---|---|---|
| bid_down | ≥10 clics o ≥1000 impr 30-60d | ≥20 clics (CVR real) o ≥2000 impr | Nunca al único convertidor; nunca <100 impr/30d |
| bid_up | ACOS < objetivo×0,7 con ≥5 clics | ≥10 clics + recomendación Amazon leída | Cap Max CPC siempre |
| negate_term | ≥12 clics, 0 pedidos, 60d | + 0 ventas en 90d Y gasto outlier (>mediana campaña) | Jamás protegidos; en campañas "LowBid" → pausa, no negativa |
| graduate_term | ≥2 pedidos y ACOS < objetivo×1,2 | + ≥3 clics para CPC base fiable | **Comprobar `/sp/keywords/list` del ad group destino ANTES de proponer.** Bid = sugerida Amazon+0,01 o CPC medio×1,1 |
| budget_change | >80% consumo recurrente y rentable | + oportunidad perdida (report budget Amazon) | Subida +15/20% |
| placement_adjust | rentabilidad diferencial clara 30d | ≥2 pedidos en el emplazamiento a subir | Array de REEMPLAZO COMPLETO |

Bajo mínima → DATOS_INSUFICIENTES (no proponer). Entre mínima y robusta → Confianza: Baja + Revisión manual. Categoría×producto con <10 clics/60d: sin base, no proponer pujas y decirlo.

## 4. Objetivo económico por categoría

Objetivo = ROAS real 60d por **categoría × mercado × producto publicitario (SP/SB/SD)** × 1,12, techo = break-even ACOS del producto (flag `categoria_bajo_breakeven`). SB y SD nunca se juzgan con el objetivo de SP. Fuente de verdad: `product_costs` (coste real por EAN) + fees reales SP-API. Anotar siempre: categoria, ad_product, roas_real_60d, roas_objetivo, target_acos, target_acos_source.

**El Max CPC también es por producto publicitario.** Un `max_cpc_*` calculado con la economía y el CVR de SP aplicado a una keyword de SB bloquea propuestas legítimas: el guardarraíl del aplicador lo lee y rechaza cualquier `bid_up` que lo supere. Calcular el Max CPC con el CVR y el ROAS reales del producto publicitario al que pertenece la entidad. Caso vivo: SB IE `crossfit grips`, puja 2,50 € contra un Max CPC de 0,48 € heredado de SP.

### Break-even con fees reales (referencia 17-ago-2026, ANEXO V3)
BE_ACOS = (PVP/(1+IVA) − referral − FBA − coste_total)/PVP. IVA: NL/BE 21%, IE/PL 23%, SE 25%.
- **Referral: 15% en NL/BE/IE/SE y 10% en PL** (verificado vía API — PL es el mercado con más margen publicitario: BE 46-56%).
- **IE**: FBA anormalmente bajo (0,78-4,31 €) → favorable a escalar. **SE**: el mercado más caro (FBA 2,8-7,5 € eq.), BE 3-6 puntos peor que NL/BE.
- BE por familia (NL/BE/IE/SE/PL aprox): GRIPS 38/38/41/35/49 · JUMP ROPES 32/32/38/29/46 · WRAPS 34/35/45/31/56 · BELTS 34/35/39/32/53 · KNEE SLEEVES 41/41/43/38/49 · STRAPS 36/37/42/31/50 · CHALK 22/n.d./n.d./16/49 · BACKPACKS 43/43/43/41/50 (%).
- **Familias frágiles**: BE <20% en un mercado (hoy CHALK en SE) → sin bid_up salvo defensa de marca. Textil y calzado NO se venden en Amazon.

## 5. Pujas: límites, redondeo y cooldown

- **Tope ±25% por ciclo** (guardarraíl de servidor). **El tope se valida DESPUÉS de redondear: redondeo siempre hacia dentro** (0,55×0,75=0,4125 → proponer 0,42, no 0,41 — causa real de 3 bloqueos el 17-ago).
- Gradualidad: puja dentro del rango sugerido de Amazon o ACOS cerca del objetivo → ±12%. ACOS a <5 puntos del objetivo → no proponer nada.
- Puja propuesta = máxima sugerida de Amazon + 0,01, cap en Max CPC del producto; sin recomendación → rango prudente 0,50-0,60 € (o equivalente PLN/SEK).
- Max CPC = PVP × margen × CVR, **con el CVR del producto publicitario de la entidad** (ver §4). CVR real si ≥20 clics/60d; si no, proxy de categoría. Anotar `cvr_source`. **Claves EXACTAS del rationale: `max_cpc_eur` / `max_cpc_pln` / `max_cpc_sek` / `max_cpc`** (el guardarraíl lee esas; `max_cpc_product_eur` lo deja inerte).
- **Cooldown por medición cerrada**: entidad con fila `status='measuring'` en `ads_measurements` → intocable hasta veredicto D14 humano (no basta la fecha). Excepción: negativizaciones por watchlist_note.
- 0 impresiones o <100 impr/30d: prohibido bid_down mecánico. Solo bid_to_visibility (si puja < 50% del sugerido bajo) o nada.
- **entity_id obligatorio** antes de insertar propuestas de puja/pausa (keywordId/targetId real resuelto por API). Si no es resoluble (p. ej. THEME de SB), marcar `requiere_id_manual` y Revisión manual.
- **`ad_group_id` obligatorio en TODA propuesta de SB** (puja, pausa, negativa): el aplicador lo necesita porque Amazon exige `adGroupId` en el cuerpo de `PUT /sb/keywords` y `PUT /sb/targets`. Sin él, la propuesta se puede aplicar solo si el aplicador consigue resolverlo leyendo el recurso.
- **Sesgo simétrico**: buscar bid_up activamente (ACOS < objetivo×0,7 con impression share bajo; ToS share <40% en keywords que convierten). Si solo salen bajadas, justificar que es real.

## 6. Emplazamientos

Habitual +10 a +25 puntos; máximo estándar +50 por acción; >50 solo revisión manual. Prohibidos saltos 0→100 sin datos. Array de REEMPLAZO COMPLETO (incluir SIEMPRE el estado actual completo en el rationale). SP: PLACEMENT_TOP / PLACEMENT_PRODUCT_PAGE / PLACEMENT_REST_OF_SEARCH (report spCampaigns con `placementClassification` obligatorio). SB: TOP_OF_SEARCH / HOME / DETAIL_PAGE / OTHER (sin report de rendimiento por emplazamiento → solo evidencia indirecta → Confianza Baja). SD: no tiene emplazamientos.

## 7. Presupuestos

- Subir +15/20% solo con >80% consumo recurrente Y rentabilidad contra objetivo. Señal principal: report de budget de Amazon (oportunidad perdida). No subir si el problema es puja/relevancia. No reducir drásticamente en BRAND/MARCA/DEFENSE.
- **Formato: `proposed_value.budget` es NÚMERO plano en los tres productos** (el aplicador lo lee con Number(); un objeto anidado envía null a Amazon).
- Recortes de control de riesgo (utilización <5%) chocan con el tope ±25%: el aplicador los BLOQUEA (no los recorta en silencio) y requieren force + force_reason.

## 8. Escalado — módulo de crecimiento (buscar activamente las 4)

Scale Bid (rentable + buena conversión + impression share bajo → bid_up) · Scale Budget (rentable + limitada por presupuesto → budget_change) · Scale Placement (emplazamiento especialmente rentable → placement_adjust) · Scale Coverage (search term rentable sin exacta propia → graduate_term).

## 9. Migraciones y dependencias

- **Pre-chequeo obligatorio de existencia (graduate_term).** Antes de proponer la creación de una keyword, leer `/sp/keywords/list` filtrando por el `adGroupId` destino y comparar texto + match type. Tres desenlaces:
  - **No existe** → proponer `graduate_term` normal.
  - **Existe y ENABLED** → NO proponer creación (Amazon devuelve `duplicateValueError` / `DUPLICATE_VALUE`). Si su puja está por debajo de lo que justifica la evidencia, proponer `bid_up` sobre el keywordId real; si no, no proponer nada.
  - **Existe y PAUSED** → NO proponer creación. La acción útil es reactivar, que el aplicador no soporta: dejar constancia como alerta de estructura para consola (`tipo_no_soportado`), con keywordId, puja y estado.
  Registrar en el rationale `existencia_verificada: true` y la lista de variantes cercanas encontradas (singular/plural, con y sin acento), para que el revisor decida si va a convivir con un casi-duplicado.
- Alta en Manual primero; negativa en Auto SOLO tras alta aprobada, implementada y confirmada activa. Pareja "1 de 2"/"2 de 2" con nombres legibles y qué pasa si solo se aprueba una.
- **Excepción de mismo ad group**: si el término se origina en una PHRASE/BROAD del MISMO ad group donde se crea la EXACT, NO crear negativa (bloquearía también la nueva; la EXACT ya prevalece en subasta).
- Nunca crear negativa pareja que contenga "picsil" (guardarraíl de marca manda sobre el patrón de pareja).
- Campañas **BrandPure** admiten SOLO términos puros de marca ("picsil", "picsil sport"). Cualquier graduación de marca+producto ("picsil condor", "picsil jump rope") va a la campaña Brand_Key correspondiente, no a BrandPure.

## 10. Formato de salida

Campos: Prioridad, Acción, Campaña (nombre completo legible; IDs solo en metadatos), Targeting, Métricas 30/60d, Motivo, Impacto, Estado + **Confianza** (Alta/Media/Baja según evidencia), **Tipo** (Crecimiento/Eficiencia/Protección/Estructura/Alerta), **Dependencia** (Ninguna/ID previa). Rationale con: window_days, clicks, orders, spend, sales, acos, ad_product ("SP"/"SB"/"SD" — OBLIGATORIO, sin él se asume SP), categoria, roas_real_60d, roas_objetivo, target_acos, target_acos_source, campaign, campaign_id, ad_group_id, amazon_bid_min/suggested/max, cvr_source, max_cpc_*, y una `nota` en lenguaje llano. Prohibido inventar métricas: dato ausente = "No disponible". impact_eur_month solo si es calculable con datos reales.

## 11. Cobertura y transparencia

Barrido de TODAS las categorías activas y los tres productos (SP/SB/SD). Grupo sin sugerencias → DATOS_INSUFICIENTES, RENDIMIENTO_ESTABLE, RESTRICCIÓN_ESTRATÉGICA o RESTRICCIÓN_TÉCNICA (palanca inexistente en ese producto — el silencio se interpreta como "no se revisó"). SB/SBV con menos ASINs mostrados que configurados → alerta informativa cruzada con stock. Cruzar ASINs anunciados con inventario → alerta (no propuesta de puja) por campaña activa promocionando ASIN sin stock.

## 12. Alcance y particularidades por producto publicitario (verificado contra API)

- **SP**: todo el catálogo de acciones. Endpoints v3 con content-types vendor (`application/vnd.spKeyword.v3+json` etc.). Reports: spCampaigns/spTargeting/spSearchTerm, MÁXIMO 31 días por report. **Recomendaciones de puja: usar `application/vnd.spthemebasedbidrecommendation.v5+json`** (v3 devuelve 422 en NL/BE/PL/SE; en IE el endpoint no está soportado en ninguna versión → rango prudente).
- **SB**: budget es NÚMERO plano; estados de campaña (recurso v4) en MAYÚSCULAS, pero los del recurso keyword v3 en minúsculas (`enabled`/`paused`, `matchType: "exact"`); placements TOP_OF_SEARCH/HOME/DETAIL_PAGE/OTHER con `bidAdjustmentsByPlacement` (array completo fusionado). Lectura de keywords: GET /sb/keywords con `Accept: application/vnd.sbkeyword.v3+json` (POST /sb/keywords/list = 404; application/json = 406); acepta `?campaignId=` y `?adGroupId=` como filtros. Reports SB/SD usan columnas `purchases`/`sales` (NO purchases14d/sales14d). sbSearchTerm tiene retención corta (~60d). No hay graduate_term ni report de emplazamientos en SB.
  **Escritura SB (resuelto, apply-proposal v20 del 24-ago)**: `PUT /sb/keywords` y `PUT /sb/targets` usan Content-Type `application/json` con Accept vendor-type (`application/vnd.sbkeywordresponse.v3+json`) — eso se arregló en v18 — y además **exigen `adGroupId` en cada objeto del cuerpo**. Sin él Amazon responde HTTP 207 `INVALID_ARGUMENT` / `KEYWORD_MISSING_AD_GROUP_ID` en `$.adGroupId`. No enviar `campaignId` en las escrituras: es inmutable y solo añade motivos de rechazo. La ruta SB ya está verificada: no marcar `ruta_no_probada`.
- **SD**: API antigua (arrays planos, estados en minúsculas). Puja en el target (entity_type='target') o defaultBid del ad group (entity_type='ad_group'). Sin search terms, sin emplazamientos, sin graduate. Negativas = negativeTargets con `proposed_value.expression`; sin expresión → pause_target. sdTargeting rechaza columnas identificadoras del target en varios mercados → análisis a nivel target limitado. **SD no existe en el marketplace IE (403).**
- `pause_target` solo está soportado en SD por el aplicador; en SP va vía consola (marcar `tipo_no_soportado_en_SP`).
- **Reactivar una keyword pausada no está soportado por el aplicador** en ningún producto: va por consola como alerta de estructura.

## 13. Reglas de sistema innegociables

- Human Approval: insertar con status='pending'. NUNCA aplicar en Amazon desde el generador.
- Dedupe por clave natural: unique (cliente_id, marketplace, type, coalesce(entity_id,entity_name)) con WHERE NOT EXISTS. Si existe (aunque sea rejected/approved), NO reinsertar. No re-proponer lo rechazado hace <30 días (leer rejected_reason).
- Moneda nativa: PL en PLN, SE en SEK; NL/BE/IE en EUR. impact_eur_month siempre en EUR.
- 20-30 propuestas por marketplace y tanda (menos en mercados pequeños, justificándolo), ordenadas por impact_eur_month desc, repartidas entre categorías con gasto y los tres productos.
- Informe de barrido obligatorio por mercado × producto × categoría con motivos de descarte.
- Umbrales `ads_rules` (verificar en Supabase): negativizar ≥12 clics/0 pedidos/60d · graduar ≥2 pedidos y ACOS < objetivo×1,2 · Δbid máx ±25% · presupuesto mín 20 €/día · zombi = PAUSED >12m o ENABLED sin targets servibles y sin ventas.

## Fuentes de datos externas — Helium 10 (MCP)

1. **Jerarquía**: Ads API/SP-API son la verdad para gasto, ventas, ACOS y toda MEDICIÓN. H10 es contexto de mercado para PROPONER (volúmenes, competencia, ranks, bids sugeridos). Nunca medir resultados con H10.
2. **Cuándo consultar H10**: bid_up de escalado y bid_to_visibility (volumen y bid sugerido); negativizaciones con muchas impresiones (distinguir término irrelevante de problema de listing); SEO Matrix y mercados nuevos (Cerebro de 3-5 competidores top + Magnet de seeds locales); checkpoints D7/D14 opcionales (rank orgánico como métrica colateral).
3. **Cobertura**: NL/BE volúmenes NO fiables (usar ranks, densidad competitiva, Amazon's Choice). DE/FR/IT/ES/UK dataset completo fiable.

Fixture DE: "zughilfen" = keyword reina de straps/calleras en DE (15.220 búsquedas/mes, bid sugerido 0,84 $, ABA click share 48%).

## Fixtures de referencia (casos reales)

- NL `RG20240418_Calleras_SP_NL_Brand_Key`: campaña estrangulada (candidata a bid_to_visibility), NO zombi.
- NL: typo `wrips wraps picsil` propagado en ≥3 campañas; keywords francesas en campañas NL; ASIN `b0cwmwyj99` usado como keyword de texto.
- NL: la campaña RG20230504_Combas_SP_NL_Market_Key tuvo una negativa exacta "springtouw" anulando su propia keyword positiva EXACT — revisar siempre colisiones negativa↔positiva en el mismo ad group.
- NL ad group `235093453915078` (RG20240418_Combas_SP_NL_Brand_Key): 42 keywords, muchas ENABLED con puja 0,07 € (no compiten en ninguna subasta) y varios duplicados pausados. Patrón de saturación a revisar antes de añadir nada nuevo.
- IT: Heron fragmentado en ≥3 padres; fuga "calleras" (ES) en descripción italiana; magnesite/magnesio mezclados.
- **Duplicados de graduación (20-ago y 24-ago)**: 3 propuestas fallaron con `DUPLICATE_VALUE` y ~8 se cerraron automáticamente porque la EXACT ya existía en el ad group con 0 impresiones. Casos concretos del 24-ago: NL `picsil rope` (existía a 2,16 € ENABLED, se proponía crear a 1,86 €), NL `picsil springtouw` (existía a 3,44 € ENABLED, se proponía a 0,60 €), BE `gant crossfit` (existía a 3,97 € PAUSED). Las dos de NL eran además recortes encubiertos en términos de marca con ROAS alto.
- Rechazos históricos que NO repetir: bid_down a marca sin impresiones (10+ casos PL/NL/BE), bid_down al único convertidor (muñequeras BE, skakanka PL, straps PL), negativizar términos de marca (picsil grips NL), budget_change a campaña que nunca llega al cap (Calleras NL), bid_up sin especificar campaña+ad group, graduar a BrandPure un término marca+producto (picsil condor NL).

## Plantilla de referencia — propuestas de listing (Heron IT, 2026-08-06)

Medición: separación de métricas (orgánico = total − ads), guardarraíles sin congelación. ANTES DE APROBAR: (1) rellenar 'current' con el texto vivo de los ASINs, (2) validación nativa de Michele sobre título e Item Highlights.

```sql
INSERT INTO listing_proposals
  (cliente_id, asin, marketplace, current, proposed, keyword_rationale, status)
VALUES
(
  'PICSIL', 'B0GXKVWR7W', 'IT',
  NULL,  -- PENDIENTE: capturar título/bullets/descripción vivos antes de aprobar
  jsonb_build_object(
    'change_type','listing_rewrite',
    'title','PICSIL Heron Paracalli Palestra Calisthenics Ginnastica Artistica',
    'title_chars',65,
    'title_alt_b','PICSIL Heron Paracalli Senza Magnesite Palestra Calisthenics Ginnastica',
    'title_alt_b_chars',72,
    'item_highlights','Non serve magnesite: grip diretto su sbarra liscia e ruvida. Protezione mani per trazioni, muscle up e sollevamento pesi.',
    'highlights_chars',121,
    'backend_search_terms','crossfit paracalli crossfit para calli senza magnesio senza magnesite grip grips polsini anti calli calli mani protezione mani trazioni'
  ),
  jsonb_build_object(
    'fuente','Helium 10 Cerebro + Magnet, consulta 2026-08-06',
    'anclas_front', jsonb_build_object('paracalli',3161,'calisthenics',6271,'paracalli ginnastica artistica',1214,'grip palestra',1668),
    'diferenciador','Heron/Hawk funcionan sin magnesio. Amazon recomienda "paracalli senza magnesite/magnesio": demanda probada, ausente de la ficha.',
    'restriccion_legal','CrossFit: marca registrada, solo backend/PPC. Stamina Fitness (competidor nº1 IT): prohibida en todo el listing, atacar solo vía PPC.',
    'nota_magnesite','magnesite (7.861/mes) NO en título: tráfico mayoritario busca comprar magnesio, riesgo de hundir CVR. Sí en Item Highlights.',
    'baseline_ranks', jsonb_build_object('paracalli crossfit',35,'paracalli',31,'picsil paracalli',6,'paracalli picsil',1,'magnesite',null),
    'medicion','Orgánico = total − ads (v_listing_organic). Veredicto sobre ventas/sesiones orgánicas + rank orgánico H10. D14 señal, D30 veredicto. Ads del producto siguen en mantenimiento (bids ±15%); se aplazan solo cambios estructurales; etiquetado cruzado en propuestas de ads durante la ventana.',
    'validacion_nativa','Pendiente: Michele revisa naturalidad del italiano antes de aplicar.'
  ),
  'pending'
),
(
  'PICSIL', 'B0GXKVWR7W', 'IT',
  jsonb_build_object('parents', jsonb_build_array('B0GXKVWR7W','B0GXL5R3ZR','B0DPX6RZX8')),
  jsonb_build_object(
    'change_type','asin_consolidation',
    'target_parent','B0GXKVWR7W',
    'action','Fusionar los 3 padres del Heron en una sola familia de variantes bajo B0GXKVWR7W'
  ),
  jsonb_build_object(
    'fuente','Helium 10 Cerebro, consulta 2026-08-06',
    'evidencia','B0GXKVWR7W único con rank orgánico de categoría (paracalli crossfit 35, paracalli 31) y líder en marca (picsil 1, paracalli picsil 1). B0DPX6RZX8 sin rank orgánico de categoría; rankea por marca competidora Stamina.',
    'autocanibalizacion','En "picsil paracalli" (722 búsq/mes) los padres compiten entre sí y ninguno es nº1.',
    'medicion','Baseline = SUMA de sesiones/ventas orgánicas de los 3 padres (30d previos, v_listing_organic). Comparar contra el consolidado a D30. Sin esta suma, el resultado es artefacto contable.'
  ),
  'pending'
);
```

