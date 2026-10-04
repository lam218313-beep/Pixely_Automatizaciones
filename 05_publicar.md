---
description: publicar_final
---

# Herramienta 5: El Distribuidor Automático (Metricool)
**Comando de Activación:** `/05_publicar [nombre_del_cliente] [mes o fecha]`

**Rol:** Eres el Traffic Manager y Content Publisher. Tomas las piezas que **el cliente ya aprobó en Partners**, las programas en Metricool en las redes que corresponden a cada formato y dejas el registro actualizado.

> **Nota de fusión con Partners (Supabase):** la cola sale de `content_pieces` en Supabase (ya no de Airtable ni de carpetas locales), con las mismas variables que `/03_generar` (`SUPABASE_URL` y la service key en `$SUPABASE_KEY`). **La aprobación es del cliente, en Partners (Validación) — este proceso nunca la decide ni la salta:** solo publica filas con `estado_aprobacion = Aprobado`. Al programar, la pieza pasa en Partners de "Aprobada, por programar" a **"Programada"** (Publicación) y, el día siguiente a su fecha, al archivo del **Repositorio**.
> **Ya no hay turnos Mañana/Tarde/Noche:** cada pieza tiene una sola `fecha`; la hora se decide en la Fase 2.

---

> **PREREQUISITO DEL SISTEMA (CRÍTICO):**
> Este workflow requiere el conector **Metricool** activo en Claude. Si no está disponible, DETENTE y notifica: *"⚠️ Metricool no está disponible. Por favor, conéctalo antes de continuar."*
> **Ejemplo:** `/05_publicar Pixely Octubre` (todo lo aprobado y aún sin programar de ese mes) o `/05_publicar Pixely 2026-10-14` (solo ese día).

---

**Reglas Inquebrantables de Ejecución:**

1. **FASE 1: COLA DE PIEZAS APROBADAS (Supabase):**
   ```bash
   curl -s "$SUPABASE_URL/rest/v1/content_pieces?client_id=eq.<client_id>&fecha=gte.<desde>&fecha=lt.<hasta>&estado_aprobacion=eq.Aprobado&estado_publicado=eq.Pendiente&order=fecha.asc&select=*" \
     -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY"
   ```
   (`<desde>`/`<hasta>`: el mes completo, o un solo día con `fecha=eq.YYYY-MM-DD`).
   - Verifica en cada fila: `url_piezas_finales` con al menos una URL `https://` que cargue, y el copy de cada red a la que va (ver Fase 3). Si algo falta, sáltala y repórtala — **nunca** publiques una pieza incompleta ni la "completes" aquí.
   - Descarta y reporta las filas con `fecha` ya pasada: aprobadas tarde, el usuario decide si se reprograman a otra fecha (si dice que sí, cambia su `fecha` en Supabase con un `PATCH` antes de seguir).
   - Cuenta también las piezas **aún no aprobadas** del mismo periodo (`estado_aprobacion` distinto de `Aprobado`) y menciónalas: siguen en Validación esperando al cliente; no se tocan.
   - Muestra la tabla `| Fecha | Formato | Tópico | Redes | Hora propuesta |` y **espera confirmación** antes de programar nada.

2. **FASE 2: CONEXIÓN A METRICOOL Y HORA DE PUBLICACIÓN:**
   - `getBrandSettings` para obtener el `blogId` y la zona horaria del cliente.
   - Hora: usa `getBestTimeToPostByNetwork` (Instagram/LinkedIn) para la semana de cada pieza y toma la mejor franja de ese día; si no hay datos, usa 13:00 hora del cliente. Una sola hora por pieza para todas sus redes, salvo que el usuario pida otra cosa.

3. **FASE 3: PROGRAMACIÓN MULTIPLATAFORMA (`createScheduledPost`, programación directa):**
   Usa siempre `createScheduledPost` con `autoPublish: true`. **Nunca uses `createScheduledPostForReview` ni `sendScheduledPostForReview`:** la revisión ya la hizo el cliente en Partners, y pasarla por el flujo de revisión de Metricool le haría aprobar dos veces. Una llamada por pieza (todas sus redes en `providers`) o una por red, como resulte más claro. `media` = las URLs de `url_piezas_finales` en orden.
   - **Instagram** (`instagram`, texto de `copy_instagram`):
     - `Imagen` → `instagramData.type = POST`, 1 imagen.
     - `Carrusel` → `POST` con todas las láminas de `url_piezas_finales` en `media`, en orden.
     - `Estado` → `STORY` (sin texto si Instagram es la única red de esa llamada).
     - `Reel` → `REEL`, el `.mp4` en `media`. Es la única red de un Reel.
     - Marca `isAiGenerated` con el valor de `generada_con_ia` de la fila (lo indica el equipo al subir la pieza final en Partners). Si viene vacío, pregunta antes de programar.
   - **LinkedIn** (`linkedin`, texto de `copy_linkedin`, primera imagen de `url_piezas_finales`): Imagen y Carrusel. Si `copy_linkedin` está vacío, no va a LinkedIn.
   - **Pinterest** (`pinterest`, texto de `copy_pinterest`, `pinTitle` = su título SEO, primera imagen): Imagen y Carrusel. Pregunta al usuario el nombre exacto del tablero si no lo tienes.
   - **Google Business** (`gmb`, `gmbData.type = publication`, texto de `copy_gbp`, máx. 1500 caracteres): Imagen y Carrusel.
   - **X:** se publica manualmente (restricciones de la API gratuita). No lo programes por Metricool: deja el `copy_x` listo en el chat para copiar y pegar.
   - `Estado` solo va a Instagram (historia); `Reel` solo a Instagram.

4. **FASE 4: ACTUALIZACIÓN EN SUPABASE (solo de las piezas que Metricool aceptó):**
   ```bash
   curl -s -X PATCH "$SUPABASE_URL/rest/v1/content_pieces?id=in.(<id1>,<id2>,...)" \
     -H "apikey: $SUPABASE_KEY" -H "Authorization: Bearer $SUPABASE_KEY" \
     -H "Content-Type: application/json" -H "Prefer: return=minimal" \
     -d '{ "estado_publicado": "✅ Programado Metricool" }'
   ```
   Si una red falló para una pieza y las demás no, márcala igual como programada pero repórtalo en el chat con la red que falló.

5. **FASE 5: CONFIRMACIÓN EN CHAT:**
   - Confirma qué se programó, en qué redes y a qué hora (hora del cliente).
   - Lista los textos de X para publicar a mano, con su fecha.
   - Menciona las piezas saltadas (incompletas, con fecha pasada) y cuántas siguen esperando aprobación del cliente en Partners.
