# Proposal: Etapa 1 — Andamiaje, usuarios, autenticación y roles

## Why

El sistema de reservas necesita una base reproducible sobre la que construir las etapas siguientes. Sin un proyecto ejecutable, una base de datos PostgreSQL versionada, una suite de pruebas y un modelo de identidad con roles, no es posible implementar espacios, reservas ni aprobaciones con permisos verificables. La Etapa 1 establece esa base y el control de acceso que todas las etapas posteriores reutilizan.

## What Changes

- Se crea el andamiaje del proyecto: estructura modular de FastAPI (`app/main.py`, `app/core/`, `app/usuarios/`, `app/espacios/`, `app/reservas/`, `app/aprobaciones/`), configuración por variables de entorno y arranque reproducible con Docker Compose (aplicación + PostgreSQL).
- Se incorpora `pytest` como suite de pruebas con las carpetas `tests/unitarias/`, `tests/integracion/` y `tests/concurrencia/`, ejecutable dentro del entorno del proyecto.
- Se implementa el registro público de usuarios con contraseña almacenada como hash (mínimo 8 caracteres) y los campos definidos en el plan (identificador, nombre, correo, rol, activo). Todo registro crea un Solicitante; los demás roles solo los asigna un Administrador.
- Se implementa la administración de usuarios: consulta, consulta por identificador, edición de nombre o correo, asignación de rol y activación/desactivación, reservada al Administrador.
- Se implementa un comando idempotente para crear el primer Administrador desde variables de entorno, solo si no existe ninguno.
- Se implementa el inicio de sesión que emite un token JWT (HS256, vigencia configurable de 60 minutos por defecto, claims `sub`, `iat` y `exp`, sin rol) y la autenticación de peticiones protegidas mediante ese token.
- Se implementan los tres roles (Solicitante, Coordinador, Administrador) y la matriz de permisos de la sección 2.5 del plan como política autoritativa del sistema.
- No se implementan espacios, reservas ni aprobaciones: sus operaciones quedan definidas en la política de permisos, pero sus endpoints pertenecen a etapas posteriores.

## Capabilities

### New Capabilities

- `platform-bootstrap`: proyecto FastAPI modular ejecutable con Docker Compose, PostgreSQL conectado mediante SQLAlchemy, configuración por entorno y suite pytest lista para ejecutarse.
- `user-accounts`: registro público de usuarios como Solicitante, datos de la cuenta, hash y longitud mínima de contraseña, estado activo, inicialización del primer Administrador y administración de usuarios por parte del Administrador.
- `authentication`: inicio de sesión con correo y contraseña, emisión de token JWT y protección de endpoints mediante autenticación.
- `role-authorization`: modelo de roles Solicitante, Coordinador y Administrador, y verificación de permisos conforme a la matriz de la sección 2.5.

### Modified Capabilities

- Ninguna. El proyecto no tiene especificaciones principales previas.

## Impact

- **Código nuevo**: `app/main.py`, `app/core/` (configuración, seguridad, base de datos), `app/usuarios/` (rutas, servicios, acceso a datos), `tests/unitarias/`, `tests/integracion/`.
- **Infraestructura**: `docker-compose.yml`, Dockerfile de la aplicación, variables de entorno de ejemplo y mecanismo de creación/versionado del esquema de base de datos.
- **Dependencias**: FastAPI, Uvicorn, SQLAlchemy, driver de PostgreSQL, PyJWT, biblioteca de hash de contraseñas, pytest y cliente de pruebas HTTP.
- **API**: endpoints de registro público, inicio de sesión, consulta de la propia cuenta y administración de usuarios, más el comando de inicialización del primer Administrador.
- **Etapas futuras**: la política de roles y permisos queda disponible para que las Etapas 2 a 4 la apliquen en espacios, reservas y aprobaciones.
- **Decisiones resueltas**: las preguntas Q1–Q16 quedaron resueltas y documentadas en `design.md`; no quedan supuestos pendientes para implementar esta etapa.
