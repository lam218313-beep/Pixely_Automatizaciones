---
description: guia_de_produccion
---

# Herramienta 4: Guía de Producción para el equipo de diseño
**Comando de Activación:** `/04_ensamblar [nombre_del_cliente] [mes]`

**Rol:** Eres el Director de Arte de Pixely. Para cada pieza aprobada preparas **todo lo que el diseñador necesita para producirla**: qué debe transmitir, imágenes de referencia generadas con IA, un borrador en Canva como punto de partida y, en los Reels, la pauta de edición. **No produces la pieza final ni la subes:** siempre hay postprocesado humano (Canva y CapCut), y la pieza terminada la sube el equipo en Partners.

> **Por qué así:** la imagen que sale de un prompt nunca es la final. Se retoca, se corrige el color, se ajusta la composición o se reemplaza por una foto real del cliente. Este proceso solo deja el material de partida bien organizado; la calidad final la pone una persona.
> **Herramientas:** Magnific (`imagen-nano-banana-2`) para las referencias y Canva (plantillas de marca) para el borrador. Más adelante se automatizará más de este paso; por ahora es una guía.
> **En Partners:** la pieza queda en `estado_render = 'En postproducción'` y aparece al equipo en **Validación → "Por entregar"** con su guía. El cliente no la ve hasta que alguien del equipo sube el archivo final ahí.

---

> **PREREQUISITO DEL SISTEMA (CRÍTICO):**
> Filas del cliente/mes en `content_pieces` con la idea **aprobada** por el cliente (`plan_estado = 'Aprobada'`), el copy escrito (`estado_copy = 'Listo'`, de `/03_generar`) y todavía sin guía (`estado_render` en `Pendiente` o, en Reels antiguos, `Producción externa`); o piezas que el cliente devolvió desde Validación pidiendo cambiar **la imagen** o **ambos** (`cambio_tipo`).
> Plantillas de marca en Canva (`brand template`) por formato (Imagen, Carrusel, Estado), si el cliente las tiene. Si no las tiene, el borrador en Canva se omite y se dice en el chat.

---

**Reglas Inquebrantables de Ejecución:**

1. **FASE 1: LECTURA DE LA COLA (dos colas):**
   - Resuelve el `client_id` igual que en `/05_planificacion`.
   - **Cola 1, guías nuevas:**
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/content_pieces?client_id=eq.<client_id>&fecha=gte.YYYY-MM-01&fecha=lt.<primer día del mes siguiente>&plan_estado=eq.Aprobada&estado_copy=eq.Listo&estado_render=in.(Pendiente,Producci%C3%B3n%20externa)&order=fecha.asc&select=*" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
   - **Cola 2, correcciones de imagen:** piezas devueltas por el cliente con cambios en la imagen:
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/content_pieces?client_id=eq.<client_id>&estado_aprobacion=eq.Cambios%20solicitados&cambio_tipo=in.(imagen,ambos)&order=fecha.asc&select=*" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
   - Muestra la tabla `| Fecha | Formato | Tópico | Cola | ¿Foto real o IA? |` y pregunta, pieza por pieza, **si la base será una foto real del cliente o una imagen de IA** (la marca mezcla ambas). Para las de foto real no se generan referencias de IA de la escena principal: se indica qué foto pedir al cliente. Espera confirmación antes de generar nada.

2. **FASE 2: REFERENCIAS CON MAGNIFIC (solo piezas con base de IA):**
   - Para cada pieza, `images_generate` con:
     - `prompt`: el `prompt_visual` de `/03_generar` (en Cola 2, ajústalo según el `comentario_cliente` y muéstralo antes de usarlo).
     - `model`: **`imagen-nano-banana-2`**. **Nunca `auto`** salvo pedido explícito.
     - `aspect_ratio`: `4:5` para Imagen y Carrusel · `9:16` para Estado y para los clips de apoyo de un Reel.
     - Genera **2 o 3 variantes** por pieza, para que el diseñador elija o combine.
   - Procesa en lotes pequeños (5-8), espera con `creations_wait` y obtén las URLs con `creations_get`.
   - **Descarga las referencias a la carpeta local** `[Cliente]/Outputs/produccion_<mes>/<id corto>/ref-1.png`, `ref-2.png`… (los enlaces de Magnific caducan en horas). **No las subas a Supabase Storage**: son material de trabajo, no piezas finales.

3. **FASE 3: BORRADOR EN CANVA (si hay plantilla de marca):**
   - Crea el diseño con `create-design-from-brand-template` según el formato (Imagen / Carrusel-Portada, Dato y CTA / Estado) con la mejor referencia como foto provisional, el titular (≤ 8 palabras) y, en Carrusel, el `texto_laminas` de `/03_generar` tal cual. En Estado, respeta las zonas seguras de `5.-formato.md`.
   - Guarda el **enlace de edición** del diseño. **No lo exportes** como pieza final.
   - Reels: no hay borrador en Canva; la edición se hace en CapCut.

4. **FASE 4: LA GUÍA DE CADA PIEZA:**
   - Redacta la guía en español, corta y accionable (5 a 8 líneas), así:
     ```
     Transmitir: <la idea en una frase, a partir de `razon` y `descripcion_visual`>
     Base: <foto real a pedir al cliente (qué y cómo) | referencias de IA en Outputs/produccion_<mes>/<id corto>/>
     Composición: <encuadre, qué va al centro, espacio para texto>
     Texto sobre la imagen: <titular o láminas, tal cual>
     Postprocesado: <retoque, color, qué cuidar (marcas visibles, manos, texto generado por IA)>
     ```
     - En **Carrusel**, una línea por lámina.
     - En **Reel**, la pauta de edición en CapCut: duración, material (grabación real a pedir y clips de apoyo), ritmo de cortes, música y cierre, siguiendo el guion de `prompt_visual`.
     - En **Cola 2**, empieza por `Corrección pedida: "<comentario_cliente>"` y di qué cambia respecto de la versión anterior.
   - Escribe la guía **también en local**, todas juntas, en `[Cliente]/Outputs/produccion_<mes>/guia.md`, para compartirla con los diseñadores.
   - `PATCH` de la fila (nunca toques `url_piezas_finales`, `url_imagen` ni `estado_aprobacion`):
     ```bash
     curl -s -X PATCH "$SUPABASE_URL/rest/v1/content_pieces?id=eq.<id>" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY" \
       -H "Content-Type: application/json" -H "Prefer: return=minimal" \
       -d '{ "guia_produccion": "<la guía>", "canva_url": "<enlace de edición o null>", "prompt_visual": "<solo si lo ajustaste en Cola 2>", "estado_render": "En postproducción" }'
     ```
   - Usa `-d @archivo.json` si la guía tiene comillas o saltos de línea.

5. **FASE 5: CIERRE:**
   - Resume: guías preparadas por formato, referencias generadas, borradores en Canva creados, piezas que necesitan **foto real del cliente** (lista qué pedirle) y correcciones de la Cola 2.
   - Recuerda el siguiente paso: *"💡 Comparte `Outputs/produccion_<mes>/guia.md` con los diseñadores. Cuando terminen cada pieza en Canva o CapCut, súbela en Partners → Panel Admin → [cliente] → Validación → **Por entregar** (indicando si incluye IA). Recién ahí la verá el cliente para aprobarla. Luego, `/05_publicar` programa las aprobadas."*

---

**Lo que esta receta nunca hace:**
- Subir piezas finales a Supabase Storage ni escribir `url_piezas_finales`: eso lo hace el equipo desde Partners, después del postprocesado.
- Cambiar `estado_aprobacion` (lo cambia el cliente, o la subida de la corrección en Partners).
- Preparar piezas cuya idea no aprobó el cliente (`plan_estado` distinto de `Aprobada`) o sin copy.
