# Clinica PetSalud – Backend con Spring Security y JWT

API REST de la Clinica Veterinaria PetSalud con autenticacion sin estado mediante **JWT**, contraseñas cifradas con **BCrypt** y autorizacion por roles con **`@PreAuthorize`**.

Actividad sumativa Semana 3, *Integrando seguridad en aplicaciones Backend*, de Desarrollo Backend II (PBY2202), Duoc UC.
Autora: Heydis Herrera.

## Tecnologias

- Java 17
- Spring Boot 3.3.6 (Web, Data JPA, Security)
- jjwt 0.12.6
- MySQL 8
- Lombok

## Como ejecutar

1. Tener MySQL corriendo en `localhost:3306`. La base de datos `petsalud_db` se crea sola al iniciar.
2. Revisar `src/main/resources/application.properties` y ajustar el usuario y la contraseña de MySQL. `jwt.secret` debe ser una clave Base64 de al menos 32 bytes (256 bits).

   >  En un entorno real, `jwt.secret` y las credenciales deben configurarse con variables de entorno y no subirse al repositorio.

3. Ejecutar:

   ```bash
   ./mvnw spring-boot:run
   ```

   La API queda en `http://localhost:8080`.

## Autenticación

| Paso | Petición |
|---|---|
| Registrarse (siempre queda como `CLIENTE`) | `POST /api/usuarios/registro` |
| Iniciar sesión (devuelve el token) | `POST /api/auth/login` con `{"email": "...", "password": "..."}` |
| Usar la API | Header `Authorization: Bearer <token>` |

El token expira en 1 hora. Los roles `EMPLEADO` y `GERENTE` los asigna un administrador directamente en la base de datos.

## Roles y permisos

| Recurso / operación | CLIENTE | EMPLEADO | GERENTE |
|---|:-:|:-:|:-:|
| Ver productos | ✔ | ✔ | ✔ |
| Crear/editar productos, ver stock bajo | ✘ | ✔ | ✔ |
| Eliminar productos | ✘ | ✘ | ✔ |
| Agendar, ver y cancelar citas | ✔ | ✔ | ✔ |
| Cambiar estado de citas, agenda por veterinario o fechas | ✘ | ✔ | ✔ |
| Mascotas | ✔ | ✔ | ✔ |
| Expedientes médicos | ✘ | ✔ | ✔ |
| Ventas | ✘ | ✔ | ✔ |
| Ver y marcar notificaciones | ✔ | ✔ | ✔ |
| Enviar/eliminar notificaciones | ✘ | ✔ | ✔ |
| Ver recompensas | ✔ | ✔ | ✔ |
| Asignar/eliminar recompensas | ✘ | ✔ | ✔ |
| Listar/eliminar usuarios | ✘ | ✘ | ✔ |

Códigos de respuesta: **401** = sin token o token inválido · **403** = token válido, pero el rol no tiene permiso.

## Estructura de seguridad

```
security/
├── SecurityConfig.java         Cadena de filtros, rutas públicas, STATELESS, 401 y @EnableMethodSecurity
├── JwtUtil.java                Genera y valida tokens (HS256, expiración 1 h)
├── JwtFilter.java              Valida el token en cada petición y registra los tokens rechazados
├── UsuarioDetailsService.java  Carga usuarios desde MySQL por email
└── UsuarioDetails.java         Adapta Usuario a UserDetails con prefijo ROLE_
controller/AuthController.java  POST /api/auth/login; registra los logins fallidos
dto/LoginRequest, LoginResponse
```

## Medidas de seguridad

- **Contraseñas** cifradas con BCrypt; nunca se incluyen en las respuestas (`@JsonProperty(WRITE_ONLY)`).
- **Registro sin escalamiento de privilegios:** todo registro público queda con el rol `CLIENTE`.
- **Inyección SQL:** todas las consultas usan Spring Data JPA, que genera consultas parametrizadas.
- **CSRF:** la API no usa sesiones ni cookies; el token viaja en el header `Authorization`.
- **XSS:** las respuestas son JSON, con los headers de seguridad de Spring Security (`X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`).
- **Integridad del token:** firma HMAC-SHA256; un token alterado se rechaza con 401.
- **Monitoreo:** se registran con nivel `WARN` los logins fallidos y los tokens rechazados, con la IP y la ruta.
