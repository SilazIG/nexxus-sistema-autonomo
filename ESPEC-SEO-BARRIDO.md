# ESPECIFICACIÓN — Barrido SEO de Amazon (app Inngest `picsil-seo`)

Vas a crear, dentro de este repositorio, una aplicación Node/TypeScript nueva en
`apps/seo-barrido/`. Es un worker de Inngest (modo connect) que, al recibir un
evento, recorre todos los ASIN × marketplace activos de PICSIL en Amazon EU,
genera para cada uno una propuesta de listing (título, bullets/Item Highlights,
backend search terms) con la API de Anthropic siguiendo las reglas del documento
`docs/picsil-amazon-skill.md` (créalo con el contenido que se te adjunta al final
de esta tarea), la valida, y la inserta en Supabase en `listing_proposals` con
`status='pending'`. **Nunca escribe en Amazon. Nunca propone precios.** La
aprobación la hace un humano en otra pantalla.

Trabaja en una sola rama y deja un único Pull Request con TODO lo de abajo,
`npm test` en verde dentro de `apps/seo-barrido/`. No toques nada fuera de
`apps/seo-barrido/` y `docs/`.

## 1. Estructura a crear

```
apps/seo-barrido/
  package.json          scripts: worker (tsx src/connect.ts), test (vitest run), typecheck (tsc --noEmit), seo:ping (tsx src/scripts/ping.ts)
  tsconfig.json         strict, ES2022, moduleResolution bundler
  .env.example          todas las variables del §7, sin valores
  .gitignore            .env, node_modules, dist
  README.md             qué hace, cómo se despliega (pm2), cómo se lanza un barrido, cómo se lee el informe
  src/inngest/client.ts           new Inngest({ id: "picsil-seo" })
  src/inngest/functions/barrido.ts   función padre
  src/inngest/functions/propuesta.ts función hija
  src/connect.ts                  connect({ apps: [{ client, functions: [barrido, propuesta] }] }) con drain en SIGTERM
  src/seo/supabase.ts             cliente (service role) + consultas
  src/seo/spapi.ts                LWA token, getListingsItem, mapa asin→sku, inventario
  src/seo/claude.ts               llamada a Anthropic Messages API y parseo
  src/seo/validar.ts              validación dura (pura, sin IO) — es lo que más se testea
  src/seo/informe.ts              texto del informe + POST a n8n
  src/scripts/ping.ts             lee 1 listing vivo por SP-API y lo imprime
  test/validar.test.ts, test/informe.test.ts, test/dedupe.test.ts
```
Dependencias: `inngest@^4`, `@anthropic-ai/sdk`, `@supabase/supabase-js`,
`zod`, `dotenv`, dev: `tsx`, `typescript`, `vitest`. Carga `dotenv/config` al
arrancar (el worker corre bajo pm2). SDK de Inngest v4: los `triggers` van
dentro del primer objeto de `createFunction`.

## 2. Funciones Inngest

### `seo-barrido` (padre) — trigger `picsil/seo.barrido.pedido`
`data: { marketplaces?: string[], asins?: string[], dryRun?: boolean }`.
`concurrency: { key: '"seo"', limit: 1 }`, `retries: 2`.
1. `step.run("inventario")`: lista de pares {asin, sku, marketplace} activos
   (ver §5). Aplica filtros de `data`. Si está vacía → informe "sin pares" y fin.
2. `step.run("crear-run")`: inserta fila en `seo_runs` (status running, dry_run,
   pares_total, inngest_run_id) y devuelve su id.
3. Procesa los pares en lotes de 10 con `Promise.all` de
   `step.invoke("par-<asin>-<mkt>", { function: propuesta, data: {...par, runId, dryRun} })`.
   Cada hijo devuelve `{ resultado: "propuesta"|"saltado"|"descartado", motivo?, usd }`.
   Tras cada lote acumula el coste; si supera `SEO_BUDGET_USD` → para, marca el
   run `failed` con motivo `presupuesto`, envía informe y termina.
4. `step.run("cerrar-run")`: actualiza `seo_runs` (finished_at, status done,
   contadores, coste_usd, detalle por marketplace con motivos de descarte).
5. `step.run("telegram")`: informe (§8) por POST a `N8N_TELEGRAM_WEBHOOK_URL`.

### `seo-propuesta-asin` (hijo) — trigger `picsil/seo.propuesta.pedida`
`concurrency: { key: '"claude-seo"', limit: 2 }`, `retries: 3`.
1. `step.run("dedupe")`: si existe `listing_proposals` para (cliente_id='PICSIL',
   asin, marketplace, change_type='listing_rewrite') con status pending/approved,
   o rejected hace <30 días → devolver `saltado`.
2. `step.run("listing-vivo")`: SP-API `getListingsItem` → `current` =
   `{ title, bullets[], description, backend_search_terms, fetched_at }`. Sin
   listing → `descartado: sin_listing_vivo`.
3. `step.run("contexto")`: `seo_market_rules` del mercado (si `active=false` →
   `descartado: mercado_inactivo`), `brand_protected_terms` activos,
   `product_costs` por sku (familia/categoría; el PVP solo como contexto),
   `v_listing_organic` 60d si tiene filas (hoy puede estar vacía: entonces
   "No disponible").
4. `step.run("redactar")`: Claude (§3). Si devuelve `{descartar:true}` →
   `descartado: <motivo>`.
5. `step.run("validar")`: `validar(propuesta, reglas, current)` (§4). Si falla →
   `descartado: <primera regla incumplida>`.
6. `step.run("insertar")`: si `dryRun` → no insertar, devolver la propuesta en el
   resultado para el informe. Si no → INSERT del §6; si no inserta fila → `saltado`.

## 3. Llamada a Claude (`src/seo/claude.ts`)

- Modelo `claude-sonnet-5`, `max_tokens: 4000`, `temperature: 0.2`.
- `system` = `docs/picsil-amazon-skill.md` completo (léelo del disco en arranque)
  + JSON de la fila `seo_market_rules` + lista `brand_protected_terms` + este
  texto fijo: *"Devuelve SOLO un JSON válido con las claves `proposed` y
  `keyword_rationale` siguiendo la plantilla Heron IT de la skill
  (`change_type` = 'listing_rewrite', `title`, `title_chars`, `bullets` (array),
  `backend_search_terms`). Escribe en el idioma del mercado con términos locales
  reales; prohibido traducir keywords. No propongas precio. Si los datos son
  insuficientes devuelve {"descartar": true, "motivo": "..."}."*
- `user` = JSON con asin, marketplace, current, contexto, familia del producto.
- Parseo con zod. Si el JSON no es válido, un reintento con el mensaje "Responde
  solo con el JSON"; si vuelve a fallar → `descartado: json_invalido`.
- Devuelve también `usd` estimado a partir de `usage` (constantes de precio en
  un fichero `precios.ts`, fáciles de cambiar).

## 4. Validación dura (`src/seo/validar.ts`, función pura con tests)

Devuelve `{ ok: true } | { ok: false, regla: string }`. Reglas, en este orden:
1. `change_type === 'listing_rewrite'`.
2. `title` empieza por "PICSIL" y su longitud real ≤ `title_max_chars`.
3. Ningún término de `terminos_prohibidos` en título ni bullets (case-insensitive; en backend sí se permite).
4. Ninguna marca competidora en título/bullets/backend: velites, wodies, rx smart gear, gornation, bear grips, versa gripps, stamina fitness (lista en constante, ampliable).
5. `backend_search_terms`: ≤ `backend_max_bytes` en UTF-8, sin comas, sin palabras ya presentes en el título.
6. Al menos 2 de `seeds_locales` aparecen en título o bullets (control barato de "no está traducido"). Si `seeds_locales` tiene <2 elementos, esta regla se salta.
7. `bullets.length ≤ bullets_count` y cada bullet ≤ `bullet_max_chars`.
8. `keyword_rationale` contiene `fuente`, `anclas_front`, `baseline_ranks`, `medicion`, `validacion_nativa`.
9. `current` no es null.
`status` se fija a `'pending'` en código; nunca se toma del JSON de Claude.

## 5. Inventario de pares (`src/seo/spapi.ts`)

- LWA: `POST https://api.amazon.com/auth/o2/token` con refresh_token → access token (cachea 50 min).
- Endpoint EU: `https://sellingpartnerapi-eu.amazon.com`. Marketplace IDs:
  NL `A1805IZSGTT6HS`, BE `AMEN7PMS3EDWL`, IE `A28R8C7NBKEWEA`, SE `A2NODRKZP88ZB9`, PL `A1C3SOZRARQ6R3`.
- Inventario: `GET /fba/inventory/v1/summaries?granularityType=Marketplace&granularityId=<id>&marketplaceIds=<id>` paginando `nextToken`; par activo = `totalQuantity > 0` (o `fulfillableQuantity > 0`). Devuelve {asin, sku, marketplace}.
- Listing vivo: `GET /listings/2021-08-01/items/{SPAPI_SELLER_ID}/{sku}?marketplaceIds=<id>&includedData=attributes,summaries`.
  Título `attributes.item_name[0].value`, bullets `attributes.bullet_point[].value`,
  descripción `attributes.product_description[0].value`, backend `attributes.generic_keyword[].value`.
- Respeta 429: reintento con backoff (Inngest ya reintenta el step; dentro del step un solo reintento con 2 s).
- **Prohibido implementar ningún `PUT`/`PATCH`/`DELETE` contra SP-API.** Solo GET.

## 6. Inserción con dedupe (`src/seo/supabase.ts`)

Usa `supabase.rpc` o SQL vía `postgres` — lo más simple y verificable: una
función SQL NO (no toques el esquema). Hazlo con el cliente supabase-js en dos
pasos dentro del mismo `step.run`: (a) `select id,status,created_at` con los
filtros de dedupe; (b) si no hay bloqueo, `insert`. Documenta en README que la
ventana entre (a) y (b) está cubierta por `concurrency` del hijo + dedupe previo.

```
insert listing_proposals { cliente_id:'PICSIL', asin, marketplace, current, proposed, keyword_rationale, status:'pending' }
bloquea si existe fila con mismo (cliente_id, asin, marketplace) y proposed->>'change_type'='listing_rewrite'
  y (status in ('pending','approved') o (status='rejected' y created_at > now()-30d))
```
Las tablas `seo_market_rules` y `seo_runs` **ya existen** en Supabase (creadas
el 2026-09-04) con las columnas que usas aquí; no crees migraciones.
`seo_market_rules`: marketplace, idioma, title_max_chars, title_target_chars,
bullets_count, bullet_max_chars, backend_max_bytes, seeds_locales(jsonb),
terminos_prohibidos(jsonb), restricciones_legales, validador_nativo, notas,
active, updated_at. `seo_runs`: id, inngest_run_id, started_at, finished_at,
status, dry_run, pares_total, propuestas, saltados, descartados, coste_usd, detalle.

## 7. Variables de entorno (`.env.example`)

```
INNGEST_EVENT_KEY=
INNGEST_SIGNING_KEY=
ANTHROPIC_API_KEY=
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
SPAPI_LWA_CLIENT_ID=
SPAPI_LWA_CLIENT_SECRET=
SPAPI_REFRESH_TOKEN=
SPAPI_SELLER_ID=
SPAPI_REGION=eu
N8N_TELEGRAM_WEBHOOK_URL=
SEO_BUDGET_USD=25
```
Valida al arrancar con zod que todas existen; si falta una, el proceso sale con
un mensaje claro que nombra la variable (sin imprimir valores).

## 8. Informe (texto plano, sin HTML)

```
📦 Barrido SEO Amazon — {fecha}{ " (ENSAYO)" si dryRun }
Pares revisados: {n} · Propuestas nuevas: {p} · Ya tenían: {s} · Descartados: {d}
{por marketplace: "NL 12 · BE 9 · IE 7 · SE 6 · PL 11"}
Motivos de descarte: {motivo count, ...}
Coste Claude: {usd} USD · Duración: {h}h {m}m
Run Inngest: {inngest_run_id}
```
En dryRun añade debajo la primera propuesta completa (título, bullets, backend)
para que el humano la lea. POST `{ "message": texto }` a `N8N_TELEGRAM_WEBHOOK_URL`.
Telegram corta a 4096 caracteres: trunca con "…".

## 9. Tests (vitest, sin red)

- `validar.test.ts`: cada regla del §4 con un caso que pasa y uno que falla
  (título sin PICSIL, término prohibido en bullet, backend con coma, backend que
  repite palabras del título, marca competidora, sin seeds locales, rationale incompleto).
- `informe.test.ts`: formato del texto y truncado a 4096.
- `dedupe.test.ts`: la función pura que decide "bloquea/no bloquea" dada una
  lista de filas existentes (pending, approved, rejected reciente, rejected antiguo, otro change_type).
- Nada de tests que llamen a Amazon, Supabase o Anthropic.

## 10. README — despliegue (para quien lo ponga en marcha)

```
cd apps/seo-barrido && npm ci && cp .env.example .env   # rellenar .env
npm run typecheck && npm test && npm run seo:ping
pm2 start "npm run worker" --name picsil-seo --cwd apps/seo-barrido && pm2 save
# lanzar barrido de ensayo de un ASIN:
curl -X POST https://inn.gs/e/$INNGEST_EVENT_KEY -H 'content-type: application/json' \
  -d '{"name":"picsil/seo.barrido.pedido","data":{"asins":["B0..."],"marketplaces":["NL"],"dryRun":true}}'
```

## Reglas innegociables

Solo `status='pending'`; nada se aplica en Amazon; el PVP no se toca; nunca
traducir keywords; marca PICSIL protegida y marcas de terceros fuera del
listing; dedupe por (cliente_id, asin, marketplace, change_type); dato ausente =
"No disponible"; guardarraíl de gasto por barrido; concurrency 2 hijo / 1 padre.
