# ADR-0007: React con TypeScript como Framework de Frontend

## Contexto

El sistema PADI requiere una interfaz de usuario web que:

- Presente información diferenciada según el rol del usuario (cuatro roles con dashboards distintos).
- Gestione flujos complejos como el proceso de evaluación pedagógica multi-paso.
- Visualice datos estadísticos mediante gráficos interactivos.
- Navegue entre múltiples secciones (evaluaciones, estudiantes, docentes, escuelas, estadísticas) sin recargas de página completa.
- Sea mantenible a lo largo del ciclo de vida del proyecto de tesis, incluyendo la posibilidad de ser extendida por futuros desarrolladores.

El frontend debía ser una **Single Page Application (SPA)** para garantizar una experiencia fluida sin recargas, dado que los formularios de evaluación son extensos e interactivos.

## Decisión

Se adoptó **React 19** con **TypeScript** como framework y lenguaje principal del frontend. La herramienta de construcción seleccionada fue **Vite 6**.

La combinación elegida incluye:
- **React 19** para la composición de la interfaz mediante componentes.
- **TypeScript ~5.7** en modo estricto para garantizar seguridad de tipos en tiempo de compilación.
- **React Router DOM v7** para el enrutamiento del lado del cliente (client-side routing).
- **Material-UI (MUI) v6** como biblioteca de componentes UI, que provee un sistema de diseño coherente y accesible.
- **Vite 6** como bundler y servidor de desarrollo por su velocidad superior respecto a Create React App o Webpack.

## Alternativas Consideradas

- **Vue.js 3 con TypeScript:** Framework moderno con una curva de aprendizaje más amigable que React. Fue descartado porque el equipo de desarrollo tenía mayor experiencia y familiaridad con React, lo que reducía el riesgo de demoras por aprendizaje en el contexto del proyecto de tesis.

- **Angular:** Framework de Google con TypeScript incorporado por defecto, arquitectura opinada y herramientas integradas (CLI, testing). Fue descartado por su complejidad de configuración inicial y la verbosidad de su arquitectura (módulos, decoradores, inyección de dependencias), que resultan excesivos para los requisitos del proyecto.

- **Svelte / SvelteKit:** Framework compilado que produce código muy eficiente. Fue descartado por su ecosistema más limitado en comparación con React (especialmente en cuanto a bibliotecas de componentes UI compatibles) y por la menor familiaridad del equipo con el paradigma de compilación.

- **Next.js (como framework principal):** Framework React con renderizado del lado del servidor (SSR). Fue evaluado y existe evidencia de su consideración en el directorio `app/` del proyecto. Fue descartado como framework principal porque el sistema no requiere SEO (es una aplicación privada con autenticación requerida) ni renderizado del servidor, y Next.js agrega complejidad para estos requisitos.

- **JavaScript sin TypeScript:** Eliminaría la necesidad de compilación y reduciría la configuración inicial. Fue descartado porque el backend también adoptó TypeScript, y la consistencia entre ambas capas es valiosa para compartir tipos de interfaces de API y reducir errores de contrato.

## Consecuencias

### Positivas
- **Ecosistema maduro:** React cuenta con una amplia comunidad, abundante documentación y una gran cantidad de bibliotecas complementarias compatibles.
- **Seguridad de tipos end-to-end:** El uso de TypeScript en ambas capas (frontend y backend) permite detectar inconsistencias de tipos en la interfaz de la API durante el desarrollo, incluso sin herramientas específicas de validación de contratos.
- **Componentización:** La arquitectura de componentes de React facilita la reutilización de elementos de UI (formularios, tablas, gráficos) entre distintas secciones de la aplicación.
- **Herramientas de desarrollo:** Vite ofrece Hot Module Replacement (HMR) casi instantáneo, reduciendo el ciclo de feedback durante el desarrollo. Las React DevTools permiten inspeccionar el estado de la aplicación en tiempo real.
- **Storybook integrado:** La elección de React facilita la integración con Storybook para el desarrollo y documentación aislada de componentes.

### Negativas / Compromisos
- **Complejidad de configuración:** Comparado con frameworks como SvelteKit o Next.js que proveen una estructura más opinionada, React con Vite requiere configurar manualmente el enrutamiento, la gestión de estado y otras preocupaciones transversales.
- **Tamaño del bundle:** React 19 y sus dependencias (MUI, React Router, React Query, Recharts) generan un bundle significativo. Aunque Vite aplica code splitting, el tamaño inicial puede ser mayor que el de alternativas compiladas como Svelte.
- **Curva de aprendizaje de hooks:** Los patrones modernos de React (hooks, context, concurrent features) tienen una curva de aprendizaje no trivial para desarrolladores nuevos en el framework.
