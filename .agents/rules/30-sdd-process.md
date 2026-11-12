# Glob: **/*
# Regla: 30-sdd-process (Always On)

## Flujo SDD
Cada feature vive en `specs/[id-feature]/` y se avanza en este orden estricto.
Al terminar cada paso, detente y espera aprobación antes de pasar al siguiente.

1. **`spec.md` (El Qué):** objetivos, casos de uso y criterios de aceptación.
   No escribas código ni decisiones técnicas aquí.
2. **`plan.md` (El Cómo):** arquitectura técnica dividida en "Contratos",
   "Backend" y "Webapp".
3. **`tasks.md` (La Ejecución):** primero genera la lista de tareas; cada una
   referencia el criterio de aceptación que cubre. Luego ejecútalas una por una,
   marcando `[x]` solo cuando la tarea esté verificada.

Si `spec.md` cambia, `plan.md` y `tasks.md` quedan obsoletos: reporta y revisa.

## Fases de implementación
Aplica solo las fases que la feature requiera, siempre en este orden.
Una fase no inicia hasta que la anterior cumpla la definición de "completado".

- **Fase 1 (Contratos):** interfaces y DTOs en `packages/shared-contracts/`.
  Verificación: `npm run build -w packages/shared-contracts` y typecheck.
- **Fase 2 (Backend):** lógica en `apps/backend/` usando los contratos de la Fase 1.
  Verificación: build, typecheck, lint y tests del backend.
- **Fase 3 (Webapp):** UI y estado en `apps/webapp/`.
  Verificación: build, typecheck, lint y tests de la webapp.

## Cambios de contrato
Si durante la Fase 2 o 3 necesitas modificar un contrato, detente, actualiza
`plan.md`, vuelve a la Fase 1 y reejecuta las fases posteriores afectadas.
Nunca definas tipos duplicados localmente para evitarlo.