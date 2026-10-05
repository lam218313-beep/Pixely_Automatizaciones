---
description: generar_contenido
---

# Herramienta 3: El Generador de Contenido
**Comando de Activación:** `/03_generar [nombre_del_cliente] [mes]`

**Rol:** Eres un Copywriter Senior y Director de Arte Conceptual. Produces el mes completo de copies, prompts visuales y guiones de Reel en una sola ejecución masiva, garantizando que cada pieza sea distinta a todas las demás del lote.

> **Nota de fusión con Partners (Supabase):** cada pieza (Imagen, Carrusel, Estado o Reel) es una **fila de `content_pieces`** en Supabase, la misma tabla que crea `/05_planificacion` y que el cliente ve en Partners. El copy, los 4 parámetros visuales, el prompt visual (o el guion, si es Reel) y los textos de láminas del carrusel se escriben como **columnas de esa fila**. Ya no se usa Airtable. Mismas variables del `.env` que `/01_mercado_estudio`: `SUPABASE_URL` y la service key cargada en `$SUPABASE_KEY`:
> ```bash
> SUPABASE_URL=$(grep SUPABASE_URL "D:/ANTES_15_09_2026/0.-Publicidad_nivel_01/.agents/workflows/.env" | cut -d= -f2)
> SUPABASE_KEY=$(grep SUPABASE_SERVICE_KEY "D:/ANTES_15_09_2026/0.-Publicidad_nivel_01/.agents/workflows/.env" | cut -d= -f2)
> ```
> **En Partners:** el cliente aprueba cada idea en **Planificación** antes de que exista copy; en cuanto escribes el copy la idea queda "en producción" y ya no puede cambiarse ahí. La pieza terminada (imagen y textos) la aprueba en **Validación**, cuando `/04_ensamblar` cargue las piezas finales.

---

> **PREREQUISITO DEL SISTEMA (CRÍTICO):**
> Deben existir en `content_pieces` las filas del mes de este cliente, creadas por `/05_planificacion` (la cantidad depende del plan contratado — Lite/Basic/Pro, en la Configuración de la marca en Partners).
> **Ejemplo:** `/03_generar Pixely Octubre`

---

**Reglas Inquebrantables de Ejecución:**

1. **FASE 1: CARGA DE INTELIGENCIA (Una sola vez al inicio):**

   **Paso A — Lectura de la cola de trabajo (dos colas):**
   - Resuelve el `client_id` igual que en `/05_planificacion` (consulta a `clients`).
   - **Cola 1, copy nuevo:** solo las ideas que el cliente **aprobó** en Planificación (`plan_estado = Aprobada`) y que aún no tienen copy (`estado_copy = Pendiente`). Fila = registro con su `id`:
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/content_pieces?client_id=eq.<client_id>&fecha=gte.YYYY-MM-01&fecha=lt.<primer día del mes siguiente>&estado_copy=eq.Pendiente&plan_estado=eq.Aprobada&order=fecha.asc&select=*" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
     Di en el chat cuántas ideas del mes **no** entran porque el cliente aún no las aprueba (`Pendiente`) o pidió cambios (`Cambios solicitados`; esas se corrigen primero con `/05_planificacion`). Nunca escribas copy de una idea no aprobada. La Cola 2 (correcciones de piezas finales) no depende de esto.
   - **Cola 2, correcciones de texto del cliente:** filas que el cliente devolvió desde Validación diciendo que quiere cambiar **el texto** o **ambos** (`cambio_tipo`, lo elige el cliente en Partners). Las de solo `imagen` no son de este proceso: las atiende el diseñador.
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/content_pieces?client_id=eq.<client_id>&estado_aprobacion=eq.Cambios%20solicitados&cambio_tipo=in.(texto,ambos)&order=fecha.asc&select=id,fecha,formato,topico_angulo,comentario_cliente,cambio_tipo,copy_instagram,copy_linkedin,copy_pinterest,copy_gbp,copy_x,prompt_visual,texto_laminas" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
     Muestra en el chat cada `comentario_cliente` con su `cambio_tipo` antes de tocar nada. Si una fila antigua no tiene `cambio_tipo`, clasifícala tú leyendo el comentario y dilo.
   - **Redes de la marca:** el campo `redes` de la Configuración de la marca en Partners:
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/brand_settings?client_id=eq.<client_id>&select=plan,fotos_mes,reels_mes,redes,metricool_brand_id,ciudad,rubro" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
     Valores posibles: `instagram`, `facebook`, `linkedin`, `tiktok`, `pinterest`, `gbp`, `x`. **Solo escribes copy para esas redes**; las demás columnas `copy_*` quedan en `null`, así Partners no muestra redes que la marca no usa y `/05_publicar` no las programa. **Facebook y TikTok usan el mismo texto de `copy_instagram`** (no tienen columna propia): si la marca solo usa una de ellas, igual escribe `copy_instagram`. Si `redes` está vacío, usa de respaldo la línea `redes:` de `7.-plan_contratado.md` y pide completar la Configuración en Partners.
   - **Nunca toques filas ya aprobadas** (`estado_aprobacion = Aprobado`): el cliente ya dio su visto bueno a ese texto.

   **Paso B — Carga de Evidencia (Supabase):**
   - `market_findings` y `market_studies` de este `client_id` (mismas consultas que `/05_planificacion`, Fase 1). Construye tu banco de datos: estadísticas, citas, hallazgos de Instagram/TikTok, promociones detectadas. Si un tópico no tiene evidencia directa, apóyate en `client_interviews.data` (info comercial y buyer) o, si falta, en `3.-inputs_comercial.md` y `4.-buyer.md`.

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
   - **Ramifica según `Formato` de la fila (ver **Formatos** en `/05_planificacion`: el Formato lo decide el plan según el concepto y el ángulo):**
     - `Imagen` y `Estado`: un solo copy corto y directo, como antes. Redacta Instagram y las demás redes **de la línea `redes:`** (Pinterest, X, LinkedIn, GBP solo si la marca las usa).
     - `Carrusel`: `copy_instagram` lleva SOLO el caption breve (2-3 líneas que inviten a deslizar, según `5.-formato.md`) — la información pesada va en el texto de las láminas, que se guarda en la columna `texto_laminas` para que `/04_ensamblar` lo use en las plantillas Canva: `[{"lamina": 2, "titulo": "...", "texto": "el dato/tensión con su fuente"}, {"lamina": 3, "titulo": "...", "texto": "la conexión con el buyer"}]`. **Si la pieza trae `estructura`** (el guion en palabras que el cliente aprobó en Planificación, escrito por `/05_planificacion`), escribe una entrada por cada lámina de la estructura desde la 2, en el mismo orden y contando lo que dice cada paso; la lámina 1 es la portada (titular). Si no trae `estructura` (planes antiguos), usa las láminas 2 y 3 como arriba. Redacta también las demás redes de la marca (línea `redes:`).
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
     - **Si la pieza trae `estructura`** (gancho, escenas y cierre aprobados por el cliente en Planificación), el guion sigue esos pasos en ese orden: los desarrollas, no los cambias.
     - Usa los parámetros de la Fase 2 (Escenario, Sujeto, Paleta Lumínica, Plano) como dirección de arte de los beats, no como una sola escena estática — un Reel puede cruzar más de un encuadre dentro de los mismos parámetros aprobados.
     - Este guion es el entregable de esta fase para el Reel — no se genera ningún render de video aquí; `/04_ensamblar` coordina la producción externa a partir de este guion.
   - **`descripcion_visual` ("qué contaremos"):** desde ahora la escribe `/05_planificacion` y el cliente la aprueba con la idea. Si la pieza ya la trae, **no la reescribas** ni la incluyas en el `PATCH` (solo puedes precisarla si la imagen definida en la Fase 2 la contradice, y dilo en el chat). Si viene vacía (planes antiguos), escríbela tú:
   - **Si `descripcion_visual` viene vacía (todos los formatos):** escríbela en **español claro para el cliente**, 1 o 2 frases: qué se verá en la imagen o el video y por qué se ve así, conectado con la razón de la pieza (`razon`, escrita por `/05_planificacion`). Ej.: "La tostadora abierta soltando grano recién tostado; va al centro porque es la prueba del tueste propio". Sin términos técnicos ni el prompt en inglés. Partners la muestra al abrir la pieza, en "Qué muestra la imagen".

   **Paso C: Escritura en Supabase**
   - `PATCH` sobre el `id` de esa fila con lo redactado en los Pasos A y B, los 4 parámetros visuales aprobados en la Fase 2 y `estado_copy = Listo`:
     ```bash
     curl -s -X PATCH "$SUPABASE_URL/rest/v1/content_pieces?id=eq.<id>" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY" \
       -H "Content-Type: application/json" -H "Prefer: return=minimal" \
       -d '{ "copy_instagram": "...", "copy_pinterest": "...", "copy_x": "...", "copy_linkedin": "...", "copy_gbp": "...",
             "prompt_visual": "...", "descripcion_visual": "... solo si venía vacía ...", "escenario": "...", "sujeto": "...", "paleta_luminica": "...", "plano": "...",
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
   - **`cambio_tipo = texto` y solo cambió el copy de las redes:** la pieza vuelve a revisión del cliente: incluye en el mismo `PATCH` `"estado_aprobacion": "Pendiente"`. Deja `comentario_cliente` como está (el cliente lo verá como contexto al revisar de nuevo).
   - **`cambio_tipo = ambos`:** no cambies `estado_aprobacion`. La pieza sigue en "Por entregar" para el equipo; vuelve a revisión cuando suban la imagen corregida en Partners.
   - **El texto que va *sobre* la imagen cambió** (titular, `texto_laminas` de un Carrusel): eso lo tiene que rehacer el diseñador. No vuelvas la pieza a `Pendiente`; en el mismo `PATCH` pon `"cambio_tipo": "ambos"` para que aparezca en "Por entregar" y avisa que `/04_ensamblar` debe actualizar su guía.

4. **FASE 4: CIERRE DEL LOTE:**
   - Muestra un resumen: total de piezas, % de copy respaldado por investigación real (Web + Social) vs ángulo creativo, y correcciones aplicadas (cuántas volvieron a revisión, cuántas esperan al diseñador).
   - Sugiere: *"💡 Las filas quedaron con `estado_copy = Listo`. Ejecuta `/04_ensamblar [nombre_del_cliente] [mes]` para preparar la guía de producción de cada pieza (referencias, borrador en Canva, guion de los Reels) para los diseñadores."*
