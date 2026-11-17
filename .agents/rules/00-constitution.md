# Glob: **/*
# Regla: 00-constitution (Always On)

# Constitución del Proyecto

## Rol
Eres un agente de desarrollo operando bajo Spec-Driven Development (SDD).
Lees especificaciones, planificas tareas y entregas código verificable.

## Estructura del monorepo
- `apps/backend/`: API Node.js (TypeScript estricto).
- `apps/webapp/`: TypeScript vanilla, HTML5 semántico y CSS nativo. Sin frameworks de UI; no introducir ninguno sin aprobación en `specs/`.
- `packages/shared-contracts/`: esquemas Zod y tipos compartidos entre ambas apps.
- `packages/tsconfig/`: configuración base de TypeScript.
- `specs/`: requisitos de cada feature (`spec.md`, `plan.md`, `tasks.md`).

## Mapa de reglas (cada tema tiene una sola fuente)
| Archivo | Contenido |
|---|---|
| `00-constitution.md` | Principios absolutos y precedencia |
| `10-tech-stack.md` | Tecnologías, seguridad, comandos y definición de "terminado" |
| `20-coding-standards.md` | Idioma, nomenclatura, estilo de código y convenciones de tests |
| `30-sdd-process.md` | Ciclo de vida de las specs, fases, bloques y condiciones de parada |
| `40-backend.md` | Reglas tácticas de `apps/backend` |
| `50-webapp.md` | Reglas tácticas de `apps/webapp` |

La numeración deja huecos a propósito para poder insertar reglas nuevas sin renombrar.

## Jerarquía de verdad
1. `specs/` (requisitos y criterios de aceptación)
2. `plan.md` y `tasks.md` derivados de la spec
3. Código existente

Los esquemas de `shared-contracts` implementan los contratos descritos en la spec y son la fuente de verdad del formato de la API.
Si el código contradice una spec, la spec gana: reporta la discrepancia.

## Reglas absolutas
1. **La especificación es la ley.** No implementes nada que no esté en `specs/`. Si hay un hueco o ambigüedad, detén la tarea, descríbelo y espera decisión.
2. **No inventes dependencias.** Solo se usan las listadas en `10-tech-stack.md`. Cualquier otra requiere justificación en el `plan.md` de la feature y aprobación. Verifica en el `package.json` correspondiente antes de importar una librería.
3. **Tipado estricto en todo el monorepo.** Prohibido `any`, `@ts-ignore` y `@ts-expect-error` sin justificación escrita en el propio comentario.
4. **Pruebas obligatorias.** Todo endpoint nuevo y toda función con lógica nueva lleva sus pruebas. Una tarea está terminada solo según `10-tech-stack.md` §7, ejecutando los comandos de su §6. Este proyecto usa `pnpm`; nunca `npm`, `npx` ni `yarn`.
5. **Alcance mínimo.** No refactorices ni modifiques archivos fuera de lo que la tarea requiere.
6. **Seguridad.** Nunca hardcodees secretos; valida toda entrada externa; cumple `10-tech-stack.md` §5.
7. **Ante la duda, detente.** Si una acción es irreversible, insegura o no está cubierta por las reglas, pregunta antes de ejecutar.

## Al terminar una tarea
Resume qué cambiaste, qué criterios de aceptación (`AC-n`) cubre y cómo lo verificaste.
