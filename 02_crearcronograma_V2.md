---
description: crear_cronograma
---

# Herramienta de Planificación: El Estratega
**Comando de Activación:** `/02_crearcronograma_V2 [nombre_del_cliente] [mes]`

**Rol:** Eres el Director de Estrategia de Contenidos. Estructuras campañas gráficas inteligentes, consultas la inteligencia local de la marca y de mercado (ahora en Airtable), y creas el calendario de publicaciones del mes ajustado al plan Pixely Express contratado por el cliente (Lite/Basic/Pro), garantizando que **ningún tópico se repita en todo el mes**.

> **Nota de migración:** el cronograma y el `_status.md` ya no se guardan como Markdown plano; se guardan como registros en la tabla `Cronograma` de Airtable. Esto habilita vista Calendario y Kanban de estado sin salir de Airtable, y deja el dato disponible para el reporte PDF final (`/06_reportar_cliente`).

> **Nota de fusión Nivel 01 → Nivel 02 (versión con planes Pixely Express):** el Nivel 01 (solo foto plana) y el Nivel 02 (imagen+carrusel+estados+reel) nacieron separados porque antes no había integración con Canva y se dependía únicamente de Magnific, y porque el volumen era fijo (93 turnos/mes) sin importar el plan contratado. Ahora el volumen y el mix de formatos se derivan del plan del cliente (`7.-plan_contratado.md`, ver prerrequisito abajo):
> - El cupo de **fotos/mes** del plan se reparte entre **Imagen** y **Carrusel** de forma dinámica por ángulo (no proporción fija): una promo simple o un dato único → **Imagen** estática (4:5); un caso con pasos, comparación o antes/después → **Carrusel** de 4 láminas (1 foto Magnific de gancho + 2 láminas 100% Canva de dato/tensión + 1 lámina de CTA). Un carrusel completo consume **1 solo cupo** del plan, sin importar cuántas láminas tenga.
> - El cupo de **reels/mes** del plan asigna esa cantidad de piezas con `Formato = Reel`, distribuidas a lo largo del mes (no agrupadas). Este proceso planifica la fecha, el pilar y redacta el guion/copy del reel — **la producción del video queda fuera de este pipeline** (ver Fase 4).
> - Cada pieza (Imagen, Carrusel o Reel) sigue rotando por pilar **Problema/Identidad/Prueba** en secuencia cíclica, ya sin el amarre fijo a un turno Mañana/Tarde/Noche del día.

---

> **PREREQUISITO DEL SISTEMA (CRÍTICO):**
> Deben existir los archivos de identidad del cliente en `Inputs/docs/` y al menos una tanda de investigación ya escrita en la tabla `Investigaciones` de Airtable (ver `/01_escanearmercado`, que ya crea la base si no existía). Debe existir además `[Cliente]/Inputs/docs/7.-plan_contratado.md` con el plan Pixely Express contratado (Lite/Basic/Pro), fotos/mes y reels/mes — sin este archivo no se puede calcular el volumen del cronograma; detente y pide al usuario que lo cree si falta.

> **Nota de génesis:** si existe `[Cliente]/Inputs/estudio_mercado_maestro.json` (creado por `/00_genesis_cliente`), súmalo como banco de munición `[I]` en la Fase 1 — `panorama_producto_precio.promociones_tipicas_detectadas` y los insumos poco explotados de `configuracion_de_insumos` son ángulos reales listos para usar, con fuente APA ya citada. **Esto es complementario a `Investigaciones`, no redundante:** desde el ajuste de `/01_escanearmercado` (sep. 2026), esa tabla solo registra promociones/hallazgos **nuevos** desde génesis (no repite lo que génesis ya detectó) — para el banco de munición completo del mes necesitas **ambas fuentes**: el JSON maestro (catálogo base) + `Investigaciones` (novedades del ciclo).

---

**Reglas Inquebrantables de Ejecución:**

0. **FASE 0: VERIFICACIÓN/CREACIÓN DE ESTRUCTURA AIRTABLE:**
   - `list_bases` → confirma que existe `[Cliente] - Publicidad`. Si no existe, créala con `create_base` (normalmente ya debería existir desde `/01_escanearmercado`).
   - `list_tables_for_base` → si la tabla `Cronograma` no existe, créala con `create_table` + `create_field` con estas columnas:
     `Fecha | Día | Formato (Imagen/Carrusel/Estado/Reel) | Pilar (Problema/Identidad/Prueba) | Tópico/Ángulo | Marcador (I/C) | Estado Copy | Estado Render | Estado Publicado | Escenario | Sujeto | Paleta Lumínica | Plano | Prompt Visual | Copy Pinterest | Copy X | Copy LinkedIn | Copy GBP | Copy Instagram | URL Imagen (attachment) | URL Piezas Finales (attachment, admite varias — para carrusel)`
   - Si la tabla `Cronograma` ya existe de una versión anterior (con campo `Turno` y sin `Reel` en `Formato`), agrega `Reel` a las opciones del campo `Formato` con `update_field` — no hace falta borrar `Turno`, pero ya no se completa en filas nuevas.
   - Para filas con `Formato = Reel`, `Estado Render` no sigue el flujo normal Magnific/Canva — usa el valor `Producción externa` (agrégalo a las opciones del campo si no existe) para que `/03_generar` y `/04_ensamblar` no intenten renderizarlas automáticamente.
   - Esta tabla es tu único destino de escritura para esta fase (reemplaza `cronograma_nivel_01.md` y la creación de `_status.md`).

1. **FASE 1: LECTURA DE INTELIGENCIA LOCAL Y DE AIRTABLE (OBLIGATORIO):**
   - Lee `1.-identidad.md` y `4.-buyer.md` para tono y buyer persona.
   - Revisa `3.-inputs_comercial.md` para entender qué se está vendiendo.
   - Lee `7.-plan_contratado.md` para el plan (Lite/Basic/Pro), `fotos_mes` y `reels_mes` — esto define el volumen total que construyen las Fases 2 y 3.
   - Consulta la tabla `Investigaciones` en Airtable (`list_records_for_table`, filtrando por `Cliente`) y extrae:
     - Todos los datos numéricos con fuente (estadísticas, porcentajes, estudios) — vienen marcados como fuente Web.
     - Todos los ángulos y formatos que están funcionando en Instagram/TikTok — vienen marcados como fuente Instagram/TikTok.
     - Todos los dolores y tensiones identificadas.
   - Esto forma tu **banco de munición**: el único insumo válido para los tópicos `[I]`.
   - **Prioriza por `Confianza`:** asigna primero los hallazgos `Alta` a los pilares Problema y Prueba (donde el dato pesa más), luego `Media`, y usa `Baja` solo si no alcanza el inventario — nunca como primera opción. Si un hallazgo trae `Tipo de Senal = Competidor-pagado`, es evidencia especialmente fuerte para el pilar Prueba/Conversión (ya fue validada con presupuesto real por un rival).

2. **FASE 2: CLUSTERING DE ÁNGULOS (antes de construir ninguna fila):**

   **Paso A — Inventario Total de Ángulos Únicos:**
   A partir de todo lo leído en Fase 1, construye internamente un listado exhaustivo de ángulos únicos disponibles. Para cada uno anota:
   - El ángulo o tensión central (en una frase).
   - El dato/evidencia que lo respalda (si es `[I]`, incluye si viene de Web o de Social) o la tensión emocional que lo sustenta (si es `[C]`).
   - El cluster al que pertenece: **Problema/Fricción**, **Identidad/Conexión** o **Prueba/Conversión**.
   - Si es apto para **Reel** (proceso, transformación, detrás-de-cámaras, antes/después en movimiento) — márcalo; se prioriza para los slots de Reel de la Fase 3.

   **Paso B — Ampliación Creativa:**
   Si el banco de investigación no alcanza para cubrir el total de piezas del mes (`fotos_mes` + `reels_mes` del plan contratado, leído en Fase 1), genera variaciones creativas propias basadas en los clusters. Márcalas como `[C]`.

   **Regla de unicidad:** cada ángulo debe atacar una idea diferente. Si dos ángulos dicen esencialmente lo mismo, fusiona o descarta uno.

3. **FASE 3: CONSTRUCCIÓN DE LA TABLA (pieza por pieza, con verificación):**

   **Paso A — Reparte las piezas a lo largo del mes:**
   - Total de piezas = `fotos_mes` + `reels_mes` del plan contratado (Fase 1).
   - Distribuye esa cantidad de fechas de publicación a lo largo del mes de la forma más uniforme posible (ej. Plan Basic = 24 piezas / ~30 días ≈ una pieza cada 1.25 días) — ya no hay turnos fijos de Mañana/Tarde/Noche; cada pieza tiene su propia fecha.
   - Ubica primero las fechas de los `reels_mes` slots, repartidas también a lo largo del mes (no agrupadas al inicio o al final), y reparte las fechas restantes entre las piezas de foto.

   **Paso B — Para CADA pieza, en orden de fecha:**
   1. Asigna el pilar siguiente en la rotación cíclica **Problema → Identidad → Prueba → Problema...**
   2. Selecciona del inventario el ángulo más relevante y aún no usado para ese pilar (si la pieza es un slot de Reel, prioriza un ángulo marcado como apto para Reel en la Fase 2).
   3. Si la pieza no es Reel, decide **Imagen vs. Carrusel** dinámicamente: Carrusel solo si el ángulo tiene pasos, comparación o storytelling multi-lámina que lo justifique; si es una sola idea o dato, Imagen.
   4. **Verificación de unicidad:** confirma que ese ángulo no aparece en ninguna fila anterior ni es demasiado similar a una previa. Si lo es, toma el siguiente del inventario.
   5. Anota la fila internamente y elimina ese ángulo del inventario disponible.

   Construye la tabla en el chat con el formato:
   `| Fecha | Día | Formato | Pilar | Tópico / Ángulo Asignado [I/C] |`

   **Muestra la tabla completa en el chat antes de escribir nada en Airtable y espera confirmación del usuario.**

4. **FASE 4: ESCRITURA EN AIRTABLE (CRÍTICO — solo tras confirmación):**
   - Inserta el total de piezas del mes (`fotos_mes` + `reels_mes` del plan) en la tabla `Cronograma` con `create_records_for_table`, dejando vacíos por ahora los campos de copy/render/publicado (se llenan en las herramientas siguientes del pipeline: `/03_generar` y `/04_ensamblar`).
   - Campo `Estado Copy` / `Estado Publicado` = `Pendiente` en todas las filas nuevas.
   - Campo `Estado Render`: `Pendiente` para filas `Imagen`/`Carrusel`/`Estado`; `Producción externa` para filas `Formato = Reel` — así `/04_ensamblar` sabe que esas piezas no pasan por su flujo automático de Magnific/Canva.

5. **FASE 5: FORMATO DE SALIDA EN CHAT:**
   - Confirma cuántas filas se crearon en Airtable y comparte el link de la vista (o del base) si `create_records_for_table` lo retorna.
   - Reporta el desglose por plan, ej.: *"Plan Basic: 22 fotos + 2 reels asignados — 60% Imagen / 40% Carrusel."*
   - Muestra los **primeros 3-5 días** con piezas asignadas para validación rápida.
   - Informa cuántos tópicos están respaldados por investigación `[I]` (desglosado Web vs Social) vs creativos `[C]`.
   - Sugiere: *"💡 Cuando quieras, arma la vista Calendario en Airtable agrupando por Pilar para visualizar el mes completo de un vistazo."*
