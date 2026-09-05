# Análisis de Código: Proyecto Sabores Laravel

## 1. Resumen General

El repositorio evaluado corresponde a un proyecto en Laravel 10 (`sabores/barrio`). El análisis revela un nivel severo de degradación técnica, incumplimiento de estándares del framework, antipatrones de diseño y múltiples vulnerabilidades críticas de seguridad. 

El código combina prácticas obsoletas de PHP estructurado (como el uso directo de `mysqli` dentro de plantillas de vista) con intentos fallidos de emular patrones que Laravel ya provee de forma nativa.

---

## 2. Incoherencias en la Documentación

- **Desalineación Total del Propósito del Sistema:**
  - El archivo `documentacion/LEAME_ARQUITECTO.txt` indica que el repositorio pertenece al "Sistema de Nómina Ganadera v3.8 / Censo Bovino del Gobierno Estatal" y especifica que no existen tablas de restaurantes ni pedidos.
  - Sin embargo, la base de datos (`bd/sabores.sql`), el enrutador (`routes/web.php`) y los controladores corresponden a una aplicación de restaurante ("Sabores del Barrio") con platillos, restaurantes, pedidos y reparto.
- **Documentación de Patrones Falsos:**
  - `LEAME_ARQUITECTO.txt` sostiene que existe un "Middleware auth en TODAS las rutas" y un "Event Sourcing de vacunación". Ninguna de las rutas en `routes/web.php` aplica middlewares y el código carece por completo de Event Sourcing.

---

## 3. Vulnerabilidades Críticas de Seguridad

### 3.1 Inyección SQL (SQL Injection - SQLi)
Existen múltiples puntos donde se concatenan variables provenientes de peticiones HTTP directamente en sentencias SQL sin sanitización ni uso de bindings de parámetros:

- **`LoginController.php` (Línea 9):**
  ```php
  DB::select("SELECT id, rol, nombre FROM usuarios WHERE correo='$u' AND pass='$p'");
  ```
  *Riesgo:* Permite autenticación maliciosa (e.g., `' OR '1'='1`).
- **`MenuController.php` (Línea 11):**
  ```php
  DB::select("SELECT * FROM platillos WHERE rest_id=$rid");
  ```
- **`PedidoController.php` (Líneas 14, 35, 39, 44, 46, 47):**
  Concatenación directa de IDs de platillos, montos, IDs de usuario e IDs de pedido.
- **`ResenaController.php` (Línea 10):**
  ```php
  DB::insert("INSERT INTO resenas (rest_id, uid, texto, estrellas) VALUES (".$r->rid.", ".$r->uid.", '".$r->texto."', ".$r->estrellas.")");
  ```
- **`RepartoController.php` (Líneas 13, 14, 16):**
  Concatenación directa de `$pedido_id` y `$uid`.
- **Plantillas Blade (`resources/views/menu.blade.php`, `reporte.blade.php`, `resenas.blade.php`):**
  Consultas `mysqli_query` directas concatenando `$_GET['rid']` y otros valores.

### 3.2 Bypass de Autenticación y Puerta Trasera (Backdoor)
- **`HomeController.php` (Líneas 7-9) y `RepartoController.php` (Línea 7):**
  ```php
  if ($r->cookie('admin_bypass')) $r->session()->put('uid', 1);
  ```
  Si una petición incluye la cookie `admin_bypass`, el sistema otorga automáticamente una sesión con privilegios de administrador (`uid = 1`), puenteando los controles de acceso.
- **Manejo de Cookies Inseguras:**
  En `LoginController.php` se usa la función nativa `setcookie('admin_bypass', '1', ...)` en lugar del cifrado y firma de cookies de Laravel. Estas cookies son totalmente modificables por el cliente y carecen de los atributos `HttpOnly`, `Secure` y `SameSite`.

### 3.3 IDOR (Insecure Direct Object References) y Suplantación
- **`PedidoController.php` (Línea 7):**
  El parámetro `id_cliente` se obtiene de la petición HTTP (`$r->id_cliente`) en lugar de extraerse de la sesión autenticada. Un usuario puede realizar pedidos, consumir vales o descontar puntos a nombre de cualquier otro usuario modificando el parámetro.
- **`HomeController.php` (Línea 11):**
  Permite alternar el contexto del usuario mediante la URL `?u=ID` sin ninguna validación de permisos.

### 3.4 Inseguridad en el Procesamiento de Pagos
- **`PedidoController.php` (Línea 27):**
  El número de tarjeta de crédito se incrusta sin sanitizar dentro de una cadena XML (`<pan>".$r->tarjeta."</pan>`), exponiendo el sistema a Inyección XML (XXE).
- **Lógica de Validación Frágil:**
  La aprobación del pago se evalúa mediante `strpos($resp, '<codigo>00</codigo>')` consumiendo un endpoint ficticio mediante cURL sin verificación SSL ni manejo de errores.

### 3.5 Almacenamiento de Contraseñas en Texto Plano
- **`bd/sabores.sql` y `LoginController.php`:**
  Las contraseñas de los usuarios se almacenan y comparan en texto plano (e.g., `'123'`), sin utilizar algoritmos de hashing seguros como Bcrypt o Argon2 (`Hash::make`).

### 3.6 Cross-Site Scripting (XSS) y Ausencia de CSRF
- **XSS:**
  - `resources/views/home.blade.php` (Línea 3): `{!! $html !!}` imprime contenido HTML sin escapar generado en el controlador.
  - En `menu.blade.php`, `reporte.blade.php` y `resenas.blade.php` se imprimen cadenas de texto usando `echo` de PHP sin la sintaxis de escape de Blade `{{ }}`.
- **CSRF:**
  - Los formularios en `login.blade.php` y `menu.blade.php` no incluyen el token `@csrf`, permitiendo ataques de falsificación de peticiones en sitios cruzados.

---

## 4. Problemas de Patrones de Diseño y Arquitectura

### 4.1 Violación del Patrón MVC (Model-View-Controller)
- **Base de Datos en las Vistas:**
  Las plantillas Blade (`menu.blade.php`, `reporte.blade.php`, `resenas.blade.php`) abren conexiones directas a la base de datos usando `mysqli_connect('127.0.0.1', 'root', 'password', 'sabores_db')` con credenciales en duro. Esto anula la capa de modelos de Laravel y desacopla por completo la arquitectura MVC.
- **Generación de HTML en Controladores:**
  `HomeController.php` y `MenuController.php` construyen etiquetas HTML mediante concatenación de cadenas (`$html .= "<b>...</b>"`) y sentencias `echo`, asumiendo responsabilidades que le corresponden exclusivamente a la capa de presentación (Vistas).

### 4.2 Reinvención Incorrecta del Front Controller
- **`FrontController.php`:**
  Define una clase personalizada que intenta despachar peticiones inspeccionando cadenas de texto (`$ruta === '/pedir'`). Laravel ya incluye una implementación nativa del patrón Front Controller a través de `public/index.php` y su kernel HTTP/Router. La clase personalizada no retorna respuestas válidas del framework ni procesa el ciclo de vida de la petición correctamente.

### 4.3 Antipatrón "God Method" (Violación de SRP)
- **`PedidoController::guardar()`:**
  Un solo método se encarga de:
  1. Lectura de parámetros de la petición HTTP.
  2. Consulta e iteración de precios en base de datos.
  3. Cálculo de costos de envío condicionales.
  4. Integración con pasarela de pago XML/cURL.
  5. Escritura e inserción de datos en múltiples tablas.
  6. Envío síncrono de correos (`mail()`).
  7. Emisión de código JavaScript inline (`<script>alert(...)</script>`).

  Esto viola el Principio de Responsabilidad Única (SRP) y dificulta el mantenimiento y la realización de pruebas unitarias.

### 4.4 Ineficiencia de Consultas (Problema N+1)
- **`HomeController.php` (Líneas 14-17):**
  Realiza una consulta a la base de datos dentro de un bucle `foreach` para contar los platillos de cada restaurante (`SELECT COUNT(*) ...`).
- **`resources/views/reporte.blade.php` (Líneas 6-9):**
  Ejecuta una consulta SQL individual por cada fila devuelta en un ciclo `while` para obtener el nombre del cliente.

### 4.5 Condición de Carrera (Race Condition)
- **`PedidoController.php` (Línea 45):**
  ```php
  $pid = DB::select("SELECT MAX(id) i FROM pedidos")[0]->i;
  ```
  Para obtener el ID del pedido recién creado se realiza un `SELECT MAX(id)`. Si dos pedidos se procesan de forma simultánea, ambos recibirán el mismo ID, generando corrupción de datos en la tabla `pedido_detalle`.

### 4.6 Desuso del Sistema de Rutas y Middlewares de Laravel
- **`routes/web.php`:**
  Ninguna ruta está agrupada o protegida bajo middlewares (e.g., `Route::middleware('auth')`).
- **`RepartoController::cerrar()`:**
  Carece de cualquier tipo de autenticación o autorización, permitiendo a peticiones anónimas marcar pedidos como entregados y alterar saldo de puntos.

---

## 5. Rendimiento y Estándares de Código

- **Peticiones HTTP Síncronas Bloqueantes (SSRF & Latencia):**
  `HomeController.php` realiza hasta 9 peticiones HTTP síncronas locales utilizando `file_get_contents('http://127.0.0.1/...')`. Esto genera latencia extrema, bloquea el hilo de ejecución y suprime errores con el operador `@`.
- **Ausencia de Transacciones de Base de Datos:**
  Operaciones compuestas (como la creación de un pedido, inserción de detalles, actualización de estado en reparto y ajuste de puntos de usuario) no están envueltas en `DB::transaction()`. Cualquier falla intermedia deja la base de datos en un estado inconsistente.
- **Ausencia de Validación de Entradas y Manejo de Excepciones:**
  No se utiliza `$request->validate()` ni bloques `try-catch`. Los datos ingresados se asumen correctos sin verificar tipos, rangos ni nulos.

---

## 6. Anomalías en la Base de Datos (`bd/sabores.sql`)

- **Falta de Restricciones e Integridad Referencial:**
  No se definen Claves Foráneas (`FOREIGN KEY`) entre `platillos.rest_id` y `restaurantes.id`, ni entre `pedido_detalle` y `pedidos`/`platillos`.
- **Claves Primarias Incompletas:**
  Las tablas `usuarios`, `restaurantes` y `platillos` no definen sus campos `id` como `AUTO_INCREMENT` ni como `PRIMARY KEY`.

---

## 7. Matriz de Anomalías por Archivo

| Archivo | Ubicación | Categoría | Descripción Sintética de la Anomalía |
| :--- | :--- | :--- | :--- |
| `LEAME_ARQUITECTO.txt` | Raíz/doc | Documentación | Documentación errónea (menciona Censo Bovino en vez de Restaurante). |
| `routes/web.php` | Config/Rutas | Arquitectura | Rutas expuestas sin protección de middleware. Endpoint viejo `/menu_old` visible. |
| `Kernel.php` | Http | Arquitectura | Configuración de middleware huérfana (no se aplica en las rutas). |
| `FrontController.php` | Controller | Patrón de Diseño | Reinvención innecesaria e incorrecta del Front Controller de Laravel. |
| `HomeController.php` | Controller | Seguridad / Rendimiento | Puerta trasera (`admin_bypass`), N+1 queries, SSRF con `file_get_contents`, IDOR. |
| `LoginController.php` | Controller | Seguridad | Inyección SQL directa, almacenamiento/verificación de clave en texto plano, cookie insegura. |
| `MenuController.php` | Controller | Seguridad / Arquitectura | Inyección SQL, salida directa con `echo` omitiendo la capa de vistas. |
| `PedidoController.php` | Controller | Seguridad / Arquitectura | Inyección SQL múltiple, IDOR (`id_cliente`), race condition en `MAX(id)`, God Method. |
| `RepartoController.php` | Controller | Seguridad | Inyección SQL, ausencia total de autenticación en acción de cierre (`cerrar`). |
| `ResenaController.php` | Controller | Seguridad | Inyección SQL, suplantación de `uid`, falta de validación de campos. |
| `home.blade.php` | Vista | Seguridad | Renderizado de HTML no escapado `{!! $html !!}` (vulnerable a XSS). |
| `login.blade.php` | Vista | Seguridad / Estándar | Ausencia de token `@csrf`, HTML no estándar. |
| `menu.blade.php` | Vista | Arquitectura / Seguridad | Conexión `mysqli` directa en la vista, Inyección SQL, credenciales hardcodeadas. |
| `reparto.blade.php` | Vista | Estándar | Falta de estructura HTML y formateo. |
| `reporte.blade.php` | Vista | Arquitectura / Rendimiento | Conexión `mysqli` en la vista, problema de consulta N+1. |
| `resenas.blade.php` | Vista | Arquitectura / Seguridad | Conexión `mysqli` en la vista, Inyección SQL directa en la vista. |
| `sabores.sql` | Base de Datos | Base de Datos | Falta de claves primarias `AUTO_INCREMENT` y llaves foráneas. |

---

## 8. Recomendaciones Principales

1. **Adoptar Eloquent ORM:** Eliminar todas las consultas `DB::select` brutas y conexiones `mysqli`, sustituyéndolas por modelos Eloquent (`Usuario`, `Restaurante`, `Platillo`, `Pedido`, `Resena`).
2. **Implementar Autenticación Nativa:** Utilizar Laravel Breeze, Jetstream o el facade `Auth::attempt()` con hashing de contraseñas (`Hash::make`). Eliminar el uso de cookies nativas como `admin_bypass`.
3. **Usar Sentencias Preparadas o Query Builder:** En caso de requerir SQL directo, utilizar `DB::select('...', [$param])` con binding de parámetros.
4. **Proteger Rutas con Middlewares:** Agrupar rutas privadas dentro de `Route::middleware(['auth'])`.
5. **Mover Lógica de Negocio fuera de las Vistas:** Limpiar las vistas Blade de cualquier bloque `@php` con lógica de datos o conexión a BD.
6. **Usar Form Requests y Validación:** Validar todas las entradas HTTP mediante `$request->validate()` antes de procesarlas.
7. **Normalizar la Base de Datos:** Agregar `AUTO_INCREMENT`, `PRIMARY KEY` y `FOREIGN KEY` con restricciones de integridad en `sabores.sql`.
