# 📖 Especificaciones Técnicas (Specs) — QA Call Center

> Documentación central del sistema de Aseguramiento de Calidad (QA) bajo el enfoque de **Spec-Driven Development (SDD)**: primero se escribe la especificación, luego el plan, luego las tareas, y solo entonces se toca el código.
>
> Este directorio es la fuente de la verdad para desarrolladores y Agentes de IA que colaboran en el proyecto.

## 🏗️ Estructura del Directorio

La arquitectura de la documentación se divide en la constitución (reglas inmutables) y los features (módulos iterativos).

```text
spec/
├── constitution/            ← Reglas estables del proyecto (cambian poco)
│   ├── mission.md           ← Qué construimos (App de QA) y para quién (Auditores/Supervisores)
│   ├── tech-stack.md        ← Tecnologías (Vanilla JS, LESS) y Patrón Adaptador
│   └── roadmap.md           ← Orden de desarrollo de los features
└── features/                ← Una carpeta por módulo funcional
    └── NNN-nombre-feature/
        ├── spec.md          ← Qué hace + Modelos (DTOs) + Reglas de negocio
        ├── plan.md          ← Cómo se implementa (Lógica local vs. API REST)
        └── tasks.md         ← Checklist de tareas a nivel de código
```

## 🗺️ Mapa de Features (Roadmap)

El sistema está diseñado modularmente. Las dependencias dictan el orden estricto de desarrollo:


| **ID**  | **Feature**            | **Descripción Breve**                                                   | **Dependencias** |
| ------- | ---------------------- | ----------------------------------------------------------------------- | ---------------- |
| **001** | `user-access`          | Autenticación, sesión híbrida y RBAC (CALIDAD, SUPERVISOR, etc.).       | Ninguna          |
| **002** | `admin-panel`          | Orquestador visual (Dashboard). SPA con menú lateral plegable.          | `001`            |
| **003** | `crud-clientes`        | Catálogo maestro de clientes (Entidad raíz).                            | `002`            |
| **004** | `crud-campanas`        | Gestión de campañas atadas estrictamente a un cliente.                  | `003`            |
| **005** | `crud-users`           | Identidad unificada y accesos operativos (Carga masiva CSV).            | `004`            |
| **006** | `crud-matriz`          | Configuración de parámetros de evaluación y reglas matemáticas.         | `003`            |
| **007** | `evaluacion-calidad`   | Formulario operativo reactivo con motor `ScoreCalculator` puro.         | `005`, `006`     |
| **008** | `historial-monitoreos` | Panel de lectura y filtros con aislamiento de datos automático por rol. | `007`            |

Aquí tienes el archivo listo para que lo copies y lo guardes como `README.md` en la raíz de tu carpeta `specs/`.

```markdown
# 📖 Especificaciones Técnicas (Specs) — QA Call Center

> Documentación central del sistema de Aseguramiento de Calidad (QA) bajo el enfoque de **Spec-Driven Development (SDD)**: primero se escribe la especificación, luego el plan, luego las tareas, y solo entonces se toca el código.
>
> Este directorio es la fuente de la verdad para desarrolladores y Agentes de IA que colaboran en el proyecto.

## 🏗️ Estructura del Directorio

La arquitectura de la documentación se divide en la constitución (reglas inmutables) y los features (módulos iterativos).

```text
spec/
├── constitution/            ← Reglas estables del proyecto (cambian poco)
│   ├── mission.md           ← Qué construimos (App de QA) y para quién (Auditores/Supervisores)
│   ├── tech-stack.md        ← Tecnologías (Vanilla JS, LESS) y Patrón Adaptador
│   └── roadmap.md           ← Orden de desarrollo de los features
└── features/                ← Una carpeta por módulo funcional
    └── NNN-nombre-feature/
        ├── spec.md          ← Qué hace + Modelos (DTOs) + Reglas de negocio
        ├── plan.md          ← Cómo se implementa (Lógica local vs. API REST)
        └── tasks.md         ← Checklist de tareas a nivel de código

```

## 🗺️ Mapa de Features (Roadmap)

El sistema está diseñado modularmente. Las dependencias dictan el orden estricto de desarrollo:

| ID | Feature | Descripción Breve | Dependencias |
| --- | --- | --- | --- |
| **001** | `user-access` | Autenticación, sesión híbrida y RBAC (CALIDAD, SUPERVISOR, etc.). | Ninguna |
| **002** | `admin-panel` | Orquestador visual (Dashboard). SPA con menú lateral plegable. | `001` |
| **003** | `crud-clientes` | Catálogo maestro de clientes (Entidad raíz). | `002` |
| **004** | `crud-campanas` | Gestión de campañas atadas estrictamente a un cliente. | `003` |
| **005** | `crud-users` | Identidad unificada y accesos operativos (Carga masiva CSV). | `004` |
| **006** | `crud-matriz` | Configuración de parámetros de evaluación y reglas matemáticas. | `003` |
| **007** | `evaluacion-calidad` | Formulario operativo reactivo con motor `ScoreCalculator` puro. | `005`, `006` |
| **008** | `historial-monitoreos` | Panel de lectura y filtros con aislamiento de datos automático por rol. | `007` |

## 🚀 Flujo para crear o alterar un Feature

1. **Definir:** Crear la carpeta `features/NNN-nombre-feature/` siguiendo la numeración del `roadmap.md`.
2. **Especificar (`spec.md`):** Definir interfaces TypeScript, entidades y reglas de integridad referencial. Todo debe pasar por el Patrón Adaptador (`Local` vs `Rest`).
3. **Planear (`plan.md`):** Establecer el enfoque técnico, la UI y cómo funcionará el aislamiento (`file://` offline support) versus la conexión remota.
4. **Desglosar (`tasks.md`):** Crear el checklist granular marcando el progreso.
5. **Ejecutar:** Implementar código basado *estrictamente* en las tareas documentadas.
6. **Cerrar:** Actualizar `constitution/roadmap.md` moviendo el feature a "Hecho".

> **La Constitución manda:** Si una nueva regla en un `spec.md` choca con los principios de `mission.md` o la arquitectura Vanilla/LESS de `tech-stack.md`, se replantea el feature, no la constitución.
