# Notas rapidas para entrevista - StockHub

Conceptos y herramientas usadas en la prueba, con una explicacion corta y como se aplicaron en StockHub. Pensado para repasar antes de la entrevista y responder con seguridad.

---

## 1. Vision general del proyecto

- App full stack de gestion de inventario: empresas, productos (precio multi-moneda), categorias, clientes, ordenes e inventario con exportacion a PDF y envio por correo.
- Dos roles: ADMIN (control total) y EXTERNO (solo lectura de empresas).
- Backend Spring Boot, frontend Vue 3 + Quasar, base PostgreSQL, todo en Docker y desplegado en AWS EC2 con CI/CD.

Frase de una linea: "Es una app de inventario con autenticacion por JWT y roles, backend en Spring Boot con arquitectura hexagonal y frontend en Vue 3 con Quasar, desplegada en EC2 con GitHub Actions."

---

## 2. Frontend

### Vue 3
Framework de JavaScript para construir interfaces. Usa Composition API (logica organizada en funciones `setup`/`ref`/`computed` en vez de Options API). Reactividad: cuando cambia un dato, la vista se actualiza sola.

### Quasar v2
Framework UI sobre Vue. Trae componentes listos (tablas, formularios, dialogos, botones) y herramientas de build. En StockHub se uso para QTable, QForm, QInput, QDialog, QBtn, notificaciones (Notify) y loaders. Permite un solo codigo base para web, SPA, PWA, mobile y desktop.

### TypeScript
JavaScript con tipos estaticos. Detecta errores en tiempo de compilacion. En el front se definieron interfaces compartidas (Empresa, Producto, TokenPair, etc.) en `src/types`.

### Vite
Empaquetador y dev server rapido (usa ES modules, hot reload). Es el build tool por defecto de Quasar v2.

### Pinia
Libreria de estado global para Vue (sucesor de Vuex). En StockHub el `auth.store` guarda accessToken, refreshToken y usuario, persistidos en localStorage; expone getters `isAuthenticated` e `isAdmin`.

### Vue Router
Enrutador SPA. Se uso `beforeEach` (navigation guard) para proteger rutas: si la ruta requiere auth y no hay sesion redirige a login; si requiere ADMIN y el usuario es EXTERNO lo manda a empresas.

### Axios
Cliente HTTP. Se configuro con interceptores:
- Request: inyecta el header `Authorization: Bearer <token>`.
- Response: ante un 401 intenta refrescar el token una sola vez (flag `_retry`) y reintenta la peticion.

### Arquitectura del front
Separacion por responsabilidad:
- `pages/`: vistas (LoginPage, EmpresaPage, ProductoPage, InventarioPage).
- `components/`: piezas reutilizables (EmpresaForm, ProductoForm, PrecioMonedaList).
- `stores/`: estado con Pinia.
- `services/`: clientes HTTP tipados por recurso (empresa, producto, inventario...).
- `router/`: rutas y guards por rol.
- `layouts/`: estructura comun (MainLayout con tabs segun rol).
- `types/`: interfaces TypeScript compartidas.

Es una SPA: el navegador carga la app una vez y la navegacion ocurre en el cliente; las rutas inexistentes caen al `index.html` (SPA fallback en nginx).

---

## 3. Backend

### Java 17
Version LTS. Se aprovechan records (DTOs inmutables), var, switch moderno.

### Spring Boot 3.5
Framework para servicios web en Java. Auto-configuracion, servidor embebido (Tomcat), inyeccion de dependencias. Reduce el boilerplate de Spring clasico.

### Arquitectura hexagonal (puertos y adaptadores)
Separa el nucleo de negocio de la infraestructura. Capas:
- `domain`: modelos, puertos (interfaces) y excepciones. No depende de frameworks.
- `application`: casos de uso (services), DTOs, mappers.
- `infrastructure`: adaptadores concretos (JPA, seguridad, PDF, mail, config).
- `interfaces`: controladores REST y manejo de errores.

Idea clave: el dominio define interfaces (puertos como `PdfPort`, `MailPort`) y la infraestructura las implementa (adaptadores). Permite cambiar la tecnologia sin tocar el negocio y testear con mocks.

### Spring Data JPA / Hibernate
JPA es la especificacion de persistencia; Hibernate es la implementacion (ORM). Mapea clases Java a tablas. Spring Data JPA genera los repositorios a partir de interfaces (`JpaRepository`). Relaciones LAZY por defecto para no cargar de mas.

### PostgreSQL 16
Base de datos relacional. Modelo con N:M (Producto-Categoria, Orden-Producto vias tablas puente) y 1:N (Cliente-Ordenes).

### Flyway
Migraciones de base de datos versionadas (`V1__create_schema.sql`, `V2__seed_catalogo.sql`). Garantiza que el esquema evolucione de forma controlada y reproducible. Las contrasenas NO se siembran por SQL (se hashean en runtime).

### Spring Security
Autenticacion y autorizacion. Configurado STATELESS (sin sesion de servidor). Autorizacion a nivel de metodo con `@PreAuthorize("hasRole('ADMIN')")` y `@EnableMethodSecurity`.

### JWT (JSON Web Token)
Token firmado que lleva la identidad y el rol del usuario. En StockHub:
- Access token: 60 min, viaja en cada peticion.
- Refresh token: 7 dias, persistido en BD, con rotacion (al refrescar se revoca el viejo y se emite uno nuevo). Esto permite revocar sesiones, algo que un JWT puro no permite.
Flujo: login devuelve ambos tokens; el front usa el access; cuando expira, usa el refresh para obtener uno nuevo.

### BCrypt
Algoritmo de hashing de contrasenas con salt y costo configurable (cost 12). Lento a proposito para resistir fuerza bruta. Nunca se guarda la contrasena en texto plano.

### MapStruct
Generador de mappers entre entidades y DTOs en tiempo de compilacion (sin reflexion, mas rapido). Evita escribir conversiones a mano.

### DTO (Data Transfer Object)
Objeto que viaja entre capas/cliente y separa el modelo interno de la API publica. En el front no se exponen las entidades JPA directamente.

### Bean Validation
Validacion declarativa con anotaciones (`@NotBlank`, `@Email`, `@Pattern`, `@PositiveOrZero`). Los errores se devuelven como un mapa campo-mensaje (400 VALIDATION_ERROR).

### GlobalExceptionHandler
Manejador central de errores (`@RestControllerAdvice`) que traduce excepciones a respuestas HTTP consistentes (404, 400, 401, 403, 409, 500) sin filtrar el stack trace.

### iText 8
Libreria para generar PDFs. Se uso para exportar el inventario en una tabla (Codigo, Nombre, Empresa, Precio COP, Categorias).

### JavaMailSender
API de Spring para enviar correos. Se usa con `MimeMessageHelper` para adjuntar el PDF. En desarrollo apunta a Mailpit (captura local); en produccion a Gmail con App Password.

### springdoc-openapi (Swagger UI)
Genera documentacion interactiva de la API a partir del codigo. Disponible en `/swagger-ui/index.html`.

### Spring Boot Actuator
Endpoints de observabilidad: `/actuator/health` (estado de la app), usado tambien por el HEALTHCHECK de Docker y el reverse-proxy.

---

## 4. Pruebas

### JUnit 5
Framework de testing en Java. Tests unitarios y de integracion.

### Mockito
Libreria de mocks: simula dependencias para probar un service en aislamiento, sin levantar Spring ni BD.

### Testcontainers
Levanta una PostgreSQL real en un contenedor Docker durante los tests de integracion, en vez de usar una BD en memoria. Pruebas mas fieles a produccion.

Resumen StockHub: 38 tests (unit con Mockito + integration con Testcontainers y MockMvc).

---

## 5. DevOps y despliegue

### Docker
Empaqueta la app y sus dependencias en imagenes reproducibles. Se uso multi-stage build (una etapa compila, otra solo ejecuta) para imagenes pequenas, usuario no-root y HEALTHCHECK.

### Docker Compose
Orquesta varios contenedores. En produccion: postgres, api, web y nginx en una red bridge. En dev: postgres + Mailpit.

### Nginx
Servidor web y reverse-proxy. En produccion enruta `/` al frontend y `/api`, `/swagger`, `/actuator/health` al backend. Tambien aplica headers de seguridad OWASP y el SPA fallback.

### GitHub Actions
CI/CD por repositorio. Tres jobs: build-and-test, docker-push (a ghcr.io) y deploy (SSH al EC2). Se dispara en cada push a `main`.

### ghcr.io (GitHub Container Registry)
Registro donde se publican las imagenes Docker (`ghcr.io/mrdavidalv/stockhub-api` y `-web`).

### AWS EC2
Maquina virtual en la nube donde corre el stack. StockHub vive en el puerto 8080; convive con otro proyecto en el 80 sin interferir.

---

## 6. Conceptos transversales

### SOLID
Cinco principios de diseno orientado a objetos:
- S: responsabilidad unica (cada clase una razon para cambiar).
- O: abierto/cerrado (extender sin modificar).
- L: sustitucion de Liskov (un subtipo debe poder reemplazar al tipo base).
- I: segregacion de interfaces (interfaces pequenas y especificas).
- D: inversion de dependencias (depender de abstracciones, no de implementaciones). Es justo lo que habilita la arquitectura hexagonal con sus puertos.

### Clean Architecture / Arquitectura por capas
El negocio en el centro, sin depender de detalles externos (BD, framework, UI). Las dependencias apuntan hacia adentro.

### REST
Estilo de API sobre HTTP: recursos con URLs, verbos (GET, POST, PUT, DELETE) y codigos de estado. Sin estado entre peticiones.

### CORS
Politica del navegador que controla que origenes pueden llamar a la API. Se configura con la variable `CORS_ORIGINS`.

### Headers de seguridad OWASP
Cabeceras HTTP que mitigan ataques comunes: CSP, X-Frame-Options (clickjacking), X-Content-Type-Options (MIME sniffing), Referrer-Policy, Permissions-Policy. Aplicadas en nginx.

### Variables de entorno y secrets
La configuracion sensible (contrasenas, JWT secret, credenciales de correo) no va en el codigo: se inyecta por variables de entorno y se guarda como secrets en GitHub.

---

## 7. Posibles preguntas y respuesta corta

- Por que hexagonal? Para aislar el negocio de la infraestructura y poder testear y cambiar tecnologias sin reescribir la logica.
- Por que JWT y no sesiones? La API es STATELESS y escala mejor sin estado de sesion en el servidor; el refresh token en BD recupera la capacidad de revocar.
- Por que refresh token en BD? Un JWT firmado no se puede invalidar antes de que expire; guardarlo permite revocarlo y rotarlo.
- Por que BCrypt y no SHA? BCrypt es lento y con salt por diseno, pensado para contrasenas; SHA es rapido y vulnerable a fuerza bruta.
- Por que Pinia y no props? El estado de autenticacion lo necesitan varias vistas; centralizarlo evita pasar props en cascada.
- Como se evita exponer entidades? Con DTOs y mappers (MapStruct) entre capas.
- Como se prueban los servicios? Unitarios con Mockito (aislados) e integracion con Testcontainers (PostgreSQL real).
- Como llega el codigo a produccion? Push a main, GitHub Actions compila, testea, publica la imagen en ghcr.io y hace deploy por SSH al EC2.

---

## 8. Guion de demo en vivo

URL: http://3.93.170.171:8080 | Swagger: http://3.93.170.171:8080/swagger-ui/index.html
Credenciales: ADMIN admin@stockhub.local / EXTERNO externo@stockhub.local

Orden sugerido (8 a 10 minutos). La idea es mostrar valor primero y detalle tecnico despues.

1. Contexto (30 s)
   - "Es una app de inventario full stack: backend Spring Boot hexagonal, frontend Vue 3 + Quasar, PostgreSQL, todo dockerizado y desplegado en EC2 con CI/CD."

2. Login como EXTERNO (1 min)
   - Mostrar la vista de login y entrar con el usuario externo.
   - Senalar que solo aparece la pestana Empresas (las tabs se muestran segun rol).
   - Intentar (o explicar) que no puede crear ni editar: el boton de acciones no aparece. Refuerza el control de acceso por rol en el front, respaldado por el back.

3. Logout y login como ADMIN (1 min)
   - Ahora aparecen Empresas, Productos e Inventario.
   - Mencionar que el rol viene dentro del JWT y el front lo lee del store (Pinia).

4. CRUD de Empresa (1.5 min)
   - Crear una empresa nueva (mostrar validaciones: NIT requerido, formato).
   - Editar una. Senalar que el NIT no se puede cambiar (regla de negocio).
   - Provocar un error de validacion a proposito (campo vacio) para mostrar que el mensaje viene del backend mapeado al campo.

5. Producto multi-moneda y categorias (2 min)
   - Crear un producto: codigo, nombre, seleccionar empresa, varias categorias (N:M), y agregar precios en COP y USD (sub-formulario).
   - Es el punto mas rico del modelo: relacion N:M con categorias y lista de precios por moneda.

6. Inventario: PDF y correo (1.5 min)
   - Descargar el PDF del inventario (se genera con iText en el backend).
   - Enviar el inventario por correo a una direccion (en produccion sale por Gmail).

7. Cierre tecnico (1.5 min)
   - Abrir Swagger UI y mostrar los endpoints documentados.
   - Mostrar /actuator/health respondiendo UP.
   - Mencionar: "El mismo push a main dispara los tests, publica la imagen en ghcr.io y la despliega por SSH al EC2."

Plan B si falla la red o el EC2:
- Tener los repos abiertos en GitHub para recorrer la estructura hexagonal y los archivos clave (ver documento de referencias).
- Levantar local con docker compose si hay tiempo, o mostrar capturas.

Tips:
- No te disculpes por la simpleza: di que el alcance se mantuvo acotado al reto a proposito y que la arquitectura permite crecer.
- Si preguntan por algo que no hiciste (ej. paginacion, cache), reconoce el gap y explica donde encajaria.
