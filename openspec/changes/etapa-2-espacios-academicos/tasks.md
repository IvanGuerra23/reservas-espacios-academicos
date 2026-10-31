# Tasks

## 1. Prerrequisitos y decisiones

- [ ] 1.1 Verificar que la Etapa 1 esté implementada (autenticación, `app/core/permisos.py`, `requiere_permiso`, migración inicial y base de datos de pruebas); verificación: `docker compose run --rm api pytest` ejecuta la suite de la Etapa 1 sin fallos.
- [x] 1.2 Confirmar con el autor las preguntas Q1–Q10 de `design.md` y registrar las decisiones sobre reactivación, visibilidad, validaciones, contrato, unicidad, normalización, idempotencia, edición de estado, auditoría y 2.6.4; verificación: `design.md` contiene la sección "Decisiones resueltas" y las especificaciones y pruebas se ajustan a lo acordado.
- [x] 1.3 Confirmar que la reactivación forma parte del alcance y que la sección 2.6.4 queda resuelta; verificación: `proposal.md`, `design.md`, `specs/space-management/spec.md` y las tareas usan la misma decisión.

## 2. Modelo y migración

- [ ] 2.1 Implementar el modelo `Espacio` con id, nombre, tipo restringido a `laboratorio`, `auditorio` y `sala_computo`, capacidad, ubicación y activo; verificación: prueba de integración `test_persistencia_espacio` y prueba unitaria `test_tipos_espacio`.
- [ ] 2.2 Crear la migración de Alembic para `espacios` con sus restricciones y `downgrade`; verificación: `alembic upgrade head` crea la tabla, `alembic downgrade -1` la elimina y la prueba de integración `test_esquema_espacios_existe` la consulta.
- [ ] 2.3 Implementar los esquemas de creación y edición parcial, con validación de campos obligatorios, recorte de `nombre` y `ubicacion`, rechazo de valores vacíos, tipo permitido y capacidad; verificación: pruebas unitarias `test_espacio_requiere_campos_obligatorios`, `test_espacio_tipo_invalido`, `test_espacio_capacidad_invalida`, `test_espacio_recorta_nombre_y_ubicacion`, `test_espacio_nombre_vacio_se_rechaza` y `test_edicion_parcial_no_cambia_activo`.

## 3. Repositorio y servicios

- [ ] 3.1 Implementar el repositorio de espacios con creación, obtención por id, listado, actualización y cambio de estado para desactivar y reactivar; verificación: pruebas de integración `test_repositorio_crea_espacio`, `test_repositorio_lista_espacios`, `test_repositorio_desactiva_espacio` y `test_repositorio_reactiva_espacio`.
- [ ] 3.2 Implementar el servicio de creación con `activo` verdadero y sin lógica de negocio en la ruta; verificación: prueba unitaria `test_servicio_crea_espacio_activo` y prueba de integración `test_servicio_creacion_persiste_campos`.
- [ ] 3.3 Implementar el servicio de edición de nombre, tipo, capacidad y ubicación sin modificar `activo`; verificación: pruebas unitarias `test_servicio_edita_campos` y `test_servicio_edicion_conserva_activo`.
- [ ] 3.4 Implementar el servicio de desactivación que conserva el registro, cambia `activo` a falso y es idempotente si ya está inactivo; verificación: pruebas unitarias `test_servicio_desactiva_conserva_registro` y `test_servicio_desactiva_inactivo_sin_cambios`, y prueba de integración `test_desactivacion_persiste_estado`.
- [ ] 3.5 Implementar el servicio de reactivación que conserva el registro, cambia `activo` a verdadero y es idempotente si ya está activo; verificación: pruebas unitarias `test_servicio_reactiva_conserva_registro` y `test_servicio_reactiva_activo_sin_cambios`, y prueba de integración `test_reactivacion_persiste_estado`.

## 4. API y permisos

- [ ] 4.1 Registrar el router de espacios y exponer `POST /espacios` para el Administrador; verificación: pruebas de integración `test_crear_espacio` (201), `test_crear_espacio_sin_token` (401), `test_crear_espacio_rol_no_autorizado` (403) y `test_crear_espacio_datos_invalidos` (422).
- [ ] 4.2 Exponer `GET /espacios` para cualquier usuario autenticado; verificación: pruebas de integración `test_listar_espacios` (200), `test_listar_espacios_sin_token` (401) y `test_listar_espacios_incluye_estado_activo`.
- [ ] 4.3 Exponer `GET /espacios/{id}` para cualquier usuario autenticado; verificación: pruebas de integración `test_detalle_espacio` (200), `test_detalle_espacio_inexistente` (404) y `test_detalle_espacio_sin_token` (401).
- [ ] 4.4 Exponer `PATCH /espacios/{id}` para el Administrador; verificación: pruebas de integración `test_editar_espacio` (200), `test_editar_espacio_inexistente` (404), `test_editar_espacio_rol_no_autorizado` (403), `test_editar_espacio_datos_invalidos` (422) y `test_editar_espacio_sin_token` (401).
- [ ] 4.5 Exponer `POST /espacios/{id}/desactivar` para el Administrador; verificación: pruebas de integración `test_desactivar_espacio` (200), `test_desactivar_espacio_inactivo` (200), `test_desactivar_espacio_inexistente` (404), `test_desactivar_espacio_rol_no_autorizado` (403) y `test_desactivar_espacio_sin_token` (401).
- [ ] 4.6 Exponer `POST /espacios/{id}/reactivar` para el Administrador; verificación: pruebas de integración `test_reactivar_espacio` (200), `test_reactivar_espacio_activo` (200), `test_reactivar_espacio_inexistente` (404), `test_reactivar_espacio_rol_no_autorizado` (403) y `test_reactivar_espacio_sin_token` (401).
- [ ] 4.7 Documentar los endpoints, sus códigos, la idempotencia y los tipos de espacio en `README.md`; verificación: la documentación coincide con los escenarios de `specs/space-management/spec.md` y con `test_openapi_espacios`.

## 5. Integración y cierre de la etapa

- [ ] 5.1 Ejecutar la suite completa de las Etapas 1 y 2 contra PostgreSQL; verificación: `docker compose run --rm api pytest` termina con todas las pruebas en verde.
- [ ] 5.2 Verificar que cada escenario de `space-management` tiene al menos una prueba cuyo nombre lo referencia; verificación: tabla de trazabilidad requisito → prueba → código incluida en el README y revisada contra la especificación.
- [ ] 5.3 Revisar el diff completo y confirmar que se añadió reactivación, que no se añadió disponibilidad horaria, reservas, aprobaciones ni eliminación de espacios, y que la edición no modifica `activo`; verificación: revisión del autor registrada en el commit de la etapa.
- [ ] 5.4 Confirmar que las decisiones Q1–Q10 están aplicadas y que la sección 2.6.4 quedó documentada igual que en las Etapas 3 y 4: desactivar no modifica reservas existentes y solo impide nuevas solicitudes de reserva sobre el espacio; verificación: pruebas de listado/estado e inspección de `design.md`, `proposal.md` y `specs/space-management/spec.md`.

## Workflow follow-up

- Archivar el cambio después de que el autor revise y apruebe las especificaciones y la implementación.
- Verificar el resultado archivado con `openspec validate` y la suite completa.
