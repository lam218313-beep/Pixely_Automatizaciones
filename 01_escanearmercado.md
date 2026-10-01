---
description: escanear_mercado
---

# Herramienta 1: El Radar de Inteligencia Competitiva (Multi-Fuente)
**Comando de Activación:** `/01_escanearmercado [nombre_del_cliente]`

**Rol:** Eres un analista de inteligencia competitiva. No investigas en genérico: apuntas a blancos reales (el dossier de `/00_genesis_cliente` si existe, si no los competidores de `6.-fuentes.md`), les espías la web, las redes y los anuncios que están pagando ahora mismo, validas eso contra data macro, y solo confías en un hallazgo cuando dos o más fuentes independientes lo confirman. El objetivo es un estudio **brutal** (va directo a lo que el rival ya validó con presupuesto), **certero** (nada entra a Airtable sin cruce de fuentes) y **eficiente** (las 3 fuentes corren en paralelo, no en cadena).

> **Nota de migración:** esta versión reemplaza Tavily-only + archivos `.md` sueltos (pensado para Antigravity) por un stack multi-fuente con espionaje competitivo dirigido, que escribe directo a Airtable con nivel de confianza por hallazgo.

> **Nota de génesis:** si existe `[Cliente]/Inputs/estudio_mercado_maestro.json` (creado por `/00_genesis_cliente`), léelo antes de la Fase 0 — ya trae el universo de competidores mapeado, sus cartas/precios y su dossier profundo. Úsalo para no repetir descubrimiento desde cero; este proceso se enfoca entonces en **vigilancia recurrente** (qué cambió: nuevos anuncios, nuevos posts, nuevos precios) sobre ese universo ya conocido.

---

**Herramientas requeridas (conectores de Claude):**
- **Tavily** (`tavily_search`, `tavily_research`) — validación macro: tendencias, estudios, datos de industria/gobierno.
  > **Fallback obligatorio si Tavily no está conectado o falla ("Connection closed"):** no bloquees todo el proceso por esto — usa `WebFetch` + `Firecrawl` sobre las mismas fuentes/medios que usarías con Tavily. Este fallback ya fue validado en producción (sesión WTB, sep. 2026) y produce resultados equivalentes, solo más manual. El hallazgo se sigue clasificando `Tipo de Señal = Macro` sin importar qué herramienta lo obtuvo.
- **Firecrawl** (`firecrawl_search`) — disección de los sitios web de los competidores: página de precios/servicios, propuesta de valor, CTAs, temas de blog.
- **Apify (vía API REST, no MCP)** — espionaje social y publicitario de competidores + comportamiento de audiencia general. El token vive en `D:\ANTES_15_09_2026\0.-Publicidad_nivel_01\.agents\workflows\.env` (`APIFY_API_TOKEN=...`) — **este archivo nunca debe subirse a un repositorio ni compartirse**; si este proyecto se convierte en repo git en el futuro, agrégalo a `.gitignore` de inmediato. Cada llamada debe leer el valor de ese archivo (las variables de entorno no persisten entre llamadas de shell independientes). Actors confirmados y accesibles con este token:
  - `apify/instagram-scraper` y `clockworks/tiktok-scraper` — perfiles de competidores Y hashtags de nicho (cuenta sin actors propios, se usan los públicos del store).
  - `apify/facebook-ads-scraper` — Meta Ad Library: qué anuncios está pagando cada competidor **ahora mismo** en Facebook/Instagram. Esta es la señal más fuerte: un rival no paga por un ángulo que no le funciona.
- **Airtable** — destino de los hallazgos estructurados, con nivel de confianza y tipo de señal (reemplaza `investigacion_[tema].md`).

Si alguno de estos conectores no está activo, detente y pide al usuario que lo conecte antes de continuar esa fuente específica (no bloquees las demás fuentes por una que falte).

---

**Reglas Inquebrantables de Ejecución:**

0. **FASE 0: DEFINIR BLANCOS (antes de investigar nada):**
   - **Fuente primaria:** si existe `[Cliente]/Inputs/estudio_mercado_maestro.json` (de `/00_genesis_cliente`), extrae los blancos de `dossier_profundo` (ya trae nombre, website, handle de Instagram y hasta carta/precios verificados) — no repitas descubrimiento que ya está hecho.
   - **Fallback:** si no existe ese JSON, extrae la lista de `6.-fuentes.md`: nombre, URL del sitio, y handle de Instagram/TikTok si está documentado (si no lo está, búscalo con una consulta rápida de Tavily/WebFetch antes de continuar — no adivines el handle).
   - Sin una lista concreta de blancos no hay espionaje posible; esta fase es obligatoria y precede a las hipótesis.

0-B. **FASE 0-B: EXPANSIÓN DE FUENTES (detectar competidores que génesis no mapeó):**
   - El set de blancos de la Fase 0 no es una lista fija para siempre — el mercado sigue moviéndose después de `/00_genesis_cliente`. Esta fase busca candidatos **nuevos**, relacionados a los blancos ya conocidos, que génesis no capturó:
     - **Maps:** re-corre la(s) misma(s) query(s) amplia(s) que usó génesis en su Fase 1 (`compass/crawler-google-places`) — no es un censo nuevo completo, es solo un diff: cualquier resultado que no esté ya en `universo_competidores.listado` del JSON maestro es un candidato (negocio nuevo, o que el filtro original dejó fuera).
     - **Ad Library:** además de buscar por nombre de cada competidor conocido (Fase 3-C2), busca también por categoría + ciudad (ej. `q=hamburguesas&country=PE` sin nombre) — así aparecen anunciantes activos que génesis nunca vio, porque génesis no espía anuncios pagados.
     - **Hashtags de nicho (Fase 3-C3):** cualquier cuenta que aparezca repetidamente ahí y no esté en el dossier profundo es candidata.
   - **Alcance de esta fase (importante):** solo detecta y registra — **no** dispares una captura profunda (carta, reseñas, dossier) para estos candidatos; eso sigue siendo trabajo exclusivo de `/00_genesis_cliente`. Si un candidato amerita un estudio completo, coméntalo en el resumen del chat (Fase 6) como sugerencia, pero no lo ejecutes automáticamente ni modifiques `estudio_mercado_maestro.json` desde este proceso.
   - Cada candidato nuevo se registra como un hallazgo normal en la Fase 5 con `Tipo de Señal = Nuevo-competidor-detectado` y `Competidor` = el nombre detectado.

1. **FASE 1: ASIMILACIÓN DEL CONTEXTO:**
   - Lee con la herramienta `Read` los siguientes archivos de `D:\ANTES_15_09_2026\0.-Publicidad_nivel_01\[nombre_del_cliente]\Inputs\docs\`:
     - `1.-identidad.md` (Quiénes somos)
     - `3.-inputs_comercial.md` (Qué vendemos exactamente y a qué precio)
     - `4.-buyer.md` (A quién se lo vendemos y qué le duele)
     - `6.-fuentes.md` (Competidores, fuentes confiables y cuentas/hashtags de referencia)

2. **FASE 2: HIPÓTESIS SOCRÁTICAS:**
   - Genera 3 a 5 preguntas estratégicas cruzando marca + buyer + blancos, separadas en dos carriles:
     - **Carril Macro** (para Tavily): tendencias de industria, estadísticas, comportamiento de compra.
     - **Carril Competitivo** (para Firecrawl + Apify): qué está prometiendo, publicando y pagando cada competidor de la Fase 0 ahora mismo.
   - **Si existe `estudio_mercado_maestro.json`:** no partas de cero en el Carril Competitivo — lee la síntesis analítica de génesis (`mapa_competidores_completo` y el gap diferenciador ya identificado en la Fase 6/Cap.3 de `/00_genesis_cliente`) y formula las hipótesis del mes como una verificación de ese gap: "¿sigue abierto, o algún competidor ya empezó a cubrirlo?" — 01 audita el hallazgo de génesis, no lo redescubre.
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
     > **Anti-redundancia con génesis (obligatorio si existe `estudio_mercado_maestro.json`):** este mismo actor ya corrió en la Fase 3 de `/00_genesis_cliente` con `resultsLimit: 50`. No lo vuelvas a tratar como descubrimiento desde cero — filtra el resultado a posts con fecha posterior a `fecha_estudio` (o a la fecha de la última corrida de `/01_escanearmercado` si ya hubo una anterior) y solo registra como hallazgo lo que sea genuinamente nuevo. Antes de loguear una promoción como hallazgo, crúzala contra `panorama_producto_precio.promociones_tipicas_detectadas` del JSON maestro — si ya está ahí, no es una promo nueva, es la misma que génesis ya detectó (ignórala salvo que haya cambiado el descuento/condición).

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
     `Fecha | Cliente | Fuente (Web/Instagram/TikTok) | Tema | Dato o Ángulo | Evidencia/Cita | Cluster (Problema/Identidad/Prueba) | Confianza (Alta/Media/Baja) | Tipo de Señal | Competidor | Link`
   - **Obligación de Escritura:** guarda estas filas en la tabla `Investigaciones` de la base de Airtable del cliente (`[Cliente] - Publicidad`).
     - Verifica primero con `list_bases` si la base existe. Si no existe, créala con `create_base` (nombre `[Cliente] - Publicidad`) y notifica al usuario que se creó.
     - Verifica con `list_tables_for_base` si la tabla `Investigaciones` existe y tiene los campos `Confianza`, `Tipo de Senal` y `Competidor`; si faltan, créalos con `create_field`.
     - Inserta las filas con `create_records_for_table`.

6. **FASE 6: FEEDBACK Y CONFIRMACIÓN EN CHAT:**
   - Notifica cuántas filas se escribieron, desglosadas por Confianza (Alta/Media/Baja) y por Tipo de Señal.
   - Muestra en el chat las hipótesis socráticas, el hallazgo de mayor confianza de cada competidor, y el gap diferenciador identificado en la Fase 4.
   - Si la Fase 0-B detectó candidatos nuevos (`Nuevo-competidor-detectado`), lístalos aparte y sugiere si alguno amerita un `/00_genesis_cliente` de refresco — es solo una sugerencia en el chat, nunca lo dispares automáticamente ni edites el JSON maestro desde aquí.
