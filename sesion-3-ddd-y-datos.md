<img src="logo.png" alt="NobleProg" width="280">

# Sesión 3 — Domain-Driven Design y descomposición de datos

**Duración:** 3 h · **Objetivo:** aprender a encontrar buenos límites de servicio (DDD) y abordar la parte más difícil de toda migración: los datos.

---

## 1. Domain-Driven Design (DDD)

### 1.1 Por qué DDD y microservicios van de la mano

La pregunta más cara de toda la migración es: **¿por dónde corto?** Un corte malo produce el monolito distribuido (Sesión 1). DDD (Eric Evans, 2003) es, hoy por hoy, la mejor herramienta conocida para responder, porque propone alinear los límites del software con los límites **del negocio** — y los límites del negocio son los más estables que existen.

DDD tiene dos planos:

- **DDD estratégico** — cómo trocear un dominio grande: *bounded contexts*, lenguaje ubicuo, *context maps*. **Esto es lo que necesita una migración.**
- **DDD táctico** — patrones de diseño dentro de un contexto: entidades, agregados, repositorios… Útil, pero secundario para decidir límites de servicio (mencionaremos solo los *agregados*).

### 1.2 Lenguaje ubicuo

El **lenguaje ubicuo** es un vocabulario compartido entre negocio y desarrollo, usado de forma consistente en conversaciones, documentos **y código**. Si negocio dice "póliza" y el código dice `ContractRecord`, cada conversación necesita traducción y cada traducción pierde información.

La observación clave para encontrar límites: **el lenguaje cambia de significado según el contexto.** "Cliente" no significa lo mismo para ventas (un lead con potencial), para facturación (un titular con datos fiscales y formas de pago) y para soporte (alguien con tickets y un historial de incidencias). No es un fallo de comunicación: son **conceptos genuinamente distintos** que comparten nombre.

### 1.3 Bounded contexts (contextos delimitados)

Un **bounded context** es la frontera dentro de la cual un modelo y su lenguaje son válidos y consistentes. Dentro del contexto "Facturación", `Cliente` significa exactamente una cosa; fuera, no se garantiza nada.

Consecuencia liberadora: **no hay que construir un modelo único de toda la empresa** (el intento de "el modelo canónico corporativo de Cliente" con 200 campos que no satisface a nadie). Cada contexto modela *su* versión del concepto, con solo los atributos que le importan:

```
┌── Ventas ─────────────┐  ┌── Facturación ────────┐  ┌── Soporte ───────────┐
│ Cliente:              │  │ Cliente:              │  │ Cliente:             │
│  · interés            │  │  · NIF, dirección     │  │  · tickets           │
│  · historial contacto │  │    fiscal             │  │  · nivel SLA         │
│  · probabilidad cierre│  │  · forma de pago      │  │  · idioma preferido  │
└───────────────────────┘  └───────────────────────┘  └──────────────────────┘
        mismo nombre, tres modelos distintos — y está bien
```

**Cómo detectar fronteras de contexto en la práctica:**

- Cambios de **significado** de una misma palabra según el departamento.
- Sinónimos: dos áreas llaman distinto a lo mismo.
- Fronteras organizativas y de proceso de negocio.
- En el monolito existente: grupos de tablas/módulos que cambian juntos, "traducciones" implícitas en el código (mappers, columnas que se reinterpretan).
- Talleres colaborativos tipo **EventStorming**: negocio y desarrollo mapean juntos los eventos del dominio en una pared; los racimos de eventos y los cambios de actor/vocabulario sugieren contextos.

### 1.4 Microservicios por dominio de negocio

La regla de oro de la descomposición:

> **Un microservicio no debe ser más pequeño que un bounded context** y, como punto de partida, un bounded context ≈ un servicio.

Por qué funciona: dentro de un contexto la cohesión es alta (las cosas cambian juntas → viven juntas) y entre contextos el acoplamiento es bajo (las interacciones son pocas y formalizables como contratos). Eso es exactamente lo que pide el despliegue independiente.

Por qué fracasa lo contrario (servicios por capa técnica o por entidad): un cambio de negocio típico ("añadir descuentos por volumen") atraviesa todas las capas y varias entidades → toca N servicios → monolito distribuido.

**Ejemplo.** Un e-commerce descompuesto por dominio: `catalogo`, `pedidos`, `pagos`, `envios`, `clientes`. El cambio "permitir reservar stock al añadir al carrito" toca *un* servicio (pedidos o catálogo, según el modelo). El mismo sistema descompuesto en `web-api` / `business-logic` / `data-access` obligaría a tocar los tres para *cualquier* cambio.

---

## 2. Encontrar los límites

### 2.1 Context maps (mapas de contexto)

Un **context map** es el plano de los contextos y de las **relaciones entre ellos**. No basta saber cuáles son los contextos; hay que saber cómo se influyen, porque eso determina los contratos y el poder de negociación entre equipos. Relaciones clásicas de DDD:

| Relación | Significado |
|---|---|
| **Customer–Supplier** | El contexto proveedor tiene en cuenta las necesidades del consumidor (hay negociación) |
| **Conformist** | El consumidor acepta el modelo del proveedor tal cual (sin poder de negociación) |
| **Anticorruption Layer** | El consumidor se protege traduciendo el modelo del proveedor (Sesión 2) |
| **Shared Kernel** | Dos contextos comparten una parte del modelo (acoplamiento fuerte y deliberado — usar con muchísima moderación) |
| **Open Host Service / Published Language** | El proveedor publica una API/formato estable y documentado para muchos consumidores |
| **Separate Ways** | No se integran: duplicar es más barato que coordinar |

En una migración, el context map sirve para: ver qué fronteras ya existen de facto en el monolito, decidir dónde hacen falta ACLs, y detectar relaciones enfermas (p. ej. todo el mundo *conformista* con el modelo de la base de datos central).

### 2.2 Descomposición por capacidad de negocio vs. por subdominio

Dos heurísticas complementarias para proponer contextos:

- **Por capacidad de negocio:** ¿qué *hace* la empresa? (gestionar pedidos, facturar, atender clientes). Mirada de fuera hacia dentro, estable, entendible por negocio. Riesgo: quedarse en el organigrama actual (que puede ser accidental).
- **Por subdominio:** ¿de qué *se compone* el dominio del problema? DDD clasifica los subdominios en:
  - **Core (núcleo):** lo que diferencia a la empresa de la competencia. Merece el mejor equipo y el mejor diseño.
  - **Supporting (soporte):** necesario pero no diferencial; hecho a medida porque no hay alternativa.
  - **Generic (genérico):** lo resuelve igual una solución comprada/estándar (contabilidad, login, email).

La clasificación core/supporting/generic es oro para **priorizar la migración**: el esfuerzo de modelado fino (bubble contexts, DDD táctico) se concentra en el *core*; lo genérico, a menudo, ni se migra — se reemplaza por SaaS.

### 2.3 Granularidad: ¿cuándo es demasiado pequeño?

Señales de que has cortado **demasiado fino**:

- **Cambios en cascada:** una funcionalidad media toca varios servicios (el test del monolito distribuido).
- **Charla excesiva:** dos servicios que se llaman constantemente y siempre aparecen juntos en las trazas.
- **Transacciones partidas:** necesitas consistencia fuerte *entre* servicios continuamente (la frontera parte un invariante de negocio por la mitad).
- **Datos siameses:** dos servicios necesitan leer/escribir las mismas tablas.
- Más servicios que desarrolladores, o equipos que poseen tantos servicios que viven haciendo mantenimiento de plataforma.

> **Heurística práctica:** es mucho más barato dividir un servicio que resultó grande que fusionar dos servicios que resultaron pequeños (fusionar implica unir modelos, datos y contratos ya publicados). **Ante la duda, corta más grueso.** Empieza con pocos servicios alineados con contextos y subdivide cuando el dolor lo justifique.

---

## 3. Descomposición de datos

### 3.1 El problema: la base de datos compartida

Se puede extraer todo el código que se quiera: mientras los servicios compartan base de datos, **siguen acoplados**:

- El **esquema es un contrato implícito** entre todos: nadie puede cambiar una tabla sin romper a los demás (adiós despliegue independiente).
- La lógica acaba en el lugar equivocado: con acceso directo a las tablas de otro, su lógica de negocio se reimplementa (y diverge) en cada consumidor.
- Imposible cambiar de tecnología de almacenamiento por servicio.
- El rendimiento de uno afecta a todos (bloqueos, pools, picos).

Por eso el objetivo es **base de datos por servicio**: cada servicio es el único que accede a sus datos; los demás pasan por su API. Los datos son la parte más difícil y más aplazada de toda migración — y mientras no se haga, la migración no está hecha.

### 3.2 Dividir los datos: propiedad primero

Pasos conceptuales para romper una BD compartida:

1. **Mapear propiedad:** para cada tabla, ¿qué contexto la *escribe* y cuál es su dueño natural? (Las tablas que escribe todo el mundo son las campanas de alarma.)
2. **Separar el esquema lógicamente** primero: esquemas/usuarios distintos dentro de la misma instancia, prohibiendo los cruces. Barato y detecta todos los acoplamientos sin mover un byte.
3. **Romper los accesos cruzados** uno a uno, sustituyéndolos por:
   - llamada a la **API** del servicio dueño (cuando se necesita el dato fresco), o
   - una **copia local de lectura** alimentada por eventos (cuando se tolera un pequeño desfase), o
   - mover el dato de dueño (a veces el reparto inicial estaba mal).
4. **Separar físicamente** (instancia propia) cuando ya no hay cruces.

Casos típicos difíciles:

- **Tabla compartida de verdad** (p. ej. `paises`, datos de referencia): duplicarla en cada servicio (cambia poco) o un servicio de datos de referencia.
- **Una tabla, dos dueños** (`pedidos` con columnas de pedido y columnas de envío): partirla — cada contexto se lleva sus columnas; la clave compartida (`pedido_id`) se convierte en referencia entre servicios.
- **Joins entre dominios** (informe que cruza clientes con pedidos): ya no hay JOIN entre BDs distintas → composición en el llamante, o una vista materializada alimentada por eventos (ver §4.4).

### 3.3 El coste de los datos distribuidos

Hay que decirlo sin anestesia: al partir los datos **se pierden** cosas que la BD única regalaba:

| Se pierde | Qué lo sustituye (peor y con más trabajo) |
|---|---|
| Transacciones ACID entre dominios | Sagas + consistencia eventual |
| JOIN entre cualquier par de tablas | Composición por API o vistas materializadas |
| Integridad referencial (FKs) global | Validación en aplicación; tolerar referencias rotas temporales |
| Un único lugar donde mirar los datos | Datos repartidos; necesidad de catálogo/linaje |

Este coste es la razón por la que los límites importan tanto: si el corte es bueno, las transacciones y joins *entre* servicios son raros; si es malo, son constantes y el sistema se vuelve inmanejable. **Los invariantes que necesitan consistencia fuerte deben quedar dentro de un mismo servicio** (en DDD táctico: dentro de un *agregado* — la unidad de consistencia transaccional).

---

## 4. Consistencia sin transacciones distribuidas

### 4.1 Por qué no usamos transacciones distribuidas (2PC)

Existe la transacción distribuida clásica (*two-phase commit*): un coordinador pide a todos los participantes que voten y luego confirma. En microservicios se evita porque: bloquea recursos en todos los participantes mientras dura (mata el rendimiento), el coordinador es un punto único de fallo con estados "en duda" horribles de operar, acopla la disponibilidad de todos (la transacción avanza al ritmo del más lento/caído) y muchas tecnologías modernas (colas, BDs NoSQL, APIs HTTP) simplemente no lo soportan.

### 4.2 Consistencia eventual

La alternativa: aceptar que el sistema pase por estados intermedios visibles y garantizar que **converge** al estado correcto. "El pedido está creado pero el stock aún no está descontado… durante 200 ms (o 2 s)." 

Lo crucial es que esto **no es (solo) un problema técnico, es una conversación de negocio**: casi todos los procesos de negocio reales *ya son* eventualmente consistentes (la tienda física te vende algo y el stock central se entera por la noche; los bancos liquidan transferencias por lotes). Las preguntas correctas son: ¿qué desfase es aceptable aquí? ¿qué hacemos si al final no se puede completar? — y esas las responde negocio.

### 4.3 El patrón Saga

Una **saga** es una secuencia de transacciones **locales** (cada una ACID dentro de su servicio) que en conjunto implementan un proceso de negocio. Si un paso falla, no hay rollback global: se ejecutan **compensaciones** — acciones de negocio que deshacen los efectos de los pasos ya confirmados.

Ejemplo — "realizar pedido": 1) Pedidos crea el pedido → 2) Stock reserva los artículos → 3) Pagos cobra → 4) Pedidos confirma. Si el cobro (3) falla: compensar 2 (liberar la reserva) y compensar 1 (marcar el pedido como cancelado — nótese: *cancelado*, no *borrado*; la compensación es una acción de negocio con su propia semántica, no un "deshacer" mágico).

**Dos estilos de coordinación:**

- **Orquestación:** un coordinador central (p. ej. el propio servicio de Pedidos o un motor de procesos) llama a cada paso y decide compensar.
  - ✅ El proceso es explícito y visible en un sitio; fácil de razonar, depurar y modificar; el estado del proceso vive en el orquestador.
  - ⚠️ El orquestador concentra acoplamiento (conoce a todos) y puede volverse un "dios"; punto de fallo a vigilar.
- **Coreografía:** no hay coordinador; cada servicio reacciona a eventos y emite los suyos (`PedidoCreado` → Stock reserva y emite `StockReservado` → Pagos cobra y emite `PagoRealizado`…).
  - ✅ Acoplamiento mínimo, servicios autónomos, encaja con arquitecturas de eventos.
  - ⚠️ El proceso global no está escrito en ninguna parte: emerge. Difícil saber "¿dónde está el pedido 4711 y por qué se ha quedado a medias?"; los ciclos de eventos y los cambios del flujo son delicados.

**Heurística:** procesos cortos y estables entre pocos servicios → coreografía; procesos largos, con ramas, compensaciones complejas o necesidad de visibilidad → orquestación. Mezclar también vale (orquestar el núcleo, coreografiar las reacciones periféricas).

### 4.4 La idea del Outbox

Problema sutil pero omnipresente: un servicio debe **guardar en su BD y publicar un evento**. Si guarda y luego se cae antes de publicar → los demás no se enteran (saga colgada). Si publica y luego falla el guardado → el mundo cree algo que no pasó. No hay transacción que abarque BD *y* broker.

**Patrón Outbox (transactional outbox):** en la **misma transacción local** en que se guarda el cambio de negocio, se inserta el evento en una tabla `outbox` de la misma BD (eso sí es atómico). Un proceso aparte (relay) lee la outbox y publica los eventos al broker, marcándolos como enviados; si se cae, reintenta.

```
┌───────────── Servicio Pedidos ─────────────┐
│  TX local: INSERT pedido + INSERT outbox   │ ← atómico ✔
│                 ┌────────┐                 │
│                 │ outbox │ ──► relay ──────┼──► broker ──► suscriptores
│                 └────────┘  (reintenta)    │
└────────────────────────────────────────────┘
```

Consecuencia: la entrega es **al-menos-una-vez** (el relay puede reenviar tras un fallo) → los consumidores deben ser **idempotentes** (procesar el mismo evento dos veces sin daño; p. ej. deduplicando por ID de evento). Este dúo outbox+idempotencia es el fundamento de fiabilidad de casi toda arquitectura de eventos (más sobre idempotencia en la Sesión 4).

### 4.5 Consultas e informes entre servicios (a nivel conceptual)

Sin JOINs globales, ¿cómo se responde "los 10 clientes con más pedidos este mes"? Opciones conceptuales:

1. **Composición de APIs:** el llamante consulta varios servicios y combina en memoria. Bien para pocas entidades; mal para agregaciones masivas (N llamadas, paginación, latencia).
2. **Vista materializada / CQRS a nivel de sistema:** un componente de consulta se **suscribe a los eventos** de los servicios implicados y mantiene una BD de lectura ya cruzada y optimizada para esas consultas. Eventualmente consistente (suele ser aceptable para informes).
3. **Plataforma analítica:** para informes pesados/históricos, los servicios alimentan (por eventos o extracciones) un data warehouse/lake. El reporting *no* se hace contra las BDs operacionales de los servicios.

Regla práctica: las consultas que cruzan dominios son **otro consumidor más de eventos**, no una excusa para volver a la BD compartida.

---

## Ejercicios

### Ejercicio 3.1 — Detectar bounded contexts por el lenguaje

En **VuelaYa** (aerolínea low-cost) recoges estas frases:

- Operaciones: "Un *vuelo* es un avión concreto despegando un día concreto; si hay niebla, el vuelo se retrasa."
- Ventas: "El *vuelo* MAD–LIS de las 7:40 es nuestro producto estrella; lo vendemos a 39 €."
- Programa de fidelidad: "Cada *vuelo* te da puntos según la tarifa."
- Ventas llama "*pasajero*" a quien compra; Operaciones llama "*pasajero*" a quien está en la lista de embarque (¡aunque el billete lo comprara otra persona!).
- Mantenimiento: "Ese avión no puede *volar*: toca revisión de las 1.000 horas."

**Tareas:**
1. ¿Cuántos significados distintos de "vuelo" identificas? Da a cada uno un nombre desambiguado.
2. Propón los bounded contexts que sugieren estas frases.
3. ¿Qué pasaría si se construyera una única entidad `Vuelo` compartida para todos?

<details>
<summary>💡 Solución</summary>

**1. Significados de "vuelo":**

- **Operación de vuelo** (Operaciones): un despegue concreto de un avión concreto en una fecha concreta. Tiene estado físico: retrasado, embarcando, aterrizado.
- **Producto/ruta comercial** (Ventas): el "MAD–LIS de las 7:40" como artículo vendible que existe todos los días, con precio. No se retrasa: se *vende*.
- **Vuelo puntuable** (Fidelidad): un consumo que genera puntos según tarifa. Solo le importan el pasajero, la tarifa y que efectivamente se voló.
- (Mantenimiento usa "volar" como **aeronavegabilidad** del avión: ni siquiera habla de vuelos, habla de aviones y horas — concepto distinto.)

**2. Bounded contexts sugeridos:** `Operaciones de vuelo` (aviones, slots, retrasos, embarque), `Ventas/Reservas` (rutas, tarifas, precios, compradores), `Fidelidad` (cuentas de puntos, acumulación, canje), `Mantenimiento` (aeronaves, horas, revisiones). El dato del doble "pasajero" refuerza la frontera Ventas/Operaciones: *comprador* (paga el billete) vs. *pasajero embarcado* (vuela) son conceptos distintos que comparten palabra.

**3. La entidad `Vuelo` única:** sería la unión de todos los atributos (precio + estado de embarque + puntos + matrícula del avión + …), con todos los equipos tocando la misma clase/tabla. Consecuencias: cada cambio de cualquier área impacta a todas (la clase cambia constantemente y por motivos no relacionados — violación masiva de cohesión); imposible asignar un dueño; reglas contradictorias conviviendo (¿puede "cancelarse" un vuelo que es un producto de catálogo?); y en una migración, esa entidad-dios se convertiría en la tabla compartida que impide separar las bases de datos. Es exactamente el "modelo canónico corporativo" contra el que previene DDD: mejor cuatro modelos pequeños y un contrato claro entre contextos (p. ej. Operaciones publica `VueloAterrizado` y Fidelidad lo consume para puntuar).

</details>

---

### Ejercicio 3.2 — Core, supporting o generic

Clasifica los subdominios de **VuelaYa** en core / supporting / generic y extrae las consecuencias para la migración:

A. Motor de precios dinámicos (ajusta precios por demanda, el orgullo de la empresa)
B. Gestión de tripulaciones (asignar tripulaciones cumpliendo normativa de descansos)
C. Envío de emails transaccionales (confirmaciones, tarjetas de embarque)
D. Contabilidad financiera
E. Programa de fidelidad (similar al de cualquier aerolínea)

**Pregunta extra:** ¿qué subdominios merecen bubble context + modelado fino, y cuáles posiblemente ni se migran?

<details>
<summary>💡 Solución</summary>

| Subdominio | Clasificación | Razonamiento |
|---|---|---|
| A. Precios dinámicos | **Core** | Es la ventaja competitiva explícita ("el orgullo de la empresa"); diferencia a la aerolínea |
| B. Tripulaciones | **Supporting** (probablemente) | Necesario y complejo (normativa), pero no diferencia frente a la competencia; quizá no exista software estándar que encaje del todo → a medida |
| C. Emails transaccionales | **Generic** | Cualquier proveedor SaaS de email lo hace mejor y más barato |
| D. Contabilidad | **Generic** | Problema 100 % estándar; se compra (ERP), no se construye |
| E. Fidelidad | **Supporting o generic** | "Similar al de cualquier aerolínea" = no diferencial; existen plataformas de fidelización compradas → tiende a generic, salvo necesidades muy particulares |

(La clasificación admite matices — lo importante es el razonamiento "¿esto nos diferencia?" — y puede cambiar con la estrategia de la empresa.)

**Consecuencias para la migración:**

- **A (core):** máxima inversión — bubble context con modelo limpio, el mejor equipo, posiblemente lo primero que se extrae como servicio para darle velocidad de evolución independiente.
- **B (supporting):** servicio propio probablemente, pero con modelado pragmático; no es donde se gana la guerra.
- **C, D (generic):** **no se migran a microservicios propios — se reemplazan**: el email por un SaaS de envío, la contabilidad por un ERP. "Migrar" aquí significa construir la integración (¡con su ACL!) y apagar el módulo legacy. Es la forma más barata de reducir el monolito: borrar en lugar de reescribir.
- **E:** decisión de compra vs. construcción; si se compra, igual que C y D.

**Lección:** la clasificación de subdominios evita el error de gastar el mismo esfuerzo de diseño (y de migración) en todo por igual.

</details>

---

### Ejercicio 3.3 — Partir la base de datos compartida

En el monolito de **ComidaFlash** (delivery), los nuevos servicios `pedidos` y `restaurantes` aún comparten la BD, con estas tablas y accesos:

| Tabla | Escriben | Leen |
|---|---|---|
| `pedidos` | pedidos | pedidos, restaurantes (para ver sus pedidos entrantes) |
| `restaurantes` | restaurantes | restaurantes, pedidos (nombre y dirección al crear pedido) |
| `menu_items` | restaurantes | restaurantes, pedidos (precio y disponibilidad al crear pedido) |
| `codigos_postales` | nadie (carga anual) | ambos |

**Tareas:**
1. Asigna cada tabla a su servicio dueño.
2. Para cada acceso cruzado de lectura, propón cómo eliminarlo (API vs. copia local por eventos) y justifica según frescura y frecuencia.
3. ¿Qué haces con `codigos_postales`?
4. Al crear un pedido hay que comprobar que el `menu_item` está disponible y su precio. Tras la separación, ¿qué problema de consistencia aparece y cómo lo tratarías?

<details>
<summary>💡 Solución</summary>

**1. Propiedad** (regla: el dueño es quien escribe): `pedidos` → servicio pedidos; `restaurantes` y `menu_items` → servicio restaurantes; `codigos_postales` → datos de referencia, sin dueño natural (ver punto 3).

**2. Accesos cruzados:**

- **restaurantes lee `pedidos`** (sus pedidos entrantes): es una necesidad de *notificación + listado*. Mejor opción: el servicio pedidos **publica eventos** (`PedidoCreado`, `PedidoCancelado`) y restaurantes mantiene su propia vista de "pedidos entrantes" (copia local de lo que necesita: id, artículos, hora). Frescura: segundos de desfase son aceptables; frecuencia alta → eventos mejor que polling por API.
- **pedidos lee `restaurantes`** (nombre, dirección): datos que cambian rarísimamente y se leen en cada pedido → **copia local alimentada por eventos** (`RestauranteActualizado`). Llamar a la API en cada creación de pedido también funciona, pero acopla la disponibilidad: si restaurantes está caído, no se pueden crear pedidos por un dato casi estático. La copia local da autonomía.
- **pedidos lee `menu_items`** (precio, disponibilidad): el caso interesante — ver punto 4. Ambas opciones son defendibles: API síncrona (dato fresco) o copia local por eventos (autonomía). Dado el volumen (cada creación de pedido) y que la disponibilidad cambia a menudo, una copia local por eventos con desfase de segundos suele ser el equilibrio correcto… si negocio acepta el punto 4.

**3. `codigos_postales`:** datos de referencia inmutables a efectos prácticos (carga anual). La solución pragmática: **cada servicio lleva su propia copia** (tabla propia, o incluso fichero/configuración), actualizada con la carga anual. Montar un "servicio de códigos postales" sería sobreingeniería: añadiría una dependencia de runtime para un dato que cambia una vez al año.

**4. El problema de consistencia:** con copia local (o incluso con API: el dato puede cambiar entre la lectura y la confirmación), puede ocurrir que el cliente pida un plato que **acaba de agotarse** o cuyo **precio acaba de cambiar**. Antes lo "resolvía" la transacción/lectura en la misma BD; ahora es imposible de eliminar del todo — solo se puede decidir cómo tratarlo. Tratamiento:
- **Pregunta a negocio:** ¿qué hacemos si el plato se agotó entre el clic y la confirmación? Respuesta típica: el restaurante confirma o rechaza el pedido (¡el proceso de negocio ya funcionaba así con teléfono!) → el rechazo es la **compensación** de una pequeña saga: `PedidoCreado` → restaurante acepta/rechaza → `PedidoConfirmado`/`PedidoRechazado` (+ reembolso si ya se cobró).
- **Precio:** política explícita, p. ej. "vale el precio mostrado al cliente si la diferencia es < X%" o "el precio se fija al confirmar el restaurante".
- **Lección:** la consistencia eventual se gestiona con **semántica de negocio** (confirmaciones, compensaciones, políticas), no intentando recrear la transacción global.

</details>

---

### Ejercicio 3.4 — Diseñar una saga

En **ViajesTotal** se vende un paquete: vuelo + hotel + cobro. Servicios: `reservas` (el proceso), `vuelos`, `hoteles`, `pagos`. Reglas de negocio: no cobrar si no hay vuelo y hotel; los hoteles permiten cancelación gratuita en 24 h; la aerolínea cobra 10 € por cancelación de reserva; un paquete sin pago completado debe deshacerse entero.

**Tareas:**
1. Diseña la saga con **orquestación**: pasos, transacciones locales y compensaciones. ¿En qué orden pones los pasos y por qué?
2. ¿Qué pasa exactamente si falla el cobro?
3. Esboza la alternativa **coreografiada** y di cuál elegirías aquí y por qué.
4. La compensación de la aerolínea cuesta 10 €. ¿Quién "paga" eso y qué te dice sobre las compensaciones en general?

<details>
<summary>💡 Solución</summary>

**1. Saga orquestada** (orquestador: `reservas`):

| Paso | Transacción local | Compensación |
|---|---|---|
| 1. `reservas` | Crear reserva en estado `PENDIENTE` | Marcar `CANCELADA` |
| 2. `hoteles` | Reservar habitación | Cancelar reserva de hotel (gratis < 24 h) |
| 3. `vuelos` | Reservar plaza | Cancelar plaza (cuesta 10 €) |
| 4. `pagos` | Cobrar el paquete | Reembolsar |
| 5. `reservas` | Marcar `CONFIRMADA` | — |

**Orden — el criterio es minimizar el coste esperado de compensar:** se ponen antes los pasos **baratos de compensar** y al final los caros/los más propensos a fallar. El hotel (cancelación gratis) va antes que el vuelo (10 €): si el vuelo falla, cancelar el hotel no cuesta nada; en el orden inverso, un fallo de hotel costaría 10 € de la aerolínea. El **cobro va lo más tarde posible**: reembolsar es lento, genera costes (comisiones) y fricción con el cliente; además la regla de negocio dice "no cobrar sin vuelo y hotel". (Ordenar también por probabilidad de fallo si se conoce: lo que más falla, cuanto antes.)

**2. Si falla el cobro (paso 4):** el orquestador ejecuta las compensaciones en orden inverso: cancelar vuelo (asumiendo los 10 €), cancelar hotel (gratis), marcar la reserva `CANCELADA` (no borrarla: queda el rastro y el motivo). El cliente recibe la notificación correspondiente. Detalles importantes: las compensaciones también pueden fallar → reintentos e idempotencia; y si una compensación no progresa, alerta para intervención manual (las sagas necesitan observabilidad — el orquestador facilita esto porque el estado del proceso está en un solo sitio).

**3. Coreografía:** `reservas` emite `ReservaSolicitada` → `hoteles` reserva y emite `HotelReservado` → `vuelos` escucha, reserva y emite `VueloReservado` → `pagos` escucha y emite `PagoCompletado` → `reservas` confirma. Los fallos se propagan con eventos (`PagoFallido` → vuelos y hoteles escuchan y cancelan…).
**Elección aquí: orquestación.** Razones: proceso de varios pasos **con orden de negocio significativo** (hotel antes que vuelo por costes — en coreografía ese orden queda implícito y frágil, repartido en qué-escucha-a-qué); compensaciones con costes que exigen visibilidad ("¿cuántas reservas se quedaron a medias y cuánto nos costó?"); y `reservas` ya es el dueño natural del proceso. La coreografía brillaría si fuese una simple cascada de reacciones sin orden crítico.

**4. Los 10 €:** los paga el negocio (margen del paquete) o se repercuten según términos y condiciones — pero la decisión es **de negocio, no técnica**. Lección general: **las compensaciones no son rollbacks gratuitos**; son acciones de negocio con coste, consecuencias y a veces imposibilidades (no se puede "descancelar"). Diseñar una saga = diseñar el proceso de negocio incluyendo sus caminos de fallo, con negocio en la mesa. Por eso el orden de los pasos era una decisión de dinero, no de estilo.

</details>

---

### Ejercicio 3.5 — ¿Dónde falla esto? (outbox e idempotencia)

El servicio `pedidos` hace esto al crear un pedido:

```
1. BEGIN TX
2.   INSERT INTO pedidos (...)
3. COMMIT
4. broker.publish("PedidoCreado", pedido)   // publica al broker de eventos
```

Y el consumidor en `facturacion`:

```
on PedidoCreado(evento):
    INSERT INTO facturas (pedido_id, importe, ...)
```

**Tareas:**
1. Encuentra los dos fallos de fiabilidad (uno en el productor, uno en el consumidor) y describe el escenario concreto en que cada uno causa daño.
2. Corrige ambos con los patrones de la sesión (pseudocódigo o prosa).

<details>
<summary>💡 Solución</summary>

**1a. Fallo del productor — "commit sin publicar":** entre el paso 3 y el 4 el proceso puede morir (crash, deploy, OOM) o el broker puede estar caído. Resultado: el pedido **existe** pero el evento **no se publicó nunca** → facturación no se entera → pedido sin factura, saga colgada, y nadie lo detecta porque no hay error visible. (El orden inverso —publicar antes del commit— es igual de malo: evento publicado de un pedido cuya transacción luego falla.) La raíz: no existe transacción atómica que cubra BD + broker.

**1b. Fallo del consumidor — duplicados:** los brokers (y el outbox que vamos a añadir) entregan **al-menos-una-vez**: tras un fallo de red o un timeout de confirmación, el mismo `PedidoCreado` puede llegar dos veces. El consumidor hace un INSERT a ciegas → **factura duplicada** (y al cliente se le cobra dos veces). 

**2. Correcciones:**

**Productor — patrón Outbox:**

```
1. BEGIN TX
2.   INSERT INTO pedidos (...)
3.   INSERT INTO outbox (event_id, tipo='PedidoCreado', payload, publicado=false)
4. COMMIT                          -- atómico: o las dos filas o ninguna

-- proceso relay (aparte, en bucle):
   leer outbox WHERE publicado=false (en orden)
   broker.publish(evento)
   marcar publicado=true
   -- si el relay muere entre publish y marcar → reenvía: al-menos-una-vez
```

Ahora es imposible el estado "pedido sin evento": si la TX confirmó, el evento está en la outbox y el relay lo acabará publicando (reintentando lo que haga falta).

**Consumidor — idempotencia:** procesar el mismo evento N veces debe equivaler a procesarlo una. Dos formas habituales:

```
on PedidoCreado(evento):
    -- opción A: deduplicación por id de evento
    si existe processed_events[evento.event_id]: return
    BEGIN TX
      INSERT INTO facturas (...)
      INSERT INTO processed_events (evento.event_id)
    COMMIT

    -- opción B: idempotencia natural por clave de negocio
    INSERT INTO facturas (pedido_id, ...) ON CONFLICT (pedido_id) DO NOTHING
    -- (una factura por pedido: la restricción única hace el trabajo)
```

Nótese que en la opción A la marca de "procesado" va **en la misma transacción** que el efecto — si no, reaparece el mismo problema del productor en miniatura. La pareja **outbox (productor) + idempotencia (consumidor)** es el estándar de fiabilidad de las arquitecturas de eventos.

</details>

---

## Resumen de la sesión

- El corte correcto sigue al **negocio**: bounded contexts con su lenguaje ubicuo; un mismo término con significados distintos marca una frontera.
- **Context maps** para gobernar las relaciones entre contextos; **core/supporting/generic** para repartir el esfuerzo (lo genérico se compra, no se migra).
- Granularidad: ante la duda, **cortar grueso** — dividir luego es barato, fusionar es caro.
- Sin **BD por servicio** no hay despliegue independiente; partir datos = asignar propiedad, separar esquemas, eliminar cruces (API o copia por eventos), separar físicamente.
- Sin transacciones distribuidas: **consistencia eventual** gestionada con semántica de negocio, **sagas** (orquestación vs. coreografía) con compensaciones que son acciones de negocio con coste, **outbox** para publicar de forma fiable e **idempotencia** en los consumidores.

**Próxima sesión:** comunicación entre servicios, resiliencia, observabilidad, pruebas y el lado organizativo (Conway, Team Topologies) — más la síntesis final en una hoja de ruta.
