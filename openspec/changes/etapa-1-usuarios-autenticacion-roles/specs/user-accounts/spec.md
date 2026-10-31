# Spec Delta

## Purpose

Define el registro y la administración de las cuentas de usuario del sistema, sus datos, el almacenamiento seguro de la contraseña y el estado activo de la cuenta.

## ADDED Requirements

### Requirement: Registro público de usuario
El sistema SHALL ofrecer registro público y SHALL crear toda cuenta registrada con rol Solicitante y estado activo. Solo un Administrador SHALL poder asignar los roles Coordinador o Administrador. El registro SHALL rechazar un correo ya registrado, sin distinguir mayúsculas de minúsculas.

#### Scenario: Registro exitoso
- **WHEN** una persona envía un nombre, un correo no registrado y una contraseña de al menos 8 caracteres
- **THEN** el sistema crea una cuenta activa con rol Solicitante y no devuelve la contraseña en la respuesta

#### Scenario: Correo duplicado
- **WHEN** una persona intenta registrarse con un correo que ya pertenece a una cuenta
- **THEN** el sistema responde 409 e informa que el correo ya está en uso

#### Scenario: Correo duplicado con distinto uso de mayúsculas
- **WHEN** una persona intenta registrarse con un correo que solo difiere en mayúsculas o minúsculas de una cuenta existente
- **THEN** el sistema responde 409 e informa que el correo ya está en uso

#### Scenario: Datos obligatorios incompletos
- **WHEN** una persona envía el registro sin nombre, sin correo o sin contraseña
- **THEN** el sistema responde 422 y señala los datos faltantes

### Requirement: Longitud mínima de contraseña
El sistema SHALL exigir que toda contraseña tenga al menos 8 caracteres.

#### Scenario: Contraseña demasiado corta
- **WHEN** una persona intenta registrarse con una contraseña de menos de 8 caracteres
- **THEN** el sistema responde 422 y rechaza el registro

#### Scenario: Contraseña con longitud mínima
- **WHEN** una persona se registra con una contraseña de exactamente 8 caracteres
- **THEN** el sistema acepta el registro

### Requirement: Almacenamiento seguro de la contraseña
El sistema SHALL almacenar la contraseña únicamente como hash y SHALL comparar la contraseña presentada con el hash almacenado durante la autenticación, sin exponer la contraseña ni su hash en las respuestas.

#### Scenario: Contraseña persistida con hash
- **WHEN** se crea una cuenta con una contraseña
- **THEN** el valor persistido de la contraseña es un hash y no el texto plano

#### Scenario: Hash no expuesto
- **WHEN** se consultan los datos de una cuenta
- **THEN** la respuesta no incluye la contraseña ni el hash almacenado

### Requirement: Datos de la cuenta
El sistema SHALL mantener para cada usuario un identificador, nombre, correo, hash de contraseña, rol y estado activo, conforme al modelo de datos del plan.

#### Scenario: Consulta de la propia cuenta
- **WHEN** un usuario autenticado consulta su propia cuenta
- **THEN** el sistema devuelve identificador, nombre, correo, rol y estado activo de esa cuenta

### Requirement: Inicialización del primer Administrador
El sistema SHALL proporcionar un comando de inicialización idempotente que crea la primera cuenta con rol Administrador a partir de variables de entorno, solo cuando no existe ninguna cuenta con ese rol.

#### Scenario: Creación del primer Administrador
- **WHEN** se ejecuta el comando de inicialización con las variables de entorno configuradas y no existe ninguna cuenta con rol Administrador
- **THEN** el sistema crea una cuenta activa con rol Administrador y contraseña almacenada como hash

#### Scenario: Ejecución repetida del comando
- **WHEN** el comando de inicialización se ejecuta de nuevo después de haber creado el primer Administrador
- **THEN** el sistema no crea cuentas duplicadas ni modifica la cuenta existente

#### Scenario: Ya existe un Administrador
- **WHEN** el comando de inicialización se ejecuta y ya existe al menos una cuenta con rol Administrador
- **THEN** el sistema no crea ninguna cuenta nueva

### Requirement: Administración de usuarios por el Administrador
El sistema SHALL permitir al Administrador listar usuarios, consultar una cuenta por identificador, editar nombre o correo, asignar rol, activar y desactivar cuentas. Los roles Solicitante y Coordinador SHALL recibir 403 al intentar estas operaciones.

#### Scenario: Administrador lista usuarios
- **WHEN** un Administrador autenticado solicita la lista de usuarios
- **THEN** el sistema responde 200 con las cuentas registradas, sus datos y su rol

#### Scenario: Administrador consulta una cuenta por identificador
- **WHEN** un Administrador autenticado solicita una cuenta por su identificador
- **THEN** el sistema responde 200 con los datos de esa cuenta

#### Scenario: Administrador edita nombre o correo
- **WHEN** un Administrador autenticado modifica el nombre o el correo de una cuenta
- **THEN** el sistema responde 200 y persiste los cambios

#### Scenario: Administrador edita el correo a uno duplicado
- **WHEN** un Administrador autenticado intenta cambiar el correo de una cuenta a uno que ya pertenece a otra
- **THEN** el sistema responde 409 e informa que el correo ya está en uso

#### Scenario: Administrador asigna un rol
- **WHEN** un Administrador cambia el rol de una cuenta a Coordinador o Administrador
- **THEN** el sistema responde 200, persiste el nuevo rol y las peticiones posteriores de esa cuenta usan el rol actualizado

#### Scenario: Administrador desactiva una cuenta
- **WHEN** un Administrador marca una cuenta como inactiva
- **THEN** el sistema responde 200 y la cuenta conserva sus datos y queda inactiva

#### Scenario: Administrador reactiva una cuenta
- **WHEN** un Administrador marca como activa una cuenta inactiva
- **THEN** el sistema responde 200 y la cuenta vuelve al estado activo

#### Scenario: Rol no autorizado intenta administrar usuarios
- **WHEN** un usuario con rol Solicitante o Coordinador intenta listar, consultar, editar, asignar rol, activar o desactivar usuarios
- **THEN** el sistema responde 403

### Requirement: Efecto del estado activo
Una cuenta inactiva SHALL conservar sus datos y su historial, pero SHALL no poder iniciar sesión ni acceder a endpoints protegidos.

#### Scenario: Cuenta inactiva intenta autenticarse
- **WHEN** una cuenta inactiva presenta credenciales correctas
- **THEN** el sistema responde 401 y no emite token
