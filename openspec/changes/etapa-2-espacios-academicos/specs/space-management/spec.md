# Spec Delta

## Purpose

Permite administrar el catálogo de espacios académicos —laboratorios, auditorios y salas de cómputo— y consultarlo, aplicando la política de permisos de la Etapa 1.

## ADDED Requirements

### Requirement: Crear espacio académico
El sistema SHALL permitir al Administrador crear un espacio con nombre, tipo, capacidad, ubicación y estado activo inicial verdadero. El tipo SHALL ser `laboratorio`, `auditorio` o `sala_computo`. Los roles Solicitante y Coordinador SHALL recibir 403 y las peticiones sin token 401.

#### Scenario: Creación exitosa
- **WHEN** un Administrador autenticado envía un nombre, un tipo permitido, una capacidad y una ubicación
- **THEN** el sistema responde 201 con el espacio creado, su identificador y `activo` en verdadero

#### Scenario: Creación con rol no autorizado
- **WHEN** un usuario autenticado con rol Solicitante o Coordinador intenta crear un espacio
- **THEN** el sistema responde 403 y no crea el espacio

#### Scenario: Creación sin autenticación
- **WHEN** se intenta crear un espacio sin token
- **THEN** el sistema responde 401 y no crea el espacio

#### Scenario: Creación con datos inválidos
- **WHEN** un Administrador envía un tipo distinto de los tres permitidos o datos inválidos según las validaciones del modelo
- **THEN** el sistema responde 422 y no crea el espacio

### Requirement: Editar espacio académico
El sistema SHALL permitir al Administrador editar el nombre, el tipo, la capacidad y la ubicación de un espacio existente. La edición SHALL no modificar el estado activo. Los roles Solicitante y Coordinador SHALL recibir 403.

#### Scenario: Edición exitosa
- **WHEN** un Administrador autenticado modifica el nombre, el tipo, la capacidad o la ubicación de un espacio existente
- **THEN** el sistema responde 200 y persiste los cambios sin alterar el estado activo

#### Scenario: Edición de un espacio inexistente
- **WHEN** un Administrador autenticado intenta editar un identificador que no corresponde a ningún espacio
- **THEN** el sistema responde 404

#### Scenario: Edición con rol no autorizado
- **WHEN** un usuario autenticado con rol Solicitante o Coordinador intenta editar un espacio
- **THEN** el sistema responde 403 y no modifica el espacio

#### Scenario: Edición con datos inválidos
- **WHEN** un Administrador envía en la edición un tipo no permitido o datos inválidos según las validaciones del modelo
- **THEN** el sistema responde 422 y no modifica el espacio

#### Scenario: Edición sin autenticación
- **WHEN** se intenta editar un espacio sin token
- **THEN** el sistema responde 401 y no modifica el espacio

### Requirement: Listar espacios académicos
El sistema SHALL permitir a cualquier usuario autenticado consultar el listado de espacios académicos, incluyendo los campos del modelo y el estado activo.

#### Scenario: Listado por usuario autenticado
- **WHEN** un usuario autenticado con rol Solicitante, Coordinador o Administrador solicita el listado de espacios
- **THEN** el sistema responde 200 con los espacios registrados y su estado activo

#### Scenario: Listado sin autenticación
- **WHEN** se solicita el listado de espacios sin token
- **THEN** el sistema responde 401

### Requirement: Consultar el detalle de un espacio
El sistema SHALL permitir a cualquier usuario autenticado consultar el detalle de un espacio por su identificador, incluyendo nombre, tipo, capacidad, ubicación y estado activo.

#### Scenario: Detalle por usuario autenticado
- **WHEN** un usuario autenticado con rol Solicitante, Coordinador o Administrador solicita un espacio existente por su identificador
- **THEN** el sistema responde 200 con los datos del espacio

#### Scenario: Detalle de un espacio inexistente
- **WHEN** un usuario autenticado solicita un identificador que no corresponde a ningún espacio
- **THEN** el sistema responde 404

#### Scenario: Detalle sin autenticación
- **WHEN** se solicita el detalle de un espacio sin token
- **THEN** el sistema responde 401

### Requirement: Desactivar espacio académico
El sistema SHALL permitir al Administrador desactivar un espacio existente. La desactivación SHALL cambiar el estado a inactivo y SHALL no eliminar el registro. Los roles Solicitante y Coordinador SHALL recibir 403.

#### Scenario: Desactivación exitosa
- **WHEN** un Administrador autenticado desactiva un espacio existente
- **THEN** el sistema responde 200, conserva el registro y devuelve `activo` en falso

#### Scenario: Desactivación de un espacio inexistente
- **WHEN** un Administrador autenticado intenta desactivar un identificador que no corresponde a ningún espacio
- **THEN** el sistema responde 404

#### Scenario: Desactivación con rol no autorizado
- **WHEN** un usuario autenticado con rol Solicitante o Coordinador intenta desactivar un espacio
- **THEN** el sistema responde 403 y no modifica el espacio

#### Scenario: Desactivación sin autenticación
- **WHEN** se intenta desactivar un espacio sin token
- **THEN** el sistema responde 401 y no modifica el espacio
