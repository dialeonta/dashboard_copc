# Glob: **/*
# Regla: 10-tech-stack (Always On)


# Tech Stack y Políticas de Seguridad

> Regla siempre activa. Define QUÉ tecnologías se usan y QUÉ está prohibido.
> El estilo de código (nombres, formato, estructura de funciones) está en `20-coding-standards.md`.
> Si una tarea requiere algo que no está en este documento, DETENTE y propón el cambio
> en el `plan.md` de la feature antes de implementarlo.

---

## 1. Visión general

- Monorepo con **pnpm workspaces + Turborepo**.
- Dos aplicaciones y paquetes compartidos:
  - `apps/backend` — API REST en Node.js + Fastify + PostgreSQL (code first con Drizzle).
  - `apps/webapp` — Frontend en TypeScript vanilla + HTML5 + CSS nativo, empaquetado con Vite.
  - `packages/shared-contracts` — Esquemas zod y tipos compartidos (fuente de verdad de la API).
  - `packages/tsconfig` — Configuración base de TypeScript.
- Principio rector: **la solución más simple que cumpla la spec**. No añadir capas, patrones
  ni librerías "por si acaso".

### Arquitectura de despliegue (un solo origen)

```
Desarrollo:   Navegador ──► Vite (:5173) ──┬── /*     → frontend
                                           └── /api/* → proxy a Fastify (:3000)

Producción:   Navegador ──► Fastify ──┬── /api/* → rutas de la API
                                      └── /*     → apps/webapp/dist (estáticos)
```

- Frontend y API se sirven SIEMPRE desde el mismo origen. No se configura CORS para el navegador.
- En producción no existe servidor de Vite; Fastify sirve `apps/webapp/dist` con `@fastify/static`.

---

## 2. Stack aprobado

### 2.1 Base y tooling

| Área | Tecnología | Notas |
|---|---|---|
| Runtime | Node.js 24 LTS | Fijar en `.nvmrc` y `engines` |
| Lenguaje | TypeScript (modo `strict`) | En backend, frontend y paquetes |
| Gestor de paquetes | pnpm | Usar `pnpm`; nunca `npm` ni `yarn` |
| Monorepo | Turborepo | Tareas: `dev`, `build`, `test`, `lint`, `typecheck` |
| Lint y formato | Biome | Única herramienta de lint/format. No añadir ESLint ni Prettier |
| Tests | Vitest | Backend y frontend |
| Base de datos local | PostgreSQL vía `docker-compose.yml` | Misma versión mayor que producción |

### 2.2 Backend (`apps/backend`)

| Área | Librería | Uso |
|---|---|---|
| Framework HTTP | `fastify` | Servidor y rutas |
| Validación y tipado de rutas | `zod` + `fastify-type-provider-zod` | Esquemas importados de `shared-contracts` |
| ORM (code first) | `drizzle-orm` + `drizzle-kit` (dev) | Esquema en TS → migraciones generadas |
| Driver PostgreSQL | `postgres` (postgres.js) | Conexión usada por Drizzle |
| Autenticación | `@fastify/jwt` | Access tokens JWT |
| Cookies | `@fastify/cookie` | Transporte de tokens al navegador |
| Hash de contraseñas | `@node-rs/argon2` | argon2id |
| Cabeceras de seguridad | `@fastify/helmet` | CSP, HSTS, etc. |
| Rate limiting | `@fastify/rate-limit` | Global y estricto en auth |
| Archivos estáticos | `@fastify/static` | Servir `apps/webapp/dist` en producción |
| Logs | `pino` (incluido en Fastify) | Con redaction obligatoria |

### 2.3 Frontend (`apps/webapp`)

| Área | Tecnología | Uso |
|---|---|---|
| Lenguaje | TypeScript vanilla | Sin frameworks (React, Vue, Svelte, Lit, jQuery: PROHIBIDOS) |
| Marcado | HTML5 semántico | `index.html` como punto de entrada |
| Estilos | CSS nativo | Custom properties, anidamiento nativo. Sin Sass, Less ni Tailwind |
| Build y dev server | `vite` (dev) | Bundling, hash de assets, proxy `/api` en desarrollo |
| Router | Propio (`src/router.ts`, History API) | Sin librerías de routing |
| Estado | Módulo propio simple (`src/state.ts`) | Sin librerías de estado |
| HTTP | `fetch` nativo envuelto en `src/api/` | Respuestas validadas con `shared-contracts` |
| Tests | `vitest` + `happy-dom` | Tests de componentes y lógica |

### 2.4 Shared (`packages/shared-contracts`)

- Única dependencia permitida: `zod`.
- Exporta TypeScript directamente (`"main": "./src/index.ts"`); no tiene paso de build propio.
- Organizado por dominio: `src/<dominio>/<dominio>.schemas.ts` y `<dominio>.routes.ts`.

---

## 3. Estructura de carpetas

```
apps/backend/src/
├── modules/<dominio>/
│   ├── <dominio>.routes.ts       # HTTP: valida entrada, llama al service, responde
│   ├── <dominio>.service.ts      # Lógica de negocio y AUTORIZACIÓN
│   └── <dominio>.repository.ts   # Único lugar con acceso a Drizzle
├── db/
│   ├── schema/                   # Tablas en TS (code first)
│   ├── migrations/               # Generadas por drizzle-kit, versionadas en git
│   └── client.ts
├── plugins/                      # auth, helmet, rate-limit, error-handler, static
├── config/env.ts                 # Variables de entorno validadas con zod
├── app.ts                        # buildApp(): construye la app (usada por tests)
└── server.ts                     # Arranque

apps/webapp/
├── index.html
├── public/                       # Assets copiados tal cual (favicon, imágenes)
└── src/
    ├── main.ts
    ├── router.ts
    ├── state.ts
    ├── api/                      # Clientes fetch por dominio
    ├── pages/                    # Una función por página → HTMLElement
    ├── components/               # Funciones que crean elementos DOM
    ├── lib/dom.ts                # Helper h() para crear elementos de forma segura
    └── styles/
        ├── tokens.css            # Variables: colores, tipografía, espaciado
        ├── base.css
        └── components/*.css
```

Reglas de capas en el backend:
- `routes` NO accede a la base de datos; `repository` NO contiene lógica de negocio.
- Las dependencias solo fluyen hacia abajo: `routes → service → repository`.
- Prohibido importar código de `apps/webapp` desde `apps/backend` y viceversa.
  Lo compartido va en `packages/shared-contracts`.

---

## 4. Base de datos (code first)

- El esquema se define SOLO en `apps/backend/src/db/schema/*.ts`.
- Flujo de cambios:
  1. Modificar el esquema en TS.
  2. `pnpm --filter backend db:generate` → genera la migración SQL.
  3. Revisar el SQL generado antes de aplicarlo.
  4. `pnpm --filter backend db:migrate`.
- Prohibido editar a mano migraciones ya aplicadas y prohibido usar `drizzle-kit push`
  fuera de entornos locales desechables.
- Toda tabla incluye `id` (UUID), `created_at` y `updated_at`.
- Usar restricciones en la base de datos (`NOT NULL`, `UNIQUE`, `FOREIGN KEY`, `CHECK`),
  no solo validación en código.

---

## 5. Políticas de seguridad (OBLIGATORIAS)

### 5.1 Autenticación con JWT

Diseño pensado para el navegador hoy y para clientes externos (por ejemplo, móvil) en el futuro:

| Token | Formato | Vida | Almacenamiento |
|---|---|---|---|
| Access token | JWT firmado | 15 minutos | Navegador: cookie `HttpOnly; Secure; SameSite=Lax; Path=/api`. Clientes externos: cabecera `Authorization: Bearer` |
| Refresh token | Valor opaco aleatorio (32 bytes) | 7 días | Navegador: cookie `HttpOnly; Secure; SameSite=Strict; Path=/api/auth/refresh`. En BD solo su hash (SHA-256) |

Reglas:
- PROHIBIDO guardar tokens en `localStorage`, `sessionStorage` o variables globales del frontend.
- Algoritmo fijo y lista blanca explícita en la verificación (`HS256` con secreto de al menos
  32 bytes aleatorios). Rechazar `alg: none` y cualquier algoritmo no listado.
- Validar siempre `exp`, `iss` y `aud`.
- El payload contiene solo `sub` (id de usuario), `role` y claims estándar. Nunca datos personales
  ni secretos: un JWT es legible por cualquiera.
- Rotación de refresh tokens: cada uso invalida el anterior y emite uno nuevo. Si se detecta la
  reutilización de un refresh token ya usado, revocar todos los tokens de ese usuario.
- Logout y cambio de contraseña revocan los refresh tokens en la base de datos.
- Contraseñas hasheadas con argon2id. Nunca guardar, devolver ni loguear contraseñas en claro.
- Mensajes de error genéricos en login y recuperación ("Credenciales inválidas"), sin revelar
  si un email existe.

### 5.2 Autorización

- Denegar por defecto: toda ruta requiere autenticación salvo las incluidas en una lista
  explícita de rutas públicas en `plugins/auth.ts`.
- La autorización se comprueba en la capa `service`, nunca solo en el frontend.
- Toda consulta de un recurso de usuario filtra por el propietario (prevención de IDOR).
- Los roles se leen del token verificado, nunca de datos enviados por el cliente.

### 5.3 Validación de entrada

- Todo lo que entra a la API (body, params, query) se valida con un esquema zod de
  `shared-contracts`. Sin esquema, no hay endpoint.
- Los esquemas de objetos usan `.strict()` para rechazar campos desconocidos.
- Límite de body en Fastify: `bodyLimit` de 1 MB (ajustar solo si la spec lo justifica).
- Las variables de entorno se validan con zod al arrancar; si falta una, la app no arranca.

### 5.4 Base de datos e inyección

- Acceso a datos SOLO mediante el query builder de Drizzle o el template `sql` parametrizado.
- PROHIBIDO concatenar o interpolar strings para construir SQL.
- PROHIBIDO usar `sql.raw()` con cualquier dato proveniente del usuario.
- Dos usuarios de PostgreSQL:
  - `app_user`: solo `SELECT`, `INSERT`, `UPDATE`, `DELETE` sobre las tablas de la aplicación.
  - `migrator`: privilegios DDL, usado solo por las migraciones.

### 5.5 Frontend y XSS

- PROHIBIDO: `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`, `eval`,
  `new Function`, y `setTimeout`/`setInterval` con strings.
- El texto dinámico se inserta con `textContent` o con el helper `h()` de `src/lib/dom.ts`.
- Los atributos `href`/`src` dinámicos solo aceptan URLs `https:`, `http:` o rutas relativas
  (nunca `javascript:`).
- Las respuestas de la API se validan con el esquema de `shared-contracts` antes de usarlas.
- Prohibidos los manejadores inline en el HTML (`onclick="..."`) y los `<script>` inline:
  incumplen la CSP.

### 5.6 Cabeceras HTTP y CSRF

- `@fastify/helmet` con CSP estricta:
  `default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:;
  connect-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none';
  form-action 'self'`.
  Prohibido añadir `'unsafe-inline'` o `'unsafe-eval'`.
- HSTS activo en producción.
- CSRF: además de `SameSite` en las cookies, toda petición `POST`, `PUT`, `PATCH` o `DELETE`
  autenticada por cookie verifica que la cabecera `Origin` coincida con el origen configurado.
- No se habilita CORS para orígenes de navegador. Si en el futuro hace falta, se define
  una lista blanca explícita en el `plan.md` de la feature (nunca `*` con credenciales).

### 5.7 Rate limiting

- Global: 100 peticiones por minuto por IP.
- `POST /api/auth/login`, `/register` y `/refresh`: 5 peticiones por minuto por IP.

### 5.8 Errores y logs

- Un único manejador global de errores en `plugins/error-handler.ts` que responde con el
  formato `ApiError` de `shared-contracts`.
- En producción nunca se devuelven stack traces, mensajes de la base de datos ni rutas internas.
- Redaction obligatoria en pino para: `password`, `token`, `accessToken`, `refreshToken`,
  `authorization`, `cookie`, `set-cookie`.
- Registrar eventos de seguridad: login fallido, refresh token reutilizado y acceso denegado.

### 5.9 Secretos y configuración

- Los secretos solo se leen de variables de entorno (Node `--env-file` en local).
- `.env` en `.gitignore`. Mantener `.env.example` con todas las claves y sin valores reales.
- PROHIBIDO escribir secretos, tokens o contraseñas en el código, los tests, los logs o los
  archivos de `specs/`.

### 5.10 Dependencias y cadena de suministro

- PROHIBIDO añadir una dependencia que no esté en este documento sin justificarla en el
  `plan.md` de la feature (problema que resuelve, alternativa nativa descartada, mantenimiento
  activo y licencia).
- Preferir siempre las APIs nativas de Node.js y del navegador antes que una librería.
- `pnpm-lock.yaml` versionado. En CI: `pnpm install --frozen-lockfile` y `pnpm audit --prod`.
- Dependencias de solo build o test en `devDependencies`.

---

## 6. Comandos

| Acción | Comando |
|---|---|
| Instalar | `pnpm install` |
| Levantar todo | `pnpm dev` |
| Solo backend | `pnpm turbo run dev --filter=backend` |
| Solo webapp | `pnpm turbo run dev --filter=webapp` |
| Tests de una app | `pnpm turbo run test --filter=<app>` |
| Verificar lo afectado | `pnpm turbo run lint typecheck test --filter=...[origin/main]` |
| Añadir dependencia | `pnpm add <pkg> --filter <app>` (solo si cumple 5.10) |
| Generar migración | `pnpm --filter backend db:generate` |
| Aplicar migraciones | `pnpm --filter backend db:migrate` |
| Base de datos local | `docker compose up -d` |

---

## 7. Definición de "terminado" (técnica)

### Para marcar una tarea `[x]` en `tasks.md`:
1. `pnpm turbo run lint typecheck test --filter=<paquete-de-la-tarea>` pasa sin errores.
2. Si la tarea toca `apps/webapp`, `npx modern-web-guidance legacy-scan apps/webapp/` pasa sin antipatrones legacy.
3. No introduce dependencias fuera de este documento (o están aprobadas en `plan.md`).
4. Cumple todas las políticas de la sección 5 aplicables.
5. Los endpoints nuevos incluyen tests de: caso feliz, entrada inválida (422) y falta de autenticación o autorización (401/403).

### Para dar por cerrada la feature completa (Fase Close):
1. Todas las tareas de `tasks.md` están en `[x]`.
2. `pnpm turbo run lint typecheck test --filter=...[origin/main]` pasa en todos los paquetes afectados y sus dependientes.
3. Todos los criterios de aceptación (`AC-n`) de `spec.md` cuentan con pruebas automatizadas verificadas.
