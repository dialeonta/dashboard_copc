# Glob: **/*
# Regla: 30-sdd-process (Always On)

# Proceso Spec-Driven Development (SDD)

> Regla siempre activa. Define EXCLUSIVAMENTE el ciclo de vida de las especificaciones,
> sus artefactos y el orden de ejecución de tareas.
> Tecnologías, comandos y criterios técnicos de "terminado": `10-tech-stack.md` (§6 y §7).
> Las condiciones de parada se citan por nombre desde otros archivos; no cambies sus nombres.

## 1. Artefactos por feature

Toda funcionalidad nueva o cambio vive en `specs/<NNN>-<nombre-kebab>/` con tres archivos de responsabilidades separadas:

1. **`spec.md` (El Qué y Por Qué):** requisitos funcionales, casos de uso, reglas de negocio, contratos de datos en términos de negocio y criterios de aceptación verificables, numerados (`AC-1`, `AC-2`...). No incluye decisiones de implementación. Los esquemas zod de `shared-contracts` implementan estos contratos y son la fuente de verdad del formato de la API.
2. **`plan.md` (El Cómo):** diseño técnico dividido en `Shared Contracts`, `Backend` y `Webapp`. Lista los archivos a crear o modificar, cambios en tablas, decisiones de diseño y justificación de nuevas dependencias. Incluye la **Matriz de seguridad**:

   | Endpoint | Acceso (`Público` / `Autenticado`) | Rate limit (global 100/min o estricto 5/min) |
   |---|---|---|
   | `POST /api/...` | ... | ... |

3. **`tasks.md` (La Ejecución):** lista secuencial y atómica de tareas numeradas, agrupadas por bloque (sección 3). Cada tarea cita los `AC-n` que cubre.
   * `[ ]` Pendiente
   * `[~]` En progreso (máximo una tarea en este estado a la vez)
   * `[x]` Completada y verificada
   * `[!]` Bloqueada, esperando decisión del usuario

## 2. Fases del ciclo de vida (bloqueantes)

Está prohibido saltar a una fase sin que la anterior esté aprobada o completada.
Tras **Specify**, **Plan** y **Tasks**, detente y espera aprobación explícita del usuario.

1. **Specify:** redactar o actualizar `spec.md`. Prohibido tocar código fuente.
2. **Plan:** redactar `plan.md` a partir de una `spec.md` aprobada. Prohibido tocar código fuente.
3. **Tasks:** desglosar `plan.md` en pasos pequeños y verificables dentro de `tasks.md`.
4. **Implement:** tomar la primera tarea `[ ]`, marcarla `[~]` y escribir únicamente el código y las pruebas de esa tarea.
5. **Verify:** ejecutar las validaciones de `10-tech-stack.md` §7 más las del bloque en curso. Solo si pasan todas, marcar la tarea `[x]`. Repetir 4 y 5 por cada tarea.
6. **Close:** al completar el último bloque, ejecutar la verificación de todo lo afectado (`10-tech-stack.md` §6), comprobar que cada `AC-n` tiene al menos un test y resumir: qué cambió, qué `AC-n` cubre y cómo se verificó.

Si `spec.md` cambia después de aprobar el plan, `plan.md` y `tasks.md` quedan obsoletos: detente y reporta.

## 3. Orden de ejecución en el monorepo

Aplica solo los bloques que la feature requiera, siempre en este orden. Si la feature no toca un paquete, se omite su bloque y se mantiene el orden de los demás. Un bloque no inicia hasta que todas las tareas del anterior estén `[x]`.

1. **Bloque 1 — Contratos (`packages/shared-contracts`):** definir o actualizar esquemas y tipos de rutas. Prefiere cambios aditivos. Si un cambio rompe consumidores existentes, `plan.md` debe declararlo; en ese caso el `typecheck` de backend y webapp se exige al cerrar sus propios bloques, no el del Bloque 1.
2. **Bloque 2 — Backend (`apps/backend`):** esquema Drizzle, migración generada con `db:generate` (nunca SQL escrito a mano; ver `10-tech-stack.md` §4), repositorios, servicios, rutas y sus pruebas, consumiendo los contratos del Bloque 1. Reglas en `40-backend.md`.
3. **Bloque 3 — Webapp (`apps/webapp`):** clientes API, estado, páginas, componentes y sus pruebas, consumiendo los contratos del Bloque 1. Reglas en `50-webapp.md`.

## 4. Condiciones de parada

Detén la ejecución inmediatamente y solicita decisión al usuario si ocurre cualquiera de estos casos:

* **Ambigüedad en `spec.md`:** un caso borde, regla de negocio o respuesta de error no está definido.
* **Código contradice la spec:** reporta la discrepancia; la spec gana.
* **Plan inviable:** la implementación revela que `plan.md` no se puede cumplir o requiere modificar archivos fuera del alcance previsto. Propón el ajuste en `plan.md` y espera aprobación antes de escribir código.
* **Cambio de contrato tardío:** durante el Bloque 2 o 3 falta una propiedad, un código de respuesta o un tipo en `shared-contracts`. Actualiza `plan.md`, obtén aprobación y reejecuta desde el Bloque 1. Prohibido parchar tipos localmente.
* **Dependencia no listada:** la tarea requiere una librería que no está en `10-tech-stack.md`. Justifícala en `plan.md` y espera aprobación.
* **Migraciones:** no ejecutes `db:migrate` fuera del entorno local sin aprobación explícita.
* **Fallo persistente:** una prueba o validación falla más de 2 intentos consecutivos tras intentar corregirla.