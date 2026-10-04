---
description: mercado_estudio
---

# Herramienta 0: La Génesis del Cliente (Estudio de Mercado Fundacional)
**Comando de Activación:** `/01_mercado_estudio [nombre_del_cliente] [ciudad] [rubro]`

**Rol:** Eres el investigador fundacional. Antes de que exista una sola pieza de contenido, un cronograma o un informe mensual para un cliente, tiene que existir esto: un mapeo real y verificado de su mercado, su competencia y el hueco que puede ocupar. Sin este proceso, todo lo que hacen las Herramientas 1-6 se construye sobre supuestos — con él, se construye sobre evidencia. Este proceso corre **una sola vez por cliente** (o se re-ejecuta deliberadamente cuando el mercado cambió lo suficiente como para justificar un nuevo estudio, no en cada ciclo mensual).

> **Cuándo usar esto vs. `/03_mercado_vigilancia`:** este proceso es el **génesis** — construye desde cero el universo de competidores, el dossier profundo y el tamaño de mercado de un cliente nuevo (existe o no existe todavía como negocio). `/03_mercado_vigilancia` es **mantenimiento continuo** — asume que ese universo ya existe (en `6.-fuentes.md` o en la fila de `market_studies` de este proceso) y lo usa para vigilancia competitiva recurrente. Si no existe todavía una fila en `market_studies` para este cliente, corre este proceso primero.

> **Nota de fusión con Partners (Supabase):** este proceso se sigue operando a mano, en Claude Desktop, exactamente igual que hasta ahora — nadie lo automatiza sin supervisión, porque el mercado peruano no es confiable solo con datos scrapeados. Lo único que cambia es el destino final: el estudio ya no vive solo en un JSON local, también se escribe en la tabla `market_studies` de Supabase (proyecto `pixely_partners`, ref `zvpisdftltnukbozyuge`) para que el cliente lo vea dentro de la app de Partners, en la fase Mercado. Esto requiere dos variables nuevas en el mismo `.env` donde ya vive `APIFY_API_TOKEN` (`D:\ANTES_15_09_2026\0.-Publicidad_nivel_01\.agents\workflows\.env`, nunca subir este archivo a git):
> ```
> SUPABASE_URL=https://zvpisdftltnukbozyuge.supabase.co
> SUPABASE_SERVICE_KEY=<service role key — en Railway (proyecto pixely-partners, servicio backend, pestaña Variables) es la variable que se llama literalmente SUPABASE_SERVICE_KEY, NO la variable SUPABASE_KEY (esa es la llave pública/anon, un valor distinto y mucho más restringido). Si hace falta copiarla de nuevo, está en Supabase → Project Settings → API → "Legacy anon, service_role API keys" → fila service_role, botón Reveal.>
> ```

> **Nota de origen:** esta metodología nace de la ejecución real para World Tasty Burguer (dark kitchen nueva, Trujillo, sep. 2026) — cada fase abajo fue validada en producción, no es teoría.

---

**Herramientas requeridas (conectores de Claude / APIs):**
- **Metricool (conector de Claude)** — **fuente principal para Instagram y Facebook de la competencia**, estandarizada y sin créditos extra. Cada cliente es una **marca** en Metricool; su `brandId` está en la línea `metricool_brand_id:` de `[Cliente]/Inputs/docs/7.-plan_contratado.md` (si falta, búscalo con `getBrandSettings` y pide agregarlo al archivo). Los competidores se agregan **a mano** en Metricool (marca del cliente → Competidores, en Instagram y Facebook); el conector solo los lee. Datos con `getAnalyticsDataByMetrics(brandId, from, to, metrics)`:
  - Perfil por competidor: Instagram `IGCO02` (usuario), `IGCO07` (seguidores), `IGCO08` (posts), `IGCO12` (reels), `IGCO09` (likes prom.), `IGCO06` (comentarios), `IGCO10` (engagement por 1000 seguidores) · Facebook `FBCO02`, `FBCO06`, `FBCO07`, `FBCO08` (reacciones prom.), `FBCO05`, `FBCO09`, `FBCO10`.
  - Cada post del competidor: Instagram `IGCP01` (competidor), `IGCP04` (texto), `IGCP06` (fecha y hora), `IGCP07` (likes), `IGCP08` (comentarios), `IGCP09` (interacciones), `IGCP10` (engagement), `IGCP11` (url) · reels `IGCR01`, `IGCR03`, `IGCR06`, `IGCR09`, `IGCR04` · Facebook `FBCP01`, `FBCP04`, `FBCP06`, `FBCP07`, `FBCP08`, `FBCP09`.
  - Si un ID cambió, `getAnalyticsAvailableMetrics(network, connector="competitors" | "competitor posts" | "competitor reels")` da la lista vigente.
  - **No cubre** TikTok, Google Maps ni la biblioteca de anuncios de Meta: para eso sigue Apify.
- **Apify (vía API REST, token en `.env`)** — para lo que Metricool no cubre. Actors validados:
  - `compass/crawler-google-places` — censo de competidores en Google Maps (nombre, dirección, teléfono, web, rating, reseñas, coordenadas).
  - `compass/Google-Maps-Reviews-Scraper` — texto real de reseñas (no solo estrellas) por negocio.
  - `apify/instagram-scraper` — **solo de respaldo**: para un competidor de Instagram que aún no está en Metricool o si el conector falla. Hasta 50 publicaciones recientes por cuenta.
  - `clockworks/tiktok-scraper` — análogo para TikTok si el rubro lo amerita.
- **WebFetch / Bash (`curl`)** — extracción de cartas desde Rappi vía el blob `__NEXT_DATA__` (Next.js SSR): `curl -A "<user-agent real de Chrome>" <url-rappi>` y parsear `fallback[key].corridors[].products[]`. Más confiable que fetch genérico. PedidosYa normalmente bloquea esto (Cloudflare 403) — ver Fase 3.
- **Firecrawl** — sitios propios de competidores que no están en un agregador de delivery.
- **Fuentes oficiales del sector** (INEI, PRODUCE, SUNAT, cámaras de comercio) — para la validación top-down del tamaño de mercado; se descargan con `curl -k` si el certificado SSL falla en `WebFetch`.
- **reportlab + matplotlib + pymupdf** (skill nativa de PDF) — para el informe final.
- **Bash (`curl`) contra la API REST de Supabase** — mismo patrón que Apify: token leído del `.env` en el momento de la llamada, nunca asumido en memoria entre pasos.

Si algún actor de Apify no está en el plan del token, no adivines un actor alternativo desconocido (quemarás créditos sin certeza del schema) — documenta el gap honestamente y sigue con las demás fuentes.

---

**Reglas Inquebrantables de Ejecución:**

0. **FASE 0: DIAGNÓSTICO DEL CLIENTE:**
   - ¿El negocio ya opera (tiene local, ventas, presencia) o es un lanzamiento nuevo (como World Tasty Burguer)? Esto no cambia el proceso, pero sí el tono del informe final (Fase 7): un negocio nuevo necesita una "guía de problemáticas" de apertura; uno existente puede saltar directo al diagnóstico competitivo.
   - Confirma con el usuario: nombre del cliente, ciudad, rubro/categoría exacta a investigar (ej. "hamburgueserías", no "restaurantes" en general — cuanto más preciso el rubro, más limpio el censo de Maps).
   - Crea `[Cliente]/Inputs/` y `[Cliente]/Outputs/` si no existen — esto sigue siendo la carpeta de trabajo local, independiente de Partners.
   - **Resuelve el `client_id` de Partners (obligatorio para la Fase 5):** este cliente debe existir ya en la tabla `clients` de Supabase (se crea desde el panel de Admin de Partners). Búscalo por nombre:
     ```bash
     SUPABASE_URL=$(grep SUPABASE_URL "D:/ANTES_15_09_2026/0.-Publicidad_nivel_01/.agents/workflows/.env" | cut -d= -f2)
     SUPABASE_KEY=$(grep SUPABASE_SERVICE_KEY "D:/ANTES_15_09_2026/0.-Publicidad_nivel_01/.agents/workflows/.env" | cut -d= -f2)
     curl -s "$SUPABASE_URL/rest/v1/clients?nombre=ilike.*[nombre_del_cliente]*&select=id,nombre" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
     Si no hay match o hay más de uno, muestra las opciones (o la ausencia de resultados) al usuario y pide el `client_id` exacto antes de continuar — nunca lo inventes ni asumas el primero de la lista.

1. **FASE 1: CENSO TOTAL DE COMPETIDORES (Google Maps vía Apify):**
   - Corre `compass/crawler-google-places` con 2-3 queries de búsqueda que cubran variaciones del rubro + ciudad (ej. "hamburguesas Trujillo", "hamburgueseria Trujillo", "burger Trujillo").
   - Guarda el resultado crudo (`_maps_raw.json`) — no lo descartes, es evidencia de cuántos negocios existen en total.
   - Filtra a los competidores **directos y relevantes** (mismo rubro exacto, negocio activo, no duplicados de cadena) → este es el `universo_competidores.listado`, con: nombre, categoría, dirección, teléfono, website, rating, reseñas, coordenadas, place_id.
   - Documenta `total_detectado_maps` vs `total_relevante_filtrado` — la diferencia importa para el Capítulo de geografía del informe.

2. **FASE 2: SELECCIÓN DEL TOP N PARA DOSSIER PROFUNDO:**
   - Ordena el universo relevante por número de reseñas (proxy de relevancia/tracción real) y selecciona el top 10 (ajustable según el tamaño del mercado — pilotea con 2-3 antes de escalar a los 10, para no quemar créditos de Apify si algo del schema falla).
   - Confirma con el usuario el alcance (10 es el default validado) antes de escalar.
   - **Alta en Metricool:** con el top N confirmado, lista sus cuentas de Instagram y Facebook y pide al usuario que las agregue como competidores en la marca del cliente en Metricool. Espera su confirmación antes de la Fase 3 (sin eso, el conector no devuelve sus datos). Si el cliente aún no tiene marca en Metricool, sigue con Apify de respaldo y anótalo como limitación.

3. **FASE 3: CAPTURA PROFUNDA POR COMPETIDOR (el corazón del proceso):**
   Para cada uno de los N seleccionados, captura:
   - **Carta completa con precios:** primero Rappi (técnica `__NEXT_DATA__` de la sección de herramientas); si el negocio no está ahí, prueba PedidosYa o su web propia.
     - **Cuando todo falla (403, sin presencia digital):** ofrece al usuario suministrar manualmente una foto de carta física, un PDF, o capturas de pantalla — **esto es una fuente 100% confiable, no un parche**. Se usó así para 3 de 10 competidores en el caso de origen (Don Tito, Cocoliche, Mr. Luca's) y cerró el dataset sin comprometer calidad. Documenta la fuente exacta en formato APA (`fuente_apa`) sea cual sea el origen.
   - **Instagram y Facebook (Metricool):** perfil del competidor y sus posts y reels de los **últimos 90 días** (IDs en Herramientas). De ahí sale el engagement promedio, la cadencia (posts y reels por mes), sus publicaciones con más interacciones y las **promociones**, escaneando los textos por palabras clave (`promo`, `descuento`, `2x1`, `combo`, `%`, etc.). Cita cada dato como `(Metricool, Instagram @cuenta, fecha)`. Solo si un competidor no está en Metricool, usa `apify/instagram-scraper`.
   - **TikTok:** `clockworks/tiktok-scraper` si el rubro lo amerita (Metricool no da competencia en TikTok).
   - **Reseñas reales de Google:** vía `compass/Google-Maps-Reviews-Scraper`, guarda el **texto completo**, no solo las estrellas — el texto es lo que luego revela patrones (quejas de atención, elogios de sabor, alertas de higiene).
   - Si una fuente no está disponible tras intentarlo, regístralo como limitación honesta explícita (`carta_nota_metodologica`) — nunca inventes o extrapoles un dato que no se pudo verificar.

4. **FASE 4: TAMAÑO DE MERCADO (cruce top-down × bottom-up, nunca una sola cifra):**
   - **Top-down:** busca cifras oficiales del sector (valor agregado, ventas totales, % del PBI) a nivel nacional, y la participación de la región/ciudad del cliente (% de empresas, % de gasto de hogares, gasto per cápita). Escala a la ciudad exacta por población (INEI/censos).
   - **Bottom-up:** con el censo propio de la Fase 1 (número de negocios) × precio real promedio (de las cartas de la Fase 3) × 2-3 escenarios de volumen (bajo/medio/alto órdenes por día, calibrados con el número de reseñas de Google como proxy de tráfico).
   - **Cruce:** el rango bottom-up debería representar una fracción coherente del rango top-down (ej. 1-5% si es una sub-categoría dentro de un sector más amplio). Esa coincidencia es lo que da credibilidad — preséntalo siempre como rango con metodología visible, nunca como una cifra puntual sin sustento.

5. **FASE 5: CONSOLIDACIÓN — EL JSON MAESTRO:**
   - **Promedios de la competencia para Partners** (`competitor_benchmarks`, uno por competidor, red y mes): con los datos de Metricool del mes en curso escribe `seguidores`, `posts`, `reels`, `engagement` e `interacciones_prom` (= interacciones promedio por publicación del mes: suma de `IGCP09`/`IGCR09` del competidor ÷ número de publicaciones; en Facebook, reacciones + comentarios + compartidos ÷ publicaciones). Publicaciones de Partners lo usa para comparar cada pieza del cliente:
     ```bash
     curl -s -X POST "$SUPABASE_URL/rest/v1/competitor_benchmarks?on_conflict=client_id,mes,red,competidor" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY" \
       -H "Content-Type: application/json" -H "Prefer: resolution=merge-duplicates" \
       -d '[{ "client_id": "<client_id>", "mes": "YYYY-MM", "red": "instagram", "competidor": "<usuario sin @>", "seguidores": 0, "posts": 0, "reels": 0, "interacciones_prom": 0, "engagement": 0 }]'
     ```
     Escríbelo junto con la fila de `market_studies`, tras el mismo visto bueno.
   - Todo lo anterior converge en una sola estructura, fuente de verdad del cliente (fila en `market_studies` de Supabase; el JSON local `[Cliente]/Inputs/estudio_mercado_maestro.json` sigue existiendo como copia de trabajo), con esta estructura mínima:
     ```
     cliente, ciudad, fecha_estudio, version, fecha_actualizacion,
     universo_competidores: { total_detectado_maps, total_relevante_filtrado, fuente, listado[] },
     dossier_profundo: [ { competidor, ficha_maps, carta_completa[], estadisticas_precio, instagram{}, opiniones_google_reales[] } ],
     tamano_mercado: { datos_oficiales_base, metodo_top_down, metodo_bottom_up_*, cruce_de_metodos, rango_estimado },
     panorama_producto_precio: { tipos_de_producto, configuracion_de_insumos, rango_de_precios, variacion_geografica_de_precios, promociones_tipicas_detectadas },
     mapa_competidores_completo: {...},
     notas_metodologicas: { fuentes_*, formato_citas: "APA", limitaciones_honestas[] }
     ```
   - **Campos que la pantalla de Mercado de Partners dibuja** (si faltan o cambian de nombre, esa sección simplemente no aparece — no se rompe, pero el cliente ve menos):
     - `universo_competidores.listado[]`: cada competidor como `{ "nombre": "...", "rating": 4.6, "reseñas": 1240, "direccion": "...", "website": "..." }` — `rating` y `reseñas` **como números**, no texto. Alimentan el medidor "¿Qué tan exigente es tu mercado?", el ranking "¿Quiénes lideran tu mercado?" y el mapa competitivo (rating × reseñas).
     - `universo_competidores.total_relevante_filtrado` y `universo_competidores.total_detectado_maps`, como números: la cifra "Competidores directos — de N negocios en Google Maps".
     - `dossier_profundo[].competidor` + `dossier_profundo[].estadisticas_precio`: `{ "min": 9, "max": 32, "promedio": 16.5 }` en soles, como números. Alimentan "¿Cuánto cobra tu competencia?" y el ticket promedio del mercado.
     - `tamano_mercado.rango_estimado`: `{ "min": 1200000, "max": 2800000, "moneda": "PEN", "periodo": "anual" }` — el resultado final del cruce de la Fase 4 en números, para que se muestre como cifra grande. `tamano_mercado.cruce_de_metodos` (texto) se muestra debajo como explicación.
     - `panorama_producto_precio.promociones_tipicas_detectadas`: lista de textos cortos (`["2x1 en bebidas de 3 a 5 pm", ...]`).
   - **Todas las fuentes en formato APA**, sin excepción — es el estándar del cliente para este archivo hacia adelante.
   - El JSON local sigue siendo tu copia de trabajo (útil para depurar sin reconsultar APIs), pero **la fuente de verdad ahora es Supabase**: inserta o actualiza la fila de este cliente en `market_studies` (la tabla tiene una restricción `UNIQUE` sobre `client_id`; con `Prefer: resolution=merge-duplicates` **y** `on_conflict=client_id` en la URL, una re-ejecución para el mismo cliente actualiza esa fila en vez de crear una nueva — sin el parámetro `on_conflict`, Supabase compara contra el `id` interno, que siempre es nuevo, y termina duplicando igual):
     ```bash
     curl -s -X POST "$SUPABASE_URL/rest/v1/market_studies?on_conflict=client_id" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY" \
       -H "Content-Type: application/json" -H "Prefer: resolution=merge-duplicates,return=representation" \
       -d '{
         "client_id": "<el client_id resuelto en la Fase 0>",
         "ciudad": "...", "rubro": "...", "fecha_estudio": "YYYY-MM-DD", "version": "1",
         "universo_competidores": { ... }, "dossier_profundo": [ ... ],
         "tamano_mercado": { ... }, "panorama_producto_precio": { ... },
         "mapa_competidores_completo": { ... }, "notas_metodologicas": { ... }
       }'
     ```
     Los 6 campos `jsonb` reciben el objeto correspondiente tal cual del JSON maestro, sin aplanar ni renombrar claves — eso es lo que después lee la pantalla de Mercado en Partners.
   - Este estudio es lo que **"queda registrado y se puede consultar siempre"**: los procesos 1, 2 y 6 deben leerlo (desde Supabase, no desde el JSON) como contexto de fondo cuando exista (ver nota al pie de cada uno).

6. **FASE 6: SÍNTESIS ANALÍTICA:**
   - **Matriz de insumos** (competidor × ingrediente/atributo del producto): revela cuáles son estándar de mercado (≥50% de los competidores) y cuáles son diferenciadores reales (usados por 0-1 competidor).
   - **Arquitectura de precios**: min/max/spread por competidor — revela arquetipos de estrategia (red amplia / escalonada / mono-precio).
   - **Redes sociales**: engagement real (no solo seguidores) — busca la desconexión entre tamaño del negocio y actividad digital (suele haber una, y suele ser la oportunidad más accionable del informe).
   - **Reseñas reales**: lee el texto completo, no solo el rating, y busca patrones repetidos (3+ menciones) — ahí vive la ventaja competitiva más barata de conseguir (casi siempre es confiabilidad operativa: tiempos, exactitud de pedido, consistencia — no sabor).

7. **FASE 7: INFORME ESTRATÉGICO MULTI-CAPÍTULO:**
   - Construye el informe en capítulos separados (permite iterar y corregir uno sin rehacer todo), luego **fusiónalos en un único PDF final** (portada única, índice con `TableOfContents` + números de página reales vía `multiBuild`, numeración continua, `BaseDocTemplate` con `PageTemplate`/`NextPageTemplate` por capítulo para el pie de página contextual).
   - Estructura validada (ajustable, pero probada):
     1. **Guía de problemáticas** del dueño del negocio — tabla de preguntas reales de decisión → sección del informe que la resuelve. Encuadra todo el informe como herramienta, no como lectura pasiva.
     2. **Cap. 1 — Panorama y dimensión**: tamaño de mercado (Fase 4), producto/insumos/precios (Fase 6), geografía competitiva (mapa con Fase 1).
     3. **Cap. 2 — Radiografía de competidores**: arquitectura de precios, matriz de insumos, redes, voz real del cliente (reseñas), matriz de síntesis fortaleza/debilidad/oportunidad.
     4. **Cap. 3 — Marco de decisión**: **regla crítica** — nunca entregues una carta, un nombre de producto o una decisión de marca ya armada. Da el dato, el método de 4-5 pasos para decidir, y cierra con 15-20 preguntas directas ancladas cada una a un hallazgo específico del informe (formato: pregunta + por qué importa, citando el capítulo). El cliente decide; el informe informa. (Ver memoria de feedback guardada: un borrador anterior de este capítulo fue rechazado exactamente por invertir esta regla.)
     5. **Cap. 4 — Ejecución**: zona de cobertura, canales, fases de lanzamiento (no el calendario mensual — ese lo arma `/05_planificacion`). Cierra el informe completo.
   - **Control de calidad obligatorio antes de entregar:** renderiza cada página a PNG (`pymupdf`) y revísalas tú mismo — errores típicos ya detectados en producción: texto sin tildes/eñes, tablas partidas a mitad de página en el salto, párrafos huérfanos en página casi en blanco (usa `KeepTogether`), colores de marca incorrectos (usa el color real de la marca, sampleado del logo si hace falta, no un color por defecto).

8. **FASE 8: ENTREGA Y CONEXIÓN CON EL RESTO DEL PIPELINE:**
   - Guarda el PDF final en `[Cliente]/Outputs/` y entrégalo con `SendUserFile` (como hasta ahora, para quien está corriendo el proceso).
   - Súbelo también al bucket `market-studies` de Supabase Storage, para que el cliente pueda descargarlo desde Partners:
     ```bash
     curl -s -X POST "$SUPABASE_URL/storage/v1/object/market-studies/<client_id>/informe-<fecha_estudio>.pdf" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY" \
       -H "Content-Type: application/pdf" --data-binary @"[Cliente]/Outputs/informe.pdf"
     ```
     Luego actualiza `pdf_url` en la fila de `market_studies` de este cliente con `https://zvpisdftltnukbozyuge.supabase.co/storage/v1/object/public/market-studies/<client_id>/informe-<fecha_estudio>.pdf` (mismo endpoint de la Fase 5, con `PATCH` filtrando por `client_id`).
   - Si se generaron versiones intermedias (por capítulo) antes de la fusión final, bórralas del Output una vez fusionadas — un solo archivo vigente, no versiones duplicadas.
   - Confirma en el chat: cuántos competidores en el dossier profundo, cuántas cartas 100% verificadas, el rango final de tamaño de mercado, y el hueco/oportunidad principal identificado.
   - A partir de aquí, la fila de `market_studies` de este cliente es un insumo disponible para `/03_mercado_vigilancia` (contexto de competidores ya mapeados), `/05_planificacion` (ángulos de contenido respaldados por hallazgos reales) y `/06_reportar_cliente` (contexto de mercado para el informe mensual) — no hace falta repetir este proceso para usarlo, solo consultar Supabase.
