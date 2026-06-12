# Sesión 1 — Fundamentos y la justificación de migrar

**Duración:** 3 h · **Objetivo:** establecer un vocabulario común y un encuadre claro y honesto de cuándo (y cuándo no) migrar de un monolito a microservicios.

---

## 1. Introducción: monolitos vs. microservicios

### 1.1 ¿Qué es un monolito?

Un **monolito** es una aplicación cuya unidad de despliegue es única: todo el código se compila, empaqueta y despliega junto. Es importante separar dos ideas que a menudo se confunden:

- **Monolito como unidad de despliegue:** un solo artefacto (un WAR, un binario, una imagen).
- **Monolito como unidad de organización del código:** todo el código vive en un mismo proyecto, con (o sin) estructura interna.

Un monolito **no es intrínsecamente malo**. Tiene ventajas reales:

| Ventaja | Por qué |
|---|---|
| Simplicidad operativa | Un solo proceso que desplegar, monitorizar y escalar |
| Transacciones ACID | Una sola base de datos: consistencia "gratis" |
| Refactorización fácil | El IDE ve todo el código; renombrar es seguro |
| Latencia mínima | Las llamadas internas son invocaciones de método, no red |
| Depuración sencilla | Un stack trace cuenta toda la historia |

Sus problemas aparecen con la **escala** (del sistema y de la organización):

- **Acoplamiento creciente:** sin disciplina, todo acaba dependiendo de todo ("big ball of mud").
- **Despliegues acoplados:** un cambio pequeño exige redesplegar todo; el riesgo de cada despliegue crece y la frecuencia baja.
- **Escalado grueso:** si solo un módulo necesita más recursos, hay que escalar la aplicación entera.
- **Cuellos de botella organizativos:** muchos equipos tocando el mismo código → conflictos, colas de integración, "trenes de release".
- **Atadura tecnológica:** una sola pila tecnológica para todos los problemas.

### 1.2 ¿Qué es un microservicio?

Un **microservicio** es un servicio desplegable de forma independiente, modelado alrededor de un **dominio de negocio**, que se comunica con otros servicios a través de la red y **oculta sus datos** tras una interfaz.

Las propiedades clave (más importantes que el tamaño):

1. **Despliegue independiente** — puedo cambiar y desplegar el servicio A sin coordinarme con el servicio B. Esta es *la* propiedad definitoria.
2. **Modelado en torno al negocio** — los límites siguen el dominio (pedidos, facturación), no las capas técnicas (UI, lógica, datos).
3. **Propiedad de sus datos** — ningún otro servicio accede directamente a su base de datos.
4. **Autonomía del equipo** — un equipo puede ser dueño del ciclo de vida completo del servicio.

> **Idea fuerza:** "micro" no significa "pequeño en líneas de código", significa "pequeño en superficie de acoplamiento". La pregunta correcta no es "¿cuántas líneas?" sino "¿puedo desplegarlo sin pedir permiso?".

### 1.3 Lo que se gana y lo que se paga

Los microservicios **intercambian complejidad local por complejidad distribuida**:

| Se gana | Se paga |
|---|---|
| Despliegue independiente, releases frecuentes | Sistema distribuido: latencia, fallos parciales |
| Escalado selectivo por servicio | Operación mucho más compleja (orquestación, observabilidad) |
| Autonomía de equipos | Necesidad de contratos y versionado entre servicios |
| Aislamiento de fallos (potencial) | Consistencia de datos sin transacciones globales |
| Libertad tecnológica por servicio | Coste de plataforma (CI/CD, descubrimiento, gateway…) |

Las famosas **falacias de la computación distribuida** (Deutsch) resumen lo que deja de ser cierto al pasar por la red: la red *no* es fiable, la latencia *no* es cero, el ancho de banda *no* es infinito, la red *no* es segura, la topología *cambia*…

---

## 2. Más allá de la dicotomía: el espectro arquitectónico

No hay que elegir entre "monolito caótico" y "200 microservicios". Existe un **espectro**:

```
Big ball of mud → Monolito estructurado → Monolito modular → SOA / servicios gruesos → Microservicios
```

### 2.1 El monolito modular

Un **monolito modular** mantiene una sola unidad de despliegue, pero el código está organizado en **módulos con límites explícitos**:

- Cada módulo expone una **interfaz pública pequeña** y oculta su interior.
- Los módulos se comunican solo a través de esas interfaces (idealmente verificado por herramientas).
- Cada módulo puede tener **su propio esquema de datos** lógico (aunque compartan instancia de BD).

El monolito modular es:

- Un **destino válido** en sí mismo: muchas organizaciones obtienen el 80 % del beneficio (límites claros, equipos con propiedad) sin pagar el coste de la distribución.
- El **mejor punto de partida** para una futura migración: si los límites de módulo son buenos, extraer un módulo a servicio es mucho más barato.

> **Regla práctica:** si no eres capaz de construir un monolito modular bien estructurado, los microservicios no van a arreglarlo — van a distribuir el desorden y añadirle red.

### 2.2 ¿Dónde situarse en el espectro?

Depende de:

- **Tamaño y número de equipos** (el factor más determinante).
- **Necesidades de escalado diferencial** entre partes del sistema.
- **Madurez operativa** (CI/CD, observabilidad, automatización).
- **Velocidad de cambio requerida** y en qué partes del sistema se concentra.

---

## 3. Justificar la división

### 3.1 Empezar por el porqué: objetivos y criterios de éxito

Migrar a microservicios es un **medio**, nunca un fin. Antes de empezar hay que poder completar esta frase:

> "Migramos porque queremos conseguir **X**, lo mediremos con **Y**, y sabremos que hemos terminado/avanzado cuando **Z**."

Motivaciones legítimas habituales y su métrica natural:

| Objetivo | Métrica de éxito posible |
|---|---|
| Acelerar la entrega | Frecuencia de despliegue, lead time de cambios |
| Escalar la organización | Nº de equipos que entregan de forma autónoma |
| Escalar técnicamente una parte | Coste de infraestructura, latencia bajo carga |
| Aislar fallos | Radio de impacto de incidentes, MTTR |
| Modernizar tecnología | % de tráfico servido por la nueva pila |

Si el objetivo es difuso ("modernizar", "es lo que hace todo el mundo"), la migración no tendrá criterio de parada ni forma de priorizar.

### 3.2 Cuándo *no* merece la pena migrar

Señales de que los microservicios son probablemente una mala idea ahora:

- **Equipo pequeño** (p. ej. < 10–15 personas): el coste operativo se come la ganancia de autonomía.
- **Dominio aún poco entendido o producto en búsqueda de mercado:** los límites cambiarán; mover límites entre servicios es carísimo, dentro de un monolito es barato.
- **El problema real es otro:** falta de tests, despliegues manuales, deuda de calidad. La distribución no arregla nada de eso; lo amplifica.
- **Madurez operativa insuficiente:** sin CI/CD, sin observabilidad, sin automatización de infraestructura, operar N servicios es inviable.
- **Producto que va a ser reemplazado o tiene vida corta.**

Alternativas que conviene considerar **antes** de dividir:

- **Re-platforming:** mover el monolito tal cual a mejor infraestructura (contenedores, nube, autoescalado). A menudo resuelve los problemas de escalado.
- **Modularización interna:** convertirlo en monolito modular. Resuelve los problemas de acoplamiento y de equipos.
- **Extraer solo 1–2 servicios** donde haya un dolor concreto (p. ej. el módulo que necesita escalar), y parar.

### 3.3 El anti-patrón del monolito distribuido

El **monolito distribuido** es el peor resultado posible: muchos servicios que **deben desplegarse juntos** porque siguen acoplados.

Síntomas:

- Un cambio funcional típico toca 3–4 servicios a la vez.
- Los despliegues se coordinan en "trenes": "el viernes desplegamos todos".
- Servicios que comparten base de datos.
- Cadenas largas de llamadas síncronas: la caída de uno tumba a todos.
- Contratos que cambian sin versionado: hay que actualizar consumidores y productor al unísono.

Causas raíz típicas: dividir por capas técnicas en lugar de por dominio, dividir antes de entender el dominio, no romper la base de datos compartida.

> **Test rápido:** si para entregar una funcionalidad media necesitas coordinar despliegues de varios servicios, no tienes microservicios; tienes un monolito con latencia de red.

---

## 4. Fundamentos de estrategia de migración

### 4.1 Incremental vs. big-bang

- **Big-bang:** reescribir el sistema completo y cambiar de golpe. Casi siempre fracasa en sistemas no triviales: el monolito sigue evolucionando mientras reescribes (objetivo móvil), no entregas valor en meses/años, y el riesgo se concentra en un único evento de corte. (Lo analizaremos como patrón en la Sesión 2.)
- **Incremental:** extraer funcionalidad pieza a pieza, manteniendo el sistema funcionando en todo momento. El monolito y los servicios **conviven durante años**, y eso no es un fallo del plan: *es* el plan.

Principios del enfoque incremental:

1. **Pasos pequeños y reversibles.** Cada paso debe poder deshacerse (rollback) sin drama.
2. **Entregar valor por el camino.** Cada extracción debe justificar su coste por sí misma.
3. **El sistema funciona siempre.** Nunca hay una fase "ahora estamos rotos durante 3 meses".
4. **Aprender y ajustar.** Las primeras extracciones enseñan; el plan se revisa.

### 4.2 Gestión del riesgo

- **Empieza por algo que enseñe mucho y arriesgue poco:** un módulo razonablemente desacoplado, con valor visible, pero no el corazón crítico del negocio.
- **Separa el despliegue de la activación:** poder desplegar el servicio nuevo sin enviarle tráfico real (feature flags, enrutado gradual) reduce el riesgo de cada paso.
- **Mide antes y después:** sin línea base (rendimiento, errores, lead time) no podrás demostrar mejora ni detectar regresión.
- **Plan de retirada del legacy:** la migración no termina cuando el servicio nuevo funciona, sino cuando el código antiguo **se borra**. El "modo híbrido eterno" es un coste permanente.

### 4.3 Cómo secuenciar una migración

Criterios para decidir **qué extraer primero**:

| Criterio | Favorece extraer pronto |
|---|---|
| Acoplamiento | Módulos con pocas dependencias entrantes/salientes |
| Valor | Donde la autonomía/escalado aporta beneficio medible |
| Volatilidad | Código que cambia mucho (la independencia de despliegue rinde más) |
| Riesgo | Funcionalidad no crítica para los primeros pasos |
| Datos | Módulos con datos poco compartidos con el resto |

Herramienta útil: una matriz **valor de extraer × facilidad de extraer**. Empezar por el cuadrante alto-valor/alta-facilidad; cuestionar si el cuadrante bajo-valor/baja-facilidad debe extraerse jamás.

### 4.4 Buenas prácticas generales

- Trata la migración como **producto**, no como proyecto big-bang: backlog, prioridades, métricas, revisión continua.
- **No congeles el monolito** salvo en las zonas en extracción activa; el negocio no puede parar.
- Invierte pronto en la **plataforma** (CI/CD, observabilidad, plantillas de servicio): el segundo y tercer servicio deben ser mucho más baratos que el primero.
- Comunica el **estado de la convivencia**: qué vive ya en servicios, qué sigue en el monolito, qué está a medias.

---

## 5. El panorama de patrones (mapa de la formación)

Los patrones que veremos se agrupan así:

```
┌─ Patrones de MIGRACIÓN (Sesión 2) ────────────────────────────┐
│  ¿Cómo muevo funcionalidad del monolito a servicios?          │
│  · Strangler Fig        · Branch by Abstraction               │
│  · Anticorruption Layer · Parallel Run                        │
│  · Bubble Context       · (Big-Bang, como contraste)          │
└───────────────────────────────────────────────────────────────┘
┌─ Patrones de DESCOMPOSICIÓN (Sesión 3) ───────────────────────┐
│  ¿Dónde corto? ¿Qué hago con los datos?                       │
│  · DDD: bounded contexts, lenguaje ubicuo, context maps       │
│  · BD por servicio · Saga · Outbox · Consistencia eventual    │
└───────────────────────────────────────────────────────────────┘
┌─ Patrones de OPERACIÓN del resultado (Sesión 4) ──────────────┐
│  ¿Cómo hago robusto el sistema distribuido resultante?        │
│  · Sync/async, eventos · API Gateway · Descubrimiento         │
│  · Circuit breaker, retries, timeouts · Observabilidad        │
│  · Conway, Team Topologies                                    │
└───────────────────────────────────────────────────────────────┘
```

Relación entre grupos: los patrones de **descomposición** dicen *dónde* cortar; los de **migración** dicen *cómo* ejecutar el corte de forma segura; los de **operación** hacen vivible el resultado. Una migración real los combina todos.

---

## Ejercicios

### Ejercicio 1.1 — ¿Monolito culpable o inocente?

La empresa **LibroExpress** (e-commerce de libros, 8 desarrolladores en 1 equipo) tiene un monolito en Java. Se quejan de:

1. Los despliegues son manuales y ocurren una vez al mes, en sábado.
2. No hay tests automatizados; cada release rompe algo.
3. En campaña de Navidad, la web entera va lenta porque el módulo de búsqueda consume toda la CPU.
4. El CTO propone: "Migremos a microservicios, así desplegaremos a diario y escalará mejor."

**Pregunta:** ¿Cuáles de los problemas (1–3) resolverían realmente los microservicios? ¿Qué propondrías tú en su lugar? Justifica con los conceptos de la sesión.

<details>
<summary>💡 Solución</summary>

**Análisis problema a problema:**

1. **Despliegues manuales y mensuales** — No es un problema del monolito, es un problema de **madurez operativa** (falta de CI/CD). Los microservicios lo *empeorarían*: pasarían a tener N despliegues manuales en vez de 1. Se puede desplegar un monolito varias veces al día (muchas empresas lo hacen). Solución: pipeline de CI/CD.

2. **Sin tests, cada release rompe algo** — Problema de **calidad/deuda técnica**. Los microservicios lo amplifican: a los bugs locales se suman los fallos de integración entre servicios, mucho más difíciles de depurar. Solución: invertir en tests automatizados primero. Además, sin tests, cualquier migración (que es una refactorización gigante) sería extremadamente arriesgada.

3. **La búsqueda consume toda la CPU en picos** — Este es el único problema donde la división aporta algo: **escalado selectivo**. Pero hay alternativas más baratas que una migración completa: re-platforming (autoescalado horizontal del monolito entero, que suele ser suficiente), o extraer **solo el servicio de búsqueda** (a menudo además con tecnología especializada tipo motor de búsqueda) y dejar el resto como está.

**Propuesta razonable:**
- Con **8 personas en 1 equipo**, no existe el problema organizativo que los microservicios resuelven (no hay equipos pisándose). El coste operativo de N servicios para un equipo tan pequeño es desproporcionado.
- Orden sugerido: (1) CI/CD y tests → despliegues frecuentes y seguros; (2) modularizar internamente el monolito; (3) si el escalado de búsqueda sigue doliendo, extraer únicamente ese servicio.
- Al CTO: los objetivos "desplegar a diario" y "escalar mejor" son legítimos, pero el medio propuesto no es el más barato para conseguirlos. Definir métricas (frecuencia de despliegue, coste/latencia en picos) y atacar con la intervención mínima.

**Conceptos aplicados:** migrar es un medio y no un fin; cuándo no migrar (equipo pequeño, problema real es otro); re-platforming/modularización como alternativas; extraer 1 servicio y parar.

</details>

---

### Ejercicio 1.2 — Detectar el monolito distribuido

El equipo de **PagoSeguro** migró hace dos años a 12 microservicios. Observas lo siguiente:

- A. Para añadir un campo al formulario de alta de cliente hay que modificar y desplegar `customer-api`, `customer-validation`, `customer-persistence` y `notification-service`.
- B. `customer-persistence` y `billing-service` leen y escriben en la misma base de datos Oracle.
- C. Los despliegues se hacen todos juntos el segundo martes de cada mes ("release train").
- D. `payment-service` se despliega solo, varias veces por semana, sin coordinar con nadie.
- E. Cuando `customer-validation` se cae, el alta de clientes falla por completo con timeout de 30 s.

**Preguntas:**
1. ¿Qué síntomas de monolito distribuido reconoces y por qué?
2. ¿Hay alguna parte sana del sistema?
3. Para los síntomas A y B, ¿cuál es probablemente la causa raíz?

<details>
<summary>💡 Solución</summary>

**1. Síntomas de monolito distribuido:**

- **A — Cambio funcional que atraviesa 4 servicios.** El test rápido falla: un cambio medio exige tocar y desplegar varios servicios coordinadamente. Además, los nombres (`customer-api`, `customer-validation`, `customer-persistence`) delatan que se dividió por **capas técnicas** (API/lógica/persistencia) en lugar de por dominio: cualquier cambio del dominio "cliente" atraviesa todas las capas.
- **B — Base de datos compartida.** Acoplamiento por el esquema: ninguno de los dos servicios puede cambiar sus tablas sin romper al otro. Viola la propiedad de "cada servicio es dueño de sus datos".
- **C — Release train mensual.** La consecuencia directa de A y B: como los servicios están acoplados, nadie se atreve a desplegarlos por separado. Se ha perdido *la* propiedad definitoria de los microservicios (despliegue independiente) pero se paga todo el coste de la distribución.
- **E — Fallo en cascada con timeouts largos.** Acoplamiento síncrono fuerte sin mecanismos de resiliencia (sin circuit breaker, timeout excesivo). Es señal de que la división creó una cadena síncrona obligatoria. (Lo trataremos a fondo en la Sesión 4.)

**2. Parte sana:** **D — `payment-service`**. Se despliega de forma independiente y frecuente: exactamente el comportamiento que define a un microservicio de verdad. Probablemente su límite coincide con un buen límite de dominio (pagos) y posee sus propios datos. Sirve de referencia interna: "queremos que el resto se parezca a este".

**3. Causa raíz de A y B:**

- **A:** descomposición por **capas técnicas** en vez de por **dominio de negocio**. La unidad que cambia junta (el dominio "cliente") quedó repartida en varios servicios. La corrección de fondo sería fusionar esas capas en un único servicio `customer` modelado por dominio — fusionar servicios también es una decisión válida en una migración.
- **B:** la migración dividió el **código** pero nunca dividió los **datos**. La descomposición de datos (BD por servicio) es la parte más difícil y a menudo se aplaza indefinidamente; mientras no se haga, el acoplamiento persiste por debajo. (Sesión 3.)

</details>

---

### Ejercicio 1.3 — Objetivos y criterios de éxito

Tu directora dice: *"Quiero que migremos a microservicios para ser más ágiles."*

**Tarea:** Reformula este encargo en (a) un objetivo concreto, (b) 2–3 métricas medibles con su valor actual hipotético y su valor objetivo, y (c) un criterio explícito de *parada* (cómo sabríais que ya no merece la pena seguir extrayendo servicios). Inventa cifras razonables.

<details>
<summary>💡 Solución (ejemplo de respuesta)</summary>

No hay una única respuesta correcta; lo que se evalúa es la estructura. Un ejemplo:

**(a) Objetivo concreto:**
"Queremos que los 4 equipos de producto puedan entregar cambios en producción de forma autónoma, sin coordinar releases entre sí, reduciendo el tiempo desde commit hasta producción."

**(b) Métricas (línea base → objetivo):**

| Métrica | Hoy | Objetivo (18 meses) |
|---|---|---|
| Frecuencia de despliegue | 1/mes (release train) | ≥ 2/semana por equipo |
| Lead time (commit → producción) | 3 semanas | < 2 días |
| Despliegues que requieren coordinación entre equipos | 100 % | < 10 % |

**(c) Criterio de parada:**
"Dejamos de extraer servicios cuando cada equipo posea los servicios que necesita para entregar de forma autónoma. Si tras extraer un candidato la matriz valor×facilidad solo deja módulos de bajo valor o alta dificultad, esos módulos se quedan en el monolito (modularizado) de forma deliberada y documentada."

**Puntos clave que debería tener cualquier solución:**
- El objetivo habla de un **resultado de negocio/entrega**, no de tecnología ("tener microservicios" no es un objetivo).
- Las métricas tienen **línea base**: sin medir el hoy, no se puede demostrar mejora.
- Existe un **criterio de parada**: una buena migración no implica extraerlo todo; "monolito modular + algunos servicios" es un final perfectamente válido.

</details>

---

### Ejercicio 1.4 — Secuenciar la migración

El monolito de **AseguraTodo** (seguros) tiene estos módulos. Las flechas indican "depende de":

```
Tarificación  ──→  Catálogo de productos
Contratación  ──→  Tarificación, Clientes, Documentos
Siniestros    ──→  Clientes, Documentos, Pagos
Clientes      ──→  (nada)
Documentos    ──→  (nada)        [genera PDFs; consume mucha CPU]
Pagos         ──→  Clientes      [cambia poco; crítico y regulado]
Catálogo      ──→  (nada)        [cambia cada semana por negocio]
```

Datos adicionales: *Documentos* provoca picos de CPU que degradan todo el sistema. *Catálogo* es donde negocio pide cambios constantemente. *Pagos* no ha cambiado en un año. *Contratación* es el corazón del negocio.

**Tarea:** Propón los **dos primeros módulos** a extraer y **uno que dejarías para el final (o nunca)**. Justifica con los criterios de secuenciación (acoplamiento, valor, volatilidad, riesgo, datos).

<details>
<summary>💡 Solución</summary>

**Primeras extracciones recomendadas:**

1. **Documentos** — Candidato casi perfecto:
   - *Acoplamiento:* sin dependencias salientes; lo consumen otros, así que basta exponer una API estable.
   - *Valor:* altísimo y medible — sus picos de CPU degradan todo el monolito; extraído, se escala de forma independiente y el beneficio se nota de inmediato (caso clásico de escalado selectivo).
   - *Datos:* un generador de PDFs suele tener pocos datos propios compartidos.
   - *Riesgo:* no es el corazón del negocio; un fallo es molesto, no catastrófico.

2. **Catálogo** — Segundo candidato fuerte:
   - *Acoplamiento:* sin dependencias salientes; solo Tarificación lo consume.
   - *Volatilidad:* es el módulo que más cambia (cambios semanales de negocio) — es donde la independencia de despliegue rinde más: cada cambio de catálogo dejaría de requerir redesplegar el monolito.
   - *Riesgo:* moderado y acotable.

   (*Clientes* también es defendible por su bajo acoplamiento saliente, pero tiene muchas dependencias entrantes —Contratación, Siniestros, Pagos— y probablemente datos muy compartidos, lo que encarece la extracción. Y su volatilidad parece baja: el beneficio es menor.)

**Dejar para el final (o nunca):**

- **Pagos** — *Volatilidad casi nula* (un año sin cambios): la independencia de despliegue no aporta apenas valor. *Riesgo altísimo* (crítico y regulado): cualquier error en la extracción es un incidente grave. En la matriz valor×facilidad cae en bajo-valor/alto-riesgo: candidato a quedarse en el monolito de forma deliberada.
- **Contratación** también se deja para tarde, pero por la razón opuesta: es el corazón del negocio y el más acoplado (3 dependencias). Conviene extraerlo cuando el equipo ya haya aprendido con extracciones menores y cuando sus dependencias (Clientes, Documentos, Tarificación) ya sean servicios con APIs estables.

**Lección general:** se empieza por módulos *hoja* (sin dependencias salientes), con dolor concreto (Documentos) o alta volatilidad (Catálogo); se aplaza lo crítico-y-estable (Pagos) y lo central-y-acoplado (Contratación).

</details>

---

## Resumen de la sesión

- Monolito y microservicios son **puntos de un espectro**; el monolito modular es un destino válido y el mejor punto de partida.
- Los microservicios compran **despliegue independiente y autonomía de equipos** pagando con **complejidad distribuida**.
- Antes de migrar: objetivo concreto, métricas con línea base, criterio de parada — y considerar re-platforming o modularización como alternativas más baratas.
- El **monolito distribuido** (servicios que deben desplegarse juntos) es el peor resultado posible; se detecta con el test "¿cuántos servicios toca un cambio medio?".
- La migración sana es **incremental**: pasos pequeños y reversibles, valor entregado por el camino, convivencia prolongada de lo viejo y lo nuevo, y borrado final del legacy.

**Próxima sesión:** la caja de herramientas de patrones de migración — Strangler Fig, Anticorruption Layer, Branch by Abstraction, Parallel Run, Bubble Context y la comparativa entre ellos.
