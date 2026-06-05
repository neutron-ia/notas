# Reporte de calidad - StockHub (API + Web)

Fecha: 2026-06-05. Alcance: `stockhub-api` y `stockhub-web` en el estado actual del workspace.
Revisado: capa de seguridad, REST, servicios, persistencia, frontend (store/router/interceptores),
Dockerfiles, nginx, docker-compose y workflows de CI/CD. Cruzado con `Docs/CHECKLIST-RETO.md`.

Severidad por palabras: Critico, Alto, Medio, Bajo, Informativo.

Nota de contexto: es un reto tecnico, no un sistema en produccion real. Varios hallazgos
"Alto" en terminos absolutos son limitaciones aceptadas y documentadas del reto (ej. sin HTTPS).
Se marcan igual para que la decision sea explicita.

---

## 1. Resumen ejecutivo

El proyecto esta bien construido para un reto: arquitectura hexagonal limpia, seguridad JWT
con refresh rotado y persistido, manejo de errores centralizado y consistente, validacion en
DTOs, headers OWASP, CSP estricto, Docker no-root con healthcheck, y CI/CD con tests de
integracion sobre Testcontainers. Cumple los requisitos del enunciado uno a uno.

Lo que conviene atender, en orden:

1. Manejo de `DataIntegrityViolationException` ausente: varias entradas invalidas devuelven 500
   en vez de 400/409 (bug funcional visible).
2. Carrera en el refresh del frontend: peticiones concurrentes tras expirar el access token
   pueden desloguear al usuario por la rotacion de refresh tokens.
3. `ENTRYPOINT` con `sh -c` sin `exec`: el apagado graceful del contenedor no funciona.
4. Sin rate limiting en login (fuerza bruta) y tokens en `localStorage` (exposicion ante XSS).
5. N+1 al listar productos/inventario.

---

## 2. Seguridad

### 2.1 Alto

- Sin HTTPS en produccion. La app sirve en `http://3.93.170.171:8080`. Access y refresh tokens
  viajan en claro; interceptables en red. Es el techo conocido (Lighthouse BP 78). Para el reto
  es aceptable y esta documentado; para uso real es bloqueante. Minimo: TLS via Caddy/Traefik o
  Nginx + Let's Encrypt, o un ALB delante del EC2.

- Login sin rate limiting ni bloqueo por intentos. `/api/auth/login` es `permitAll` y no hay
  throttling. BCrypt cost 12 encarece cada intento pero no frena credential stuffing. Anadir
  bucket4j a nivel app o `limit_req` en nginx para `/api/auth/login`.

### 2.2 Medio

- Tokens en `localStorage` (`stockhub.accessToken`, `stockhub.refreshToken`). Ante un XSS se
  roban ambos, incluyendo el refresh de 7 dias. El riesgo esta mitigado por un CSP estricto
  (`script-src 'self'`), lo que reduce mucho la probabilidad de XSS, pero el patron sigue siendo
  el menos seguro. Alternativa: refresh token en cookie `HttpOnly; Secure; SameSite=Strict` y
  access token solo en memoria. Para el reto es defendible si se justifica el trade-off.

- Refresh no revalida `usuario.activo`. `RefreshTokenService.rotate` valida vigencia y revocacion
  del token, pero no comprueba que el usuario siga activo. Un usuario desactivado con refresh
  vigente puede seguir emitiendo access tokens. El impacto real esta acotado porque
  `JwtAuthFilter.authenticate` si chequea `isEnabled()` en cada peticion, asi que ese access token
  no autenticaria. Aun asi, conviene revocar los refresh tokens del usuario al desactivarlo y/o
  chequear `activo` dentro de `rotate`.

### 2.3 Bajo / Informativo

- Rotacion de refresh sin deteccion de reuso (token families). Si un refresh se filtra, no hay
  mecanismo que detecte el uso del token viejo ya rotado para invalidar toda la familia. Para el
  reto es suficiente; en real se anade deteccion de reuso.

- `CorsConfiguration.setAllowCredentials(true)` es innecesario: la API usa Bearer en header, no
  cookies. Combinado con `allowedHeaders("*")` amplia superficie sin aportar. Poner `false`.

- Swagger UI y `/v3/api-docs` publicos en prod. Es intencional para la evaluacion, pero expone el
  contrato completo. En real se restringe por red o auth.

- `actuator/metrics` esta en `exposure.include` y requiere auth (cae en `anyRequest().authenticated()`),
  ademas el nginx de prod solo proxya `/actuator/health`, asi que no es accesible publicamente. OK,
  se deja la nota para que sea decision consciente.

- Limpieza de refresh tokens: los expirados/revocados nunca se borran. Crecimiento ilimitado de la
  tabla. Anadir un job programado o borrado en `rotate`/`revoke`. (Tambien es deuda operacional.)

- Cuidado con `Docs/SECRETS-VALUES.md`: contiene credenciales reales. Verificar que NO quede en
  ningun repo git (`stockhub-api`/`stockhub-web`); vive en el workspace padre, que no es repo, pero
  conviene confirmarlo antes de cualquier `git add` amplio.

---

## 3. Bugs

### 3.1 Medio

- `DataIntegrityViolationException` no se maneja en `GlobalExceptionHandler`. Cualquier violacion de
  constraint de BD cae en el handler generico y devuelve `500 INTERNAL_ERROR`. Casos reales:
  - Enviar dos precios con la misma `moneda` para un producto (constraint `UNIQUE(producto_id, moneda)`).
  - Carreras en alta de empresa/producto duplicado entre el `existsBy...` y el `save`.
  Deberia responder `409 CONFLICT` o `400`. Anadir un `@ExceptionHandler(DataIntegrityViolationException.class)`.

- `ProductoRequest.precios` no valida monedas duplicadas. Hoy el unico freno es el constraint de BD,
  que produce el 500 anterior. Validar unicidad de `moneda` en la lista (validador a nivel de record
  o chequeo en `ProductoServiceImpl.attachPrecios`).

- Carrera en el refresh del frontend (`api.service.ts`). Si varias peticiones reciben 401 a la vez
  (access token recien expirado), cada una llama `auth.tryRefresh()` por separado. Como el backend
  rota el refresh (revoca el viejo, emite uno nuevo), la primera rotacion invalida el token que usan
  las demas, que fallan y disparan `auth.clear()`: deslogueo espurio. Implementar single-flight: una
  unica promesa de refresh compartida y encolar las peticiones que llegan mientras se refresca.

### 3.2 Bajo

- `Dockerfile` (api): `ENTRYPOINT ["sh","-c","java $JAVA_OPTS -jar app.jar"]`. `sh` queda como PID 1 y
  no reenvia `SIGTERM` a java, asi que `docker stop` no apaga la app de forma graceful (espera y luego
  `SIGKILL`). Usar `exec`: `ENTRYPOINT ["sh","-c","exec java $JAVA_OPTS -jar app.jar"]`. Combinar con
  `server.shutdown: graceful` en `application.yml`.

---

## 4. Rendimiento

### 4.1 Medio

- N+1 al listar productos e inventario. `ProductoServiceImpl.findAll`/`findByEmpresa` e
  `InventarioServiceImpl.getInventario`/`generarPdf` mapean cada `Producto` accediendo a `precios`
  (OneToMany LAZY) y `categorias` (ManyToMany LAZY). Con muchos productos se dispara una consulta por
  coleccion y producto. Usar `@EntityGraph` o `JOIN FETCH` en las queries de listado. Es el endpoint
  con mas probabilidad de crecer (inventario completo / PDF).

### 4.2 Bajo

- `InventarioServiceImpl.generarPdf` con `empresaNit` consulta la empresa dos veces: `loadProductos`
  hace `existsById` + `findByEmpresaNit`, y luego `generarPdf` llama `loadEmpresa` (`findById`).
  Reutilizar una sola carga.

- `JwtService` parsea el token dos veces por peticion: `isValid(token)` y luego `extractEmail(token)`.
  Parsear una vez y reutilizar los claims (o exponer un metodo que valide y devuelva el subject).

---

## 5. Code smells y mantenibilidad

### API

- Cobertura de tests desigual. Hay unit de `JwtService`, `JwtAuthFilter`, `RefreshTokenService`,
  `PdfAdapter`, `ProductoServiceImpl`, `EmpresaServiceImpl`, e integration de Auth y Empresa. Faltan:
  `AuthServiceImpl`, `InventarioServiceImpl`, `CategoriaServiceImpl`, e integration de Producto e
  Inventario (incluido el caso de moneda duplicada que hoy da 500). El objetivo del plan es >=70% en
  `service` y `infrastructure.security`: conviene cerrar esos huecos.

- Entidades `Cliente`, `Orden`, `OrdenProducto` y sus repos (`ClienteRepository`, `OrdenRepository`)
  existen para cumplir el modelo ER del enunciado pero no tienen servicio ni controller: codigo sin uso
  funcional. Es aceptable porque el reto pide el modelo, no su CRUD; dejarlo documentado para que no se
  lea como olvido.

- Inconsistencia de propiedad de BD: `application.yml` fija `spring.datasource.password: ${POSTGRES_PASSWORD}`,
  pero `docker-compose.prod.yml` inyecta `SPRING_DATASOURCE_PASSWORD`. Funciona por precedencia (la env
  var sobreescribe el yml), pero es confuso. Unificar a una sola variable.

- README dice "Spring Boot 3.5" y el `pom.xml` usa 3.5.14, mientras `CLAUDE.md`/`PLAN.md` fijan 3.3.x.
  Alinear la documentacion de decisiones con la version real.

### Web

- Restos del scaffold de Quasar sin uso: `components/ExampleComponent.vue`, `components/EssentialLink.vue`,
  `components/models.ts`, `pages/IndexPage.vue` (las rutas redirigen a `/empresas`, no lo usan) y
  `assets/quasar-logo-vertical.svg`. Borrarlos para que el repo refleje solo lo del reto.

- `baseURL` duplicado en `auth.store.ts` y `api.service.ts`. Centralizar en un unico modulo de config.

- El uso de `axios` crudo (no la instancia `api`) en login/refresh/logout es correcto para evitar el
  bucle del interceptor; dejarlo comentado como decision intencional.

- CI web (`stockhub-web/.github/workflows/ci.yml`) solo hace `build`. No corre `eslint` ni typecheck
  explicito pese a tener ESLint configurado. Anadir un paso `npm run lint` (la build de Quasar ya hace
  type-check, pero el lint queda fuera).

### CI/CD / Infra

- Sin escaneo de imagen ni de dependencias (Trivy/Dependabot). Anadir al menos Trivy a los workflows.
- `build-push-action` sin cache de capas: builds mas lentos. Anadir `cache-from`/`cache-to` (gha).
- El deploy escribe `.env` por SSH con heredoc citado (`'ENVEOF'`) y `chmod 600`: correcto.
- Acoplamiento documentado: el deploy de `stockhub-web` asume que el stack de `/opt/stockhub`
  (propiedad de `stockhub-api`) ya existe. Esta explicado en el workflow; ok para el reto.

### nginx

- `deploy/nginx.conf` (reverse proxy :8080) no aplica rate limiting ni HTTPS (ver seccion 2).
- Repeticion de los bloques de headers OWASP en `stockhub-web/nginx.conf` por la regla de herencia de
  `add_header` de nginx. Es correcto tecnicamente; se puede factorizar con un `include` de un snippet
  comun para reducir duplicacion y el riesgo de que un bloque quede desincronizado.

---

## 6. Lo que esta bien hecho (para no tocarlo)

- Arquitectura hexagonal coherente y puertos bien definidos.
- `JwtAuthFilter` con try/catch documentado y chequeo de `isEnabled()` en cada request (stateless real).
- `JwtAuthEntryPoint` y `RestAccessDeniedHandler` devuelven el mismo `ErrorResponse` que el resto.
- Validacion en DTOs (`@Email`, `@NotBlank`, `@Pattern` en NIT, `@Digits`/`@PositiveOrZero` en precios).
- BCrypt cost 12 con seeder en runtime (sin hashes en SQL).
- Flyway con `ddl-auto: validate` y `open-in-view: false`.
- Docker no-root con HEALTHCHECK; CSP estricto y headers OWASP; SPA fallback.
- Refresh token rotado y persistido con revocacion en logout.

---

## 7. Plan de accion sugerido (orden)

1. `@ExceptionHandler(DataIntegrityViolationException.class)` -> 409, y validar monedas unicas en
   `ProductoRequest`. Cubrir con un test.
2. Single-flight del refresh en `api.service.ts`.
3. `exec` en el `ENTRYPOINT` + `server.shutdown: graceful`.
4. `@EntityGraph`/`JOIN FETCH` en listados de producto e inventario.
5. Rate limiting en `/api/auth/login`.
6. Cerrar huecos de tests (`AuthServiceImpl`, `InventarioServiceImpl`, integration de Producto).
7. Limpieza de scaffold muerto en web y `allowCredentials(false)` en CORS.
8. (Si hay tiempo / para destacar) TLS delante del EC2.
