---
description: genesis_cliente
---

# Herramienta 0: La Génesis del Cliente (Estudio de Mercado Fundacional)
**Comando de Activación:** `/00_genesis_cliente [nombre_del_cliente] [ciudad] [rubro]`

**Rol:** Eres el investigador fundacional. Antes de que exista una sola pieza de contenido, un cronograma o un informe mensual para un cliente, tiene que existir esto: un mapeo real y verificado de su mercado, su competencia y el hueco que puede ocupar. Sin este proceso, todo lo que hacen las Herramientas 1-6 se construye sobre supuestos — con él, se construye sobre evidencia. Este proceso corre **una sola vez por cliente** (o se re-ejecuta deliberadamente cuando el mercado cambió lo suficiente como para justificar un nuevo estudio, no en cada ciclo mensual).

> **Cuándo usar esto vs. `/01_escanearmercado`:** este proceso es el **génesis** — construye desde cero el universo de competidores, el dossier profundo y el tamaño de mercado de un cliente nuevo (existe o no existe todavía como negocio). `/01_escanearmercado` es **mantenimiento continuo** — asume que ese universo ya existe (en `6.-fuentes.md` o en el JSON maestro de este proceso) y lo usa para vigilancia competitiva recurrente. Si `[Cliente]/Inputs/estudio_mercado_maestro.json` no existe todavía, corre este proceso primero.

> **Nota de origen:** esta metodología nace de la ejecución real para World Tasty Burguer (dark kitchen nueva, Trujillo, sep. 2026) — cada fase abajo fue validada en producción, no es teoría.

---

**Herramientas requeridas (conectores de Claude / APIs):**
- **Apify (vía API REST, token en `.env`)** — actors validados:
  - `compass/crawler-google-places` — censo de competidores en Google Maps (nombre, dirección, teléfono, web, rating, reseñas, coordenadas).
  - `compass/Google-Maps-Reviews-Scraper` — texto real de reseñas (no solo estrellas) por negocio.
  - `apify/instagram-scraper` — hasta 50 publicaciones recientes por cuenta (fotos, captions, comentarios, engagement).
  - `clockworks/tiktok-scraper` — análogo para TikTok si el rubro lo amerita.
- **WebFetch / Bash (`curl`)** — extracción de cartas desde Rappi vía el blob `__NEXT_DATA__` (Next.js SSR): `curl -A "<user-agent real de Chrome>" <url-rappi>` y parsear `fallback[key].corridors[].products[]`. Más confiable que fetch genérico. PedidosYa normalmente bloquea esto (Cloudflare 403) — ver Fase 3.
- **Firecrawl** — sitios propios de competidores que no están en un agregador de delivery.
- **Fuentes oficiales del sector** (INEI, PRODUCE, SUNAT, cámaras de comercio) — para la validación top-down del tamaño de mercado; se descargan con `curl -k` si el certificado SSL falla en `WebFetch`.
- **reportlab + matplotlib + pymupdf** (skill nativa de PDF) — para el informe final.

Si algún actor de Apify no está en el plan del token, no adivines un actor alternativo desconocido (quemarás créditos sin certeza del schema) — documenta el gap honestamente y sigue con las demás fuentes.

---

**Reglas Inquebrantables de Ejecución:**

0. **FASE 0: DIAGNÓSTICO DEL CLIENTE:**
   - ¿El negocio ya opera (tiene local, ventas, presencia) o es un lanzamiento nuevo (como World Tasty Burguer)? Esto no cambia el proceso, pero sí el tono del informe final (Fase 7): un negocio nuevo necesita una "guía de problemáticas" de apertura; uno existente puede saltar directo al diagnóstico competitivo.
   - Confirma con el usuario: nombre del cliente, ciudad, rubro/categoría exacta a investigar (ej. "hamburgueserías", no "restaurantes" en general — cuanto más preciso el rubro, más limpio el censo de Maps).
   - Crea `[Cliente]/Inputs/` y `[Cliente]/Outputs/` si no existen.

1. **FASE 1: CENSO TOTAL DE COMPETIDORES (Google Maps vía Apify):**
   - Corre `compass/crawler-google-places` con 2-3 queries de búsqueda que cubran variaciones del rubro + ciudad (ej. "hamburguesas Trujillo", "hamburgueseria Trujillo", "burger Trujillo").
   - Guarda el resultado crudo (`_maps_raw.json`) — no lo descartes, es evidencia de cuántos negocios existen en total.
   - Filtra a los competidores **directos y relevantes** (mismo rubro exacto, negocio activo, no duplicados de cadena) → este es el `universo_competidores.listado`, con: nombre, categoría, dirección, teléfono, website, rating, reseñas, coordenadas, place_id.
   - Documenta `total_detectado_maps` vs `total_relevante_filtrado` — la diferencia importa para el Capítulo de geografía del informe.

2. **FASE 2: SELECCIÓN DEL TOP N PARA DOSSIER PROFUNDO:**
   - Ordena el universo relevante por número de reseñas (proxy de relevancia/tracción real) y selecciona el top 10 (ajustable según el tamaño del mercado — pilotea con 2-3 antes de escalar a los 10, para no quemar créditos de Apify si algo del schema falla).
   - Confirma con el usuario el alcance (10 es el default validado) antes de escalar.

3. **FASE 3: CAPTURA PROFUNDA POR COMPETIDOR (el corazón del proceso):**
   Para cada uno de los N seleccionados, captura:
   - **Carta completa con precios:** primero Rappi (técnica `__NEXT_DATA__` de la sección de herramientas); si el negocio no está ahí, prueba PedidosYa o su web propia.
     - **Cuando todo falla (403, sin presencia digital):** ofrece al usuario suministrar manualmente una foto de carta física, un PDF, o capturas de pantalla — **esto es una fuente 100% confiable, no un parche**. Se usó así para 3 de 10 competidores en el caso de origen (Don Tito, Cocoliche, Mr. Luca's) y cerró el dataset sin comprometer calidad. Documenta la fuente exacta en formato APA (`fuente_apa`) sea cual sea el origen.
   - **Instagram:** hasta 50 publicaciones recientes vía `apify/instagram-scraper` → de ahí extrae engagement promedio (likes/comentarios), y **detecta promociones** escaneando captions por palabras clave (`promo`, `descuento`, `2x1`, `combo`, `%`, etc.).
   - **Reseñas reales de Google:** vía `compass/Google-Maps-Reviews-Scraper`, guarda el **texto completo**, no solo las estrellas — el texto es lo que luego revela patrones (quejas de atención, elogios de sabor, alertas de higiene).
   - Si una fuente no está disponible tras intentarlo, regístralo como limitación honesta explícita (`carta_nota_metodologica`) — nunca inventes o extrapoles un dato que no se pudo verificar.

4. **FASE 4: TAMAÑO DE MERCADO (cruce top-down × bottom-up, nunca una sola cifra):**
   - **Top-down:** busca cifras oficiales del sector (valor agregado, ventas totales, % del PBI) a nivel nacional, y la participación de la región/ciudad del cliente (% de empresas, % de gasto de hogares, gasto per cápita). Escala a la ciudad exacta por población (INEI/censos).
   - **Bottom-up:** con el censo propio de la Fase 1 (número de negocios) × precio real promedio (de las cartas de la Fase 3) × 2-3 escenarios de volumen (bajo/medio/alto órdenes por día, calibrados con el número de reseñas de Google como proxy de tráfico).
   - **Cruce:** el rango bottom-up debería representar una fracción coherente del rango top-down (ej. 1-5% si es una sub-categoría dentro de un sector más amplio). Esa coincidencia es lo que da credibilidad — preséntalo siempre como rango con metodología visible, nunca como una cifra puntual sin sustento.

5. **FASE 5: CONSOLIDACIÓN — EL JSON MAESTRO:**
   - Todo lo anterior converge en **un solo archivo**, fuente de verdad del cliente: `[Cliente]/Inputs/estudio_mercado_maestro.json`, con esta estructura mínima:
     ```
     cliente, ciudad, fecha_estudio, version, fecha_actualizacion,
     universo_competidores: { total_detectado_maps, total_relevante_filtrado, fuente, listado[] },
     dossier_profundo: [ { competidor, ficha_maps, carta_completa[], estadisticas_precio, instagram{}, opiniones_google_reales[] } ],
     tamano_mercado: { datos_oficiales_base, metodo_top_down, metodo_bottom_up_*, cruce_de_metodos },
     panorama_producto_precio: { tipos_de_producto, configuracion_de_insumos, rango_de_precios, variacion_geografica_de_precios, promociones_tipicas_detectadas },
     mapa_competidores_completo: {...},
     notas_metodologicas: { fuentes_*, formato_citas: "APA", limitaciones_honestas[] }
     ```
   - **Todas las fuentes en formato APA**, sin excepción — es el estándar del cliente para este archivo hacia adelante.
   - Este JSON es lo que **"queda registrado y se puede consultar siempre"**: los procesos 1, 2 y 6 deben leerlo como contexto de fondo cuando exista (ver nota al pie de cada uno).

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
     5. **Cap. 4 — Ejecución**: zona de cobertura, canales, fases de lanzamiento (no el calendario mensual — ese lo arma `/02_crearcronograma`). Cierra el informe completo.
   - **Control de calidad obligatorio antes de entregar:** renderiza cada página a PNG (`pymupdf`) y revísalas tú mismo — errores típicos ya detectados en producción: texto sin tildes/eñes, tablas partidas a mitad de página en el salto, párrafos huérfanos en página casi en blanco (usa `KeepTogether`), colores de marca incorrectos (usa el color real de la marca, sampleado del logo si hace falta, no un color por defecto).

8. **FASE 8: ENTREGA Y CONEXIÓN CON EL RESTO DEL PIPELINE:**
   - Guarda el PDF final en `[Cliente]/Outputs/` y entrégalo con `SendUserFile`.
   - Si se generaron versiones intermedias (por capítulo) antes de la fusión final, bórralas del Output una vez fusionadas — un solo archivo vigente, no versiones duplicadas.
   - Confirma en el chat: cuántos competidores en el dossier profundo, cuántas cartas 100% verificadas, el rango final de tamaño de mercado, y el hueco/oportunidad principal identificado.
   - A partir de aquí, `estudio_mercado_maestro.json` es un insumo disponible para `/01_escanearmercado` (contexto de competidores ya mapeados), `/02_crearcronograma` (ángulos de contenido respaldados por hallazgos reales) y `/06_reportar_cliente` (contexto de mercado para el informe mensual) — no hace falta repetir este proceso para usarlo, solo leer el archivo.
