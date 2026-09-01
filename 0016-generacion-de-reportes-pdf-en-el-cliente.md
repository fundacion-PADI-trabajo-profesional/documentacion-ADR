---
title: "ADR-0016: Generación de reportes PDF en el cliente con @react-pdf/renderer"
nav_order: 16
---

# ADR-0016: Generación de reportes PDF en el cliente con @react-pdf/renderer

## Contexto

La fundación necesita entregar a cada escuela un **reporte en PDF** con los resultados de la Prueba
PADI (portada, resumen, una sección por sala, en modos inicial / cierre / inicial vs cierre). Es la
primera funcionalidad del sistema que produce un documento paginado con identidad visual propia.

El proyecto no tenía infraestructura de PDF. El backend corre como Firebase Cloud Function v2, con
límites de memoria y cold start que penalizan procesos pesados. Ya existía un precedente de export
en el cliente: el Excel de evaluaciones se genera en el navegador con ExcelJS cargado por import
dinámico (ADR implícito en `feat/export-excel-evaluaciones`).

## Decisión

Generar el PDF **en el navegador** con `@react-pdf/renderer` (v4.9, compatible con React 19):

1. Un endpoint nuevo (`GET /reportes/escuela`) devuelve el reporte **ya calculado** por una función
   pura del backend (testeada con fixtures); el frontend solo lo renderiza.
2. El documento se describe con componentes React propios en `src/pdf/reporteEscuela/`
   (páginas, cuadrículas, nóminas, pautas), con las decisiones de layout que no dependen de
   react-pdf en funciones puras de `src/utils/reporteEscuela.ts` (cubiertas por el umbral de
   cobertura del frontend).
3. La librería y el documento solo se cargan con `import()` dinámico desde la página
   `/reportes/escuela`, igual que ExcelJS: no entran al bundle inicial.
4. Las tipografías (League Spartan y Montserrat, licencia OFL) se embeben como TTF estáticos en
   `src/pdf/fonts/` y se registran con `Font.register`.
5. Un **smoke test** renderiza el documento completo a buffer en Node (`renderToBuffer`) con
   fixtures de 24, 44 y 62 chicos y verifica la cantidad de páginas, que es parte de la spec
   (paginación "fluye").

## Alternativas Consideradas

1. **CSS de impresión + diálogo del navegador (`window.print()`)** — cero dependencias y fue
   suficiente para la maqueta, pero el resultado depende del navegador (márgenes, fondos
   desactivados por defecto, saltos de página caprichosos) y el usuario debe pasar por el diálogo
   de impresión: no es "un botón que descarga un PDF".
2. **Chromium/Puppeteer en la Cloud Function** — fidelidad perfecta con el mismo HTML, pero suma
   cientos de MB a la función, cold starts de varios segundos, más memoria y timeouts; además
   introduce en el servidor un proceso pesado para un documento que el cliente puede armar solo.
3. **html2canvas + jsPDF** — captura rasterizada: texto borroso, no seleccionable, sin control real
   de paginación.

## Consecuencias

### Positivas

- PDF determinístico (mismas fuentes y layout en cualquier máquina), con texto seleccionable.
- Nada nuevo en el servidor: sin dependencias pesadas, sin costo extra, sin datos persistidos.
- La lógica de negocio queda en una función pura del backend y la de layout en utils puros del
  frontend: ambas testeadas de forma barata.
- El patrón de import dinámico ya usado para Excel se repite: el bundle inicial no crece.

### Negativas / Compromisos

- ~1 MB extra (comprimido) en un chunk lazy del frontend, más ~500 KB de fuentes embebidas.
- El documento se describe con primitivas de react-pdf: no se reutilizan MUI ni recharts, y el
  estilo del PDF evoluciona por separado del de la app.
- La generación corre en el hilo principal del navegador (1–3 s para una escuela grande); si en la
  práctica supera los ~3 s se moverá a un Web Worker (react-pdf lo soporta) — no antes.
- `aprobacionPorPregunta` (stat existente) calcula por sub-pregunta sin respetar
  `puntaje_invertido`; el reporte usa la regla correcta compartida en `src/utils/pautas.ts`. El bug
  latente de la stat queda anotado y fuera del alcance de este cambio.
