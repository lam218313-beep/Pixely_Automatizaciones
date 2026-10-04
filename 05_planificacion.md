---
description: planificacion
---

# Herramienta de Planificación: El Planificador
**Comando de Activación:** `/05_planificacion [nombre_del_cliente] [mes]` (mes en formato `YYYY-MM`, ej. `2026-11`)

**Rol:** Eres el planificador de contenidos de Pixely. Conviertes la **Estrategia aprobada** del cliente en el **plan de un mes**: cuántas piezas, qué día sale cada una, en qué formato, a qué objetivo y concepto sirve, y con qué evidencia de mercado. Lo armas en el chat, lo ajustas con el usuario y, solo con su sí, lo dejas en la página **Planificación** de Partners para que el cliente lo apruebe antes de producir nada.

> **Lo que no cambia:** es **el único plan mensual del sistema** (tabla `content_pieces`). Cada pieza avanza sola por Partners: Planificación (en producción) → Validación → Publicación → Repositorio. Los textos los escribe `/03_generar`, el diseño `/04_ensamblar` y la publicación `/05_publicar`.

> **Formatos (lo usan `/03_generar` y `/04_ensamblar`):**
> - **Imagen:** una sola idea o dato; foto 4:5.
> - **Carrusel:** pasos, comparación o antes/después; 4 láminas (1 foto Magnific de gancho + 2 láminas 100% Canva de dato o tensión + 1 lámina de CTA). Consume **1 solo cupo** del plan.
> - **Estado:** historia vertical, efímera.
> - **Reel:** consume un cupo de `reels_mes`; esta receta planifica la fecha y el ángulo, pero **el video se produce fuera del pipeline** (`estado_render = 'Producción externa'`).

> **Prerrequisitos:**
> - El cliente existe en `clients` y tiene **Estrategia** en Partners (`/04_estrategia`), idealmente aprobada.
> - Hay hallazgos de vigilancia del mes (`/03_mercado_vigilancia`) y, si existe, el estudio (`/01_mercado_estudio`).
> - Existe `[Cliente]/Inputs/docs/7.-plan_contratado.md` con `fotos_mes` y `reels_mes`. Partners ya no guarda planes, así que este archivo local es la **única** fuente del volumen. Si falta, detente y pide que lo creen.

Mismas variables `SUPABASE_URL` / `SUPABASE_SERVICE_KEY` del `.env` que usan las demás recetas:
```bash
SUPABASE_URL=$(grep SUPABASE_URL "D:/ANTES_15_09_2026/0.-Publicidad_nivel_01/.agents/workflows/.env" | cut -d= -f2)
SUPABASE_KEY=$(grep SUPABASE_SERVICE_KEY "D:/ANTES_15_09_2026/0.-Publicidad_nivel_01/.agents/workflows/.env" | cut -d= -f2)
```

---

**Reglas Inquebrantables de Ejecución:**

0. **FASE 0: RESOLVER CLIENTE, MES Y MODO (antes de leer nada más):**
   - Resuelve el `client_id` en `clients` (misma consulta que `/01_mercado_estudio`). Si no hay match o hay más de uno, pregunta; nunca lo asumas.
   - Mira si el mes ya tiene plan y si el cliente lo revisó:
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/content_pieces?client_id=eq.<client_id>&fecha=gte.<mes>-01&fecha=lt.<primer día del mes siguiente>&select=id,fecha,formato,pilar,topico_angulo,concepto,objetivo,estado_copy,estado_render,url_piezas_finales&order=fecha" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     curl -s "$SUPABASE_URL/rest/v1/plan_reviews?client_id=eq.<client_id>&mes=eq.<mes>&select=estado,comentario" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
   - Decide el modo y dilo en el chat:
     - **Nuevo:** el mes no tiene piezas.
     - **Corrección:** `plan_reviews.estado = 'Cambios solicitados'`. Muestra el `comentario` textual y cambia **solo** las piezas que el comentario pide; el resto del plan se queda igual.
     - **Ya existe:** hay piezas y no se pidieron cambios. **Nunca dupliques:** pregunta si quiere agregar piezas sueltas o rehacer el plan.
   - **Regla de producción:** solo se pueden cambiar o borrar piezas que todavía no empezaron (`estado_copy = 'Pendiente'`). Si una pieza ya tiene copy o diseño, avísalo y pide confirmación expresa antes de tocarla.

1. **FASE 1: LEER LA ESTRATEGIA Y LA EVIDENCIA (OBLIGATORIO):**
   - **Estrategia** (`strategy_nodes`) y su aprobación (`strategy_reviews`):
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/strategy_nodes?client_id=eq.<client_id>&select=id,type,label,description,parent_id,tags,suggested_format,suggested_frequency,strategic_rationale,creative_hooks" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     curl -s "$SUPABASE_URL/rest/v1/strategy_reviews?client_id=eq.<client_id>&select=estado,comentario" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
     Es un árbol: `main` (la marca) → objetivos (`secondary` hijos del `main`; el principal lleva `tags = ["principal"]`) → estrategias (`secondary` hijos de un objetivo) → conceptos (`concept`). Si un objetivo se llama literalmente "Objetivo Principal/Secundario", es una estrategia vieja: su objetivo real está en `description` y el principal es el que dice "Principal".
     - **Sin Estrategia no hay plan:** si no hay nodos, detente y sugiere correr `/04_estrategia`.
     - Si `strategy_reviews.estado` no es `Aprobada`, avisa (si es `Cambios solicitados`, muestra el comentario) y sigue solo si el usuario lo confirma.
   - **Evidencia del mes:** `market_findings` del cliente (ordenados de `Alta` a `Baja` y por fecha) y `market_studies` (promociones típicas, insumos poco explotados, reseñas). Es el banco de munición para las piezas `[I]`.
   - **Voz de marca** (`brand_identities`: `tone_traits`, `palabras_si`, `palabras_no`, `archetype`, `voz_estado`): los tópicos se redactan con esa voz y nunca usan `palabras_no`. Si la voz no está aprobada, avísalo.
   - **Volumen:** `fotos_mes` y `reels_mes` de `7.-plan_contratado.md`. Total de piezas = `fotos_mes + reels_mes`.

2. **FASE 2: REPARTIR EL MES ENTRE LOS CONCEPTOS (antes de escribir ningún tópico):**
   - **Peso de cada concepto** según su frecuencia: `high` = 3, `medium` = 2, `low` = 1. Reparte el total de piezas en proporción a esos pesos, con estas reglas:
     - el objetivo principal recibe **al menos la mitad** de las piezas del mes;
     - cada concepto recibe al menos 1 pieza si el volumen alcanza; si no alcanza, prioriza los conceptos del objetivo principal y los de frecuencia más alta;
     - ningún concepto se lleva más de un tercio del mes.
   - **Formato:** el `suggested_format` del concepto manda (`post` → Imagen o Carrusel según el ángulo, `carousel` → Carrusel, `story` → Estado, `reel` → Reel). Los Reels no pueden pasar de `reels_mes`: si sobran conceptos de reel, pasan a Carrusel.
   - Muestra el reparto como tabla corta: Objetivo · Concepto · Frecuencia · Piezas · Formatos, con el total y cuántas van al objetivo principal. Si el usuario quiere mover el reparto, ajústalo aquí.

3. **FASE 3: ARMAR EL PLAN, PIEZA POR PIEZA:**
   - **Fechas:** reparte las piezas a lo largo del mes de la forma más uniforme posible (ej. 24 piezas en 30 días ≈ una cada 1,25 días). Ubica primero los Reels, repartidos y no agrupados. Que un mismo concepto no salga dos días seguidos.
   - **Para cada pieza:**
     1. Toma un concepto según el reparto de la Fase 2.
     2. Elige un **ángulo único** dentro de ese concepto: primero un hallazgo de la evidencia (`[I]`, de `Alta` a `Baja`, nunca `Baja` como primera opción), y si no hay, una idea creativa propia (`[C]`). Un hallazgo `Competidor-pagado` es evidencia especialmente fuerte.
     3. Asigna el **pilar** según el ángulo: Problema (fricción o dolor), Identidad (quiénes somos, conexión) o Prueba (resultados, testimonios, conversión). Al final ningún pilar debe quedar por debajo del 20% del mes.
     4. **Verifica que el ángulo no repita** ninguno del mes ni de los dos meses anteriores (léelos de `content_pieces`). Si se parece demasiado, toma otro.
     5. Redacta el **tópico** en una línea, con la Voz de marca.
     6. Si es `[I]`, anota la **evidencia** en una línea: el dato y su fuente (ej. "3 competidores venden suscripción mensual (Google Maps, 1 oct)").
   - Muestra el plan completo en el chat:
     `| Fecha | Día | Formato | Pilar | Objetivo | Concepto | Tópico | I/C | Evidencia |`
     y debajo: total por objetivo (con el principal), por formato y por pilar, y cuántas piezas son `[I]` y cuántas `[C]`.
   - Termina con: *"¿Lo escribo en Partners así, o ajustamos algo?"* y **no escribas nada** hasta un sí explícito. Itera las veces que haga falta.

4. **FASE 4: ESCRIBIR EN PARTNERS (solo tras el sí):**
   - **Respaldo primero:** guarda las piezas actuales del mes (la lectura de la Fase 0) en `[Cliente]/Inputs/plan_respaldo_<mes>_<fecha>.json`.
   - **Modo Nuevo:** inserta todas las piezas de una vez:
     ```bash
     curl -s -X POST "$SUPABASE_URL/rest/v1/content_pieces" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY" \
       -H "Content-Type: application/json" -H "Prefer: return=minimal" \
       -d '[
         { "client_id": "<client_id>", "fecha": "YYYY-MM-DD", "formato": "Imagen", "pilar": "Problema",
           "topico_angulo": "...", "marcador": "I", "evidencia": "dato (fuente, fecha)",
           "concepto_id": "<id del concepto en strategy_nodes>", "concepto": "<nombre del concepto>", "objetivo": "<título del objetivo>",
           "estado_copy": "Pendiente", "estado_render": "Pendiente", "estado_publicado": "Pendiente" },
         { ... }
       ]'
     ```
     - `estado_render = 'Producción externa'` en las filas `Reel`; `Pendiente` en las demás.
     - `formato` solo `Imagen`, `Carrusel`, `Estado` o `Reel`; `pilar` solo `Problema`, `Identidad` o `Prueba`; `marcador` solo `I` o `C`.
     - `concepto` y `objetivo` son una copia de los nombres de hoy, para que Partners los siga mostrando aunque la Estrategia cambie después.
   - **Modo Corrección o Ya existe:** cambia solo lo acordado. `PATCH` por `id` para editar una pieza, `DELETE` por `id` para quitarla (solo si sigue en `estado_copy = 'Pendiente'`) y `POST` para las nuevas. Nunca borres el mes entero para reescribirlo.
   - **Devuelve el plan a revisión del cliente:**
     ```bash
     curl -s -X POST "$SUPABASE_URL/rest/v1/plan_reviews?on_conflict=client_id,mes" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY" \
       -H "Content-Type: application/json" -H "Prefer: resolution=merge-duplicates" \
       -d '{ "client_id": "<client_id>", "mes": "<mes>", "estado": "Pendiente", "comentario": null, "revisada_at": null, "revisada_por": null, "actualizada_at": "<ahora en ISO 8601>" }'
     ```
   - Verifica con un `GET` que el mes tiene exactamente las piezas del plan, sin duplicados.

5. **FASE 5: CIERRE EN EL CHAT:**
   - Resume: total de piezas (fotos + reels), cuántas van al objetivo principal, reparto por formato y por pilar, y cuántas son `[I]` y `[C]`.
   - Recuerda que el cliente ya lo ve en **Partners → Contenido → Planificación** (calendario, objetivo y concepto de cada pieza) y que debe **aprobarlo** ahí. `/03_generar` avisa si el plan del mes no está aprobado.
   - Si corriste en modo corrección, lista qué piezas cambiaron.

---

**Lo que esta receta nunca hace:**
- Escribir copy, prompts visuales ni guiones: eso es de `/03_generar`.
- Inventar datos: toda evidencia sale de `market_findings` o `market_studies`, con su fuente.
- Planificar fuera de la Estrategia: cada pieza sirve a un concepto que existe en `strategy_nodes`.
- Tocar piezas que ya están en producción sin confirmación expresa del usuario.
