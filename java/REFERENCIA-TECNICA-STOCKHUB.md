# Referencia tecnica de StockHub

Documento de estudio en profundidad. Explica que es cada concepto, donde esta en el codigo y como se implemento, con enlace directo al archivo en GitHub. Pensado para sustentar decisiones en una entrevista tecnica.

Repositorios:
- Backend: https://github.com/MrDavidAlv/stockhub-api
- Frontend: https://github.com/MrDavidAlv/stockhub-web

Convencion de enlaces: todos apuntan a la rama `main`. Si en la entrevista comparten pantalla, abre el archivo y busca el metodo por nombre.

Indice:
1. Arquitectura general
2. Inyeccion de dependencias
3. Patrones de diseno
4. Spring Data JPA / Hibernate
5. Carga LAZY y N+1
6. Flyway
7. Spring Security
8. JWT y refresh tokens
9. BCrypt y el seeder
10. MapStruct y DTOs
11. Manejo de errores
12. CORS y headers de seguridad
13. Pinia (estado en el front)
14. Axios e interceptores
15. Vue Router y guards
16. PDF y correo (adaptadores)
17. Pruebas
18. Resumen de archivos clave

---

## 1. Arquitectura general

El backend usa arquitectura hexagonal (puertos y adaptadores), una forma de Clean Architecture. La regla central: las dependencias apuntan hacia el dominio; el dominio no conoce a Spring, JPA, iText ni nada externo.

Cuatro capas (paquetes bajo `com.stockhub`):

- domain: el nucleo. Modelos de negocio, puertos (interfaces) y excepciones. Sin anotaciones de framework salvo JPA en las entidades.
  - Modelos: https://github.com/MrDavidAlv/stockhub-api/tree/main/src/main/java/com/stockhub/domain/model
  - Puertos: https://github.com/MrDavidAlv/stockhub-api/tree/main/src/main/java/com/stockhub/domain/port
  - Excepciones: https://github.com/MrDavidAlv/stockhub-api/tree/main/src/main/java/com/stockhub/domain/exception
- application: casos de uso. Interfaces de servicio, sus implementaciones, DTOs y mappers.
  - https://github.com/MrDavidAlv/stockhub-api/tree/main/src/main/java/com/stockhub/application
- infrastructure: adaptadores concretos. Seguridad, persistencia, generacion de PDF, envio de correo, configuracion y seeder.
  - https://github.com/MrDavidAlv/stockhub-api/tree/main/src/main/java/com/stockhub/infrastructure
- interfaces: la capa que expone la app al mundo. Controladores REST y manejo global de errores.
  - https://github.com/MrDavidAlv/stockhub-api/tree/main/src/main/java/com/stockhub/interfaces

Idea clave de los puertos y adaptadores: el dominio declara que necesita "generar un PDF" o "enviar un correo" mediante interfaces (puertos), y la infraestructura provee la implementacion (adaptador). Asi el caso de uso de inventario no depende de iText ni de JavaMailSender, solo de sus puertos.

- Puerto de PDF: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/domain/port/PdfPort.java
- Adaptador de PDF: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/pdf/PdfAdapter.java
- Puerto de correo: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/domain/port/MailPort.java
- Adaptador de correo: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/mail/MailAdapter.java

Flujo de una peticion (ejemplo POST /api/empresas):
Controller (interfaces) -> Service interface (application) -> ServiceImpl (application) -> Repository port (domain) -> Spring Data implementa el port contra PostgreSQL.

---

## 2. Inyeccion de dependencias

Que es: en vez de que una clase cree sus dependencias con `new`, las recibe desde afuera. Quien las crea y las "inyecta" es el contenedor de Spring (el ApplicationContext). Esto es el principio D de SOLID (inversion de dependencias): se depende de abstracciones, no de implementaciones concretas.

Como se hace en StockHub: inyeccion por constructor, que es la forma recomendada (permite campos `final`, facilita los tests y deja explicitas las dependencias). Se usa Lombok con `@RequiredArgsConstructor`, que genera el constructor con todos los campos `final` en tiempo de compilacion.

Ejemplo, el servicio de empresas declara sus dos dependencias como campos final y Lombok genera el constructor:
- https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/application/service/impl/EmpresaServiceImpl.java (lineas 17-23: `@RequiredArgsConstructor` + `private final EmpresaRepository`, `private final EmpresaMapper`).

Otro ejemplo con cuatro dependencias (servicio de autenticacion): AuthenticationManager, UsuarioRepository, JwtService y RefreshTokenService:
- https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/application/service/impl/AuthServiceImpl.java (lineas 20-27).

Como sabe Spring que objeto inyectar:
- Las clases se registran como beans con anotaciones de estereotipo: `@Service` (servicios), `@Component` (componentes genericos como filtros, seeder, adaptadores), `@RestController` (controladores), `@Configuration` (configuracion), `@Repository` (Spring Data lo aplica solo).
- Cuando un campo es una interfaz (por ejemplo `EmpresaService`), Spring inyecta la unica implementacion disponible (`EmpresaServiceImpl`). Esto es lo que permite que el controlador dependa de la interfaz, no de la clase concreta.

Beans definidos manualmente con `@Bean` (cuando no basta con anotar la clase, por ejemplo porque el tipo viene de una libreria):
- https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/config/SecurityConfig.java
  - `passwordEncoder()` (linea 40): expone un `BCryptPasswordEncoder(12)` como bean. Cualquiera que pida un `PasswordEncoder` recibe este.
  - `authenticationManager(...)` (linea 45), `authProvider()` (linea 50), `securityFilterChain(...)` (linea 58), `corsConfigurationSource()` (linea 76).

Inyeccion de valores de configuracion con `@Value`: no solo se inyectan objetos, tambien properties. Ejemplos:
- El secreto y la duracion del JWT: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/security/JwtService.java (lineas 21-26, `@Value("${jwt.secret}")` y `@Value("${jwt.access-token-minutes}")` en el constructor).
- Las credenciales seed: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/seeder/DataSeeder.java (lineas 21-31).
- Los origenes CORS: SecurityConfig linea 37-38.

Estos valores salen de `application.yml`:
- https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/resources/application.yml

---

## 3. Patrones de diseno

Patrones presentes en el codigo y donde verlos:

- Repository (repositorio): abstrae el acceso a datos detras de una interfaz. Aqui los repositorios son los puertos del dominio y Spring Data genera la implementacion.
  - https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/domain/port/ProductoRepository.java (note los metodos derivados `findByCodigo`, `findByEmpresaNit`, `existsByCodigo`: Spring los implementa a partir del nombre).

- Adapter (adaptador): adapta una libreria externa a un puerto del dominio. iText y JavaMailSender quedan encapsulados.
  - PDF: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/pdf/PdfAdapter.java
  - Mail: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/mail/MailAdapter.java

- DTO (Data Transfer Object): objetos planos para la entrada y salida de la API, separados de las entidades JPA. Implementados como `record` de Java 17 (inmutables).
  - https://github.com/MrDavidAlv/stockhub-api/tree/main/src/main/java/com/stockhub/application/dto

- Mapper: conversion entre entidad y DTO, generada por MapStruct.
  - https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/application/mapper/ProductoMapper.java

- Builder: construccion fluida de objetos, via Lombok `@Builder`. Util para entidades con muchos campos.
  - Entidad Producto con `@Builder`: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/domain/model/Producto.java
  - Uso del builder al crear el refresh token: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/security/RefreshTokenService.java (lineas 26-31).
  - Uso del builder al crear un usuario seed: DataSeeder lineas 43-49.

- Strategy (estrategia) via interfaz: `PasswordEncoder` es una abstraccion; la estrategia concreta es BCrypt. Cambiar de algoritmo es cambiar el bean, no el codigo que lo usa.
  - SecurityConfig linea 40-43.

- Chain of Responsibility (cadena de filtros): el JwtAuthFilter se inserta en la cadena de filtros de Spring Security antes del filtro de usuario/contrasena.
  - Filtro: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/security/JwtAuthFilter.java
  - Registro en la cadena: SecurityConfig linea 72 (`addFilterBefore`).

- Singleton: todos los beans de Spring son singleton por defecto (una sola instancia gestionada por el contenedor). No se implementa a mano; lo da el framework.

- Interceptor (en el front): interceptores de Axios para request y response.
  - https://github.com/MrDavidAlv/stockhub-web/blob/main/src/services/api.service.ts

---

## 4. Spring Data JPA / Hibernate

Que es cada cosa:
- JPA (Jakarta Persistence API) es la especificacion estandar de mapeo objeto-relacional en Java. Define anotaciones como `@Entity`, `@Id`, `@ManyToOne`.
- Hibernate es la implementacion de JPA que usa Spring Boot por defecto. Traduce las operaciones sobre objetos a SQL.
- Spring Data JPA es una capa encima: genera la implementacion de los repositorios a partir de una interfaz que extiende `JpaRepository`. No se escribe el CRUD a mano.

Donde verlo:
- Una entidad completa con relaciones: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/domain/model/Producto.java
  - `@Entity` y `@Table(name = "productos")` mapean la clase a la tabla.
  - `@Id` + `@GeneratedValue(strategy = IDENTITY)`: clave primaria autoincremental delegada a PostgreSQL.
  - `@ManyToOne` hacia Empresa (muchos productos por empresa).
  - `@OneToMany(mappedBy = "producto", cascade = ALL, orphanRemoval = true)` hacia PrecioMoneda: los precios viven y mueren con el producto.
  - `@ManyToMany` con `@JoinTable(name = "producto_categoria")`: relacion N:M con tabla puente.
- Un repositorio con consultas derivadas: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/domain/port/ProductoRepository.java
  - `findByCodigo`, `findByEmpresaNit`, `existsByCodigo`: Spring construye el SQL a partir del nombre del metodo.

Configuracion relevante en application.yml (lineas 13-20):
- `ddl-auto: validate`: Hibernate NO crea ni altera tablas; solo valida que el esquema (creado por Flyway) coincide con las entidades. Es la opcion segura para produccion.
- `open-in-view: false`: desactiva el patron Open Session In View. Obliga a que las consultas LAZY ocurran dentro de la capa transaccional (servicios), no durante el render de la respuesta. Evita sorpresas de carga perezosa y consultas ocultas en la serializacion.
- `batch_size: 25`: agrupa inserts/updates en lotes.

Transacciones: los servicios se anotan con `@Transactional`. Las lecturas con `@Transactional(readOnly = true)`.
- Ejemplo: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/application/service/impl/EmpresaServiceImpl.java (clase con `@Transactional`, lecturas con `readOnly = true` en lineas 26 y 34).

---

## 5. Carga LAZY y el problema N+1

Que es LAZY vs EAGER:
- LAZY (perezosa): la relacion no se carga de la base hasta que se accede a ella. Es el default recomendado para `@ManyToOne` y `@OneToMany` aqui.
- EAGER (ansiosa): se carga siempre junto a la entidad. Puede traer datos de mas.

En Producto, la relacion con Empresa es `@ManyToOne(fetch = FetchType.LAZY)`:
- https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/domain/model/Producto.java (linea 38).

Por que importa: cargar siempre todo en EAGER degrada el rendimiento. Con LAZY se controla cuando traer cada relacion.

Riesgos que se manejaron:
- LazyInitializationException: si se intenta leer una relacion LAZY fuera de la transaccion (sesion de Hibernate cerrada), Hibernate lanza esta excepcion. Apareció al rotar el refresh token, porque el Usuario era un proxy LAZY y se leia tras cerrar la sesion. Se resolvio anotando el servicio con `@Transactional` a nivel de clase para que la sesion siga abierta mientras se construye la respuesta.
  - https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/application/service/impl/AuthServiceImpl.java (linea 21, `@Transactional` de clase).
- Como `open-in-view` esta en false, todo acceso a relaciones LAZY debe ocurrir dentro del servicio transaccional. Por eso el PdfAdapter recibe los productos ya cargados con sus precios y categorias desde el servicio de inventario, y no navega relaciones por su cuenta fuera de transaccion.

Problema N+1 (concepto que suelen preguntar): cargar una lista de N entidades y luego, por cada una, disparar una consulta extra para una relacion (1 + N consultas). Se mitiga con consultas con `JOIN FETCH` o `@EntityGraph` cuando se necesita traer la relacion en bloque. En el alcance actual los volumenes son bajos; es un punto honesto para mencionar como mejora futura.

---

## 6. Flyway

Que es: herramienta de migraciones de base de datos versionadas. Cada cambio de esquema es un script SQL con un numero de version. Flyway lleva una tabla de control (`flyway_schema_history`) y aplica en orden los scripts que falten. Garantiza que todos los entornos tengan el mismo esquema de forma reproducible.

Donde verlo:
- Esquema completo (11 tablas, constraints, enums via CHECK, FKs): https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/resources/db/migration/V1__create_schema.sql
- Datos semilla del catalogo (categorias y empresas de ejemplo): https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/resources/db/migration/V2__seed_catalogo.sql

Configuracion: application.yml lineas 21-23 (`flyway.enabled: true`, `locations: classpath:db/migration`).

Decision importante: los usuarios NO se crean por Flyway. Las contrasenas no deben ir en texto plano ni hardcodeadas en SQL. Se crean en runtime con BCrypt mediante el DataSeeder (ver seccion 9). Flyway solo siembra catalogo no sensible.

Convencion de nombres: `V<numero>__<descripcion>.sql`. El doble guion bajo separa version y descripcion.

---

## 7. Spring Security

Configuracion central: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/config/SecurityConfig.java

Puntos clave (con linea):
- `@EnableWebSecurity` y `@EnableMethodSecurity` (lineas 29-30): habilitan la seguridad web y la seguridad a nivel de metodo (`@PreAuthorize`).
- `csrf.disable()` (linea 61): se desactiva CSRF porque la API es stateless y no usa cookies de sesion; la proteccion CSRF aplica a sesiones con cookies.
- `sessionCreationPolicy(STATELESS)` (linea 63): el servidor no guarda sesion; cada peticion se autentica por su JWT.
- Reglas de autorizacion por ruta (lineas 64-70):
  - Publico: `/api/auth/**`, `/actuator/health`, `/actuator/info`, Swagger, y GET de `/api/empresas/**`.
  - Todo lo demas: autenticado.
- `addFilterBefore(jwtAuthFilter, ...)` (linea 72): inserta el filtro JWT antes del filtro estandar de usuario/contrasena.

Autorizacion por rol a nivel de metodo: los endpoints de escritura exigen rol ADMIN con `@PreAuthorize("hasRole('ADMIN')")`.
- Empresa (GET publico, escritura solo ADMIN): https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/interfaces/rest/controller/EmpresaController.java (lineas 41-58).
- Producto (toda la clase ADMIN): https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/interfaces/rest/controller/ProductoController.java
- Inventario (toda la clase ADMIN): https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/interfaces/rest/controller/InventarioController.java

De donde salen los roles: el UserDetails se construye en UserDetailsServiceImpl a partir del Usuario y su Rol.
- https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/security/UserDetailsServiceImpl.java
- Enum de roles: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/domain/model/Rol.java

Como autentica el login: AuthServiceImpl delega en el AuthenticationManager de Spring, que usa el DaoAuthenticationProvider (UserDetailsService + PasswordEncoder) para validar credenciales.
- https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/application/service/impl/AuthServiceImpl.java (lineas 30-40).

---

## 8. JWT y refresh tokens

Que es un JWT: un token firmado que contiene claims (datos) sobre el usuario. Aqui lleva el email (subject) y el rol. El servidor lo verifica con su clave secreta sin consultar la base, lo que lo hace stateless.

Generacion y validacion del access token: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/security/JwtService.java
- Algoritmo HMAC con la clave derivada del secreto (`Keys.hmacShaKeyFor`, linea 24). La firma simetrica HS-256/384 segun el tamano de la clave.
- `generateAccessToken` (linea 28): subject = email, claim `rol`, fechas de emision y expiracion (60 min, configurable).
- `extractEmail` (linea 39) y `isValid` (linea 43): parsean y verifican la firma. Si falla, devuelve false (no propaga la excepcion).

Como se usa el access token en cada peticion: el JwtAuthFilter intercepta, lee el header Authorization, valida el token, carga el usuario y coloca la autenticacion en el contexto de seguridad.
- https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/security/JwtAuthFilter.java (extiende OncePerRequestFilter; lineas 33-44).

Refresh token: el access token dura poco (60 min) por seguridad. Para no obligar al usuario a re-loguear, existe un refresh token de larga duracion (7 dias) que se persiste en base de datos.
- Servicio: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/security/RefreshTokenService.java
  - `create` (linea 24): genera un token aleatorio y lo guarda con fecha de expiracion.
  - `rotate` (linea 35): valida el viejo, lo marca como revocado y emite uno nuevo. Esto es rotacion: cada refresh invalida el anterior, lo que limita el dano si uno se filtra.
  - `revoke` (linea 43): para el logout.
  - `validate` (linea 51): rechaza tokens inexistentes, revocados o expirados con InvalidTokenException.
- Entidad persistida: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/domain/model/RefreshToken.java

Por que persistir el refresh token: un JWT firmado no se puede invalidar antes de que expire (no hay estado en el servidor). Guardar el refresh token en base permite revocarlo (logout) y rotarlo. Es el equilibrio entre el stateless del access token y la necesidad de revocacion.

Flujo completo (orquestado en AuthServiceImpl):
- login -> valida credenciales -> crea refresh token + genera access token -> devuelve el par.
- refresh -> rota el refresh -> genera nuevo access.
- logout -> revoca el refresh.
- https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/application/service/impl/AuthServiceImpl.java

Endpoints: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/interfaces/rest/controller/AuthController.java

---

## 9. BCrypt y el seeder

BCrypt: algoritmo de hashing de contrasenas con salt incorporado y factor de costo (work factor). Aqui cost 12. Es lento a proposito: dificulta los ataques de fuerza bruta. La misma contrasena produce hashes distintos por el salt aleatorio.
- Bean del encoder: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/config/SecurityConfig.java (linea 40-43, `new BCryptPasswordEncoder(12)`).

Seeder de usuarios: crea admin y externo en el arranque si no existen, hasheando la contrasena con BCrypt. Implementa CommandLineRunner (se ejecuta al iniciar la app).
- https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/seeder/DataSeeder.java
  - `passwordEncoder.encode(rawPassword)` (linea 45): nunca se guarda la contrasena en claro.
  - Las contrasenas se leen de properties (`seed.admin-password`, `seed.externo-password`), que en produccion vienen de secrets, no del codigo.

Por que en runtime y no en SQL: poner hashes en una migracion SQL acopla el hash al codigo y los expone en el repositorio. Generarlos en runtime mantiene el secreto fuera del codigo y permite cambiar la contrasena via variable de entorno.

---

## 10. MapStruct y DTOs

MapStruct: genera el codigo de conversion entre entidades y DTOs en tiempo de compilacion (sin reflexion, por eso es rapido y verificable). Se declara una interfaz anotada con `@Mapper` y MapStruct crea la implementacion.
- https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/application/mapper/ProductoMapper.java
  - `@Mapper(componentModel = "spring", uses = {CategoriaMapper.class})`: la implementacion se registra como bean de Spring y reutiliza el mapper de categorias para anidar.
  - `@Mapping(target = "empresaNit", source = "empresa.nit")`: aplana la relacion (de la entidad Empresa al campo plano del DTO).
- Otros mappers: https://github.com/MrDavidAlv/stockhub-api/tree/main/src/main/java/com/stockhub/application/mapper

DTOs como records: entrada con Bean Validation, salida plana.
- https://github.com/MrDavidAlv/stockhub-api/tree/main/src/main/java/com/stockhub/application/dto
- Ejemplo de request validado: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/application/dto/EmpresaRequest.java

Por que DTOs: no exponer las entidades JPA directamente evita filtrar la estructura interna, romper la serializacion con relaciones LAZY y acoplar la API al modelo de base.

---

## 11. Manejo de errores

GlobalExceptionHandler centraliza la traduccion de excepciones a respuestas HTTP consistentes, con un cuerpo de error uniforme. Usa `@RestControllerAdvice`.
- https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/interfaces/rest/exception/GlobalExceptionHandler.java
- Cuerpo de error: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/interfaces/rest/exception/ErrorResponse.java

Mapeos (codigo de negocio a HTTP):
- NotFoundException -> 404 NOT_FOUND.
- DuplicateException -> 409 CONFLICT.
- InvalidCredentialsException -> 401 INVALID_CREDENTIALS.
- InvalidTokenException -> 401 INVALID_TOKEN.
- AccessDeniedException -> 403 FORBIDDEN.
- Validacion (Bean Validation) -> 400 con mapa de errores por campo.
- MailDeliveryException -> 502, PdfGenerationException -> 500.
- Generico -> 500 sin filtrar el stack trace.

Excepciones de dominio: https://github.com/MrDavidAlv/stockhub-api/tree/main/src/main/java/com/stockhub/domain/exception

El frontend mapea estos errores al campo correspondiente del formulario:
- https://github.com/MrDavidAlv/stockhub-web/blob/main/src/utils/error.ts

---

## 12. CORS y headers de seguridad

CORS (Cross-Origin Resource Sharing): politica del navegador que decide que origenes pueden llamar a la API. Se configura en SecurityConfig leyendo los origenes desde la variable `CORS_ORIGINS`.
- https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/config/SecurityConfig.java (lineas 76-86; origenes en linea 37-38 y 79).

Headers de seguridad OWASP: se aplican en el nginx del frontend y en el reverse-proxy de produccion (CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy).
- nginx del front: https://github.com/MrDavidAlv/stockhub-web/blob/main/nginx.conf
- reverse-proxy de produccion: https://github.com/MrDavidAlv/stockhub-api/blob/main/deploy/nginx.conf

Detalle aprendido: nginx no hereda `add_header` en una location que define su propio `add_header`; por eso los headers se repiten en cada location.

---

## 13. Pinia (estado en el front)

Que es: la libreria oficial de manejo de estado para Vue 3 (sucesor de Vuex). Centraliza estado compartido entre componentes en "stores" con state, getters y actions.

Store de autenticacion: https://github.com/MrDavidAlv/stockhub-web/blob/main/src/stores/auth.store.ts
- state (lineas 28-32): accessToken, refreshToken y user, inicializados desde localStorage para sobrevivir recargas de pagina.
- getters (lineas 34-37):
  - `isAuthenticated`: hay access token?
  - `isAdmin`: el rol del usuario es ADMIN? Este getter alimenta tanto la UI (mostrar botones) como el router (proteger rutas).
- actions (lineas 39-87):
  - `login` (linea 40): pide tokens al backend y los aplica. Usa axios directo (no la instancia con interceptores) para no inyectarse el token a si mismo.
  - `tryRefresh` (linea 45): intenta renovar con el refresh token. Lo llama el interceptor de Axios ante un 401.
  - `logout` (linea 58): limpia el estado local y notifica al backend best-effort (si falla, el usuario ya quedo deslogueado localmente).
  - `applyTokens` (linea 70) y `clear` (linea 79): escriben y borran en localStorage con claves prefijadas `stockhub.*`.

Por que un store y no props: el estado de sesion lo necesitan el layout, el router y varias vistas. Centralizarlo evita pasar props en cascada y mantiene una sola fuente de verdad.

Registro de Pinia (boot file de Quasar): https://github.com/MrDavidAlv/stockhub-web/blob/main/src/boot/pinia.ts

---

## 14. Axios e interceptores

Que es Axios: cliente HTTP para el navegador. Aqui se crea una instancia configurada con baseURL y timeout, y dos interceptores.
- https://github.com/MrDavidAlv/stockhub-web/blob/main/src/services/api.service.ts
  - Interceptor de request (lineas 19-25): inyecta `Authorization: Bearer <accessToken>` en cada peticion si hay sesion.
  - Interceptor de response (lineas 27-47): ante un 401, si la peticion no es de `/auth/` y no se ha reintentado, marca `_retry = true`, intenta refrescar el token (`auth.tryRefresh()`) y reintenta la peticion original una sola vez. Si el refresh falla, limpia la sesion. Evita bucles infinitos y re-logueos innecesarios.

Servicios tipados por recurso (usan esta instancia):
- https://github.com/MrDavidAlv/stockhub-web/blob/main/src/services/empresa.service.ts
- https://github.com/MrDavidAlv/stockhub-web/blob/main/src/services/producto.service.ts
- https://github.com/MrDavidAlv/stockhub-web/blob/main/src/services/categoria.service.ts
- https://github.com/MrDavidAlv/stockhub-web/blob/main/src/services/inventario.service.ts

Tipos compartidos (TypeScript): https://github.com/MrDavidAlv/stockhub-web/blob/main/src/types/index.ts

---

## 15. Vue Router y guards

Que es: el enrutador SPA. Un navigation guard (`beforeEach`) corre antes de cada cambio de ruta y decide si permitir, redirigir o bloquear.
- https://github.com/MrDavidAlv/stockhub-web/blob/main/src/router/index.ts (lineas 24-36):
  - Si la ruta requiere auth y no hay sesion -> redirige a login.
  - Si la ruta requiere ADMIN y el usuario no lo es -> redirige a empresas.
  - Si la ruta es solo para invitados (login) y ya hay sesion -> redirige a empresas.
- Definicion de rutas y sus meta-flags (`requiresAuth`, `requiresAdmin`, `guestOnly`): https://github.com/MrDavidAlv/stockhub-web/blob/main/src/router/routes.ts

Importante para la entrevista: el guard del front es UX, no seguridad real. La seguridad de verdad esta en el backend (Spring Security + @PreAuthorize). El front oculta lo que el usuario no puede hacer, pero aunque alguien manipule el front, el backend rechaza la operacion.

Vistas y layout:
- Layout con tabs por rol: https://github.com/MrDavidAlv/stockhub-web/blob/main/src/layouts/MainLayout.vue
- Login: https://github.com/MrDavidAlv/stockhub-web/blob/main/src/pages/LoginPage.vue
- Empresas: https://github.com/MrDavidAlv/stockhub-web/blob/main/src/pages/EmpresaPage.vue
- Productos: https://github.com/MrDavidAlv/stockhub-web/blob/main/src/pages/ProductoPage.vue
- Inventario: https://github.com/MrDavidAlv/stockhub-web/blob/main/src/pages/InventarioPage.vue

---

## 16. PDF y correo (adaptadores)

PDF con iText 8: el adaptador genera una tabla de inventario (Codigo, Nombre, Empresa, Precio COP, Categorias).
- https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/pdf/PdfAdapter.java
- Detalle tecnico aprendido: el PDF se escribe sobre un ByteArrayOutputStream que se declara FUERA del try-with-resources, y `toByteArray()` se llama DESPUES de cerrar el Document/PdfWriter. Si se llamaba antes, el PDF no se finalizaba y salia incompleto (solo el header). Ver el patron en lineas 37-83.

Correo con JavaMailSender: el adaptador arma un mensaje MIME multipart UTF-8 y adjunta el PDF.
- https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/mail/MailAdapter.java
- En desarrollo apunta a Mailpit (captura local de correos); en produccion a Gmail con App Password. La salud del correo NO condiciona el health de la app (`management.health.mail.enabled: false` en application.yml) para que un problema de SMTP no tumbe el deploy.

Orquestacion (servicio de inventario que carga datos y delega en los puertos):
- https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/application/service/impl/InventarioServiceImpl.java
- Controlador: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/interfaces/rest/controller/InventarioController.java

---

## 17. Pruebas

38 tests en total (unitarios + integracion).

Unitarios (Mockito, sin Spring): prueban un service o componente en aislamiento, simulando sus dependencias.
- Servicio de empresa: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/test/java/com/stockhub/unit/service/EmpresaServiceImplTest.java
- Servicio de producto: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/test/java/com/stockhub/unit/service/ProductoServiceImplTest.java
- JwtService: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/test/java/com/stockhub/unit/security/JwtServiceTest.java
- RefreshTokenService: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/test/java/com/stockhub/unit/security/RefreshTokenServiceTest.java
- PdfAdapter (extrae el texto del PDF y verifica el contenido): https://github.com/MrDavidAlv/stockhub-api/blob/main/src/test/java/com/stockhub/unit/pdf/PdfAdapterTest.java

Integracion (Testcontainers PostgreSQL 16 + MockMvc): levantan una base real en Docker y prueban el flujo completo.
- Base abstracta (arranca el contenedor): https://github.com/MrDavidAlv/stockhub-api/blob/main/src/test/java/com/stockhub/integration/AbstractIntegrationTest.java
- Flujo de autenticacion: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/test/java/com/stockhub/integration/AuthIntegrationTest.java
- CRUD de empresa con roles: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/test/java/com/stockhub/integration/EmpresaCrudIntegrationTest.java

Detalle aprendido: el contenedor de Testcontainers se arranca en un bloque `static {}` (no con `@Container`), para que sobreviva entre clases de test gracias al cache de contexto de Spring. Con `@Container` el contenedor se apagaba al terminar la primera clase y la siguiente reusaba un JDBC URL muerto.

---

## 18. Resumen de archivos clave

Backend (los que mas se suelen preguntar):
- Seguridad: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/config/SecurityConfig.java
- JWT: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/security/JwtService.java
- Filtro JWT: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/security/JwtAuthFilter.java
- Refresh tokens: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/security/RefreshTokenService.java
- Auth (caso de uso): https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/application/service/impl/AuthServiceImpl.java
- Seeder BCrypt: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/infrastructure/seeder/DataSeeder.java
- Entidad con relaciones: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/domain/model/Producto.java
- Manejo de errores: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/java/com/stockhub/interfaces/rest/exception/GlobalExceptionHandler.java
- Configuracion: https://github.com/MrDavidAlv/stockhub-api/blob/main/src/main/resources/application.yml
- Migraciones: https://github.com/MrDavidAlv/stockhub-api/tree/main/src/main/resources/db/migration

Frontend:
- Store de auth (Pinia): https://github.com/MrDavidAlv/stockhub-web/blob/main/src/stores/auth.store.ts
- Axios + interceptores: https://github.com/MrDavidAlv/stockhub-web/blob/main/src/services/api.service.ts
- Router + guards: https://github.com/MrDavidAlv/stockhub-web/blob/main/src/router/index.ts
- Tipos compartidos: https://github.com/MrDavidAlv/stockhub-web/blob/main/src/types/index.ts

DevOps:
- Dockerfile backend: https://github.com/MrDavidAlv/stockhub-api/blob/main/Dockerfile
- Dockerfile frontend: https://github.com/MrDavidAlv/stockhub-web/blob/main/Dockerfile
- Compose de produccion: https://github.com/MrDavidAlv/stockhub-api/blob/main/docker-compose.prod.yml
- CI/CD backend: https://github.com/MrDavidAlv/stockhub-api/blob/main/.github/workflows/ci.yml
- CI/CD frontend: https://github.com/MrDavidAlv/stockhub-web/blob/main/.github/workflows/ci.yml

---

## 19. Diagrama del flujo de autenticacion (JWT + refresh)

Login y uso del access token:

```
  Frontend (Vue)                 Backend (Spring Boot)              PostgreSQL
       |                                |                               |
       | POST /api/auth/login           |                               |
       | { email, password }            |                               |
       |------------------------------->|                               |
       |                                | AuthenticationManager valida   |
       |                                | con UserDetailsService +       |
       |                                | BCrypt                         |
       |                                |------------------------------>|
       |                                |  busca usuario por email       |
       |                                |<------------------------------|
       |                                | crea refresh token y lo guarda |
       |                                |------------------------------>|
       |                                | genera access token (JWT,60min)|
       | 200 { accessToken,             |                               |
       |       refreshToken, rol,       |                               |
       |       nombre, email }          |                               |
       |<-------------------------------|                               |
       | Pinia guarda en localStorage   |                               |
       |                                |                               |
       | GET /api/inventario            |                               |
       | Authorization: Bearer <access> |                               |
       |------------------------------->|                               |
       |                                | JwtAuthFilter valida la firma  |
       |                                | y pone la auth en el contexto  |
       |                                | @PreAuthorize chequea el rol   |
       | 200 datos                      |                               |
       |<-------------------------------|                               |
```

Renovacion cuando el access token expira (lo dispara el interceptor de Axios ante un 401):

```
       | GET /api/... (access vencido)  |
       |------------------------------->|
       | 401 Unauthorized               |
       |<-------------------------------|
       | interceptor: _retry = true     |
       | POST /api/auth/refresh         |
       | { refreshToken }               |
       |------------------------------->|
       |                                | rotate(): revoca el viejo y    |
       |                                | crea uno nuevo                 |
       |                                |------------------------------>|
       | 200 { nuevo access, nuevo      |                               |
       |       refresh }                |                               |
       |<-------------------------------|                               |
       | reintenta la peticion original |
       |------------------------------->|
       | 200 datos                      |
       |<-------------------------------|
```

Si el refresh tambien falla (vencido o revocado): el interceptor limpia la sesion y el guard del router manda al login.

---

## 20. Preguntas y respuestas de entrevista (Java + Vue)

Respuestas cortas y honestas, alineadas con lo que hace StockHub. Si no sabes algo, dilo y razona en voz alta; eso vale mas que inventar.

### Generales y de arquitectura

- Que es Clean Architecture y que ganaste con la hexagonal?
  Separar el negocio de los detalles tecnicos. El dominio no conoce Spring ni la base. Gane testeo facil (mockeo puertos) y libertad para cambiar tecnologias sin tocar la logica.

- Diferencia entre autenticacion y autorizacion?
  Autenticacion es verificar quien eres (login). Autorizacion es que puedes hacer (roles, permisos). En StockHub: login con JWT (autenticacion), `@PreAuthorize("hasRole('ADMIN')")` (autorizacion).

- Por que JWT y no sesiones de servidor?
  La API es stateless: no guarda sesion, escala horizontalmente sin sticky sessions ni almacen de sesiones compartido. El token lleva la identidad firmada y se verifica sin tocar la base.

- Desventaja del JWT y como la mitigaste?
  Un JWT firmado no se puede invalidar antes de que expire. Por eso el access dura poco (60 min) y uso un refresh token persistido en base que puedo revocar y rotar.

- Que es la rotacion de refresh tokens?
  Cada vez que se usa un refresh, se revoca y se emite uno nuevo. Si un refresh se filtra y el atacante lo usa, el legitimo deja de servir y se detecta el problema.

- Donde guardas el token en el front? Riesgos?
  En localStorage. Es practico pero vulnerable a XSS (un script malicioso podria leerlo). La alternativa es una cookie HttpOnly + SameSite, que protege de XSS pero exige manejar CSRF. Es un trade-off conocido; para el alcance del reto localStorage es razonable.

- REST: que lo define?
  Recursos identificados por URL, verbos HTTP con semantica (GET lee, POST crea, PUT reemplaza, DELETE borra), sin estado entre peticiones, codigos de estado significativos.

- Que verbos son idempotentes?
  GET, PUT y DELETE (repetir la peticion deja el mismo estado). POST no lo es (crea uno nuevo cada vez).

- Como manejas errores de forma consistente?
  Un GlobalExceptionHandler con @RestControllerAdvice que mapea cada excepcion a un codigo HTTP y un cuerpo uniforme.

### Java y Spring

- Que es la inyeccion de dependencias y como la haces?
  Que el contenedor provea las dependencias en vez de crearlas con new. La hago por constructor (campos final, `@RequiredArgsConstructor` de Lombok), que facilita tests y deja claras las dependencias.

- Diferencia entre inyeccion por constructor, por campo y por setter?
  Constructor: dependencias obligatorias e inmutables, la preferida. Campo (`@Autowired` en el atributo): comoda pero dificil de testear y oculta dependencias. Setter: para dependencias opcionales.

- Que diferencia hay entre @Component, @Service, @Repository y @Controller?
  Todas registran un bean. Son estereotipos semanticos: @Service para logica, @Repository para acceso a datos (y traduce excepciones de persistencia), @Controller/@RestController para la web. @Component es el generico.

- Scope por defecto de un bean de Spring?
  Singleton: una sola instancia por contenedor.

- Que es @Transactional y que hace readOnly?
  Demarca una transaccion: o todo se confirma o todo se revierte. `readOnly = true` es una pista de optimizacion para lecturas (no flush de cambios).

- Diferencia entre JPA, Hibernate y Spring Data JPA?
  JPA es la especificacion (interfaces y anotaciones). Hibernate es la implementacion. Spring Data JPA genera repositorios sobre Hibernate a partir de interfaces.

- Que es LAZY vs EAGER? Que es el N+1?
  LAZY carga la relacion al accederla; EAGER siempre. El N+1 es disparar 1 consulta por la lista y N adicionales por cada relacion; se evita con JOIN FETCH o @EntityGraph.

- Que es el equals/hashCode y por que importa en entidades JPA?
  Definen igualdad. En entidades hay que tener cuidado: basarse en el id generado puede romper sets antes de persistir. Es un tema clasico de Hibernate.

- Checked vs unchecked exceptions?
  Checked obligan a declararlas o capturarlas (Exception). Unchecked extienden RuntimeException y no obligan. En la API uso unchecked de dominio y las traduce el handler.

- Que es el garbage collector?
  El recolector de basura de la JVM: libera automaticamente la memoria de objetos sin referencias en el heap, para no gestionarla a mano.

- Que hace `final` en Java?
  En variable: no se reasigna. En metodo: no se sobreescribe. En clase: no se hereda.

- record vs class?
  record (Java 16+) es una clase inmutable y concisa para portar datos; genera constructor, getters, equals, hashCode y toString. Lo uso para DTOs.

### Vue y frontend

- Composition API vs Options API?
  Options organiza el componente por opciones (data, methods, computed). Composition agrupa la logica por funcionalidad con `setup`, `ref`, `computed`; escala mejor y reutiliza logica con composables.

- Que es la reactividad en Vue?
  Vue rastrea dependencias: cuando cambia un dato reactivo (`ref`/`reactive`), las vistas y computeds que lo usan se actualizan solos.

- ref vs reactive?
  `ref` envuelve cualquier valor (se accede con `.value`); `reactive` hace reactivo un objeto. ref es mas comun y compone mejor.

- Que es Pinia y por que no solo props?
  Estado global compartido entre componentes. Evita pasar props en cascada cuando varias vistas necesitan lo mismo (ej. la sesion). Una sola fuente de verdad.

- Que es un SPA y que implica para el enrutado y el servidor?
  Single Page Application: el navegador carga la app una vez y navega en el cliente. El servidor debe devolver index.html para rutas desconocidas (SPA fallback en nginx).

- Como proteges rutas en el front y por que no basta?
  Con navigation guards (`beforeEach`) que leen el rol del store. No basta porque es UX: la seguridad real esta en el backend; el front solo oculta.

- Para que sirven los interceptores de Axios?
  Centralizar logica transversal: inyectar el token en cada request y, en el response, refrescar el token ante un 401 y reintentar una vez.

- Que es el Virtual DOM?
  Una representacion en memoria del DOM. Vue compara (diffing) el estado nuevo con el anterior y aplica solo los cambios necesarios al DOM real, que es costoso.

- Que aporta TypeScript?
  Tipos estaticos: errores en compilacion, autocompletado y contratos claros (interfaces compartidas entre servicios y componentes).

### DevOps

- Por que multi-stage en el Dockerfile?
  Una etapa compila con el JDK/Node y otra solo ejecuta con el runtime. La imagen final es mas pequena y sin herramientas de build.

- Por que usuario no-root en el contenedor?
  Principio de menor privilegio: si se compromete el proceso, no corre como root dentro del contenedor.

- Que valida el HEALTHCHECK?
  Que la app responde (`/actuator/health`). Docker y el reverse-proxy lo usan para saber si el contenedor esta sano antes de enviarle trafico.

- Que hace tu pipeline de CI/CD?
  En push a main: compila y corre tests, construye y publica la imagen en ghcr.io, y despliega por SSH al EC2. Si algo falla, no llega a produccion.

---

## 21. Glosario

Terminos por area, con una definicion breve. Util para repasar vocabulario antes de la entrevista.

### Java y JVM

- JVM (Java Virtual Machine): maquina virtual que ejecuta el bytecode de Java. Hace a Java portable ("compila una vez, corre en cualquier lado").
- JDK (Java Development Kit): kit de desarrollo; incluye el compilador (javac) y herramientas. Necesario para compilar.
- JRE (Java Runtime Environment): entorno de ejecucion; solo corre programas, no compila.
- Bytecode: codigo intermedio (.class) que produce el compilador y ejecuta la JVM.
- JIT (Just-In-Time compiler): compila el bytecode mas usado a codigo nativo en tiempo de ejecucion para acelerar.
- GC (Garbage Collector): recolector de basura; libera memoria de objetos sin referencias en el heap automaticamente.
- Heap: zona de memoria donde viven los objetos. La gestiona el GC.
- Stack: pila de llamadas; guarda variables locales y marcos de metodo. Se libera al salir del metodo.
- POJO (Plain Old Java Object): objeto Java simple, sin dependencias de un framework.
- Bean: objeto gestionado por un contenedor (en Spring, por el ApplicationContext).
- JAR / WAR: empaquetados de Java. JAR es una libreria o app; WAR es una app web para un servidor de aplicaciones. StockHub usa JAR con Tomcat embebido.
- Lombok: libreria que genera codigo repetitivo (getters, constructores, builder) con anotaciones.
- Stream: API para procesar colecciones de forma declarativa (map, filter, collect).
- Optional: contenedor que puede tener o no un valor; evita NullPointerException explicitando la ausencia.
- Checked / unchecked exception: las checked obligan a manejarlas; las unchecked (RuntimeException) no.
- Generics: tipos parametrizados (List<Empresa>) que dan seguridad de tipos en compilacion.

### Spring y backend

- IoC (Inversion of Control): el framework controla el ciclo de vida y el ensamblado de objetos, no tu codigo.
- DI (Dependency Injection): forma de IoC; las dependencias se inyectan desde afuera.
- ApplicationContext: el contenedor de Spring que crea y gestiona los beans.
- AOP (Aspect Oriented Programming): separa preocupaciones transversales (transacciones, seguridad) en aspectos. `@Transactional` se apoya en AOP.
- ORM (Object-Relational Mapping): mapea objetos a tablas. Hibernate es un ORM.
- JPA: especificacion estandar de persistencia en Java.
- DTO (Data Transfer Object): objeto plano para transportar datos entre capas o hacia el cliente.
- DAO / Repository: capa de acceso a datos. En Spring Data, una interfaz que extiende JpaRepository.
- Entity (entidad): clase mapeada a una tabla con @Entity.
- Migracion: script versionado que evoluciona el esquema de base (Flyway).
- Transaccion: unidad de trabajo atomica; cumple ACID.
- ACID: Atomicidad, Consistencia, Aislamiento, Durabilidad. Propiedades de una transaccion confiable.
- Indice: estructura que acelera busquedas en una tabla a costa de espacio y escrituras.
- Connection pool: conjunto de conexiones reutilizables a la base (HikariCP por defecto en Spring Boot).
- Actuator: modulo de Spring Boot con endpoints de salud y metricas.
- Filter: componente que intercepta peticiones HTTP antes de llegar al controlador (ej. JwtAuthFilter).
- Middleware: termino general para codigo que se ejecuta entre la peticion y la respuesta.

### Seguridad

- Autenticacion: verificar quien eres.
- Autorizacion: verificar que puedes hacer.
- JWT (JSON Web Token): token firmado con claims; permite autenticacion stateless.
- Claim: dato dentro del JWT (ej. subject, rol, expiracion).
- Access token / refresh token: el primero es de vida corta para autorizar peticiones; el segundo, de vida larga, para renovar el access.
- BCrypt: algoritmo de hashing de contrasenas con salt y costo configurable.
- Salt: dato aleatorio que se anade a la contrasena antes de hashear para que hashes iguales no coincidan.
- Hashing: transformacion unidireccional; no se puede revertir (a diferencia del cifrado).
- CORS (Cross-Origin Resource Sharing): politica que controla que origenes pueden llamar a la API.
- CSRF (Cross-Site Request Forgery): ataque que abusa de la sesion con cookies; no aplica a APIs stateless con token en header.
- XSS (Cross-Site Scripting): inyeccion de scripts en el navegador; puede robar tokens de localStorage.
- TLS / HTTPS: cifrado del transporte entre cliente y servidor.
- OWASP: organizacion de referencia en seguridad web; sus headers mitigan ataques comunes.
- Principio de menor privilegio: dar solo los permisos estrictamente necesarios.

### Arquitectura y patrones

- Arquitectura hexagonal (puertos y adaptadores): aisla el nucleo de negocio de la infraestructura mediante interfaces.
- Clean Architecture: dependencias apuntando hacia el dominio; los detalles externos son reemplazables.
- SOLID: cinco principios de diseno OO (responsabilidad unica, abierto/cerrado, Liskov, segregacion de interfaces, inversion de dependencias).
- Acoplamiento / cohesion: acoplamiento es cuanto depende un modulo de otro (mejor bajo); cohesion es cuanto se relacionan las responsabilidades dentro de un modulo (mejor alta).
- Puerto: interfaz que define una necesidad del dominio (PdfPort, MailPort).
- Adaptador: implementacion concreta de un puerto (PdfAdapter con iText).
- Patron Repository: abstrae el acceso a datos.
- Patron Adapter: adapta una API externa a una interfaz propia.
- Patron Builder: construccion fluida de objetos complejos.
- Patron Strategy: intercambiar algoritmos detras de una interfaz comun.
- Patron Singleton: una unica instancia (los beans de Spring lo son por defecto).
- Idempotencia: una operacion que repetida da el mismo resultado (GET, PUT, DELETE).

### Frontend

- SPA (Single Page Application): la app se carga una vez y navega en el cliente.
- CSR (Client-Side Rendering): el navegador construye la UI con JavaScript. Es lo que hace StockHub.
- SSR (Server-Side Rendering): el servidor renderiza el HTML inicial (mejor SEO y primer pintado).
- DOM (Document Object Model): representacion en arbol del HTML que el navegador manipula.
- Virtual DOM: copia en memoria del DOM; Vue calcula la diferencia y aplica solo los cambios necesarios.
- Reactividad: el sistema que actualiza la vista cuando cambian los datos.
- Componente: unidad reutilizable de UI con su plantilla, logica y estilo.
- Props: datos que un componente padre pasa al hijo.
- Store (Pinia): contenedor de estado global.
- Composable: funcion que encapsula logica reutilizable con la Composition API.
- Router guard: funcion que se ejecuta antes de navegar y decide permitir o redirigir.
- Bundler: herramienta que empaqueta el codigo y sus dependencias (Vite).
- Tree-shaking: eliminar del bundle el codigo que no se usa.
- Transpilar: convertir codigo de un lenguaje/version a otro (TypeScript a JavaScript).
- Hot reload: recarga en caliente de los cambios durante el desarrollo.
- Promise / async-await: manejo de operaciones asincronas en JavaScript.
- localStorage: almacenamiento clave-valor persistente en el navegador.

### DevOps

- Docker: empaqueta la app y sus dependencias en imagenes.
- Imagen / contenedor: la imagen es la plantilla; el contenedor es una instancia en ejecucion.
- Dockerfile: receta para construir una imagen.
- Multi-stage build: build en varias etapas para una imagen final ligera.
- Docker Compose: orquesta varios contenedores con un archivo declarativo.
- Registry: repositorio de imagenes (ghcr.io, Docker Hub).
- CI (Integracion Continua): construir y testear en cada cambio.
- CD (Despliegue/Entrega Continua): llevar el cambio a produccion de forma automatizada.
- Pipeline: secuencia de jobs de CI/CD.
- Reverse proxy: servidor que recibe el trafico y lo enruta a los servicios internos (nginx).
- Healthcheck: chequeo periodico de que un servicio responde.
- Secret: dato sensible (contrasena, clave) gestionado fuera del codigo.
- Variable de entorno: configuracion inyectada al proceso en tiempo de ejecucion.
- EC2: instancia de computo de AWS (maquina virtual).
