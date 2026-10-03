# Pixely Automatizaciones

Procesos de Claude Desktop (slash-commands) que investigan mercado y planifican contenido a mano, con mucha supervisión humana — **nunca automatización ciega**, porque el mercado peruano no es confiable solo con datos scrapeados.

## Procesos activos (escriben en Supabase, influyen en Partners)

| Archivo | Qué hace | Tablas de Supabase |
|---|---|---|
| `00_genesis_cliente.md` | Estudio de mercado fundacional (una vez por cliente). Partners lo dibuja en Mercado y lo usa para generar la Estrategia | escribe `market_studies` |
| `01_escanearmercado.md` | Vigilancia competitiva recurrente. Partners la dibuja en Mercado y la usa para generar la Estrategia | lee `brand_identities`/`client_interviews`/`market_studies`, escribe `market_findings` |
| `02_crearcronograma_V2.md` | Cronograma de contenido del mes — **el único plan mensual del sistema** | lee `brand_identities`/`client_interviews`/`strategy_nodes`/`market_findings`/`market_studies`, escribe `content_pieces` |
| `03_generar.md` | Copy por red, prompt visual o guion de Reel, textos de láminas; corrige textos que el cliente devolvió | lee `content_pieces`/`market_*`/`brand_identities`/`client_interviews`, actualiza `content_pieces` |
| `04_ensamblar.md` | Foto Magnific + montaje Canva, guardado permanente en Storage; corrige lo visual que el cliente devolvió; sube los videos de Reel | actualiza `content_pieces`, sube a Storage `content-pieces` |
| `05_publicar.md` | Programa en Metricool solo lo que el cliente aprobó | lee y actualiza `content_pieces` |

## Proceso sin terminar (no toca Supabase todavía)

`06_reportar_cliente.md` — sigue apuntando al flujo viejo (local/Airtable). No lo uses asumiendo que está conectado a Partners.

### Contrato de `content_pieces` (02–05 y Partners)

Partners muestra `content_pieces` como **una línea de producción de 4 estaciones** (pasos 5 a 8); cada pieza está en una sola a la vez, y en cuál depende solo de estos campos:

| Paso | Quién escribe | Qué debe dejar escrito | Estación en Partners |
|---|---|---|---|
| Cronograma | `02` | fila nueva; `estado_copy`/`estado_render`/`estado_publicado` = `'Pendiente'` (Reel → `estado_render = 'Producción externa'`); `estado_aprobacion` queda en `'Pendiente'` por defecto | 5. Planificación ("En producción") |
| Copy | `03` | `copy_instagram`, `copy_linkedin`, `copy_pinterest`, `copy_gbp`, `copy_x`; `estado_copy = 'Listo'` | sigue en Planificación (pasa de "Copy" a "En diseño") |
| Render | `04` | `url_imagen` (portada) y `url_piezas_finales` (lista JSON de URLs `https://`, una por lámina); `estado_render = '✅ Magnific + Canva'`. **Mientras `url_piezas_finales` esté vacío, la pieza sigue en Planificación y el cliente no puede aprobarla.** | 6. Validación ("Por revisar") |
| Validación | **el cliente, desde Partners** | `estado_aprobacion` = `'Aprobado'` o `'Cambios solicitados'`, más `comentario_cliente`, `revisado_at`, `revisado_por`. Ninguna automatización debe escribir aquí, salvo lo de la fila siguiente. | Aprobado → 7. Publicación; Cambios → sigue en Validación ("Cambios pedidos") |
| Corrección | `04` | si `estado_aprobacion = 'Cambios solicitados'`: leer `comentario_cliente`, re-renderizar, sobrescribir `url_imagen`/`url_piezas_finales` y **volver a poner `estado_aprobacion = 'Pendiente'`** (si no, la pieza se queda en "Cambios pedidos" aunque ya esté corregida). | vuelve a "Por revisar" |
| Publicación | `05` | publicar **solo** filas con `estado_aprobacion = 'Aprobado'`, con `createScheduledPost` (programación directa); **nunca** con el flujo de revisión de Metricool (`createScheduledPostForReview`), o el cliente aprobaría dos veces. Al programar: `estado_publicado = '✅ Programado Metricool'`. | 7. Publicación ("Programada"); el día después de su `fecha` → 8. Repositorio ("Publicada") |


Notas del contrato:
- **Archivos:** `url_imagen` y `url_piezas_finales` siempre apuntan a Storage (`$SUPABASE_URL/storage/v1/object/public/content-pieces/<client_id>/<id>/<n>.png`), nunca a un enlace temporal de Canva o Magnific.
- **Carrusel:** `03` escribe `texto_laminas` (`[{"lamina":2,"titulo","texto"},{"lamina":3,...}]`); `04` lo usa tal cual para las láminas 2–3.
- **Reel:** el video se produce fuera del pipeline; `04` lo sube a Storage como `1.mp4` y marca `estado_render = '✅ Video externo'`. Partners lo muestra como video.
- **Correcciones:** `03` corrige texto, `04` corrige lo visual; quien termine la corrección pone `estado_aprobacion = 'Pendiente'` (si hay de ambos tipos, lo hace `04` al final).
---

## ⚠️ Este repo depende del esquema real de Partners — revisar tras cada cambio ahí

Los procesos activos asumen nombres exactos de tablas, columnas, restricciones y variables de entorno que viven en **otro repositorio** (`lam218313-beep/Pixely`, el backend/frontend de Partners). Nada sincroniza esto automáticamente — si algo cambia en Partners y nadie revisa estos `.md`, quedan desactualizados en silencio (ya pasó una vez: ver Changelog abajo).

**Cada vez que se modifique el esquema de Supabase o se quite/renombre algo en Partners, volver a verificar en los archivos activos:**

- [ ] Nombres de tabla y columna usados en los `curl` (`clients`, `market_studies`, `market_findings`, `content_pieces`, `brand_identities`, `client_interviews`, `strategy_nodes`) — ¿siguen existiendo exactamente así?
- [ ] Valores permitidos por restricciones `CHECK` (`cluster`, `confianza` en `market_findings`; `formato`, `pilar`, `marcador`, `estado_aprobacion` en `content_pieces`) — ¿la lista de valores sigue siendo la misma?
- [ ] Campos JSON que la pantalla de Mercado dibuja (lista en la Fase 5 de `00_genesis_cliente.md`) — ¿el componente `MercadoView.tsx` de Partners sigue leyendo esas mismas claves? El generador de Estrategia (`backend_v2/app/services/strategy_context.py`) lee las mismas, así que una clave renombrada empobrece las dos cosas.
- [ ] Nombres de canal en `market_findings.fuente` (lista en la Fase 5 de `01_escanearmercado.md`) — Mercado agrupa por ese texto exacto.
- [ ] Restricciones `UNIQUE` que algún `curl` asuma para `on_conflict` (ej. `market_studies.client_id`) — ¿siguen ahí si se recrea la tabla?
- [ ] Cualquier mención a una columna/feature específica de Partners (ej. el extinto `clients.plan`) — si se elimina algo en Partners, buscar su nombre en estos 3 archivos y corregir la nota.
- [ ] Nombres de las variables de entorno en Railway (`SUPABASE_URL`, `SUPABASE_SERVICE_KEY`) — si alguna vez se renombran ahí, actualizar la nota de cada archivo que las menciona.

Forma rápida de auditar: `grep -rn "clients\.\|market_studies\.\|market_findings\.\|content_pieces\." *.md` y comparar contra el esquema real en Supabase (`information_schema.columns`).

### Changelog de desincronizaciones encontradas y corregidas

- **2026-10-03** — `00_genesis_cliente.md` decía que la `SUPABASE_SERVICE_KEY` "está en Railway como `SUPABASE_KEY`" (es la llave anon, un valor distinto); `02_crearcronograma_V2.md` decía que "Partners ya guarda el plan contratado en `clients.plan`" (esa columna fue eliminada por completo); y el `POST` a `market_studies` usaba `Prefer: resolution=merge-duplicates` sin que la tabla tuviera una restricción `UNIQUE (client_id)` ni el `curl` incluyera `on_conflict=client_id`, así que cada re-ejecución duplicaba la fila en vez de actualizarla. Se corrigieron los 3 y se agregó la restricción `UNIQUE` en Supabase.
- **2026-10-03 (2)** — Partners agregó las fases Repositorio, Validación y Publicación (leen `content_pieces`) y nuevas columnas de revisión del cliente (`estado_aprobacion`, `comentario_cliente`, `revisado_at`, `revisado_por`); la fase Beneficios se eliminó. `02_crearcronograma_V2.md` decía que las piezas aparecían en "la fase Planificación de Partners" (esa fase lee `tasks`, no `content_pieces`) — corregido. La pantalla de Mercado se rediseñó y ahora dibuja `tamano_mercado.rango_estimado` y los números de `listado`/`estadisticas_precio` — documentado en la Fase 5 de `00_genesis_cliente.md`. Se agregó arriba el contrato que deberán cumplir 03–05 (sin editar esos archivos todavía).
- **2026-10-03 (3)** — Había **dos planes mensuales en paralelo**: el de `02` (`content_pieces`) y otro que Partners generaba con IA desde su panel de Admin (tabla `tasks`, mostrado como Kanban en Planificación, con su propio flujo de aprobación). Se eliminó el segundo por completo (vistas, endpoints `/planning` y `/tasks`, generador): `content_pieces` es ahora el único plan y Planificación lo muestra. Como el generador eliminado se basaba en la estrategia de Partners, `02` ahora lee `strategy_nodes` en su Fase 1. Además, los pasos 6–8 se reordenaron (Validación → Publicación → Repositorio) para que cada pieza esté en una sola estación; ver la columna "Estación en Partners" del contrato. La tabla `tasks` sigue en la base de datos, sin uso.
- **2026-10-03 (4)** — Se conectaron `03`, `04` y `05` a Supabase (antes leían y escribían Airtable). Para eso se agregaron en Supabase la columna `content_pieces.texto_laminas` (textos de las láminas 2–3 del carrusel, de `03` a `04`) y el bucket público de Storage `content-pieces` (los enlaces de Canva/Magnific caducan en horas; el archivo del Repositorio debe durar meses). `03` y `04` ahora atienden también la cola de correcciones del cliente (`Cambios solicitados`), cada uno su parte (texto / visual), y devuelven la pieza a `Pendiente` al corregirla. `05` publica solo lo `Aprobado`, ya sin turnos Mañana/Tarde/Noche, e incluye Instagram (post, carrusel, historia o reel según el formato). El comando pasó a ser `/05_publicar`.
- **2026-10-03 (5)** — El Manual de marca de Partners pasó a ser **Voz de marca**: solo lo que usan 01–03 (rasgos de tono con ejemplos sí/no, `palabras_si`, `palabras_no`, arquetipo y su razón, `ejemplo_post`) más los colores reales que fija el cliente. El cliente la aprueba (`voz_estado`). Nuevas columnas en `brand_identities`; misión, visión y valores ya no se generan. 01–03 leen esos campos; 03 prohíbe las `palabras_no` y avisa si la voz no está aprobada. El generador de Partners tenía un error que le pasaba la entrevista vacía (por eso salían manuales genéricos): corregido.
- **2026-10-03 (6)** — Partners eliminó **Análisis** (el estudio de los comentarios del Instagram del cliente, Q1–Q10) y su Wiki: con pocas interacciones no era confiable. Desde ahora la **Estrategia se genera con la Ficha + Mercado**: el estudio de `00`, los hallazgos de `01` (los de mayor confianza primero) y la Voz de marca. Así, `00` y `01` ya no solo alimentan la pantalla de Mercado: también la Estrategia que `02` lee en `strategy_nodes`. La pantalla de Mercado se rediseñó con gráficos y ahora dibuja más campos (`total_relevante_filtrado`, `total_detectado_maps`; documentado en la Fase 5 de `00`) y agrupa los hallazgos por `fuente`, así que `01` fija una lista de nombres de canal (Fase 5 de `01`). La tabla `analysis_reports` queda en la base de datos, sin uso. Ningún proceso de este repo usaba Análisis.
- **2026-10-03 (7)** — La Estrategia de Partners perdía su mejor contenido: el generador escribía `strategic_rationale`, `creative_hooks` y `execution_guidelines` pero no se guardaban, y cada autoguardado de la pantalla los borraba (por eso `02` siempre los recibía vacíos). Corregido; además abrir la pantalla ya no reescribe la estrategia. Los objetivos ahora tienen títulos concretos (antes todos se llamaban "Objetivo Principal/Secundario") y la prioridad va en `tags` (`principal`/`secundario`); las estrategias guardan su porqué en `description`. Nota agregada en la Fase 1 de `02`. Las estrategias existentes necesitan regenerarse para tener estos datos.
