# Spec Delta

## Purpose

Garantiza que dos reservas pendientes o aprobadas del mismo espacio no puedan solaparse, incluso bajo solicitudes concurrentes, apoyándose en una restricción de exclusión de PostgreSQL además de la validación del servicio.

## ADDED Requirements

### Requirement: Restricción de exclusión en la base de datos
La base de datos SHALL impedir que existan dos reservas en estado `pendiente` o `aprobada` con rangos `[inicio, fin)` solapados para el mismo espacio, mediante una restricción de exclusión. La restricción SHALL no considerar bloqueantes los estados `rechazada` y `cancelada`.

#### Scenario: Inserción solapada con una reserva pendiente
- **WHEN** se intenta insertar directamente una reserva pendiente o aprobada que solapa otra reserva pendiente del mismo espacio
- **THEN** la base de datos rechaza la inserción por violación de la restricción de exclusión

#### Scenario: Inserción solapada con una reserva aprobada
- **WHEN** se intenta insertar directamente una reserva pendiente o aprobada que solapa otra reserva aprobada del mismo espacio
- **THEN** la base de datos rechaza la inserción por violación de la restricción de exclusión

#### Scenario: Rangos contiguos permitidos en la base de datos
- **WHEN** se inserta una reserva que empieza exactamente cuando termina otra reserva bloqueante del mismo espacio
- **THEN** la base de datos acepta la inserción

#### Scenario: Mismo rango en espacios distintos
- **WHEN** se insertan dos reservas con el mismo rango pero espacios diferentes
- **THEN** la base de datos acepta ambas inserciones

#### Scenario: Reservas no bloqueantes no impiden insertar
- **WHEN** existe una reserva `rechazada` o `cancelada` y se inserta una nueva reserva pendiente o aprobada que solapa su rango en el mismo espacio
- **THEN** la base de datos acepta la nueva inserción

### Requirement: Validación de solapamiento en el servicio
El servicio SHALL comprobar el solapamiento antes de persistir una solicitud y SHALL responder 409 cuando exista una reserva pendiente o aprobada que solape el rango. Si la restricción de base de datos rechaza la operación por una carrera, el sistema SHALL responder 409 y no crear la reserva.

#### Scenario: El servicio detecta un solapamiento
- **WHEN** un usuario autenticado crea una solicitud que solapa una reserva pendiente o aprobada del mismo espacio
- **THEN** el sistema responde 409 y no crea la solicitud

#### Scenario: La base de datos rechaza una carrera
- **WHEN** dos solicitudes superan la comprobación del servicio y una de ellas es rechazada por la restricción de exclusión
- **THEN** el sistema responde 409 a la solicitud rechazada y no crea una reserva adicional

### Requirement: Garantía bajo concurrencia
El sistema SHALL garantizar que dos solicitudes simultáneas sobre el mismo espacio y rango no produzcan dos reservas bloqueantes.

#### Scenario: Dos solicitudes simultáneas sobre el mismo rango
- **WHEN** dos solicitudes se envían simultáneamente para el mismo espacio y el mismo rango válido
- **THEN** una responde 201, la otra responde 409 y la base de datos contiene una sola reserva para ese rango

#### Scenario: Dos solicitudes simultáneas para espacios distintos
- **WHEN** dos solicitudes se envían simultáneamente para espacios distintos con el mismo rango válido
- **THEN** ambas responden 201 y la base de datos contiene las dos reservas
