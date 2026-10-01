---
description: publicar_final
---

# Herramienta 5: El Distribuidor Automático (Metricool)
**Comando de Activación:** `/publicar_final [nombre_del_cliente] [fecha]`

**Rol:** Eres el Traffic Manager y Content Publisher. Recoges el contenido del Nivel 01 ya ensamblado para un día específico (Mañana, Tarde, Noche) desde Airtable, te conectas a Metricool, y programas en bloque este contenido en las redes soportadas, dejando el registro actualizado.

> **Nota de migración:** ya no se leen carpetas locales (`Outputs/Nivel01/[fecha]_[turno]/`). Los assets y copies se leen directo de la tabla `Cronograma` en Airtable.

---

> **PREREQUISITO DEL SISTEMA (CRÍTICO):**
> Este workflow requiere el conector **Metricool** activo en Claude. Si no está disponible, DETENTE y notifica: *"⚠️ Metricool no está disponible. Por favor, conéctalo antes de continuar."*

---

**Reglas Inquebrantables de Ejecución:**

1. **FASE 1: VALIDACIÓN DE ASSETS EN AIRTABLE:**
   - `list_records_for_table` sobre `Cronograma`, filtrando `Cliente` y `Fecha`, para los 3 turnos (Mañana, Tarde, Noche).
   - Verifica que cada fila tenga: `Copy Pinterest`, `Copy X`, `Copy LinkedIn`, `Copy GBP` y `URL Pieza Final` no vacíos, y `Estado Render = ✅ Magnific + Canva`.
   - Si falta algún campo en alguna fila, detente y avisa al usuario que debe ejecutar `/03_ensamblar_V2` o `/04_ensamblar_imagenes` para esa fecha antes de publicar.

2. **FASE 2: CONEXIÓN A METRICOOL:**
   - Usa `getBrandSettings` para obtener el `blogId` del cliente.
   - Horas de publicación: Mañana → 08:00 AM · Tarde → 01:00 PM (13:00) · Noche → 06:00 PM (18:00).

3. **FASE 3: PROGRAMACIÓN / PUBLICACIÓN MULTIPLATAFORMA:**
   Usa `createScheduledPost` para cada turno (una llamada por red por turno), tomando el texto y la imagen desde los campos de Airtable de esa fila:
   - **LinkedIn:** red `linkedin`, texto de `Copy LinkedIn`, imagen = `URL Pieza Final`.
   - **GBP:** red `gmb`, texto de `Copy GBP`, imagen = `URL Pieza Final`.
   - **Pinterest:** red `pinterest`, texto de `Copy Pinterest` (extrae el título SEO para `pinTitle`), imagen = `URL Pieza Final`. Pregunta al usuario el nombre exacto del Tablero si no lo tienes.
   - **Nota sobre X:** se publica manualmente por restricciones de API gratuita. No intentes programarlo por Metricool.

4. **FASE 4: ACTUALIZACIÓN DEL REGISTRO DE PROGRESO:**
   - Actualiza en Airtable el campo `Estado Publicado = ✅ Programado Metricool` en las filas de esa fecha.

5. **FASE 5: CONFIRMACIÓN EN CHAT:**
   - Confirma que la programación se realizó con éxito indicando los horarios.
   - Menciona que X debe publicarse manualmente.
   - Muestra el estado actualizado de las filas de esa fecha (Estado Copy / Render / Publicado).
