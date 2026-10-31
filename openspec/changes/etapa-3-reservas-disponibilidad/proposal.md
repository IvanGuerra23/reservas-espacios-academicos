# Proposal: Etapa 3 — Solicitudes de reserva, disponibilidad y no solapamiento

## Why

La Etapa 3 es el núcleo del sistema: resuelve los conflictos de horario y la duplicidad de solicitudes que hoy se gestionan por correo y hojas de cálculo. Sin creación de solicitudes, consulta de disponibilidad y una garantía real de no solapamiento, el catálogo de espacios de la Etapa 2 no aporta valor operativo. La no superposición debe resistir condiciones de carrera, por lo que la base de datos debe imponerla además del servicio.

## What Changes

- Se implementa el módulo `reservas` con la separación de capas de las etapas anteriores: rutas, servicios, repositorio, modelos y esquemas.
- Se añade la tabla `reservas` con espacio, solicitante, inicio, fin, estado, motivo de rechazo, revisor y fechas de creación y actualización, conforme a la sección 3.3 del plan.
- Cualquier usuario autenticado (Solicitante, Coordinador o Administrador) puede crear una solicitud de reserva; la solicitud nace en estado `pendiente` y el solicitante se toma del token.
- Se aplican las reglas R1–R3: solo `pendiente` y `aprobada` bloquean; las reservas se alinean a bloques de 30 minutos con duración mínima de 30 minutos y sin duración máxima; rige un horario global de operación y solo se permite reservar en fechas futuras.
- Se implementa la consulta de disponibilidad para un espacio y un rango, con intervalos `[inicio, fin)` y estados bloqueantes.
- Se añade una restricción de exclusión de PostgreSQL sobre el rango de cada reserva por espacio, aplicada solo a `pendiente` y `aprobada`, junto con la validación en el servicio y la traducción de conflictos a 409.
- Se añade una prueba de concurrencia que lanza dos solicitudes simultáneas sobre el mismo espacio y rango y demuestra que solo una se persiste.
- No se implementan aprobación, rechazo, cancelación ni historial: pertenecen a la Etapa 4. Tampoco se define duración máxima ni se resuelve la zona horaria, que queda como pregunta abierta.
- Esta etapa supone las Etapas 1 y 2 implementadas (autenticación, permisos, espacios y Alembic). El repositorio aún no contiene código de aplicación; la verificación de ese prerrequisito es parte de las tareas.

## Capabilities

### New Capabilities

- `reservation-requests`: creación de una solicitud de reserva en estado pendiente, con solicitante tomado del token y validaciones de bloques de 30 minutos, duración mínima, horario global, fecha futura, espacio existente y activo.
- `space-availability`: consulta de disponibilidad de un espacio para un rango, considerando únicamente las reservas en estados que bloquean el horario.
- `overlap-prevention`: garantía de no solapamiento para reservas pendientes y aprobadas del mismo espacio, impuesta por una restricción de exclusión de PostgreSQL y comprobada también en el servicio, con prueba de concurrencia.

### Modified Capabilities

- Ninguna. La política de permisos de la Etapa 1 ya define `crear_reserva` y `consultar_espacios`; no cambian sus requisitos.

## Impact

- **Código nuevo**: `app/reservas/` (rutas, servicios, repositorio, modelos y esquemas), integración en `app/main.py` y una migración de Alembic con `btree_gist` y la restricción de exclusión.
- **API**: endpoint de creación de solicitudes y endpoint de consulta de disponibilidad.
- **Base de datos**: nueva tabla `reservas`, índices y restricción de exclusión sobre rangos `[inicio, fin)` por espacio y estado.
- **Permisos**: se reutilizan `crear_reserva` y `consultar_espacios` de `app/core/permisos.py`.
- **Pruebas**: unitarias de reglas de tiempo, integración de creación/disponibilidad y una prueba de concurrencia con dos solicitudes simultáneas.
- **Etapa 4**: los estados y la restricción de exclusión quedan preparados para aprobación, rechazo y cancelación.
- **Preguntas abiertas**: zona horaria (sección 2.6.2), horario global exacto, semántica de "fecha futura", espacios desactivados y reservas futuras (sección 2.6.4), forma de la consulta de disponibilidad y rango entre días; se documentan en `design.md`.
