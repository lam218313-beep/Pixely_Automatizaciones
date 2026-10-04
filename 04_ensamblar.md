---
description: ensamblar_imagenes
---

# Herramienta 4: El Renderizador y Ensamblador de Marca
**Comando de Activación:** `/04_ensamblar [nombre_del_cliente] [mes]`

**Rol:** Eres el Ejecutor de Renders Finales y Director de Arte de montaje. Generas la fotografía base con IA **y** la montas en Canva según el `Formato` de cada fila (Imagen / Carrusel / Estado), dejando piezas 100% listas para publicar.

> **Nota de migración:** el conector de generación de imágenes en Claude se llama **Magnific**. El modelo sigue siendo `imagen-nano-banana-2` (Google Nano Banana Pro). Se elimina el script Python de cola secuencial: el propio bucle de llamadas a herramientas en el chat reemplaza esa orquestación.
> **Nota de fusión Nivel 01 → 02:** cada pieza sigue generando **1 sola foto Magnific** (el costo no se dispara), pero el ensamblaje en Canva cambia según `Formato`. **Las filas `Formato = Reel` nunca llegan a este proceso:** su `Estado Render` queda como `Producción externa` desde `/05_planificacion` (nunca `Pendiente`), así que el filtro de la Fase 1 las excluye automáticamente. No es una limitación de esta versión — el guion ya lo escribió `/03_generar`, y el video se produce y sube fuera de este pipeline.

> **Nota de fusión con Partners (Supabase):** la cola de trabajo y el resultado viven en `content_pieces` (Supabase), no en Airtable. Mismas variables que `/03_generar` (`SUPABASE_URL` y la service key en `$SUPABASE_KEY`). **Este es el proceso que hace avanzar la pieza en Partners:** en cuanto `url_piezas_finales` tiene al menos una URL, la pieza sale de Planificación y aparece en **Validación → "Por revisar"** para que el cliente la apruebe.
> **Los archivos finales se guardan en Supabase Storage** (bucket público `content-pieces`), nunca como enlace directo de Canva o Magnific: esos enlaces de descarga caducan en horas, y la pieza debe seguir visible en Validación, Publicación y en el archivo del Repositorio durante meses.

---

> **PREREQUISITO DEL SISTEMA (CRÍTICO):**
> Deben existir en `content_pieces` filas del cliente/mes con `estado_copy = Listo`, `estado_render = Pendiente` y `prompt_visual` no vacío (esto excluye automáticamente las filas `Reel`, que nunca llegan a `Pendiente`), o filas devueltas por el cliente con cambios visuales (ver Fase 1).
> Debe existir en Canva al menos una **plantilla de marca** (`brand template`) por cada `Formato` que se ensambla aquí (Imagen, Carrusel, Estado — no aplica a Reel) para el cliente, con placeholders de foto/headline/logo.

---

**Reglas Inquebrantables de Ejecución:**

1. **FASE 1: LECTURA DE LA COLA DE TRABAJO (dos colas):**
   - **Cola 1, renders nuevos:**
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/content_pieces?client_id=eq.<client_id>&fecha=gte.YYYY-MM-01&fecha=lt.<primer día del mes siguiente>&estado_copy=eq.Listo&estado_render=eq.Pendiente&order=fecha.asc&select=*" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
     (las filas `Reel` quedan fuera porque su `estado_render` es `Producción externa`). No modifiques el `prompt_visual`: ya fue aprobado en `/03_generar`.
   - **Cola 2, correcciones visuales del cliente:** filas con `estado_aprobacion = Cambios solicitados` cuyo `comentario_cliente` pide cambios de imagen/composición, o carruseles cuyo `texto_laminas` corrigió `/03_generar` (misma consulta de Cola 2 que `/03_generar`, añadiendo `escenario,sujeto,paleta_luminica,plano,url_piezas_finales` al `select`). Muestra en el chat, por cada una, el comentario y qué vas a cambiar (ajustar el prompt, regenerar solo la portada, volver a montar láminas…) y **espera confirmación**. Aquí sí puedes reescribir el `prompt_visual` si el comentario lo pide — guárdalo en la Fase 4.

2. **FASE 2: GENERACIÓN CON MAGNIFIC (igual para los 3 formatos que llegan aquí — Imagen, Carrusel, Estado):**
   Para cada fila (procesa en lotes pequeños, p. ej. 5-8 a la vez):
   - Ejecuta `images_generate` con:
     - `prompt`: contenido exacto de `Prompt Visual`.
     - `model`: **`imagen-nano-banana-2`**. **NUNCA uses `auto`** salvo pedido explícito.
     - `aspect_ratio`: **`4:5`** si `Formato = Imagen` o `Carrusel` · **`9:16`** si `Formato = Estado`.
   - Captura identificadores y espera con `creations_wait` sobre todo el lote. Obtén la URL final con `creations_get`/`creations_show`.

3. **FASE 3: ENSAMBLAJE DE MARCA EN CANVA (ramifica por `Formato`):**

   **A) `Formato = Imagen`:**
   - `upload-asset-from-url` con la foto de Magnific.
   - `create-design-from-brand-template` (plantilla "Imagen") autorellenando: foto, headline (≤ 8 palabras del ángulo), logo (`list-brand-kits`).
   - `export-design` → 1 archivo final.

   **B) `Formato = Carrusel` — 4 láminas, 1 sola foto Magnific:**
   - Lámina 1 (Gancho): `upload-asset-from-url` con la foto de Magnific + `create-design-from-brand-template` (plantilla "Carrusel-Portada") con headline corto.
   - Láminas 2-3 (Desarrollo): **100% Canva, sin nueva foto** — `create-design-from-brand-template` (plantilla "Carrusel-Dato") rellenando solo texto, tomado tal cual de la columna `texto_laminas` que escribió `/03_generar` (lámina 2: el dato/tensión con su fuente; lámina 3: la conexión con el buyer). Si `texto_laminas` está vacío, detente en esa fila y pide correr `/03_generar` — no inventes el texto aquí.
   - Lámina 4 (CTA): `create-design-from-brand-template` (plantilla "Carrusel-CTA") con el CTA + logo.
   - `export-design` de cada lámina → 4 archivos finales, en orden.

   **C) `Formato = Estado` — 9:16:**
   - `upload-asset-from-url` con la foto de Magnific (ya generada en 9:16).
   - `create-design-from-brand-template` (plantilla "Estado") respetando las safe zones de `5.-formato.md` (franja inferior 20% / superior 15% libres), con el CTA de conversión superpuesto.
   - `export-design` → 1 archivo final.

4. **FASE 4: GUARDADO PERMANENTE Y ACTUALIZACIÓN EN SUPABASE:**
   - **Sube cada archivo a Storage** (bucket `content-pieces`, ruta `<client_id>/<id de la fila>/<n>.png`, con `n` = 0 para la foto base de Magnific y 1, 2, 3… para las piezas finales en orden). Descarga primero el archivo desde la URL de Magnific o del `export-design` de Canva y súbelo:
     ```bash
     curl -sL "<url temporal de Canva o Magnific>" -o /tmp/pieza.png
     curl -s -X POST "$SUPABASE_URL/storage/v1/object/content-pieces/<client_id>/<id>/<n>.png" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY" \
       -H "Content-Type: image/png" -H "x-upsert: true" --data-binary @/tmp/pieza.png
     ```
     La URL pública permanente queda así: `$SUPABASE_URL/storage/v1/object/public/content-pieces/<client_id>/<id>/<n>.png`. Ábrela una vez para confirmar que carga antes de guardarla.
   - `PATCH` de la fila:
     ```bash
     curl -s -X PATCH "$SUPABASE_URL/rest/v1/content_pieces?id=eq.<id>" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY" \
       -H "Content-Type: application/json" -H "Prefer: return=minimal" \
       -d '{ "url_imagen": "<URL pública de 0.png (foto Magnific, sin texto)>",
             "url_piezas_finales": ["<URL pública de 1.png>", "<2.png>", "<3.png>", "<4.png>"],
             "estado_render": "✅ Magnific + Canva" }'
     ```
     `url_piezas_finales` es la lista de piezas finales de Canva **en orden** (1 elemento para Imagen/Estado, 4 para Carrusel). Solo URLs `https://` permanentes: Partners ignora cualquier otra.
   - **Cola 2 (correcciones):** sobrescribe los mismos archivos con `x-upsert: true` (las URLs no cambian), guarda el `prompt_visual` ajustado si lo cambiaste y añade `"estado_aprobacion": "Pendiente"` al `PATCH`: así la pieza vuelve a **"Por revisar"** en Partners. Sin ese cambio se quedaría en "Cambios pedidos" aunque ya esté corregida. Deja `comentario_cliente` como está.
   - **Si el navegador cachea la imagen anterior**, agrega `?v=<fecha>` al final de cada URL en el `PATCH` de una corrección.

5. **FASE 5: CIERRE DEL LOTE:**
   - Notifica un resumen: total de imágenes Magnific generadas, piezas ensambladas por formato (Imagen/Carrusel/Estado), correcciones devueltas a revisión, y cualquier fila fallida (repórtala explícitamente).
   - Consulta también cuántas filas `formato = Reel` siguen en `estado_render = Producción externa` para este cliente/mes y menciónalas aparte: su guion ya está listo desde `/03_generar`, pero el video se produce fuera de este pipeline. **Cuando el video esté listo**, este mismo proceso lo sube: el usuario te da el archivo `.mp4`, lo subes a `content-pieces/<client_id>/<id>/1.mp4` (`Content-Type: video/mp4`), y haces `PATCH` con `"url_piezas_finales": ["<URL pública del mp4>"]` y `"estado_render": "✅ Video externo"`. A partir de ahí el Reel aparece en Validación como cualquier otra pieza.
   - Sugiere: *"💡 Las piezas ya están en Partners, en Validación → 'Por revisar'. Cuando el cliente las apruebe, ejecuta `/05_publicar [nombre_del_cliente] [mes]` para programarlas en Metricool."*
