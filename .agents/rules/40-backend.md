# Glob: apps/backend/**
# Regla: 40-backend

# Reglas Tácticas de Backend (Bloque 2 — Fastify + Drizzle)

> Se activa únicamente al operar en `apps/backend/`.
> Complementa a `10-tech-stack.md`, `20-coding-standards.md` y `30-sdd-process.md`.
> Lo que ya está definido en el 10 se referencia, no se repite.

## 1. Definición de Rutas y Validación (`src/modules/<dominio>/<dominio>.routes.ts`)
* **Contratos compartidos obligatorios:** Toda ruta debe registrarse utilizando `fastify-type-provider-zod` vinculando los esquemas (`body`, `params`, `querystring`, `response`) importados exclusivamente desde `packages/shared-contracts`. Está prohibido declarar esquemas Zod locales o inline en el backend.
* **Cambio de contrato tardío:** Si falta una propiedad, un tipo o un código de respuesta en `shared-contracts`, **DETENTE** (condición "Cambio de contrato tardío" de `30-sdd-process.md`). Prohibido parchar tipos localmente o editar `shared-contracts` sin volver al Bloque 1.
* **Seguridad perimetral y Rate Limit:** Las rutas públicas deben declararse explícitamente en la lista blanca de `src/plugins/auth.ts`; cualquier otra ruta exige autenticación por defecto. Configura en la definición de la ruta el `rateLimit` exacto especificado en la Matriz de seguridad del `plan.md`.
* **Métodos mutantes con cookie:** Pasan por el plugin de verificación de `Origin` (10 §5.6). No se reimplementa por ruta.
* **Delgadez de la capa HTTP:** El manejador de ruta (*handler*) se limita exclusivamente a extraer datos tipados del request, invocar al servicio correspondiente y retornar la respuesta HTTP. Cero lógica de negocio en las rutas.

## 2. Lógica de Negocio, Autorización y Errores (`src/modules/<dominio>/<dominio>.service.ts`)
* **Autorización estricta y Anti-IDOR:** Toda función de servicio que opere sobre recursos protegidos recibe la identidad del token (`sub`, `role`), valida permisos de negocio y filtra obligatoriamente las consultas a repositorios por el identificador del usuario propietario (`userId`).
* **Recurso ajeno = 404:** Si un usuario autenticado accede a un recurso que no le pertenece, lanza `ResourceNotFoundError` (no revela su existencia). `ForbiddenError` (403) se reserva para falta de rol sobre una acción conocida.
* **Errores de dominio:** Viven en `src/errors/` y extienden la clase base `DomainError`, que expone un `code` estable (ej. `INVALID_CREDENTIALS`). Preserva `{ cause }` al envolver otro error.
* **Desacople de Fastify:** Los servicios y repositorios no deben importar tipos de Fastify (`FastifyRequest`, `FastifyReply`) ni manipular objetos de respuesta HTTP o códigos de estado.

## 3. Manejador de Errores (`src/plugins/error-handler.ts`)
Es el único lugar que traduce errores a HTTP y al contrato `ApiError`:
* Entrada que no cumple el esquema → `422` (Fastify responde `400` por defecto; debe convertirse aquí).
* Error de serialización de la respuesta → `500` genérico, sin detalles; se registra en el log y nunca se devuelve.
* `DomainError` → status según su `code`, en un único mapa. Un `code` sin entrada responde `500` y se registra.
* Registra los eventos de seguridad del 10 §5.8 (login fallido, refresh reutilizado, acceso denegado).

## 4. Persistencia y Migraciones Code-First (`src/modules/<dominio>/<dominio>.repository.ts` y `src/db/`)
* **Proyección explícita de columnas:** Prohibido `select()` sin columnas. Selecciona únicamente las necesarias; ningún campo `*_hash` sale del repository salvo en la función que lo necesita para verificar.
* **Índices:** Toda consulta nueva en un repository debe tener su índice declarado en la tabla "Índices" del `plan.md`, o indicar que usa uno existente. Reglas en `10-tech-stack.md` §4.
* **Flujo de migraciones:** Sigue `10-tech-stack.md` §4. Delta de este archivo:
  1. Modifica exclusivamente los archivos de esquema en `src/db/schema/*.ts`.
  2. Genera la migración con `pnpm --filter backend db:generate` y revisa el SQL en `src/db/migrations/`.
  3. Aplícala con `pnpm --filter backend db:migrate` solo contra la base local (ver parada "Migraciones" del 30).
  * **Prohibido** escribir o editar migraciones `.sql` a mano y usar `drizzle-kit push`, salvo migraciones `--custom` justificadas en el `plan.md`.

## 5. Configuración y Logs
* `process.env` se lee únicamente en `src/config/env.ts`.
* Prohibido `console.*`; usa `request.log` o el logger inyectado.

## 6. Pruebas y Criterio de Verificación (`*.test.ts`)
* **Pruebas de servicio (`<dominio>.service.test.ts`):** Unitarias, con repositorios falsos en memoria y sin base de datos.
* **Pruebas de rutas (`<dominio>.routes.test.ts`):** Integración con `buildApp()` (`src/app.ts`) y `app.inject()`, contra PostgreSQL de pruebas.
  * Base separada (nombre terminado en `_test`) configurada en `.env.test`; si el nombre no termina en `_test`, la suite aborta.
  * Cada test parte de estado limpio. Prohibido apuntar a la base de desarrollo.
* **Casos obligatorios por endpoint:**
  1. Caso feliz (`200`, `201` o `204` con el contrato de salida validado).
  2. Entrada inválida o campos no permitidos (`422`).
  3. Falta de autenticación (`401`) y recurso ajeno (`404`) o falta de rol (`403`).
* **Trazabilidad:** Cada prueba referencia su criterio de aceptación; el nombre empieza con verbo y termina con el criterio: `it('rejects login with invalid password (AC-2)')`.
* **Verificación para marcar `[x]`:** `pnpm turbo run lint typecheck test --filter=backend` con 0 errores y 0 warnings, más los criterios de `10-tech-stack.md` §7.
