# 🏛️ Arquitectura de Software y Patrones de Diseño — Resumen Ejecutivo
> **Proyecto Cinerex** | *Resolución sintética de misiones basada en Clean Architecture, Patrones GoF y PoEAA (Martin Fowler).*

---

## 🎯 Misión 1 — El portal no es un patrón

| # | Problema identificado | Capa | Patrón elegido | Diferenciador clave frente al patrón vecino | Cuándo NO aplicarlo |
|---|---|---|---|---|---|
| **1** | Pasarela bancaria acoplada con HTTP crudo y XML manual. | **Integración** | **Adapter** | **Adapter vs. Facade:** Adapter traduce e interconecta interfaces incompatibles preexistentes; Facade crea una interfaz simplificada sobre subsistemas propios sin alterar contratos. | Si el servicio externo ya ofrece un SDK nativo con la interfaz requerida por el dominio. |
| **2** | Verificaciones manuales de sesión/permisos dispersas por vistas. | **Políticas Transversales** | **Intercepting Filter** *(Middleware)* | **Filter vs. Decorator:** Filter opera secuencialmente sobre el ciclo de vida de la petición HTTP global (pudiendo abortar la cadena); Decorator extiende instancias individuales de negocio conservando su contrato. | Si se intenta usar para validar reglas de negocio o del reglamento (ej. cupos o balances de cuenta). |
| **3** | Cálculo de tarifas resuelto con cadenas `if/elif` y valores mágicos. | **Aplicación / Dominio** | **Strategy** | **Strategy vs. State:** Strategy intercambia algoritmos de cálculo independientes del ciclo de vida; State muta el comportamiento según transiciones de estado interno finito. | Si las tarifas son fijas, uniformes o no presentan variaciones a futuro. |
| **4** | Mutaciones concurrentes (asientos, puntos, boletos) sin transaccionalidad. | **Aplicación / Dominio** | **Unit of Work** | **Unit of Work vs. Observer:** Unit of Work rastrea entidades para un commit atómico ACID coordinado; Observer solo reacciona a eventos para desencadenar efectos secundarios desacoplados. | En operaciones de solo lectura o transacciones simples de una sola tabla. |
| **5** | Consultas SQL concatenadas y dispersas en vistas/plantillas. | **Datos** | **Repository** *(con ORM / Data Mapper)* | **Repository vs. DAO:** DAO es CRUD centrado en tablas de BD; Repository simula una colección de objetos de negocio en memoria en lenguaje de dominio. | En consultas de reportes masivos de alta concurrencia donde mapear entidades degrada rendimiento frente a proyecciones planas. |
| **6** | Concatenación de HTML y SQL dentro de funciones y tags. | **Presentación** | **Template View** / **View Helper** | **Template View vs. Transform View:** Template View inserta datos en plantillas HTML estructuradas; Transform View aplica hojas de transformación externas estilo XSLT. | En APIs REST/GraphQL desacopladas donde el cliente (frontend React/Vue/móvil) dibuja la interfaz. |

---

## 🚀 Misión 2 — Una petición, varios patrones

### Flujo del Caso de Uso: «Pagar inscripción / boleto»

1. **Cliente / Navegador (POST)** $\to$ **Command / DTO**: Transporta la intención y datos en una estructura inmutable sin lógica de negocio.
2. **Servidor y Enrutador** $\to$ **Front Controller**: Punto único de entrada al sistema que normaliza la solicitud y resuelve el enrutamiento (`urls.py` / `DispatcherServlet`).
3. **Pipeline HTTP** $\to$ **Intercepting Filter**: Aplica políticas técnicas obligatorias (CSRF, autenticación, rate limit y bitácora de entrada).
4. **Controlador Web** $\to$ **Controller (MVC)**: Extrae datos, valida la sintaxis de entrada y delega al caso de uso.
5. **Servicio de Aplicación** $\to$ **Service Layer**: Orquesta la transacción de negocio y coordina el flujo entre el dominio, pagos y persistencia.
6. **Entidad del Dominio** $\to$ **Domain Model + Strategy**: Valida invariantes del negocio y aplica la estrategia algorítmica para calcular la tarifa exacta.
7. **Pasarela Externa** $\to$ **Adapter**: Traduce la llamada del dominio al formato propietario del banco (XML/REST) y normaliza la respuesta.
8. **Persistencia** $\to$ **Repository + Unit of Work**: Mapea el estado de las entidades y consolida los cambios con commit atómico ACID.
9. **Eventos y Notificaciones** $\to$ **Domain Events / Observer (Pub/Sub)**: Publica el evento de pago para enviar confirmaciones por correo o facturación de forma asíncrona y desacoplada.
10. **Respuesta** $\to$ **Template View / View**: Renderiza la confirmación visual o emite una redirección HTTP 303 (Post/Redirect/Get).

---

## 🛡️ Misión 3 — Políticas transversales

### 1. Cuándo corresponde Front Controller y por qué
* **Cuándo:** En aplicaciones web corporativas donde múltiples endpoints comparten requerimientos de ciclo de vida (rutas, seguridad, decodificación, errores globales).
* **Por qué:** Centraliza el punto de contacto para erradicar el antipatrón **Page Controller** (archivos o scripts aislados) y garantiza que ninguna solicitud esquive las políticas transversales de seguridad.

### 2. Soporte nativo en frameworks modernos
No se debe construir una cadena GoF artesanal; los frameworks ya proveen Front Controller y pipeline de middlewares integrados:
* **Django:** `WSGIHandler` / `ASGIHandler` y la lista `MIDDLEWARE` en `settings.py`.
* **Spring Boot:** `DispatcherServlet` con `HandlerInterceptor` y `SecurityFilterChain`.
* **Laravel:** `public/index.php` con pipeline de `Middleware`.
* **Express.js:** `express()` con cascada `app.use((req, res, next) => ...)`.

### 3. Orden de la cadena y justificación
* **Pipeline recomendado:**
  $$\text{Bitácora Entrada} \longrightarrow \text{Autenticación \& Autorización} \longrightarrow \text{Caso de Uso (Dominio)} \longrightarrow \text{Bitácora Salida}$$
* **Ubicación del cupo de inscripciones/asientos:**  
  **No va en los filtros HTTP.** Es una regla de negocio del modelo de dominio y debe evaluarse dentro del caso de uso bajo control transaccional en base de datos (`SELECT ... FOR UPDATE`).
* **¿Por qué autenticar después del cobro es un error de diseño?**
  * **Violación de Fail-Fast y menor privilegio:** La identidad del sujeto es condición previa indispensable para autorizar cualquier mutación financiera.
  * **Riesgo de fraude y DoS financiero:** Peticiones anónimas o maliciosas podrían saturar la pasarela bancaria o debitar dinero de cuentas ajenas sin validación.
  * **Transacciones huérfanas:** Si el cobro tiene éxito pero la autenticación falla después, se cobra dinero real sin un usuario válido asignado en el sistema, obligando a reversiones manuales complejas.

### 4. Lo que NUNCA debe hacer la cadena («Las políticas transversales no son el reglamento»)
* **Propósito exclusivo de los filtros:** Resolver aspectos ortogonales de infraestructura y transporte HTTP (CORS, CSRF, JWT/sesiones, rate limiting, logging de acceso, serialización técnica de excepciones).
* **Prohibido en la cadena de filtros:**
  * ❌ Validar disponibilidad de cupos o asientos.
  * ❌ Calcular precios, recargos o descuentos.
  * ❌ Validar requisitos institucionales o del reglamento de usuarios.
  * ❌ Descontar puntos, saldos o balances de cuenta.
* **Consecuencia:** Acoplar el reglamento a los filtros HTTP impide reutilizar la lógica de negocio en tareas CLI, colas en segundo plano, pruebas unitarias o eventos asíncronos.
