# Spec Delta

## Purpose

Establece una base ejecutable y reproducible para la API de reservas: aplicación FastAPI modular, PostgreSQL en Docker Compose, configuración por entorno y suite de pruebas automatizadas.

## ADDED Requirements

### Requirement: Arranque reproducible con Docker Compose
El sistema SHALL levantarse con un único comando de Docker Compose que inicie la aplicación y PostgreSQL, de modo que un clon limpio del proyecto quede operativo sin pasos manuales sobre la base de datos.

#### Scenario: Entorno nuevo operativo
- **WHEN** se ejecuta el comando de Docker Compose sobre un clon limpio con las variables de entorno configuradas
- **THEN** la aplicación y PostgreSQL inician y la API responde a peticiones HTTP

#### Scenario: Conexión a PostgreSQL al arrancar
- **WHEN** la aplicación inicia con PostgreSQL disponible
- **THEN** la aplicación establece conexión con la base de datos configurada y queda lista para atender peticiones

### Requirement: Estructura modular en capas
El sistema SHALL organizarse en los módulos usuarios, espacios, reservas y aprobaciones, con separación entre rutas, servicios y acceso a datos. Las reglas de negocio SHALL residir en la capa de servicios.

#### Scenario: Separación de responsabilidades por módulo
- **WHEN** se inspecciona la estructura de un módulo
- **THEN** sus rutas, servicios y acceso a datos están separados en componentes distintos y las reglas de negocio no están implementadas en las rutas

#### Scenario: Módulos de etapas futuras presentes como estructura
- **WHEN** se inspecciona el andamiaje del proyecto
- **THEN** existen los módulos usuarios, espacios, reservas y aprobaciones, aunque las etapas 2 a 4 añadan su comportamiento

### Requirement: Documentación interactiva de la API
El sistema SHALL exponer documentación interactiva de la API generada automáticamente por el framework, accesible desde el entorno de ejecución.

#### Scenario: Documentación disponible
- **WHEN** la aplicación está en ejecución y se solicita la ruta de documentación
- **THEN** la documentación interactiva de la API responde correctamente

### Requirement: Suite de pruebas ejecutable
El proyecto SHALL incluir una suite de pruebas con pytest organizada en pruebas unitarias, de integración y de concurrencia, ejecutable dentro del entorno del proyecto.

#### Scenario: Ejecución de la suite
- **WHEN** se ejecuta el comando de pruebas del proyecto
- **THEN** pytest descubre y ejecuta las pruebas de las carpetas unitarias, integración y concurrencia, e informa el resultado

#### Scenario: Pruebas de integración contra PostgreSQL
- **WHEN** se ejecutan las pruebas de integración con el entorno de Docker Compose activo
- **THEN** las pruebas usan una base de datos de pruebas separada dentro del mismo contenedor PostgreSQL y no la base de datos de desarrollo ni una base de datos en memoria con comportamiento distinto

### Requirement: Esquema de base de datos reproducible
El esquema de la base de datos SHALL crearse y versionarse mediante un mecanismo reproducible, definido en el diseño de esta etapa, que permita reconstruir el esquema desde cero.

#### Scenario: Reconstrucción del esquema
- **WHEN** se levanta un entorno nuevo con la base de datos vacía y se aplica el mecanismo de esquema definido
- **THEN** las tablas de la Etapa 1 quedan creadas y la aplicación puede operar sobre ellas

#### Scenario: Reaplicación sin duplicar cambios
- **WHEN** el mecanismo de esquema se ejecuta nuevamente sobre una base de datos ya actualizada
- **THEN** no se duplican tablas ni se corrompe el esquema existente
