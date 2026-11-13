Problemas dentro del 30

1. Fases y archivos se mezclan. Las fases dicen specify.md, implement.md y verify.md, pero la sección 1 define solo tres archivos (spec.md, plan.md, tasks.md). La fase “Plan (plan.md)” usa el mismo nombre que el artefacto. Si esos .md son workflows de Antigravity, dilo explícitamente y sepáralos de los artefactos. Si no existen, nombra las fases sin archivo. Lo mismo aplica a “Bloqueo arquitectónico en implement.md”.

2. “Aprobada o completada” no dice quién aprueba. Define las compuertas humanas: tras Specify, Plan y Tasks el agente se detiene y espera tu visto bueno. Implement y Verify se repiten por tarea sin pedir aprobación.

3. “Implementar migraciones” contradice el 10. En el 10 las migraciones se generan con db:generate a partir del esquema Drizzle, no se escriben a mano. Redáctalo así para que el agente no escriba SQL manual.

4. El filtro del 10 choca con el Bloque 1. El 10 verifica con --filter=...[origin/main], que incluye los paquetes dependientes. Si un cambio de contrato rompe consumidores, el typecheck de backend y webapp falla en el Bloque 1 y la tarea nunca puede marcarse [x]. Hay que definir el alcance de verificación por bloque y preferir cambios aditivos en contratos.

5. Falta el cambio de contrato a mitad de camino. Es el caso más común: en el Bloque 3 descubres que falta un campo. Sin regla, el agente lo parcha localmente. Debe ser una parada: volver al Bloque 1, actualizar plan.md y obtener aprobación de nuevo.

6. Falta trazabilidad. El 00 pide resumir “qué spec cubre”, pero nada obliga a que las tareas citen criterios de aceptación. Numera los criterios (AC-1...) y exige que cada tarea cite los suyos. Así Verify puede comprobar que todos tienen al menos un test.

7. No hay cierre de feature. El proceso termina en la última tarea [x]. Falta una fase final con verificación completa y revisión de que todos los AC-n estén cubiertos.

8. Paradas incompletas. Faltan: cambio en spec.md después de aprobar el plan, dependencia no listada en el 10, discrepancia entre código y spec (la constitución dice reportarla), y ejecutar db:migrate fuera de local.

Gaps frente a los otros archivos
Seguridad en el plan. El 10 exige cumplir la sección 5 y tests 401/403/422, pero el plan no obliga a declarar por endpoint si es público o autenticado ni su rate limit. Sin eso, la decisión queda implícita.
Contratos duplicados. El 00 dice que specs/ es fuente de verdad de los contratos, y el 10 dice que lo es shared-contracts. Aclara que spec.md los describe en términos de negocio y shared-contracts es su implementación.
Bloques omitidos. No dice qué hacer si la feature no toca un paquete. Basta con aclarar que se omite el bloque y se mantiene el orden.
Inconsistencias en el 00 que afectan al 30

El 30 delega en el 10 sección 7, pero el 00 contradice esa cadena:

La regla 4 del 00 usa npm test y npm run lint, mientras el 10 prohíbe npm y usa pnpm. Es la más peligrosa: el agente puede generar un package-lock.json.
El 00 y el 10 definen “terminado” con criterios distintos. Deja solo el del 10 y haz que el 00 lo referencie.
El 00 pide JSDoc en la webapp y prohíbe any solo en el backend, pero el 10 declara TypeScript strict en todo.
El 00 dice verificar dependencias en el package.json, y el 10 dice que debe estar en el documento. Son criterios distintos.
El archivo se llama 20-coding-standars.md, con una d de menos.