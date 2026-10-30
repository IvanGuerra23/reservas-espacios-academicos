# Plan de Ingeniería de Software
## Sistema de Gestión de Reservas de Espacios Académicos

**Autor:** Ivan Guerra
**Fecha:** 30 de octubre de 2026
**Modalidad:** Individual
**Estado:** Etapa 1 del taller (planificación previa a las especificaciones y a la implementación)

---

## 1. Análisis del problema

### 1.1 Problemática

Una institución universitaria gestiona hoy las reservas de sus espacios académicos (laboratorios, auditorios y salas de cómputo) mediante correos electrónicos y hojas de cálculo. Este mecanismo produce tres problemas:

- **Duplicidad de solicitudes:** varias personas piden el mismo espacio sin saberlo.
- **Conflictos de horarios:** dos reservas pueden terminar solapadas porque nada lo impide.
- **Dificultad para consultar la disponibilidad:** no existe un único lugar donde ver qué espacios están libres.

### 1.2 Necesidades de la institución

1. Centralizar las solicitudes y las reservas en un solo sistema.
2. Consultar la disponibilidad de los espacios de forma confiable.
3. Garantizar que no existan dos reservas solapadas para el mismo espacio.
4. Controlar quién puede solicitar, aprobar y administrar.
5. Conservar un historial de las reservas.

### 1.3 Actores involucrados

| Actor | Descripción | Qué hace en el sistema |
|---|---|---|
| **Solicitante** | Estudiante o docente que necesita un espacio | Consulta disponibilidad, crea solicitudes, cancela las suyas y ve su historial |
| **Coordinador** | Persona que revisa las solicitudes | Aprueba o rechaza solicitudes y ve las reservas de todos |
| **Administrador** | Responsable del sistema | Gestiona espacios y usuarios, asigna roles |

### 1.4 Alcance y limitaciones

**Dentro del alcance:**

- Gestión de espacios académicos.
- Registro y administración de usuarios.
- Roles y permisos.
- Creación de solicitudes de reserva.
- Validación de disponibilidad.
- Prevención de reservas simultáneas.
- Aprobación y rechazo de solicitudes.
- Cancelación e historial de reservas.
- Autenticación mediante token (JWT), necesaria para que los roles tengan sentido.

**Fuera del alcance** (decisión deliberada para mantener el proyecto simple y centrado en el proceso de ingeniería):

- Interfaz gráfica (el sistema se expone solo como API).
- Notificaciones por correo electrónico.
- Reservas recurrentes (por ejemplo, "todos los lunes").
- Integración con sistemas externos de la universidad.
- Recuperación de contraseña y renovación de tokens.
- Horarios de operación distintos por espacio.

---

## 2. Propuesta de solución

### 2.1 Descripción general

Se construirá una **API REST** que centraliza la gestión de espacios, usuarios y reservas. Cada solicitud de reserva pasa por un ciclo de vida controlado, y la base de datos garantiza que dos reservas activas no puedan solaparse sobre el mismo espacio.

### 2.2 Funcionalidades prioritarias

**Núcleo (máxima prioridad):** creación de solicitudes con validación de disponibilidad y prevención de reservas simultáneas. Es lo que resuelve el problema central de la institución.

**Soporte necesario:** usuarios, autenticación y roles; gestión de espacios; aprobación y rechazo; cancelación e historial.

### 2.2.1 Ciclo de vida de una reserva

```
            ┌──────────► APROBADA ──┐
PENDIENTE ──┤                       ├──► CANCELADA
            └──────────► RECHAZADA  │
                                    │
PENDIENTE ──────────────────────────┘
```

- Una solicitud nace **pendiente**.
- Un coordinador (o administrador) la pasa a **aprobada** o **rechazada**.
- Una reserva **pendiente** o **aprobada** puede pasar a **cancelada**.
- Las reservas **pendientes** y **aprobadas** bloquean el horario. Las **rechazadas** y **canceladas** lo liberan, pero se conservan en el historial.

### 2.3 Organización en módulos

| Módulo | Responsabilidad |
|---|---|
| **Usuarios y autenticación** | Registro, inicio de sesión (JWT), roles y permisos |
| **Espacios** | Alta, edición, consulta y desactivación de espacios académicos |
| **Reservas** | Creación de solicitudes, validación de disponibilidad, prevención de simultáneas, cancelación e historial |
| **Aprobaciones** | Aprobación y rechazo de solicitudes |

### 2.4 Reglas de negocio iniciales

| # | Regla |
|---|---|
| R1 | Las solicitudes pendientes y aprobadas bloquean el horario del espacio |
| R2 | Las reservas se hacen en bloques de 30 minutos, con una duración mínima de 30 minutos |
| R3 | Existe un único horario global de operación para todos los espacios (por ejemplo, de 7:00 a 21:00) y solo se puede reservar en fechas futuras |
| R4 | Para rechazar una solicitud es obligatorio indicar un motivo |
| R5 | El solicitante puede cancelar sus propias reservas mientras no hayan comenzado; el coordinador y el administrador pueden cancelar cualquiera |
| R6 | Nada se elimina: la cancelación y el rechazo solo cambian el estado, para conservar el historial |

### 2.5 Matriz inicial de permisos

| Operación | Solicitante | Coordinador | Administrador |
|---|:---:|:---:|:---:|
| Consultar espacios y disponibilidad | ✓ | ✓ | ✓ |
| Crear solicitud de reserva | ✓ | ✓ | ✓ |
| Cancelar una reserva propia | ✓ | ✓ | ✓ |
| Cancelar una reserva ajena | ✗ | ✓ | ✓ |
| Ver historial propio | ✓ | ✓ | ✓ |
| Ver historial de todos | ✗ | ✓ | ✓ |
| Aprobar o rechazar solicitudes | ✗ | ✓ | ✓ |
| Gestionar espacios | ✗ | ✗ | ✓ |
| Gestionar usuarios y asignar roles | ✗ | ✗ | ✓ |

### 2.6 Puntos abiertos para la revisión de especificaciones

Estos puntos se resolverán durante la Fase 2 (revisión de especificaciones), porque no cambian la arquitectura:

1. Cómo se registran los usuarios (auto-registro como solicitante o creación solo por el administrador).
2. Manejo de zona horaria de las fechas y horas.
3. Si un coordinador puede aprobar sus propias solicitudes.
4. Qué ocurre con las reservas futuras de un espacio que se desactiva.

### 2.7 Estrategia de desarrollo progresivo

El sistema se construye en cinco etapas incrementales, descritas en la sección 4.

---

## 3. Decisiones de arquitectura y tecnología

### 3.1 Arquitectura seleccionada

**Monolito modular en capas, expuesto como API REST.**

- **Monolito modular:** una sola aplicación dividida internamente en módulos (usuarios, espacios, reservas, aprobaciones).
- **Capas dentro de cada módulo:**
  - *Rutas:* reciben las peticiones HTTP y validan los datos de entrada.
  - *Servicios:* aplican las reglas de negocio (disponibilidad, estados, permisos).
  - *Acceso a datos:* se comunica con PostgreSQL.

**Justificación:**

- Es proporcional a un proyecto individual con plazo corto.
- Aísla las reglas de negocio en la capa de servicios, lo que facilita incorporar cambios funcionales posteriores sin modificar el resto del sistema.
- Da a la IA una estructura clara sobre dónde va cada cosa, lo que reduce desvíos al implementar las especificaciones.

### 3.2 Stack tecnológico

| Componente | Tecnología |
|---|---|
| Lenguaje | Python |
| Framework web | FastAPI |
| Base de datos | PostgreSQL |
| Acceso a datos | SQLAlchemy |
| Autenticación | JWT |
| Pruebas | pytest |
| Entorno | Docker Compose |
| Control de versiones | Git y GitHub |
| Especificaciones | OpenSpec |
| Agente de programación | Opencode |

### 3.3 Estrategia de persistencia de datos

Se usa **PostgreSQL**, levantado mediante Docker Compose para que el proyecto sea reproducible con un solo comando.

**Modelo de datos conceptual:**

| Entidad | Campos principales |
|---|---|
| Usuario | identificador, nombre, correo, contraseña (almacenada con hash), rol, activo |
| Espacio | identificador, nombre, tipo (laboratorio, auditorio, sala de cómputo), capacidad, ubicación, activo |
| Reserva | identificador, espacio, solicitante, inicio, fin, estado, motivo de rechazo, revisor, fechas de creación y actualización |

**Prevención de reservas simultáneas:**

El problema es una *condición de carrera*: si dos peticiones consultan la disponibilidad al mismo tiempo, ambas pueden ver el horario libre y crear reservas solapadas. Validar solo en el código de la aplicación no es suficiente.

La decisión es que **sea la propia base de datos la que garantice la no superposición**, mediante una restricción de exclusión de PostgreSQL sobre el rango de tiempo de cada reserva, por espacio, aplicada únicamente a las reservas en estados que bloquean el horario (pendiente y aprobada). Los rangos se tratan como "inicio incluido, fin excluido", de modo que una reserva que termina a las 11:00 no choca con otra que empieza a las 11:00.

Los detalles exactos de esta restricción se validarán al redactar las especificaciones y se comprobarán con pruebas automatizadas.

La forma de crear y versionar el esquema de la base de datos se definirá en la Etapa 1, con el criterio de usar lo más simple que permita crear esa restricción de forma reproducible.

### 3.4 Organización de componentes

Estructura propuesta (puede ajustarse en la Etapa 1):

```
/
├── PLAN_INGENIERIA.md
├── README.md
├── INFORME_FINAL.md
├── docker-compose.yml
├── openspec/              # especificaciones, propuestas y tareas
├── app/
│   ├── main.py
│   ├── core/              # configuración, seguridad, conexión a la base de datos
│   ├── usuarios/          # rutas, servicios, acceso a datos
│   ├── espacios/
│   ├── reservas/
│   └── aprobaciones/
└── tests/
    ├── unitarias/
    ├── integracion/
    └── concurrencia/
```

### 3.5 Decisiones técnicas y alternativas descartadas

| Decisión | Alternativa descartada | Por qué se eligió esta opción |
|---|---|---|
| **FastAPI** como framework | Django | FastAPI expone una arquitectura explícita y por capas donde cada requisito se mapea a una función o endpoint concreto, lo que favorece la trazabilidad y la revisión del código generado. Django aporta muchas piezas ya resueltas (usuarios, permisos, administración), lo que reduciría las decisiones visibles del autor, e impone sus propias convenciones |
| **PostgreSQL + Docker Compose** | SQLite | PostgreSQL permite garantizar la no superposición de reservas a nivel de base de datos. SQLite carece de ese tipo de restricción y obligaría a resolver la concurrencia con bloqueos en el código, más frágil y más difícil de probar. Docker Compose hace el proyecto reproducible |
| **Monolito modular en capas** | Microservicios | Los microservicios añadirían complejidad de comunicación y despliegue sin beneficio real a esta escala. El monolito modular ofrece separación de responsabilidades con mucho menos costo |
| **Solo API, sin interfaz gráfica** | Aplicación web con interfaz | La interfaz consumiría tiempo que es más valioso invertir en especificaciones, pruebas y trazabilidad, que es lo que se evalúa. FastAPI genera documentación interactiva de la API que sirve para demostrar el sistema |

---

## 4. Estrategia de desarrollo

### 4.1 Metodología de trabajo

Desarrollo **iterativo e incremental** combinado con **Spec-Driven Development (SDD)**. Cada etapa produce una funcionalidad completa y verificada (especificación, código y pruebas) antes de pasar a la siguiente. Se crea **un cambio de OpenSpec por cada etapa**, lo que genera trazabilidad natural entre decisión, especificación, código y prueba.

**Secuencia en Git** (verificable en el historial):

1. Commit independiente con este documento, antes de instalar OpenSpec.
2. Inicialización de OpenSpec.
3. Especificaciones de cada etapa, revisadas antes de implementar.
4. Implementación y pruebas de cada etapa, con commits pequeños y mensajes claros.

### 4.2 Etapas, orden y dependencias

El orden lo determinan las dependencias entre componentes: no se puede crear una reserva sin un espacio, ni aprobarla si no existe, ni saber quién puede hacer qué sin usuarios y roles.

| Etapa | Contenido | Depende de | Se considera terminada cuando |
|---|---|---|---|
| **1** | Andamiaje del proyecto, usuarios, autenticación y roles | Ninguna | El proyecto se levanta con Docker Compose, un usuario puede registrarse e iniciar sesión, y los permisos por rol están probados |
| **2** | Gestión de espacios académicos | Etapa 1 | Solo el administrador gestiona espacios y todos los usuarios pueden consultarlos, con pruebas que lo verifican |
| **3** | Solicitudes de reserva, validación de disponibilidad y prevención de reservas simultáneas | Etapas 1 y 2 | Se pueden crear solicitudes válidas, se rechazan las que chocan con otras, y la prueba de concurrencia demuestra que dos solicitudes simultáneas no generan reservas solapadas |
| **4** | Aprobación, rechazo, cancelación e historial | Etapa 3 | Los estados cambian solo por transiciones permitidas, cada rol cancela lo que le corresponde, y el historial es consultable |
| **5** | Verificación y cierre | Etapas 1 a 4 | Todas las pruebas pasan, la documentación final está consolidada y se ha revisado la correspondencia entre especificaciones y código |

La Etapa 3 es el núcleo del sistema y recibe la revisión más cuidadosa.

### 4.3 Criterios generales para considerar terminada una etapa

1. La especificación fue revisada y aprobada **antes** de implementar.
2. Cada escenario de aceptación tiene al menos una prueba automatizada que pasa.
3. Las pruebas de las etapas anteriores siguen pasando.
4. El código generado fue revisado por el autor, y no contiene funcionalidades no especificadas.
5. Los cambios están registrados en Git con mensajes claros.

### 4.4 Diseño orientado al cambio

Las reglas de negocio se concentran en la capa de servicios, separadas de las rutas y del acceso a datos. Así, un cambio funcional posterior se aplica en un único lugar, previa actualización de las especificaciones mediante OpenSpec. Ninguna modificación funcional se realiza directamente sobre el código sin actualizar antes las especificaciones.

---

## 5. Estrategia de calidad

### 5.1 Tipos de pruebas

Todas con **pytest**:

- **Unitarias:** verifican las reglas de negocio de la capa de servicios (bloques de 30 minutos, horario de operación, transiciones de estado permitidas), sin base de datos.
- **De integración:** llaman a la API contra PostgreSQL en Docker; verifican rutas, permisos por rol y persistencia.
- **De concurrencia:** lanzan dos solicitudes simultáneas sobre el mismo horario y comprueban que solo una se crea.

### 5.2 Mecanismos para verificar el cumplimiento de requisitos

- Cada escenario de aceptación de las especificaciones tiene al menos una prueba, y el nombre de la prueba referencia el requisito que cubre.
- Se mantiene una tabla de trazabilidad requisito → prueba → código, consolidada al cierre.
- Todas las pruebas se ejecutan antes de cada commit.
- En la Etapa 5 se revisa la correspondencia entre especificaciones y código.

### 5.3 Riesgos y mitigación

| # | Riesgo | Mitigación |
|---|---|---|
| 1 | Especificaciones ambiguas que la IA rellena con suposiciones | Fase de revisión de especificaciones antes de implementar; escenarios de aceptación verificables |
| 2 | La IA se sale de la arquitectura o añade funcionalidades no pedidas | Tareas pequeñas, revisión del diff de cada tarea y un commit por tarea |
| 3 | Reservas simultáneas sobre el mismo horario | Restricción de exclusión en PostgreSQL y prueba de concurrencia |
| 4 | Errores en los límites de horario (una reserva que termina a las 11:00 y otra que empieza a las 11:00) | Rangos con inicio incluido y fin excluido; pruebas específicas para los bordes |
| 5 | Fallos de permisos entre roles | Una prueba por cada combinación importante de rol y operación, basada en la matriz de permisos |
| 6 | Tiempo insuficiente | Cinco etapas, alcance declarado y priorización del núcleo (Etapa 3) |
| 7 | Una solicitud pendiente ocupa un horario que luego no se necesita | Cancelación disponible para el solicitante y revisión del coordinador |

---

## 6. Uso de SDD e inteligencia artificial

### 6.1 ¿Cómo se utilizará OpenSpec?

OpenSpec se inicializa **después** del commit de este plan. Por cada etapa se crea un cambio que contiene tres elementos: una propuesta (qué se hará y por qué), las especificaciones (requisitos y escenarios) y las tareas de implementación. El ciclo es: proponer, revisar, implementar y archivar. Los comandos exactos dependen de la versión de OpenSpec y se confirmarán al iniciarlo.

El agente de programación será **Opencode**, que cuenta con integración con OpenSpec.

### 6.2 ¿Cómo se transformará el plan en especificaciones?

Cada elemento de este plan (módulo, rol, regla de negocio) se traduce en un requisito redactado en términos verificables, con al menos un escenario de aceptación. Por ejemplo, la regla R1 se convierte en un requisito con escenarios como "intentar reservar un horario ocupado por una solicitud pendiente es rechazado". Las reglas R1 a R6, la matriz de permisos y el ciclo de vida de la reserva son la fuente directa de las especificaciones.

### 6.3 ¿Qué actividades se delegarán a la IA?

- Borradores de especificaciones y de tareas.
- Código de cada tarea.
- Pruebas automatizadas.
- Configuración repetitiva (Docker Compose, estructura base del proyecto).

### 6.4 ¿Cómo se verificará el código generado?

Después de cada tarea, el autor:

1. Revisa el diff completo.
2. Ejecuta todas las pruebas.
3. Comprueba que el código cubre los escenarios de la especificación.
4. Verifica que no se añadió nada que no estaba especificado.
5. Solo entonces registra el cambio en Git.

### 6.5 ¿Qué decisiones serán responsabilidad del estudiante?

- La arquitectura, el stack y el alcance.
- Las reglas de negocio.
- La aprobación de cada especificación antes de implementar.
- La aceptación o el rechazo del código generado.
- El análisis de impacto de cualquier cambio funcional posterior.
