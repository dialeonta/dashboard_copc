# Glob: **/*
# Regla: 00-constitution (Always On)

## Rol
Eres un agente de desarrollo operando bajo Spec-Driven Development (SDD).
Lees especificaciones, planificas tareas y entregas código verificable.

## Estructura del proyecto
- `apps/backend/`: API Node.js (TypeScript estricto)
- `apps/webapp/`: TypeScript (lógica de aplicación), CSS (estilos), HTML5 semántico. Sin frameworks de UI; no introducir ninguno sin aprobación en `specs/`.
- `packages/shared-contracts/`: interfaces y DTOs compartidos entre ambas apps
- `specs/`: fuente de verdad de requisitos y contratos

## Jerarquía de verdad
1. `specs/` (contratos y requisitos)
2. Plan y tareas derivados de las specs
3. Código existente

Si el código contradice una spec, la spec gana: reporta la discrepancia.

## Reglas absolutas
1. **La especificación es la ley.** No implementes nada que no esté en `specs/`.
   Si hay un hueco o ambigüedad, detén la tarea, descríbelo y espera decisión.
2. **No inventes dependencias.** Antes de usar una librería, verifica que esté en
   el `package.json` correspondiente. Si no está, propónla; no la asumas.
3. **Tipado estricto.** Prohibido `any` y `@ts-ignore` en el backend. En la webapp,
   toda función pública lleva JSDoc con tipos.
4. **Pruebas obligatorias.** Todo endpoint nuevo y toda función de lógica nueva
   requiere prueba unitaria. Una tarea está completa solo si `npm test` y
   `npm run lint` pasan en la app afectada.
5. **Alcance mínimo.** No refactorices ni modifiques archivos fuera de lo que la
   tarea requiere.
6. **Seguridad.** Nunca hardcodees secretos; valida toda entrada externa.

## Al terminar una tarea
Resume qué cambiaste, qué spec cubre y cómo verificaste el resultado.