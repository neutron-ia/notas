# Progreso - StockHub

Bitacora viva. Se actualiza al cierre de cada sesion o etapa.

Referencia tecnica: `PLAN.md`. Contexto de sesion: `CLAUDE.md`.

---

## Resumen

| Campo | Valor |
|---|---|
| Proyecto | StockHub |
| Candidato | Mario David Alvarez Vallejo |
| Reto | Lite Thinking Consulting - Full Stack Java/Vue |
| Stack | Java 17, Spring Boot 3.3, Hibernate, Vue 3, Quasar, PostgreSQL 16, Docker, GitHub Actions, AWS EC2 |
| Repo backend | github.com/MrDavidAlv/stockhub-api (pendiente de crear) |
| Repo frontend | github.com/MrDavidAlv/stockhub-web (pendiente de crear) |
| URL produccion | http://3.93.170.171:8080 (pendiente) |
| Inicio | 2026-06-03 |
| Entrega | Pendiente fecha del correo de Lite Thinking |

---

## Tablero de etapas

Estados: Pendiente, En progreso, Completado, Bloqueado.

| # | Etapa | Estado | Inicio | Fin | Notas |
|---:|---|---|---|---|---|
| 0 | Setup inicial | En progreso | 2026-06-03 | - | Scaffold local listo. Falta crear repos GitHub y subir secrets |
| 1 | BD + Docker Compose dev + Flyway | Completado | 2026-06-03 | 2026-06-03 | Postgres en host:5433 (host:5432 ocupado por Postgres nativo). Migraciones V1 y V2 aplicadas |
| 2 | Seguridad JWT + Seeder BCrypt | Completado | 2026-06-03 | 2026-06-03 | Auth verificado con curl: login OK, refresh con rotacion OK, logout revoca, 401/400/404 controlados |
| 3 | CRUD Empresa + Producto + Categoria | Completado | 2026-06-03 | 2026-06-03 | 19 endpoints documentados en Swagger. Producto con precios multi-moneda y N:M Categoria verificado |
| 4 | Inventario + PDF + Email | Completado | 2026-06-03 | 2026-06-03 | PDF iText 8 verificado en Mailpit (1788 bytes, valido). 3 endpoints nuevos en Swagger |
| 5 | Tests unit + integration | Completado | 2026-06-03 | 2026-06-03 | 38 tests pasando (27 unit + 11 integration), ~11s total |
| 6 | Dockerfile backend | Completado | 2026-06-03 | 2026-06-03 | Imagen 549MB, usuario `app` (uid 100), HEALTHCHECK con wget, container `healthy` y login verificado |
| 7 | Frontend setup + Auth | Completado | 2026-06-03 | 2026-06-03 | Pinia + Axios con refresh automatico, 5 services tipados, router guards. Build pasa |
| 8 | Vistas Quasar | Completado | 2026-06-03 | 2026-06-03 | MainLayout con tabs por rol, Login, Empresa, Producto, Inventario. Build pasa. Smoke E2E con CORS verificado |
| 9 | Dockerfile frontend | Completado | 2026-06-03 | 2026-06-03 | Imagen 94.7 MB, nginx con SPA fallback y 5 headers OWASP en root y assets |
| 10 | Infra EC2 | Completado | 2026-06-03 | 2026-06-03 | Stack completo verificado en local: 4 servicios, reverse-proxy en :8080, health UP, login OK, swagger OK |
| 11 | CI/CD GitHub Actions | Completado | 2026-06-03 | 2026-06-03 | 2 workflows con 3 jobs cada uno (build/test, docker push a ghcr.io, ssh deploy a EC2) |
| 12 | READMEs | Completado | 2026-06-03 | 2026-06-03 | api 152 lineas, web 100 lineas. Sin emojis ni unicode decorativo |
| 13 | Validacion final | Completado | 2026-06-03 | 2026-06-03 | 38/38 tests OK, build OK, stack productivo verificado E2E. Resto manual del usuario abajo |
| 14 | Secrets manual + remover hardcodes | Completado | 2026-06-04 | 2026-06-04 | Sin defaults sensibles en application.yml ni docker-compose. .env via spring.config.import. 4 docs en /Docs |
| 15 | Lighthouse: Accessibility + SEO + Perf (no HTTPS) | Completado | 2026-06-04 | 2026-06-04 | Desplegado y verificado en vivo: A11y 100, SEO 100, Perf 94 desktop / 88 mobile, BP 78 (techo HTTPS). |
| 16 | Endurecimiento de manejo de errores en seguridad | Completado | 2026-06-04 | 2026-06-04 | JwtAuthFilter con try/catch + chequeo isEnabled; AuthenticationEntryPoint y AccessDeniedHandler con ErrorResponse. 42 tests OK. Pendiente push |

---

## Decisiones tomadas

### 2026-06-03

- Nombre del proyecto: StockHub. Repos `stockhub-api` y `stockhub-web`.
- Coexistencia en EC2: nuevo proyecto en puerto publico 8080 del mismo EC2 3.93.170.171 que ya hospeda stocknova en :80.
- Directorio EC2: `/opt/stockhub` paralelo a `/opt/stocknova`.
- JWT: access 60 min + refresh 7 dias persistido en BD con rotacion.
- BCrypt cost 12, seeder en runtime con CommandLineRunner.
- CI/CD identico al patron de stocknova: build+test, docker push a ghcr.io, ssh deploy.
- SonarCloud como job opcional (solo si existe el secret SONAR_TOKEN).
- Mailpit en desarrollo, Gmail en produccion.
- Testcontainers PostgreSQL 16 para tests de integracion.
- Quasar v2 con TypeScript + Vite + Pinia + Vue Router + Axios + ESLint.
- Arquitectura hexagonal (domain, application, infrastructure, interfaces) alineada con repo previo del candidato.

### 2026-06-03 - Reglas de estilo

- Sin emojis ni caracteres unicode decorativos en codigo, documentos o commits.
- READMEs maximo 200 lineas, profesionales, sin "vibe IA".
- Comunicacion y documentos en espanol. Identificadores de codigo en ingles.

---

## Problemas y soluciones

Vacio por ahora.

---

## Bitacora cronologica

### 2026-06-03 - Planificacion

Realizado:
- Lectura del reto (`Docs/Reto Pratico.pdf`) y vacante (`Docs/Vacante.txt`).
- Auditoria de `stocknova-api` y `stocknova-web` para extraer patron CI/CD.
- Revision de `MrDavidAlv/ApiRest-Java-SpringBoot-PostgreSQL` para alinear arquitectura hexagonal.
- Reescritura de `PLAN.md` con CI/CD, refresh tokens, Actuator, headers OWASP, Dockerfile non-root, BCrypt seeder, docker-compose.prod, scripts deploy, lista de secrets, Testcontainers, SonarCloud opcional, reglas de estilo.
- Creacion de `progresso.md` y `CLAUDE.md`.

Bloqueantes: ninguno.

Siguiente:
- Confirmar fecha limite del reto.
- Etapa 0: crear repos `stockhub-api` y `stockhub-web`. Configurar secrets.

### 2026-06-03 - Scaffold local

Realizado:
- `stockhub-api/` generado con Spring Initializr (Spring Boot 3.5.14, Java 17). pom configurado con JJWT 0.12.6, iText 8.0.5, springdoc 2.6.0, MapStruct 1.6.3 (con lombok-mapstruct-binding) y spring-boot-starter-mail.
- Renombrado `StockhubApiApplication` a `StockHubApplication`.
- `application.properties` reemplazado por `application.yml` con perfiles y placeholders de env.
- `./mvnw compile` pasa.
- `stockhub-web/` generado con Quasar CLI 2.19 (Quasar v2 + Vite + TypeScript + SCSS + ESLint + Prettier).
- Rama git renombrada de `master` a `main` en stockhub-web.
- Instalados `axios` y `pinia` (no estaban en el wizard).
- `.env.example` con `VITE_API_URL`.
- `npm run build` pasa.

Bloqueantes: ninguno.

Siguiente:
- Crear repos `stockhub-api` y `stockhub-web` en GitHub.
- Configurar secrets en ambos repos.
- Etapa 1: Docker Compose dev + Flyway + seeds.

### 2026-06-03 - Etapa 1 completada

Realizado:
- `docker-compose.yml` con `postgres:16-alpine` (host:5433 por colision con Postgres nativo en :5432) y `axllent/mailpit` (UI :8025, SMTP :1025).
- Migracion `V1__create_schema.sql` con 11 tablas: usuarios, refresh_tokens, empresas, categorias, productos, precios_moneda, producto_categoria, clientes, ordenes, orden_producto. Constraints CHECK para enums (rol, moneda, estado) y FK con ON DELETE CASCADE donde corresponde.
- Migracion `V2__seed_catalogo.sql` con 3 categorias y 2 empresas de ejemplo.
- `application.yml` ajustado a puerto 5433.
- Arranque local: `docker compose up -d` levanta postgres y mailpit, luego `./mvnw spring-boot:run`. Flyway aplica V1 y V2 automaticamente. `/actuator/health` responde UP. Tablas y seeds verificados con psql.
- Repos GitHub creados por el usuario. Remotes `git@github.com:MrDavidAlv/stockhub-{api,web}.git` agregados localmente. No se ha hecho push aun.
- `.gitignore` de ambos repos extendido con `.env`, `.env.*`, `*.log`, OS files.

Decisiones:
- Puerto host de Postgres dev: 5433 (no 5432). El puerto interno del container sigue siendo 5432. La URL de prod no cambia porque el container se conecta por hostname `postgres` en red bridge.

Bloqueantes: ninguno.

Siguiente:
- Etapa 2: entidades JPA, JwtService, RefreshTokenService, SecurityConfig, DataSeeder, AuthController.

### 2026-06-03 - Etapa 2 completada

Realizado:
- Dominio: 3 enums (Rol, Moneda, EstadoOrden), 9 entidades JPA mas OrdenProductoId. Relaciones LAZY por defecto, @ManyToMany con tabla puente para Producto-Categoria, @EmbeddedId + @MapsId para OrdenProducto.
- Puertos: 7 interfaces JpaRepository en `domain/port` (Usuario, RefreshToken, Empresa, Producto, Categoria, Cliente, Orden).
- Excepciones de dominio: NotFoundException, DuplicateException, InvalidCredentialsException, InvalidTokenException.
- Infraestructura security: JwtService (HS384, subject=email, claim=rol, expira segun jwt.access-token-minutes), RefreshTokenService (UUID, rotacion, revocacion), UserDetailsServiceImpl, JwtAuthFilter (OncePerRequestFilter).
- SecurityConfig: STATELESS, BCryptPasswordEncoder(12), CORS por env, DaoAuthenticationProvider, @EnableMethodSecurity. Publicos: /api/auth/**, /actuator/health, /actuator/info, /swagger-ui/**, /v3/api-docs/**, GET /api/empresas/**.
- DataSeeder: CommandLineRunner que crea admin y externo solo si no existen, hasheando con BCrypt. @Profile("!test").
- Application: DTOs (LoginRequest, RefreshRequest, TokenPairResponse) como records con Bean Validation. AuthService + AuthServiceImpl (@Transactional para evitar LazyInitializationException al leer Usuario tras rotacion del refresh).
- Interfaces REST: AuthController con login/refresh/logout. GlobalExceptionHandler con respuestas consistentes (NotFound, InvalidCredentials, InvalidToken, AccessDenied, Duplicate, Validation, NoResourceFoundException -> 404, generico -> 500).
- Verificacion con curl:
  - POST /api/auth/login (admin y externo): 200 con tokens, rol, nombre, email.
  - POST /api/auth/refresh con token valido: 200 con tokens nuevos.
  - POST /api/auth/refresh con token ya rotado o revocado: 401 INVALID_TOKEN.
  - POST /api/auth/logout: 204; refresh posterior 401.
  - Login con password incorrecta: 401 INVALID_CREDENTIALS.
  - Login con email mal formado: 400 VALIDATION_ERROR con field errors.
  - GET /api/empresas sin controller: 404 NOT_FOUND.
  - Swagger UI: 200 con 3 endpoints documentados.

Problemas y soluciones:
- LazyInitializationException al rotar refresh token: el Usuario era proxy LAZY y la session se cerraba. Fix: @Transactional a nivel de clase en AuthServiceImpl.
- springdoc 2.6.0 incompatible con Spring Boot 3.5 (`NoSuchMethodError ControllerAdviceBean.<init>`). Fix: upgrade a 2.7.0.
- NoResourceFoundException caia en handler generico -> 500. Fix: handler dedicado -> 404.

Siguiente:
- Etapa 3: CRUD Empresa + Producto + Categoria con DTOs, MapStruct y @PreAuthorize.

### 2026-06-03 - Etapa 3 completada

Realizado:
- 8 DTOs como records: EmpresaRequest/Response, CategoriaRequest/Response, ProductoRequest/Response, PrecioMonedaRequest/Response. Bean Validation con @NotBlank, @Pattern, @Size, @NotNull, @PositiveOrZero, @Digits.
- 3 MapStruct mappers (EmpresaMapper, CategoriaMapper, ProductoMapper). ProductoMapper usa CategoriaMapper para anidar y mapea empresa.nit / empresa.nombre con @Mapping(source).
- 3 services + impl con @Transactional. ProductoServiceImpl maneja FK lookup (empresa, categorias), precios bidireccionales, codigo unico, cambio de empresa.
- 3 controllers REST. EmpresaController y CategoriaController abren GET para todos y exigen ADMIN en escritura. ProductoController usa @PreAuthorize a nivel de clase (ADMIN para todo).
- Verificacion end-to-end con curl:
  - GET /api/empresas publico (sin token devuelve datos).
  - POST/PUT/DELETE empresa con ADMIN: 201/200/204; con EXTERNO: 403; duplicado: 409; validacion fallida: 400 con field errors; no existe: 404.
  - Categoria CRUD con ADMIN. Nombre unico verificado.
  - Producto: creacion con precios COP+USD y 2 categorias (N:M); PUT actualiza categorias, cambia empresa, reemplaza precios; codigo duplicado 409; empresa o categoria inexistente 404; precio negativo 400 con field error `precios[0].precio`.
  - Swagger expone 19 endpoints (3 auth + 5 empresas + 5 categorias + 6 productos).

Problemas y soluciones:
- Ninguno relevante en esta etapa. Build limpio al primer intento.

Siguiente:
- Etapa 4: InventarioController, PdfAdapter con iText7, MailAdapter con JavaMailSender. Endpoint GET /pdf y POST /enviar-email.

### 2026-06-03 - Etapa 4 completada

Realizado:
- Puertos en dominio: PdfPort, MailPort.
- Adapters en infraestructura: PdfAdapter (iText 8.0.5) y MailAdapter (JavaMailSender + MimeMessageHelper con multipart UTF-8).
- DTO: EmailInventarioRequest con @Email + empresaNit opcional.
- InventarioService + Impl con `@Transactional(readOnly=true)`. Carga productos (todos o por empresa), valida existencia de empresa, delega en PdfPort y MailPort.
- InventarioController con `@PreAuthorize("hasRole('ADMIN')")` a nivel clase. Endpoints: GET /, GET /pdf (content-type application/pdf, content-disposition attachment), POST /enviar-email.
- GlobalExceptionHandler ampliado: MailDeliveryException -> 502, PdfGenerationException -> 500.
- Property nueva: `mail.from` (default `no-reply@stockhub.local`).
- Verificacion end-to-end:
  - GET /api/inventario (sin filtro y filtrado por empresa).
  - GET /api/inventario/pdf descarga 1788 bytes, PDF v1.7 valido (verificado con `file` y `pdfinfo`), iText 8.0.5 como Producer.
  - Texto del PDF (con `pdftotext`): titulo, subtitulo, 5 columnas (Codigo, Nombre, Empresa, Precio COP, Categorias), filas con datos correctos.
  - POST /api/inventario/enviar-email a Mailpit: subject "Inventario StockHub - 900123456-1", adjunto inventario.pdf de 1692 bytes verificado bajandolo de Mailpit y abriendolo con `file`.
  - Empresa inexistente -> 404. Email malformado -> 400 con field error. EXTERNO -> 403.

Problemas y soluciones:
- PDF descargaba 15 bytes (solo el header). Causa: dentro de try-with-resources, `baos.toByteArray()` se llamaba antes de cerrar Document/PdfDocument/PdfWriter, asi que el PDF no se finalizaba al stream. Fix: instanciar `baos` fuera del try-with-resources, y llamar `toByteArray()` despues del bloque (cuando ya se cerraron los writers).

Siguiente:
- Etapa 5: tests unitarios (EmpresaServiceImpl, ProductoServiceImpl, JwtService, PdfAdapter, RefreshTokenService) y tests de integracion con Testcontainers para AuthFlow y CRUDs.

### 2026-06-03 - Etapa 5 completada

Realizado:
- Tests unitarios (Mockito, sin Spring):
  - EmpresaServiceImplTest: 10 tests (findAll, findByNit ok/notfound, create ok/duplicate, update ok/duplicate/notfound, delete ok/notfound).
  - ProductoServiceImplTest: 6 tests (create ok/duplicate/empresa-missing/categoria-missing, delete notfound, findByEmpresa notfound).
  - JwtServiceTest: 4 tests (generate+extract roundtrip, malformed false, null false, signed-with-other-secret false).
  - RefreshTokenServiceTest: 5 tests (create con expiry futura, validate notfound/revoked/expired, rotate revoca antiguo y crea nuevo).
  - PdfAdapterTest: 2 tests con extraccion textual de iText (lista vacia genera PDF con mensaje, lista con producto contiene codigo, nombre, empresa, precio y categoria).
- Tests de integracion (Testcontainers PostgreSQL 16):
  - AbstractIntegrationTest: container estatico en bloque `static {}` para que sobreviva el JVM (no JUnit @Container) y compartirse entre clases gracias al cache de Spring TestContext.
  - AuthIntegrationTest: 4 tests con MockMvc (login admin OK, login externo, password mal 401, email mal formado 400 con fieldErrors).
  - EmpresaCrudIntegrationTest: 6 tests con @WithMockUser (publico GET, sin auth 403, ADMIN crea + persiste, EXTERNO 403, validacion 400, delete inexistente 404).
- DataSeeder ya corre en cualquier perfil para que los integration tests tengan usuarios listos.

Problemas y soluciones:
- 3 tests del CRUD fallaban con 500 + timeout de 30s. Causa: JUnit `@Testcontainers` + `@Container` static apaga el container al terminar la clase, pero Spring cachea el contexto y la siguiente clase reusa el viejo JDBC URL. Fix: arrancar el container en `static {}` del Abstract sin `@Container`, asi se mantiene vivo durante todo el run.

Siguiente:
- Etapa 6: Dockerfile backend multi-stage con usuario no-root y HEALTHCHECK contra /actuator/health.

### 2026-06-03 - Etapa 6 completada

Realizado:
- Dockerfile multi-stage: build con `eclipse-temurin:17-jdk-alpine`, runtime con `eclipse-temurin:17-jre-alpine`.
- Capa de cache de dependencias: `mvnw dependency:go-offline` antes de copiar el codigo, asi rebuilds sin cambios en pom no descargan deps.
- Imagen runtime: usuario `app` (uid 100, gid 101), no-root. `wget` agregado en la capa runtime para el healthcheck.
- HEALTHCHECK cada 30s contra `/actuator/health`, start-period 40s, 3 reintentos.
- JAVA_OPTS con `UseContainerSupport` y `MaxRAMPercentage=75.0`. SPRING_PROFILES_ACTIVE=prod por defecto.
- `.dockerignore` con target, .git, .idea, .vscode, .env, README, etc. para reducir el build context.
- Imagen final: 549MB.
- Verificacion: container corrido en la misma red de Docker que `stockhub-db`, login admin OK, healthcheck llega a `healthy`, usuario dentro del container `app(100):app(101)`.

Siguiente:
- Etapa 7: frontend Axios + Pinia + router guards.

### 2026-06-03 - Etapa 7 completada

Realizado:
- Boot file `src/boot/pinia.ts` que registra Pinia. Anadido `boot: ['pinia']` en `quasar.config.ts`.
- `src/types/index.ts` con interfaces TS compartidas: TokenPair, UserInfo, ApiError, Empresa/EmpresaPayload, Categoria, Producto/ProductoPayload, PrecioMoneda, EmailInventarioPayload, Rol, Moneda.
- `src/services/api.service.ts`: instancia Axios con baseURL desde `VITE_API_URL`, timeout 15s, interceptor request que inyecta `Authorization: Bearer`, interceptor response que en 401 hace refresh automatico una sola vez (con flag `_retry`), evitando reintentos sobre endpoints `/auth/` para no loopear.
- `src/services/auth.service.ts` no existe explicito porque el login/refresh/logout viven en el store (usan axios directo para no inyectar token a si mismos).
- `src/services/empresa.service.ts`, `categoria.service.ts`, `producto.service.ts`, `inventario.service.ts`: clientes tipados sobre `api`.
- `src/stores/auth.store.ts`: estado con accessToken/refreshToken/user persistido en localStorage (keys con prefijo `stockhub.*`); getters `isAuthenticated` e `isAdmin`; acciones `login`, `tryRefresh`, `logout` (notifica al backend best-effort), `applyTokens`, `clear`.
- `src/router/routes.ts`: rutas para `/login`, `/empresas`, `/productos` y `/inventario` con meta-flags `requiresAuth`, `requiresAdmin`, `guestOnly`. Las paginas reales se implementan en Etapa 8 (uso IndexPage placeholder).
- `src/router/index.ts`: `beforeEach` que redirige a login si requiere auth sin estar autenticado, a empresas si no es admin, y saca de login si ya esta autenticado.
- `npm run build` pasa.

Problemas y soluciones:
- ESLint regla `@typescript-eslint/consistent-type-imports` rompia el build por importar `AxiosError` como valor. Fix: cambiar a `type AxiosError` en el import.

Siguiente:
- Etapa 8: vistas Quasar (Login, Empresa, Producto, Inventario, MainLayout con navegacion condicional).

### 2026-06-03 - Etapa 8 completada

Realizado:
- Plugins Quasar habilitados en `quasar.config.ts`: Notify, Dialog, Loading.
- `src/utils/error.ts` con `getApiErrorMessage` y `getFieldErrors` para mapear ApiError del backend.
- `MainLayout.vue`: QHeader con tabs `Empresas`, `Productos`, `Inventario` que aparecen segun rol. Chip con rol del usuario y boton logout. Si no esta autenticado, boton de login.
- `LoginPage.vue`: QCard con QForm de email/password, rules locales, error-message conectado a fieldErrors del backend, credenciales de prueba visibles.
- `EmpresaPage.vue`: QTable con 5 columnas, boton "Nueva empresa" para ADMIN, dialog con `EmpresaForm`, confirmacion de borrado con `$q.dialog`.
- `EmpresaForm.vue`: NIT bloqueado en modo edicion, validaciones inline + propagacion de field errors del backend.
- `ProductoPage.vue`: filtro por empresa (clearable), tabla con chips para precios y categorias, dialog con `ProductoForm`.
- `ProductoForm.vue`: select de empresa y select multiple de categorias cargados en `onMounted`, sub-componente `PrecioMonedaList`, mapeo de producto existente a estado inicial.
- `PrecioMonedaList.vue`: componente v-model que mantiene una lista de PrecioMoneda, evita duplicar monedas y permite agregar/remover.
- `InventarioPage.vue`: tabla con chips, filtro por empresa, boton "Descargar PDF" que crea Blob URL para iniciar descarga, dialog para enviar por email con validacion de formato.
- `routes.ts` actualizado para usar las paginas reales.
- `.env` local con `VITE_API_URL=http://localhost:8080/api`.
- Smoke E2E con backend + `quasar dev`: SPA en 9000 sirve, preflight CORS OK, login OK desde origen 9000 contra 8080.

Problemas y soluciones:
- ESLint `@typescript-eslint/no-misused-promises` rompia el build por pasar callbacks async a `dialog.onOk()`. Fix: envolver la logica en `void (async () => { ... })()` dentro de un callback sincrono.

Siguiente:
- Etapa 9: Dockerfile + nginx.conf para servir el frontend en produccion.

### 2026-06-03 - Etapa 9 completada

Realizado:
- `.dockerignore` con node_modules, dist, .quasar, .env, .git, .vscode, cypress.
- `Dockerfile` multi-stage: stage build con `node:22-alpine` (Quasar v2.6 exige Node 22+), `npm ci --ignore-scripts` para evitar que `quasar prepare` falle antes de copiar el codigo, luego copia y `npm run build`. Stage runtime con `nginx:alpine` + nginx.conf custom. HEALTHCHECK con wget contra `/`.
- `nginx.conf` con SPA fallback `try_files`, headers OWASP completos (CSP, X-Frame-Options DENY, X-Content-Type-Options nosniff, Referrer-Policy strict-origin-when-cross-origin, Permissions-Policy restrictivo) replicados en las 3 locations porque nginx NO hereda `add_header` cuando la location define su propio `add_header`. Cache-Control diferenciado: `no-store` para index.html, `public, immutable, max-age=31536000` para assets versionados.
- Imagen final: 94.7 MB.
- Smoke test del container: GET / sirve index.html, ruta SPA inexistente devuelve el mismo index.html (fallback OK), los 5 headers OWASP llegan tanto en root como en assets, Cache-Control de 1 ano para `.css`/`.js`/`.woff2`.

Problemas y soluciones:
- `npm ci` fallaba porque el `postinstall = quasar prepare` necesita codigo fuente y aun no estaba copiado en esa capa. Fix: `npm ci --ignore-scripts`, despues `npm run build` (build no necesita prepare por separado).
- Node 20 rechazado por Quasar v2.6 (`requires Node 22.22.0 or superior`). Fix: subir base a `node:22-alpine`.
- Headers CSP/Referrer-Policy/Permissions-Policy no aparecian en GET / por la regla de nginx: un `add_header` en la location anula la herencia del server-level. Fix: repetir los 5 headers en cada location que tiene `add_header` propio.

Siguiente:
- Etapa 10: docker-compose.prod.yml + deploy/setup-ec2.sh + deploy/deploy.sh + deploy/nginx.conf reverse-proxy en :8080.

### 2026-06-03 - Etapa 10 completada

Realizado:
- `docker-compose.prod.yml`: 4 servicios (postgres, api, web, nginx). API y web solo `expose`, nginx publica `8080:80` para no chocar con stocknova. Red bridge `stockhub`. Volume `postgres_data` persistente. `depends_on` con `service_healthy` en postgres. Restart `unless-stopped`. Variables sensibles desde `.env`.
- `deploy/nginx.conf`: reverse-proxy con headers de seguridad (X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy). Rutas: `/` -> web, `/api/` -> api:8080, `/swagger-ui`, `/swagger-ui.html`, `/v3/api-docs` -> api, `/actuator/health` con `access_log off` para no contaminar logs. Forward de `Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`. `proxy_hide_header X-Powered-By`.
- `deploy/setup-ec2.sh`: idempotente, instala Docker solo si falta, crea `/opt/stockhub/deploy`, chown a ubuntu. Recordatorio explicito al usuario de abrir 8080 en el Security Group y de no tocar el :80 de stocknova.
- `deploy/deploy.sh`: pull, up -d, prune. Healthcheck loop de 6 intentos x 10s contra `http://localhost:8080/actuator/health`. Si falla, vuelca los ultimos 80 logs del api. Exit codes correctos.
- Smoke test del stack completo en local con imagenes `stockhub-api:dev` y `stockhub-web:dev`, mapeando el nginx en `18090:80`. Verificado: `/actuator/health` UP, frontend SPA sirve `<title>StockHub</title>`, login admin OK con rol ADMIN, GET /api/empresas publico devuelve 2 seeds, Swagger UI 200, /v3/api-docs con 13 endpoints, ruta SPA inexistente HTTP 200 (fallback), 4 headers OWASP llegan via reverse-proxy.

Problemas y soluciones:
- Healthcheck del actuator quedaba DOWN (503) porque `MailHealthIndicator` intentaba conectarse al SMTP y `mailpit` no existia en el test stack local. En produccion seria un timeout de Gmail un dia que tenga problemas y tumbaria el deploy. Fix definitivo: `management.health.mail.enabled=false` en application.yml. Mail sigue funcionando, pero su disponibilidad no condiciona el liveness de la API.

Siguiente:
- Etapa 11: GitHub Actions pipelines para api y web (build+test, docker push a ghcr.io, ssh deploy a EC2).

### 2026-06-03 - Etapa 11 completada

Realizado:
- `stockhub-api/.github/workflows/ci.yml` con 3 jobs:
  1. `build-and-test`: setup-java 17 temurin con cache maven, `./mvnw verify` (corre los 38 tests con Testcontainers, los runners de GH Actions tienen Docker). Sube `target/surefire-reports` como artifact si falla.
  2. `docker-push` (solo push a main, needs build-and-test): login a ghcr.io con `GITHUB_TOKEN`, build y push con tags `:latest` y `:${{ github.sha }}` para rollback.
  3. `deploy` (solo push a main, needs docker-push): scp de `docker-compose.prod.yml`, `deploy/nginx.conf`, `deploy/deploy.sh` a `/opt/stockhub`. SSH con heredoc para escribir `.env` desde secrets (chmod 600). SSH con `docker login` + `bash deploy/deploy.sh`.
- `stockhub-web/.github/workflows/ci.yml` con 3 jobs:
  1. `build`: setup-node 22 con cache npm, `npm ci` y `npm run build` (el build incluye lint via vite-plugin-checker).
  2. `docker-push`: build con `--build-arg VITE_API_URL=/api` para que el bundle apunte al reverse-proxy.
  3. `deploy`: SSH al EC2 que solo refresca el contenedor web (`pull web && up -d web`), porque el stack vive en /opt/stockhub que pertenece al repo backend. Healthcheck local con curl al `:8080/`.

Secrets de GitHub que hay que configurar en ambos repos (Settings -> Secrets and variables -> Actions):
- `EC2_HOST` (3.93.170.171), `EC2_SSH_KEY` (llave privada en formato OpenSSH), `CR_PAT` (PAT con scope `read:packages` para pull desde el EC2).
- Solo en stockhub-api: `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `JWT_SECRET`, `CORS_ORIGINS`, `SEED_ADMIN_PASSWORD`, `SEED_EXTERNO_PASSWORD`, `MAIL_USERNAME` (Gmail), `MAIL_PASSWORD` (App Password), `MAIL_FROM`.

Notas:
- YAML validados con PyYAML local.
- Imagenes finales: `ghcr.io/mrdavidalv/stockhub-api:{latest,sha}` y `ghcr.io/mrdavidalv/stockhub-web:{latest,sha}`. La organizacion del registry es lowercase por requisito de ghcr.io.
- Sin SonarCloud por ahora (se puede agregar como job extra).

Siguiente:
- Etapa 12: READMEs cortos (<=200 lineas) en ambos repos.

### 2026-06-03 - Etapa 12 completada

Realizado:
- `stockhub-api/README.md` con 152 lineas: titulo + descripcion de una linea, tabla de stack, arquitectura hexagonal en 4 capas, modelo de datos en una linea, correr en local (3 comandos), tabla de 14 variables de entorno, lista de endpoints con permisos, tests (`mvnw verify` = 38 tests), empaquetado y despliegue, credenciales de prueba, autor.
- `stockhub-web/README.md` con 100 lineas: stack, estructura de carpetas, correr local (3 comandos), env var unica, build, auth y roles, vistas, despliegue con headers OWASP, credenciales, autor.
- Verificado con `grep -P` que no hay caracteres unicode decorativos (flechas, emojis, banderas, etc).
- Tono factual: sin "potente", "robusto", "estado del arte", listas exhaustivas redundantes.

Siguiente:
- Etapa 13: validacion final con checklist del reto y entregables.

### 2026-06-03 - Etapa 13 completada

Realizado:
- `./mvnw verify`: 38 tests, 0 failures, 0 errors.
- `npm run build` (stockhub-web): build OK con lint y type-check.
- `docker build` ambas imagenes: stockhub-api 549 MB, stockhub-web 94.7 MB.
- Stack productivo levantado en local (mismo `docker-compose.prod.yml` apuntando a imagenes locales, nginx en 18090:80) y smoke E2E completo.

Pruebas E2E ejecutadas contra el stack completo:

| # | Caso | Resultado |
|---|---|---|
| 1 | GET / (SPA) | HTTP 200, title "StockHub" |
| 2 | GET /actuator/health | UP |
| 3 | Login admin y externo | Tokens devueltos correctamente |
| 4 | GET /api/empresas publico | 2 empresas (seeds) |
| 5 | EXTERNO crea empresa | 403 |
| 6 | ADMIN crea empresa | 201 |
| 7 | ADMIN crea producto multi-moneda + N:M categorias | 1 producto con 2 precios y 2 categorias |
| 8 | Descarga PDF inventario | 1685 bytes, PDF v1.7 valido, contenido textual verificado con pdftotext |
| 9 | Swagger UI | 200 |
| 10 | /v3/api-docs | 13 endpoints documentados |
| 11 | Ruta SPA inexistente | 200 (fallback) |
| 12 | Headers OWASP via reverse-proxy | 4/4 presentes |

Estado del codigo:
- 78 archivos Java en stockhub-api siguiendo arquitectura hexagonal.
- 13 archivos .vue + 13 archivos .ts en stockhub-web.
- Remotes apuntando a `github.com/MrDavidAlv/stockhub-{api,web}` en rama `main`.
- Stocknova ni siquiera arranca en local, no fue tocado.

## Checklist final contra el reto

### Funcionalidad
- [x] Vista Empresa con formulario (NIT, nombre, direccion, telefono) y CRUD segun rol.
- [x] Vista Productos con formulario (codigo, nombre, caracteristicas, precio multi-moneda, empresa, categorias).
- [x] Vista Inicio de Sesion con formulario (correo y contrasena).
- [x] Vista Inventario con descarga PDF y envio por email via REST.
- [x] Usuario ADMIN: registra, edita, elimina Empresa; registra Productos por empresa; gestiona inventario.
- [x] Usuario EXTERNO: ve las empresas como visitante.
- [x] Modelo ER con Empresa, Productos, Categorias, Clientes, Ordenes.
- [x] Producto N:M Categoria (tabla puente `producto_categoria`).
- [x] Cliente 1:N Ordenes (`ordenes.cliente_id`).
- [x] Orden N:M Producto (`orden_producto` con cantidad y precio).
- [x] Password encriptada con BCrypt cost 12 para autenticacion.

### Calidad
- [x] Arquitectura hexagonal (domain, application, infrastructure, interfaces).
- [x] SOLID y patrones (Repository, DTO + MapStruct, Adapter para PdfPort/MailPort).
- [x] 38 tests (27 unit + 11 integration Testcontainers). Meta >=15 superada.
- [x] GlobalExceptionHandler con codigos consistentes (NOT_FOUND, VALIDATION_ERROR, INVALID_CREDENTIALS, INVALID_TOKEN, FORBIDDEN, CONFLICT, MAIL_ERROR, PDF_ERROR, INTERNAL_ERROR).
- [x] Bean Validation en DTOs backend y rules en formularios Quasar.
- [x] Swagger UI documentando 13 endpoints.

### DevOps
- [x] Dockerfile backend multi-stage, usuario `app` no-root (uid 100), HEALTHCHECK actuator.
- [x] Dockerfile frontend multi-stage, nginx con headers OWASP completos y SPA fallback.
- [x] `docker compose up` levanta postgres + mailpit en local.
- [x] `docker-compose.prod.yml` con 4 servicios (postgres, api, web, nginx) y reverse-proxy en :8080.
- [x] CI/CD GitHub Actions en ambos repos (build/test, docker push a ghcr.io, ssh deploy a EC2).
- [x] Scripts `setup-ec2.sh` (idempotente) y `deploy.sh` (healthcheck loop) en deploy/.
- [x] Convivencia con stocknova en el mismo EC2: stocknova en :80, StockHub en :8080.

### Entregables documentados
- [x] README de stockhub-api (152 lineas).
- [x] README de stockhub-web (100 lineas).
- [x] PLAN.md (488 lineas) con arquitectura, etapas, secrets y checklist.
- [x] progresso.md como bitacora del trabajo.
- [x] CLAUDE.md como contexto para futuras sesiones.

## Acciones manuales pendientes del usuario

Las decisiones tecnicas estan hechas y todo corre en local. Lo que queda son acciones que requieren tus credenciales y acceso a servicios externos:

1. Configurar los 13 secrets en GitHub (ambos repos): `EC2_HOST`, `EC2_SSH_KEY`, `CR_PAT`, y en stockhub-api ademas `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `JWT_SECRET`, `CORS_ORIGINS`, `SEED_ADMIN_PASSWORD`, `SEED_EXTERNO_PASSWORD`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_FROM`. Comandos para generar:
   ```bash
   openssl rand -base64 48   # JWT_SECRET
   openssl rand -base64 24   # POSTGRES_PASSWORD
   openssl rand -base64 16   # SEED_*_PASSWORD
   ```
2. Crear App Password en https://myaccount.google.com/apppasswords para `MAIL_PASSWORD`.
3. Crear PAT de GitHub con scope `read:packages` para `CR_PAT`.
4. Abrir el puerto `8080/TCP` en el Security Group del EC2 `3.93.170.171`.
5. Conectarse al EC2 y ejecutar `sudo bash deploy/setup-ec2.sh` (idempotente, no toca Docker existente).
6. Hacer commit y push de ambos repos a `main` para disparar los pipelines.
7. Actualizar el README de cada repo si la URL final cambia (o anadir el link a Swagger UI publico).
8. Responder el email del reto con: link a `stockhub-api`, link a `stockhub-web`, URL publica `http://3.93.170.171:8080`, credenciales ADMIN y EXTERNO.

Estado: codigo y planificacion en verde. Falta unicamente la accion manual del usuario para llevarlo a produccion y enviar los entregables.

### 2026-06-04 - Etapa 14 completada

Realizado:
- Eliminados defaults sensibles del codigo:
  - `application.yml`: `SPRING_DATASOURCE_PASSWORD`, `JWT_SECRET`, `SEED_ADMIN_PASSWORD`, `SEED_EXTERNO_PASSWORD` ya no tienen default. Spring falla con un mensaje claro si faltan.
  - `docker-compose.yml` dev: `POSTGRES_PASSWORD` viene de `${SPRING_DATASOURCE_PASSWORD:?...}` con mensaje de error explicito. `POSTGRES_DB` y `POSTGRES_USER` con default sensato.
- Mecanismo `.env` para dev: anadido `spring.config.import: optional:file:.env[.properties]` a `application.yml`. Tanto Spring Boot como Docker Compose leen el mismo `.env` desde la raiz de `stockhub-api`.
- Creado `stockhub-api/.env.example` con plantilla de 12 vars y `stockhub-api/.env` local (gitignored) para que sigan corriendo `./mvnw spring-boot:run` y `docker compose up`.
- Tests: `AbstractIntegrationTest` ahora setea `jwt.secret`, `seed.admin-password` y `seed.externo-password` via `@DynamicPropertySource`, independiente del .env del workspace.
- Verificado:
  - `./mvnw verify` -> 38/38 OK con los nuevos defaults removidos.
  - Sin hardcodes restantes en `src/main` (`grep -E "admin123|externo123|stockhub123|dev-secret"` vacio).
  - Imagen Docker reconstruida.

Documentos creados en `/home/mrdavidalv/Documents/Lite Thinking/Docs/` (fuera de ambos repos git):
- `SECRETS-MANUAL.md` (14 secciones, ~200 lineas): paso a paso para generar y configurar los 13 secrets, App Password de Gmail, PAT de GitHub, abrir 8080 en Security Group, provisionar EC2, disparar deploy, verificar produccion, comandos utiles, rotacion.
- `SECRETS-VALUES.md` (template personal, no committeado): tablas para que el usuario llene con los valores reales generados con `openssl rand -base64`. Incluye seccion para el correo de entrega al evaluador.
- `DEPLOY-EC2-MANUAL.md`: manual operativo, convivencia con stocknova, comandos utiles, troubleshooting, rollback con tags `:${sha}`.
- `CHECKLIST-RETO.md`: cumplimiento uno a uno de los puntos a-g del reto, entregables, buenas practicas, mapeo de skills de la vacante, gaps conscientes.

Nota: el archivo `stocknova-key.pem` ubicado en `/home/mrdavidalv/Documents/docs/Docs/` es la misma SSH key que se reutiliza para el EC2 de StockHub (es el mismo EC2 de stocknova). El path queda referenciado en SECRETS-MANUAL.md y DEPLOY-EC2-MANUAL.md.

---

### 2026-06-04 - Etapa 15: auditoria Lighthouse

Lighthouse sobre `http://3.93.170.171:8080/#/empresas`. Desktop: Perf 98, A11y 90, BP 78, SEO 91. Mobile: Perf 68, A11y 90, BP 78, SEO 91.

Arreglos aplicados en codigo (pendiente redeploy):
- A11y - botones sin nombre accesible: `aria-label` en edit/delete de EmpresaPage y ProductoPage, y en delete de PrecioMonedaList (5 botones). Logout ya tenia aria-label.
- A11y - viewport: `index.html` ya no usa `user-scalable=no, maximum-scale=1, minimum-scale=1`. Queda `initial-scale=1, width=device-width` para permitir zoom.
- SEO - `public/robots.txt` valido (User-agent / Allow). Antes `/robots.txt` caia al fallback SPA y devolvia HTML (7 errores).
- BP/perf - `nginx.conf` (web) e `deploy/nginx.conf` (reverse-proxy): header `Cross-Origin-Opener-Policy: same-origin`.
- Perf/bfcache - `index.html` location pasa de `Cache-Control: no-store` a `no-cache` (no-store bloqueaba el back/forward cache).
- `npm run build` OK. Verificado robots.txt y viewport en dist/spa.

Segunda tanda (todo lo no-HTTPS):
- A11y/SEO - `index.html` con `<html lang="es">` y `framework.lang: 'es'` en quasar.config (textos por defecto de Quasar en espanol).
- Perf - removido el extra `roboto-font` de quasar.config. Elimina 3 peticiones woff bloqueantes sin font-display. El texto usa el stack de fuentes del sistema (fallback de Quasar). Iconos `material-icons` se mantienen. Revertible con una linea.
- Build verificado: dist sin woff de Roboto (solo queda la de material-icons), `<html lang=es>`, viewport `initial-scale=1,width=device-width`.

Esperado tras redeploy: Accessibility 100, SEO 100, ligera mejora de FCP/LCP en mobile.

Best Practices 100 NO se persigue: lo decidio el usuario (no HTTPS). Queda en ~88. El item HTTPS (y HTTP/2 de "Modern HTTP") exige TLS con certificado valido, imposible en IP pelada sin dominio. Documentado como mejora conocida.

Tercera tanda (perf mobile):
- Diagnostico con stack de evaluacion local (docker-compose.eval.yml, nginx en :8090): el nginx del frontend servia el CSS de Quasar sin comprimir (197 KB) y el JS sin comprimir (108 KB). Sobre Slow 4G eso causaba el render-blocking de ~1760 ms (FCP 4.6s / LCP 5.3s, mobile score 69).
- Fix: `gzip on` en `stockhub-web/nginx.conf` (gzip_types css/js/json/svg/xml, comp_level 6). Verificado: CSS 197 KB -> 34.7 KB transferidos, JS comprime. El reverse-proxy del EC2 propaga el Content-Encoding del upstream, no requiere cambio.

Resultados Lighthouse en localhost:8090 (progresion):
- Mobile antes de gzip: Perf 69, A11y 100, BP 100, SEO 100.
- Mobile despues de gzip: Perf 88 (FCP 2.8s, LCP 3.3s, TBT 0, CLS 0.024), A11y 100, BP 100, SEO 100.
- Desktop: ~100 en las cuatro.

Best Practices 100 es artefacto de localhost (contexto seguro); en el EC2 sera ~88 por servirse en HTTP.

Decision del usuario: cerrar el mobile en 88. El resto del techo (render-blocking del CSS base de Quasar y la fuente material-icons de 126 KB) requiere HTTP/2 ("Modern HTTP") para mejorar de forma limpia, y HTTP/2 necesita HTTPS, que se descarto. Se evaluo cambiar iconos a SVG (quitaria 126 KB) pero el gano estimado (~90-93) no justificaba el riesgo de regresion visual no verificable. bfcache restante: "Internal error / Not actionable" (quirk de Chrome, no es codigo nuestro).

Cuarta tanda (fix en produccion): tras desplegar, Lighthouse desktop en el EC2 dio Best Practices 74 por el audit scoreado "Browser errors were logged to the console". Causa: el header `Cross-Origin-Opener-Policy` que habiamos agregado. Sobre HTTP (origen no confiable) el navegador lo ignora Y escribe un error en consola. COOP solo aplica en HTTPS y su audit no puntua, asi que en HTTP solo restaba. Fix: removido COOP de `stockhub-web/nginx.conf` (3 sitios) y `stockhub-api/deploy/nginx.conf`. Verificado en eval: el response ya no trae COOP, gzip intacto. Esperado: Best Practices 74 -> ~85 (el resto del techo sigue siendo HTTPS).

Push hecho a main en ambos repos (web 5c2b4dc, api c662265) el 2026-06-04. Deploy confirmado en vivo: header COOP eliminado (count 0), gzip activo, health UP. El error de consola desaparecio.

Numeros finales Lighthouse en http://3.93.170.171:8080 (desktop):
- Performance 94 (FCP 0.9s, LCP 1.2s, TBT 0, CLS 0.021)
- Accessibility 100
- Best Practices 78
- SEO 100

Best Practices 78 es el techo real en el EC2: los items restantes son todos de HTTPS (no usa HTTPS, no redirect, HSTS, Trusted Types) mas "Modern HTTP" (HTTP/2) en Performance. Todo eso requiere TLS con certificado valido sobre un dominio, descartado por decision del usuario. Etapa cerrada.

### 2026-06-04 - Etapa 16: manejo de errores en la capa de seguridad

Contexto: el evaluador del reto pregunto por el uso de try/catch. La API ya tenia 4 try/catch puntuales (AuthServiceImpl, MailAdapter, PdfAdapter, JwtService) mas el GlobalExceptionHandler centralizado (@RestControllerAdvice). Se decidio NO esparcir try/catch por services/controllers (anti-patron) y en cambio tapar tres huecos reales:

- C (bug real de authz): JwtAuthFilter construia el Authentication sin verificar isEnabled(). Un usuario con activo=false y token vigente seguia autenticado hasta expirar el token. Fix: chequeo `if (!ud.isEnabled()) return;` antes de autenticar. El DaoAuthenticationProvider solo valida disabled en el login, no en el flujo stateless.
- B: JwtAuthFilter corre antes del ExceptionTranslationFilter, asi que una excepcion ahi (ej. UsernameNotFoundException de un usuario borrado con token vigente) se escapaba del GlobalExceptionHandler y daba 500. Fix: try/catch que limpia el contexto y deja seguir la cadena sin autenticar (lo maneja el entry point como 401).
- A: no habia AuthenticationEntryPoint ni AccessDeniedHandler, asi que los 401 de seguridad salian con el JSON por defecto de Spring, inconsistente con ErrorResponse. Fix: JwtAuthEntryPoint (401) y RestAccessDeniedHandler (403) que serializan ErrorResponse con el ObjectMapper de Spring, cableados en SecurityConfig via exceptionHandling.

Archivos: JwtAuthFilter (mod), JwtAuthEntryPoint (nuevo), RestAccessDeniedHandler (nuevo), SecurityConfig (mod).
Tests: JwtAuthFilterTest (4 nuevos: enabled autentica, disabled no autentica, user-not-found limpia contexto sin romper, sin header no autentica). EmpresaCrudIntegrationTest: el caso sin-auth paso de esperar 403 a 401 con shape ErrorResponse. Total 38 -> 42 tests, todos verdes con ./mvnw verify.

Nota de diseno: NO se agregaron try/catch a services/controllers. El manejo de negocio sigue centralizado en GlobalExceptionHandler; los try/catch viven solo en los bordes (integraciones externas y el filtro de seguridad).

Pendiente: commit + push a main de stockhub-api para redeploy.

## Pendientes operativos

- [ ] Generar `JWT_SECRET` con `openssl rand -base64 48`
- [ ] Generar `POSTGRES_PASSWORD` con `openssl rand -base64 24`
- [ ] Generar `SEED_ADMIN_PASSWORD` y `SEED_EXTERNO_PASSWORD`
- [ ] Crear App Password de Gmail en https://myaccount.google.com/apppasswords
- [ ] Crear PAT de GitHub con scope `read:packages` para `CR_PAT`
- [ ] Subir los 13 secrets a ambos repos
- [ ] Abrir puerto 8080 en el Security Group del EC2
- [ ] Confirmar que la llave SSH usada por stocknova se reutiliza

---

## Metricas de calidad

| Metrica | Objetivo | Real |
|---|---|---|
| Tests backend | >= 15 | - |
| Cobertura service + security | >= 70% | - |
| Tests frontend | Smoke + 1 por vista | - |
| Endpoints en Swagger | 100% | - |
| Pipeline CI/CD verde | Si | - |
| Healthcheck EC2 200 OK | Si | - |
| README backend <= 200 lineas | Si | - |
| README frontend <= 200 lineas | Si | - |
