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

- Implementar el catálogo de espacios académicos con las operaciones crear, editar, listar, consultar detalle y desactivar.
- Aplicar la política de permisos existente sin modificarla: consulta para todos los roles autenticados y gestión solo para el Administrador.
- Añadir la tabla `espacios` mediante una migración de Alembic reproducible.
- Cubrir cada escenario de la especificación con pruebas automatizadas y códigos HTTP explícitos.

**Non-Goals:**

- Disponibilidad horaria, reservas, aprobaciones y la relación entre espacios desactivados y reservas futuras (sección 2.6.4 del plan).
- Reactivación de espacios, eliminación de registros, paginación, búsqueda o filtros de listado.
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
| `capacidad` | entero | obligatorio, mayor que cero (provisional) |
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

Los códigos 401, 403 y 422 se aplican según la convención de la Etapa 1. Se usa `PATCH` para la edición parcial y una ruta explícita de desactivación para no sugerir una eliminación, coherente con la regla R6 de conservar el historial. La edición no acepta el campo `activo`.

*Alternativa considerada:* `DELETE /espacios/{id}`. Se descarta porque el plan habla de desactivación y no de eliminación. `PUT /espacios/{id}` también se descarta porque la edición puede ser parcial.

**Preguntas abiertas:** Q4 (contrato exacto y códigos), Q5 (reactivación), Q8 (desactivar un espacio ya inactivo).

### D5. Autorización

Se reutilizan las operaciones existentes de `app/core/permisos.py`:

- `consultar_espacios`: Solicitante, Coordinador y Administrador.
- `gestionar_espacios`: únicamente Administrador.

Las rutas usan `requiere_permiso("consultar_espacios")` o `requiere_permiso("gestionar_espacios")`. No se modifica la matriz ni se añaden operaciones nuevas. Un usuario sin token recibe 401 y uno autenticado sin permiso recibe 403.

*Alternativa considerada:* comprobar el rol directamente en las rutas. Se descarta porque duplicaría la política y contradiría la decisión D7 de la Etapa 1.

### D6. Visibilidad de espacios activos e inactivos

Propuesta provisional: el listado y el detalle devuelven tanto espacios activos como inactivos a cualquier usuario autenticado, incluyendo el campo `activo`. La desactivación no oculta el registro; solo cambia su estado.

*Alternativa considerada:* ocultar los inactivos a los roles no administrativos. Se descarta provisionalmente porque el plan no lo define y porque la consulta de espacios de la matriz no distingue estados. Queda como pregunta Q2.

### D7. Semántica de la desactivación

Desactivar cambia `activo` a falso y conserva el registro. No elimina reservas ni modifica datos históricos. El efecto sobre reservas futuras del espacio desactivado no se implementa en esta etapa: la sección 2.6.4 del plan lo deja abierto y corresponde a las Etapas 3 y 4. No se añade reactivación porque no está solicitada ni definida en el plan.

**Preguntas abiertas:** Q1 (2.6.4), Q5 (reactivación), Q8 (idempotencia).

### D8. Validación de entrada

Se usan esquemas de Pydantic para creación y edición. Propuesta provisional:

- `nombre` obligatorio y no vacío, sin regla de unicidad.
- `tipo` obligatorio y restringido a los tres valores permitidos.
- `capacidad` entero mayor que cero.
- `ubicacion` obligatoria y no vacía.
- La edición es parcial y no acepta `activo`.

Los errores de validación se traducen a 422 mediante el manejo estándar de FastAPI.

**Preguntas abiertas:** Q3 (longitudes, rangos, formato de ubicación), Q6 (unicidad de nombre), Q7 (normalización de espacios en blanco).

### D9. Estrategia de pruebas

- **Unitarias:** validación de esquemas, enumeración de tipos, regla de capacidad, servicio de creación/edición/desactivación y mapeo de errores.
- **Integración:** contra PostgreSQL y la base de datos de pruebas de la Etapa 1; creación, listado, detalle, edición, desactivación, espacio inexistente, datos inválidos, falta de token (401) y roles no autorizados (403).
- Cada prueba referencia en su nombre el escenario o requisito que cubre.
- La suite de la Etapa 1 debe seguir pasando.

### D10. Prerrequisito de implementación

La Etapa 1 debe estar implementada antes de aplicar esta etapa: estructura del proyecto, migración inicial, autenticación, dependencia `requiere_permiso`, base de datos de pruebas y suite pytest. Como el repositorio aún no contiene código, la primera tarea de implementación verifica ese prerrequisito y no reimplementa la Etapa 1 dentro de este cambio.

## Risks / Trade-offs

- **La Etapa 1 no está implementada** → la implementación de esta etapa no puede ejecutar pruebas hasta completarla; se verifica como prerrequisito en las tareas.
- **Decisiones provisionales sobre validaciones y visibilidad** → se documentan como preguntas Q1–Q10 y se confirman antes de implementar; las pruebas se ajustan a la decisión.
- **La sección 2.6.4 queda sin resolver** → la Etapa 2 solo cambia el estado del espacio; las Etapas 3 y 4 deben definir el efecto sobre reservas futuras antes de implementarlo.
- **Divergencia de permisos con la Etapa 1** → se reutiliza `app/core/permisos.py` y se prueba 401/403 en cada operación de gestión.
- **Listado sin paginación** → aceptable para el alcance académico y el plan no la exige; si el catálogo crece, se añadirá en una etapa posterior.
- **Confusión entre etiquetas y códigos de tipo** → un único enumerado y una tabla de etiquetas en la documentación.

## Migration Plan

1. Confirmar que la Etapa 1 está aplicada y que su migración inicial está en la base de datos.
2. Añadir la migración de `espacios` con `alembic revision` y aplicarla con `alembic upgrade head`.
3. Implementar el módulo y ejecutar la suite completa.
4. Ante un fallo del esquema, `alembic downgrade` a la revisión anterior. No hay datos que conservar en esta etapa. No aplica despliegue en producción.

## Open Questions

Las preguntas que afectan el comportamiento observable o la implementación deben confirmarse antes de aplicar el cambio. La columna "propuesta provisional" indica lo que asumen las especificaciones y las tareas actuales.

**Bloqueantes antes de implementar:**

- **Q1. Espacios desactivados con reservas futuras (plan 2.6.4):** el plan no define qué ocurre con reservas futuras de un espacio desactivado ni si se deben cancelar o bloquear. Etapa 2 solo cambia `activo`; la decisión corresponde a las Etapas 3 y 4.
- **Q2. Visibilidad de espacios inactivos:** ¿los roles no administrativos deben ver los espacios inactivos en listado y detalle? Propuesta provisional: sí, con el campo `activo`.
- **Q3. Validaciones de campos:** el plan no define longitud de nombre o ubicación, rangos de capacidad ni formato de ubicación. Propuesta provisional: nombre y ubicación no vacíos, capacidad entera mayor que cero.
- **Q4. Contrato de API y códigos:** el plan no define rutas ni códigos de éxito/404. Propuesta provisional: la tabla de D4 con 201, 200 y 404.
- **Q5. Reactivación:** el plan solo pide desactivar. ¿Se necesita reactivar un espacio? No se incluye en esta etapa.
- **Q6. Unicidad de nombre:** ¿pueden existir dos espacios con el mismo nombre? Propuesta provisional: no se exige unicidad.
- **Q8. Desactivar un espacio ya inactivo:** ¿responde 200 idempotente o 409? Propuesta provisional: 200 sin cambios.

**Diferibles:**

- **Q7. Normalización de espacios en blanco:** recortar o rechazar nombres y ubicaciones con espacios al inicio o al final.
- **Q9. Edición del estado activo:** la edición no cambia `activo`; la desactivación es la única operación de estado en esta etapa.
- **Q10. Marcas de tiempo y auditoría:** el plan no las define para espacios; no se añaden.
- **Paginación, búsqueda y filtros de listado:** no solicitados; se dejan para una etapa posterior si el catálogo lo requiere.
