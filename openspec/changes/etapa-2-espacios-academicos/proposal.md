# Proposal: Etapa 2 — Gestión de espacios académicos

## Why

Las reservas de la Etapa 3 necesitan un catálogo confiable de espacios. Sin un módulo que registre laboratorios, auditorios y salas de cómputo, no hay sobre qué validar disponibilidad ni a qué asociar una solicitud. Esta etapa habilita además la primera aplicación concreta de la política de permisos de la Etapa 1: solo el Administrador gestiona el catálogo y cualquier usuario autenticado puede consultarlo.

## What Changes

- Se implementa el módulo `espacios` con la separación de capas de la Etapa 1: rutas, servicios y acceso a datos.
- Se añade el modelo de espacio con identificador, nombre, tipo (`laboratorio`, `auditorio`, `sala_computo`), capacidad, ubicación y estado activo, conforme a la sección 3.3 del plan.
- Se agrega la migración de Alembic para la tabla de espacios, manteniendo el esquema reproducible ya definido en la Etapa 1.
- El Administrador puede crear, editar y desactivar espacios; los roles Solicitante y Coordinador reciben 403 en esas operaciones.
- Cualquier usuario autenticado puede consultar el listado y el detalle de los espacios; sin token la respuesta es 401.
- Se reutiliza la política central de `app/core/permisos.py`: las operaciones `gestionar_espacios` y `consultar_espacios` ya están definidas y no se modifica la matriz.
- No se implementan reservas, disponibilidad horaria ni aprobaciones; la sección 2.6.4 sobre espacios desactivados con reservas futuras queda como pregunta abierta para la Etapa 3.
- Esta etapa supone la Etapa 1 implementada (autenticación, roles, dependencia de permisos y Alembic). El repositorio aún no contiene código de aplicación; la verificación de ese prerrequisito es parte de las tareas.

## Capabilities

### New Capabilities

- `space-management`: alta, edición, consulta (listado y detalle) y desactivación de espacios académicos, con tipos y campos del modelo del plan y con control de acceso por rol.

### Modified Capabilities

- Ninguna. La política de permisos y las capacidades de identidad de la Etapa 1 no cambian sus requisitos; solo se aplican al nuevo módulo.

## Impact

- **Código nuevo**: `app/espacios/` (rutas, servicios, repositorio, modelos y esquemas), integración en `app/main.py` y una migración de Alembic.
- **API**: endpoints de creación, listado, detalle, edición y desactivación de espacios.
- **Dependencias**: las de la Etapa 1; no se añaden bibliotecas nuevas.
- **Permisos**: se reutilizan `gestionar_espacios` y `consultar_espacios` de `app/core/permisos.py`.
- **Pruebas**: unitarias de validación y servicios, e integración contra PostgreSQL para cada operación, sus códigos 200/201/401/403/404/422 y el estado activo.
- **Etapa 3**: el catálogo de espacios y su estado activo quedan disponibles para la validación de disponibilidad y la creación de reservas.
- **Preguntas abiertas**: las decisiones que el plan no define (sección 2.6.4, visibilidad de espacios inactivos, validaciones de campos, reactivación y semántica exacta de la desactivación) se documentan en `design.md`.
