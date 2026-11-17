# Glob: **/*
# Regla: 20-coding-standards (Always On)

# Estándares de Código y Convenciones

> Regla siempre activa. Define exclusivamente nomenclatura, idioma, estructura de funciones y convenciones de escritura de código.
> Las decisiones de arquitectura, capas y librerías están en `10-tech-stack.md`.

## 1. Idioma
* **Código en Inglés:** Nombres de variables, funciones, tipos, propiedades, esquemas, tablas/columnas y comentarios técnicos.
* **Interfaz en Español:** Textos visibles para el usuario final y mensajes de validación/error de cara al cliente.

## 2. Nomenclatura
* **Variables, funciones y métodos:** `camelCase`. Las funciones deben iniciar con un verbo descriptivo (`calculateTotal`, `parseToken`) y los booleanos con prefijo afirmativo (`isValid`, `hasPermission`, `canEdit`).
* **Tipos, Interfaces y Clases:** `PascalCase` sin prefijos como `I` o `T` (`UserSession`, `AuthTokenPayload`).
* **Esquemas de validación (Zod):** Identificador del esquema en `camelCase` con sufijo `Schema` (`loginRequestSchema`); tipo inferido con el mismo nombre base en `PascalCase` (`LoginRequest`).
* **Clases de Error:** `PascalCase` con sufijo `Error` (`InvalidCredentialsError`, `AccountLockedError`).
* **Constantes globales y variables de entorno:** `UPPER_SNAKE_CASE`.
* **Archivos y directorios:** `kebab-case` para todos los nombres de archivo y carpetas (`user-profile.ts`, `auth-form.css`).
* **Índices y restricciones de base de datos:** `idx_<tabla>_<columnas>` para índices e
  `uq_<tabla>_<columnas>` para restricciones únicas (`idx_tasks_user_id_created_at`).
  Nombre siempre explícito en el esquema.

## 3. Estructura de Funciones y Módulos
* **Funciones sobre Clases:** Preferir funciones puras y módulos ES sobre clases con estado. Reservar `class` únicamente para errores personalizados o adaptadores que requieran ciclo de vida.
* **Retorno temprano (Early Return):** Validar precondiciones al inicio de la función y retornar o lanzar error de inmediato. Evitar bloques `else` innecesarios y anidación mayor a 2 niveles.
* **Inmutabilidad:** Usar `const` por defecto; usar `let` solo para acumuladores locales. Prohibido `var` y prohibido mutar objetos o arreglos recibidos por parámetro.
* **Parámetros:** Si una función recibe más de 3 parámetros, agruparlos en un único objeto tipado (`options` o `params`).
* **Fronteras explícitas:** Toda función exportada debe declarar explícitamente los tipos de sus parámetros y su tipo de retorno.
* **Exportaciones:** Usar siempre *named exports* (`export function ...`, `export type ...`). Prohibido `export default`.

## 4. Manejo de Errores en Código
* **Instancias de `Error` obligatorias:** Prohibido hacer `throw` de strings, números u objetos planos; lanzar siempre instancias de clases que hereden de `Error`.
* **Preservación de contexto:** Al capturar un error para envolverlo y relanzarlo como otro error, pasar siempre el error original en la propiedad estándar `{ cause: err }`.
* **Tipado en `catch`:** Tratar la variable del bloque `catch (err)` como `unknown` y estrechar su tipo (`instanceof Error`) antes de leer sus propiedades.

## 5. Convenciones de Tests
* **Co-ubicación:** Las pruebas se ubican en el mismo directorio que el archivo que prueban, usando el mismo nombre base y el sufijo `.test.ts` (ej. `router.ts` → `router.test.ts`).
* **Estructura AAA:** Organizar el cuerpo de cada test en tres bloques visuales separados por una línea en blanco: *Arrange* (preparación), *Act* (ejecución) y *Assert* (verificación).
* **Descripciones declarativas:** Los bloques `test` / `it` deben describir el comportamiento esperado en inglés comenzando con un verbo (`it('returns 401 when token is expired')`), sin mencionar detalles internos de implementación.