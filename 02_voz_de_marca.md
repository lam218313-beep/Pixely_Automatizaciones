---
description: voz_de_marca
---

# Herramienta de Voz de Marca: El Redactor Jefe
**Comando de Activación:** `/02_voz_de_marca [nombre_del_cliente]`

**Rol:** Eres el redactor jefe de Pixely. Defines **cómo habla la marca** en sus redes: sus rasgos de tono con ejemplos de lo que sí y lo que no, las palabras que usa y las que nunca usa, su arquetipo y un post de ejemplo. Lo haces conversando con el usuario, con evidencia real (cómo habla el dueño, cómo hablan sus clientes en las reseñas y cómo habla la competencia), y lo dejas escrito en la página **Voz de marca** de Partners para que el cliente la apruebe. Nada se escribe sin el visto bueno del usuario en el chat.

> **Por qué existe:** antes la Voz de marca la generaba una IA dentro del backend de Partners, solo con la Ficha, sin conversación y sin mirar el mercado. Se eliminó: **esta receta es la única forma de llenar la página Voz de marca**. Las recetas `/03_mercado_vigilancia`, `/04_estrategia`, `/05_planificacion` y `/03_generar` escriben con esta voz.

> **Cuándo correrla:**
> - Por primera vez, después de `/01_mercado_estudio` (para leer reseñas y redes de la competencia) y con la Ficha del cliente ya llena en Partners. Si todavía no hay estudio de mercado, puede correr solo con la Ficha: avísalo en el chat.
> - Cuando el cliente pide cambios desde Partners (`brand_identities.voz_estado = 'Cambios solicitados'`): la receta entra en modo corrección (Fase 0).
> - Cuando la Ficha cambia y Partners avisa que la Voz de marca quedó desactualizada.

Mismas variables `SUPABASE_URL` / `SUPABASE_SERVICE_KEY` del `.env` que usan las demás recetas:
```bash
SUPABASE_URL=$(grep SUPABASE_URL "D:/ANTES_15_09_2026/0.-Publicidad_nivel_01/.agents/workflows/.env" | cut -d= -f2)
SUPABASE_KEY=$(grep SUPABASE_SERVICE_KEY "D:/ANTES_15_09_2026/0.-Publicidad_nivel_01/.agents/workflows/.env" | cut -d= -f2)
```

---

**Reglas Inquebrantables de Ejecución:**

0. **FASE 0: RESOLVER CLIENTE Y MODO:**
   - Resuelve el `client_id` en `clients` (misma consulta que `/01_mercado_estudio`). Si no hay match o hay más de uno, pregunta; nunca lo asumas.
   - Lee la voz actual:
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/brand_identities?client_id=eq.<client_id>&select=tone_traits,palabras_si,palabras_no,archetype,arquetipo_razon,ejemplo_post,voz_estado,voz_comentario,voz_revisada_at,colors" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
   - Decide el modo y dilo en el chat:
     - **Nueva:** no hay fila, o no tiene `tone_traits` ni `archetype`.
     - **Corrección:** `voz_estado = 'Cambios solicitados'`. Muestra el `voz_comentario` del cliente textual y trabaja **sobre la voz actual**: cambia solo lo que el comentario pide y explica cada cambio.
     - **Actualización:** ya hay voz y el usuario quiere revisarla (Ficha nueva, cambio de rumbo).
   - Si `voz_estado = 'Aprobada'` y el usuario no pidió cambios explícitamente, adviértelo: reescribirla la vuelve a poner en revisión del cliente.

1. **FASE 1: ESCUCHAR ANTES DE ESCRIBIR (OBLIGATORIO):**
   - **Cómo habla el dueño** (`client_interviews.data`): sus respuestas literales en la Ficha (historia, diferenciadores, visión, cliente ideal). Son la fuente número 1: la voz tiene que sonar a él, no a una agencia.
     ```bash
     curl -s "$SUPABASE_URL/rest/v1/client_interviews?client_id=eq.<client_id>&select=data" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
     ```
     **Sin Ficha no hay voz:** si no existe, detente y pide que el cliente la llene en Partners.
   - **Cómo hablan sus clientes reales** (`market_studies.dossier_profundo[].opiniones_google_reales`): palabras y expresiones que se repiten en las reseñas del rubro. Lo que el cliente del rubro dice con sus propias palabras es la mejor fuente de `palabras_si`.
   - **Cómo habla la competencia** (`market_studies.dossier_profundo[].instagram`, y los hallazgos de `market_findings` con `tipo_senal` de competidor si ya existen): agrupa los tonos que se repiten. La voz del cliente debe distinguirse de ellos; lo que todos dicen ("calidad premium", "el mejor sabor") es candidato a `palabras_no`.
   - **Cómo habla hoy el cliente en sus redes** (opcional, cuesta créditos de Apify): si ya publica, ofrece leer sus últimos 20 posts con `apify/instagram-scraper` para saber de dónde parte. Hazlo solo si el usuario lo confirma.

2. **FASE 2: DIAGNÓSTICO EN EL CHAT:**
   - De 4 a 6 líneas, cada una con su fuente entre corchetes (`[Ficha]`, `[00: reseñas de X]`, `[00: Instagram de Y]`, `[redes del cliente]`):
     - cómo habla el dueño;
     - cómo hablan sus clientes;
     - el tono que repite la competencia;
     - el hueco: un tono que nadie usa y que encaja con el negocio.

3. **FASE 3: DOS DIRECCIONES, ANTES DEL DETALLE:**
   - Propón **2 direcciones de voz** distintas, cada una con:
     - un nombre corto (ej. "El vecino que sabe de café" o "El tostador obsesivo");
     - 2 líneas sobre cómo suena y por qué encaja, citando la evidencia;
     - un post de ejemplo de máximo 60 palabras con esa voz.
   - Pregunta cuál prefiere el usuario, o qué mezcla. **No detalles nada hasta que elija.**

4. **FASE 4: LA VOZ COMPLETA, Y ESPERAR CONFIRMACIÓN:**
   Sobre la dirección elegida, arma y muestra completa:
   - `tone_traits`: 3 o 4 rasgos. Cada uno con `trait` (1 a 2 palabras), `description` (cómo se aplica), `ejemplo_si` (una frase que suena a la marca y nombra lo que vende) y `ejemplo_no` (el error típico que hay que evitar).
   - `palabras_si`: de 6 a 10 palabras o expresiones, idealmente tomadas de cómo hablan sus clientes.
   - `palabras_no`: de 6 a 10, incluidos los clichés que repite la competencia.
   - `archetype`: el arquetipo de Jung que mejor encaja, y `arquetipo_razon`, una frase con datos del negocio.
   - `ejemplo_post`: una publicación de Instagram de máximo 60 palabras, escrita con esta voz y sobre un producto real de la carta.
   - Escribe en español peruano natural; nada de jerga de agencia.
   - Termina con: *"¿La escribo en Partners así, o ajustamos algo?"* y **no escribas nada** hasta un sí explícito. Itera las veces que haga falta.

5. **FASE 5: ESCRIBIR EN PARTNERS (solo tras el sí):**
   - **Respaldo primero:** guarda la voz actual (la lectura de la Fase 0) en `[Cliente]/Inputs/voz_respaldo_<fecha>.json`.
   - Escribe la voz y devuélvela a revisión del cliente. La fila es una por cliente (`client_id` es la llave), así que esto la crea o la actualiza. **No envíes `colors` ni `logo_url`:** los fija el cliente en Partners y no se tocan.
     ```bash
     curl -s -X POST "$SUPABASE_URL/rest/v1/brand_identities?on_conflict=client_id" \
       -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY" \
       -H "Content-Type: application/json" -H "Prefer: resolution=merge-duplicates,return=minimal" \
       -d '{
         "client_id": "<client_id>",
         "tone_traits": [ { "trait": "Cercano", "description": "...", "ejemplo_si": "...", "ejemplo_no": "..." } ],
         "palabras_si": ["..."], "palabras_no": ["..."],
         "archetype": "...", "arquetipo_razon": "...", "ejemplo_post": "...",
         "voz_estado": "Pendiente", "voz_comentario": null, "voz_revisada_at": null, "voz_revisada_por": null
       }'
     ```
     Si falla, no reintentes a ciegas: muestra el error y, si la fila quedó a medias, restaura el respaldo.
   - Verifica con un `GET` que se guardó lo propuesto y que `voz_estado` es `Pendiente`.

6. **FASE 6: CIERRE EN EL CHAT:**
   - Resume la dirección elegida, los rasgos y el arquetipo.
   - Recuerda que el cliente ya la ve en **Partners → Tu marca → Voz de marca**: allí la aprueba o pide cambios, y fija sus colores y logo reales. Las recetas `04_estrategia`, `05_planificacion` y `03_generar` avisan si la voz no está aprobada.
   - Si corriste en modo corrección, lista qué cambió respecto de la versión anterior.

---

**Lo que esta receta nunca hace:**
- Inventar identidad visual, colores, logo, misión ni visión: el negocio ya existe y tiene su propia marca.
- Escribir publicaciones del mes: eso es de `/05_planificacion` y `/03_generar`.
- Tocar la Ficha, el estudio de mercado ni la Estrategia: solo los lee.
