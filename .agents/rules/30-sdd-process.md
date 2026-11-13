# Glob: **/*
# Regla: 30-sdd-process (Always On)

# Proceso Spec-Driven Development (SDD)

> Regla siempre activa. Define el ciclo de vida de una feature, sus artefactos,
> las compuertas de aprobación humana, el alcance de verificación por bloque y las condiciones de parada.

---

## 1. Artefactos de la Feature vs. Workflows del Agente

No confundir los **workflows de instrucción** (en `.agent/workflows/`, solo lectura) con los **artefactos de la feature** (en `specs/<NNN>-<nombre-kebab>/`, donde se documenta el trabajo):

### 1.1 Artefactos en `specs/<NNN>-<nombre-kebab>/`

1. **`spec.md` (El Qué y Por Qué — Negocio):**
   * Contexto, alcance funcional, reglas de negocio y casos borde.
   * **Descripción funcional de contratos:** qué datos entran y salen en términos de negocio (su implementación técnica en Zod vivirá en `packages/shared-contracts`).
   * **Criterios de Aceptación numerados (`AC-1`, `AC-2`, ...):** condiciones verificables y atómicas. Todo requisito debe tener su código `AC-n`.

2. **`plan.md` (El Cómo — Diseño Técnico):**
   * Dividido en tres secciones: `Shared Contracts`, `Backend` y `Webapp` (indicando `N/A` si una sección no aplica).
   * Lista exacta de archivos a crear o modificar por paquete.
   * Cambios en el esquema Drizzle (`apps/backend/src/db/schema/*.ts`) e índices/restricciones requeridas.
   * **Matriz de Seguridad por Endpoint (obligatoria si toca Backend):**
     Para cada ruta nueva o modificada, declarar explícitamente:
     * Método y ruta (`/api/...`)
     * Acceso: `Público` (requiere alta en lista blanca de `plugins/auth.ts`) o `Autenticado`
     * Regla de autorización y filtro de propiedad (anti-IDOR) a aplicar en `service`
     * Política de `rate-limit` (global 100/min o estricto 5/min)
   * Justificación de nuevas dependencias si aplican (según sección 5.10 de `10-tech-stack.md`).

3. **`tasks.md` (La Ejecución — Lista de Tareas):**
   * Tareas secuenciales numeradas (`T-01`, `T-02`...) agrupadas por bloques.
   * **Trazabilidad obligatoria:** cada tarea debe indicar qué criterios cubre (ej. `[Cubre: AC-1, AC-3]` o `[Soporte técnico para: AC-2]`).
   * Casillas de estado:
     * `[ ]` Pendiente
     * `[~]` En progreso (máximo UNA tarea en `[~]` a la vez)
     * `[x]` Completada y verificada

---

## 2. Fases del Ciclo SDD y Compuertas de Aprobación Humana

El desarrollo avanza en 6 fases estrictas. Respeta quién aprueba el paso a la siguiente fase:

| Fase | Workflow a seguir | Artefacto que produce/actualiza | Compuerta de control |
|---|---|---|---|
| **1. Specify** | `.agent/workflows/specify.md` | `specs/<feature>/spec.md` | **HUMANA (STOP):** Detente y espera aprobación del usuario. Prohibido escribir código o crear `plan.md`. |
| **2. Plan** | `.agent/workflows/plan.md` | `specs/<feature>/plan.md` | **HUMANA (STOP):** Detente y espera aprobación del usuario. Prohibido escribir código o crear `tasks.md`. |
| **3. Tasks** | `.agent/workflows/tasks.md` | `specs/<feature>/tasks.md` | **HUMANA (STOP):** Detente y espera aprobación del usuario antes de iniciar la implementación. |
| **4. Implement** | `.agent/workflows/implement.md` | Código fuente y tests de UNA tarea | **AUTOMÁTICA:** Toma la primera tarea `[ ]`, márcala `[~]`, impleméntala y pasa directo a **Verify**. |
| **5. Verify** | `.agent/workflows/verify.md` | Tarea marcada `[x]` en `tasks.md` | **AUTOMÁTICA POR TAREA:** Si las validaciones del paquete pasan, marca `[x]` y continúa con la siguiente tarea `[ ]` sin pedir aprobación humana (salvo condición de parada). |
| **6. Close** | `.agent/workflows/verify.md` | Resumen final de la feature | **CIERRE DE FEATURE:** Se ejecuta al marcar la última tarea `[x]`. Corre validación global y entrega reporte final al usuario. |

---

## 3. Orden de Bloques en el Monorepo y Alcance de Verificación

### 3.1 Orden de ejecución en `tasks.md`
Las tareas en `tasks.md` se organizan y ejecutan en este orden bloqueante. **Si una feature no requiere cambios en uno de los paquetes, se omite ese bloque manteniendo el orden relativo de los restantes:**

1. **Bloque 1 — Contratos (`packages/shared-contracts`):**
   * Crear o actualizar esquemas Zod (`.strict()`) y tipos inferidos.
   * **Regla de compatibilidad:** Preferir siempre cambios aditivos (nuevos campos opcionales o nuevos esquemas) para no romper consumidores existentes antes de tiempo.
2. **Bloque 2 — Backend (`apps/backend`):**
   * Si hay cambios de BD: modificar el esquema en `src/db/schema/*.ts`, generar la migración con `pnpm --filter backend db:generate`, inspeccionar el SQL generado y aplicar en local con `pnpm --filter backend db:migrate` (prohibido escribir archivos de migración SQL a mano).
   * Implementar `repository` → `service` (con autorización) → `routes` y sus pruebas (`*.test.ts`, incluyendo casos 2xx, 422 y 401/403).
3. **Bloque 3 — Webapp (`apps/webapp`):**
   * Implementar clientes en `src/api/` validando respuestas con `shared-contracts`, estado, páginas/componentes con DOM seguro (`h()` / `textContent`), estilos CSS y pruebas unitarias.

### 3.2 Alcance de verificación: Por Tarea vs. Cierre de Feature
Para evitar que un cambio en el Bloque 1 falle porque los consumidores de los Bloques 2 y 3 aún no han sido actualizados:

* **Durante Fase 5 (Verify de cada tarea individual):**
  Ejecuta la validación filtrando **únicamente el paquete que modifica la tarea actual**:
  `pnpm turbo run lint typecheck test --filter=<paquete-de-la-tarea>`
  *(Si la tarea es de `webapp`, ejecuta además `npx modern-web-guidance legacy-scan apps/webapp/`).*
* **Durante Fase 6 (Close al terminar todas las tareas de la feature):**
  Es obligatorio ejecutar la verificación cruzada completa de todos los paquetes afectados desde `main`:
  1. `pnpm turbo run lint typecheck test --filter=...[origin/main]`
  2. Comprobar que el 100% de los criterios `AC-1..N` de `spec.md` están cubiertos por al menos un test automatizado que pasa.

---

## 4. Condiciones de Parada (Stop Conditions)

Detén inmediatamente el ciclo automático **Implement ↔ Verify**, deja la tarea actual en `[~]`, explica el problema al usuario y espera su decisión en cualquiera de estos casos:

1. **Hueco o ambigüedad en `spec.md`:** Un caso borde, regla de validación o comportamiento de error no está especificado.
2. **Cambio en `spec.md` post-aprobación:** Si `spec.md` se modifica después de haber aprobado `plan.md` o `tasks.md`, el plan queda invalidado. Detente, actualiza `plan.md` y `tasks.md` y solicita aprobación humana nuevamente.
3. **Cambio de contrato a mitad de camino (Bloques 2 o 3):** Si al implementar `backend` o `webapp` descubres que falta un campo o necesitas alterar un esquema en `packages/shared-contracts`, **está prohibido parcharlo localmente o editar `shared-contracts` en silencio**. Detente, propón el cambio en `plan.md`, obtén aprobación y regresa al Bloque 1.
4. **Discrepancia entre código y `spec.md`:** El código existente contradice lo que asume la especificación.
5. **Desviación arquitectónica o de alcance:** La tarea requiere tocar archivos no previstos en `plan.md` o cambiar el diseño acordado.
6. **Dependencia no aprobada:** Se requiere una librería ausente en `10-tech-stack.md` o en el `package.json` del paquete.
7. **Operación de BD restringida:** Se intenta editar una migración ya aplicada o ejecutar `db:migrate` contra un entorno que no sea la base de datos local de desarrollo.
8. **Fallo persistente en Verify:** Un test, linter o `typecheck` falla tras 2 intentos consecutivos de corrección sobre la misma tarea.