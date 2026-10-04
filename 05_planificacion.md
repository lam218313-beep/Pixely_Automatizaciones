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
> - **Reel:** consume un cupo de `reels_mes`; esta receta planifica la fecha y el ángulo, pero **el video se produce fuera del pipeline** (su guía la prepara `/04_ensamblar` y el equipo lo edita en CapCut).

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
   - Mira si el mes ya tiene plan y qué decidió el cliente sobre cada idea (el cliente aprueba **pieza por pieza** en Planificación):
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/content_pieces?client_id=eq.<client_id>&fecha=gte.<mes>-01&fecha=lt.<primer día del mes siguiente>&select=id,fecha,formato,pilar,topico_angulo,concepto,objetivo,razon,plan_estado,plan_comentario,estado_copy,estado_render,url_piezas_finales&order=fecha" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
   - Decide el modo y dilo en el chat:
     - **Nuevo:** el mes no tiene piezas.
     - **Corrección:** hay piezas con `plan_estado = 'Cambios solicitados'`. Lista cada una con su `plan_comentario` textual y propón el ajuste **solo de esas piezas** (otro tópico, ángulo, fecha, formato o concepto, según lo que pidió); las demás se quedan igual. Las `Aprobada` no se tocan.
     - **Ya existe:** hay piezas y ninguna tiene cambios pedidos (pueden estar `Pendiente` o `Aprobada`). **Nunca dupliques:** pregunta si quiere agregar piezas sueltas o rehacer el plan.
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
     1. Toma un concepto según el reparto de la Fase 2. Ese es su **concepto principal**. Si el ángulo sirve de verdad a otro concepto (del mismo objetivo o de otro), puede **combinarlo** con uno más, como máximo; la pieza sigue contando para el reparto solo en su concepto principal.
     2. Elige un **ángulo único** dentro de ese concepto: primero un hallazgo de la evidencia (`[I]`, de `Alta` a `Baja`, nunca `Baja` como primera opción), y si no hay, una idea creativa propia (`[C]`). Un hallazgo `Competidor-pagado` es evidencia especialmente fuerte.
     3. Asigna el **pilar** según el ángulo: Problema (fricción o dolor), Identidad (quiénes somos, conexión) o Prueba (resultados, testimonios, conversión). Al final ningún pilar debe quedar por debajo del 20% del mes.
     4. **Verifica que el ángulo no repita** ninguno del mes ni de los dos meses anteriores (léelos de `content_pieces`). Si se parece demasiado, toma otro.
     5. Redacta el **tópico** en una línea, con la Voz de marca.
     6. Si es `[I]`, anota la **evidencia** en una línea: el dato y su fuente (ej. "3 competidores venden suscripción mensual (Google Maps, 1 oct)").
     7. Escribe la **razón** en 1 o 2 frases para el cliente: por qué existe la pieza y cómo se juntaron objetivo, estrategia, concepto(s) y evidencia para llegar a ese tópico (ej. "Tostado Co. paga anuncios de su suscripción; respondemos haciendo la cuenta visible y recordando que cada bolsa sale tostada esa semana"). Sin jerga ni nombres internos.
   - Muestra el plan completo en el chat:
     `| Fecha | Día | Formato | Pilar | Objetivo | Estrategia | Concepto(s) | Tópico | I/C | Evidencia | Razón |`
     (si combina dos conceptos, el principal va primero) y debajo: total por objetivo (con el principal), por estrategia, por formato y por pilar, y cuántas piezas son `[I]`, `[C]` y cuántas combinan conceptos.
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
           "concepto_id": "<id del concepto principal>", "concepto_ids": ["<id del concepto principal>", "<id del segundo, si combina>"],
           "concepto": "<nombre del concepto principal>", "objetivo": "<título del objetivo del concepto principal>",
           "razon": "<por qué existe y cómo se combinó la estrategia, 1-2 frases>",
           "estado_copy": "Pendiente", "estado_render": "Pendiente", "estado_publicado": "Pendiente" },
         { ... }
       ]'
     ```
     - `estado_render = 'Pendiente'` en todas las filas (también los Reels: `/04_ensamblar` prepara su guía).
     - `formato` solo `Imagen`, `Carrusel`, `Estado` o `Reel`; `pilar` solo `Problema`, `Identidad` o `Prueba`; `marcador` solo `I` o `C`.
     - `concepto_ids` siempre lleva al menos el concepto principal y en primer lugar (igual a `concepto_id`). Partners lee de ahí el objetivo, la estrategia y los conceptos de cada pieza para sus gráficos y su detalle.
     - `concepto` y `objetivo` son una copia de los nombres de hoy, para que Partners los siga mostrando aunque la Estrategia cambie después.
     - `descripcion_visual` (qué muestra la imagen) no se escribe aquí: la escribe `/03_generar` cuando define la imagen.
   - **Modo Corrección o Ya existe:** cambia solo lo acordado. `PATCH` por `id` para editar una pieza, `DELETE` por `id` para quitarla (solo si sigue en `estado_copy = 'Pendiente'`) y `POST` para las nuevas. Nunca borres el mes entero para reescribirlo.
   - **Devuelve a revisión del cliente solo lo que cambiaste:** cada pieza corregida (y cada pieza nueva) queda en `plan_estado = 'Pendiente'`. En el mismo `PATCH` de la corrección incluye `"plan_estado": "Pendiente"` y deja `plan_comentario` como está, para que el cliente vea a qué respondiste. Las piezas nuevas nacen en `Pendiente` solas (valor por defecto). Nunca escribas `Aprobada`: eso solo lo hace el cliente desde Partners.
   - Verifica con un `GET` que el mes tiene exactamente las piezas del plan, sin duplicados.

5. **FASE 5: CIERRE EN EL CHAT:**
   - Resume: total de piezas (fotos + reels), cuántas van al objetivo principal, reparto por formato y por pilar, y cuántas son `[I]` y `[C]`.
   - Recuerda que el cliente ya lo ve en **Partners → Contenido → Planificación** (gráficos del mes por objetivo, estrategia, concepto, formato y pilar; calendario; y al abrir cada idea, de dónde sale en la estrategia y la razón, sin imágenes ni copy) y que debe **aprobar cada idea** ahí (o todas las pendientes de una vez). `/03_generar` solo produce las ideas aprobadas.
   - Si corriste en modo corrección, lista qué piezas cambiaron.

---

**Lo que esta receta nunca hace:**
- Escribir copy, prompts visuales ni guiones: eso es de `/03_generar`.
- Inventar datos: toda evidencia sale de `market_findings` o `market_studies`, con su fuente.
- Planificar fuera de la Estrategia: cada pieza sirve a un concepto que existe en `strategy_nodes`.
- Tocar piezas que ya están en producción sin confirmación expresa del usuario.
