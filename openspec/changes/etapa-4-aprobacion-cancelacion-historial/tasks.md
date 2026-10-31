# Tasks

## 1. Prerrequisitos y decisiones

- [ ] 1.1 Verificar que las Etapas 1 a 3 estén implementadas (autenticación, permisos, espacios, modelo `Reserva`, creación, disponibilidad y restricción de exclusión); verificación: `docker compose run --rm api pytest` ejecuta las suites de las Etapas 1 a 3 sin fallos.
- [ ] 1.2 Confirmar con el autor las preguntas Q1–Q9 de `design.md` y registrar las decisiones sobre autoaprobación, cancelación de reservas comenzadas, aprobación de solicitudes pasadas, espacios inactivos, registro de cancelador, historial y borrado; verificación: `design.md` deja de tener preguntas bloqueantes y las especificaciones, el diseño y las pruebas usan el comportamiento acordado.
- [ ] 1.3 Confirmar que la Etapa 4 no requiere migración de esquema y que los campos `estado`, `motivo_rechazo`, `revisor_id` y `actualizado_en` existen en `reservas`; verificación: `alembic upgrade head` no deja cambios pendientes y la prueba de integración `test_esquema_reservas_conserva_campos` los inspecciona.

## 2. Núcleo de transiciones

- [ ] 2.1 Implementar `app/reservas/transiciones.py` con la validación pura de las transiciones permitidas y el rechazo de estados terminales o repeticiones; verificación: pruebas unitarias `test_transiciones_permitidas`, `test_transiciones_prohibidas`, `test_estados_terminales` y `test_misma_transicion_repetida`.
- [ ] 2.2 Implementar el error de dominio de transición inválida y su traducción a 409 en la capa de servicios, sin lógica de estado en las rutas; verificación: pruebas unitarias `test_error_transicion_invalida` y prueba de integración `test_transicion_invalida_responde_409`.
- [ ] 2.3 Documentar en `README.md` la tabla de transiciones y los estados terminales; verificación: la tabla coincide con `test_transiciones_permitidas` y con la sección 2.2.1 del plan.

## 3. Aprobación y rechazo

- [ ] 3.1 Implementar en `app/reservas/repositorio.py` la carga de una reserva con bloqueo de fila (`SELECT ... FOR UPDATE`) y la actualización de estado, revisor, motivo y `actualizado_en`; verificación: pruebas de integración `test_repositorio_bloquea_reserva` y `test_repositorio_actualiza_revision`.
- [ ] 3.2 Implementar `app/aprobaciones/servicios.py` con la aprobación y el rechazo, el revisor tomado del token, el motivo obligatorio recortado y no vacío, y la aplicación de Q1–Q4; verificación: pruebas unitarias `test_aprobar_pendiente`, `test_rechazar_pendiente_con_motivo`, `test_rechazo_sin_motivo`, `test_rechazo_motivo_vacio`, `test_aprobar_solicitud_no_pendiente` y `test_rechazar_solicitud_no_pendiente`.
- [ ] 3.3 Exponer `POST /reservas/{id}/aprobar` y `POST /reservas/{id}/rechazar` con `requiere_permiso("aprobar_rechazar")`; verificación: pruebas de integración `test_aprobar_reserva` (200), `test_rechazar_reserva` (200), `test_aprobar_sin_token` (401), `test_aprobar_como_solicitante` (403), `test_aprobar_reserva_inexistente` (404), `test_aprobar_reserva_no_pendiente` (409), `test_rechazar_sin_motivo` (422) y `test_rechazo_motivo_vacio` (422).
- [ ] 3.4 Verificar que el revisor y el motivo quedan correctamente persistidos y que aprobar no altera el motivo; verificación: pruebas de integración `test_aprobacion_registra_revisor`, `test_rechazo_registra_motivo_y_revisor` y `test_aprobacion_deja_motivo_nulo`.
- [ ] 3.5 Documentar en `README.md` los endpoints de aprobación y rechazo, sus cuerpos y códigos; verificación: la documentación coincide con los escenarios de `reservation-workflow` y con `test_openapi_aprobaciones`.

## 4. Cancelación

- [ ] 4.1 Implementar en `app/reservas/servicios.py` la cancelación propia y ajena, la comprobación de propiedad, el límite temporal del Solicitante, la transición válida y la aplicación de Q2 y Q8; verificación: pruebas unitarias `test_solicitante_cancela_propia_futura`, `test_solicitante_no_cancela_propia_comenzada`, `test_solicitante_no_cancela_ajena`, `test_coordinador_cancela_ajena` y `test_administrador_cancela_ajena`.
- [ ] 4.2 Exponer `POST /reservas/{id}/cancelar` con autenticación y `requiere_permiso("cancelar_reserva_propia")`, aplicando `cancelar_reserva_ajena` desde el servicio cuando corresponda; verificación: pruebas de integración `test_cancelar_reserva_propia` (200), `test_cancelar_sin_token` (401), `test_cancelar_ajena_como_solicitante` (403), `test_cancelar_reserva_inexistente` (404), `test_cancelar_reserva_comenzada` (409) y `test_cancelar_reserva_terminal` (409).
- [ ] 4.3 Verificar la cancelación por Coordinador y Administrador, incluidas las reservas de otros usuarios y el comportamiento acordado en Q2; verificación: pruebas de integración `test_coordinador_cancela_ajena`, `test_administrador_cancela_ajena` y `test_coordinador_cancela_reserva_comenzada` con el código acordado.
- [ ] 4.4 Documentar en `README.md` la regla de cancelación por rol, la restricción temporal del Solicitante y el código 409; verificación: la documentación coincide con los escenarios de `reservation-workflow` y con `test_openapi_cancelacion`.

## 5. Historial

- [ ] 5.1 Implementar en `app/reservas/repositorio.py` las consultas de historial propio por `solicitante_id` y global, con todos los estados, y el esquema de respuesta sin contraseñas ni datos de autenticación; verificación: pruebas de integración `test_historial_propio_solo_del_usuario`, `test_historial_global_incluye_todos` y `test_historial_no_expone_datos_sensibles`.
- [ ] 5.2 Exponer `GET /reservas/mias` con `ver_historial_propio` y `GET /reservas` con `ver_historial_global`, declarando la ruta literal antes de las rutas con parámetro; verificación: pruebas de integración `test_historial_propio` (200), `test_historial_global` (200), `test_historial_sin_token` (401), `test_historial_global_como_solicitante` (403) y `test_rutas_historial_no_colisionan`.
- [ ] 5.3 Verificar que ambos historiales incluyen reservas pendientes, aprobadas, rechazadas y canceladas, con su motivo y revisor, y que el propio no incluye reservas ajenas; verificación: pruebas de integración `test_historial_incluye_todos_los_estados`, `test_historial_propio_no_incluye_ajenas` y `test_historial_global_incluye_varios_usuarios`.
- [ ] 5.4 Verificar la conservación permanente: rechazo y cancelación no eliminan filas, `DELETE /reservas/{id}` responde 404 y `DELETE /reservas` responde 405; verificación: pruebas de integración `test_rechazo_conserva_fila`, `test_cancelacion_conserva_fila`, `test_eliminacion_reserva_no_soportada` (404) y `test_eliminacion_coleccion_no_soportada` (405).
- [ ] 5.5 Documentar en `README.md` los endpoints de historial, el orden provisional y la ausencia de eliminación; verificación: la documentación coincide con los escenarios de `reservation-history` y con `test_openapi_historial`.

## 6. Integración del ciclo de vida

- [ ] 6.1 Verificar que rechazar y cancelar liberan el horario y permiten una nueva solicitud; verificación: pruebas de integración `test_rechazo_libera_horario`, `test_cancelacion_libera_horario` y `test_nueva_solicitud_tras_liberacion` (201).
- [ ] 6.2 Implementar la prueba de concurrencia de dos aprobaciones simultáneas con sesiones independientes y barrera, y verificar que solo una prevalece; verificación: `tests/concurrencia/test_transiciones_concurrentes.py::test_dos_aprobaciones_simultaneas` obtiene un 200, un 409 y un único revisor.
- [ ] 6.3 Verificar que la restricción de exclusión sigue actuando después de las transiciones y que aprobar una solicitud pendiente no crea un nuevo rango; verificación: pruebas de integración `test_aprobacion_no_duplica_rango` y `test_exclusion_sigue_activa_tras_aprobacion`.
- [ ] 6.4 Verificar el estado correcto de `actualizado_en` y que las transiciones exitosas lo modifican; verificación: prueba de integración `test_transicion_actualiza_marca_de_tiempo`.

## 7. Integración y cierre de la etapa

- [ ] 7.1 Ejecutar la suite completa de las Etapas 1 a 4 contra PostgreSQL; verificación: `docker compose run --rm api pytest` termina con todas las pruebas en verde.
- [ ] 7.2 Verificar que cada escenario de `reservation-workflow` y `reservation-history` tiene al menos una prueba que lo referencia; verificación: tabla de trazabilidad requisito → prueba → código incluida en el README y revisada contra las especificaciones.
- [ ] 7.3 Revisar el diff completo y confirmar que no se añadió notificaciones, reservas recurrentes, edición de reservas, eliminación, auditoría de transiciones ni migraciones de esquema no previstas; verificación: revisión del autor registrada en el commit de la etapa.
- [ ] 7.4 Verificar que las decisiones Q1–Q9 quedaron aplicadas y documentadas; verificación: `design.md` y `README.md` reflejan las respuestas y las pruebas correspondientes pasan.

## Workflow follow-up

- Archivar el cambio después de que el autor revise y apruebe las especificaciones y la implementación.
- Verificar el resultado archivado con `openspec validate` y la suite completa.
