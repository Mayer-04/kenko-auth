# Kenko — Microservicio `Authentication`

## 1. Contexto

- **Proyecto:** portal de salud para una EPS (entidad promotora de salud) en Colombia, con Frontend y Backend. Solo trabajamos el **Backend (microservicios)**.
- **Microservicios del proyecto (5):** `Authentication`, `Profile`, `Documents`, `Appointments`, `Notifications`.
- **Comunicación entre microservicios:** siempre asíncrona, mediante **eventos**.
- **Microservicio actual: `Authentication`.** Se encarga de:
  - Registrar usuarios nuevos.
  - Iniciar sesión.
  - Recuperar contraseña.
  - MFA con TOTP.

### Datos del registro

Nombres, apellidos, fecha de nacimiento, género, correo electrónico, teléfono, dirección de residencia, ciudad, número de documento de identidad (cédula; **es el nombre de usuario**) y contraseña (con campo de confirmación).

### Datos del inicio de sesión

Número de documento de identidad y contraseña.

---

## 2. Eventos

`Authentication` publica estos eventos. Los consumen `Profile` y `Notifications`.

| Evento | Se dispara cuando… | Consumidor → acción |
| --- | --- | --- |
| `UserRegistered` | El registro fue exitoso | `Profile`: crea el registro base del perfil. `Notifications`: envía correo de bienvenida confirmando el registro. |
| `PasswordResetRequested` | El usuario solicita recuperar su contraseña | `Notifications`: envía correo con el enlace de recuperación. |
| `MfaEnabled` | El usuario activa MFA (TOTP) | `Notifications`: envía código o notificación de verificación. *(payload por definir)* |

### Estructura de los eventos

> El bloque `detail.data` puede cambiar o mejorarse, con el fin de darle a los demás microservicios datos más estructurados y que realmente necesiten. `source`, `detail-type` y `detail.metadata` son la estructura base.

**UserRegistered** (consumen `Profile` y `Notifications`)

```json
{
  "source": "kenko.auth",
  "detail-type": "UserRegistered",
  "detail": {
    "metadata": {
      "eventId": "0192b3c4-7d8e-7f10-a2b3-4c5d6e7f8091",
      "eventVersion": "1",
      "occurredAt": "2026-10-06T15:04:05Z",
      "correlationId": "5b1c9e0a-3f42-4c7d-9a1e-8d2f6b7c4e10"
    },
    "data": {
      "userId": "0192b3c4-7d8e-7f10-a2b3-4c5d6e7f8000",
      "documentType": "CC",
      "documentNumber": "1020304050",
      "firstName": "María Fernanda",
      "lastName": "Gómez Rojas",
      "birthDate": "1995-03-21",
      "gender": "female",
      "email": "maria.gomez@example.com",
      "phone": "+573001234567",
      "address": { "line": "Cra 15 # 93-60 Apto 402", "city": "Bogotá" }
    }
  }
}
```

**PasswordResetRequested** (consume `Notifications`)

```json
{
  "source": "kenko.auth",
  "detail-type": "PasswordResetRequested",
  "detail": {
    "metadata": {
      "eventId": "0192b3d1-1a2b-7c3d-8e4f-5a6b7c8d9e0f",
      "eventVersion": "1",
      "occurredAt": "2026-10-06T16:10:00Z",
      "correlationId": "c3d4e5f6-0a1b-4c2d-8e3f-9a0b1c2d3e4f"
    },
    "data": {
      "userId": "0192b3c4-7d8e-7f10-a2b3-4c5d6e7f8000",
      "email": "maria.gomez@example.com",
      "firstName": "María Fernanda",
      "resetToken": "q8Zs1x...base64url...",
      "expiresAt": "2026-10-06T16:30:00Z"
    }
  }
}
```

---

## 3. Infraestructura AWS (2026)

Servicios: **EventBridge** (bus y reglas), **SQS** (colas y DLQ) y **RDS PostgreSQL**. La infraestructura se define con **Terraform**.

- **Event Bus (el bus que se crea es `personalizado`, no `classic`):** `kenko-dev-bus`

| Consumidor | Regla | Cola | DLQ |
| --- | --- | --- | --- |
| `Profile` | `auth-to-profile` | `kenko-dev-profile-auth` | `kenko-dev-profile-auth-dlq` |
| `Notifications` | `auth-to-notifications` | `kenko-dev-notifications-auth` | `kenko-dev-notifications-auth-dlq` |

---

## 4. Tecnologías

Si hace falta otra librería, se analiza según la necesidad.

- **Lenguaje:** Go 1.27 (última versión), solo con la librería estándar: `net/http` y `NewServeMux`.
- **Estándar de Go:**
  - `uuid` (nativo desde Go 1.27): solo **`uuid v7`** para todos los identificadores. **Prohibido `uuid v4`.**
  - `log/slog`: logging de toda la aplicación.
- **Externas:**
  - `bcrypt` (`golang.org/x/crypto/bcrypt`)
  - `pgx` (PostgreSQL): <https://github.com/jackc/pgx>
  - `validator`: <https://github.com/go-playground/validator>
  - `jwx` (JWT): <https://github.com/lestrrat-go/jwx>
  - `pquerna/otp` (TOTP): <https://github.com/pquerna/otp>
  - `godotenv`: <https://github.com/joho/godotenv>
  - `air` (hot reload): <https://github.com/air-verse/air>
- **Herramientas:** Docker, Docker Compose, AWS SDK para Go.
- **Base de datos:** PostgreSQL (RDS).

### Posible uso futuro (NO implementar sin orden explícita)

Observabilidad y monitoreo en AWS, OpenTelemetry, Prometheus, Grafana, `gobreaker`, `swag`, `env` (caarlos0), `cors` (rs/cors) y migraciones de base de datos. Se deben tener presentes para el futuro, pero no se implementan hasta que yo lo pida.

---

## 5. Reglas

**Código:**

- Prioriza código reutilizable. Si algo se repite con frecuencia, conviértelo en una funcionalidad común.
- Usa código moderno y buenas prácticas, apoyándote en las `skills` del proyecto (`.agents/skills/`).
- Todo nativo: sin frameworks web.

**Logging:**

- Usa siempre `log/slog`, centralizado en un único lugar (no llamarlo ni pasarlo en todas partes). La estrategia (inyección de dependencias u otra) la definimos sobre la marcha.

**Datos y tiempo:**

- Guarda todo en **UTC**. En Postgres la columna es `timestamptz`, nunca `timestamp`. Convierte a la zona del usuario solo al mostrar, nunca al guardar.

**Consistencia entre base de datos y eventos:**

- Si se guarda el usuario en Postgres y luego falla la publicación, el evento se pierde. Usa el **patrón outbox** (tabla + relay).

**Seguridad:**

- **JWT** con buenas prácticas: Access Token, Refresh Token, rotación de tokens y revocación.
- El token viaja en cookies **HttpOnly** (inaccesibles para JavaScript) y **Secure** (solo conexiones cifradas), con protección contra CSRF.

**Resiliencia entre microservicios:**

- Si un servicio se vuelve lento y agota el tiempo de espera, se puede aplicar: timeouts, circuit breakers, fallback y retries con backoff. Debe ser simple, claro, fácil de mantener y de testear.
- No es necesario implementar todos. **Pregúntame siempre antes de implementar cualquiera.**

---

## 6. Arquitectura y estructura del proyecto

La arquitectura y las carpetas ya están definidas. **Analiza los directorios (vacíos y con archivos) antes de proponer cambios.**

Arquitectura: **hexagonal** con **vertical slice** e inversión de dependencias.
