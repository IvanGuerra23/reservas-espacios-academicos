# Tasks

## 1. Prerrequisitos y decisiones

- [ ] 1.1 Verificar que las Etapas 1 y 2 estén implementadas (autenticación, `app/core/permisos.py`, `requiere_permiso`, tabla `espacios`, migraciones y base de datos de pruebas); verificación: `docker compose run --rm api pytest` ejecuta las suites de las Etapas 1 y 2 sin fallos.
- [ ] 1.2 Confirmar con el autor las preguntas bloqueantes Q1, Q2, Q3, Q4, Q5, Q6, Q7, Q8 y Q9 de `design.md` y registrar las decisiones; verificación: `design.md` deja de tener preguntas bloqueantes y las especificaciones, el diseño y las pruebas se ajustan a lo acordado.
- [ ] 1.3 Confirmar o aceptar las preguntas diferibles Q10–Q16 y dejar constancia de que no se impone duración máxima (Q11); verificación: `design.md` refleja las decisiones y las tareas de validación usan el comportamiento acordado.

## 2. Modelo, migración y esquemas

- [ ] 2.1 Implementar el modelo `Reserva` con `espacio_id`, `solicitante_id`, `inicio`, `fin`, `estado`, `motivo_rechazo`, `revisor_id`, `creado_en` y `actualizado_en`, y la restricción de estado válido; verificación: prueba de integración `test_persistencia_reserva` y prueba unitaria `test_estados_reserva_validos`.
- [ ] 2.2 Crear la migración de Alembic que habilita `btree_gist`, crea `reservas`, sus índices, la restricción `fin > inicio` y la restricción de exclusión `reservas_sin_solapamiento` con rango `[inicio, fin)` y predicado `estado IN ('pendiente', 'aprobada')`; verificación: `alembic upgrade head` crea la extensión, la tabla y la restricción, `alembic downgrade -1` las elimina y la prueba de integración `test_esquema_reservas_existe` inspecciona sus metadatos.
- [ ] 2.3 Implementar los esquemas de entrada y salida de creación de reserva y de consulta de disponibilidad, sin aceptar `solicitante_id`, `estado`, `motivo_rechazo` ni `revisor_id` como campos de entrada; verificación: pruebas unitarias `test_esquema_reserva_ignora_solicitante`, `test_esquema_reserva_ignora_estado` y `test_esquema_disponibilidad_requiere_inicio_y_fin`.

## 3. Reglas de tiempo y validación de espacio

- [ ] 3.1 Implementar el reloj inyectable y la configuración `HORARIO_APERTURA`/`HORARIO_CIERRE` con valores por defecto 07:00 y 21:00; verificación: pruebas unitarias `test_horario_configurable` y `test_reloj_inyectable`.
- [ ] 3.2 Implementar la validación de alineación a bloques de 30 minutos, duración mínima de 30 minutos, ausencia de duración máxima, horario global, fecha futura estricta y rechazo de rangos que cruzan de día; verificación: pruebas unitarias `test_inicio_alineado`, `test_fin_alineado`, `test_duracion_minima`, `test_sin_duracion_maxima`, `test_inicio_en_apertura`, `test_fin_en_cierre`, `test_fuera_de_horario`, `test_fecha_pasada`, `test_mismo_dia_futuro` y `test_rango_cruza_dia`.
- [ ] 3.3 Implementar la comprobación de espacio existente y activo, con 404 para inexistente y 409 para inactivo; verificación: pruebas de integración `test_reserva_espacio_inexistente` y `test_reserva_espacio_inactivo`.

## 4. Repositorio y creación de solicitudes

- [ ] 4.1 Implementar el repositorio de reservas con creación, obtención por id, consulta de solapamiento por espacio y rango, y consulta de disponibilidad limitada a estados `pendiente` y `aprobada`; verificación: pruebas de integración `test_repositorio_crea_reserva`, `test_repositorio_detecta_solapamiento` y `test_repositorio_ignora_estados_no_bloqueantes`.
- [ ] 4.2 Implementar el servicio de creación que toma el solicitante del token, fuerza estado `pendiente`, aplica las reglas de tiempo y espacio, comprueba solapamiento y traduce la violación de exclusión `23P01` a 409 con rollback; verificación: prueba unitaria `test_servicio_fuerza_estado_pendiente`, prueba de integración `test_creacion_conflicto_409` y prueba de integración `test_creacion_revierte_transaccion_tras_exclusion`.
- [ ] 4.3 Exponer `POST /reservas` para cualquier usuario autenticado con `requiere_permiso("crear_reserva")`; verificación: pruebas de integración `test_crear_reserva` (201), `test_crear_reserva_sin_token` (401), `test_crear_reserva_espacio_inexistente` (404), `test_crear_reserva_solapada` (409), `test_crear_reserva_datos_invalidos` (422) y `test_crear_reserva_solicitante_del_token`.
- [ ] 4.4 Verificar las reglas de creación de extremo a extremo, incluidos los tres roles y la ausencia de máximo; verificación: pruebas de integración `test_crear_reserva_cualquier_rol`, `test_crear_reserva_30_minutos`, `test_crear_reserva_varias_horas` y `test_crear_reserva_duracion_cero`.

## 5. Consulta de disponibilidad

- [ ] 5.1 Implementar el servicio de disponibilidad que valida el rango, comprueba el espacio activo y consulta solapamientos con `inicio < fin_consulta AND fin > inicio_consulta` sobre estados bloqueantes; verificación: pruebas unitarias `test_disponibilidad_intervalo_semiabierto` y `test_disponibilidad_solo_estados_bloqueantes`.
- [ ] 5.2 Exponer `GET /espacios/{id}/disponibilidad` para cualquier usuario autenticado con `requiere_permiso("consultar_espacios")`, devolviendo `espacio_id`, `inicio`, `fin` y `disponible` sin datos de otras reservas; verificación: pruebas de integración `test_disponibilidad_libre` (200), `test_disponibilidad_ocupada` (200), `test_disponibilidad_sin_token` (401), `test_disponibilidad_espacio_inexistente` (404), `test_disponibilidad_espacio_inactivo` (409) y `test_disponibilidad_rango_invalido` (422).
- [ ] 5.3 Verificar la semántica de intervalos y los estados no bloqueantes; verificación: pruebas de integración `test_disponibilidad_rangos_contiguos`, `test_disponibilidad_solapamiento_parcial`, `test_disponibilidad_rango_contiene_reserva`, `test_disponibilidad_otro_espacio` y `test_disponibilidad_ignora_rechazada_y_cancelada`.

## 6. Concurrencia y no solapamiento

- [ ] 6.1 Implementar la prueba de concurrencia con dos hilos, sesiones independientes y barrera, lanzando dos solicitudes simultáneas para el mismo espacio y rango; verificación: `tests/concurrency/test_solapamiento_concurrente.py::test_dos_solicitudes_simultaneas_mismo_rango` obtiene un 201, un 409 y una sola reserva persistida.
- [ ] 6.2 Verificar que la concurrencia no bloquea espacios distintos; verificación: `tests/concurrency/test_solapamiento_concurrente.py::test_dos_solicitudes_simultaneas_espacios_distintos` obtiene dos 201 y dos reservas persistidas.
- [ ] 6.3 Verificar la estabilidad de la prueba de concurrencia y su independencia del orden de ejecución; verificación: ejecutar `pytest tests/concurrency/test_solapamiento_concurrente.py -q` diez veces consecutivas sin fallos intermitentes.
- [ ] 6.4 Verificar directamente la restricción de exclusión con inserciones SQL; verificación: pruebas de integración `test_exclusion_rechaza_pendiente_solapada`, `test_exclusion_rechaza_aprobada_solapada`, `test_exclusion_permite_contiguas`, `test_exclusion_permite_espacios_distintos` y `test_exclusion_ignora_no_bloqueantes`.

## 7. Integración y cierre de la etapa

- [ ] 7.1 Registrar el router de reservas en `app/main.py` y documentar los endpoints, códigos y reglas en `README.md`; verificación: `test_openapi_reservas` confirma las rutas y la documentación coincide con los escenarios de `specs/`.
- [ ] 7.2 Ejecutar la suite completa de las Etapas 1, 2 y 3 contra PostgreSQL; verificación: `docker compose run --rm api pytest` termina con todas las pruebas en verde.
- [ ] 7.3 Verificar que cada escenario de `reservation-requests`, `space-availability` y `overlap-prevention` tiene al menos una prueba que lo referencia; verificación: tabla de trazabilidad requisito → prueba → código incluida en el README y revisada contra las especificaciones.
- [ ] 7.4 Revisar el diff completo y confirmar que no se añadió aprobación, rechazo, cancelación, historial, notificaciones, reservas recurrentes ni duración máxima; verificación: revisión del autor registrada en el commit de la etapa.

## Workflow follow-up

- Archivar el cambio después de que el autor revise y apruebe las especificaciones y la implementación.
- Verificar el resultado archivado con `openspec validate` y la suite completa.
