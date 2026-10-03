# Pixely Automatizaciones

Procesos de Claude Desktop (slash-commands) que investigan mercado y planifican contenido a mano, con mucha supervisión humana — **nunca automatización ciega**, porque el mercado peruano no es confiable solo con datos scrapeados.

## Procesos activos (escriben en Supabase, influyen en Partners)

| Archivo | Qué hace | Tablas de Supabase |
|---|---|---|
| `00_genesis_cliente.md` | Estudio de mercado fundacional (una vez por cliente) | escribe `market_studies` |
| `01_escanearmercado.md` | Vigilancia competitiva recurrente | lee `brand_identities`/`client_interviews`/`market_studies`, escribe `market_findings` |
| `02_crearcronograma_V2.md` | Cronograma de contenido del mes | lee `brand_identities`/`client_interviews`/`market_findings`/`market_studies`, escribe `content_pieces` |

## Procesos sin terminar (no tocan Supabase todavía)

`03_generar.md`, `04_ensamblar.md`, `05_publicar.md`, `06_reportar_cliente.md` — siguen apuntando al flujo viejo (local/Airtable). No los uses asumiendo que están conectados a Partners.

---

## ⚠️ Este repo depende del esquema real de Partners — revisar tras cada cambio ahí

Los 3 procesos activos asumen nombres exactos de tablas, columnas, restricciones y variables de entorno que viven en **otro repositorio** (`lam218313-beep/Pixely`, el backend/frontend de Partners). Nada sincroniza esto automáticamente — si algo cambia en Partners y nadie revisa estos `.md`, quedan desactualizados en silencio (ya pasó una vez: ver Changelog abajo).

**Cada vez que se modifique el esquema de Supabase o se quite/renombre algo en Partners, volver a verificar en los 3 archivos:**

- [ ] Nombres de tabla y columna usados en los `curl` (`clients`, `market_studies`, `market_findings`, `content_pieces`, `brand_identities`, `client_interviews`) — ¿siguen existiendo exactamente así?
- [ ] Valores permitidos por restricciones `CHECK` (`cluster`, `confianza` en `market_findings`; `formato`, `pilar`, `marcador` en `content_pieces`) — ¿la lista de valores sigue siendo la misma?
- [ ] Restricciones `UNIQUE` que algún `curl` asuma para `on_conflict` (ej. `market_studies.client_id`) — ¿siguen ahí si se recrea la tabla?
- [ ] Cualquier mención a una columna/feature específica de Partners (ej. el extinto `clients.plan`) — si se elimina algo en Partners, buscar su nombre en estos 3 archivos y corregir la nota.
- [ ] Nombres de las variables de entorno en Railway (`SUPABASE_URL`, `SUPABASE_SERVICE_KEY`) — si alguna vez se renombran ahí, actualizar la nota de cada archivo que las menciona.

Forma rápida de auditar: `grep -rn "clients\.\|market_studies\.\|market_findings\.\|content_pieces\." *.md` y comparar contra el esquema real en Supabase (`information_schema.columns`).

### Changelog de desincronizaciones encontradas y corregidas

- **2026-10-03** — `00_genesis_cliente.md` decía que la `SUPABASE_SERVICE_KEY` "está en Railway como `SUPABASE_KEY`" (es la llave anon, un valor distinto); `02_crearcronograma_V2.md` decía que "Partners ya guarda el plan contratado en `clients.plan`" (esa columna fue eliminada por completo); y el `POST` a `market_studies` usaba `Prefer: resolution=merge-duplicates` sin que la tabla tuviera una restricción `UNIQUE (client_id)` ni el `curl` incluyera `on_conflict=client_id`, así que cada re-ejecución duplicaba la fila en vez de actualizarla. Se corrigieron los 3 y se agregó la restricción `UNIQUE` en Supabase.
