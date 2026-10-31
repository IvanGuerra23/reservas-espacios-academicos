# Spec Delta

## Purpose

Permite consultar si un espacio académico está disponible para un rango de tiempo, considerando únicamente las reservas que bloquean el horario y la semántica de intervalos del plan.

## ADDED Requirements

### Requirement: Consultar disponibilidad de un espacio
El sistema SHALL permitir a cualquier usuario autenticado consultar si un espacio está disponible para un rango de inicio y fin. La respuesta SHALL incluir el espacio, el rango consultado y un indicador `disponible` en verdadero cuando no exista una reserva pendiente o aprobada que solape el rango.

#### Scenario: Rango disponible
- **WHEN** un usuario autenticado consulta un espacio y un rango válido sin reservas pendientes o aprobadas que lo solapen
- **THEN** el sistema responde 200 con `disponible` en verdadero

#### Scenario: Rango ocupado por una solicitud pendiente
- **WHEN** un usuario autenticado consulta un rango que solapa una reserva en estado `pendiente`
- **THEN** el sistema responde 200 con `disponible` en falso

#### Scenario: Rango ocupado por una reserva aprobada
- **WHEN** un usuario autenticado consulta un rango que solapa una reserva en estado `aprobada`
- **THEN** el sistema responde 200 con `disponible` en falso

#### Scenario: Reservas rechazadas o canceladas no bloquean
- **WHEN** un usuario autenticado consulta un rango que solapa únicamente reservas en estado `rechazada` o `cancelada`
- **THEN** el sistema responde 200 con `disponible` en verdadero

#### Scenario: Cualquier rol autenticado consulta disponibilidad
- **WHEN** un usuario con rol Solicitante, Coordinador o Administrador consulta la disponibilidad de un espacio
- **THEN** el sistema responde 200 con el resultado de disponibilidad

#### Scenario: Consulta sin autenticación
- **WHEN** se consulta la disponibilidad sin token
- **THEN** el sistema responde 401

#### Scenario: Consulta de un espacio inexistente
- **WHEN** un usuario autenticado consulta la disponibilidad de un identificador que no corresponde a ningún espacio
- **THEN** el sistema responde 404

#### Scenario: Consulta de un espacio inactivo
- **WHEN** un usuario autenticado consulta la disponibilidad de un espacio marcado como inactivo
- **THEN** el sistema responde 409 y no evalúa la disponibilidad

#### Scenario: Consulta con rango inválido
- **WHEN** un usuario autenticado envía un fin que no es posterior al inicio, un rango no alineado a 30 minutos, un rango fuera del horario de operación o un rango en el pasado
- **THEN** el sistema responde 422

### Requirement: Semántica de intervalos y estados bloqueantes
El sistema SHALL interpretar los rangos como `[inicio, fin)` y SHALL considerar bloqueantes únicamente los estados `pendiente` y `aprobada`. Dos rangos contiguos no SHALL considerarse solapados.

#### Scenario: Rangos contiguos no se solapan
- **WHEN** existe una reserva pendiente de 09:00 a 10:00 y se consulta el rango de 10:00 a 11:00 del mismo espacio
- **THEN** el sistema responde 200 con `disponible` en verdadero

#### Scenario: Solapamiento parcial en el inicio
- **WHEN** existe una reserva pendiente de 09:00 a 10:00 y se consulta el rango de 09:30 a 10:30 del mismo espacio
- **THEN** el sistema responde 200 con `disponible` en falso

#### Scenario: Solapamiento que contiene la reserva existente
- **WHEN** existe una reserva aprobada de 09:00 a 10:00 y se consulta el rango de 08:00 a 11:00 del mismo espacio
- **THEN** el sistema responde 200 con `disponible` en falso

#### Scenario: Otro espacio no afecta la disponibilidad
- **WHEN** existe una reserva pendiente en un espacio y se consulta otro espacio para el mismo rango
- **THEN** el sistema responde 200 con `disponible` en verdadero
