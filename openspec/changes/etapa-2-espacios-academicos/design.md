# Design: Etapa 2 — Gestión de espacios académicos

## Context

La Etapa 1 está especificada pero aún no implementada: el repositorio no contiene `app/` ni `tests/`, y solo existen los artefactos de planificación del cambio `etapa-1-usuarios-autenticacion-roles`. Esta etapa depende de que la Etapa 1 aporte autenticación JWT, la política de permisos en `app/core/permisos.py`, la dependencia `requiere_permiso`, SQLAlchemy y Alembic. Ver `proposal.md` para la motivación y `specs/space-management/spec.md` para el comportamiento exigido.

Convenciones heredadas de la Etapa 1:

- Monolito modular en capas: `rutas.py`, `servicios.py`, `repositorio.py`, `modelos.py` y `esquemas.py` dentro de cada módulo.
- Política única de permisos en `app/core/permisos.py`, con las operaciones `consultar_espacios` y `gestionar_espacios` ya definidas.
- Respuestas de error con el formato `detail` de FastAPI.
- Códigos HTTP 401 para falta de autenticación, 403 para permisos insuficientes y 422 para datos inválidos.
- Pruebas unitarias en `tests/unitarias/` e integración contra PostgreSQL en `tests/integracion/`.

## Goals / Non-Goals

**Goals:**

- Implementar el catálogo de espacios académicos con las operaciones crear, editar, listar, consultar detalle, desactivar y reactivar.
- Aplicar la política de permisos existente sin modificarla: consulta para todos los roles autenticados y gestión solo para el Administrador.
- Añadir la tabla `espacios` mediante una migración de Alembic reproducible.
- Cubrir cada escenario de la especificación con pruebas automatizadas y códigos HTTP explícitos.

**Non-Goals:**

- Disponibilidad horaria, reservas, aprobaciones y el detalle operativo de la relación entre espacios desactivados y reservas futuras; la decisión se documenta en D7 y su implementación corresponde a las Etapas 3 y 4.
- Eliminación de registros, paginación, búsqueda o filtros de listado.
- Horarios de operación por espacio, notificaciones o interfaz gráfica.
- Cambios en la matriz de permisos o en las capacidades de identidad de la Etapa 1.

## Decisions

### D1. Estructura del módulo

Se implementa `app/espacios/` con la misma separación de la Etapa 1:

```
app/espacios/
├── rutas.py
├── servicios.py
├── repositorio.py
├── modelos.py
└── esquemas.py
```

Las rutas validan la entrada, aplican la dependencia de permisos y delegan en el servicio. El servicio aplica las reglas de creación, edición y desactivación y no conoce detalles HTTP. El repositorio encapsula SQLAlchemy. El router se registra en `app/main.py`.

*Alternativa considerada:* reutilizar un módulo genérico de catálogos. Se descarta porque el plan exige módulos por dominio y las etapas 3 y 4 necesitan un módulo de espacios reconocible.

### D2. Modelo de datos

Tabla `espacios`:

| Columna | Tipo | Restricciones |
|---|---|---|
| `id` | entero autoincremental | clave primaria |
| `nombre` | texto | obligatorio, no vacío |
| `tipo` | texto con restricción de valores | obligatorio; `laboratorio`, `auditorio`, `sala_computo` |
| `capacidad` | entero | obligatorio, mayor que cero |
| `ubicacion` | texto | obligatorio, no vacío |
| `activo` | booleano | obligatorio, valor inicial verdadero |

Se mantienen únicamente los campos del modelo conceptual de la sección 3.3. No se añaden marcas de tiempo porque el plan no las define para espacios.

El tipo se persiste con códigos estables en minúsculas y sin acentos; las etiquetas en español se usan en la documentación. Esta decisión es coherente con los roles de la Etapa 1.

*Alternativa considerada:* una tabla de tipos de espacio. Se descarta porque el plan enumera exactamente tres tipos y no prevé gestionarlos por separado.

### D3. Migración Alembic

Se añade una migración posterior a la migración inicial de usuarios. Crea la tabla `espacios` con su clave primaria, la restricción de valores de `tipo`, los campos obligatorios y el valor inicial de `activo`. El `downgrade` elimina la tabla. No hay datos previos que migrar.

*Alternativa considerada:* crear la tabla al arrancar con `create_all`. Se descarta por la decisión de Alembic de la Etapa 1 y para mantener un historial reproducible.

### D4. Contrato de API

| Método | Ruta | Acceso | Código esperado | Propósito |
|---|---|---|---|---|
| POST | `/espacios` | Administrador | 201 | Crear espacio |
| GET | `/espacios` | Cualquier rol autenticado | 200 | Listar espacios |
| GET | `/espacios/{id}` | Cualquier rol autenticado | 200 / 404 | Consultar detalle |
| PATCH | `/espacios/{id}` | Administrador | 200 / 404 / 422 | Editar nombre, tipo, capacidad o ubicación |
| POST | `/espacios/{id}/desactivar` | Administrador | 200 / 404 | Desactivar sin eliminar |
| POST | `/espacios/{id}/reactivar` | Administrador | 200 / 404 | Reactivar sin eliminar |

Los códigos 401, 403 y 422 se aplican según la convención de la Etapa 1. Se usa `PATCH` para la edición parcial y rutas explícitas de desactivación y reactivación para no sugerir una eliminación, coherente con la regla R6 de conservar el historial. La edición no acepta el campo `activo`; el estado solo cambia mediante desactivación o reactivación.

*Alternativa considerada:* `DELETE /espacios/{id}`. Se descarta porque el plan habla de desactivación y no de eliminación. `PUT /espacios/{id}` también se descarta porque la edición puede ser parcial. La reactivación no estaba en el plan original, pero se incorpora por decisión del autor para completar la gestión del estado.

**Decisiones resueltas:** Q4 (contrato exacto y códigos), Q5 (reactivación) y Q8 (idempotencia); el contrato es el de esta tabla.

### D5. Autorización

Se reutilizan las operaciones existentes de `app/core/permisos.py`:

- `consultar_espacios`: Solicitante, Coordinador y Administrador.
- `gestionar_espacios`: únicamente Administrador.

Las rutas usan `requiere_permiso("consultar_espacios")` o `requiere_permiso("gestionar_espacios")`. No se modifica la matriz ni se añaden operaciones nuevas. Un usuario sin token recibe 401 y uno autenticado sin permiso recibe 403.

*Alternativa considerada:* comprobar el rol directamente en las rutas. Se descarta porque duplicaría la política y contradiría la decisión D7 de la Etapa 1.

### D6. Visibilidad de espacios activos e inactivos

Decisión resuelta: el listado y el detalle devuelven tanto espacios activos como inactivos a cualquier usuario autenticado, incluyendo el campo `activo`. La desactivación no oculta el registro; solo cambia su estado.

*Alternativa considerada:* ocultar los inactivos a los roles no administrativos. Se descarta porque el plan no lo define y porque la consulta de espacios de la matriz no distingue estados. Queda resuelta como Q2.

### D7. Semántica de desactivación y reactivación

Desactivar cambia `activo` a falso y conserva el registro. Reactivar cambia `activo` a verdadero y también conserva el registro. Ambas operaciones son idempotentes: desactivar un espacio ya inactivo o reactivar un espacio ya activo responde 200 sin cambios. Ninguna elimina registros ni modifica los demás campos.

Desactivar un espacio no modifica las reservas existentes: las reservas `pendiente` o `aprobada` conservan su estado y pueden aprobarse, rechazarse o cancelarse según la Etapa 4. El único efecto sobre el uso futuro es impedir nuevas solicitudes de reserva sobre el espacio; la consulta de disponibilidad de un espacio inactivo responde 409 según la Etapa 3. No se cancela ni bloquea automáticamente ninguna reserva existente. Esta es la decisión que resuelve la sección 2.6.4 del plan.

La edición sigue sin modificar `activo`: el estado solo cambia con las operaciones explícitas de desactivación y reactivación.

**Decisiones resueltas:** Q1 (2.6.4), Q5 (reactivación), Q8 (idempotencia) y Q9 (la edición no cambia `activo`).

### D8. Validación de entrada

Se usan esquemas de Pydantic para creación y edición. Decisión resuelta:

- `nombre` obligatorio, recortado y no vacío, sin regla de unicidad.
- `tipo` obligatorio y restringido a los tres valores permitidos.
- `capacidad` entero mayor que cero.
- `ubicacion` obligatoria, recortada y no vacía.
- La edición es parcial y no acepta `activo`.

Los errores de validación se traducen a 422 mediante el manejo estándar de FastAPI.

**Decisiones resueltas:** Q3 (longitudes, rangos y formato), Q6 (sin unicidad de nombre) y Q7 (recortar espacios al inicio y al final; rechazar con 422 si el valor queda vacío).

### D9. Estrategia de pruebas

- **Unitarias:** validación de esquemas, enumeración de tipos, regla de capacidad, servicio de creación/edición/desactivación/reactivación y mapeo de errores.
- **Integración:** contra PostgreSQL y la base de datos de pruebas de la Etapa 1; creación, listado, detalle, edición, desactivación, reactivación, idempotencia de activar/desactivar, espacio inexistente, datos inválidos, falta de token (401) y roles no autorizados (403).
- Cada prueba referencia en su nombre el escenario o requisito que cubre.
- La suite de la Etapa 1 debe seguir pasando.
- La decisión de 2.6.4 se verifica documentalmente en esta etapa; su comportamiento de reservas se prueba en las Etapas 3 y 4.

### D10. Prerrequisito de implementación

La Etapa 1 debe estar implementada antes de aplicar esta etapa: estructura del proyecto, migración inicial, autenticación, dependencia `requiere_permiso`, base de datos de pruebas y suite pytest. Como el repositorio aún no contiene código, la primera tarea de implementación verifica ese prerrequisito y no reimplementa la Etapa 1 dentro de este cambio.

## Risks / Trade-offs

- **La Etapa 1 no está implementada** → la implementación de esta etapa no puede ejecutar pruebas hasta completarla; se verifica como prerrequisito en las tareas.
- **Decisiones resueltas sobre validaciones, visibilidad y estado** → las preguntas Q1–Q10 ya están cerradas; las pruebas deben cubrir la idempotencia de desactivar/reactivar y la visibilidad de inactivos.
- **La sección 2.6.4 se resuelve en este cambio** → desactivar no modifica reservas existentes y solo impide nuevas solicitudes de reserva; la disponibilidad inactiva se rige por la Etapa 3. Esta etapa no implementa reservas y debe limitarse a documentar la decisión.
- **Divergencia de permisos con la Etapa 1** → se reutiliza `app/core/permisos.py` y se prueba 401/403 en cada operación de gestión.
- **Listado sin paginación** → aceptable para el alcance académico y el plan no la exige; si el catálogo crece, se añadirá en una etapa posterior.
- **Confusión entre etiquetas y códigos de tipo** → un único enumerado y una tabla de etiquetas en la documentación.
- **Reactivación no prevista en el plan original** → se incorpora por decisión del autor con endpoint, prueba e idempotencia explícita; no modifica la matriz de permisos.

## Migration Plan

1. Confirmar que la Etapa 1 está aplicada y que su migración inicial está en la base de datos.
2. Añadir la migración de `espacios` con `alembic revision` y aplicarla con `alembic upgrade head`.
3. Implementar el módulo y ejecutar la suite completa.
4. Ante un fallo del esquema, `alembic downgrade` a la revisión anterior. No hay datos que conservar en esta etapa. No aplica despliegue en producción.

## Decisiones resueltas

Todas las preguntas de diseño quedan cerradas. Estas decisiones son la base de las especificaciones y las tareas.

- **Q1. Espacios desactivados con reservas futuras (plan 2.6.4):** desactivar no modifica las reservas existentes; conservan su estado y se rigen por la Etapa 4. El único efecto sobre el uso futuro es impedir nuevas solicitudes de reserva; la consulta de disponibilidad del espacio inactivo responde 409 según la Etapa 3. No se cancelan ni bloquean automáticamente reservas existentes.
- **Q2. Visibilidad de espacios inactivos:** el listado y el detalle incluyen espacios activos e inactivos para todos los roles autenticados, con el campo `activo`.
- **Q3. Validaciones de campos:** `nombre` y `ubicacion` son obligatorios, se recortan y no pueden quedar vacíos; `capacidad` es un entero mayor que cero; `tipo` es uno de los tres valores permitidos. No se imponen longitudes ni formatos adicionales.
- **Q4. Contrato de API y códigos:** se acepta la tabla de D4, con 201 para creación, 200 para consultas y cambios de estado, 404 para recurso inexistente, 401 sin token, 403 sin permiso y 422 para datos inválidos.
- **Q5. Reactivación:** se incluye. El Administrador puede reactivar un espacio inactivo mediante `POST /espacios/{id}/reactivar`, con 200, 404, 401 y 403.
- **Q6. Unicidad de nombre:** no se exige; pueden existir espacios con el mismo nombre.
- **Q7. Normalización de espacios en blanco:** se recortan `nombre` y `ubicacion`; si el valor recortado queda vacío, se responde 422.
- **Q8. Idempotencia de estado:** desactivar un espacio ya inactivo o reactivar uno ya activo responde 200 sin cambios.
- **Q9. Edición del estado activo:** la edición no acepta ni modifica `activo`; solo desactivar y reactivar cambian el estado.
- **Q10. Marcas de tiempo y auditoría:** el plan no las define para espacios; no se añaden.
- **Paginación, búsqueda y filtros:** no se implementan; se dejan para una etapa posterior si el catálogo lo requiere.
