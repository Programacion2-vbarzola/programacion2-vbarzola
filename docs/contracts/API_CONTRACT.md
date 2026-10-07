# Contrato REST entre KMP y los backends propios

Este documento define los endpoints que la app KMP consume de cada backend propio.
Es la referencia contractual para los tres repositorios: si algo cambia acá, cambia en los tres.

## Convenciones generales

- Todas las fechas usan `yyyy-MM-dd`.
- Todos los instantes usan ISO-8601 UTC: `2026-09-10T14:30:00Z`.
- Las horas de turno usan `HH:mm` o `HH:mm:ss`.
- Los bodies usan `camelCase`.
- Los errores usan `application/problem+json` con un campo `code` funcional estable.
- Los endpoints protegidos requieren `Authorization: Bearer <jwt-propio>`.
- El JWT propio es emitido por Repo 3 al autenticar un usuario final. Nunca es el JWT técnico de cátedra.

## Formato de error

Todos los errores de ambos backends siguen este formato:

```json
{
  "type": "https://www.jhipster.tech/problem/problem-with-message",
  "title": "Descripción corta",
  "status": 400,
  "detail": "Descripción larga legible",
  "code": "CODIGO_FUNCIONAL_ESTABLE"
}
```

Los clientes deben decidir por `status` y `code`, nunca por `detail`.

---

## Repo 2 — Backend de Catálogo y Sincronización

Base URL: `http://localhost:8081` (desarrollo local)

### GET /api/catalog/version

Retorna la versión actual del catálogo. KMP la consulta para saber si su copia local en SQLDelight está desactualizada. Es el endpoint más liviano — no devuelve datos del catálogo.

**Autenticación:** Bearer JWT propio

**Response 200:**
```json
{
  "currentVersion": 7,
  "generatedAt": "2026-09-10T13:44:59Z"
}
```

**Errores:**

| Status | Code | Condición |
|--------|------|-----------|
| 401 | — | JWT ausente o inválido |

---

### GET /api/catalog/snapshot

Devuelve el catálogo completo con su versión. KMP lo usa para la sincronización inicial y para reconstruir su copia local cuando detecta que está desactualizada.

**Autenticación:** Bearer JWT propio

**Response 200:**
```json
{
  "snapshotVersion": 7,
  "generatedAt": "2026-09-10T13:44:59Z",
  "professionalCategories": [
    {
      "id": 1,
      "name": "Clínica médica",
      "description": "Atención clínica general",
      "enabled": true
    }
  ],
  "professionals": [
    {
      "id": 15,
      "categoryId": 1,
      "firstName": "Ana",
      "lastName": "Gómez",
      "enabled": true
    }
  ],
  "weeklySchedules": [
    {
      "id": 21,
      "professionalId": 15,
      "dayOfWeek": "MONDAY",
      "startTime": "09:00:00",
      "endTime": "12:00:00",
      "slotDurationMinutes": 30,
      "enabled": true
    }
  ]
}
```

`dayOfWeek` admite: `MONDAY`, `TUESDAY`, `WEDNESDAY`, `THURSDAY`, `FRIDAY`, `SATURDAY`, `SUNDAY`.

Las entidades con `enabled: false` se incluyen (representan bajas lógicas que KMP debe reflejar).

**Errores:**

| Status | Code | Condición |
|--------|------|-----------|
| 401 | — | JWT ausente o inválido |

---

### GET /api/categories

Lista las categorías de profesionales. KMP la usa para poblar los filtros de búsqueda en la pantalla de catálogo.

**Autenticación:** Bearer JWT propio

**Response 200:**
```json
[
  {
    "id": 1,
    "name": "Clínica médica",
    "description": "Atención clínica general",
    "enabled": true
  }
]
```

**Errores:**

| Status | Code | Condición |
|--------|------|-----------|
| 401 | — | JWT ausente o inválido |

---

### GET /api/professionals

Busca y filtra profesionales. Todos los filtros son opcionales y combinables. La búsqueda se resuelve sobre datos locales del backend — nunca consulta a la cátedra por request.

**Autenticación:** Bearer JWT propio

**Query params:**

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `categoryId` | integer | Filtrar por categoría |
| `name` | string | Búsqueda parcial por nombre o apellido |
| `enabled` | boolean | Filtrar por estado habilitado (`true`/`false`) |
| `availableOn` | date (`yyyy-MM-dd`) | Solo profesionales con horario habilitado ese día de la semana |
| `page` | integer | Página (default: 0) |
| `size` | integer | Tamaño de página (default: 20) |

**Response 200:**
```json
[
  {
    "id": 15,
    "categoryId": 1,
    "categoryName": "Clínica médica",
    "firstName": "Ana",
    "lastName": "Gómez",
    "enabled": true
  }
]
```

La respuesta incluye headers de paginación: `X-Total-Count` y `Link`.

**Errores:**

| Status | Code | Condición |
|--------|------|-----------|
| 400 | `VALIDATION_ERROR` | Parámetro con formato inválido |
| 401 | — | JWT ausente o inválido |

---

## Repo 3 — Backend de Turnos y Reservas

Base URL: `http://localhost:8082` (desarrollo local)

### POST /api/register

Registra un nuevo usuario final. No requiere autenticación.
El modelo es compatible con el usuario generado por JHipster.

**Autenticación:** ninguna

**Request:**
```json
{
  "login": "valentin.barzola",
  "password": "clave-segura-123",
  "firstName": "Valentín",
  "lastName": "Barzola",
  "email": "valentin@example.com",
  "imageUrl": "",
  "langKey": "es"
}
```

| Campo | Obligatorio | Restricciones |
|-------|-------------|---------------|
| `login` | sí | 3-50 caracteres, letras/números/guion/punto |
| `password` | sí | 4-100 caracteres |
| `firstName` | sí | máximo 50 |
| `lastName` | sí | máximo 50 |
| `email` | sí | email válido, 5-254 caracteres, único |
| `imageUrl` | no | máximo 256 |
| `langKey` | no | 2-10 caracteres, default `es` |

**Response 201:** body vacío. El usuario queda activo inmediatamente.

**Errores:**

| Status | Code | Condición |
|--------|------|-----------|
| 400 | `VALIDATION_ERROR` | Campo faltante o formato inválido |
| 400 | `LOGIN_ALREADY_USED` | `login` ya registrado |
| 400 | `EMAIL_ALREADY_USED` | `email` ya registrado |

---

### POST /api/authenticate

Autentica un usuario y devuelve el JWT propio de sesión.

**Autenticación:** ninguna

**Request:**
```json
{
  "username": "valentin.barzola",
  "password": "clave-segura-123",
  "rememberMe": true
}
```

`rememberMe: true` emite un JWT de larga duración. La app siempre debe enviarlo en `true`.

**Response 200:**
```json
{
  "id_token": "<jwt-propio-del-usuario>"
}
```

**Errores:**

| Status | Code | Condición |
|--------|------|-----------|
| 401 | — | Credenciales incorrectas |

---

### GET /api/account

Devuelve los datos del usuario actualmente autenticado. KMP lo usa al iniciar sesión para mostrar el nombre y configurar la sesión local.

**Autenticación:** Bearer JWT propio

**Response 200:**
```json
{
  "id": 1,
  "login": "valentin.barzola",
  "firstName": "Valentín",
  "lastName": "Barzola",
  "email": "valentin@example.com",
  "imageUrl": "",
  "langKey": "es",
  "authorities": ["ROLE_USER"]
}
```

**Errores:**

| Status | Code | Condición |
|--------|------|-----------|
| 401 | — | JWT ausente o inválido |

---

### GET /api/availability

Devuelve los slots disponibles de un profesional en una fecha específica. Combina los horarios semanales del catálogo con las ocupaciones informadas por la cátedra.

**Autenticación:** Bearer JWT propio

**Query params:**

| Parámetro | Tipo | Obligatorio | Descripción |
|-----------|------|-------------|-------------|
| `professionalId` | integer | sí | ID del profesional |
| `date` | date | sí | Fecha a consultar (`yyyy-MM-dd`) |

**Response 200:**
```json
{
  "professionalId": 15,
  "date": "2026-09-14",
  "slots": [
    {
      "startTime": "09:00",
      "endTime": "09:30",
      "available": true
    },
    {
      "startTime": "09:30",
      "endTime": "10:00",
      "available": false
    }
  ]
}
```

`available: false` significa que el slot está ocupado (confirmado o con hold activo). KMP solo muestra los slots con `available: true`.

**Errores:**

| Status | Code | Condición |
|--------|------|-----------|
| 400 | `VALIDATION_ERROR` | Parámetro faltante o inválido |
| 401 | — | JWT ausente o inválido |
| 404 | `PROFESSIONAL_NOT_FOUND` | Profesional inexistente |

---

### POST /api/holds

Solicita un bloqueo temporal sobre un slot. Si el slot ya está ocupado o bloqueado, falla.

**Autenticación:** Bearer JWT propio

**Request:**
```json
{
  "professionalId": 15,
  "date": "2026-09-14",
  "startTime": "09:30"
}
```

**Response 201:**
```json
{
  "holdId": "hold-6c0c2f41",
  "reservationProcessId": "process-a92b47d0",
  "status": "HELD",
  "expiresAt": "2026-09-10T14:44:50Z"
}
```

`expiresAt` es la fecha límite real del hold. KMP debe mostrarlo como countdown. Un hold local no extiende su vigencia en el servidor.

**Errores:**

| Status | Code | Condición |
|--------|------|-----------|
| 400 | `VALIDATION_ERROR` | Campo faltante o inválido |
| 400 | `INVALID_SLOT` | Slot fuera de la agenda habilitada |
| 401 | — | JWT ausente o inválido |
| 404 | `PROFESSIONAL_NOT_FOUND` | Profesional inexistente |
| 409 | `SLOT_ALREADY_HELD` | Hold activo sobre ese slot |
| 409 | `SLOT_ALREADY_RESERVED` | Slot confirmado por otra reserva |

---

### POST /api/holds/{holdId}/confirm

Confirma inicialmente el hold. Inicia el proceso asincrónico: el backend escucha a cátedra vía Kafka y solicita el teléfono del paciente.

**Autenticación:** Bearer JWT propio

**Request:**
```json
{
  "patientFirstName": "Valentín",
  "patientLastName": "Barzola"
}
```

**Response 202:**
```json
{
  "reservationProcessId": "process-a92b47d0",
  "status": "WAITING_FOR_PHONE",
  "expiresAt": "2026-09-10T14:44:50Z"
}
```

Después de esto, KMP debe hacer polling a `/api/processes/{id}/status` para saber cuando el backend recibió el pedido de teléfono de cátedra.

**Errores:**

| Status | Code | Condición |
|--------|------|-----------|
| 400 | `VALIDATION_ERROR` | Campo faltante o inválido |
| 401 | — | JWT ausente o inválido |
| 404 | `HOLD_NOT_FOUND` | Hold inexistente |
| 404 | `HOLD_EXPIRED` | Hold vencido |
| 409 | `SLOT_ALREADY_RESERVED` | Slot ya confirmado |

---

### GET /api/processes/{reservationProcessId}/status

Consulta el estado actual de un proceso de reserva. KMP hace polling a este endpoint después de confirmar el hold, esperando que el backend informe que necesita el teléfono.

**Autenticación:** Bearer JWT propio

**Response 200:**
```json
{
  "reservationProcessId": "process-a92b47d0",
  "status": "PHONE_REQUESTED",
  "expiresAt": "2026-09-10T14:44:50Z",
  "message": "Ingrese un número de teléfono para continuar."
}
```

Estados posibles de `status` (ver tabla completa al final del documento).

**Errores:**

| Status | Code | Condición |
|--------|------|-----------|
| 401 | — | JWT ausente o inválido |
| 403 | `PROCESS_OWNER_MISMATCH` | El proceso pertenece a otro usuario |
| 404 | `PROCESS_NOT_FOUND` | Proceso inexistente |

---

### POST /api/processes/{reservationProcessId}/phone

Envía el número de teléfono del paciente para completar la reserva. Puede reintentarse con un número diferente mientras el proceso no haya vencido.

**Autenticación:** Bearer JWT propio

**Request:**
```json
{
  "phoneNumber": "+5492615551234"
}
```

El backend valida el formato antes de enviarlo a cátedra. Formato aceptado: dígitos con `+` opcional al inicio, entre 7 y 15 dígitos (se ignoran espacios, guiones, paréntesis).

**Response 202:**
```json
{
  "reservationProcessId": "process-a92b47d0",
  "status": "PHONE_SUBMITTED"
}
```

Después de esto KMP debe seguir haciendo polling a `/api/processes/{id}/status` para conocer el resultado final.

**Errores:**

| Status | Code | Condición |
|--------|------|-----------|
| 400 | `VALIDATION_ERROR` | Teléfono con formato inválido |
| 400 | `INVALID_PHONE_NUMBER` | Teléfono rechazado por cátedra |
| 401 | — | JWT ausente o inválido |
| 403 | `PROCESS_OWNER_MISMATCH` | El proceso pertenece a otro usuario |
| 404 | `PROCESS_NOT_FOUND` | Proceso inexistente |
| 409 | `PROCESS_EXPIRED` | Proceso vencido, no se puede reintentar |
| 409 | `PROCESS_ALREADY_FINAL` | Proceso ya en estado final (confirmado/inválido) |

---

### GET /api/appointments

Lista las reservas del usuario autenticado. El backend filtra automáticamente por el usuario del JWT — nunca devuelve reservas de otros usuarios.

**Autenticación:** Bearer JWT propio

**Query params (todos opcionales):**

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `status` | string | Filtrar por estado: `CONFIRMED`, `CANCELLED`, `FAILED` |
| `from` | date | Fecha mínima inclusiva |
| `to` | date | Fecha máxima inclusiva |
| `page` | integer | Página (default: 0) |
| `size` | integer | Tamaño de página (default: 20) |

**Response 200:**
```json
[
  {
    "reservationId": 8251,
    "reservationProcessId": "process-a92b47d0",
    "appointmentDate": "2026-09-14",
    "startTime": "09:30:00",
    "endTime": "10:00:00",
    "professionalId": 15,
    "professionalFirstName": "Ana",
    "professionalLastName": "Gómez",
    "patientFirstName": "Valentín",
    "patientLastName": "Barzola",
    "patientPhone": "+5492615551234",
    "status": "CONFIRMED",
    "createdAt": "2026-09-10T14:35:00Z",
    "confirmedAt": "2026-09-10T14:36:00Z",
    "cancelledAt": null,
    "cancellationReason": null
  }
]
```

Incluye headers: `X-Total-Count` y `Link`.

**Errores:**

| Status | Code | Condición |
|--------|------|-----------|
| 400 | `INVALID_DATE_RANGE` | `from` posterior a `to` |
| 401 | — | JWT ausente o inválido |

---

### POST /api/appointments/{reservationProcessId}/cancel

Cancela una reserva propia confirmada. Solo el dueño de la reserva puede cancelarla. La operación es idempotente: cancelar una reserva ya cancelada devuelve su estado actual sin error.

**Autenticación:** Bearer JWT propio

**Request (opcional):**
```json
{
  "reason": "Cambio solicitado por el usuario"
}
```

**Response 200:**
```json
{
  "reservationProcessId": "process-a92b47d0",
  "status": "CANCELLED",
  "cancelledAt": "2026-09-11T15:10:00Z"
}
```

**Errores:**

| Status | Code | Condición |
|--------|------|-----------|
| 401 | — | JWT ausente o inválido |
| 403 | `RESERVATION_OWNER_MISMATCH` | La reserva pertenece a otro usuario |
| 404 | `RESERVATION_NOT_FOUND` | Reserva inexistente |
| 409 | `RESERVATION_NOT_CANCELLABLE` | Estado distinto de `CONFIRMED` o `CANCELLED` |

---

## Estados de un proceso de reserva

Esta tabla define los estados que usa Repo 3 internamente y expone en `/api/processes/{id}/status`. KMP los usa para decidir qué mostrar en pantalla. Los estados finales son terminales: una vez alcanzados no pueden retroceder.

| Estado | Descripción | ¿Final? | Qué hace KMP |
|--------|-------------|---------|--------------|
| `HELD` | Hold creado, esperando confirmación inicial | no | Mostrar countdown de `expiresAt` |
| `WAITING_FOR_PHONE` | Hold confirmado, cátedra aún no pidió el teléfono | no | Hacer polling |
| `PHONE_REQUESTED` | Cátedra pidió el teléfono vía Kafka | no | Mostrar formulario de teléfono |
| `PHONE_SUBMITTED` | Teléfono enviado, esperando resultado de cátedra | no | Hacer polling |
| `CONFIRMED` | Reserva confirmada exitosamente | **sí** | Mostrar confirmación |
| `REJECTED` | Teléfono rechazado, reintentable hasta `expiresAt` | no | Mostrar error, permitir reintento |
| `EXPIRED` | Proceso vencido sin completarse | **sí** | Mostrar vencimiento |
| `INVALID` | Proceso inválido detectado por cátedra | **sí** | Mostrar error, sugerir nueva reserva |
| `CANCELLED` | Reserva cancelada por el usuario | **sí** | Mostrar cancelación |
| `FAILED` | Error interno no recuperable | **sí** | Mostrar error, sugerir nueva reserva |
