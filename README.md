 

DEFENSA TÉCNICA 

Evaluación Parcial 2 — DSY1103 Desarrollo FullStack 1 

 

Proyecto: Arquitectura de Microservicios 

Dominio: E-Commerce / Tienda Online 

10 Microservicios con Spring Boot + MySQL + JPA + Feign 

2025 

 

1. Visión General del Proyecto 

 

Este proyecto implementa una arquitectura de microservicios para un sistema de E-Commerce completo. Cada microservicio es completamente independiente, posee su propia base de datos MySQL y se comunica con otros servicios mediante Feign Client (HTTP REST). El dominio elegido modela todas las operaciones de una tienda online, desde el catálogo de productos hasta el envío de pedidos. 

 

1.1 Los 10 Microservicios Implementados 

 

Microservicio 

Base de Datos 

Puerto 

Responsabilidad 

ms-user 

db_user 

8081 

Gestión de usuarios y perfiles 

ms-product 

db_productos 

8082 

Catálogo de productos 

ms-category 

db_category 

8086 

Categorías de productos 

ms-cart 

db_cart 

8087 

Carrito de compras 

ms-order 

db_order 

8085 

Gestión de órdenes/pedidos 

ms-payment 

db_payment 

8091 

Procesamiento de pagos 

ms-shipping 

db_shipping 

8088 

Gestión de envíos 

ms-inventario 

db_inventario 

8084 

Control de inventario 

ms-sucursal 

db_sucursal 

8089 

Gestión de sucursales 

ms-review 

db_reviews 

8090 

Reseñas y puntuaciones 

 

2. Patrón CSR (Controller – Service – Repository) 

 

Todos los microservicios siguen estrictamente el patrón CSR. Esto significa que las responsabilidades están completamente separadas entre tres capas: el Controller recibe las solicitudes HTTP, el Service contiene toda la lógica de negocio, y el Repository se encarga del acceso a datos. 

 

2.1 Estructura de Paquetes (misma en los 10 microservicios) 

com.example.ms_[nombre]/ 

   ├── Controller/     → Recibe peticiones HTTP, delega al Service 

   ├── Service/        → Lógica de negocio, reglas, validaciones 

   ├── Repository/     → Extiende JpaRepository, acceso a BD 

   ├── Model/          → Entidades JPA (@Entity) 

   │   └── DTO/        → Objetos de transferencia entre microservicios 

   └── Client/         → Interfaces Feign para llamadas remotas 

 

2.2 Ejemplo Concreto: ms-user 

Controller — UserController.java 

Controller : recibe la consulta, llama al Service y retorna la respuesta. No contiene lógica de negocio. 

@RestController 

@RequestMapping("/api/v1/usuarios") 

public class UserController { 

    @Autowired 

    private UserService service; 

 

    @PostMapping 

    public ResponseEntity<User> crear(@RequestBody User usuario) { 

        User nuevoUsuario = service.save(usuario); 

        return new ResponseEntity<>(nuevoUsuario, HttpStatus.CREATED); 

    } 

 

    @GetMapping("/{id}") 

    public ResponseEntity<Map<String, Object>> obtenerDetalleCompleto 

                                              (@PathVariable Integer id) { 

        Map<String, Object> respuesta = service.buscarUsuarioCompleto(id); 

        if (respuesta.isEmpty()) return ResponseEntity.notFound().build(); 

        return ResponseEntity.ok(respuesta); 

    } 

} 

Service — UserService.java 

El Service contiene la lógica. En este caso, cuando se busca un usuario, también llama a ms-product usando Feign para obtener los productos del usuario. 

@Service 

public class UserService { 

    @Autowired private UserRepository repository; 

    @Autowired private ProductoFeignClient productoClient; 

 

    public User save(User usuario) { 

        // Lógica: asignar relación bidireccional antes de guardar 

        if (usuario.getPerfil() != null) { 

            usuario.getPerfil().setUsuario(usuario); 

        } 

        return repository.save(usuario); 

    } 

 

    public Map<String, Object> buscarUsuarioCompleto(Integer id) { 

        User usuario = repository.findById(id).orElse(null); 

        Map<String, Object> respuesta = new HashMap<>(); 

        if (usuario != null) { 

            // Por cada ID de producto, llama al otro microservicio 

            List<ProductoDTO> productos = usuario.getProductosIds().stream() 

                .map(pId -> productoClient.obtenerProductoPorId(pId)) 

                .collect(Collectors.toList()); 

            respuesta.put("id", usuario.getId()); 

            respuesta.put("nombre", usuario.getNombre()); 

            respuesta.put("perfil", usuario.getPerfil()); 

            respuesta.put("productoFavoritos", productos); 

        } 

        return respuesta; 

    } 

} 

Repository — UserRepository.java 

El Repository extiende JpaRepository, lo que proporciona métodos CRUD: save(), findById(), findAll(), deleteById(), sin necesidad de escribir SQL. 

public interface UserRepository extends JpaRepository<User, Integer> { 

    // JpaRepository provee: save, findById, findAll, deleteById, etc. 

} 

 

3. Modelado de Datos y Entidades JPA 

 

Cada microservicio tiene su propia base de datos MySQL separada, con entidades JPA que mapean las tablas. Usamos anotaciones estándar de Jakarta Persistence para definir la estructura relacional. 

3.1 Entidad Principal: User (ms-user) 

La entidad User modela la tabla 'usuario' en db_user. Tiene una relación @OneToOne con Perfil y usa @ElementCollection para almacenar IDs de productos de otro microservicio. 

@Entity 

@Table(name="usuario") 

public class User { 

    @Id 

    @GeneratedValue(strategy = GenerationType.IDENTITY) 

    private Integer id; 

 

    @Column(nullable=false) 

    private String nombre; 

 

    @JsonManagedReference // Evita loops infinitos en serialización JSON 

    @OneToOne(mappedBy = "usuario", cascade = CascadeType.ALL) 

    private Perfil perfil; // Relación 1:1 con tabla 'perfil' 

 

    @ElementCollection 

    @CollectionTable(name="usuario_producto_ids", 

                     joinColumns = @JoinColumn(name="usuario_id")) 

    @Column(name="producto_id") 

    private List<Integer> productosIds; // IDs del ms-product 

} 

3.2 Migración de Base de Datos con Flyway (ms-user) 

El microservicio ms-user usa Flyway para gestionar las migraciones de base de datos de manera controlada y versionada. El archivo V1__CREATE_TABLE_USUARIO.sql crea las tablas con integridad referencial: 

-- V1__CREATE_TABLE_USUARIO.sql 

CREATE TABLE `usuario` ( 

  `id` INT NOT NULL AUTO_INCREMENT, 

  `nombre` VARCHAR(255) NOT NULL, 

  PRIMARY KEY (`id`) 

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4; 

 

CREATE TABLE `perfil` ( 

  `id` INT NOT NULL AUTO_INCREMENT, 

  `activo` TINYINT(1) NOT NULL, 

  `usuario_id` INT DEFAULT NULL, 

  PRIMARY KEY (`id`), 

  CONSTRAINT `fk_perfil_usuario` FOREIGN KEY (`usuario_id`) 

       REFERENCES `usuario` (`id`)  -- Integridad referencial 

) ENGINE=InnoDB; 

 

CREATE TABLE `usuario_producto_ids` ( 

  `usuario_id` INT NOT NULL, 

  `producto_id` INT DEFAULT NULL, 

  CONSTRAINT `fk_usuarioproducto_usuario` FOREIGN KEY (`usuario_id`) 

       REFERENCES `usuario` (`id`) 

) ENGINE=InnoDB; 

3.3 Resumen de Entidades por Microservicio 

Microservicio 

Entidad principal 

Tabla BD 

Relaciones JPA 

ms-user 

User + Perfil 

usuario, perfil 

@OneToOne(cascade=ALL) 

ms-product 

Productos 

productos 

@ElementCollection (categorias) 

ms-category 

Category 

category 

@ElementCollection (productos) 

ms-cart 

Cart 

cart, cart_producto_ids 

@ElementCollection (productos) 

ms-order 

Orders 

orders, user_order 

@ElementCollection (usuarios) 

ms-payment 

Payment 

payment, payment_user 

@ElementCollection (usuarios) 

ms-shipping 

Shipping 

shipping, shipping_orders 

@ElementCollection (ordenes) 

ms-inventario 

Inventory 

inventory, inventory_products 

@ElementCollection (productos) 

ms-sucursal 

Sucursal 

sucursal, sucursal_inventory 

@ElementCollection (inventarios) 

ms-review 

Review 

review 

Sin relaciones JPA (independiente) 

 

4. CRUD Completo y Endpoints REST 

 

Todos los microservicios implementan operaciones CRUD conectadas con JpaRepository. Todos los endpoints retornan JSON estructurado con ResponseEntity y códigos HTTP apropiados. 

4.1 Endpoints por Microservicio 

MS 

Método 

Ruta 

Descripción 

ms-user 

POST 

/api/v1/usuarios 

Crear usuario + perfil 

ms-user 

GET 

/api/v1/usuarios/{id} 

Usuario + productos remotos 

ms-product 

POST 

/api/v1/productos 

Crear producto 

ms-product 

GET 

/api/v1/productos 

Listar productos 

ms-product 

GET 

/api/v1/productos/{id} 

Buscar por ID 

ms-category 

POST 

/api/v1/category 

Crear categoría 

ms-category 

GET 

/api/v1/category 

Listar categorías 

ms-category 

DELETE 

/api/v1/category/{id} 

Eliminar categoría 

ms-cart 

GET 

/api/v1/cart 

Listar carrito 

ms-cart 

GET 

/api/v1/cart/usuario/{id} 

Carrito por usuario 

ms-cart 

DELETE 

/api/v1/cart/{id} 

Eliminar ítem 

ms-order 

POST 

/api/v1/ordenes 

Crear orden 

ms-order 

GET 

/api/v1/ordenes 

Listar órdenes 

ms-order 

GET 

/api/v1/ordenes/estado/{estado} 

Filtrar por estado 

ms-payment 

POST 

/api/v1/pagos 

Registrar pago 

ms-payment 

GET 

/api/v1/pagos/tipo/{tipo} 

Filtrar por tipo de pago 

ms-shipping 

GET 

/api/v1/envios/mesinicio/{mes} 

Filtrar por mes inicio 

ms-shipping 

GET 

/api/v1/envios/mesllegada/{mes} 

Filtrar por mes llegada 

ms-inventario 

GET 

/api/v1/inventarios 

Listar inventarios 

ms-sucursal 

GET 

/api/v1/sucursales/region/{r} 

Filtrar por región 

ms-sucursal 

GET 

/api/v1/sucursales/comuna/{c} 

Filtrar por comuna 

ms-review 

POST 

/api/v1/reviews 

Crear reseña (con validación) 

ms-review 

GET 

/api/v1/reviews/producto/{id} 

Reseñas por producto 

ms-review 

GET 

/api/v1/reviews/usuario/{id} 

Reseñas por usuario 

ms-product 

DELETE 

/api/v1/product/{id} 

Eliminar Producto 

ms-user 

DELETE 

/api/v1/user/{id} 

Eliminar usuario 

ms-order 

DELETE 

/api/v1/order/{id} 

Eliminar orden 

ms-payment 

DELETE 

/api/v1/payment/{id} 

Eliminar pago 

ms-shipping 

DELETE 

/api/v1/shipping/{id} 

Eliminar envio 

ms-review 

DELETE 

/api/v1/review/{id} 

Eliminar review 

 

5. Validaciones con Bean Validation y Manejo de Excepciones 

 

El microservicio ms-review implementa el ejemplo más completo de validaciones con Bean Validation (JSR 380) y manejo centralizado de excepciones con @RestControllerAdvice. 

5.1 Validaciones en la Entidad Review 

La entidad Review usa anotaciones de validación que garantizan la integridad de los datos antes de persistirlos: 

@Entity 

@Table(name = "review") 

public class Review { 

    @Column(name="usuario_id", nullable = false) 

    private Integer usuarioId; 

 

    @Column(name="producto_id", nullable = false) 

    private Integer productoId; 

 

    @NotBlank                    // El comentario no puede estar vacío 

    @Column(length = 1000) 

    private String comentario; 

 

    @Min(1)                      // Puntuación mínima: 1 

    @Max(5)                      // Puntuación máxima: 5 

    @Column(nullable = false) 

    private Integer puntuacion; 

} 

5.2 Activación de Validaciones en el Controller 

La anotación @Valid en el Controller activa automáticamente todas las validaciones Bean Validation antes de procesar la solicitud: 

@PostMapping 

public ResponseEntity<Review> guardarReview(@Valid @RequestBody Review review) { 

    Review guardada = service.saveReview(review); 

    return new ResponseEntity<>(guardada, HttpStatus.CREATED); 

} 

5.3 GlobalExceptionHandler con @RestControllerAdvice 

El GlobalExceptionHandler captura excepciones de toda la aplicación y retorna respuestas JSON estructuradas con el código HTTP apropiado: 

@RestControllerAdvice 

public class GlobalExceptionHandler { 

 

    // Captura errores de validación Bean Validation (@Valid) 

    @ExceptionHandler(MethodArgumentNotValidException.class) 

    public ResponseEntity<Map<String, String>> handleValidationErrors( 

            MethodArgumentNotValidException ex) { 

        Map<String, String> errores = new HashMap<>(); 

        ex.getBindingResult().getFieldErrors().forEach(error -> 

            errores.put(error.getField(), error.getDefaultMessage()) 

        ); 

        return new ResponseEntity<>(errores, HttpStatus.BAD_REQUEST); // 400 

    } 

 

    // Captura cualquier otra excepción no controlada 

    @ExceptionHandler(Exception.class) 

    public ResponseEntity<Map<String, String>> handleGenericError(Exception ex) { 

        Map<String, String> error = new HashMap<>(); 

        error.put("error", "Error interno del servidor"); 

        return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR); // 500 

    } 

} 

 

6. Comunicación entre Microservicios con Feign Client 

 

Los microservicios se comunican entre sí usando Feign Client, que permite hacer llamadas HTTP REST a otros microservicios como si fueran simples métodos Java. Tenemos 5 comunicaciones Feign implementadas en el proyecto. 

6.1 Mapa de Comunicaciones Feign 

Quien llama 

→ 

Quien responde 

Qué datos obtiene 

ms-user 

→ 

ms-product 

Obtiene productos favoritos del usuario (lista de ProductoDTO) 

ms-product 

→ 

ms-category 

Obtiene categoría del producto (CategoryDTO) 

ms-order 

→ 

ms-user 

Obtiene datos del usuario que hizo la orden (UserDTO) 

ms-payment 

→ 

ms-user 

Obtiene usuario asociado al pago (UserDTO) 

ms-sucursal 

→ 

ms-inventario 

Obtiene inventario de la sucursal (InventoryDTO) 

ms-cart 

→ 

ms-product 

Obtiene detalle de productos en el carrito (ProductoDTO) 

ms-category 

→ 

ms-product 

Obtiene productos de la categoría (ProductoDTO) 

ms-inventario 

→ 

ms-product 

Obtiene productos del inventario (ProductoDTO) 

ms-shipping 

→ 

ms-order 

Obtiene datos de la orden a enviar (OrderDTO) 

ms-review 

→ 

ms-product 

Valida que el producto existe (ProductDTO) 

 

6.2 Implementación Detallada: ms-user → ms-product 

Paso 1: Interfaz Feign Client 

Se define la interfaz ProductoFeignClient con la anotación @FeignClient, indicando el nombre del servicio y su URL: 

@FeignClient(name = "ms-producto", url = "localhost:8082") 

public interface ProductoFeignClient { 

    @GetMapping("/api/v1/productos/{id}") 

    ProductoDTO obtenerProductoPorId(@PathVariable("id") Integer id); 

} 

Paso 2: DTO de Transferencia 

El DTO (Data Transfer Object) es una clase simple que representa los datos que viajan entre microservicios, sin exponer la entidad JPA completa: 

// ProductoDTO.java — en ms-user 

public class ProductoDTO { 

    private Integer id; 

    private String nombre; 

    // getters / setters (o @Data de Lombok) 

} 

Paso 3: Uso en el Service 

El Service inyecta el Feign Client y lo usa como si fuera un método Java normal. Feign se encarga de hacer la llamada HTTP REST internamente: 

@Autowired 

private ProductoFeignClient productoClient; 

 

// Llamada remota: equivale a GET localhost:8082/api/v1/productos/3 

ProductoDTO producto = productoClient.obtenerProductoPorId(3); 

 

7. Logs Estructurados con SLF4J 

 

El microservicio ms-review implementa logs estructurados usando SLF4J, lo cual permite trazabilidad de las operaciones entre capas. 

7.1 Configuración del Logger 

Se declara el Logger como constante estática en la clase Service, con referencia a la clase actual para facilitar la identificación en los logs: 

import org.slf4j.Logger; 

import org.slf4j.LoggerFactory; 

 

@Service 

public class ReviewService { 

    private static final Logger log = LoggerFactory.getLogger(ReviewService.class); 

    // ... 

} 

7.2 Dónde Colocar los Logs (Puntos Estratégicos) 

Los logs deben ubicarse en puntos clave del flujo para permitir trazabilidad completa: 

public Review saveReview(Review review) { 

    log.info("[ReviewService] Intentando guardar review del usuario: {}", 

              review.getUsuarioId()); 

    Review guardada = repository.save(review); 

    log.info("[ReviewService] Review guardada exitosamente con ID: {}", 

              guardada.getId());  // Esto simula que Review tiene @Id 

    return guardada; 

} 

 

public List<Review> getReviewsByProducto(Integer productoId) { 

    log.info("[ReviewService] Buscando reviews del producto ID: {}", productoId); 

    List<Review> reviews = repository.findByProductoId(productoId); 

    log.info("[ReviewService] Se encontraron {} reviews", reviews.size()); 

    return reviews; 

} 

 

8. Configuración de application.properties 

 

Cada microservicio tiene su propio archivo application.properties con configuración independiente de base de datos, puerto y JPA. 

8.1 Ejemplo: ms-user (application.properties) 

# Nombre del microservicio (usado por Feign para identificarse) 

spring.application.name=ms-user 

 

# Puerto único para este microservicio 

server.port=8081 

 

# Conexión a su propia base de datos MySQL 

spring.datasource.url=jdbc:mysql://localhost:3306/db_user?createDatabaseIfNotExist=true 

spring.datasource.username=root 

spring.datasource.password= 

spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver 

 

# JPA/Hibernate: 'update' crea/actualiza tablas automáticamente 

spring.jpa.hibernate.ddl-auto=update 

spring.jpa.show-sql=true 

8.2 Ejemplo: ms-sucursal con Flyway 

spring.application.name=ms-sucursal 

server.port=8084 

 

spring.datasource.url=jdbc:mysql://localhost:3306/db_sucursal?createDatabaseIfNotExist=true 

 

# Con Flyway, usamos 'validate' porque Flyway crea las tablas 

spring.jpa.hibernate.ddl-auto=validate 

spring.jpa.show-sql=true 

 

# Flyway gestiona las migraciones versionadas 

spring.flyway.enabled=true 

spring.flyway.baseline-on-migrate=true 

8.3 Diferencia entre ddl-auto: update vs validate vs none 

Valor 

Comportamiento 

update 

Crea o modifica tablas automáticamente según las entidades JPA. Ideal para desarrollo. 

validate 

Solo valida que las tablas existen y coinciden con las entidades. No crea nada. Usado con Flyway. 

none 

No hace nada. Las tablas deben existir previamente (creadas por Flyway u otro medio). 

create 

Elimina y recrea todas las tablas al iniciar. Solo para pruebas, destruye datos. 

 

9. Preguntas Frecuentes de la Defensa 

 

Esta sección prepara respuestas claras y directas para las preguntas más probables durante la defensa técnica. Estúdialas bien, son el 70% de tu nota. 

9.1 Preguntas sobre Arquitectura y CSR 

P: ¿Por qué separamos en Controller, Service y Repository? 

R: Para aplicar el principio de separación de responsabilidades. El Controller solo maneja HTTP (rutas, parámetros, respuestas). El Service contiene la lógica de negocio y reglas. El Repository solo accede a la base de datos. Si mezclamos capas, el código es difícil de mantener, probar y escalar. 

 

P: ¿Qué pasaría si pongo lógica de negocio en el Controller? 

R: Violaríamos el principio de responsabilidad única (SRP). El Controller quedaría acoplado a la lógica, lo que dificulta la reutilización, las pruebas unitarias y el mantenimiento. Por eso en nuestro proyecto el Controller solo llama al Service y retorna ResponseEntity. 

 

P: ¿Qué es un DTO y por qué lo usamos? 

R: DTO significa Data Transfer Object. Es una clase simple que encapsula datos que viajan entre capas o entre microservicios, sin anotaciones JPA ni lógica de negocio. Lo usamos para no exponer las entidades JPA directamente al exterior, proteger la base de datos y desacoplar los microservicios entre sí. 

9.2 Preguntas sobre JPA y Base de Datos 

P: ¿Qué hace @GeneratedValue(strategy = GenerationType.IDENTITY)? 

R: Le dice a JPA que la clave primaria será generada automáticamente por la base de datos usando AUTO_INCREMENT de MySQL. Cuando insertamos un registro, MySQL asigna el siguiente ID disponible y JPA lo devuelve en el objeto guardado. 

 

P: ¿Qué es @ElementCollection? ¿Por qué lo usamos en lugar de @OneToMany? 

R: @ElementCollection crea una tabla intermedia para almacenar colecciones simples (como lista de enteros). Lo usamos para guardar IDs de entidades de OTRO microservicio, porque no podemos hacer un @OneToMany real entre bases de datos diferentes. Cada microservicio tiene su propia BD, así que solo guardamos el ID y luego consultamos el otro servicio con Feign. 

 

P: ¿Qué hace CascadeType.ALL en la relación @OneToOne de User con Perfil? 

R: CascadeType.ALL propaga todas las operaciones (guardar, actualizar, eliminar) del Usuario al Perfil. Cuando guardamos un User con su Perfil incluido, JPA automáticamente también guarda el Perfil en su tabla. Si eliminamos el User, también se elimina su Perfil. 

 

P: ¿Qué es @JsonManagedReference y por qué lo usamos? 

R: Evita el bucle infinito (StackOverflow) al serializar entidades con relaciones bidireccionales a JSON. Si User tiene Perfil y Perfil tiene User, al convertir a JSON se generaría un ciclo infinito. @JsonManagedReference en User y @JsonBackReference en Perfil rompe el ciclo, serializando el lado 'managed' e ignorando el 'back'. 

9.3 Preguntas sobre Feign y Comunicación 

P: ¿Cómo funciona Feign Client internamente? 

R: Feign es una librería de Spring Cloud que genera una implementación automática de la interfaz en tiempo de ejecución. Cuando llamas a productoClient.obtenerProductoPorId(3), Feign construye una petición HTTP GET a localhost:8082/api/v1/productos/3, la ejecuta, recibe el JSON de respuesta y lo deserializa en un objeto ProductoDTO. Todo sin que tengamos que escribir código HTTP manualmente. 

 

P: ¿Qué pasaría si ms-product está caído cuando ms-user lo llama? 

R: Feign lanzaría una excepción (FeignException o HystrixException dependiendo de la configuración). Sin un manejador de errores, la petición fallaría con 500. La solución ideal es implementar un Circuit Breaker (con Resilience4j) o al menos capturar la excepción en el Service con try/catch y retornar una respuesta parcial o un mensaje de error controlado. 

9.4 Preguntas sobre Validaciones y Excepciones 

P: ¿Qué sucede exactamente cuando envías una review con puntuación = 0? 

R: Spring activa el proceso de validación Bean Validation porque el endpoint tiene @Valid. La anotación @Min(1) en el campo puntuacion falla. Spring lanza automáticamente MethodArgumentNotValidException. Nuestro GlobalExceptionHandler la captura con @ExceptionHandler, construye un Map con el campo y el mensaje de error, y retorna HTTP 400 BAD REQUEST con un JSON como: {"puntuacion": "must be greater than or equal to 1"}. 

 

P: ¿Cuál es la diferencia entre @ControllerAdvice y @RestControllerAdvice? 

R: @RestControllerAdvice es una combinación de @ControllerAdvice + @ResponseBody. Significa que los métodos del handler retornan directamente objetos que se serializan a JSON (como si tuvieran @ResponseBody). Usamos @RestControllerAdvice porque nuestros servicios son REST APIs y siempre retornamos JSON. 

9.5 Preguntas sobre Git y Colaboración 

P: ¿Cómo distribuiste los commits en el equipo? 

R: Cada integrante trabajó en sus microservicios asignados y realizó commits frecuentes con mensajes técnicos descriptivos, como: 'feat(ms-review): add GlobalExceptionHandler with Bean Validation', 'feat(ms-user): implement Feign Client for ms-product communication', 'fix(ms-payment): correct Flyway migration script for payment table'. 

 

P: ¿Cómo usaron Trello u otra herramienta de gestión? 

R: Organizamos el tablero en columnas: Backlog, En progreso, En revisión y Completado. Cada microservicio fue una tarea asignada a un integrante, con subtareas para: entidad JPA, repositorio, service, controller, endpoints y pruebas Postman. 

 

10. Guía de Ejecución en la Defensa 

 

Esta sección es un checklist paso a paso para ejecutar correctamente el proyecto durante la defensa. Memoriza este flujo para no cometer errores bajo presión. 

10.1 Requisitos Previos 

Laragon corriendo con MySQL en puerto 3306 

Java 21+ instalado 

Maven o el wrapper mvnw disponible 

IDE (IntelliJ IDEA o VS Code) con Spring Boot configurado 

Postman listo con las colecciones de prueba 

10.2 Orden de Inicio de Microservicios 

IMPORTANTE: Iniciar primero los microservicios que no dependen de otros (los que no consumen Feign), luego los que sí consumen: 

1° ms-product (8082) — es dependencia de varios otros 

2° ms-category (8082) — puede tener conflicto de puerto con ms-product, revisar 

3° ms-inventario (8081) — dependencia de ms-sucursal 

4° ms-user (8081) — revisar puerto, puede conflictuar 

5° ms-order (8085) 

6° ms-sucursal (8084) — consume ms-inventario 

7° ms-payment (8082) 

8° ms-shipping (8081) 

9° ms-cart (8081) 

10° ms-review — consume ms-product 

10.3 Pruebas con Postman — Flujo Recomendado 

Flujo 1: Crear y obtener usuario con productos remotos 

POST http://localhost:8081/api/v1/usuarios 

{ 

  "nombre": "Juan Pérez", 

  "perfil": { "activo": true }, 

  "productosIds": [1, 2] 

} 

 

GET http://localhost:8081/api/v1/usuarios/1 

// Retorna usuario + perfil + detalle completo de productos del ms-product 

Flujo 2: Validaciones en ms-review 

POST http://localhost:[puerto]/api/v1/reviews 

{ 

  "usuarioId": 1, 

  "productoId": 1, 

  "comentario": "", 

  "puntuacion": 8 

} 

// Respuesta esperada: 400 BAD REQUEST 

// { "comentario": "must not be blank", "puntuacion": "must be <= 5" } 

Flujo 3: Filtros personalizados 

GET http://localhost:8085/api/v1/ordenes/estado/PENDIENTE 

GET http://localhost:[puerto]/api/v1/pagos/tipo/CREDITO 

GET http://localhost:[puerto]/api/v1/envios/mesinicio/Enero 

GET http://localhost:8084/api/v1/sucursales/region/Metropolitana 

 

11. Modificaciones de Código en Vivo 

 

 

11.1 Agregar un Nuevo Endpoint 

En ReviewRepository: 

List<Review> findByPuntuacion(Integer puntuacion); 

En ReviewService: 

public List<Review> getReviewsByPuntuacion(Integer puntuacion) { 

    log.info("Buscando reviews con puntuacion: {}", puntuacion); 

    return repository.findByPuntuacion(puntuacion); 

} 

En ReviewController: 

  

@GetMapping("/puntuacion/{puntuacion}")  

  

public ResponseEntity<List<Review>> buscarPorPuntuacion(  

  

        @PathVariable Integer puntuacion) {  

  

    List<Review> reviews = service.getReviewsByPuntuacion(puntuacion);  

  

    return ResponseEntity.ok(reviews);  

  

} 

11.2 Agregar una Validación Nueva 

@NotBlank  

  

@Size(min = 10, message = "El comentario debe tener al menos 10 caracteres")  

  

@Column(length = 1000)  

  

private String comentario; 

 

11.3 Agregar un Log Estratégico 

public void deleteReview(Integer id) {  

  

    log.warn("[ReviewService] Eliminando review con ID: {}", id);  

  

    repository.deleteById(id);  

  

    log.info("[ReviewService] Review ID: {} eliminada exitosamente", id);  

  

} 

11.4 Corregir un Código HTTP Incorrecto 

// ANTES (incorrecto):  

  

return ResponseEntity.ok(nueva);  // 200 OK  

  

  

  

// DESPUÉS (correcto):  

  

return new ResponseEntity<>(nueva, HttpStatus.CREATED);  // 201 Created  

  

// O equivalente:  

  

return ResponseEntity.status(201).body(nueva) 

 

 

// ANTES (incorrecto): 

return ResponseEntity.ok(nueva);  // 200 OK 

 

// DESPUÉS (correcto): 

return new ResponseEntity<>(nueva, HttpStatus.CREATED);  // 201 Created 

// O equivalente: 

return ResponseEntity.status(201).body(nueva) 
