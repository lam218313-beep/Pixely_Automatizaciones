# Convenciones de todas las recetas

Valen para cada receta de esta carpeta, en cualquier computadora del equipo.

## Llaves

- Viven **solo** en el archivo `.env` de esta carpeta (la de las recetas). Cada persona tiene el suyo; la plantilla es `.env.example`.
- Se leen en el momento de usarlas, en la misma llamada de shell (las variables no persisten entre llamadas):
  ```bash
  SUPABASE_URL=$(grep -m1 '^SUPABASE_URL=' .env | cut -d= -f2- | tr -d '\r')
  SUPABASE_KEY=$(grep -m1 '^SUPABASE_SERVICE_KEY=' .env | cut -d= -f2- | tr -d '\r')
  TOKEN=$(grep -m1 '^APIFY_API_TOKEN=' .env | cut -d= -f2- | tr -d '\r')
  ```
  (`tr -d '\r'` quita el salto de línea de Windows que deja el Bloc de notas.)
- **Nunca** imprimas un valor, lo muestres en el chat ni pidas que lo peguen en el chat. Si falta una variable, di cuál falta y pide agregarla al `.env` siguiendo `.env.example`.

## Carpeta de cada cliente

- Donde una receta dice `[Cliente]/...` o `[nombre_del_cliente]/...`, la ruta real es **`$CARPETA_CLIENTES/[Cliente]/...`**, con `CARPETA_CLIENTES` leída del `.env` igual que las llaves.
- Si `CARPETA_CLIENTES` no está, pregunta dónde guarda el equipo las carpetas de los clientes y pide agregar la línea al `.env`.
- **Nunca** crees carpetas de clientes dentro de esta carpeta de recetas: es un repositorio y se publicaría.
