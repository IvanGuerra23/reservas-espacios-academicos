# Tasks

## 1. Decisiones previas (resueltas)

- [x] 1.1 Confirmar con el autor las decisiones Q1, Q4, Q5, Q12, Q13 y Q15 de `design.md` y registrarlas; verificación: la sección "Decisiones resueltas" de `design.md` contiene cada decisión y las especificaciones afectadas están alineadas.
- [x] 1.2 Confirmar o aceptar los supuestos Q2, Q3 y Q6–Q11, Q14 y Q16 y anotarlos como resueltos; verificación: "Decisiones resueltas" en `design.md` refleja cada decisión y las pruebas planificadas usan el comportamiento acordado.

## 2. Andamiaje del proyecto

- [ ] 2.1 Crear la estructura de paquetes `app/core` (incluido `permisos.py`), `app/usuarios`, `app/espacios`, `app/reservas`, `app/aprobaciones` y `tests/{unitarias,integracion,concurrencia}`; verificación: `python -c "import app"` funciona y la estructura coincide con el diseño.
- [ ] 2.2 Declarar dependencias (FastAPI, Uvicorn, SQLAlchemy, psycopg, Alembic, pydantic-settings, PyJWT, pwdlib, pytest, httpx) y fijar versiones; verificación: la instalación en el contenedor termina sin errores y `pytest --version` responde.
- [ ] 2.3 Implementar la configuración tipada por entorno y `.env.example`, con `JWT_ALGORITMO=HS256`, `JWT_MINUTOS_VIGENCIA=60` y fallo al arrancar si falta una variable crítica; verificación: pruebas unitarias `test_configuracion_falla_sin_secreto` y `test_configuracion_carga_valores`.
- [ ] 2.4 Crear `docker-compose.yml` y el Dockerfile con los servicios `api` y `db`, healthcheck de PostgreSQL y volumen persistente; verificación: `docker compose config` es válido y `docker compose up -d` deja ambos servicios saludables.
- [ ] 2.5 Configurar SQLAlchemy, la sesión de base de datos y Alembic con la migración inicial de `usuarios`; verificación: `alembic upgrade head` crea la tabla y la prueba de integración `test_esquema_usuarios_existe` la consulta.
- [ ] 2.6 Crear `app/main.py` con la aplicación FastAPI y la documentación interactiva; verificación: pruebas de integración `test_documentacion_disponible` y `test_openapi_disponible` reciben respuesta correcta.
- [ ] 2.7 Configurar pytest y `tests/conftest.py` con el cliente de pruebas y una base de datos de pruebas separada dentro del mismo contenedor PostgreSQL; verificación: `pytest --collect-only` descubre las carpetas y la prueba de integración `test_usa_base_de_datos_de_pruebas` confirma la conexión a esa base.
- [ ] 2.8 Documentar en `README.md` el arranque, las migraciones, la ejecución de pruebas y las variables de entorno; verificación: los comandos documentados se ejecutan tal cual en un entorno limpio.

## 3. Usuarios y registro

- [ ] 3.1 Implementar el modelo `Usuario` con id, nombre, correo único normalizado a minúsculas, hash de contraseña, rol, activo y marcas de tiempo; verificación: prueba de integración `test_persistencia_usuario` y prueba unitaria `test_correo_se_normaliza`.
- [ ] 3.2 Implementar los esquemas de entrada y salida, excluyendo contraseña y hash de las respuestas y validando la longitud mínima de 8 caracteres; verificación: pruebas unitarias `test_esquema_usuario_no_expone_hash`, `test_registro_requiere_campos_obligatorios` y `test_contrasena_menos_de_8_se_rechaza`.
- [ ] 3.3 Implementar hash y verificación de contraseña con Argon2id; verificación: pruebas unitarias `test_contrasena_se_almacena_con_hash`, `test_verificacion_contrasena_correcta` y `test_verificacion_contrasena_incorrecta`.
- [ ] 3.4 Implementar el repositorio y el servicio de registro público con rol Solicitante, cuenta activa y rechazo de correo duplicado sin distinguir mayúsculas; verificación: pruebas de integración `test_registro_exitoso`, `test_registro_correo_duplicado`, `test_registro_correo_duplicado_sin_distinguir_mayusculas` y `test_registro_datos_incompletos` con los códigos 409 y 422.
- [ ] 3.5 Exponer `POST /auth/registro` y conectar la ruta con el servicio sin lógica de negocio en la ruta; verificación: prueba de integración `test_endpoint_registro_crea_cuenta` y revisión de que la ruta solo valida y delega.
- [ ] 3.6 Exponer `GET /usuarios/me` para la propia cuenta autenticada; verificación: prueba de integración `test_usuario_consulta_su_cuenta` sin contraseña ni hash en la respuesta.
- [ ] 3.7 Implementar el comando idempotente de inicialización del primer Administrador desde `ADMIN_NOMBRE`, `ADMIN_CORREO` y `ADMIN_CONTRASENA`, solo si no existe ningún Administrador; verificación: pruebas de integración `test_inicializacion_primer_admin` y `test_inicializacion_no_duplica_admin`.

## 4. Autenticación con JWT

- [ ] 4.1 Implementar la creación y decodificación de tokens JWT con PyJWT, HS256 y la vigencia configurada, usando los claims `sub`, `iat` y `exp` sin claim de rol; verificación: pruebas unitarias `test_token_contiene_sub_iat_exp`, `test_token_no_contiene_rol`, `test_token_expirado_se_rechaza` y `test_token_con_firma_invalida_se_rechaza`.
- [ ] 4.2 Implementar `POST /auth/login` validando correo, contraseña y estado activo, con respuesta únicamente `access_token` y `token_type`; verificación: pruebas de integración `test_login_exitoso`, `test_login_credenciales_invalidas` (401) y `test_login_cuenta_inactiva` (401).
- [ ] 4.3 Implementar la dependencia de autenticación que carga la cuenta y responde 401 ante token ausente, inválido o expirado; verificación: pruebas de integración `test_endpoint_protegido_sin_token`, `test_endpoint_protegido_token_invalido`, `test_endpoint_protegido_token_expirado` y `test_endpoint_protegido_token_valido`.
- [ ] 4.4 Usar el rol vigente de la cuenta cargada, no el claim del token; verificación: prueba de integración `test_cambio_de_rol_afecta_peticiones_posteriores`, incluida la respuesta 403 cuando el nuevo rol carece del permiso.
- [ ] 4.5 Rechazar con 401 las cuentas desactivadas en endpoints protegidos aunque el token siga vigente; verificación: prueba de integración `test_cuenta_desactivada_con_token_vigente`.

## 5. Roles y permisos

- [ ] 5.1 Codificar en `app/core/permisos.py` la matriz de permisos de la sección 2.5 como política única de operaciones y roles; verificación: prueba unitaria parametrizada `test_matriz_permisos` que cubre las 27 combinaciones de 9 operaciones por 3 roles.
- [ ] 5.2 Implementar la dependencia `requiere_permiso` que autentica y evalúa la política, respondiendo 403 cuando se deniega; verificación: pruebas unitarias `test_permiso_concedido` y `test_permiso_denegado`, y prueba de integración `test_endpoint_deniega_rol_no_autorizado`.
- [ ] 5.3 Implementar los endpoints de administración `GET /usuarios`, `GET /usuarios/{id}`, `PATCH /usuarios/{id}` y `PUT /usuarios/{id}/rol`, reservados al Administrador; verificación: pruebas de integración `test_admin_lista_usuarios`, `test_admin_consulta_usuario_por_id`, `test_admin_edita_usuario`, `test_admin_edita_correo_duplicado` (409), `test_admin_asigna_rol` y `test_solicitante_y_coordinador_no_administran_usuarios` (403).
- [ ] 5.4 Validar que solo se asignen los roles Solicitante, Coordinador y Administrador; verificación: prueba unitaria `test_rol_desconocido_se_rechaza` y prueba de integración `test_asignacion_rol_invalido` (422).
- [ ] 5.5 Desactivar y reactivar cuentas conservando sus datos y verificar el efecto del estado activo; verificación: pruebas de integración `test_admin_desactiva_cuenta`, `test_cuenta_inactiva_no_inicia_sesion` (401), `test_admin_reactiva_cuenta` y `test_cuenta_reactivada_inicia_sesion`.
- [ ] 5.6 Documentar en `README.md` la tabla de permisos y la correspondencia con la matriz del plan; verificación: la tabla documentada coincide con `test_matriz_permisos`.

## 6. Integración y cierre de la etapa

- [ ] 6.1 Ejecutar toda la suite contra PostgreSQL y corregir cualquier fallo; verificación: `docker compose run --rm api pytest` termina con todas las pruebas en verde.
- [ ] 6.2 Verificar que cada escenario de las especificaciones tiene al menos una prueba cuyo nombre lo referencia; verificación: tabla de trazabilidad requisito → prueba → código incluida en el README y revisada contra `specs/`.
- [ ] 6.3 Revisar el diff completo y confirmar que no se añadió comportamiento fuera de las especificaciones (espacios, reservas o aprobaciones); verificación: revisión del autor registrada en el commit de la etapa.
- [ ] 6.4 Confirmar que las pruebas de etapas previas no aplican y que la carpeta de concurrencia queda creada sin implementar comportamiento de reservas; verificación: `pytest` pasa y `tests/concurrencia/` solo contiene el marcador de la carpeta.

## Workflow follow-up

- Archivar el cambio después de que el autor revise y apruebe las especificaciones y la implementación.
- Verificar el resultado archivado con `openspec validate` y la suite completa.
