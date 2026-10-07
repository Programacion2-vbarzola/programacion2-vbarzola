# Planificación de sprints — Proyecto Integrador 2026

Este documento define los sprints de trabajo para los 3 repositorios del proyecto. Está pensado para que una IA (o el propio alumno) genere issues de GitHub a partir de cada bloque "Issue".

## Cómo usar este documento

- Cada repo tiene su propia lista de sprints, **independiente de los otros dos**. Se puede avanzar más en un repo sin esperar a los demás.
- Cada issue indica su **Dependencia**:
  - `Ninguna`: se puede hacer ya mismo, sin tocar otro repo.
  - `Soft (mockeable)`: idealmente depende de otro repo, pero se puede avanzar contra un mock/stub propio sin bloquearse.
  - `Hard`: necesita que el otro repo ya tenga ese endpoint/evento funcionando de verdad (no alcanza con mock) para cerrarse del todo. Se puede empezar antes, pero no se puede dar por terminado sin eso.
- Al crear los issues en GitHub, se puede usar el título entre corchetes como prefijo (`[S0]`, `[S1]`, etc.) y labels por sprint (`sprint-0`, `sprint-1`, ...).
- Cada sprint cierra con tests del incremento agregado y actualización de la documentación correspondiente — no se posponen a un sprint final.

Decisiones ya tomadas (ver justificación completa en cada repo):
- Los 2 backends usan **JHipster** (Java + Spring Boot) con **PostgreSQL** como motor servidor, esquema/usuario propio por servicio.
- Comunicación Turnos → Catálogo usa **JWT técnico de servicio** (no propaga el JWT del usuario final). Justificación: Catálogo es de solo lectura y no necesita conocer usuarios finales; la trazabilidad por usuario vive en Turnos.
- La app KMP usa **Compose Multiplatform** para la UI y **SQLDelight** para persistencia local.
- Ningún repo accede a las tablas internas de otro: toda integración es HTTP + JWT.

---

## Repo 1 — App KMP (Android)

Responsabilidad: UI + lógica cliente. Consume exclusivamente los contratos propios expuestos por Repo 2 y Repo 3. Nunca ve el JWT técnico de cátedra.

### Sprint 0 — Scaffolding y contrato

- **[S0-KMP-1] Estructura del proyecto KMP**
  - Crear módulos `shared` (modelos, red, DB, repos) y `androidApp` (Compose Multiplatform).
  - Configurar Gradle version catalog, `.gitignore`.
  - Dependencia: Ninguna.

- **[S0-KMP-2] Documentar contrato REST propio esperado de Repo 2 y Repo 3**
  - Escribir en `docs/contracts/` (OpenAPI o Markdown) los endpoints que esta app espera de ambos backends: auth, snapshot/búsqueda de catálogo, disponibilidad, holds, confirmación, estado de proceso, listado/cancelación de reservas.
  - Este documento es la referencia que Repo 2 y Repo 3 deben cumplir.
  - Dependencia: Ninguna (se escribe antes de que existan los backends).

- **[S0-KMP-3] Mock server del contrato propio**
  - Levantar un mock (Prism sobre el OpenAPI, o WireMock/json-server) que implemente el contrato de S0-KMP-2.
  - Permite desarrollar toda la app sin depender de que los otros repos avancen al mismo ritmo.
  - Dependencia: Ninguna.

### Sprint 1 — Autenticación de usuario final

- **[S1-KMP-1] Pantallas de registro y login**
  - Formulario con validaciones de UI equivalentes a las del backend (login 3-50 caracteres, password 4-100, email opcional).
  - Dependencia: Soft (mockeable) — contra Repo 3 o su mock.

- **[S1-KMP-2] Almacenamiento seguro del JWT propio**
  - Persistencia cifrada del JWT de usuario final (nunca en texto plano, nunca loguearlo).
  - Dependencia: Ninguna.

- **[S1-KMP-3] Cliente HTTP con interceptor de auth**
  - Interceptor que agrega `Authorization: Bearer` a cada request protegida y maneja 401 (logout forzado, limpieza de sesión local).
  - Dependencia: Ninguna.

### Sprint 2 — Catálogo local y búsqueda

- **[S2-KMP-1] Esquema SQLDelight de catálogo**
  - Tablas para categorías, profesionales, horarios semanales + versión local aplicada.
  - Dependencia: Ninguna.

- **[S2-KMP-2] Sincronización completa (snapshot) contra Repo 2**
  - Consumir el endpoint propio de snapshot del backend de catálogo, persistir de forma consistente (todo o nada).
  - Dependencia: Soft (mockeable) — contra el mock del contrato hasta que Repo 2 tenga el endpoint real.

- **[S2-KMP-3] Pantalla de búsqueda y filtros**
  - Filtros obligatorios: categoría, nombre, estado habilitado, disponibilidad. Resueltos 100% contra SQLDelight local.
  - Dependencia: Ninguna (usa datos ya sincronizados localmente).

### Sprint 3 — Disponibilidad y creación de hold

- **[S3-KMP-1] Pantalla de horarios disponibles**
  - Por profesional y fecha, consumiendo el endpoint propio de disponibilidad de Repo 3.
  - Dependencia: Soft (mockeable).

- **[S3-KMP-2] Flujo de creación de hold**
  - Selección de slot → `POST` hold propio → mostrar countdown con `expiresAt`.
  - Mapeo de errores (slot ya reservado/held, profesional deshabilitado, slot inválido) a mensajes de usuario.
  - Dependencia: Soft (mockeable).

### Sprint 4 — Confirmación asíncrona (teléfono)

- **[S4-KMP-1] Confirmar hold y esperar pedido de teléfono**
  - Llamada a confirmación inicial → estado `WAITING_FOR_PHONE`.
  - Polling a endpoint propio de estado de proceso para detectar cuándo Repo 3 recibió `AdditionalInformationRequested` desde Kafka.
  - Dependencia: Hard — necesita que Repo 3 tenga el consumidor Kafka real funcionando; con mock solo se puede simular la UI.

- **[S4-KMP-2] Pantalla de ingreso de teléfono**
  - Envío del teléfono al backend propio, reintentable mientras no se venza el proceso.
  - Dependencia: Soft (mockeable) para el envío; Hard para ver el resultado real.

- **[S4-KMP-3] Pantalla de resultado final del proceso**
  - Estados: confirmado, rechazado (reintentable), expirado, inválido. Tratar los estados finales como terminales en la UI.
  - Dependencia: Hard.

### Sprint 5 — Mis reservas

- **[S5-KMP-1] Listado de reservas propias**
  - Consumir el endpoint propio de Repo 3, filtrado automáticamente por el usuario logueado.
  - Dependencia: Soft (mockeable).

- **[S5-KMP-2] Cancelación de reserva confirmada**
  - Botón de cancelar con estado idempotente (deshabilitar tras click, reflejar estado actual).
  - Dependencia: Soft (mockeable).

### Sprint 6 — Robustez

- **[S6-KMP-1] Manejo centralizado de errores**
  - Mapeo de `application/problem+json` / códigos propios a mensajes de usuario.
  - Dependencia: Ninguna.

- **[S6-KMP-2] Retry/backoff acotado y estados offline**
  - Indicadores de offline/reconectando, catálogo "stale" visible cuando no se puede sincronizar.
  - Dependencia: Ninguna.

### Sprint 7 — Tests, documentación y auditoría final

- **[S7-KMP-1] Unit tests de `shared`**
  - Repositorios, mappers, máquina de estados de reserva.
  - Dependencia: Ninguna.

- **[S7-KMP-2] README y diagrama de arquitectura**
  - Pasos reproducibles, incluyendo cómo apuntar a mock vs. backends reales.
  - Dependencia: Ninguna.

- **[S7-KMP-3] Auditoría contra checklist de evidencias mínimas**
  - Verificar cobertura de los ítems de la sección 11 del enunciado que involucran a esta app (búsquedas/filtros, reserva confirmada, reserva expirada, consulta/cancelación propia sin acceso cruzado).
  - Dependencia: Hard (necesita los otros 2 repos funcionando de verdad, no mocks).

---

## Repo 2 — Backend de Catálogo y Sincronización

Responsabilidad: única fuente local vigente de categorías, profesionales y horarios. Sincroniza con la cátedra vía REST/Redis/Kafka. Expone contrato propio de solo lectura a Repo 3 y Repo 1.

### Sprint 0 — Scaffolding

- **[S0-CAT-1] Proyecto JHipster + entidades base**
  - Entidades `ProfessionalCategory`, `Professional`, `WeeklySchedule` + tabla de versión de sincronización local.
  - Dependencia: Ninguna.

- **[S0-CAT-2] Externalización de configuración de cátedra**
  - Variables de entorno para URL REST, host/puerto Redis, bootstrap servers Kafka de cátedra (nunca hardcodeadas ni commiteadas).
  - Dependencia: Ninguna.

- **[S0-CAT-3] Docker Compose con PostgreSQL propio**
  - Levanta el servicio + su base, sin depender de infraestructura de cátedra para desarrollo local básico.
  - Dependencia: Ninguna.

- **[S0-CAT-4] Registro de cuenta técnica contra cátedra**
  - Una vez, vía Postman (`POST /api/student/register`), guardar JWT técnico + config de forma segura y externa.
  - Dependencia: Ninguna (requiere que cátedra ya haya habilitado el entorno).

### Sprint 1 — Seguridad base y contrato propio hacia afuera

- **[S1-CAT-1] JWT técnico de servicio para llamadas de Repo 3**
  - Endpoint/mecanismo de autenticación que usará Repo 3 para autenticarse como servicio autorizado (no como usuario final).
  - Dependencia: Ninguna (Repo 3 lo consume después).

- **[S1-CAT-2] Configuración CORS**
  - Permitir únicamente los orígenes necesarios (en este caso, llamadas server-to-server desde Repo 3; no debería exponerse directo a KMP).
  - Dependencia: Ninguna.

- **[S1-CAT-3] Documentar contrato propio de catálogo**
  - Endpoints que expone este servicio hacia Repo 1 y Repo 3 (búsqueda/filtros, horarios vigentes de un profesional).
  - Dependencia: Ninguna.

### Sprint 2 — Sincronización completa

- **[S2-CAT-1] Cliente REST contra cátedra: snapshot**
  - Consumir `GET /api/synchronization/snapshot`, aplicar de forma transaccional (todo o nada), guardar `snapshotVersion` solo si las 3 colecciones se aplicaron correctamente.
  - Dependencia: Ninguna (depende de cátedra, no de otros repos propios).

- **[S2-CAT-2] Endpoint propio de búsqueda/filtros**
  - `GET /professionals` con filtros por categoría, nombre, habilitado, disponibilidad — resuelto 100% sobre datos locales.
  - Dependencia: Ninguna.

- **[S2-CAT-3] Endpoint propio de horarios vigentes por profesional**
  - Usado por Repo 3 para validar antes de crear un hold.
  - Dependencia: Ninguna.

### Sprint 3 — Sincronización incremental

- **[S3-CAT-1] Consumidor Kafka `CatalogUpdated`**
  - Deduplicación por `eventId`, lectura de metadata en Redis al recibir notificación.
  - Dependencia: Ninguna.

- **[S3-CAT-2] Aplicación de cambios incrementales (`changes:{version}`)**
  - Aplicar en orden, avanzar versión local solo si la unidad de trabajo completa fue exitosa. Procesamiento repetido de la misma notificación no debe alterar el resultado.
  - Dependencia: Ninguna.

- **[S3-CAT-3] Detección de discontinuidad y recuperación**
  - Si la versión local es menor que `oldestAvailableVersion`, disparar snapshot completo (reusa S2-CAT-1).
  - Dependencia: Ninguna.

### Sprint 4 — Robustez de sincronización

- **[S4-CAT-1] Idempotencia verificada ante duplicados Kafka**
  - Test explícito: mismo `eventId` recibido dos veces no duplica efectos.
  - Dependencia: Ninguna.

- **[S4-CAT-2] Observabilidad de sincronización**
  - Endpoint o log estructurado con estado y errores de la última sincronización (completa/incremental).
  - Dependencia: Ninguna.

### Sprint 5 — Integración real con Repo 3

- **[S5-CAT-1] Habilitar y probar JWT técnico de servicio con Repo 3 real**
  - Verificar que Repo 3 puede autenticarse y consultar horarios vigentes de punta a punta.
  - Dependencia: Hard — necesita que Repo 3 ya llame de verdad (Sprint 3 de Repo 3).

- **[S5-CAT-2] Habilitar y probar endpoints con Repo 1 real**
  - Verificar snapshot y búsqueda consumidos de punta a punta desde la app KMP real.
  - Dependencia: Hard — necesita que Repo 1 deje de usar el mock (Sprint 2 de Repo 1).

### Sprint 6 — Tests y documentación

- **[S6-CAT-1] Tests automatizados mínimos exigidos**
  - Sincronización completa e incremental, idempotencia, autorización del endpoint propio.
  - Dependencia: Ninguna.

- **[S6-CAT-2] Documentación**
  - Modelo de datos y propiedad, estrategia de sync completa/incremental, decisiones de seguridad (JWT técnico, CORS).
  - Dependencia: Ninguna.

### Sprint 7 — Auditoría final

- **[S7-CAT-1] Checklist de evidencias mínimas (sección 11) para este repo**
  - Inicialización desde snapshot, actualización incremental, recuperación tras discontinuidad, búsquedas/filtros, separación efectiva de datos, rechazo de accesos no autenticados.
  - Dependencia: Ninguna para la mayoría; Hard solo para la verificación end-to-end con Repo 1/3 reales.

---

## Repo 3 — Backend de Turnos y Reservas

Responsabilidad: disponibilidad, holds, máquina de estados de reserva, integración asincrónica vía Kafka con cátedra. Único propietario de holds/procesos/reservas. Consulta a Repo 2 para datos de catálogo vigentes, nunca los replica como fuente propia.

### Sprint 0 — Scaffolding

- **[S0-TUR-1] Proyecto JHipster + entidades base**
  - Entidades `Hold`, `ReservationProcess`, `Reservation` (con estado, timestamps, FK a usuario final).
  - Dependencia: Ninguna.

- **[S0-TUR-2] Externalización de configuración de cátedra**
  - Variables de entorno para REST, Kafka de cátedra (bootstrap servers, topics, consumer group) — nunca hardcodeadas.
  - Dependencia: Ninguna.

- **[S0-TUR-3] Docker Compose con PostgreSQL propio**
  - Dependencia: Ninguna.

- **[S0-TUR-4] Registro de cuenta técnica contra cátedra**
  - Una vez, vía Postman, guardar JWT técnico + config de forma segura.
  - Dependencia: Ninguna.

- **[S0-TUR-5] Diseño de `externalPatientId`**
  - Definir cómo se genera/mapea un identificador estable del usuario final de esta app para enviarlo como `externalPatientId` a cátedra, y cómo se usa después para filtrar `GET /appointments`.
  - Dependencia: Ninguna. Decisión de diseño que condiciona el modelo de usuario del Sprint 1.

### Sprint 1 — Usuarios finales y seguridad base

- **[S1-TUR-1] Modelo de usuario final compatible JHipster**
  - Registro/login, emisión de JWT propio, hashing de password.
  - Dependencia: Ninguna.

- **[S1-TUR-2] Cliente HTTP autenticado hacia Repo 2 (JWT técnico de servicio)**
  - Consumir el mecanismo definido en S1-CAT-1.
  - Dependencia: Soft (mockeable) hasta que Repo 2 tenga ese endpoint real.

- **[S1-TUR-3] Configuración CORS**
  - Permitir únicamente el origen de la app KMP.
  - Dependencia: Ninguna.

### Sprint 2 — Disponibilidad

- **[S2-TUR-1] Cliente REST contra cátedra: ocupaciones**
  - `GET /api/appointment-occupancies`.
  - Dependencia: Ninguna (depende de cátedra).

- **[S2-TUR-2] Endpoint propio de disponibilidad**
  - Combina horarios vigentes (consultados a Repo 2) + ocupaciones (cátedra) + fecha/profesional elegidos.
  - Dependencia: Soft (mockeable) contra el mock de Repo 2 hasta integrar real.

### Sprint 3 — Holds y confirmación inicial

- **[S3-TUR-1] Crear hold**
  - `POST /api/appointment-holds` contra cátedra, antes validar vigencia contra Repo 2.
  - Dependencia: Soft (mockeable) para Repo 2; depende de cátedra para el hold real.

- **[S3-TUR-2] Confirmar hold inicialmente**
  - `POST /api/appointment-holds/{holdId}/confirm`, pasar a `WAITING_FOR_PHONE`.
  - Dependencia: Ninguna adicional (depende de cátedra).

- **[S3-TUR-3] Endpoint propio de estado de proceso**
  - Para que Repo 1 pueda hacer polling (`GET /processes/{id}/status`).
  - Dependencia: Ninguna.

### Sprint 4 — Intercambio asincrónico

- **[S4-TUR-1] Consumidor Kafka `AdditionalInformationRequested`**
  - Guardar `requestEventId`, actualizar estado local.
  - Dependencia: Ninguna (depende de cátedra).

- **[S4-TUR-2] Endpoint propio para recibir el teléfono**
  - Valida formato antes de enviar, guarda localmente.
  - Dependencia: Ninguna.

- **[S4-TUR-3] Productor Kafka `AdditionalInformationSubmitted`**
  - Message key = `reservationProcessId`, incluye `requestEventId`.
  - Dependencia: Ninguna.

### Sprint 5 — Resultados finales y gestión de reservas

- **[S5-TUR-1] Consumidores Kafka de resultado final**
  - `AppointmentConfirmed`, `AdditionalInformationRejected`, `AppointmentProcessExpired`, `AppointmentProcessInvalid`.
  - Tratar estados finales como terminales (un evento tardío no reabre un proceso cerrado).
  - Dependencia: Ninguna (depende de cátedra).

- **[S5-TUR-2] Endpoint propio: listar reservas del usuario**
  - Filtrado por el usuario final logueado usando el mapeo de `externalPatientId` (S0-TUR-5).
  - Dependencia: Ninguna.

- **[S5-TUR-3] Endpoint propio: cancelar reserva**
  - `POST /api/appointments/{id}/cancel` contra cátedra, idempotente, solo si pertenece al usuario autenticado.
  - Dependencia: Ninguna (depende de cátedra).

### Sprint 6 — Robustez

- **[S6-TUR-1] Idempotencia verificada ante duplicados Kafka**
  - Test explícito con `eventId` repetido.
  - Dependencia: Ninguna.

- **[S6-TUR-2] Vencimiento de holds/procesos**
  - Cierre local correcto, sin nuevos intentos de confirmación tras `HOLD_EXPIRED` / `AppointmentProcessExpired`.
  - Dependencia: Ninguna.

- **[S6-TUR-3] Recuperación de procesos pendientes tras reinicio**
  - Al reiniciar el servicio, los procesos en estados intermedios deben poder seguir avanzando o cerrarse correctamente.
  - Dependencia: Ninguna.

### Sprint 7 — Tests, documentación y auditoría final

- **[S7-TUR-1] Tests automatizados mínimos exigidos**
  - Reservas, autorización (aislamiento entre usuarios), idempotencia.
  - Dependencia: Ninguna.

- **[S7-TUR-2] Documentación**
  - Contrato entre Repo 2 y Repo 3, máquina de estados, decisiones de seguridad y de `externalPatientId`.
  - Dependencia: Ninguna.

- **[S7-TUR-3] Checklist de evidencias mínimas (sección 11) para este repo**
  - Reserva confirmada end-to-end, reserva expirada, cancelación propia sin acceso cruzado, mensaje Kafka duplicado procesado correctamente.
  - Dependencia: Hard — necesita integración real con Repo 1 y Repo 2.

---

## Apéndice — Evidencias mínimas (sección 11 del enunciado) vs. sprints

| Evidencia exigida | Repo(s) responsables | Sprint |
| --- | --- | --- |
| Inicialización desde snapshot | Repo 2, Repo 1 | S2 |
| Actualización incremental del catálogo | Repo 2 | S3 |
| Recuperación mediante snapshot tras discontinuidad | Repo 2 | S3 |
| Búsquedas resueltas sobre datos locales | Repo 2, Repo 1 | S2 |
| Filtros obligatorios (categoría, nombre, habilitado, disponibilidad) | Repo 2, Repo 1 | S2-S3 |
| Reserva confirmada vía flujo REST + Kafka | Repo 3, Repo 1 | S4-S5 |
| Reserva que expire sin completar información | Repo 3, Repo 1 | S4-S6 |
| Consulta y cancelación de reserva propia, sin acceso cruzado | Repo 3, Repo 1 | S5 |
| Procesamiento idempotente de mensaje duplicado | Repo 2, Repo 3 | S4 (cat) / S6 (tur) |
| Separación efectiva de datos entre los dos servicios | Repo 2, Repo 3 | S0 (diseño) / S7 (auditoría) |
| Rechazo de accesos no autenticados o no autorizados | Repo 2, Repo 3, Repo 1 | S1 |
| Pruebas automatizadas de comportamientos principales | Repo 2, Repo 3, Repo 1 | S6-S7 |
