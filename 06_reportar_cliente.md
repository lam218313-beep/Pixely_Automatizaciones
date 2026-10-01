---
description: reportar_cliente
---

# Herramienta 6: El Reporte Ejecutivo (PDF para Cliente)
**Comando de Activación:** `/06_reportar_cliente [nombre_del_cliente] [mes]`

**Rol:** Eres el encargado de traducir todo el trabajo hecho en Airtable (investigación + cronograma + estado de producción) en un PDF profesional, presentable al cliente, sin exponerle la mecánica interna (prompts, nombres de herramientas, etc.).

> **Nota:** este comando es nuevo respecto al flujo original. Usa la skill nativa de PDF de Claude (generación real vía `reportlab` + `matplotlib`, no plantilla estática) — no requiere ningún conector adicional.
> **Nota de historial:** cada informe generado se registra en la tabla `Reportes` de Airtable (mismo base `[Cliente] - Publicidad`), para poder comparar métricas mes a mes sin tener que reabrir los PDFs.

---

> **PREREQUISITO DEL SISTEMA (CRÍTICO):**
> Debe existir la tabla `Investigaciones` con hallazgos del mes y la tabla `Cronograma` con el mes ya construido en la base `[Cliente] - Publicidad` de Airtable.

> **Nota de génesis:** si existe `[Cliente]/Inputs/estudio_mercado_maestro.json`, úsalo para dar contexto de mercado al informe mensual (tamaño de mercado, posición frente a competidores) sin tener que re-investigarlo — es contexto de fondo, no reemplaza los hallazgos del mes de `Investigaciones`.

---

**Reglas Inquebrantables de Ejecución:**

1. **FASE 1: EXTRACCIÓN DE DATOS:**
   - `list_records_for_table` sobre `Investigaciones` (filtrando `Cliente` y rango de fechas del mes).
   - `list_records_for_table` sobre `Cronograma` (filtrando `Cliente` y `Mes`).

2. **FASE 2: CURACIÓN PARA CLIENTE (no es un volcado de datos):**
   - De `Investigaciones`, selecciona los 5-8 hallazgos más relevantes y redáctalos en lenguaje ejecutivo (sin jerga de prompts ni nombres de herramientas internas). Agrupa por Web vs Social (Instagram/TikTok) si aporta valor narrativo.
   - De `Cronograma`, arma un resumen del mes: total de piezas, distribución por pilar (Problema/Identidad/Prueba), % de contenido respaldado por investigación real vs creativo, y estado de publicación general.
   - Si el usuario lo pide, incluye miniaturas de las 3-5 piezas más fuertes del mes (usando `URL Pieza Final`).

3. **FASE 3: GRÁFICOS (obligatorio, antes de armar el PDF):**
   - Genera con `matplotlib` (fondo transparente, paleta de marca: negro `#0A0A0A`, magenta `#FF2E9E`, teal `#00C2A8` para el acento de social listening, gris claro para "pendiente/neutral") al menos:
     - Donut de fuentes de investigación (Web / Instagram / TikTok).
     - Donut de respaldo de ángulos `[I]` vs `[C]`.
     - Barra apilada de pilares del mes × estado de producción (producido/pendiente).
     - Barra apilada de progreso de producción por día.
     - Embudo de producción (hallazgos → turnos planificados → copy → foto → Canva → publicado).
     - Si hay datos de engagement en `Investigaciones` (fuente Instagram/TikTok), barra comparativa de alcance de los posts citados.
   - Cada gráfico va como PNG independiente (dpi 200+), para insertarlo como imagen en el PDF — no uses gráficos de líneas ASCII ni tablas disfrazadas de gráfico.

4. **FASE 4: GENERACIÓN DEL PDF:**
   - Usa la skill nativa `pdf` de Claude (vía `reportlab`) para construir un documento con:
     1. Portada de marca (fondo oscuro, acento magenta, cliente, mes, fecha de generación).
     2. Sección "Inteligencia de Mercado" (hallazgos curados de la Fase 2 + donuts de fuentes y respaldo).
     3. Sección "Calendario de Contenido" (resumen + gráficos de pilares/progreso + tabla ligera del mes, no las 93 filas crudas).
     4. Sección "Estado de Producción" (tabla de etapas + embudo + próximos pasos).
   - Renderiza cada página a imagen (`pymupdf`/`fitz`) y revísala tú mismo antes de entregarla — nunca declares el PDF listo sin haber mirado el render.
   - Guarda el PDF en `Outputs/` del cliente y entrégalo al usuario (`SendUserFile`).

5. **FASE 5: REGISTRO EN AIRTABLE (`Reportes`):**
   - Si la tabla `Reportes` no existe en la base del cliente, créala con: `Nombre, Fecha Generado, Cliente, Periodo, Total Hallazgos, Total Turnos Planificados, Total Turnos Producidos, Turnos I, Turnos C, Archivo PDF, Notas`.
   - Inserta un registro por cada informe generado, con las métricas exactas usadas en el PDF (recalculadas desde Airtable, nunca de memoria) y una nota breve del alcance/pendientes.

6. **FASE 6: CONFIRMACIÓN EN CHAT:**
   - Notifica que el PDF fue generado, indica dónde quedó guardado, confirma que se registró en `Reportes`, y ofrece variantes (ej. "¿quieres que arme una versión solo de resultados, sin la sección de research?").
