# Proposal: Etapa 4 — Aprobación, rechazo, cancelación e historial

## Why

La Etapa 3 permite crear solicitudes pendientes y evita solapamientos, pero todavía no existe forma de revisarlas ni de cerrar su ciclo de vida. Sin aprobación, rechazo, cancelación e historial, el sistema no puede sustituir el proceso actual: las solicitudes quedarían pendientes para siempre y no habría trazabilidad de lo ocurrido. Esta etapa completa el ciclo de vida de la sección 2.2.1 y las reglas R4–R6.

## What Changes

- Se implementa el módulo `aprobaciones` para que Coordinador y Administrador aprueben o rechacen solicitudes pendientes.
- El rechazo exige un motivo no vacío (R4); la aprobación y el rechazo registran al usuario revisor y conservan el motivo cuando corresponde.
- Se implementa la cancelación de reservas pendientes o aprobadas (R5): el solicitante puede cancelar las propias mientras no hayan comenzado; Coordinador y Administrador pueden cancelar cualquiera.
- Se implementa el historial propio para los tres roles y el historial global para Coordinador y Administrador, incluyendo reservas en todos los estados.
- Se aplican las transiciones permitidas del ciclo de vida: `pendiente` a `aprobada`, `pendiente` a `rechazada`, `pendiente` a `cancelada` y `aprobada` a `cancelada`; `rechazada` y `cancelada` son terminales. Toda transición no permitida responde 409.
- Se garantiza que nada se elimina (R6): aprobar, rechazar y cancelar solo cambian el estado; no se expone ni se implementa eliminación de reservas.
- Se reutiliza la política central de permisos (`aprobar_rechazar`, `cancelar_reserva_propia`, `cancelar_reserva_ajena`, `ver_historial_propio`, `ver_historial_global`) y los códigos 401, 403, 404, 409 y 422 de las etapas anteriores.
- No se añaden notificaciones, reservas recurrentes, edición de reservas ni eliminación física. La posibilidad de que un Coordinador apruebe o rechace sus propias solicitudes (sección 2.6.3) queda como pregunta abierta, junto con los demás puntos que el plan no define.
- Esta etapa supone las Etapas 1 a 3 implementadas: autenticación, permisos, espacios, creación de reservas, disponibilidad y restricción de exclusión. El repositorio aún no contiene código de aplicación; la verificación de ese prerrequisito forma parte de las tareas.

## Capabilities

### New Capabilities

- `reservation-workflow`: aprobación y rechazo de solicitudes pendientes, cancelación de reservas propias y ajenas, transiciones de estado permitidas y liberación del horario al rechazar o cancelar.
- `reservation-history`: consulta del historial propio y global en todos los estados, con conservación permanente de rechazos y cancelaciones y sin eliminación de reservas.

### Modified Capabilities

- Ninguna. La creación, la disponibilidad y la prevención de solapamientos de la Etapa 3 no cambian sus requisitos; las transiciones se apoyan en los estados y en la restricción de exclusión ya definidos.

## Impact

- **Código nuevo**: `app/aprobaciones/` para aprobación y rechazo; ampliación de `app/reservas/` para cancelación e historial; registro de routers en `app/main.py`.
- **API**: endpoints para aprobar, rechazar y cancelar reservas, y para consultar el historial propio y global.
- **Base de datos**: se reutilizan las columnas `estado`, `motivo_rechazo`, `revisor_id` y `actualizado_en` creadas en la Etapa 3; no se requiere una migración de esquema, salvo índices de consulta que se justifiquen en el diseño.
- **Permisos**: se reutilizan las cinco operaciones ya definidas en `app/core/permisos.py`; no cambia la matriz.
- **Pruebas**: unitarias de transiciones y reglas de cancelación, integración de cada operación y sus códigos HTTP, verificación de que rechazar/cancelar libera el horario y de que las filas permanecen.
- **Dependencias**: Etapas 1 a 3 implementadas; reloj inyectable y zona horaria ya definidos en la Etapa 3.
- **Preguntas abiertas**: autoaprobación de Coordinador (sección 2.6.3), cancelación de reservas ya comenzadas por parte de Coordinador/Administrador, aprobación de solicitudes cuyo inicio ya pasó, espacios inactivos con reservas futuras (sección 2.6.4), orden y paginación del historial; se documentan en `design.md`.
