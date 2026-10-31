# Spec Delta

## Purpose

Permite a cualquier usuario autenticado crear una solicitud de reserva sobre un espacio académico, aplicando las reglas de tiempo, horario y fecha del plan y dejándola en estado pendiente.

## ADDED Requirements

### Requirement: Crear solicitud de reserva pendiente
El sistema SHALL permitir a cualquier usuario autenticado con rol Solicitante, Coordinador o Administrador crear una solicitud de reserva para un espacio. La solicitud SHALL nacer en estado `pendiente`, con el solicitante tomado del usuario autenticado y sin aceptar estado ni solicitante enviados por el cliente.

#### Scenario: Creación exitosa de la solicitud
- **WHEN** un usuario autenticado envía un espacio existente, una fecha futura y un rango válido dentro del horario de operación
- **THEN** el sistema responde 201 con la solicitud creada, estado `pendiente` y el usuario autenticado como solicitante

#### Scenario: Solicitante tomado del token
- **WHEN** un usuario autenticado envía en el cuerpo un solicitante distinto de su propia cuenta
- **THEN** el sistema responde 201 y registra como solicitante al usuario autenticado, ignorando el valor enviado

#### Scenario: Cualquier rol autenticado puede crear
- **WHEN** un usuario con rol Solicitante, Coordinador o Administrador crea una solicitud válida
- **THEN** el sistema responde 201 y persiste la solicitud

#### Scenario: Creación sin autenticación
- **WHEN** se intenta crear una solicitud sin token
- **THEN** el sistema responde 401 y no crea la solicitud

#### Scenario: Espacio inexistente
- **WHEN** un usuario autenticado intenta reservar un espacio que no existe
- **THEN** el sistema responde 404 y no crea la solicitud

#### Scenario: Espacio inactivo
- **WHEN** un usuario autenticado intenta reservar un espacio marcado como inactivo
- **THEN** el sistema responde 409 y no crea la solicitud

### Requirement: Bloques de 30 minutos y duración mínima
El inicio y el fin de una solicitud SHALL estar alineados a bloques de 30 minutos. La duración SHALL ser de al menos 30 minutos. El sistema SHALL no imponer una duración máxima.

#### Scenario: Duración mínima de 30 minutos
- **WHEN** un usuario autenticado solicita un rango de 30 minutos alineado a bloques
- **THEN** el sistema responde 201 y crea la solicitud

#### Scenario: Inicio no alineado a 30 minutos
- **WHEN** un usuario autenticado envía un inicio con minutos distintos de 00 o 30
- **THEN** el sistema responde 422 y no crea la solicitud

#### Scenario: Fin no alineado a 30 minutos
- **WHEN** un usuario autenticado envía un fin con minutos distintos de 00 o 30
- **THEN** el sistema responde 422 y no crea la solicitud

#### Scenario: Duración menor a 30 minutos
- **WHEN** un usuario autenticado envía un fin que no supera el inicio en al menos 30 minutos
- **THEN** el sistema responde 422 y no crea la solicitud

#### Scenario: Sin duración máxima
- **WHEN** un usuario autenticado solicita un rango de varias horas alineado a bloques y dentro del horario de operación
- **THEN** el sistema responde 201 y no aplica un límite de duración máxima

### Requirement: Horario global y fechas futuras
La solicitud SHALL quedar dentro de un único horario global de operación y SHALL usar una fecha futura. El horario SHALL ser configurable y su valor propuesto por defecto es de 07:00 a 21:00, sin reservas que crucen de un día a otro.

#### Scenario: Inicio exactamente en la apertura
- **WHEN** un usuario autenticado solicita un rango que comienza exactamente a la hora de apertura y termina antes del cierre
- **THEN** el sistema responde 201 y crea la solicitud

#### Scenario: Fin exactamente en el cierre
- **WHEN** un usuario autenticado solicita un rango que termina exactamente a la hora de cierre
- **THEN** el sistema responde 201 y crea la solicitud

#### Scenario: Inicio antes de la apertura
- **WHEN** un usuario autenticado solicita un rango que comienza antes de la hora de apertura
- **THEN** el sistema responde 422 y no crea la solicitud

#### Scenario: Fin después del cierre
- **WHEN** un usuario autenticado solicita un rango que termina después de la hora de cierre
- **THEN** el sistema responde 422 y no crea la solicitud

#### Scenario: Fecha pasada
- **WHEN** un usuario autenticado solicita un rango cuyo inicio ya ocurrió
- **THEN** el sistema responde 422 y no crea la solicitud

#### Scenario: Reserva del mismo día en una hora futura
- **WHEN** un usuario autenticado solicita un rango del día actual cuyo inicio es posterior al momento de la solicitud
- **THEN** el sistema responde 201 y crea la solicitud

#### Scenario: Rango que cruza de día
- **WHEN** un usuario autenticado solicita un rango cuyo fin corresponde al día siguiente
- **THEN** el sistema responde 422 y no crea la solicitud
