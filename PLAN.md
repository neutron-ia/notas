# Plan tecnico - StockHub

Reto Lite Thinking Consulting. Backend Spring Boot + Frontend Quasar/Vue, desplegado en AWS EC2.

- Candidato: Mario David Alvarez Vallejo
- GitHub: MrDavidAlv
- Repositorios objetivo: `stockhub-api`, `stockhub-web`
- URL produccion objetivo: `http://3.93.170.171:8080`

Documento vivo. Progreso diario en `progresso.md`. Contexto para nuevas sesiones en `CLAUDE.md`.

---

## 1. Objetivo y mapeo

Construir una aplicacion full stack que cumpla el reto de Lite Thinking y demuestre las habilidades de la vacante. Cada decision esta justificada por uno de los dos.

| Requisito del reto | Decision en el plan | Habilidad de la vacante |
|---|---|---|
| Java + Spring Boot + Hibernate | Spring Boot 3.5 + Spring Data JPA | Spring Boot, Hibernate, JPA |
| Quasar + Vue.js | Quasar v2 + Vue 3 + TypeScript + Pinia | Vue.js, Quasar, TypeScript |
| PostgreSQL | PostgreSQL 16 + Flyway | PostgreSQL, modelado relacional |
| SOLID + Clean Architecture | Capas domain / application / infrastructure / interfaces | Clean Architecture, SOLID |
| Password encriptada | BCrypt cost 12 + seeder en runtime | Seguridad, autenticacion |
| Roles ADMIN / EXTERNO | Spring Security + @PreAuthorize | Spring Security, OAuth, JWT |
| PDF + envio email | iText7 + JavaMailSender | Integraciones, APIs |
| Despliegue AWS | Docker + GitHub Actions + EC2 | AWS, Docker, CI/CD |
| Pruebas unitarias | JUnit 5 + Mockito + Testcontainers | Calidad, testing |
| Documentacion | README + Swagger UI + diagrama ER | Documentacion tecnica |
| Uso de IA | Claude Code documentado en CLAUDE.md | Desarrollo asistido por IA |

---

## 2. Arquitectura

### 2.1 Vista general

```
                     GitHub
              stockhub-api    stockhub-web
                  |               |
                  | push main     | push main
                  v               v
              GH Actions       GH Actions
              build + test     build + lint
                  |               |
                  v               v
              docker push      docker push
                  |               |
                  +-------+-------+
                          |
                          v
             ghcr.io/mrdavidalv/stockhub-*
                          |
                          v
              AWS EC2  3.93.170.171
              ----------------------
              /opt/stocknova   puerto 80   (ya existente)
              /opt/stockhub    puerto 8080 (nuevo)
                  - nginx :8080 (reverse proxy)
                  - web   (Quasar SPA)
                  - api   (Spring Boot)
                  - db    (PostgreSQL)
```

### 2.2 Modelo en desarrollo (Docker Compose)

```
Browser :9000  ->  Quasar dev server
HTTP request   ->  Spring Boot API :8080
JDBC           ->  PostgreSQL :5433  (host:5433 -> container:5432)
SMTP           ->  Mailpit :1025  (UI :8025)
```

### 2.3 Modelo en produccion (EC2 :8080)

```
Cliente -> http://3.93.170.171:8080/  -> Nginx -> web (Quasar SPA)
                                    /api/     -> api (Spring Boot 8080 interno)
                                    /swagger  -> api
                                    /actuator -> api (solo health/info publicos)
```

---

## 3. Estructura de repositorios

### 3.1 `stockhub-api`

Arquitectura hexagonal alineada con `MrDavidAlv/ApiRest-Java-SpringBoot-PostgreSQL`.

```
stockhub-api/
  .github/workflows/ci.yml
  deploy/
    setup-ec2.sh
    deploy.sh
    nginx.conf
  src/main/java/com/stockhub/
    StockHubApplication.java
    domain/
      model/        (Empresa, Producto, Categoria, Cliente, Orden, OrdenProducto, PrecioMoneda, Usuario, RefreshToken, Rol enum)
      port/         (interfaces de repositorios y puertos de salida: PdfPort, MailPort)
      exception/    (NotFoundException, DuplicateNitException, InvalidCredentialsException)
    application/
      service/      (interfaces de casos de uso)
      service/impl/ (implementaciones)
      dto/          (Request y Response DTOs)
      mapper/       (MapStruct)
    infrastructure/
      persistence/  (adapters JPA que implementan los puertos del dominio)
      security/     (JwtService, JwtAuthFilter, UserDetailsServiceImpl, RefreshTokenService, SecurityConfig)
      pdf/          (PdfAdapter con iText7)
      mail/         (MailAdapter con JavaMailSender)
      seeder/       (DataSeeder CommandLineRunner)
      config/       (OpenApiConfig, CorsConfig, BeanConfig)
    interfaces/
      rest/         (controllers REST, request mappers)
      rest/exception/ (GlobalExceptionHandler, ErrorResponse)
  src/main/resources/
    application.yml
    application-dev.yml
    application-prod.yml
    db/migration/   (V1__create_schema.sql, V2__seed_catalogo.sql)
  src/test/java/com/stockhub/
    unit/           (services, JwtService, PdfAdapter)
    integration/    (Testcontainers + MockMvc)
  Dockerfile
  docker-compose.yml
  docker-compose.prod.yml
  .env.example
  .dockerignore
  pom.xml
  README.md
```

### 3.2 `stockhub-web`

```
stockhub-web/
  .github/workflows/ci.yml
  src/
    pages/        (LoginPage, EmpresaPage, ProductoPage, InventarioPage)
    components/   (EmpresaForm, ProductoForm, PrecioMonedaList, InventarioTable)
    stores/       (auth.store.ts, empresa.store.ts, producto.store.ts)
    services/     (api.service.ts, auth.service.ts, empresa.service.ts, producto.service.ts, inventario.service.ts)
    router/       (index.ts con guards por rol)
    layouts/      (MainLayout, AuthLayout)
    types/        (interfaces TS compartidas)
  Dockerfile
  nginx.conf
  quasar.config.ts
  package.json
  .env.example
  .dockerignore
  README.md
```

---

## 4. Modelo entidad-relacion

```
USUARIO       (id PK, email UQ, password_hash, nombre, rol [ADMIN|EXTERNO], activo, created_at)
REFRESH_TOKEN (id PK, usuario_id FK, token UQ, expires_at, revoked, created_at)

EMPRESA       (nit PK, nombre, direccion, telefono, created_at)
CATEGORIA     (id PK, nombre UQ, descripcion)
PRODUCTO      (id PK, codigo UQ, nombre, caracteristicas, empresa_nit FK, created_at)
PRECIO_MONEDA (id PK, producto_id FK, moneda [COP|USD|EUR|GBP], precio, UQ(producto_id, moneda))
PRODUCTO_CATEGORIA (producto_id FK, categoria_id FK)    -- N:M

CLIENTE       (id PK, nombre, email, telefono)
ORDEN         (id PK, cliente_id FK, empresa_nit FK, fecha, estado, total)
ORDEN_PRODUCTO (orden_id FK, producto_id FK, cantidad, precio_unitario)  -- N:M
```

Reglas del reto cubiertas: Producto N:M Categoria, Cliente 1:N Ordenes, Orden N:M Producto.

---

## 5. Seguridad

- Hashing: BCrypt cost 12.
- Autenticacion: JWT access token 60 min + refresh token 7 dias persistido en BD con rotacion al refresh.
- Autorizacion: Spring Security a nivel de metodo con `@PreAuthorize("hasRole('ADMIN')")`.
- CORS: configurado con origenes desde variable de entorno `CORS_ORIGINS`.
- Headers OWASP: aplicados en el Nginx reverse-proxy (CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy).
- Endpoints publicos: `POST /api/auth/login`, `POST /api/auth/refresh`, `GET /api/empresas/**`, `/actuator/health`, `/swagger-ui/**`, `/v3/api-docs/**`.
- Endpoints ADMIN: CRUD escritura de Empresa, todo Producto, todo Inventario, todo Orden.

### 5.1 Matriz de permisos

| Accion | ADMIN | EXTERNO |
|---|---|---|
| Ver empresas | Si | Si |
| Crear/editar/eliminar empresa | Si | No |
| Registrar/editar productos | Si | No |
| Ver inventario | Si | No |
| Descargar PDF inventario | Si | No |
| Enviar PDF por email | Si | No |

---

## 6. API REST

```
Auth
POST   /api/auth/login        { email, password }              -> { accessToken, refreshToken, rol, nombre }
POST   /api/auth/refresh      { refreshToken }                  -> { accessToken, refreshToken }
POST   /api/auth/logout       { refreshToken }                  -> 204

Empresas
GET    /api/empresas
GET    /api/empresas/{nit}
POST   /api/empresas                                            (ADMIN)
PUT    /api/empresas/{nit}                                      (ADMIN)
DELETE /api/empresas/{nit}                                      (ADMIN)

Productos
GET    /api/productos
GET    /api/productos/{id}
GET    /api/productos/por-empresa/{nit}
POST   /api/productos                                           (ADMIN)
PUT    /api/productos/{id}                                      (ADMIN)
DELETE /api/productos/{id}                                      (ADMIN)

Inventario                                                      (todos ADMIN)
GET    /api/inventario?empresaNit=...
GET    /api/inventario/pdf?empresaNit=...
POST   /api/inventario/enviar-email

Catalogos
GET    /api/categorias
GET    /api/clientes
GET    /api/ordenes

Observabilidad
GET    /actuator/health
GET    /actuator/info
GET    /swagger-ui.html
GET    /v3/api-docs
```

---

## 7. Etapas de desarrollo

Tiempo total estimado: 30 a 36 horas en 5 dias.

### Etapa 0. Setup inicial (~2 h)

1. Crear los dos repos publicos en GitHub.
2. Configurar 13 secrets en cada repo (ver seccion 8).
3. Generar backend con Spring Initializr: Maven, Java 17, Spring Boot 3.5.14, group `com.stockhub`, artifact `stockhub-api`. Dependencias: Web, Security, Data JPA, Validation, PostgreSQL Driver, Flyway, Lombok, Actuator, DevTools, Testcontainers.
4. Anadir al pom: JJWT 0.12.6, iText 8.0.5, springdoc-openapi 2.7.0 (la 2.6.0 es incompatible con Spring Boot 3.5 por cambio binario en `ControllerAdviceBean`), MapStruct 1.6.3 (con lombok-mapstruct-binding 0.2.0 en el annotation processor), spring-boot-starter-mail.
5. Generar frontend con Quasar CLI: Quasar v2, TypeScript, Vite, Composition API, Vue Router, Pinia, Axios, ESLint.

### Etapa 1. Base de datos y configuracion (~2 h)

1. `docker-compose.yml` local con `postgres:16-alpine` y `axllent/mailpit`.
2. `application.yml` con perfiles dev y prod, variables de entorno con defaults razonables.
3. Migracion `V1__create_schema.sql` (DDL completo de seccion 4).
4. Migracion `V2__seed_catalogo.sql` con 3 categorias y 2 empresas de ejemplo.
5. Los usuarios NO van en Flyway. Los crea `DataSeeder` en runtime con BCrypt.

### Etapa 2. Seguridad y autenticacion (~5 h)

1. Entidades JPA: Usuario, RefreshToken, Empresa, Producto, PrecioMoneda, Categoria, Cliente, Orden, OrdenProducto. Relaciones LAZY por defecto.
2. `JwtService` (generate, extract username, validate).
3. `RefreshTokenService` (create, rotate, revoke).
4. `JwtAuthFilter` extiende `OncePerRequestFilter`.
5. `SecurityConfig` con `@EnableMethodSecurity`, CORS, sesion STATELESS.
6. `PasswordEncoder` = `BCryptPasswordEncoder(12)`.
7. `DataSeeder` implementa `CommandLineRunner` y hashea las passwords leyendo `seed.admin-password` y `seed.externo-password` desde `application.yml`.
8. `AuthController` con `/login`, `/refresh`, `/logout`.

### Etapa 3. CRUD Empresa y Producto (~4 h)

Patron por recurso:
1. `XxxRepository extends JpaRepository<Xxx, Id>`.
2. `XxxRequest` con Bean Validation, `XxxResponse`.
3. `XxxMapper` MapStruct.
4. `XxxService` interface + `XxxServiceImpl`.
5. `XxxController` con `@PreAuthorize` en endpoints de escritura.

Recursos: Empresa, Producto, Categoria, Cliente, Orden.

`GlobalExceptionHandler` maneja: 404 NotFound, 400 Validation con mapa de errores por campo, 403 AccessDenied, 409 Conflict, 500 generico sin filtrar stack.

### Etapa 4. Inventario, PDF, Email (~3 h)

1. `PdfAdapter` implementa `PdfPort`. Genera tabla con columnas: Codigo, Nombre, Empresa, Precio COP, Categorias.
2. `MailAdapter` implementa `MailPort`. Usa `JavaMailSender` con `MimeMessageHelper`.
3. `InventarioController` con tres endpoints (listar, descargar PDF, enviar por email).
4. En dev, el email apunta a Mailpit; en prod, a Gmail con App Password.

### Etapa 5. Tests (~3 h)

- Unit (sin Spring): EmpresaServiceImplTest, ProductoServiceImplTest, JwtServiceTest, PdfAdapterTest. Mockito.
- Integration (Testcontainers PostgreSQL 16): AuthIntegrationTest, EmpresaCrudIntegrationTest.

Meta: al menos 15 tests, cobertura mayor a 70% en `service` y `infrastructure.security`.

### Etapa 6. Dockerfile backend (~30 min)

Multi-stage con Eclipse Temurin 17. Usuario no-root `app`. HEALTHCHECK con `wget` contra `/actuator/health`. JAVA_OPTS con `MaxRAMPercentage=75`.

### Etapa 7. Frontend setup y auth (~2 h)

1. `services/api.service.ts` con Axios + interceptores para inyectar `Authorization: Bearer` y para hacer refresh automatico en 401 con `_retry` flag.
2. `stores/auth.store.ts` con Pinia: `accessToken`, `refreshToken`, `user` en localStorage; getters `isAuthenticated` e `isAdmin`; acciones `login`, `tryRefresh`, `logout`.
3. `router/index.ts` con `beforeEach` que valida `requiresAuth` y `requiresAdmin`.

### Etapa 8. Vistas Quasar (~5 h)

| Vista | Componentes clave |
|---|---|
| LoginPage | QCard con QForm + QInput email/password + QBtn submit con loader |
| EmpresaPage | QTable con columnas NIT, Nombre, Direccion, Telefono. Botones Nuevo/Editar/Eliminar visibles solo si `isAdmin`. Dialog con EmpresaForm |
| ProductoPage | QTable filtrable por empresa. Dialog con ProductoForm: codigo, nombre, caracteristicas, select empresa, select multiple categorias, sub-tabla PrecioMonedaList |
| InventarioPage | QTable agrupada por empresa. Botones Descargar PDF (blob) y Enviar Email (dialog con campo email) |
| MainLayout | QHeader con tabs condicionales segun rol |

### Etapa 9. Dockerfile y Nginx frontend (~30 min)

Multi-stage: build con Node 20, serve con Nginx alpine. Aplica headers OWASP. SPA fallback `try_files`. Cache largo para assets versionados.

### Etapa 10. Infraestructura EC2 (~2 h)

1. `docker-compose.prod.yml` con servicios postgres, api, web, nginx. Red bridge. `nginx` mapea `8080:80`.
2. `deploy/nginx.conf` reverse-proxy con headers OWASP y rutas `/`, `/api/`, `/swagger`, `/v3/api-docs`, `/actuator/health`.
3. `deploy/setup-ec2.sh` idempotente (no reinstala Docker si ya existe). Crea `/opt/stockhub`. Recuerda abrir puerto 8080 en Security Group.
4. `deploy/deploy.sh` con pull, up -d, prune, healthcheck loop de 6 intentos por 10s contra `http://localhost:8080/actuator/health`.

### Etapa 11. CI/CD GitHub Actions (~2 h)

Pipeline backend (`stockhub-api/.github/workflows/ci.yml`) con 3 jobs:

1. `build-and-test`: setup-java 17 temurin, cache maven, `mvn verify`, upload JaCoCo.
2. `docker-push` (solo push a main): build y push a `ghcr.io/mrdavidalv/stockhub-api:latest` y `:${{ github.sha }}`.
3. `deploy` (solo push a main): scp de `docker-compose.prod.yml`, `deploy/nginx.conf`, `deploy/deploy.sh`. Genera `.env` con heredoc desde secrets. Ejecuta `bash deploy/deploy.sh`.

Pipeline frontend equivalente: `npm ci` + `npm run build` + docker push + deploy (`docker compose pull web && up -d`).

Opcional: job `sonar` que solo corre si existe el secret `SONAR_TOKEN`.

### Etapa 12. README y documentacion (~1 h)

READMEs cortos, profesionales, sin emojis. Maximo 200 lineas cada uno. Ver seccion 9 para reglas de estilo.

### Etapa 13. Validacion final (~1 h)

Recorrido funcional completo con ambos usuarios en local y en produccion. Checklist final (seccion 10).

---

## 8. Secrets de GitHub

Configurar en Settings -> Secrets and variables -> Actions, en ambos repos.

| Secret | Descripcion |
|---|---|
| EC2_HOST | IP publica del EC2 (`3.93.170.171`) |
| EC2_SSH_KEY | Llave privada SSH en formato OpenSSH |
| CR_PAT | Personal Access Token con scope `read:packages` |
| POSTGRES_DB | `stockhub_db` |
| POSTGRES_USER | `stockhub` |
| POSTGRES_PASSWORD | Generar con `openssl rand -base64 24` |
| JWT_SECRET | Generar con `openssl rand -base64 48` |
| MAIL_USERNAME | Gmail del candidato |
| MAIL_PASSWORD | App Password de Gmail (no la password normal) |
| CORS_ORIGINS | `http://3.93.170.171:8080,http://localhost:9000` |
| SEED_ADMIN_PASSWORD | Password inicial del admin |
| SEED_EXTERNO_PASSWORD | Password inicial del externo |
| SONAR_TOKEN | Opcional. Si esta presente activa job de SonarCloud |

---

## 9. Reglas de documentacion

Para evitar que los READMEs y comentarios parezcan generados por IA:

- Sin emojis. Sin iconos. Sin caracteres unicode decorativos (`->`, `<-`, `*`, etc.). Solo ASCII basico + tildes y enie del espanol.
- Sin frases promocionales tipo "potente solucion", "robusto", "moderno y escalable".
- Sin listas exhaustivas redundantes. Si una lista tiene mas de 10 items, partir en sub-secciones.
- READMEs maximo 200 lineas. Estructura objetiva:
  1. Titulo + una linea de descripcion.
  2. Stack (tabla corta).
  3. Como correr en local (3-5 comandos).
  4. Variables de entorno (tabla).
  5. Endpoints principales o link a Swagger.
  6. Tests.
  7. Despliegue (1-2 parrafos).
  8. Credenciales de prueba.
- Comentarios en codigo: solo cuando el porque no es obvio. Nunca explicar el que.
- Mensajes de commit en espanol, formato conciso: `feat: descripcion corta`, `fix: descripcion corta`, `docs: descripcion corta`.

---

## 10. Checklist de entrega

### Funcionalidad del reto

- [ ] Vista Empresa con CRUD segun rol
- [ ] Vista Producto con codigo, nombre, caracteristicas, precio multi-moneda, empresa
- [ ] Vista Login con email y password
- [ ] Vista Inventario con descarga PDF y envio por email
- [ ] Usuario ADMIN con permisos completos
- [ ] Usuario EXTERNO solo ve empresas
- [ ] Password encriptada con BCrypt
- [ ] BD con Empresa, Productos, Categorias, Clientes, Ordenes
- [ ] Producto N:M Categoria
- [ ] Cliente 1:N Ordenes
- [ ] Orden N:M Producto

### Calidad

- [ ] Arquitectura hexagonal (domain, application, infrastructure, interfaces)
- [ ] SOLID aplicado y mencionado brevemente en README
- [ ] Al menos 15 tests
- [ ] Cobertura mayor a 70% en service e infrastructure.security
- [ ] Swagger UI funcionando
- [ ] GlobalExceptionHandler con respuestas consistentes
- [ ] Validaciones Bean Validation en backend y Quasar rules en frontend

### DevOps

- [ ] Dockerfile backend multi-stage con usuario no-root y HEALTHCHECK
- [ ] Dockerfile frontend multi-stage con Nginx y headers OWASP
- [ ] `docker-compose up` levanta todo en local
- [ ] CI/CD GitHub Actions en ambos repos en verde
- [ ] Imagenes en ghcr.io
- [ ] Deploy automatico a EC2 en cada push a main
- [ ] App accesible en `http://3.93.170.171:8080`
- [ ] Spring Boot Actuator respondiendo
- [ ] Stocknova en :80 sin tocar y funcionando

### Entregables

- [ ] Repo `stockhub-api` publico con README corto
- [ ] Repo `stockhub-web` publico con README corto
- [ ] Credenciales ADMIN y EXTERNO documentadas
- [ ] URL de la app desplegada
- [ ] `progresso.md` con bitacora
- [ ] Email con todos los entregables enviado a Lite Thinking antes de la fecha limite

---

## 11. Estimacion

| Etapa | Horas |
|---|---:|
| 0. Setup inicial | 2 |
| 1. BD + Compose dev + Flyway | 2 |
| 2. Seguridad JWT + Seeder | 5 |
| 3. CRUD Empresa + Producto + Categoria | 4 |
| 4. Inventario + PDF + Email | 3 |
| 5. Tests | 3 |
| 6. Dockerfile backend | 0.5 |
| 7. Frontend setup + auth | 2 |
| 8. Vistas Quasar | 5 |
| 9. Dockerfile frontend | 0.5 |
| 10. Infra EC2 | 2 |
| 11. CI/CD | 2 |
| 12. README | 1 |
| 13. Validacion final | 1 |
| Total | 33 |

---

## 12. Referencias

- Reto: `Docs/Reto Pratico.pdf`
- Vacante: `Docs/Vacante.txt`
- Proyecto previo del candidato (estilo a replicar): https://github.com/MrDavidAlv/ApiRest-Java-SpringBoot-PostgreSQL
- Proyecto previo en EC2 (patron de despliegue): https://github.com/MrDavidAlv/stocknova-api y https://github.com/MrDavidAlv/stocknova-web
- Spring Initializr: https://start.spring.io
- Quasar v2: https://quasar.dev
- JJWT 0.12: https://github.com/jwtk/jjwt
- iText 8: https://itextpdf.com/products/itext-core
- Flyway: https://documentation.red-gate.com/fd
- SpringDoc: https://springdoc.org
- Mailpit: https://mailpit.axllent.org
- Testcontainers: https://testcontainers.com
