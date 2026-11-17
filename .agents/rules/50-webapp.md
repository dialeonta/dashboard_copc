# Glob: apps/webapp/**
# Regla: 50-webapp

# Reglas Tácticas de Webapp (Bloque 3 — Vanilla TS + HTML5 + CSS Nativo)

> Se activa únicamente al operar en `apps/webapp/`.
> Complementa a `10-tech-stack.md`, `20-coding-standards.md` y `30-sdd-process.md`.

## 1. Guía Web Moderna (Consulta Obligatoria)
* Antes de implementar componentes interactivos, modales, popovers, formularios o layouts complejos, consulta el estándar Baseline ejecutando:
  `pnpm exec modern-web-guidance search "<patrón o componente>"`
* Prioriza APIs nativas (`<dialog>`, Popover API, Constraint Validation API, CSS Subgrid, `:has()`) sobre librerías o lógica imperativa innecesaria, siempre dentro del objetivo de navegadores de `10-tech-stack.md` §2.3. Una API fuera de Baseline *widely available* (por ejemplo `field-sizing`) requiere justificación en el `plan.md`.
* Si la herramienta no está disponible o falla por infraestructura, **DETENTE** y avisa. No continúes sin la consulta ni la ignores.

## 2. Clientes HTTP y Comunicación con la API (`src/api/`)
* **Cliente centralizado y validación:** Toda comunicación HTTP pasa por `src/api/client.ts`. Las respuestas JSON se validan en tiempo de ejecución con los esquemas Zod de `packages/shared-contracts` antes de entregarlas a la vista o al estado.
* **Sesión segura:** Las cookies de sesión las gestiona el navegador con `credentials: 'same-origin'`. Prohibido leer, guardar o manipular tokens en `localStorage`, `sessionStorage` o `src/state.ts`.
* **Renovación de sesión:** Ante un `401`, `client.ts` ejecuta un único refresh en vuelo y reintenta la petición una sola vez.
* **Traducción de errores:** La UI no muestra el `message` del servidor. Los errores se mapean por `code` a mensajes en español en `src/api/error-messages.ts`.
* **Cambio de contrato tardío:** Si la UI necesita un campo ausente en `shared-contracts`, **DETENTE** (condición "Cambio de contrato tardío" de `30-sdd-process.md`). Prohibido extender tipos localmente.
* **Aserciones:** Prohibido `as` sobre datos de red. Para el DOM, estrecha con `instanceof` o genéricos tipados (`querySelector<HTMLInputElement>`).

## 3. Construcción de DOM, Estado y Estilos (`src/pages/`, `src/components/`, `src/styles/`)
* **DOM seguro contra XSS:** Las páginas y componentes son funciones tipadas que retornan un `HTMLElement` construido con el helper `h()` de `src/lib/dom.ts`. En `h(tag, props, ...children)` los hijos `string` se insertan como nodos de texto, los handlers son funciones (nunca strings) y los `href`/`src` dinámicos se validan.
* **Prohibido** (además de lo del 10 §5.5): `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`, `eval`, `new Function`, el atributo `style` y `setAttribute('style', ...)`. Para estilos dinámicos usa clases o `el.style.setProperty(...)`.
* **Recursos autoalojados:** Fuentes, librerías y assets se sirven desde el propio origen; nada de CDNs ni Google Fonts (incumple la CSP). Las librerías se instalan con `pnpm` y se empaquetan en el build. Las fuentes son `.woff2` locales; Vite no debe incrustarlas como `data:` (`assetsInlineLimit: 0`).
* **Ciclo de vida:** `router.ts` crea un `AbortController` por navegación y lo aborta al desmontar la vista. Toda página o componente que registre listeners globales (`window`, `document`) o se suscriba a `src/state.ts` recibe la señal (`params: { signal: AbortSignal }`) y la pasa a `addEventListener(..., { signal })` y a la suscripción.
* **Estilos CSS nativos:** Variables solo en `src/styles/tokens.css`; en `src/styles/components/*.css` usa anidamiento nativo y consume siempre tokens (`var(--color-*)`, `var(--space-*)`), sin colores ni medidas hardcodeadas. No se permiten preprocesadores ni frameworks de CSS.
* **Accesibilidad:** `<html lang="es">`, todo input con `<label>`, errores de formulario en una región `aria-live`, gestión de foco al abrir y cerrar diálogos y respeto de `prefers-reduced-motion`.

## 4. Gráficos (Chart.js)
* **Ubicación:** Un componente por tipo en `src/components/charts/`. Recibe `{ data, signal }` y devuelve un `HTMLElement` (contenedor con `<canvas>` y alternativa textual).
* **Registro mínimo:** Prohibido `chart.js/auto`. Los controladores, elementos y escalas se registran una sola vez en `src/lib/chart-setup.ts` (solo los de donas, barras, líneas y radar, más `Tooltip` y `Legend`).
* **Tema desde tokens:** Colores y tipografía se leen de las custom properties con `getComputedStyle` en `src/lib/chart-theme.ts`. Prohibido hardcodear colores en la configuración de un gráfico.
* **Tooltips:** Usa los tooltips nativos de Chart.js (dibujados en el canvas). Si la spec exige tooltips HTML, se construyen con `h()` y clases; prohibido el atributo `style`.
* **Datos agregados:** El backend entrega los datos ya agregados (por día, categoría, etc.), validados con un esquema de `shared-contracts`. La webapp no recibe registros crudos para graficar salvo que la spec lo requiera. Si una serie supera unos 1.000 puntos, agrega en el backend o usa la decimación de Chart.js.
* **Ciclo de vida:** Llama a `chart.destroy()` cuando `signal` se aborta.
* **Movimiento reducido:** Si `prefers-reduced-motion: reduce`, desactiva las animaciones.
* **Accesibilidad:** El canvas lleva `role="img"` y `aria-label`. Junto a él, una tabla con los mismos datos (oculta visualmente con una clase, no con `display: none`), porque los tooltips solo funcionan con puntero.
* **Pruebas:** `happy-dom` no dibuja canvas. Prueba (a) las funciones puras que convierten los datos del contrato en configuración de Chart.js, y (b) que el componente renderiza el canvas, su `aria-label` y la tabla alternativa, con `Chart` mockeado (`vi.mock`).
* **Verificación:** Toda tarea con gráficos ejecuta el paso 3 de verificación (build y CSP sobre el resultado servido por Fastify).

## 4. Pruebas y Criterio de Verificación (`*.test.ts`)
* **Pruebas de componentes y lógica:** Co-ubicadas junto al código fuente, con Vitest y `happy-dom`. Verifican renderizado, interactividad, estados de carga y error, validación de formularios y limpieza de eventos.
* **Límite de `happy-dom`:** No implementa todas las APIs nativas. Si un componente usa `<dialog>`, Popover o Constraint Validation y `happy-dom` no las cubre, pruébalo en el modo navegador de Vitest; un test que pasa sin ejercitar la API no cuenta.
* **Trazabilidad:** Cada prueba referencia su criterio de aceptación; el nombre empieza con verbo y termina con el criterio: `it('shows an error message when login fails (AC-3)')`.
* **Verificación para marcar `[x]`**, en orden:
  1. `pnpm turbo run lint typecheck test --filter=webapp` con 0 errores y 0 warnings.
  2. `pnpm exec modern-web-guidance legacy-scan apps/webapp/` con 0 antipatrones reportados.
  3. Si la tarea añade estilos, fuentes, assets o scripts: `pnpm turbo run build --filter=webapp` y comprobar que el resultado servido por Fastify no genera violaciones de CSP (en desarrollo, Vite no aplica las cabeceras de Helmet).
  4. Criterios de `10-tech-stack.md` §7.
