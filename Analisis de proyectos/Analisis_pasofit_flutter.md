# Análisis de Arquitectura - pasofit_flutter

## Comentarios Generales del Proyecto

El proyecto es una app Flutter (paquete `pasofit`, "V6 FINAL") con seis pantallas, un bus de oyentes casero, un `Cubit` huérfano y SQL embebido en widgets. Problemas detectados:

- No hay separación de capas: las pantallas (`pantallas/*.dart`) hacen HTTP, SQL y lógica de UI en el mismo archivo.
- Hay SQL directo en `rutina.dart`, `peso.dart` y `logros.dart` (`rawInsert`, `rawQuery`) usando concatenación de strings, lo que abre la puerta a inyección SQL.
- Se reimplementa un bus de oyentes propio (`RelojBus`, `PesoBus` en `oyentes.dart`) en paralelo a `flutter_bloc`, que ya está en `pubspec.yaml`. `RelojCubit` existe pero no se usa.
- Las llamadas HTTP se hacen con `http.get`/`http.post` desde `initState`, sin token de sesión real y contra `http://127.0.0.1/fit/...`, que apunta al host del backend (no del dispositivo emulado).
- El control de acceso es simulado: el login valida con `r.body.contains('ok')` y se "salta" la sesión con la cadena `admin_bypass=1` en `inicio.dart`.
- El `DropdownButton` de `rutina.dart` no está conectado al estado (`onChanged: (_) {}`), así que el `tipo` seleccionado en pantalla no se usa para insertar; llega siempre el de la ruta.
- `docs/ARQUITECTURA.md` está pegado de otro proyecto (habla de "banca móvil" y de "rutinas" como si fueran prohibidas); no aplica a este código.

## Misión 1 - El portal no es un patrón

| Problema | Capa | Patrón que corresponde | Por qué ese y no el vecino más fácil de confundir | Cuándo NO aplicaría |
| :--- | :--- | :--- | :--- | :--- |
| `RelojBus` y `PesoBus` en `oyentes.dart` reinventan un bus de eventos con `List<Function>` y duplican lo que ya hace `flutter_bloc`. | Estado / Presentación | Observer (canónico, vía Bloc/Cubit) | Es Observer porque varios widgets reaccionan a un mismo cambio (segundos, kilos). No es Mediator porque el bus no coordina objetos con referencias entre sí: solo difunde. No es Publish-Subscribe corporativo: para eso ya existe `flutter_bloc`, que ya es Observer de fábrica. | Si la app no tuviera listeners múltiples (un solo widget observando el mismo valor). |
| `rutina.dart` ejecuta dos `rawInsert` directos a `sqflite` desde el widget, con strings concatenados (`'$tipo'`, `$kcal`). | Datos | Repository | Es Repository porque aísla el SQL detrás de una interfaz y permite probar la lógica sin abrir una BD. No es Active Record porque la clase `RutinaPantalla` no es una entidad de dominio: es un widget. | Si solo se tratara de un cache temporal sin reglas ni agregados. |
| `inicio.dart` decide el acceso por una comparación débil (`cookie=='si' \|\| admin_bypass=='1'`) y un comentario "a veces entra igual". | Políticas transversales | Intercepting Filter (middleware) | Es Intercepting Filter porque la decisión de "puedes entrar" es transversal a todas las pantallas, no del dominio de Inicio. No es Page Controller porque el Page Controller no autentica: enruta. El login real también está en LoginPantalla sin una sesión formal. | Si la app no requiriera autenticación (modo solo-lectura público). |
| `login.dart` decide éxito por `r.body.contains('ok') \|\| r.body.contains('id')` y manda argumentos literales por `Navigator.pushNamed`. | Presentación / Integración | Adapter (cliente HTTP) + DTO/Command | Es Adapter porque traduce el protocolo del backend (un texto plano "ok/id") a un evento de UI. No es Facade porque el backend no es un subsistema propio. No es Strategy porque no hay políticas intercambiables. | Si el backend ya entregara un JSON con un campo `success` o un token JWT, no haría falta adaptar. |
| `cronometro.dart` y `peso.dart` se suscriben a un bus global en `initState` y nunca se dan de baja (`dispose`). | Estado / Presentación | Observer con ciclo de vida correcto | Es Observer (correcto), pero mal aplicado. El bus guarda la función en una lista estática y nunca se quita: cada vez que se abre la pantalla se acumulan listeners, multiplicando los `setState`. No es Pub/Sub porque no hay broker; es un bus estático de proceso. | Si la pantalla se montara una sola vez y nunca se destruyera (raro en móvil). |
| `logros.dart` arma el HTML del contador con un `for` que anida dos `rawQuery` y concatena el resultado en un `setState`. | Datos / Vista | Repository + Transform View | Es Repository porque las dos consultas (`logros_pend` y el `COUNT` por tipo) deberían vivir juntas en un repositorio y devolver un agregado. No es Active Record porque no hay entidad. La vista tampoco debería componer el HTML; le debería llegar el número y un View Template. | Si los logros fueran un campo calculado trivial sin joins. |

## Misión 2 - Una petición, varios patrones (Pagar / registrar sesión de entrenamiento)

Aunque pasofit no hace cobros, el flujo más cercano al "caso de uso de la unidad" es **"Iniciar"** en `rutina.dart`, porque persiste, notifica y responde. Lo recorro como si fuera la petición:

1. **Pantalla `RutinaPantalla`** recibe `Navigator.pushNamed(..., arguments: {'tipo': 'cardio'})`.  
   Patrón: Page Controller (Flutter) + Command implícito (los argumentos viajan como un mapa sin tipo).

2. **Lectura del `tipo` y selección de `kcal`** mediante un `if-else if` con cinco casos (`cardio`, `fuerza`, `yoga`, `hiit`, `bailar`).  
   Patrón: Strategy mal aplicado (es un `switch`/`if` que debería ser un mapa o una jerarquía de estrategias).  
   Vecino rechazado: Chain of Responsibility, porque no se pasa la petición entre manejadores; se calcula y ya.

3. **Persistencia local** con dos `db.rawInsert` directos: uno a `sesiones`, otro a `logros_pend`.  
   Patrón: Repository ausente (anti-patrón). Lo correcto: una unidad `SesionDeRutinaRepository` con métodos `registrar(Sesion, Kcal)`.  
   Vecino rechazado: Active Record, porque aquí no hay entidad: el widget hace el SQL él mismo.

4. **Notificación al backend** con `http.post(Uri.parse('http://127.0.0.1/fit/mail'), body: {'t': 'sesion $tipo $kcal'})`.  
   Patrón: Adapter (cliente HTTP). Traduce del modelo del dominio al formulario que espera el servicio de correo.  
   Vecino rechazado: Facade, porque el backend de correo no es nuestro subsistema.

5. **Respuesta en UI** con `setState(() => txt = 'guardado $kcal — no recargue')`.  
   Patrón: Observer local (State de Flutter). El State notifica al `build`; los textos largos deberían venir de un l10n.

6. **Ausencia de idempotencia**: el botón "Iniciar" no tiene clave y se puede pulsar dos veces; cada vez inserta otra fila.  
   Patrón ausente: clave de idempotencia (patrón de integración REST). No hay GoF cercano; es un problema de diseño del recurso, no de un objeto.

> Nota: como Flutter no tiene "Front Controller" al estilo web, la analogía correcta aquí es que `MaterialApp` + `routes` cumple ese papel (un único punto de despacho), pero `main.dart` lo monta de forma muy superficial (sin guard, sin redirección, sin carga diferida).

## Misión 3 - Políticas transversales

### 1. ¿Cuándo corresponde Front Controller y por qué?
En Flutter, el equivalente es el **`Navigator` con `routes` declaradas en `MaterialApp`**: un solo punto de despacho. Corresponde cuando varias pantallas comparten las mismas políticas (auth, bitácora, red). En `main.dart` se declara el mapa de rutas, pero no hay guard ni redirección; cualquier `pushNamed` llega sin checar sesión. Cuando hay una política transversal repetida en varios widgets, hay que centralizarla (en este caso, en una capa por encima de las pantallas).

### 2. ¿Qué mecanismo ya ofrece el marco?
- **Flutter**: `Navigator` + `MaterialApp(routes: ...)` y, sobre todo, `RouteObserver` / `NavigatorObserver` para observar transiciones. Para políticas previas al push se usa un `Middleware`/`Guard` propio o un `Router` declarativo (`go_router`), que ya da `redirect`.
- **Si se quisiera homologar con Spring/Laravel/Express** (recordatorio del enunciado): Spring tiene `DispatcherServlet` + filtros; Laravel tiene Middleware; Express tiene `app.use(...)`. La idea es la misma: una sola pila donde se aplican auth, logs y límites.

### 3. Orden de la cadena y justificación
Para pasofit, las "políticas transversales" son:
1. Parseo / cabeceras / disponibilidad de red.
2. Autenticación (¿hay sesión iniciada?).
3. Autorización (¿este usuario puede entrar a esta ruta?).
4. Bitácora / telemetría de uso.
5. Caso de uso (registrar sesión, leer kardex, etc.).

**¿Por qué autenticar después del cobro (o después del efecto) es un error de diseño?** En esta app, el botón "Iniciar" ya inserta en SQLite y avisa al backend antes de cualquier chequeo. Si la sesión se cerrara, las inserciones seguirían pasando porque el widget no consulta al backend; el resultado es basura en la BD y un correo enviado sin dueño. La analogía con el enunciado original es directa: ejecutar efectos persistentes antes de saber **quién** los pidió deja la transacción sin sujeto, impide idempotencia y rompe la auditoría.

### 4. ¿Qué NO debe hacer la cadena?
No debe contener reglas de negocio (qué tipo de rutina cuenta doble, cuándo se desbloquea un logro). Tampoco debe persistir directamente: la cadena puede decidir que se entra o no se entra, y registrar el evento, pero el SQL de `sesiones` o `logros_pend` debe seguir siendo del repositorio. En una frase: la cadena **filtra**, no **opera**.

---

## Verificación rápida de lo que sí funciona vs. lo que está roto

- **`flutter_bloc` está declarado** (`pubspec.yaml`) pero **no se usa**: el `main.dart` no provee ningún `BlocProvider`, el `RelojCubit` está huérfano y `oyentes.dart` reinventa un bus estático de oyentes. Recomendación: borrar el bus casero y migrar `cronometro`/`peso` a un `Cubit` por feature, con `close()` en `dispose`.
- **`provider` también está declarado** y tampoco se usa. Conviene decidir uno (recomendado: `flutter_bloc`) y quitar el otro, como dice el doc de Arquitectura (que, por cierto, está mal copiado en este repo).
- **Las llamadas HTTP están rotas en origen**: `http://127.0.0.1/fit/...` apunta al host, no al dispositivo. En Android emulator sería `http://10.0.2.2/...`.
- **No hay idempotencia** ni en "Iniciar" (dos clics = dos filas) ni en "Registrar peso" (dos clics = dos filas). La BD ni siquiera tiene `UNIQUE` ni `PRIMARY KEY` propios: `sesiones` y `logros_pend` no declaran PK.
- **Inyección SQL**: `rutina.dart` y `logros.dart` interpolan `$tipo` dentro de `rawInsert` / `rawQuery`. Si en el futuro el `tipo` viniera de un input real, es una vulnerabilidad directa.

## Nota final

**Recomendación:** rehacer el módulo `lib/` separando capas: `datos/` (repositorios + sqflite con migraciones), `estado/` (Cubits, uno por feature), `pantallas/` (widgets sin SQL ni HTTP), `integracion/` (clientes HTTP con DTOs y claves de idempotencia). El refactor tomaría menos tiempo que seguir parcheando los widgets con más `if-else`.

**Por qué:** el patrón aquí no falta por ignorancia, sino porque **el código no está estructurado por capas**: SQL, HTTP y UI viven en la misma clase, lo que hace que cualquier patrón que se nombre se quede en el PowerPoint. Mientras un widget siga haciendo `db.rawInsert` en `onPressed`, no hay Repository, no hay Strategy, no hay Observer "real" que valga.