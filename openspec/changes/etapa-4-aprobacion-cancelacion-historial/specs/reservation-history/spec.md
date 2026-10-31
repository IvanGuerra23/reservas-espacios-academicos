# Spec Delta

## Purpose

Permite consultar el historial de reservas propias y el de todos los usuarios, conservando las reservas rechazadas y canceladas como parte permanente de la trazabilidad del sistema.

## ADDED Requirements

### Requirement: Consultar el historial propio
El sistema SHALL permitir a cualquier usuario autenticado consultar las reservas en las que figura como solicitante, en todos los estados. La respuesta SHALL incluir los campos del modelo de reserva y SHALL limitarse a las reservas del usuario autenticado.

#### Scenario: Solicitante consulta su historial
- **WHEN** un Solicitante autenticado consulta su historial
- **THEN** el sistema responde 200 con sus reservas en todos los estados

#### Scenario: Coordinador consulta su historial
- **WHEN** un Coordinador autenticado consulta su historial
- **THEN** el sistema responde 200 con las reservas en las que figura como solicitante

#### Scenario: Administrador consulta su historial
- **WHEN** un Administrador autenticado consulta su historial
- **THEN** el sistema responde 200 con las reservas en las que figura como solicitante

#### Scenario: Historial propio vacío
- **WHEN** un usuario autenticado sin reservas consulta su historial
- **THEN** el sistema responde 200 con una lista vacía

#### Scenario: El historial propio no incluye reservas ajenas
- **WHEN** un usuario autenticado consulta su historial y existen reservas de otros usuarios
- **THEN** el sistema responde 200 y no incluye esas reservas ajenas

#### Scenario: Historial propio sin autenticación
- **WHEN** se consulta el historial propio sin token
- **THEN** el sistema responde 401

#### Scenario: Historial propio incluye estados terminales
- **WHEN** el usuario autenticado tiene reservas en estado `rechazada` o `cancelada`
- **THEN** el sistema las incluye en su historial con su estado, motivo de rechazo y revisor cuando corresponda

### Requirement: Consultar el historial global
El sistema SHALL permitir al Coordinador y al Administrador consultar todas las reservas de todos los usuarios en todos los estados. El Solicitante SHALL recibir 403 y las peticiones sin token 401.

#### Scenario: Coordinador consulta el historial global
- **WHEN** un Coordinador autenticado consulta el historial global
- **THEN** el sistema responde 200 con las reservas de todos los usuarios

#### Scenario: Administrador consulta el historial global
- **WHEN** un Administrador autenticado consulta el historial global
- **THEN** el sistema responde 200 con las reservas de todos los usuarios

#### Scenario: Solicitante intenta consultar el historial global
- **WHEN** un Solicitante autenticado intenta consultar el historial global
- **THEN** el sistema responde 403

#### Scenario: Historial global sin autenticación
- **WHEN** se consulta el historial global sin token
- **THEN** el sistema responde 401

#### Scenario: Historial global incluye todos los estados y usuarios
- **WHEN** existen reservas en estados `pendiente`, `aprobada`, `rechazada` y `cancelada` de varios usuarios
- **THEN** el sistema responde 200 e incluye todas ellas con su estado y sus datos de revisión

### Requirement: Conservación permanente del historial
El sistema SHALL conservar toda reserva creada. Rechazar o cancelar SHALL cambiar únicamente el estado y los datos de revisión, sin eliminar la fila. El sistema SHALL no exponer una operación de eliminación de reservas.

#### Scenario: Rechazo conserva la reserva
- **WHEN** una solicitud es rechazada
- **THEN** la reserva permanece en el historial con estado `rechazada`, su motivo y su revisor

#### Scenario: Cancelación conserva la reserva
- **WHEN** una reserva es cancelada
- **THEN** la reserva permanece en el historial con estado `cancelada`

#### Scenario: Historial tras una transición
- **WHEN** una reserva cambia de estado
- **THEN** el historial muestra su estado actual sin perder el identificador ni los datos de creación y actualización

#### Scenario: Intento de eliminación de una reserva
- **WHEN** se intenta eliminar una reserva con `DELETE /reservas/{id}`
- **THEN** el sistema responde 404 porque no existe una ruta de borrado y la reserva permanece en el historial

#### Scenario: Intento de eliminación de la colección de reservas
- **WHEN** se intenta eliminar la colección con `DELETE /reservas`
- **THEN** el sistema responde 405 porque la ruta existe solo para consulta y creación, y las reservas permanecen
