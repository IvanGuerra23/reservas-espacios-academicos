# Spec Delta

## Purpose

Define los roles Solicitante, Coordinador y Administrador y convierte la matriz de permisos de la sección 2.5 del plan en la política autoritativa que el sistema aplica a cada operación.

## ADDED Requirements

### Requirement: Roles del sistema
El sistema SHALL reconocer exactamente tres roles: Solicitante, Coordinador y Administrador. Toda cuenta SHALL tener asignado uno de esos roles.

#### Scenario: Cuenta con rol válido
- **WHEN** se crea o actualiza una cuenta con uno de los tres roles definidos
- **THEN** el sistema acepta el rol y lo persiste

#### Scenario: Rol desconocido
- **WHEN** se intenta asignar a una cuenta un rol distinto de Solicitante, Coordinador o Administrador
- **THEN** el sistema responde 422 y no persiste el rol

### Requirement: La matriz de permisos es autoritativa
El sistema SHALL evaluar las nueve operaciones de la matriz de la sección 2.5 contra los roles autorizados y SHALL denegar con 403 toda operación no permitida. La política SHALL ser la única fuente de autorización y no SHALL conceder permisos adicionales.

#### Scenario: Política de consultar espacios y disponibilidad
- **WHEN** la política de permisos evalúa `consultar_espacios` para Solicitante, Coordinador y Administrador
- **THEN** autoriza la operación para los tres roles

#### Scenario: Política de crear solicitud de reserva
- **WHEN** la política de permisos evalúa `crear_reserva` para Solicitante, Coordinador y Administrador
- **THEN** autoriza la operación para los tres roles

#### Scenario: Política de cancelar una reserva propia
- **WHEN** la política de permisos evalúa `cancelar_reserva_propia` para Solicitante, Coordinador y Administrador
- **THEN** autoriza la operación para los tres roles

#### Scenario: Política de cancelar una reserva ajena
- **WHEN** la política de permisos evalúa `cancelar_reserva_ajena` para Solicitante, Coordinador y Administrador
- **THEN** deniega la operación para Solicitante, con respuesta 403 al aplicarse a una petición, y la autoriza para Coordinador y Administrador

#### Scenario: Política de ver el historial propio
- **WHEN** la política de permisos evalúa `ver_historial_propio` para Solicitante, Coordinador y Administrador
- **THEN** autoriza la operación para los tres roles

#### Scenario: Política de ver el historial de todos
- **WHEN** la política de permisos evalúa `ver_historial_global` para Solicitante, Coordinador y Administrador
- **THEN** deniega la operación para Solicitante, con respuesta 403 al aplicarse a una petición, y la autoriza para Coordinador y Administrador

#### Scenario: Política de aprobar o rechazar solicitudes
- **WHEN** la política de permisos evalúa `aprobar_rechazar` para Solicitante, Coordinador y Administrador
- **THEN** deniega la operación para Solicitante, con respuesta 403 al aplicarse a una petición, y la autoriza para Coordinador y Administrador

#### Scenario: Política de gestionar espacios
- **WHEN** la política de permisos evalúa `gestionar_espacios` para Solicitante, Coordinador y Administrador
- **THEN** deniega la operación para Solicitante y Coordinador, con respuesta 403 al aplicarse a una petición, y la autoriza únicamente para Administrador

#### Scenario: Política de gestionar usuarios y asignar roles
- **WHEN** la política de permisos evalúa `gestionar_usuarios` para Solicitante, Coordinador y Administrador
- **THEN** deniega la operación para Solicitante y Coordinador, con respuesta 403 al aplicarse a una petición, y la autoriza únicamente para Administrador
