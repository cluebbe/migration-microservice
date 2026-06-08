# Workshop: Migrating Monoliths to Microservices
# Formación: Migración de Monolitos a Microservicios

**Trainer / Formador:** Carlos Georg Lübbe
**Format / Formato:** Remote · 10–12 h · ~15 participants

> 🇬🇧 English in plain text · 🇪🇸 Spanish in quoted blocks.

---

## 1. About me / Sobre mí

🇬🇧 I'm **Carlos Georg Lübbe**, a Software Architect & Multistack Engineer with over **17 years of experience** in software development and architecture, and a PhD in Computer Science (*Dr. rer. nat.*, University of Stuttgart, distributed database systems). I'm based in **Madrid**.

> 🇪🇸 Soy **Carlos Georg Lübbe**, Software Architect & Multistack Engineer con más de **17 años de experiencia** en desarrollo y arquitectura de software, y doctor en informática (*Dr. rer. nat.*, Universidad de Stuttgart, sistemas de bases de datos distribuidas). Resido en **Madrid**.

🇬🇧 Much of my career has been in the **automotive and commercial-vehicle sector**, leading international teams and designing architectures on long-running projects. I bring **direct, hands-on experience migrating monolithic applications toward microservice architectures**, from both a **development** and (complementarily) an **infrastructure** perspective:

> 🇪🇸 Buena parte de mi trayectoria se ha desarrollado en el sector de **automoción y vehículo industrial**, liderando equipos internacionales y diseñando arquitecturas en proyectos de larga duración. Aporto **experiencia práctica directa en la migración de aplicaciones monolíticas hacia arquitecturas de microservicios**, tanto desde la perspectiva de **desarrollo** como, de forma complementaria, de **infraestructura**:

| 🇬🇧 English | 🇪🇸 Español |
|---|---|
| **Migrating monoliths to the cloud and to microservices**: migrated an Angular/J2EE application to Cloud Foundry, defined CI/CD pipelines (Maven/Jenkins), supported microservice deployment. | **Migración de monolitos a la nube y a microservicios**: migración de una aplicación Angular/J2EE a Cloud Foundry, definición de pipelines CI/CD (Maven/Jenkins) y soporte al despliegue de microservicios. |
| **Designing Spring Boot / Angular architectures in cloud (PaaS) environments**, focusing on microservice scalability. | **Diseño de arquitecturas Spring Boot / Angular en entornos cloud (PaaS)**, con foco en la escalabilidad de microservicios. |
| **Integrating external microservices via REST APIs** in distributed systems (e.g. an emissions-forecasting system). | **Integración de microservicios externos vía REST APIs** en sistemas distribuidos (p. ej. un sistema de previsión de emisiones). |
| **Legacy migration and refactoring**: Eclipse 3 → E4 migration, library replacement, build-process restructuring, data-access-layer refactoring, database migrations (DB2 → MySQL → MS SQL). | **Migraciones y refactorizaciones de legacy**: migración Eclipse 3 → E4, sustitución de librerías, reestructuración del proceso de build, refactorización de la capa de acceso a datos, migraciones de base de datos (DB2 → MySQL → MS SQL). |
| **Integration patterns and interfaces**: HTTP, REST, SOAP, JAX-RS/WS, Swagger/OpenAPI, messaging (RabbitMQ, WebSocket). | **Patrones de integración e interfaces**: HTTP, REST, SOAP, JAX-RS/WS, Swagger/OpenAPI, mensajería (RabbitMQ, WebSocket). |
| **Security in distributed architectures**: OAuth 2.0 / OpenID, Spring Security, Keycloak. | **Seguridad en arquitecturas distribuidas**: OAuth 2.0 / OpenID, Spring Security, Keycloak. |
| **Infrastructure and DevOps**: Docker, Kubernetes, Cloud Foundry, AWS, Jenkins, nginx. | **Infraestructura y DevOps**: Docker, Kubernetes, Cloud Foundry, AWS, Jenkins, nginx. |

🇬🇧 Since 2014 I have combined consulting and development with **technical training**. I currently deliver online training on Angular, Java, Fullstack, Kubernetes, Docker and cloud development (velpTEC GmbH), as well as IT management and architecture modules at university level (manQ Akademie). This lets me combine real project experience with clear, practice-oriented teaching.

> 🇪🇸 Desde 2014 compagino la consultoría y el desarrollo con la **formación técnica**. Actualmente imparto formaciones online sobre Angular, Java, Fullstack, Kubernetes, Docker y desarrollo en la nube (velpTEC GmbH), así como módulos de gestión y arquitectura IT a nivel universitario (manQ Akademie). Esto me permite combinar la experiencia real de proyecto con una didáctica clara y orientada a la práctica.





---

## 2. Proposed outline / Temario propuesto

🇬🇧 Based on the client's draft. I can adapt depth and timing to the group's level and goals.
> 🇪🇸 Basado en el borrador del cliente. Puedo adaptar la profundidad y los tiempos al nivel y los objetivos del grupo.

| 🇬🇧 English | 🇪🇸 Español |
|---|---|
| **Introduction** — Monoliths vs. microservices · Motivations and challenges of migration · Why migration patterns matter | **Introducción** — Monolitos vs. microservicios · Motivaciones y retos de la migración · Importancia de los patrones de migración |
| **Migration patterns** — Incremental migration · Risks and general best practices | **Patrones de migración** — Migración incremental · Riesgos y buenas prácticas generales |
| **Strangler Pattern** — Concept and origin · Architecture and how it works · Pros and cons · Example | **Patrón Estrangulador** — Concepto y origen · Arquitectura y funcionamiento · Ventajas y limitaciones · Ejemplo |
| **Anti-Corruption Layer** — Concept and relation to DDD · The translation layer · Benefits and limitations · Example | **Capa Anticorrupción** — Concepto y relación con DDD · La capa de traducción · Beneficios y limitaciones · Ejemplo |
| **Domain-Driven Design** — DDD principles · Bounded contexts · Microservices by business domain · Example | **Domain-Driven Design** — Principios de DDD · Contextos delimitados · Microservicios por dominio de negocio · Ejemplo |
| **Pattern comparison** | **Comparativa de patrones** |

---

## 3. Scoping questions / Preguntas de scoping

🇬🇧 To tailor content, examples and level to the group's real needs.
> 🇪🇸 Para ajustar el contenido, los ejemplos y el nivel a las necesidades reales del grupo.

### Audience & goals / Perfil y objetivos

| 🇬🇧 English | 🇪🇸 Español |
|---|---|
| What is the **audience profile** (developers, architects, tech leads, DevOps, mixed)? What seniority predominates? | ¿Cuál es el **perfil de los asistentes** (desarrolladores, arquitectos, tech leads, DevOps, mixto)? ¿Qué nivel de experiencia predomina? |
| Do they already have **prior microservices experience**, or mainly a monolithic background? | ¿Tienen ya **experiencia previa con microservicios**, o parten de un contexto monolítico? |
| What is the **main goal**: leveling up the team conceptually, preparing for a concrete migration, or both? | ¿Cuál es el **objetivo principal**: nivelar al equipo conceptualmente, preparar una migración concreta, o ambas cosas? |

### Tech stack & context / Stack y contexto técnico

| 🇬🇧 English | 🇪🇸 Español |
|---|---|
| What is your **current stack** (the monolith's language/framework: Java/Spring, .NET, Node, PHP, other)? | ¿Cuál es su **stack actual** (lenguaje/framework del monolito: Java/Spring, .NET, Node, PHP, otro)? |
| Which **microservice and infrastructure technologies** do you use or plan to (Spring Boot, Docker, Kubernetes, cloud — AWS/Azure/GCP/on-prem)? | ¿Qué **tecnologías de microservicios e infraestructura** usan o prevén usar (Spring Boot, Docker, Kubernetes, cloud — AWS/Azure/GCP/on-premise)? |
| Do you use **messaging/events** (Kafka, RabbitMQ) or mostly synchronous communication (REST/gRPC)? | ¿Usan **mensajería/eventos** (Kafka, RabbitMQ) o comunicación principalmente síncrona (REST/gRPC)? |
| What **CI/CD and observability** tooling is in place? | ¿Qué herramientas de **CI/CD y observabilidad** tienen en marcha? |
| Do they already work with **DDD** (bounded contexts, ubiquitous language), or would it be their first contact? | ¿Manejan ya conceptos de **DDD** (contextos delimitados, lenguaje ubicuo) o sería su primer contacto? |

### Concrete migration project / Proyecto concreto de migración

🇬🇧 If there is a **concrete migration project** behind the training, these questions help tailor the examples and validate that splitting is actually justified:
> 🇪🇸 Si hay un **proyecto concreto de migración** detrás de la formación, estas preguntas ayudan a adaptar los ejemplos y a validar que la división esté realmente justificada:

| 🇬🇧 English | 🇪🇸 Español |
|---|---|
| Which **technologies** does the monolith and its surrounding ecosystem use (language/framework, database, integrations, deployment)? | ¿Qué **tecnologías** usan el monolito y su ecosistema (lenguaje/framework, base de datos, integraciones, despliegue)? |
| What are the **key problems** with this monolith today (maintainability, coupling, slow releases, scaling, team bottlenecks)? | ¿Cuáles son los **problemas clave** del monolito hoy (mantenibilidad, acoplamiento, releases lentas, escalado, cuellos de botella entre equipos)? |
| Are there concrete **performance issues** (latency, throughput, resource limits), and are they localized to specific modules? | ¿Existen **problemas de rendimiento** concretos (latencia, throughput, límites de recursos), y están localizados en módulos concretos? |
| Does it make sense to **split into several microservices**, or could it be migrated as a whole (re-platformed/modularized)? The effort of splitting must be **justified** — e.g. by independent **scale-out** requirements, separate release cadences, or clear bounded contexts. | ¿Tiene sentido **dividir en varios microservicios**, o podría migrarse como un todo (re-platforming/modularización)? El esfuerzo de dividir debe estar **justificado** —p. ej. por requisitos de **escalado horizontal** independiente, cadencias de release separadas o contextos delimitados claros. |

### Focus & approach / Foco y enfoque

| 🇬🇧 English | 🇪🇸 Español |
|---|---|
| Should the focus be mainly **development**, or do you want real weight on **infrastructure** (deployment, containers, orchestration, networking/security)? | ¿El foco debe ser principalmente de **desarrollo**, o quieren dar peso real a la **infraestructura** (despliegue, contenedores, orquestación, redes/seguridad)? |
| Do you prefer a more **conceptual/architectural** approach or more **hands-on** with exercises and live coding? | ¿Prefieren un enfoque más **conceptual/arquitectónico** o más **práctico (hands-on)** con ejercicios y código en vivo? |
| If we include practice: will participants have an **environment and permissions** (IDE, Docker, cluster/sandbox access), or should I prepare a self-contained one? | Si incluimos práctica: ¿los participantes dispondrán de **entorno y permisos** (IDE, Docker, acceso a clúster/sandbox), o conviene que prepare uno autocontenido? |

### Logistics / Logística

| 🇬🇧 English | 🇪🇸 Español |
|---|---|
| On **which calendar days** should the workshop take place? **Note:** from **10–30 June** I'll be in **South America** (approx. **6 h time difference** to Spain), so any sessions in that window should be scheduled for the **afternoon** (Spanish time). | ¿En **qué días concretos** debería celebrarse la formación? **Nota:** del **10 al 30 de junio** estaré en **Sudamérica** (unas **6 h de diferencia horaria** con España), así que cualquier sesión en ese periodo debería programarse **por la tarde** (hora española). |
| How will the **10–12 hours** be split (e.g. 2 days, 3–4 half-day sessions)? Which dates and time slot? | ¿Cómo se repartirán las **10–12 horas** (p. ej. 2 días, 3–4 sesiones de media jornada)? ¿Qué fechas y franja horaria? |
| Which **language** is the delivery in: Spanish, English, or either? | ¿En qué **idioma** se imparte: español, inglés o indistinto? |
| Which **video platform** do you use (Teams, Zoom, Google Meet)? | ¿Qué **plataforma** de videoconferencia usan (Teams, Zoom, Google Meet)? |
| Do you expect **supporting material** (slides, sample repo, exercises) | ¿Esperan **material de apoyo** (diapositivas, repositorio de ejemplos, ejercicios) |

### Scope of the outline / Alcance del temario

🇬🇧 Is the outline **fixed**, or is there room to complement it? Topics that could add value depending on the case:
> 🇪🇸 ¿El temario es **cerrado**, o hay margen para complementarlo? Temas que podrían aportar valor según el caso:

| 🇬🇧 English | 🇪🇸 Español |
|---|---|
| **More migration patterns**: the draft covers Strangler, ACL and DDD. Would it add value to also cover the broader landscape — Big-Bang Rewrite, Branch by Abstraction, Parallel Run, Bubble Context — so the "pattern comparison" gives a complete picture of the options and trade-offs? | **Más patrones de migración**: el borrador cubre Strangler, ACL y DDD. ¿Aportaría valor cubrir también el panorama completo —reescritura Big-Bang, Branch by Abstraction, Parallel Run, Bubble Context— para que la "comparativa de patrones" ofrezca una visión completa de las opciones y sus compensaciones? |
| **Data decomposition**: shared database → database per service, *Saga* pattern, eventual consistency. | **Descomposición de datos**: base de datos compartida → base de datos por servicio, patrón *Saga*, consistencia eventual. |
| **Inter-service communication**: sync vs. async, API Gateway, service discovery. | **Comunicación entre servicios**: síncrona vs. asíncrona, API Gateway, descubrimiento de servicios. |
| **Resilience & observability**: circuit breaker, retries, timeouts, distributed tracing. | **Resiliencia y observabilidad**: circuit breaker, reintentos, timeouts, trazado distribuido. |
| **Organizational aspects**: Conway's Law, Team Topologies, service ownership. | **Aspectos organizativos**: ley de Conway, Team Topologies, propiedad de los servicios. |
