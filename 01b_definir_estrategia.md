---
description: definir_estrategia
---

# Herramienta de Estrategia: El Arquitecto
**Comando de Activación:** `/01b_definir_estrategia [nombre_del_cliente]`

**Rol:** Eres el estratega de cuenta de Pixely. Con todo lo que ya se sabe del cliente (su Ficha, el estudio de mercado, la vigilancia de la competencia y su Voz de marca) defines **qué debe lograr el negocio, cómo y con qué tipo de contenido**, y lo dejas escrito en la página **Estrategia** de Partners para que el cliente la lea y la apruebe. Nada se escribe sin el visto bueno del usuario en el chat: esta herramienta propone, el usuario decide.

> **Por qué existe:** antes la Estrategia la generaba una IA dentro del backend de Partners, sin supervisión y sin ver el mercado real. Se eliminó: **esta receta es la única forma de llenar la página Estrategia**. Va entre `/01_escanearmercado` (de donde saca la evidencia) y `/02_crearcronograma_V2` (que la usa como brújula de cada mes).

> **Cuándo correrla:**
> - Por primera vez, después de `/00_genesis_cliente` y de una primera tanda de `/01_escanearmercado`, con la Ficha del cliente ya llena en Partners.
> - Cuando el cliente pide cambios desde Partners (`strategy_reviews.estado = 'Cambios solicitados'`): la receta entra en modo corrección (Fase 0).
> - Cuando la Ficha cambia: Partners le avisa al cliente que su Estrategia quedó desactualizada.
> - Como revisión trimestral, o antes si la vigilancia trae un hallazgo `Alta` que cambie las prioridades.

Mismas variables `SUPABASE_URL` / `SUPABASE_SERVICE_KEY` del `.env` que usan `/00_genesis_cliente` y `/01_escanearmercado`:
```bash
SUPABASE_URL=$(grep SUPABASE_URL "D:/ANTES_15_09_2026/0.-Publicidad_nivel_01/.agents/workflows/.env" | cut -d= -f2)
SUPABASE_KEY=$(grep SUPABASE_SERVICE_KEY "D:/ANTES_15_09_2026/0.-Publicidad_nivel_01/.agents/workflows/.env" | cut -d= -f2)
```

---

**Reglas Inquebrantables de Ejecución:**

0. **FASE 0: RESOLVER CLIENTE Y MODO:**
   - Resuelve el `client_id` en `clients` (misma consulta que `/00_genesis_cliente`). Si no hay match o hay más de uno, pregunta; nunca lo asumas.
   - Lee la estrategia actual y su aprobación:
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/strategy_nodes?client_id=eq.<client_id>&select=*&order=created_at" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     curl -s "$SUPABASE_URL/rest/v1/strategy_reviews?client_id=eq.<client_id>&select=*" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
   - Decide el modo y dilo en el chat:
     - **Nueva:** no hay nodos.
     - **Corrección:** `estado = 'Cambios solicitados'`. Muestra el `comentario` del cliente textual y trabaja **sobre la estrategia actual**: cambia solo lo que el comentario pide y explica cada cambio.
     - **Actualización:** hay nodos y el usuario quiere revisarla (Ficha nueva, hallazgos nuevos). Muestra qué cambió en los datos desde `strategy_reviews.actualizada_at`.
   - Si `estado = 'Aprobada'` y el usuario no pidió cambios explícitamente, adviértelo: reescribirla la vuelve a poner en revisión del cliente.

1. **FASE 1: LEER TODA LA EVIDENCIA (OBLIGATORIO, EN PARALELO):**
   - **Ficha del negocio** (`client_interviews.data`): qué vende, a qué precio, a quién, objetivos del dueño, retos, diferenciadores, competidores que nombra.
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/client_interviews?client_id=eq.<client_id>&select=data" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
     **Sin Ficha no hay estrategia:** si no existe, detente y pide que el cliente la llene en Partners.
   - **Estudio de mercado** (`market_studies`, de `/00_genesis_cliente`): `tamano_mercado.rango_estimado`, `universo_competidores.listado` (rating y reseñas), `dossier_profundo[].estadisticas_precio`, `panorama_producto_precio.promociones_tipicas_detectadas`, y las `notas_metodologicas.limitaciones_honestas`.
   - **Vigilancia** (`market_findings`, de `/01_escanearmercado`): ordénalos por `confianza` (Alta → Media → Baja) y luego por `fecha` (más recientes primero).
   - **Voz de marca** (`brand_identities`: `archetype`, `tone_traits`, `palabras_si`, `palabras_no`, `voz_estado`): los ganchos y textos de esta receta se escriben con esa voz y nunca usan `palabras_no`. Si no hay voz, o `voz_estado` no es `Aprobada`, avisa: lo ideal es correr antes `/00b_definir_voz` y que el cliente la apruebe. Sigue solo si el usuario lo confirma.
   - **Volumen contratado:** `[Cliente]/Inputs/docs/7.-plan_contratado.md` (`fotos_mes`, `reels_mes`), la misma fuente que usa `/02_crearcronograma_V2`.
   - Si no hay estudio de mercado **ni** hallazgos, avísalo: una estrategia sin mercado es una hipótesis. Sigue solo si el usuario lo confirma, y dilo en el porqué de cada objetivo.

2. **FASE 2: DIAGNÓSTICO EN EL CHAT (antes de proponer nada):**
   - De 5 a 8 líneas de "lo que dicen los datos", cada una con su fuente entre corchetes: `[Ficha]`, `[00: dossier de X]`, `[01: hallazgo Alta, Google Maps, 1 oct]`.
   - Marca las tensiones: lo que el dueño quiere vs lo que el mercado muestra (ej.: quiere vender premium pero el ticket promedio del mercado es S/ 16 y 3 competidores ya usan 2x1).
   - Un hallazgo `Baja` es una hipótesis: puede inspirar un concepto, **nunca** justificar por sí solo el objetivo principal.

3. **FASE 3: PROPUESTA (en el chat, como esquema), Y ESPERAR CONFIRMACIÓN:**
   Arma el árbol con esta forma y muéstralo completo:
   - **Objetivos (2 a 4):** uno con prioridad `principal`, el resto `secundario`.
     - Título concreto en palabras del dueño del negocio, máximo 70 caracteres. Bien: "Llenar el local de lunes a jueves". Mal: "Objetivo Principal" o "Aumentar el engagement".
     - Porqué: 1 a 3 frases que citan el dato que lo respalda.
   - **Estrategias (2 a 3 por objetivo):** el *cómo*.
     - Título accionable, sin el prefijo "Estrategia:".
     - Porqué: 1 a 2 frases, con el ángulo diferenciador frente a la competencia.
   - **Conceptos de contenido (2 a 4 por estrategia):** tipos de publicación que se repiten, **no** posts sueltos.
     - `label` de 2 a 5 palabras y `description` de 1 a 3 frases que pinten la pieza.
     - `suggested_format`: solo `post` (Imagen), `carousel` (Carrusel), `story` (Estado) o `reel`, los formatos que produce el pipeline.
     - `suggested_frequency`: `high` (3–4 por semana), `medium` (1–2 por semana) o `low` (1–2 al mes).
     - `strategic_rationale`: por qué este concepto mueve su objetivo.
     - `creative_hooks`: de 3 a 6 ganchos directamente usables, en la Voz de marca.
     - `execution_guidelines`: `structure` (paso a paso), `key_elements`, `dos` y `donts`, de 2 a 4 cada uno.
     - `tags`: de 1 a 3 etiquetas temáticas.
   - **Chequeo de volumen:** suma lo que piden las frecuencias en un mes (high ≈ 14, medium ≈ 6, low ≈ 2 piezas) y compáralo con `fotos_mes + reels_mes`. Si se pasa por más de un 50%, baja frecuencias o recorta conceptos; si se queda muy corto, súbelas. Muestra la cuenta.
   - **Reparto:** el objetivo principal debe recibir al menos la mitad de las piezas del mes.
   - Termina con: *"¿La escribo en Partners así, o ajustamos algo?"* y **no escribas nada** hasta un sí explícito. Itera las veces que haga falta.

4. **FASE 4: ESCRIBIR EN PARTNERS (solo tras el sí):**
   - **Respaldo primero:** guarda la estrategia actual (la lectura de la Fase 0) en `[Cliente]/Inputs/estrategia_respaldo_<fecha>.json`. Es lo que restauras si algo falla.
   - **Ids legibles y únicos:** `<client_id>-root`, `<client_id>-o1`, `<client_id>-o1-e2`, `<client_id>-o1-e2-c3` (objetivo 1, estrategia 2, concepto 3). No envíes `x`/`y`: valen 0 por defecto y Partners acomoda el árbol solo.
   - Borra la estrategia anterior y escribe la nueva:
     ```bash
     curl -s -X DELETE "$SUPABASE_URL/rest/v1/strategy_nodes?client_id=eq.<client_id>" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"

     curl -s -X POST "$SUPABASE_URL/rest/v1/strategy_nodes" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY" \
       -H "Content-Type: application/json" -H "Prefer: return=minimal" \
       -d '[
         { "id": "<client_id>-root", "client_id": "<client_id>", "type": "main", "parent_id": null,
           "label": "<nombre de la marca>", "description": "" },
         { "id": "<client_id>-o1", "client_id": "<client_id>", "type": "secondary", "parent_id": "<client_id>-root",
           "label": "Llenar el local de lunes a jueves", "description": "<porqué con su dato>", "tags": ["principal"] },
         { "id": "<client_id>-o1-e1", "client_id": "<client_id>", "type": "secondary", "parent_id": "<client_id>-o1",
           "label": "Convertir a los clientes de fin de semana en habituales", "description": "<porqué>" },
         { "id": "<client_id>-o1-e1-c1", "client_id": "<client_id>", "type": "concept", "parent_id": "<client_id>-o1-e1",
           "label": "Ritual del cliente fijo", "description": "...", "suggested_format": "reel", "suggested_frequency": "medium",
           "tags": ["prueba-social"], "strategic_rationale": "...", "creative_hooks": ["...", "..."],
           "execution_guidelines": { "structure": "...", "key_elements": ["..."], "dos": ["..."], "donts": ["..."] } },
         { ... }
       ]'
     ```
     El orden importa: cada padre va antes que sus hijos dentro del arreglo. Si el `POST` falla, **restaura el respaldo de inmediato** con otro `POST` y avisa al usuario: el cliente nunca debe quedarse sin estrategia.
   - Devuélvela a revisión del cliente:
     ```bash
     curl -s -X POST "$SUPABASE_URL/rest/v1/strategy_reviews?on_conflict=client_id" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY" \
       -H "Content-Type: application/json" -H "Prefer: resolution=merge-duplicates" \
       -d '{ "client_id": "<client_id>", "estado": "Pendiente", "comentario": null, "revisada_at": null, "revisada_por": null, "actualizada_at": "<ahora en ISO 8601>" }'
     ```
   - Verifica con un `GET` que la cantidad de nodos escritos es la propuesta.

5. **FASE 5: CIERRE EN EL CHAT:**
   - Resume: N objetivos, N estrategias, N conceptos, piezas por mes que piden sus frecuencias vs el plan contratado.
   - Recuerda que el cliente ya la ve en **Partners → Tu marca → Estrategia**, donde la aprueba o pide cambios. `/02_crearcronograma_V2` avisa si arma un mes sobre una estrategia no aprobada.
   - Si corriste en modo corrección, lista qué cambió respecto de la versión anterior.

---

**Lo que esta receta nunca hace:**
- Escribir en `content_pieces`: eso es de `/02_crearcronograma_V2`.
- Inventar cifras: todo número del porqué sale de la Ficha, de `market_studies` o de `market_findings`.
- Escribir posts concretos con fecha: los conceptos son territorios que el plan de cada mes convierte en piezas.
- Tocar la Voz de marca, la Ficha o el estudio de mercado: solo los lee.
