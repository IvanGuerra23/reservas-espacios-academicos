# Design: Etapa 1 — Andamiaje, usuarios, autenticación y roles

## Context

El repositorio contiene únicamente el plan de ingeniería y la configuración de OpenSpec; no existe código de aplicación ni especificaciones principales previas. Esta etapa crea el primer código del sistema y, por tanto, fija convenciones que las etapas 2 a 4 reutilizarán. Ver `proposal.md` para la motivación y `specs/` para el comportamiento exigido.

Restricciones del plan (`PLAN_INGENIERIA.md`, secciones 3 y 4):

- Monolito modular en capas: rutas, servicios y acceso a datos por módulo.
- FastAPI, SQLAlchemy, PostgreSQL en Docker Compose, JWT y pytest.
- Las reglas de negocio viven en la capa de servicios.
- La matriz de permisos de la sección 2.5 es la fuente de autorización.
- Solo API, sin interfaz gráfica; sin notificaciones, renovación de tokens ni recuperación de contraseña.

## Goals / Non-Goals

**Goals:**

- Dejar el proyecto ejecutable con un comando y con el esquema de base de datos reproducible.
- Fijar la estructura modular y las convenciones de capas para las etapas siguientes.
- Implementar el ciclo de identidad completo: registro, hash de contraseña, login JWT, consulta de la propia cuenta y administración por el Administrador.
- Implementar los tres roles y una política de permisos única, derivada de la matriz 2.5, reutilizable por las etapas posteriores.
- Cubrir cada escenario de aceptación con pruebas automatizadas y dejar la carpeta de concurrencia creada.

**Non-Goals:**

- Endpoints de espacios, reservas y aprobaciones; su comportamiento y autorización quedan definidos en la política, pero se implementan en las Etapas 2 a 4.
- Restricción de exclusión de PostgreSQL y pruebas de concurrencia de reservas (Etapa 3).
- Interfaz gráfica, correo electrónico, reservas recurrentes, recuperación de contraseña, renovación o revocación explícita de tokens y horarios por espacio.
- Un sistema genérico de administración de permisos editable en tiempo de ejecución; la matriz es una política versionada en código.

## Decisions

### D1. Estructura del proyecto y convención de capas

Se adopta la estructura del plan, con archivos de capa dentro de cada módulo:

```
app/
├── main.py
├── core/
│   ├── configuracion.py
│   ├── base_datos.py
│   ├── seguridad.py
│   ├── permisos.py
│   └── dependencias.py
├── usuarios/
│   ├── rutas.py
│   ├── servicios.py
│   ├── repositorio.py
│   ├── modelos.py
│   └── esquemas.py
├── espacios/        # solo __init__.py en esta etapa
├── reservas/        # solo __init__.py en esta etapa
└── aprobaciones/    # solo __init__.py en esta etapa
tests/
├── unitarias/
├── integracion/
└── concurrencia/
```

`core` concentra configuración, sesión de base de datos, hash/JWT, la política de permisos en `permisos.py` y las dependencias de autenticación. `usuarios` contiene el único módulo funcional de la etapa. Las rutas solo validan entrada y delegan; los servicios aplican reglas; el repositorio encapsula SQLAlchemy. Los módulos futuros existen como paquetes vacíos para no inventar comportamiento.

*Alternativa considerada:* una estructura plana por tipo (`app/routes`, `app/services`). Se descarta porque el plan exige módulos por dominio y la trazabilidad de las etapas siguientes.

### D2. Configuración por variables de entorno

Se usa `pydantic-settings` para cargar configuración tipada desde el entorno, con un archivo `.env.example` versionado y `.env` fuera de Git. Variables mínimas: `DATABASE_URL`, `JWT_SECRETO`, `JWT_ALGORITMO` (HS256), `JWT_MINUTOS_VIGENCIA` (60 por defecto) y, para el comando de inicialización, `ADMIN_NOMBRE`, `ADMIN_CORREO` y `ADMIN_CONTRASENA`. La aplicación falla al arrancar si falta una variable crítica, en lugar de usar un secreto por defecto.

*Alternativa considerada:* configuración en un módulo Python versionado. Se descarta porque expondría el secreto JWT y rompería la reproducibilidad entre entornos.

### D3. SQLAlchemy síncrono y Alembic

Se usa SQLAlchemy 2.x en modo síncrono con el driver `psycopg` y endpoints `def` de FastAPI. La simplicidad del dominio no justifica la complejidad de async, y el plan prioriza claridad y trazabilidad.

El esquema se versiona con **Alembic** desde la primera tabla (Q12 resuelta). Aunque `create_all` sería más simple ahora, no versiona cambios y no permite expresar con comodidad la restricción de exclusión que la Etapa 3 necesita; Alembic permite añadir SQL explícito en una migración. La migración inicial crea la tabla `usuarios`.

*Alternativa considerada:* `Base.metadata.create_all` al arrancar. Se descarta por falta de historial y por el requisito de esquema reproducible y versionado.

### D4. Modelo de usuario

Tabla `usuarios`:

| Columna | Tipo | Restricciones |
|---|---|---|
| `id` | entero autoincremental | clave primaria |
| `nombre` | texto | obligatorio |
| `correo` | texto | obligatorio, único, normalizado a minúsculas |
| `contrasena_hash` | texto | obligatorio |
| `rol` | texto con restricción de valores | obligatorio; `solicitante`, `coordinador`, `administrador` |
| `activo` | booleano | obligatorio, valor inicial verdadero |
| `creado_en` / `actualizado_en` | marca de tiempo con zona | generadas por la base de datos |

El identificador entero se elige por simplicidad; no hay requisito de identificadores no adivinables. Los roles se persisten como códigos estables en minúsculas y las etiquetas en español se usan en la API y la documentación. El correo se normaliza a minúsculas y su unicidad no distingue mayúsculas (Q2 resuelta). Los roles se persisten como códigos en minúsculas (Q8 resuelta).

*Alternativa considerada:* UUID como identificador. Se descarta por complejidad innecesaria para un proyecto individual; puede cambiarse si se confirma lo contrario.

### D5. Hash de contraseñas y longitud mínima

Se usa `pwdlib` con Argon2id para generar y verificar hashes, sin almacenar nunca la contraseña. La verificación se encapsula en `core/seguridad.py` para que los servicios no dependan de la biblioteca concreta. Toda contraseña debe tener al menos 8 caracteres (Q3 resuelta); no se añaden reglas de complejidad adicionales.

*Alternativa considerada:* `passlib[bcrypt]`. Se descarta por el estado de mantenimiento de `passlib`; Argon2id es el algoritmo recomendado actualmente.

### D6. Autenticación JWT y sesión de petición

- `POST /auth/registro` es público y crea siempre una cuenta con rol Solicitante (Q1 resuelta). Los roles Coordinador y Administrador solo los asigna un Administrador.
- `POST /auth/login` recibe correo y contraseña, valida la cuenta activa y devuelve únicamente `{access_token, token_type}` (Q13 resuelta).
- El token se firma con HS256 mediante **PyJWT** y contiene `sub` (identificador), `iat` y `exp`; no contiene claim de rol (Q4 resuelta). La vigencia es de 60 minutos por defecto y se configura con `JWT_MINUTOS_VIGENCIA`.
- Un endpoint protegido usa una dependencia que decodifica el token, carga la cuenta desde la base de datos y responde 401 si el token falta, es inválido, está expirado o la cuenta está inactiva.
- El rol efectivo se toma de la cuenta cargada, no del token, para que un cambio de rol o una desactivación surtan efecto en peticiones posteriores. Una cuenta desactivada con token vigente recibe 401 (Q6 resuelta).

*Alternativa considerada:* confiar en el rol del token durante toda su vigencia. Se descarta porque contradice el requisito de usar el rol vigente y permitiría que una cuenta desactivada siguiera operando hasta expirar.

### D7. Política central de permisos

La matriz 2.5 se codifica como una tabla inmutable en `app/core/permisos.py`, única ubicación de la política, que asocia cada operación con el conjunto de roles autorizados:

| Operación | Solicitante | Coordinador | Administrador |
|---|---|---|---|
| `consultar_espacios` | ✓ | ✓ | ✓ |
| `crear_reserva` | ✓ | ✓ | ✓ |
| `cancelar_reserva_propia` | ✓ | ✓ | ✓ |
| `cancelar_reserva_ajena` | ✗ | ✓ | ✓ |
| `ver_historial_propio` | ✓ | ✓ | ✓ |
| `ver_historial_global` | ✗ | ✓ | ✓ |
| `aprobar_rechazar` | ✗ | ✓ | ✓ |
| `gestionar_espacios` | ✗ | ✗ | ✓ |
| `gestionar_usuarios` | ✗ | ✗ | ✓ |

Las rutas declaran la operación requerida mediante una dependencia `requiere_permiso(...)`; la dependencia autentica y consulta la política. Las operaciones sobre recursos propios o ajenos combinan el permiso con una comprobación de propiedad en el servicio. Las operaciones de las etapas 2 a 4 ya existen en la política, pero no tienen endpoint en esta etapa. Toda denegación responde 403 (Q9 resuelta).

*Alternativa considerada:* comprobar roles con condicionales dispersos en cada ruta. Se descarta porque duplicaría la matriz y favorecería divergencias.

### D8. Superficie de API de la etapa

| Método | Ruta | Acceso | Propósito |
|---|---|---|---|
| POST | `/auth/registro` | público | Crear cuenta Solicitante |
| POST | `/auth/login` | público | Obtener `access_token` y `token_type` |
| GET | `/usuarios/me` | autenticado | Consultar la propia cuenta |
| GET | `/usuarios` | Administrador | Listar cuentas |
| GET | `/usuarios/{id}` | Administrador | Consultar una cuenta |
| PATCH | `/usuarios/{id}` | Administrador | Editar nombre, correo o estado activo |
| PUT | `/usuarios/{id}/rol` | Administrador | Asignar rol |

La API de D8 se acepta tal como está (Q13 resuelta). Las respuestas de error usan el formato `detail` de FastAPI con los códigos 401 para autenticación fallida, 403 para permisos insuficientes, 409 para correo duplicado y 422 para datos incompletos o inválidos (Q7 resuelta). La documentación interactiva de FastAPI queda en `/docs` (y el esquema en `/openapi.json`).

### D9. Docker Compose y ejecución

`docker-compose.yml` define dos servicios: `api` (construida desde el Dockerfile) y `db` (`postgres:16` con volumen persistente). `api` espera a que `db` esté saludable mediante healthcheck y `depends_on`. Las migraciones de Alembic se aplican con un comando documentado y explícito; no se ejecutan automáticamente en cada petición (Q14 resuelta). El README documenta `docker compose up`, la aplicación de migraciones y `docker compose run --rm api pytest`.

*Alternativa considerada:* PostgreSQL instalado en el host. Se descarta porque el plan exige reproducibilidad con Docker Compose.

### D10. Estrategia de pruebas

- **Unitarias:** hash y verificación de contraseña, longitud mínima, normalización de correo, validación de datos, emisión y decodificación de JWT (`sub`, `iat`, `exp`, sin rol) y cada celda de la matriz de permisos mediante pruebas parametrizadas.
- **Integración:** con `TestClient` contra PostgreSQL real de Docker; registro, duplicado sin distinguir mayúsculas, contraseña corta, login válido/inválido/inactivo, token expirado, cuenta desactivada con token vigente, `/usuarios/me`, administración por rol, consulta por id, edición y reactivación.
- **Concurrencia:** carpeta creada con un marcador; no hay comportamiento concurrente en esta etapa.
- Las pruebas de integración usan una base de datos de pruebas separada dentro del mismo contenedor PostgreSQL (Q15 resuelta), no una base en memoria, porque el plan exige PostgreSQL.
- Cada prueba referencia en su nombre el requisito o escenario que cubre.

### D11. Inicialización del primer Administrador

Se añade un comando idempotente, ejecutable contra la base de datos configurada, que lee `ADMIN_NOMBRE`, `ADMIN_CORREO` y `ADMIN_CONTRASENA` y crea una cuenta Administrador activa únicamente si no existe ninguna cuenta con rol Administrador (Q5 resuelta). La contraseña se almacena con hash y la ejecución repetida no duplica ni modifica la cuenta existente.

**Limitación conocida:** si el único Administrador se desactiva, el comando no crea otro porque ya existe una cuenta con rol Administrador; si pierde su rol, el sistema queda sin ninguna cuenta con ese rol. El plan no define un mecanismo de recuperación ni de reasignación de emergencia, por lo que esta situación requiere intervención manual sobre la base de datos o una decisión futura. No se añaden reglas no previstas en el plan.

## Risks / Trade-offs

- **Único Administrador desactivado o sin su rol** → limitación conocida documentada en D11; el plan no define recuperación y no se inventa un mecanismo en esta etapa.
- **Fuga del secreto JWT** → secreto solo por entorno, `.env` ignorado por Git y `.env.example` con valor de ejemplo no utilizable.
- **Divergencia entre la matriz y el código** → una única tabla en `app/core/permisos.py` y pruebas parametrizadas por celda.
- **Pruebas de integración acopladas a la base de desarrollo** → base de datos de pruebas separada dentro del mismo contenedor PostgreSQL y limpieza entre pruebas.
- **Rol obsoleto en el token** → el rol se lee de la cuenta en cada petición; coste de una consulta adicional, aceptable a esta escala.
- **Alembic añade complejidad inicial** → se compensa con versionado reproducible y con la restricción de exclusión de la Etapa 3.
- **Sobreingeniería en la Etapa 1** → los módulos futuros quedan vacíos y la política de permisos no implementa endpoints de etapas posteriores.

## Migration Plan

No hay sistema previo ni datos que migrar. La implementación parte de un repositorio vacío de código:

1. Crear estructura, dependencias y Docker Compose.
2. Crear la migración inicial de `usuarios` con Alembic.
3. Implementar seguridad, autenticación, administración de usuarios y comando del primer Administrador.
4. Ejecutar la suite completa antes de cada commit.

Rollback: si una migración falla, `alembic downgrade` a la revisión anterior; para descartar el entorno local, `docker compose down -v`. No aplica despliegue en producción en esta etapa.

## Decisiones resueltas

Todas las preguntas abiertas quedaron resueltas y no se trasladan a la implementación como supuestos pendientes.

- **Q1. Modo de registro:** auto-registro público siempre con rol Solicitante; los demás roles solo los asigna un Administrador.
- **Q2. Unicidad y normalización del correo:** correo único normalizado a minúsculas, sin distinguir mayúsculas.
- **Q3. Política de contraseñas:** mínimo 8 caracteres, sin reglas de complejidad adicionales.
- **Q4. JWT:** HS256, vigencia de 60 minutos configurable por variable de entorno, claims `sub`, `iat` y `exp`, sin claim de rol; el rol se lee siempre de la cuenta.
- **Q5. Primer Administrador:** comando idempotente que crea el primer Administrador desde variables de entorno solo si no existe ningún Administrador; ver limitación conocida en D11.
- **Q6. Tokens tras desactivar una cuenta:** se rechazan con 401 al cargar la cuenta inactiva.
- **Q7. Errores y códigos HTTP:** 401 para credenciales inválidas y token ausente, inválido o expirado; 403 para permisos insuficientes; 409 para correo duplicado; 422 para datos incompletos o inválidos.
- **Q8. Roles internos:** códigos estables en minúsculas (`solicitante`, `coordinador`, `administrador`).
- **Q9. Acceso denegado:** respuesta genérica de permisos insuficientes con 403.
- **Q10. Coordinador aprobando solicitudes propias:** queda para la Etapa 4, según el plan.
- **Q11. Alcance de gestionar usuarios:** listar, consultar por id, editar nombre o correo, asignar rol, activar y desactivar.
- **Q12. Migraciones:** Alembic confirmado.
- **Q13. Contrato de la API:** se acepta la API de D8; el login devuelve solo `access_token` y `token_type`.
- **Q14. Migraciones automáticas o manuales:** comando documentado y explícito.
- **Q15. Base de datos de pruebas:** base de datos separada dentro del mismo contenedor PostgreSQL.
- **Q16. Endpoint de salud:** no se incluye; la documentación interactiva sirve como verificación.
- **Fuera de alcance según el plan:** recuperación de contraseña, renovación de tokens, zona horaria de fechas (sección 2.6.2) y reservas futuras de espacios desactivados (sección 2.6.4).
