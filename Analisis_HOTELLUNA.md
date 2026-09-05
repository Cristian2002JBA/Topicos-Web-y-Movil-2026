# Reporte de Arquitectura y Diagnóstico - Hotel Luna

Proyecto: hotelluna_spring  
Fecha: Septiembre 2026

---

## Misión 1 — El portal no es un patrón

| # | Problema en el código | Capa | Patrón que corresponde | Por qué este patrón y no el más cercano | Cuándo NO aplicaría |
|---|---|---|---|---|---|
| 1 | El cálculo de tarifas y adicionales se hace con muchos `if-else if` en el controlador (`ReservaCtrl.java`). | Aplicación / Dominio | Strategy | Es Strategy porque define una familia de algoritmos de cálculo intercambiables según el tipo de cuarto y tarifa. No es Chain of Responsibility porque no se pasa la petición a través de varios manejadores en cadena. | No aplicaría si el cálculo fuera una tarifa fija sin variaciones de negocio. |
| 2 | El controlador envía directamente peticiones HTTP/XML manuales al banco (`https://banco-luna.local/pay`). | Integración | Adapter (o Gateway) | Es Adapter porque traduce la interfaz y formato XML del servicio externo del banco a lo que necesita nuestra aplicación. No es Facade porque Facade simplifica un subsistema interno propio, no integra una API externa de terceros. | No aplicaría si el banco ya nos diera un cliente SDK en Java listo para usar. |
| 3 | Los controladores ejecutan consultas SQL directas con `JdbcTemplate` sin clases de repositorio. | Datos / Persistencia | Repository | Es Repository porque abstrae el acceso a la base de datos como si fuera una colección en memoria. No es Active Record porque Active Record mezcla la entidad de negocio con los métodos SQL en la misma clase. | No aplicaría en scripts simples de migración o en aplicaciones sin modelo de dominio. |
| 4 | Las operaciones de BD en la reserva se ejecutan una por una sin control de transacciones. | Dominio / Aplicación | Unit of Work | Es Unit of Work (como `@Transactional` en Spring) porque mantiene el control de qué cambios deben guardarse o deshacerse juntos en la transacción. No es Observer porque Observer sirve para notificar eventos desacoplados, no para agrupar operaciones atómicas en BD. | No aplicaría en operaciones de solo lectura (GET) o cuando no se requiere consistencia atómica relacional. |
| 5 | Se creó un servlet manual (`FrontControllerServlet`) con `@WebServlet("/*")` e interfaz `FiltroCasero`. | Presentación / Políticas Transversales | Intercepting Filter | El Front Controller centraliza el despacho (en Spring es el `DispatcherServlet`), mientras que Intercepting Filter maneja la cadena de filtros como auth y logs. Crear un servlet frontal casero rompe el ruteo de Spring. | No aplica crear un Front Controller propio cuando se usa un framework moderno como Spring, Laravel o Express, ya que el framework ya incluye el suyo. |
| 6 | Se genera código HTML pegando cadenas en Java y se mete a Thymeleaf con `th:utext` (`FacturaSql.java`). | Presentación | View Helper | Es un mal uso de View Helper. La lógica de presentación debe entregar datos a la vista para que la plantilla los renderice. No es ViewModel porque un ViewModel solo lleva datos (DTO), no genera HTML crudo dentro del código Java. | No aplicaría en APIs REST donde solo se retorna JSON. |

---

## Misión 2 — Una petición, varios patrones

Pasos que sigue la petición `POST /reservar` desde que el usuario hace clic hasta que se guarda y notifica:

1. **Cliente / Navegador (Petición HTTP POST)**  
   Mecanismo: Formulario HTML.  
   Patrón: Remote Procedure Call / REST Request. Manda los parámetros (`huesped`, `tipo`, `tarifa`, `pago`, `correo`).

2. **Cadena de Filtros del Servidor (Filter Chain)**  
   Mecanismo: `SecurityFilterChain` o filtros Servlet.  
   Patrón: Intercepting Filter. Revisa aspectos transversales como validar el token CSRF, cookies y caducidad de sesión antes de dejar pasar la petición.

3. **Front Controller Central (`DispatcherServlet`)**  
   Mecanismo: `DispatcherServlet` de Spring MVC.  
   Patrón: Front Controller. Recibe todas las peticiones HTTP de la aplicación y las centraliza para su procesamiento.

4. **Mapeador de Peticiones (`HandlerMapping`)**  
   Mecanismo: Anotación `@PostMapping("/reservar")`.  
   Patrón: Router / Command Dispatcher. Busca qué controlador y método debe atender la URL `/reservar`.

5. **Controlador Web (`ReservaCtrl`)**  
   Mecanismo: `@Controller` en Spring.  
   Patrón: Page Controller. Recibe los parámetros de la vista, los valida formalmente y los pasa a la capa de servicios.

6. **Servicio de Aplicación (`ReservaService`)**  
   Mecanismo: Clase marcada con `@Service` y `@Transactional`.  
   Patrón: Application Service y Unit of Work. Inicia la transacción y coordina la lógica de negocio, pagos y almacenamiento.

7. **Calculador de Tarifas (`CalculadorTarifa`)**  
   Mecanismo: Interfaz y clases por cada tarifa.  
   Patrón: Strategy. Calcula el precio según la combinación seleccionada sin usar múltiples `if`.

8. **Adaptador de Pago (`PasarelaPagoAdapter`)**  
   Mecanismo: Cliente HTTP de integración.  
   Patrón: Adapter / Gateway. Construye la llamada al banco, le envía los datos y procesa la respuesta.

9. **Repositorio de Persistencia (`ReservaRepository`)**  
   Mecanismo: Interfaz de repositorio (`JpaRepository`).  
   Patrón: Repository. Guarda la entidad de la reserva y actualiza la habitación en la base de datos en la misma transacción.

10. **Servicio de Notificación (`EmailService`)**  
    Mecanismo: `JavaMailSender` o publicador de eventos.  
    Patrón: Observer / Domain Event. Emite un evento para enviar el correo de confirmación de forma asíncrona sin frenar la respuesta al usuario.

---

## Misión 3 — Políticas transversales

1. **¿Cuándo corresponde Front Controller y por qué?**  
   Corresponde en la arquitectura del framework web para tener un único punto de entrada que reciba todas las peticiones, maneje excepciones globales y reparta el trabajo a los controladores. En Spring Boot esto ya lo hace `DispatcherServlet`, por lo que hacer otro servlet casero con `@WebServlet("/*")` causa conflictos y rompe el proyecto.

2. **¿Qué mecanismos ofrecen los frameworks actuales?**  
   - Spring Boot: `DispatcherServlet` junto con `OncePerRequestFilter` y `SecurityFilterChain`.  
   - Laravel: El `HTTP Kernel` y su pila de `Middlewares`.  
   - Express: La pila de middlewares (`app.use`).

3. **Orden de la cadena de filtros:**  
   1. Bitácora / Logging (para registrar la IP y la petición que entra).  
   2. Autenticación y Autorización (para saber quién es el usuario y si tiene permiso).  
   3. Verificación de cupo / capacidad (para checar si hay disponibilidad antes de hacer operaciones pesadas).  
   4. Caso de uso (procesar la reserva y el cobro).

   *¿Por qué autenticar después del cobro es un error de diseño?*  
   Porque estarías ejecutando una transacción financiera sin saber quién la hace. Esto genera problemas graves de seguridad: cualquiera podría cobrar a tarjetas sin estar logueado, habría fraudes y no habría forma de saber qué usuario hizo la operación. Las acciones con impacto de negocio siempre van después de verificar la identidad.

4. **¿Qué NO debe hacer esta cadena de filtros?**  
   No debe meterse con la lógica de negocio. No debe calcular precios, no debe modificar inventario ni habitaciones en la base de datos, ni hacer cobros o enviar correos. Su única función es filtrar y validar cuestiones técnicas de la petición HTTP (seguridad, tokens, encabezados).

---

## Verificación de funcionalidad

Aunque el proyecto compila correctamente con Maven (`mvn clean compile`), al momento de ejecutarlo **no funciona correctamente por varios problemas**:

- **El servlet casero bloquea todo:** `FrontControllerServlet` tiene la anotación `@WebServlet("/*")`, por lo que atrapa todas las URLs y responde sólo el texto `"despacho casero"`, impidiendo que los controladores de Spring respondan.
- **Inyección SQL y vulnerabilidades:** `LoginCtrl.java` concatena las credenciales directamente en la consulta SQL. Además, en `HomeCtrl.java` si se envía la cookie `admin_bypass`, entra como administrador sin contraseña.
- **Llamadas HTTP rotas:** `ReservaCtrl` intenta conectarse a `https://banco-luna.local/pay` que no existe, marcando error al reservar. `HomeCtrl` hace 8 peticiones en bucle a `http://127.0.0.1/mod/i` que trancan la carga de la página.
- **Consultas SQL en la vista:** `FacturaSql.java` hace una consulta SQL por cada renglón dentro de un bucle y genera HTML pegado con strings que se mete a la plantilla Thymeleaf mediante `th:utext`.

---

## Nota final

**Recomendación:** Es mejor rehacerlo desde cero.

**Por qué:**  
El código actual no tiene capas bien definidas (no hay entidades JPA, repositorios ni servicios). Toda la lógica está revuelta en los controladores con consultas SQL directas y parches que rompen las funciones de Spring Boot. 

Arreglarlo tomaría más tiempo tratando de quitar los anti-patrones que volver a crear el proyecto desde cero usando la estructura limpia de Spring Boot (`Spring Data JPA`, `Spring Security`, `@Service` y `@Controller`).
