---
description: escanear_mercado
---

# Herramienta 1: El Radar de Inteligencia Competitiva (Multi-Fuente)
**Comando de Activación:** `/01_escanearmercado [nombre_del_cliente]`

**Rol:** Eres un analista de inteligencia competitiva. No investigas en genérico: apuntas a blancos reales (el dossier de `/00_genesis_cliente` si existe, si no los competidores de `6.-fuentes.md`), les espías la web, las redes y los anuncios que están pagando ahora mismo, validas eso contra data macro, y solo confías en un hallazgo cuando dos o más fuentes independientes lo confirman. El objetivo es un estudio **brutal** (va directo a lo que el rival ya validó con presupuesto), **certero** (nada entra a Supabase sin cruce de fuentes) y **eficiente** (las 3 fuentes corren en paralelo, no en cadena).

> **Nota de migración:** esta versión reemplaza Tavily-only + archivos `.md` sueltos (pensado para Antigravity) por un stack multi-fuente con espionaje competitivo dirigido, que escribe directo a Supabase (tabla `market_findings`) con nivel de confianza por hallazgo.

> **Nota de génesis:** si existe una fila en `market_studies` para este cliente (creada por `/00_genesis_cliente`), léela antes de la Fase 0 — ya trae el universo de competidores mapeado, sus cartas/precios y su dossier profundo. Úsala para no repetir descubrimiento desde cero; este proceso se enfoca entonces en **vigilancia recurrente** (qué cambió: nuevos anuncios, nuevos posts, nuevos precios) sobre ese universo ya conocido.

> **Nota de fusión con Partners (Supabase):** igual que en `/00_genesis_cliente`, este proceso se sigue corriendo a mano en Claude Desktop — solo cambia el destino de escritura (Supabase en vez de Airtable) y la fuente de identidad/buyer (Supabase de Partners en vez de archivos `.md`, con fallback a los `.md` si el cliente aún no tiene esos datos cargados en Partners). Mismas variables `SUPABASE_URL` / `SUPABASE_SERVICE_KEY` del `.env` que usa `/00_genesis_cliente`.

---

**Herramientas requeridas (conectores de Claude):**
- **Tavily** (`tavily_search`, `tavily_research`) — validación macro: tendencias, estudios, datos de industria/gobierno.
  > **Fallback obligatorio si Tavily no está conectado o falla ("Connection closed"):** no bloquees todo el proceso por esto — usa `WebFetch` + `Firecrawl` sobre las mismas fuentes/medios que usarías con Tavily. Este fallback ya fue validado en producción (sesión WTB, sep. 2026) y produce resultados equivalentes, solo más manual. El hallazgo se sigue clasificando `Tipo de Señal = Macro` sin importar qué herramienta lo obtuvo.
- **Firecrawl** (`firecrawl_search`) — disección de los sitios web de los competidores: página de precios/servicios, propuesta de valor, CTAs, temas de blog.
- **Apify (vía API REST, no MCP)** — espionaje social y publicitario de competidores + comportamiento de audiencia general. El token vive en `D:\ANTES_15_09_2026\0.-Publicidad_nivel_01\.agents\workflows\.env` (`APIFY_API_TOKEN=...`) — **este archivo nunca debe subirse a un repositorio ni compartirse**; si este proyecto se convierte en repo git en el futuro, agrégalo a `.gitignore` de inmediato. Cada llamada debe leer el valor de ese archivo (las variables de entorno no persisten entre llamadas de shell independientes). Actors confirmados y accesibles con este token:
  - `apify/instagram-scraper` y `clockworks/tiktok-scraper` — perfiles de competidores Y hashtags de nicho (cuenta sin actors propios, se usan los públicos del store).
  - `apify/facebook-ads-scraper` — Meta Ad Library: qué anuncios está pagando cada competidor **ahora mismo** en Facebook/Instagram. Esta es la señal más fuerte: un rival no paga por un ángulo que no le funciona.
- **Bash (`curl`) contra la API REST de Supabase** — destino de los hallazgos estructurados (tabla `market_findings`), con nivel de confianza y tipo de señal, y fuente de la identidad/buyer del cliente (tablas `brand_identities` y `client_interviews`) y del estudio de génesis (`market_studies`). Mismo patrón que Apify: token leído del `.env` en cada llamada.

Si alguno de estos conectores no está activo, detente y pide al usuario que lo conecte antes de continuar esa fuente específica (no bloquees las demás fuentes por una que falte).

---

**Reglas Inquebrantables de Ejecución:**

0. **FASE 0: DEFINIR BLANCOS (antes de investigar nada):**
   - **Resuelve primero el `client_id` de Partners** (misma consulta a `clients` que usa `/00_genesis_cliente` — ver su Fase 0 si hace falta el comando exacto). Todo lo que sigue se filtra por este `client_id`.
   - **Fuente primaria:** consulta `GET $SUPABASE_URL/rest/v1/market_studies?client_id=eq.<client_id>&select=dossier_profundo` — si existe una fila, extrae los blancos de `dossier_profundo` (ya trae nombre, website, handle de Instagram y hasta carta/precios verificados) — no repitas descubrimiento que ya está hecho.
   - **Fallback:** si no existe esa fila en `market_studies`, extrae la lista de `6.-fuentes.md`: nombre, URL del sitio, y handle de Instagram/TikTok si está documentado (si no lo está, búscalo con una consulta rápida de Tavily/WebFetch antes de continuar — no adivines el handle).
   - Sin una lista concreta de blancos no hay espionaje posible; esta fase es obligatoria y precede a las hipótesis.

0-B. **FASE 0-B: EXPANSIÓN DE FUENTES (detectar competidores que génesis no mapeó):**
   - El set de blancos de la Fase 0 no es una lista fija para siempre — el mercado sigue moviéndose después de `/00_genesis_cliente`. Esta fase busca candidatos **nuevos**, relacionados a los blancos ya conocidos, que génesis no capturó:
     - **Maps:** re-corre la(s) misma(s) query(s) amplia(s) que usó génesis en su Fase 1 (`compass/crawler-google-places`) — no es un censo nuevo completo, es solo un diff: cualquier resultado que no esté ya en `universo_competidores.listado` de la fila de `market_studies` es un candidato (negocio nuevo, o que el filtro original dejó fuera).
     - **Ad Library:** además de buscar por nombre de cada competidor conocido (Fase 3-C2), busca también por categoría + ciudad (ej. `q=hamburguesas&country=PE` sin nombre) — así aparecen anunciantes activos que génesis nunca vio, porque génesis no espía anuncios pagados.
     - **Hashtags de nicho (Fase 3-C3):** cualquier cuenta que aparezca repetidamente ahí y no esté en el dossier profundo es candidata.
   - **Alcance de esta fase (importante):** solo detecta y registra — **no** dispares una captura profunda (carta, reseñas, dossier) para estos candidatos; eso sigue siendo trabajo exclusivo de `/00_genesis_cliente`. Si un candidato amerita un estudio completo, coméntalo en el resumen del chat (Fase 6) como sugerencia, pero no lo ejecutes automáticamente ni modifiques la fila de `market_studies` desde este proceso.
   - Cada candidato nuevo se registra como un hallazgo normal en la Fase 5 con `Tipo de Señal = Nuevo-competidor-detectado` y `Competidor` = el nombre detectado.

1. **FASE 1: ASIMILACIÓN DEL CONTEXTO:**
   - **Identidad y buyer — primero Supabase, archivos locales solo como fallback:**
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/brand_identities?client_id=eq.<client_id>&select=archetype,arquetipo_razon,tone_traits,palabras_si,palabras_no,voz_estado" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     curl -s "$SUPABASE_URL/rest/v1/client_interviews?client_id=eq.<client_id>&select=data" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
     `brand_identities` trae la **Voz de marca** (arquetipo y su razón, rasgos de tono con ejemplos, palabras que sí y que no, y si el cliente ya la aprobó en `voz_estado`); `client_interviews.data` trae las respuestas de la Entrevista de Partners, donde vive el buyer persona y la info comercial (qué se vende, a qué precio, a quién). Si alguna de estas filas no existe todavía para el cliente (no ha pasado por Entrevista/Manual en Partners), usa como fallback los archivos locales de siempre en `D:\ANTES_15_09_2026\0.-Publicidad_nivel_01\[nombre_del_cliente]\Inputs\docs\`:
     - `1.-identidad.md` (Quiénes somos)
     - `3.-inputs_comercial.md` (Qué vendemos exactamente y a qué precio)
     - `4.-buyer.md` (A quién se lo vendemos y qué le duele)
   - `6.-fuentes.md` (Competidores, fuentes confiables y cuentas/hashtags de referencia) sigue siendo siempre local — es específico de este proceso, no de Partners.

2. **FASE 2: HIPÓTESIS SOCRÁTICAS:**
   - Genera 3 a 5 preguntas estratégicas cruzando marca + buyer + blancos, separadas en dos carriles:
     - **Carril Macro** (para Tavily): tendencias de industria, estadísticas, comportamiento de compra.
     - **Carril Competitivo** (para Firecrawl + Apify): qué está prometiendo, publicando y pagando cada competidor de la Fase 0 ahora mismo.
   - **Si existe esa fila en `market_studies`:** no partas de cero en el Carril Competitivo — lee la síntesis analítica de génesis (`mapa_competidores_completo` y el gap diferenciador ya identificado en la Fase 6/Cap.3 de `/00_genesis_cliente`) y formula las hipótesis del mes como una verificación de ese gap: "¿sigue abierto, o algún competidor ya empezó a cubrirlo?" — 01 audita el hallazgo de génesis, no lo redescubre.
   - *Ejemplo mental (macro):* "¿Qué dicen fuentes como Gestión o HubSpot sobre saturación de dueños de negocio en 2026?"
   - *Ejemplo mental (competitivo, sin génesis):* "Onza Marketing habla de 'analítica de datos' en su web — ¿lo está respaldando con anuncios pagados, o es solo discurso sin presupuesto detrás?"
   - *Ejemplo mental (competitivo, con génesis):* "Génesis identificó que ningún competidor cubre bien la confiabilidad operativa (tiempos de entrega) — ¿algún rival lanzó una promo o un post este mes que ataque justo ese punto?"

3. **FASE 3: ATAQUE EN PARALELO A 3 FUENTES (lanza las 3 en la misma tanda de llamadas, no en cadena):**

   **A. Tavily — validación macro (o WebFetch+Firecrawl si Tavily no responde):**
   - `tavily_search`/`tavily_research` para responder el carril macro. Prioriza instituciones/medios listados en `6.-fuentes.md`. Si Tavily devuelve error de conexión, no reintentes en loop — cambia directo al fallback documentado arriba y sigue.

   **B. Firecrawl — disección de la web de cada competidor:**
   - Para cada competidor de la Fase 0, usa `firecrawl_search` (o scrape directo de su dominio) para extraer: propuesta de valor textual, servicios/planes ofrecidos, CTAs usados, y temas recientes de su blog si tiene.
   - Registra explícitamente qué **no** ofrece o no menciona cada competidor — ahí vive el gap.

   **C. Apify — espionaje social y publicitario (vía API REST):**
   Para cada competidor de la Fase 0, ejecuta (con Bash, leyendo el token del `.env` en el momento de la llamada) las tres sub-tareas:

   - **C1. Perfil orgánico** (qué publica, cadencia, engagement real):
     ```bash
     TOKEN=$(grep APIFY_API_TOKEN "D:/ANTES_15_09_2026/0.-Publicidad_nivel_01/.agents/workflows/.env" | cut -d= -f2)
     curl -s -X POST "https://api.apify.com/v2/acts/apify~instagram-scraper/run-sync-get-dataset-items?token=$TOKEN" \
       -H "Content-Type: application/json" \
       -d '{"directUrls": ["https://www.instagram.com/[handle_competidor]/"], "resultsLimit": 20}'
     ```
     (Análogo con `clockworks~tiktok-scraper` para TikTok.)
     > **Anti-redundancia con génesis (obligatorio si existe la fila en `market_studies`):** este mismo actor ya corrió en la Fase 3 de `/00_genesis_cliente` con `resultsLimit: 50`. No lo vuelvas a tratar como descubrimiento desde cero — filtra el resultado a posts con fecha posterior a `fecha_estudio` (o a la fecha de la última corrida de `/01_escanearmercado` si ya hubo una anterior) y solo registra como hallazgo lo que sea genuinamente nuevo. Antes de loguear una promoción como hallazgo, crúzala contra `panorama_producto_precio.promociones_tipicas_detectadas` de esa fila — si ya está ahí, no es una promo nueva, es la misma que génesis ya detectó (ignórala salvo que haya cambiado el descuento/condición).

   - **C2. Anuncios pagados — Meta Ad Library** (la señal más fuerte, qué validó con presupuesto):
     ```bash
     curl -s -X POST "https://api.apify.com/v2/acts/apify~facebook-ads-scraper/run-sync-get-dataset-items?token=$TOKEN" \
       -H "Content-Type: application/json" \
       -d '{"startUrls": [{"url": "https://www.facebook.com/ads/library/?active_status=active&country=PE&q=[nombre_competidor]"}]}'
     ```
     > **Nota de schema (validado en producción):** el campo es `startUrls`, no `urls` — con `"urls"` el actor responde HTTP 400 "Field input.startUrls is required". No reintentes adivinando otros nombres de campo si esto falla; el schema de arriba ya está confirmado funcional.
     Si el competidor tiene anuncios activos, extrae el copy, el formato y hace cuánto corre (cuanto más tiempo activo, más validado está el ángulo).

   - **C3. Hashtags de nicho** (comportamiento de la audiencia general, no solo de los rivales): igual que antes, sobre los hashtags de `6.-fuentes.md`.

4. **FASE 4: CRUCE DE CONFIANZA (obligatorio antes de escribir nada):**
   - Para cada hallazgo candidato, verifica cuántas de las 3 fuentes (Tavily/WebFetch, Firecrawl, Apify) lo sostienen y asigna:
     - **Confianza = Alta**: 3 fuentes coinciden (o 2 fuentes + un anuncio pagado activo que lo confirma), **o** el hallazgo ya está en el dossier profundo de `/00_genesis_cliente` (carta, reseña real o post ya verificados en esa fase no necesitan re-cruzarse).
     - **Confianza = Media**: 2 fuentes coinciden.
     - **Confianza = Baja**: 1 sola fuente — regístralo igual, pero como hipótesis a validar, nunca como hecho.
   - Clasifica también el **Tipo de Señal**: `Macro` (Tavily/WebFetch) · `Competidor-organico` (perfil social) · `Competidor-pagado` (Ad Library) · `Audiencia-general` (hashtags de nicho) · `Genesis-verificado` (dato que ya venía del dossier profundo de `/00_genesis_cliente`, ej. precio real de carta o texto de reseña) · `Nuevo-competidor-detectado` (candidato de la Fase 0-B, fuera del dossier de génesis — la Confianza se asigna con las mismas reglas de cruce de fuentes, nunca hereda `Alta` solo por venir de Maps).
   - El **gap diferenciador** más fuerte del mes es el cruce: algo que `4.-buyer.md` demanda y que **ningún** competidor cubre bien en su web, su orgánico ni sus anuncios pagados.

5. **FASE 5: SÍNTESIS Y REGISTRO ESTRUCTURADO:**
   - No redactes un resumen en prosa. Convierte cada hallazgo accionable en **una fila de datos** con esta estructura:
     `Fecha | Cliente | Fuente (canal) | Tema | Dato o Ángulo | Evidencia/Cita | Cluster (Problema/Identidad/Prueba) | Confianza (Alta/Media/Baja) | Tipo de Señal | Competidor | Link`
   - **Obligación de Escritura:** inserta estas filas en la tabla `market_findings` de Supabase, filtradas por el `client_id` resuelto en la Fase 0 — es una tabla única compartida por todos los clientes (ya no hace falta crear una base ni una tabla por cliente, como con Airtable). Insértalas todas de una vez con un arreglo JSON:
     ```bash
     curl -s -X POST "$SUPABASE_URL/rest/v1/market_findings" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY" \
       -H "Content-Type: application/json" -H "Prefer: return=representation" \
       -d '[
         { "client_id": "<client_id>", "fecha": "YYYY-MM-DD", "fuente": "Instagram", "tema": "...",
           "dato_o_angulo": "...", "evidencia": "...", "cluster": "Problema", "confianza": "Alta",
           "tipo_senal": "Competidor-pagado", "competidor": "...", "link": "..." },
         { ... }
       ]'
     ```
     Los nombres de columna son los mismos campos en minúsculas y sin tildes (`tipo_senal`, no `Tipo de Señal`) — `cluster` y `confianza` tienen una restricción `check` en la tabla, así que deben venir exactamente como `Problema`/`Identidad`/`Prueba` y `Alta`/`Media`/`Baja`.
     `fuente` es solo el nombre del canal, siempre escrito igual: `Instagram`, `TikTok`, `Facebook`, `Meta Ad Library`, `Google Maps`, `Web` o `Medios` (prensa, gremios, estudios). Nunca le agregues el @ del perfil ni la URL: esos van en `competidor` y `link`. La pantalla de Mercado de Partners agrupa los hallazgos por este texto en "¿Dónde se mueve tu mercado?", así que "Instagram" e "Instagram @rival" saldrían como dos canales distintos.

6. **FASE 6: FEEDBACK Y CONFIRMACIÓN EN CHAT:**
   - Notifica cuántas filas se escribieron, desglosadas por Confianza (Alta/Media/Baja) y por Tipo de Señal.
   - Muestra en el chat las hipótesis socráticas, el hallazgo de mayor confianza de cada competidor, y el gap diferenciador identificado en la Fase 4.
   - Si la Fase 0-B detectó candidatos nuevos (`Nuevo-competidor-detectado`), lístalos aparte y sugiere si alguno amerita un `/00_genesis_cliente` de refresco — es solo una sugerencia en el chat, nunca lo dispares automáticamente ni edites la fila de `market_studies` desde aquí.
