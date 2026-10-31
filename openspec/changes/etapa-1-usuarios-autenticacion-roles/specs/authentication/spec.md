# Spec Delta

## Purpose

Permite que un usuario demuestre su identidad mediante correo y contraseña, reciba un token JWT y acceda a los endpoints protegidos presentando ese token.

## ADDED Requirements

### Requirement: Inicio de sesión con token JWT
El sistema SHALL emitir un token JWT firmado con HS256 cuando un usuario activo presenta un correo y una contraseña válidos. La respuesta SHALL contener únicamente `access_token` y `token_type`. El token SHALL incluir `sub`, `iat` y `exp`, con vigencia configurable de 60 minutos por defecto, y SHALL no incluir el rol.

#### Scenario: Inicio de sesión exitoso
- **WHEN** un usuario activo presenta su correo y su contraseña correctos
- **THEN** el sistema devuelve un token JWT que identifica a esa cuenta mediante `sub`, con `iat` y `exp`, y responde únicamente con `access_token` y `token_type`

#### Scenario: Credenciales inválidas
- **WHEN** un usuario presenta un correo inexistente o una contraseña incorrecta
- **THEN** el sistema responde 401 y no emite token

#### Scenario: Cuenta inactiva
- **WHEN** una cuenta inactiva presenta credenciales correctas
- **THEN** el sistema responde 401 y no emite token

#### Scenario: Token sin rol
- **WHEN** se inspecciona el contenido del token emitido
- **THEN** el token contiene `sub`, `iat` y `exp`, y no contiene un claim de rol

### Requirement: Protección de endpoints con token
Los endpoints protegidos SHALL exigir un token JWT válido de una cuenta activa. La petición SHALL rechazarse con 401 cuando el token falta, es inválido, está expirado o la cuenta fue desactivada.

#### Scenario: Petición sin token
- **WHEN** se solicita un endpoint protegido sin token
- **THEN** el sistema responde 401

#### Scenario: Token inválido o manipulado
- **WHEN** se solicita un endpoint protegido con un token inválido o con la firma alterada
- **THEN** el sistema responde 401

#### Scenario: Token expirado
- **WHEN** se solicita un endpoint protegido con un token cuyo `exp` ya venció
- **THEN** el sistema responde 401

#### Scenario: Cuenta desactivada con token vigente
- **WHEN** se solicita un endpoint protegido con un token aún vigente de una cuenta que fue desactivada
- **THEN** el sistema responde 401

#### Scenario: Token válido
- **WHEN** se solicita un endpoint protegido con un token JWT válido de una cuenta activa que posee el permiso requerido
- **THEN** el sistema atiende la petición

### Requirement: Identificación del usuario autenticado
El sistema SHALL derivar del token la identidad del usuario y SHALL obtener el rol vigente de la cuenta, sin confiar en un rol o identificador enviado por el cliente.

#### Scenario: Rol tomado de la cuenta autenticada
- **WHEN** un usuario autenticado realiza una petición protegida
- **THEN** el sistema evalúa los permisos con el rol vigente de la cuenta y no con un valor enviado en la petición

#### Scenario: Rol actualizado tras emitir el token
- **WHEN** un Administrador cambia el rol de una cuenta que ya tenía un token emitido
- **THEN** las peticiones siguientes de esa cuenta se autorizan con el rol vigente y, si el nuevo rol carece del permiso, responden 403

### Requirement: Confidencialidad de las credenciales
El sistema SHALL no incluir la contraseña ni su hash en la respuesta de inicio de sesión ni en la información contenida en el token JWT.

#### Scenario: Respuesta de login sin contraseña
- **WHEN** un usuario inicia sesión correctamente
- **THEN** la respuesta contiene únicamente `access_token` y `token_type`, sin contraseña ni hash

#### Scenario: Contenido del token sin contraseña
- **WHEN** se inspecciona el contenido del token emitido
- **THEN** el token no contiene la contraseña ni el hash almacenado
