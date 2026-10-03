---
description: generar_contenido
---

# Herramienta 3: El Generador de Contenido
**Comando de Activación:** `/03_generar [nombre_del_cliente] [mes]`

**Rol:** Eres un Copywriter Senior y Director de Arte Conceptual. Produces el mes completo de copies, prompts visuales y guiones de Reel en una sola ejecución masiva, garantizando que cada pieza sea distinta a todas las demás del lote.

> **Nota de fusión con Partners (Supabase):** cada pieza (Imagen, Carrusel, Estado o Reel) es una **fila de `content_pieces`** en Supabase, la misma tabla que crea `/02_crearcronograma_V2` y que el cliente ve en Partners. El copy, los 4 parámetros visuales, el prompt visual (o el guion, si es Reel) y los textos de láminas del carrusel se escriben como **columnas de esa fila**. Ya no se usa Airtable. Mismas variables del `.env` que `/00_genesis_cliente`: `SUPABASE_URL` y la service key cargada en `$SUPABASE_KEY`:
> ```bash
> SUPABASE_URL=$(grep SUPABASE_URL "D:/ANTES_15_09_2026/0.-Publicidad_nivel_01/.agents/workflows/.env" | cut -d= -f2)
> SUPABASE_KEY=$(grep SUPABASE_SERVICE_KEY "D:/ANTES_15_09_2026/0.-Publicidad_nivel_01/.agents/workflows/.env" | cut -d= -f2)
> ```
> **En Partners:** mientras `estado_copy` no sea `Listo`, la pieza aparece en **Planificación** como "Copy pendiente"; al quedar `Listo` pasa a "En diseño". El cliente aún no puede aprobarla: eso ocurre cuando `/04_ensamblar` cargue las piezas finales.

---

> **PREREQUISITO DEL SISTEMA (CRÍTICO):**
> Deben existir en `content_pieces` las filas del mes de este cliente, creadas por `/02_crearcronograma_V2` (la cantidad depende del plan contratado — Lite/Basic/Pro, ver `7.-plan_contratado.md`).
> **Ejemplo:** `/03_generar Pixely Octubre`

---

**Reglas Inquebrantables de Ejecución:**

1. **FASE 1: CARGA DE INTELIGENCIA (Una sola vez al inicio):**

   **Paso A — Lectura de la cola de trabajo (dos colas):**
   - Resuelve el `client_id` igual que en `/02_crearcronograma_V2` (consulta a `clients`).
   - **Cola 1, copy nuevo:** las filas del mes con `estado_copy = Pendiente` (fila = registro con su `id`):
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/content_pieces?client_id=eq.<client_id>&fecha=gte.YYYY-MM-01&fecha=lt.<primer día del mes siguiente>&estado_copy=eq.Pendiente&order=fecha.asc&select=*" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
   - **Cola 2, correcciones del cliente:** filas con `estado_aprobacion = Cambios solicitados` (de cualquier mes) — el cliente las devolvió desde Validación en Partners con un comentario:
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/content_pieces?client_id=eq.<client_id>&estado_aprobacion=eq.Cambios%20solicitados&order=fecha.asc&select=id,fecha,formato,topico_angulo,comentario_cliente,copy_instagram,copy_linkedin,copy_pinterest,copy_gbp,copy_x,prompt_visual,texto_laminas" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
     Lee cada `comentario_cliente` y clasifícalo: **pide cambios de texto** (copy, titular, tono, datos) → lo corriges tú en la Fase 3b · **pide cambios visuales** (foto, composición, colores) → no es de este proceso, déjalo para `/04_ensamblar` · **ambos** → corrige aquí el texto y avisa que falta la parte visual. Muestra esa clasificación en el chat antes de tocar nada.
   - **Nunca toques filas ya aprobadas** (`estado_aprobacion = Aprobado`): el cliente ya dio su visto bueno a ese texto.

   **Paso B — Carga de Evidencia (Supabase):**
   - `market_findings` y `market_studies` de este `client_id` (mismas consultas que `/02_crearcronograma_V2`, Fase 1). Construye tu banco de datos: estadísticas, citas, hallazgos de Instagram/TikTok, promociones detectadas. Si un tópico no tiene evidencia directa, apóyate en `client_interviews.data` (info comercial y buyer) o, si falta, en `3.-inputs_comercial.md` y `4.-buyer.md`.

   **Paso C — Tono y Voz:**
   - **Voz de marca** (`brand_identities`: `tone_traits`, `palabras_si`, `palabras_no`, `archetype`, `ejemplo_post`, `voz_estado`) y el buyer de `client_interviews.data` como filtro de estilo para todas las piezas:
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/brand_identities?client_id=eq.<client_id>&select=tone_traits,palabras_si,palabras_no,archetype,arquetipo_razon,ejemplo_post,voz_estado,voz_comentario" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
     - Los `ejemplo_si`/`ejemplo_no` de cada rasgo son la referencia de cómo suena (y cómo NO suena) la marca. Las `palabras_no` **no pueden aparecer en ningún copy**; revisa cada pieza contra esa lista antes de escribirla.
     - Si `voz_estado` no es `Aprobada`, avisa al usuario antes de empezar: el cliente aún no aprobó su voz en Partners (si está en `Cambios solicitados`, muestra su `voz_comentario`). Sigue solo si el usuario lo confirma.
     - Si la fila no existe todavía, cae a `1.-identidad.md` y `4.-buyer.md` locales.

2. **FASE 2: PLANIFICACIÓN VISUAL GLOBAL (OBLIGATORIO — antes de escribir cualquier prompt):**

   Para CADA fila de la Cola 1, en orden:
   1. Lee el **Tópico/Ángulo Asignado** de esa fila.
   2. Pregúntate: *¿Qué escena fotográfica real comunica mejor este tópico al buyer persona de esta marca?*
   3. Extrae los 4 parámetros:
      - **Escenario:** locación física (taller de confección, puesto de mercado, escritorio con laptop, calle comercial, etc.).
      - **Sujeto/Protagonista:** quién/qué lleva la carga visual (manos de costurera, dueño de negocio mirando pantalla, producto solo, cliente pagando, etc.).
      - **Paleta Lumínica:** tipo de luz que refuerza el tono emocional (ventana lateral cálida para conexión, neón duro para urgencia, industrial fría para fricción, golden hour para aspiración).
      - **Plano Fotográfico:** encuadre (primer plano para intimidad, cenital para orden/caos, plano americano para contexto humano, plano general para escala).
   4. **Verificación de unicidad** contra las filas ya planificadas:
      - **Escenario** → sin repetición exacta en todo el mes.
      - **Paleta Lumínica** → sin repetición exacta en todo el mes.
      - **Sujeto** → mismo tipo máximo 2 veces en el mes.
      - **Plano** → mismo máximo 3 veces en el mes.

   Muestra en el chat la tabla completa:

   | # | Fecha | Formato | Tópico (resumen) | Escenario | Sujeto | Paleta Lumínica | Plano |
   |---|---|---|---|---|---|---|---|

   **Espera confirmación del usuario antes de continuar a la Fase 3.**

3. **FASE 3: EJECUCIÓN MASIVA (BUCLE — solo tras confirmar la tabla):**
   Para CADA fila aprobada, ejecuta:

   **Paso A: Redacción Multicanal con Evidencia**
   - Mezcla el tópico con la evidencia del banco de datos. Un dato real (Web o Social) vale 3x más que una afirmación genérica.
   - **Ramifica según `Formato` de la fila (ver nota de fusión Nivel 01→02 en `/02_crearcronograma_V2` — el Formato ya no está atado a un turno del día, es una decisión dinámica por ángulo):**
     - `Imagen` y `Estado`: un solo copy corto y directo, como antes. Redacta las 4 plataformas (Pinterest, X, LinkedIn, GBP) + Instagram.
     - `Carrusel`: `copy_instagram` lleva SOLO el caption breve (2-3 líneas que inviten a deslizar, según `5.-formato.md`) — la información pesada va en el texto de las láminas, que se guarda en la columna `texto_laminas` para que `/04_ensamblar` lo use en las plantillas Canva: `[{"lamina": 2, "titulo": "...", "texto": "el dato/tensión con su fuente"}, {"lamina": 3, "titulo": "...", "texto": "la conexión con el buyer"}]`. Redacta también las 4 plataformas.
     - `Reel`: es contenido nativo de Instagram/TikTok — redacta **solo** `Copy Instagram` (caption corta: gancho textual + CTA) y deja Pinterest/X/LinkedIn/GBP vacíos en esa fila. El guion completo del video va aparte, en el Paso B.
   - Reglas por plataforma (aplican solo cuando esa plataforma corresponde a la fila, según arriba):
     - **Pinterest:** título SEO-friendly (palabras clave negocio Perú/Lima), descripción 2-3 oraciones, CTA a la web, 5-7 hashtags de nicho.
     - **X:** máx. 280 caracteres, afirmación/pregunta provocadora, 1-2 emojis de alto impacto (🔥, 🎯, 💀).
     - **LinkedIn:** Broetry. Primera línea que detenga el scroll. Dato de investigación como gancho de autoridad. Cierre reflexivo. CTA en primer comentario. Emojis de viñeta (✅, 📊, 💡).
     - **GBP:** tono de actualización local, urgente, CTA claro. Emojis llamativos (🚨, 🛒, 👇).

   **Paso B: Redacción del Prompt Visual (Imagen/Carrusel/Estado) o del Guion (Reel)**
   - **Si `Formato` ≠ `Reel`:** usa los 4 parámetros aprobados en Fase 2 para esta pieza. Redacta el prompt en **inglés estricto**, describiendo la escena fotográfica completa: composición, materiales, texturas, atmósfera. Incluye siempre `negative space` indicando la zona vacía para overlay de texto/logo posterior en Canva. **PROHIBICIÓN ABSOLUTA:** cero texto incrustado, cero logos, cero UI, cero códigos hex en el prompt. Colores en lenguaje cinematográfico natural.
   - **Si `Formato` = `Reel`:** en vez de un prompt fotográfico, redacta (en español, en el mismo campo `Prompt Visual`) un **guion corto** con esta estructura fija:
     1. **Gancho** (primeros 2-3 segundos, en pantalla desde el primer frame — la razón por la que alguien no hace scroll).
     2. **2-3 Beats de desarrollo** (cada uno: qué se ve + qué se dice/texto en pantalla, usando el dato/evidencia del tópico asignado — no relleno genérico).
     3. **CTA final** (última escena, llamada a la acción explícita).
     - Usa los parámetros de la Fase 2 (Escenario, Sujeto, Paleta Lumínica, Plano) como dirección de arte de los beats, no como una sola escena estática — un Reel puede cruzar más de un encuadre dentro de los mismos parámetros aprobados.
     - Este guion es el entregable de esta fase para el Reel — no se genera ningún render de video aquí; `/04_ensamblar` coordina la producción externa a partir de este guion.

   **Paso C: Escritura en Supabase**
   - `PATCH` sobre el `id` de esa fila con lo redactado en los Pasos A y B, los 4 parámetros visuales aprobados en la Fase 2 y `estado_copy = Listo`:
     ```bash
     curl -s -X PATCH "$SUPABASE_URL/rest/v1/content_pieces?id=eq.<id>" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY" \
       -H "Content-Type: application/json" -H "Prefer: return=minimal" \
       -d '{ "copy_instagram": "...", "copy_pinterest": "...", "copy_x": "...", "copy_linkedin": "...", "copy_gbp": "...",
             "prompt_visual": "...", "escenario": "...", "sujeto": "...", "paleta_luminica": "...", "plano": "...",
             "texto_laminas": [ ... solo en Carrusel ... ], "estado_copy": "Listo" }'
     ```
   - En filas `Reel`, deja en `null` las plataformas que no aplican en vez de rellenarlas con contenido genérico.
   - Escapa bien las comillas del copy dentro del JSON (o arma el cuerpo en un archivo temporal y usa `-d @archivo.json`): un copy con apóstrofes rompe el `curl` en silencio.
   - **NO llames a Magnific ni a Canva en este flujo** — eso ocurre en `/04_ensamblar` (tampoco produzcas el video del Reel aquí: el guion es el entregable de esta fase).

   **Paso D: Confirmación de Pieza**
   - Registra brevemente en el chat: `✅ [fecha]_[formato]` con el escenario asignado (o, si es Reel, el gancho del guion). Continúa con la siguiente fila.

   **FASE 3b: CORRECCIONES DE TEXTO DEL CLIENTE (Cola 2, solo las clasificadas como texto):**
   - Por cada una: muestra en el chat el comentario del cliente, el texto actual y tu propuesta corregida, y **espera confirmación** antes de escribir.
   - Tras confirmar, `PATCH` solo de las columnas de texto que cambian (copy, `texto_laminas`).
   - Si la corrección era **solo de texto**, la pieza vuelve a revisión del cliente: incluye en el mismo `PATCH` `"estado_aprobacion": "Pendiente"`. Deja `comentario_cliente` como está (el cliente lo verá como contexto al revisar de nuevo).
   - Si además pide cambios visuales, **no** cambies `estado_aprobacion`: lo hará `/04_ensamblar` cuando re-renderice. Para un **Carrusel** cuyo texto de láminas cambió, avisa que `/04_ensamblar` debe volver a montar las láminas aunque el comentario no hable de la foto.

4. **FASE 4: CIERRE DEL LOTE:**
   - Muestra un resumen: total de piezas, % de copy respaldado por investigación real (Web + Social) vs ángulo creativo, y correcciones aplicadas (cuántas volvieron a revisión, cuántas esperan a `/04_ensamblar`).
   - Sugiere: *"💡 Las filas quedaron con `estado_copy = Listo` en Supabase (en Partners se ven en Planificación como 'En diseño'). Cuando estés conforme, ejecuta `/04_ensamblar [nombre_del_cliente] [mes]` para generar los renders y las piezas finales de marca (las filas `Reel` se coordinan aparte para producción externa del video, a partir del guion ya escrito)."*
