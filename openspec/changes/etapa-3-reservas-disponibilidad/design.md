# Design: Etapa 3 — Solicitudes de reserva, disponibilidad y no solapamiento

## Context

Las Etapas 1 y 2 están especificadas pero aún no implementadas: no existe `app/` ni `tests/`, y solo hay artefactos de planificación. Esta etapa depende de la autenticación JWT, la política de permisos (`crear_reserva` y `consultar_espacios`), la tabla `espacios`, `requiere_permiso`, SQLAlchemy y Alembic de las etapas anteriores. Ver `proposal.md` para la motivación y `specs/` para el comportamiento exigido.

Convenciones heredadas:

- Monolito modular en capas: `rutas.py`, `servicios.py`, `repositorio.py`, `modelos.py` y `esquemas.py`.
- Política única de permisos en `app/core/permisos.py`.
- Errores con el formato `detail` de FastAPI y códigos 401, 403, 409 y 422 de las etapas anteriores.
- Pruebas unitarias, de integración contra PostgreSQL y de concurrencia en `tests/`.
- Migraciones versionadas con Alembic y una base de datos de pruebas separada.

## Goals / Non-Goals

**Goals:**

- Crear solicitudes de reserva en estado `pendiente` con las reglas R1–R3 y sin duración máxima.
- Consultar disponibilidad con semántica `[inicio, fin)` y solo estados bloqueantes.
- Garantizar el no solapamiento en PostgreSQL mediante una restricción de exclusión, además de la comprobación del servicio.
- Demostrar con una prueba de concurrencia que dos solicitudes simultáneas no generan dos reservas solapadas.
- Reutilizar la política de permisos sin modificarla.

**Non-Goals:**

- Aprobación, rechazo, cancelación e historial (Etapa 4); en esta etapa solo se crea `pendiente`.
- Máxima duración de una reserva: el plan no la define y no se impone ninguna.
- Resolución de zona horaria, horario exacto distinto del propuesto, reservas recurrentes y horarios por espacio.
- Notificaciones, interfaz gráfica y modificaciones a la matriz de permisos.

## Decisions

### D1. Estructura del módulo

Se implementa `app/reservas/`:

```
app/reservas/
├── rutas.py
├── servicios.py
├── repositorio.py
├── modelos.py
└── esquemas.py
```

La creación y la consulta de disponibilidad viven en este módulo, coherente con la responsabilidad del plan ("creación de solicitudes, validación de disponibilidad, prevención de simultáneas"). Las rutas validan entrada y permisos; el servicio aplica las reglas de tiempo, espacio y solapamiento; el repositorio encapsula SQLAlchemy y las consultas de solapamiento. El router se registra en `app/main.py`.

*Alternativa considerada:* poner la disponibilidad en `app/espacios/`. Se descarta porque depende de las reservas y el plan asigna esa responsabilidad al módulo de reservas.

### D2. Modelo de reserva

Tabla `reservas`:

| Columna | Tipo | Restricciones |
|---|---|---|
| `id` | entero autoincremental | clave primaria |
| `espacio_id` | entero | clave foránea a `espacios`, obligatorio |
| `solicitante_id` | entero | clave foránea a `usuarios`, obligatorio |
| `inicio` | timestamp sin zona (provisional) | obligatorio |
| `fin` | timestamp sin zona (provisional) | obligatorio, posterior a `inicio` |
| `estado` | texto con restricción de valores | obligatorio; `pendiente`, `aprobada`, `rechazada`, `cancelada`; inicial `pendiente` |
| `motivo_rechazo` | texto | nulo en esta etapa |
| `revisor_id` | entero | clave foránea a `usuarios`, nulo en esta etapa |
| `creado_en` / `actualizado_en` | timestamp | generadas por la base de datos |

Los estados y campos siguen el modelo conceptual de la sección 3.3. En esta etapa la API solo crea `pendiente`; los demás estados existen para la Etapa 4 y para que la restricción de exclusión y la consulta de disponibilidad tengan la semántica completa desde el inicio.

*Alternativa considerada:* limitar el estado a `pendiente` en esta etapa. Se descarta porque la restricción de exclusión y la consulta de disponibilidad deben contemplar `aprobada`, `rechazada` y `cancelada` desde su creación para no rehacer el esquema en la Etapa 4.

### D3. Restricción de exclusión de PostgreSQL

La migración habilita la extensión `btree_gist` y añade:

```
EXCLUDE USING gist (
  espacio_id WITH =,
  tsrange(inicio, fin, '[)') WITH &&
) WHERE (estado IN ('pendiente', 'aprobada'))
```

La restricción impone la no superposición a nivel de base de datos para el mismo espacio y solo para estados bloqueantes. El rango es `[inicio, fin)`, de modo que una reserva que termina a las 11:00 no choca con otra que empieza a las 11:00. La extensión `btree_gist` permite combinar la igualdad sobre `espacio_id` con el solapamiento de rangos en un índice GiST.

*Alternativa considerada:* validar solo en el servicio o usar bloqueos de aplicación. Se descarta porque dos transacciones pueden leer disponibilidad al mismo tiempo y ambas insertar; el plan exige que la base de datos garantice la no superposición.

**Riesgo:** la extensión requiere privilegios suficientes en PostgreSQL. El contenedor de Docker Compose usa el usuario administrador por defecto; si se usa un usuario restringido, la migración debe ejecutarse con permisos para crear la extensión.

### D4. Reglas de tiempo en el servicio

- **Alineación:** `inicio` y `fin` deben tener minutos 00 o 30 y segundos/microsegundos en cero.
- **Duración mínima:** `fin - inicio >= 30 minutos`.
- **Duración máxima:** no se impone ninguna (decisión explícita de esta etapa).
- **Horario global:** configuración `HORARIO_APERTURA` (por defecto `07:00`) y `HORARIO_CIERRE` (por defecto `21:00`). `inicio >= apertura` y `fin <= cierre` en el mismo día.
- **Futuro:** `inicio` debe ser estrictamente posterior al momento de la solicitud.
- **Cruce de día:** se rechaza; una reserva no puede terminar en una fecha distinta de la de inicio.

El reloj se inyecta en el servicio para que las pruebas unitarias sean deterministas.

*Alternativa considerada:* derivar los bloques de 30 minutos solo de la duración. Se descarta porque el plan habla de bloques de 30 minutos y la alineación de inicio y fin es la interpretación verificable.

**Preguntas abiertas:** Q1 (zona horaria), Q2 (horario exacto), Q3 (semántica de fecha futura), Q6 (cruce de día).

### D5. Estado del espacio

- Un espacio inexistente produce 404.
- Propuesta provisional: un espacio inactivo no admite nuevas solicitudes ni consultas de disponibilidad y produce 409.
- La sección 2.6.4 (qué ocurre con reservas futuras de un espacio desactivado) no se implementa en esta etapa; las reservas existentes no se modifican al desactivar un espacio. Queda como pregunta Q5 para las Etapas 3/4.

*Alternativa considerada:* permitir reservar espacios inactivos para conservar el historial. Se descarta provisionalmente porque la desactivación indica que el espacio no está operativo; requiere confirmación.

### D6. Consulta de disponibilidad

Endpoint propuesto: `GET /espacios/{id}/disponibilidad?inicio=<ISO>&fin=<ISO>`.

La respuesta contiene `espacio_id`, `inicio`, `fin` y `disponible` (booleano). No expone datos de otras reservas ni del solicitante. La consulta aplica las mismas validaciones de alineación, duración mínima, horario y fecha futura que la creación; un rango inválido produce 422. La disponibilidad se calcula con una consulta de solapamiento sobre estados `pendiente` y `aprobada` usando `inicio < fin_consulta AND fin > inicio_consulta`.

*Alternativa considerada:* devolver la lista de intervalos libres del día. Se descarta provisionalmente por ser una superficie mayor no exigida por el plan; queda como Q4.

**Preguntas abiertas:** Q4 (forma de la respuesta), Q7 (validaciones de la consulta), Q9 (código para espacio inactivo).

### D7. Creación de la solicitud

Endpoint propuesto: `POST /reservas`, con cuerpo `{espacio_id, inicio, fin}`. La respuesta 201 incluye la reserva creada con estado `pendiente`. El `solicitante_id` se obtiene del token y se ignoran `solicitante_id`, `estado`, `motivo_rechazo` y `revisor_id` si el cliente los envía.

El servicio comprueba la existencia del espacio, el estado activo, las reglas de tiempo y el solapamiento antes de persistir. El repositorio usa una transacción; si PostgreSQL lanza una violación de exclusión (`SQLSTATE 23P01`), el servicio la traduce a 409 y revierte la transacción. La validación del servicio mejora el mensaje, pero la base de datos es la garantía final ante carreras.

*Alternativa considerada:* `/solicitudes`. Se descarta provisionalmente porque el recurso persistido es una reserva con estado y las etapas siguientes operan sobre él; queda como Q8.

### D8. Autorización

- Creación: `requiere_permiso("crear_reserva")`, disponible para Solicitante, Coordinador y Administrador.
- Disponibilidad: `requiere_permiso("consultar_espacios")`, disponible para los tres roles.
- Sin token: 401. En esta etapa ninguna operación queda denegada a un rol autenticado, por lo que no se espera un 403; la dependencia sigue aplicando la política y respondería 403 si un rol dejara de estar autorizado.

*Alternativa considerada:* comprobar roles en las rutas. Se descarta por la política central de la Etapa 1.

### D9. Concurrencia

La prueba de concurrencia usa dos hilos con sesiones de base de datos independientes y una barrera para que ambas intenten insertar el mismo espacio y rango a la vez. Se espera que una solicitud responda 201 y la otra 409, y que una consulta de conteo confirme una sola reserva. El escenario se ejecuta contra PostgreSQL real, no con SQLite, porque la restricción de exclusión es específica de PostgreSQL. Se debe evitar que ambas transacciones queden bloqueadas o que el test dependa del orden de ejecución; la restricción y el rollback determinan el resultado.

*Alternativa considerada:* simular la carrera con mocks. Se descarta porque no demuestra la garantía de la base de datos.

### D10. Pruebas y reloj

- **Unitarias:** reglas de alineación, duración mínima, ausencia de máximo, horario, fecha futura, cruce de día, solapamiento y mapeo de errores; usar un reloj inyectable.
- **Integración:** creación (201, 401, 404, 409, 422), disponibilidad (200, 401, 404, 409, 422) y consulta de estados bloqueantes con datos insertados.
- **Concurrencia:** dos solicitudes simultáneas, una sola reserva persistida.
- Cada prueba referencia el escenario o requisito que cubre.
- Las pruebas de la base de datos usan la base de pruebas de la Etapa 1 y limpian las tablas entre casos.

### D11. Prerrequisito de implementación

Las Etapas 1 y 2 deben estar implementadas antes de aplicar esta etapa. Como no hay código en el repositorio, la primera tarea verifica el prerrequisito y no reimplementa esas etapas dentro de este cambio.

## Risks / Trade-offs

- **Las Etapas 1 y 2 no están implementadas** → la primera tarea verifica la suite previa; sin ella no se puede ejecutar esta etapa.
- **Zona horaria sin definir** → se usan timestamps sin zona y se documenta Q1; cualquier cambio a timestamps con zona exige revisar la restricción y las pruebas.
- **Horario exacto sin definir** → se usa configuración con valores 07:00–21:00; Q2 debe confirmarse antes de implementar.
- **Espacio inactivo con reservas futuras (2.6.4)** → la etapa no las modifica; Q5 debe resolverse en las Etapas 3/4.
- **Privilegios de `btree_gist`** → el Docker Compose usa el usuario administrador; si cambia, la migración necesita permisos.
- **Carrera entre servicio y base de datos** → la restricción de exclusión y el mapeo de `23P01` a 409 cubren la ventana; se prueba con concurrencia real.
- **Prueba de concurrencia inestable** → usar barrera, sesiones independientes, transacciones cortas y limpieza por caso.
- **Exposición de datos de otras reservas en disponibilidad** → la respuesta se limita a un booleano y al rango consultado.

## Migration Plan

1. Confirmar que las Etapas 1 y 2 están aplicadas y sus suites pasan.
2. Crear la migración de `reservas` con `btree_gist`, la tabla, los índices y la restricción de exclusión.
3. Aplicar `alembic upgrade head` y ejecutar las pruebas de esquema y concurrencia.
4. Rollback: `alembic downgrade` elimina la tabla y la restricción; la extensión compartida puede permanecer. No hay datos productivos que conservar.

## Open Questions

**Bloqueantes antes de implementar:**

- **Q1. Zona horaria (plan 2.6.2):** el plan no define cómo interpretar fechas y horas. Propuesta provisional: timestamps sin zona tratados como hora local de la institución.
- **Q2. Horario global exacto:** el plan da 07:00–21:00 como ejemplo. Propuesta provisional: configurable con esos valores por defecto; confirmar horas y si deben ser variables de entorno.
- **Q3. Semántica de "fecha futura":** ¿basta con que el inicio sea posterior al instante actual, o se exige que el día sea posterior a hoy? Propuesta provisional: inicio estrictamente posterior al instante actual, permitiendo el mismo día.
- **Q4. Forma de la consulta de disponibilidad:** ¿un booleano para un rango, una lista de intervalos libres o una vista por día? Propuesta provisional: booleano para el rango consultado.
- **Q5. Espacios desactivados y reservas futuras (plan 2.6.4):** el plan no define si las reservas futuras se cancelan, se bloquean o se conservan. Propuesta provisional: las reservas existentes no se modifican; las nuevas solicitudes y consultas sobre espacios inactivos responden 409. Confirmar antes de implementar.
- **Q6. Reservas que cruzan de día:** el plan no lo define. Propuesta provisional: se rechazan con 422.
- **Q7. Validaciones de la consulta de disponibilidad:** ¿aplica horario, fecha futura y alineación, o solo comprueba solapamiento? Propuesta provisional: aplica las mismas reglas y responde 422 si el rango no es válido.
- **Q8. Nombre del recurso:** `/reservas` frente a `/solicitudes`. Propuesta provisional: `/reservas`.
- **Q9. Código para espacio inactivo:** 409 provisional para creación y disponibilidad; confirmar.
- **Q10. Estados creados por la API en esta etapa:** solo `pendiente`; aprobación, rechazo y cancelación pertenecen a la Etapa 4.

**Diferibles:**

- **Q11. Duración máxima:** no se define y no se implementa; si el autor la quisiera, sería un cambio funcional posterior.
- **Q12. Redondeo distinto de bloques de 30 minutos:** se rechaza; no se permiten segundos ni minutos intermedios.
- **Q13. Coordinador aprobando sus propias solicitudes (plan 2.6.3):** corresponde a la Etapa 4.
- **Q14. Historial y consulta de reservas propias:** corresponde a la Etapa 4; esta etapa solo crea y consulta disponibilidad.
- **Q15. Notificaciones al crear o rechazar:** fuera del alcance del plan.
- **Q16. Paginación o filtros de disponibilidad:** no solicitados; el booleano por rango es suficiente.
