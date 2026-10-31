# Spec Delta

## Purpose

Define el ciclo de vida operativo de una reserva: aprobación y rechazo por Coordinador o Administrador, cancelación por el solicitante o por roles autorizados, y las transiciones de estado permitidas.

## ADDED Requirements

### Requirement: Aprobar solicitud pendiente
El sistema SHALL permitir al Coordinador y al Administrador aprobar una solicitud en estado `pendiente`. La aprobación SHALL cambiar el estado a `aprobada`, registrar al usuario autenticado como revisor y mantener nulo el motivo de rechazo. Los demás roles SHALL recibir 403 y las peticiones sin token 401.

#### Scenario: Aprobación exitosa por un Coordinador
- **WHEN** un Coordinador autenticado aprueba una solicitud pendiente de otro usuario
- **THEN** el sistema responde 200 con la reserva en estado `aprobada`, el Coordinador como revisor y `motivo_rechazo` nulo

#### Scenario: Aprobación exitosa por un Administrador
- **WHEN** un Administrador autenticado aprueba una solicitud pendiente
- **THEN** el sistema responde 200 con la reserva en estado `aprobada` y el Administrador como revisor

#### Scenario: Aprobación sin autenticación
- **WHEN** se intenta aprobar una solicitud sin token
- **THEN** el sistema responde 401 y no modifica la reserva

#### Scenario: Aprobación por un Solicitante
- **WHEN** un Solicitante autenticado intenta aprobar una solicitud
- **THEN** el sistema responde 403 y no modifica la reserva

#### Scenario: Aprobación de una solicitud inexistente
- **WHEN** un Coordinador o Administrador autenticado intenta aprobar un identificador que no corresponde a ninguna reserva
- **THEN** el sistema responde 404

#### Scenario: Aprobación de una solicitud que no está pendiente
- **WHEN** un Coordinador o Administrador autenticado intenta aprobar una reserva en estado `aprobada`, `rechazada` o `cancelada`
- **THEN** el sistema responde 409 y no modifica la reserva

### Requirement: Rechazar solicitud pendiente
El sistema SHALL permitir al Coordinador y al Administrador rechazar una solicitud en estado `pendiente` indicando un motivo obligatorio y no vacío. El rechazo SHALL cambiar el estado a `rechazada`, guardar el motivo y registrar al usuario autenticado como revisor. La ausencia o el vacío del motivo SHALL responder 422 sin modificar la reserva.

#### Scenario: Rechazo exitoso con motivo
- **WHEN** un Coordinador o Administrador autenticado rechaza una solicitud pendiente con un motivo no vacío
- **THEN** el sistema responde 200 con la reserva en estado `rechazada`, el motivo guardado y el usuario autenticado como revisor

#### Scenario: Rechazo sin motivo
- **WHEN** un Coordinador o Administrador autenticado intenta rechazar una solicitud pendiente sin enviar el motivo
- **THEN** el sistema responde 422 y no modifica la reserva

#### Scenario: Rechazo con motivo vacío
- **WHEN** un Coordinador o Administrador autenticado intenta rechazar una solicitud pendiente con un motivo vacío o compuesto solo por espacios
- **THEN** el sistema responde 422 y no modifica la reserva

#### Scenario: Rechazo sin autenticación
- **WHEN** se intenta rechazar una solicitud sin token
- **THEN** el sistema responde 401 y no modifica la reserva

#### Scenario: Rechazo por un Solicitante
- **WHEN** un Solicitante autenticado intenta rechazar una solicitud
- **THEN** el sistema responde 403 y no modifica la reserva

#### Scenario: Rechazo de una solicitud inexistente
- **WHEN** un Coordinador o Administrador autenticado intenta rechazar un identificador que no corresponde a ninguna reserva
- **THEN** el sistema responde 404

#### Scenario: Rechazo de una solicitud que no está pendiente
- **WHEN** un Coordinador o Administrador autenticado intenta rechazar una reserva en estado `aprobada`, `rechazada` o `cancelada`
- **THEN** el sistema responde 409 y no modifica la reserva

### Requirement: Cancelar una reserva propia
El sistema SHALL permitir al Solicitante cancelar una reserva propia en estado `pendiente` o `aprobada` cuyo inicio sea posterior al momento de la cancelación. El Coordinador y el Administrador SHALL poder cancelar sus propias reservas sin la restricción de inicio. La cancelación SHALL cambiar el estado a `cancelada` y conservar el registro.

#### Scenario: Solicitante cancela su reserva pendiente no iniciada
- **WHEN** un Solicitante autenticado cancela una reserva propia en estado `pendiente` cuyo inicio es futuro
- **THEN** el sistema responde 200 con la reserva en estado `cancelada`

#### Scenario: Solicitante cancela su reserva aprobada no iniciada
- **WHEN** un Solicitante autenticado cancela una reserva propia en estado `aprobada` cuyo inicio es futuro
- **THEN** el sistema responde 200 con la reserva en estado `cancelada`

#### Scenario: Coordinador o Administrador cancela una reserva propia
- **WHEN** un Coordinador o Administrador autenticado cancela una reserva propia en estado `pendiente` o `aprobada`
- **THEN** el sistema responde 200 con la reserva en estado `cancelada`

#### Scenario: Cancelación propia sin autenticación
- **WHEN** se intenta cancelar una reserva propia sin token
- **THEN** el sistema responde 401 y no modifica la reserva

#### Scenario: Cancelación de una reserva propia inexistente
- **WHEN** un usuario autenticado intenta cancelar un identificador que no corresponde a ninguna reserva
- **THEN** el sistema responde 404

#### Scenario: Solicitante cancela una reserva propia ya comenzada
- **WHEN** un Solicitante autenticado intenta cancelar una reserva propia cuyo inicio ya ocurrió
- **THEN** el sistema responde 409 y no modifica la reserva

#### Scenario: Cancelación de una reserva propia rechazada
- **WHEN** un usuario autenticado intenta cancelar una reserva propia en estado `rechazada`
- **THEN** el sistema responde 409 y no modifica la reserva

#### Scenario: Cancelación de una reserva propia ya cancelada
- **WHEN** un usuario autenticado intenta cancelar una reserva propia en estado `cancelada`
- **THEN** el sistema responde 409 y no modifica la reserva

### Requirement: Cancelar una reserva ajena
El sistema SHALL permitir al Coordinador y al Administrador cancelar cualquier reserva ajena en estado `pendiente` o `aprobada`. El Solicitante SHALL recibir 403 al intentar cancelar una reserva ajena. La cancelación SHALL cambiar el estado a `cancelada` y conservar el registro.

#### Scenario: Coordinador cancela una reserva ajena
- **WHEN** un Coordinador autenticado cancela una reserva ajena en estado `pendiente` o `aprobada`
- **THEN** el sistema responde 200 con la reserva en estado `cancelada`

#### Scenario: Administrador cancela una reserva ajena
- **WHEN** un Administrador autenticado cancela una reserva ajena en estado `pendiente` o `aprobada`
- **THEN** el sistema responde 200 con la reserva en estado `cancelada`

#### Scenario: Solicitante intenta cancelar una reserva ajena
- **WHEN** un Solicitante autenticado intenta cancelar una reserva que pertenece a otro usuario
- **THEN** el sistema responde 403 y no modifica la reserva

#### Scenario: Cancelación ajena sin autenticación
- **WHEN** se intenta cancelar una reserva ajena sin token
- **THEN** el sistema responde 401 y no modifica la reserva

#### Scenario: Cancelación de una reserva ajena inexistente
- **WHEN** un Coordinador o Administrador autenticado intenta cancelar un identificador que no corresponde a ninguna reserva
- **THEN** el sistema responde 404

#### Scenario: Cancelación de una reserva ajena en estado terminal
- **WHEN** un Coordinador o Administrador autenticado intenta cancelar una reserva ajena en estado `rechazada` o `cancelada`
- **THEN** el sistema responde 409 y no modifica la reserva

### Requirement: Transiciones de estado permitidas
El sistema SHALL permitir únicamente las transiciones `pendiente` a `aprobada`, `pendiente` a `rechazada`, `pendiente` a `cancelada` y `aprobada` a `cancelada`. Los estados `rechazada` y `cancelada` SHALL ser terminales. Cualquier otra transición SHALL responder 409 sin modificar la reserva.

#### Scenario: Transición de pendiente a aprobada
- **WHEN** se aprueba una reserva en estado `pendiente`
- **THEN** el sistema responde 200 y el estado cambia a `aprobada`

#### Scenario: Transición de pendiente a rechazada
- **WHEN** se rechaza con motivo una reserva en estado `pendiente`
- **THEN** el sistema responde 200 y el estado cambia a `rechazada`

#### Scenario: Transición de pendiente a cancelada
- **WHEN** se cancela una reserva en estado `pendiente`
- **THEN** el sistema responde 200 y el estado cambia a `cancelada`

#### Scenario: Transición de aprobada a cancelada
- **WHEN** se cancela una reserva en estado `aprobada`
- **THEN** el sistema responde 200 y el estado cambia a `cancelada`

#### Scenario: Transición de aprobada a rechazada no permitida
- **WHEN** se intenta rechazar una reserva en estado `aprobada`
- **THEN** el sistema responde 409 y el estado permanece en `aprobada`

#### Scenario: Repetición de una aprobación
- **WHEN** se intenta aprobar una reserva que ya está en estado `aprobada`
- **THEN** el sistema responde 409 y el estado permanece en `aprobada`

#### Scenario: Rechazada es terminal
- **WHEN** se intenta aprobar o cancelar una reserva en estado `rechazada`
- **THEN** el sistema responde 409 y el estado permanece en `rechazada`

#### Scenario: Cancelada es terminal
- **WHEN** se intenta aprobar, rechazar o cancelar una reserva en estado `cancelada`
- **THEN** el sistema responde 409 y el estado permanece en `cancelada`

### Requirement: Liberación del horario al rechazar o cancelar
Al pasar a `rechazada` o `cancelada`, la reserva SHALL dejar de bloquear el horario. El rango SHALL quedar disponible para nuevas solicitudes y la fila de la reserva SHALL conservarse.

#### Scenario: Rechazo libera el horario
- **WHEN** una solicitud pendiente es rechazada
- **THEN** una consulta de disponibilidad sobre su mismo espacio y rango responde 200 con `disponible` en verdadero

#### Scenario: Cancelación libera el horario
- **WHEN** una reserva pendiente o aprobada es cancelada
- **THEN** una consulta de disponibilidad sobre su mismo espacio y rango responde 200 con `disponible` en verdadero

#### Scenario: Nueva solicitud tras liberar el horario
- **WHEN** una reserva fue rechazada o cancelada y otro usuario solicita el mismo espacio y rango
- **THEN** el sistema responde 201 y crea una nueva solicitud pendiente

#### Scenario: La fila rechazada o cancelada permanece
- **WHEN** una reserva pasa a `rechazada` o `cancelada`
- **THEN** la reserva sigue existiendo con su identificador y su nuevo estado, sin eliminación física
