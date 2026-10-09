# Pixely: contexto para Claude Code

**Este repo:** las recetas que producen el trabajo de cada cliente. Reglas comunes en `CONVENCIONES.md`; cada receta se usa como comando (`/01_mercado_estudio`, etc.).

> Este archivo es igual en los 4 repos de Pixely (pixely, Pixely_web, pixely_marca y pixely_automatizaciones), salvo la línea "Este repo". Si cambias algo, cámbialo en los cuatro. Última actualización: 9 oct 2026.

## Con quién trabajas
- Quien dirige Pixely es PM y no programa. Responde siempre en español, en palabras simples y con analogías cotidianas. Si usas un término técnico, explícalo.
- Muestra el resultado (PDF, captura, enlace) y pide aprobación antes de pasar a lo siguiente. Si algo está listo y verificado, dilo claro; si falló, dilo con la evidencia.
- Al avanzar una tarea, actualiza la matriz (ver abajo).

## Reglas fijas
- Solo los nombres "Pixely" y "Pixely Partners". Nunca "Radar": se dice "Mercado" o "estudio de mercado".
- En textos para clientes no se dice "IA". En piezas de captación no van precios.
- Nunca prometer ventas. Se usa: "Los resultados pueden variar según el negocio, el sector y la constancia en la publicación".
- Las llaves y contraseñas no se comparten por chat, no se escriben en archivos y no se suben a los repos (tampoco la llave de firma de las apps). Si alguien pega una, no la repitas y recomienda cambiarla.
- Confirma antes de borrar o de hacer algo difícil de deshacer (base de datos, archivos, filas de la matriz), salvo que te lo pidan de forma explícita.
- Código o habilidades de terceros: solo con autorización explícita.
- No empezar producción de videos ni el diseño del modo simple de Partners (F1-12) sin luz verde.

## Mapa del proyecto
| Repo (GitHub lam218313-beep) | Qué es | Dónde se publica |
|---|---|---|
| pixely | Pixely Partners: backend FastAPI (`backend_v2`), app del cliente (`frontend/app`, React 19 + Tailwind 4; Android e iOS con Capacitor) y escritorio de clientes y equipo (`frontend/layout`). Documentos en `docs/` | Backend en Railway; frontends en Vercel (proyecto "frontend") → partners.pixely.pe |
| Pixely_web | pixely.pe | Vercel (proyecto "pixely-web") |
| pixely_marca | Marca: tokens, logos, catálogo de componentes (`componentes/`), PDF del kit (`pdf/`), videos y habilidades de Claude Code (`.claude/skills`, lista en `SKILLS.md`) | El catálogo es un artifact privado: https://claude.ai/artifact/GKQZcrKRtnJyPxoKEWQPJE |
| pixely_automatizaciones | Recetas (estudio de mercado, voz de marca, generar, ensamblar, estrategia, planificación, publicar) como comandos `/01_mercado_estudio`, etc. en `.claude/commands`; reglas comunes en `CONVENCIONES.md` | Se corren en la computadora del equipo |

- Base de datos: Supabase, proyecto `zvpisdftltnukbozyuge` (plan gratis). Respaldo semanal automático en GitHub Actions → "Respaldo de la base" (repo pixely, `respaldo-base.yml`). Respaldos y cambio de llaves: `pixely/docs/respaldos-y-llaves.md`.
- Computadora del equipo: todo vive en `D:\1.-Pixely` (Recetas con su `.env`, Clientes, 3.-Pixely_web, 4.-Pixely_data). `CARPETA_CLIENTES=D:/1.-Pixely/Clientes`.
- Cuenta de equipo (admin) en Partners: lucia.ramos@pixely.pe (correo ficticio de @pixely.pe). Para más cuentas: Supabase → Authentication → Add user, y luego marcarlas como equipo.
- Contacto de marca: WhatsApp +51 949 268 607 · hola@pixely.pe · @pixely_pe · pixely.pe. Razón social: Syntesia Labs E.I.R.L., RUC 20616010787.

## La matriz (plan de lanzamiento)
- Google Sheet "Pixely - Plan de lanzamiento", id `133jcpfOmlQTzljDxH7S0Sz69sJzFnrdXafCZubNaT6E`, pestaña Tareas (sheetId 1462445922).
- Columnas: ID, Frente, Tarea, Prioridad, Estado, Responsable, Fecha límite, Entregable, Depende de, Notas. Estados: Por hacer, En curso, En revisión, Hecho, Bloqueado.
- Tiene un filtro básico para ver un frente a la vez. Toda tarea nueva debe quedar dentro del rango del filtro, y no cambies el frente que dejó filtrado la persona.
- Al avanzar algo: cambia el Estado y escribe en Notas la fecha y qué se hizo, en lenguaje simple. Lo terminado pero sin aprobar va en "En revisión" y pasa a "Hecho" cuando lo aprueban.
- La persona edita la matriz a mano: léela antes de escribir y verifica la fila exacta antes de cambiar o borrar.

## Estado al 8 oct 2026
- Partners funciona en producción con la interfaz nueva (orbes de carga, foco de un borde, mazo ligero, entrada animada) y el panel del equipo con alta de cliente guiada.
- Kit de lanzamiento listo y aprobado (rehecho el 8 oct): brochure, 3 planes, manual de Partners, tarjeta para imprenta, QR y 2 videos de prueba. pixely.pe muestra capturas actuales de la app.
- Descartado (no volver a proponer): guía rápida de una página; plantillas de propuesta, contrato y firma de correo; caso de éxito; repositorios de animaciones y de imágenes; manual de marca (queda solo el catálogo); el nombre Radar; la receta 06; más videos.
- Líneas de venta: vendedores por comisión, venta por data (mensajes y llamadas) y venta directa con anuncios en redes. Todas usan el kit. El proceso de venta está en Drive: "Pixely - Proceso de venta" (carpeta 8. Estrategia y marketing). Lo que hay que habilitar para cada línea todavía no se ha contado: no lo supongas.
- Pendiente:
  - F1-9: cambiar las llaves antes del primer cliente real. Lo hace la persona siguiendo `docs/respaldos-y-llaves.md`; las llaves no pasan por el chat.
  - Subir a Drive los PDF nuevos. El conector de Drive no sube archivos grandes, así que la persona los arrastra a las carpetas del kit (1. Tarjetas y QR, 2. Brochure y presentaciones, 3. Planes, 4. Manuales).
  - Completar las variables de Supabase en el `.env` de Recetas, en la computadora del equipo.
  - En la matriz: F4-1 (oferta de publicidad pagada), F2-4 y F2-5 (cuentas publicitarias y medición en pixely.pe), la línea de base de datos (F3) y F2-8. F1-4 y F1-5 están bloqueadas.

## Cómo se hacen las cosas
- Ramas: en pixely y Pixely_web se trabaja en una rama, se fusiona a main con `git merge --no-ff` (mensaje "Fusionar <rama>") y se sube main; Vercel publica solo. pixely_marca y pixely_automatizaciones van directo a main.
- PDF del kit (pixely_marca): cada pieza es `pdf/<pieza>/index.html` sobre la base común `pdf/comun/base.css`. `npm run pdf` (o `npm run pdf brochure`) los genera en `public/piezas/pdf` y avisa si algo se sale de la hoja. Antes de entregar, revisa el PDF como imagen. Estilo: páginas negras y blancas, rosa #EB0C6E solo de acento, Unbounded en títulos y Manrope en texto.
- Capturas de la app (para la web y el manual): en `pixely/frontend/app`, `VITRINA=1 npx playwright test vitrina --project=android` (marca de ejemplo Casa Norte); luego, en Pixely_web, `npm run vitrina`.
- Elementos gráficos (pixely_marca: `elementos/elementos.json`, página «Elementos gráficos» del catálogo): los documentos solo usan elementos aprobados, con sus reglas. Primero se terminan los elementos; después cada documento se arma página por página, con aprobación de cada página.
- Catálogo de componentes: `node componentes/scripts/build.mjs` en pixely_marca genera `componentes/artifact/index.html`, que se vuelve a publicar en el artifact.
- Pruebas: app de Partners con `npm run test:e2e` (en `frontend/app`); web con `npm test` y `npx playwright test`. En la nube, Chrome está en `/opt/pw-browsers` (ver el `playwright.config` de cada repo).
