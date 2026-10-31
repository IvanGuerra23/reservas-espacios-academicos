# Design: Etapa 4 — Aprobación, rechazo, cancelación e historial

## Context

Las Etapas 1 a 3 están especificadas pero no implementadas: no existe `app/` ni `tests/`. Esta etapa depende de la autenticación JWT, la política central de permisos, el modelo `Reserva` de la Etapa 3 y su restricción de exclusión. La tabla ya tiene `estado`, `motivo_rechazo`, `revisor_id`, `creado_en` y `actualizado_en`; por tanto, el ciclo de vida se implementa sin cambiar el esquema. Ver `proposal.md` para la motivación y `specs/` para el comportamiento exigido.

Restricciones heredadas:

- Monolito modular en capas, con reglas de negocio en servicios.
- La matriz 2.5 es la única política de autorización, en `app/core/permisos.py`.
- Los estados existentes son `pendiente`, `aprobada`, `rechazada` y `cancelada`.
- Solo `pendiente` y `aprobada` bloquean el horario y participan en la restricción de exclusión.
- Reloj inyectable y timestamps sin zona provisionales de la Etapa 3.
- Errores con `detail` de FastAPI y códigos 401, 403, 404, 409 y 422.

## Goals / Non-Goals

**Goals:**

- Aprobar y rechazar solicitudes pendientes con revisor, motivo obligatorio y transición válida.
- Cancelar reservas propias o ajenas según R5 y la matriz 2.5.
- Consultar historial propio y global con todos los estados y sin eliminación física.
- Garantizar transiciones válidas incluso con dos revisores concurrentes.
- Reutilizar la restricción de exclusión: rechazar o cancelar libera el horario sin borrar la fila.

**Non-Goals:**

- Nueva tabla, columna o migración de esquema.
- Auditoría de cada transición: el historial muestra el estado actual y los datos de revisión, no una bitácora de cambios.
- Edición de fechas, espacio o solicitante de una reserva.
- Notificaciones, eliminación, restauración de reservas o reactivación de espacios.
- Paginación, filtros o búsqueda del historial; se documentan como preguntas abiertas.

## Decisions

### D1. Límites de módulos

- `app/aprobaciones/` implementa aprobación y rechazo, conforme a la responsabilidad del módulo en la sección 2.3 del plan.
- `app/reservas/` se amplía con cancelación e historial, también conforme a la sección 2.3.
- La lógica pura de transiciones vive en `app/reservas/transiciones.py` y es importada por ambos módulos.
- `aprobaciones` depende de `reservas` (modelo, repositorio y transiciones); `reservas` no importa `aprobaciones`, lo que evita ciclos.

*Alternativa considerada:* poner todo en `reservas`. Se descarta porque el plan asigna explícitamente las aprobaciones a su propio módulo y la separación mantiene la trazabilidad módulo → requisito.

### D2. Máquina de estados

La función pura `validar_transicion(estado_actual, accion)` acepta solo:

| Estado actual | Acción | Estado resultante |
|---|---|---|
| `pendiente` | aprobar | `aprobada` |
| `pendiente` | rechazar | `rechazada` |
| `pendiente` | cancelar | `cancelada` |
| `aprobada` | cancelar | `cancelada` |

`rechazada` y `cancelada` son terminales. Cualquier otra combinación, incluidas las repeticiones sobre el mismo estado, produce 409 y no modifica la fila. La función no accede a la base de datos ni conoce HTTP; los servicios traducen el resultado a error de dominio y luego a 409.

*Alternativa considerada:* condiciones `if` dispersas en cada servicio. Se descarta porque la matriz de transiciones quedaría duplicada y divergiría entre aprobación y cancelación.

### D3. Aprobar y rechazar

Endpoints:

| Método | Ruta | Acceso | Cuerpo |
|---|---|---|---|
| POST | `/reservas/{id}/aprobar` | Coordinador, Administrador | ninguno |
| POST | `/reservas/{id}/rechazar` | Coordinador, Administrador | `{ "motivo": "texto" }` |

Ambos exigen `requiere_permiso("aprobar_rechazar")`. La ruta autentica, valida el cuerpo y delega; el servicio bloquea la fila con `SELECT ... FOR UPDATE`, comprueba que el estado sea `pendiente`, aplica la transición y registra `revisor_id` desde el token.

- Aprobar deja `motivo_rechazo` nulo.
- Rechazar exige un motivo que, tras eliminar espacios al inicio y al final, no quede vacío; el motivo se guarda recortado. La ausencia, el vacío o solo espacios producen 422.
- No se impone longitud máxima al motivo porque el plan no la define.
- La aprobación no puede violar la restricción de exclusión: la solicitud ya estaba `pendiente` y bloqueaba el mismo rango. Cambiar el estado no crea un rango nuevo.
- No se comprueba el estado activo del espacio al aprobar o rechazar; queda como Q4.
- No se añade una restricción de autoaprobación; la matriz autoriza `aprobar_rechazar` a Coordinador y Administrador. La sección 2.6.3 queda como Q1.

Respuestas: 200 con la reserva actualizada, 401 sin token, 403 para Solicitante, 404 si la reserva no existe y 409 si la transición no es válida.

*Alternativa considerada:* `PATCH /reservas/{id}` con `{estado, motivo}`. Se descarta porque permitiría estados arbitrarios y no expresaría la operación de negocio ni su autorización por separado.

### D4. Cancelación

Endpoint: `POST /reservas/{id}/cancelar`, sin cuerpo. La ruta exige `requiere_permiso("cancelar_reserva_propia")`; después de cargar la reserva, el servicio determina la propiedad:

- Si el usuario es el solicitante, aplica la operación propia. El Solicitante solo puede cancelar si `inicio > ahora`; Coordinador y Administrador pueden cancelar sus propias reservas sin esa restricción.
- Si el usuario no es el solicitante, exige `cancelar_reserva_ajena` mediante la política central. El Solicitante recibe 403; Coordinador y Administrador pueden cancelar cualquier reserva `pendiente` o `aprobada`.
- Si la reserva no existe, 404.
- Si el estado no es `pendiente` ni `aprobada`, 409.
- Si el Solicitante intenta cancelar una reserva propia ya comenzada, 409.
- La cancelación no pide motivo y no modifica `revisor_id`; no existe una columna para registrar quién cancela (Q5).
- Coordinador y Administrador pueden cancelar reservas ya comenzadas o de espacios inactivos de forma provisional (Q2 y Q8).

El servicio bloquea la fila y valida la transición en la misma transacción que actualiza el estado, para que una cancelación concurrente con una aprobación no produzca una transición perdida.

*Alternativa considerada:* endpoints separados para cancelación propia y ajena. Se descarta porque el recurso es el mismo y la diferencia depende de la propiedad, no de una ruta distinta; la política se aplica igualmente.

### D5. Historial

Endpoints:

| Método | Ruta | Acceso | Resultado |
|---|---|---|---|
| GET | `/reservas/mias` | Solicitante, Coordinador, Administrador | reservas donde `solicitante_id` es el usuario autenticado |
| GET | `/reservas` | Coordinador, Administrador | todas las reservas |

Ambos exigen autenticación y la operación correspondiente de la política (`ver_historial_propio` o `ver_historial_global`). Incluyen todos los estados y no filtran por fecha ni por estado. La respuesta contiene los campos del modelo de reserva y referencias legibles al espacio, al solicitante y al revisor; no expone contraseñas ni datos de autenticación.

Propuesta provisional: ordenar por `inicio` descendente y luego por `id` descendente, sin paginación ni filtros (Q6). El historial incluye reservas de espacios inactivos (Q7). La ruta `/reservas/mias` se declara antes de cualquier ruta con parámetro para evitar colisiones.

*Alternativa considerada:* un único endpoint con parámetro `alcance=mio|global`. Se descarta porque la autorización de cada alcance es distinta y las rutas separadas la hacen explícita.

### D6. Conservación y ausencia de borrado

Rechazar y cancelar solo ejecutan `UPDATE` sobre `estado`, `motivo_rechazo`, `revisor_id` y `actualizado_en`. No se implementa ninguna ruta de borrado: `DELETE /reservas/{id}` no coincide con una ruta existente y responde 404, mientras que `DELETE /reservas` responde 405 porque la colección ya existe para `GET` y `POST`. En ambos casos la fila permanece. No se usan borrados en cascada sobre reservas.

*Alternativa considerada:* eliminación lógica con una columna `eliminada`. Se descarta porque el plan pide conservar el historial, no ocultar registros, y no define esa columna.

### D7. Concurrencia de transiciones

Aprobar, rechazar y cancelar bloquean la reserva con `SELECT ... FOR UPDATE`. Dos aprobaciones simultáneas se serializan: la primera responde 200 y la segunda encuentra el estado `aprobada` y responde 409. Una aprobación y una cancelación simultáneas también se serializan: según el orden, puede resultar aprobada y luego cancelada (ambas 200, transiciones válidas en secuencia) o cancelada y luego 409 para la aprobación. La prueba de concurrencia cubre el caso de doble aprobación, que es el más directo para demostrar que no hay doble revisor ni doble actualización.

*Alternativa considerada:* comparar y actualizar sin bloqueo (`UPDATE ... WHERE estado = 'pendiente'`). Es correcto para una transición, pero el bloqueo permite devolver el estado resultante y aplicar la misma validación central sin duplicar condiciones; se usará `UPDATE` condicional como defensa adicional si el repositorio lo permite.

### D8. Errores y permisos

- 401: token ausente, inválido o expirado.
- 403: Solicitante en aprobación/rechazo, cancelación ajena o historial global.
- 404: reserva inexistente.
- 409: transición no permitida, reserva terminal, cancelación propia ya comenzada o conflicto de concurrencia.
- 422: motivo de rechazo ausente o vacío; cuerpo inválido.
- 404 o 405: intento de eliminación; 404 para el recurso y 405 para la colección, sin borrar datos.
- La política central no cambia; las cinco operaciones ya existen en `app/core/permisos.py`.
- La comprobación de propiedad se hace en el servicio, como indica la Etapa 1 para operaciones propias/ajenas.

### D9. Pruebas

- **Unitarias:** matriz completa de transiciones, validación del motivo, regla de cancelación propia antes del inicio, autorización por rol y permisos propia/ajena.
- **Integración:** aprobar (200, 401, 403, 404, 409), rechazar (200, 401, 403, 404, 409, 422), cancelar (200, 401, 403, 404, 409), historial propio/global (200, 401, 403), liberación del horario y conservación de filas.
- **Concurrencia:** dos aprobaciones simultáneas producen un 200, un 409 y un solo revisor.
- **Contrato:** `DELETE /reservas/{id}` responde 404 y `DELETE /reservas` responde 405, sin eliminar datos.
- Las pruebas usan PostgreSQL y el reloj inyectable de la Etapa 3; cada una referencia el escenario que cubre.

### D10. Prerrequisito

Las Etapas 1 a 3 deben estar implementadas. La primera tarea verifica la suite previa; este cambio no reimplementa autenticación, espacios, creación, disponibilidad ni la restricción de exclusión.

## Risks / Trade-offs

- **Autoaprobación de Coordinador (2.6.3)** → se permite provisionalmente porque la matriz no la prohíbe; Q1 debe confirmarse antes de implementar.
- **Cancelación de reservas ya comenzadas por Coordinador/Administrador** → se permite provisionalmente por la lectura literal de R5; Q2.
- **Aprobación de solicitudes cuyo inicio ya pasó** → se permite provisionalmente porque el ciclo de vida solo exige estado `pendiente`; Q3.
- **Espacios inactivos con reservas futuras (2.6.4)** → no se modifica su comportamiento; Q4 y Q8.
- **No se registra quién cancela** → el modelo del plan solo tiene `revisor`; Q5.
- **Historial sin paginación** → puede crecer; Q6. Se puede añadir paginación en un cambio posterior.
- **Zona horaria heredada** → la comparación `inicio > ahora` depende de la Q1 de la Etapa 3; se usa el mismo reloj y la misma interpretación.
- **Bloqueos de fila** → transacciones cortas y una sola fila reducen el riesgo de espera; las pruebas de concurrencia lo verifican.
- **Ruta `/reservas` global y `/reservas/mias`** → declarar la ruta literal antes de las rutas con parámetros y cubrirlo con una prueba de contrato.

## Migration Plan

1. Confirmar que las Etapas 1 a 3 están implementadas y sus suites pasan.
2. Implementar servicios, rutas y pruebas sin migración de esquema.
3. Ejecutar `alembic upgrade head` para confirmar que no hay cambios pendientes y correr la suite completa.
4. Rollback: revertir el código de la etapa; las reservas y sus estados permanecen en la base de datos. No se requiere `downgrade`.

## Open Questions

- **Q1. Autoaprobación de Coordinador (plan 2.6.3):** ¿puede aprobar o rechazar una solicitud propia? Propuesta provisional: sí, porque la matriz 2.5 autoriza `aprobar_rechazar` sin excepción de propiedad.
- **Q2. Cancelación de reservas ya comenzadas por Coordinador o Administrador:** R5 dice que pueden cancelar cualquiera, pero no aclara el límite temporal. Propuesta provisional: pueden cancelar reservas `pendiente` o `aprobada` aunque ya hayan comenzado.
- **Q3. Aprobación o rechazo de solicitudes cuyo inicio ya pasó:** el ciclo de vida no impone límite temporal. Propuesta provisional: se permite mientras el estado sea `pendiente`.
- **Q4. Espacios inactivos y reservas futuras (plan 2.6.4):** ¿se bloquea la aprobación, se cancela automáticamente o se conserva? Propuesta provisional: la desactivación no altera las reservas y la aprobación/rechazo sigue su curso; hereda la Q5 de la Etapa 3.
- **Q5. Registro de quién cancela:** el modelo solo tiene `revisor_id`. Propuesta provisional: no se añade una columna; la cancelación no registra actor. Confirmar si se requiere trazabilidad adicional.
- **Q6. Orden, filtros y paginación del historial:** el plan no los define. Propuesta provisional: lista completa ordenada por `inicio` descendente, sin filtros ni paginación.
- **Q7. Espacios inactivos en el historial:** propuesta provisional: se incluyen las reservas históricas aunque el espacio esté inactivo.
- **Q8. Cancelación de reservas de espacios inactivos:** propuesta provisional: se permite cancelar reservas `pendiente` o `aprobada` de un espacio inactivo.
- **Q9. Código para un borrado no soportado:** propuesta provisional: 404 para `DELETE /reservas/{id}` y 405 para `DELETE /reservas`; confirmar si se prefiere otro contrato.
