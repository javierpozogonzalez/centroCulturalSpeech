# Speech — Presentación Arquitectura Centro Cultural Municipal
**Duración estimada: 10 minutos**

---

## 1. INTRODUCCIÓN (30 segundos) — Slide 1

Buenos días / Buenas tardes. Antes de nada, queremos agradeceros que nos hayáis dado la oportunidad de trabajar en este proyecto. Habéis llegado a nosotros con una necesidad muy clara: el centro cultural está creciendo, las actividades aumentan cada temporada, y gestionar inscripciones, salas, monitores y notificaciones con hojas de cálculo o sistemas dispersos ya no es viable.

Hoy os venimos a presentar la arquitectura completa de la plataforma que vamos a construir para vosotros — una solución diseñada no solo para resolver vuestros problemas de hoy, sino para crecer con vosotros durante los próximos años.

---

## 2. EL PROBLEMA (30 segundos) — Slide 3

*[Los bullets aparecen uno a uno]*

Como podéis ver aquí, los problemas que identificamos son cinco. El crecimiento de actividades hace la gestión manual inviable. Los picos de demanda a inicio de temporada colapsan cualquier sistema sencillo. Hay riesgo real de overbooking si dos personas intentan la misma plaza a la vez. No existe ningún mecanismo de notificaciones automáticas, lo que genera una carga administrativa enorme. Y el sistema actual vive completamente aislado del resto de herramientas del ayuntamiento.

Estos cinco puntos son los que guiaron cada decisión arquitectónica que vamos a explicaros ahora.

---

## 3. ELECCIÓN DE ARQUITECTURA Y JUSTIFICACIÓN (3 minutos — la parte más importante) — Slides 4 y 5

### Slide 4 — Por qué microservicios

*[La columna de microservicios se ilumina fila a fila]*

Cuando analizamos estos requisitos, la primera opción obvia sería una aplicación web clásica: un solo bloque de código, una sola base de datos. Sencillo de construir, pero con un problema grave: **no escala**.

Imaginad el inicio de temporada en septiembre. En cuestión de horas, cientos de ciudadanos intentan inscribirse simultáneamente en los talleres más populares. Un sistema monolítico colapsaría bajo esa carga. Y si el módulo de notificaciones falla, arrastra consigo a toda la plataforma.

Por eso hemos optado por una **arquitectura orientada a microservicios**. Cada funcionalidad crítica es un servicio independiente: actividades, usuarios, inscripciones, salas, monitores y notificaciones. Si hay un pico de demanda en inscripciones, solo escalamos ese servicio. Y si el servicio de notificaciones tiene un problema puntual, las inscripciones siguen funcionando con total normalidad.

### Slide 5 — Requisitos no funcionales

*[Cada tarjeta se ilumina al hablar de ella]*

Esta decisión arquitectónica no es caprichosa — responde directamente a vuestros cuatro requisitos no funcionales principales:

**Seguridad.** Al ser una plataforma del ayuntamiento, la autenticación se integra directamente con el servicio centralizado de credenciales municipales. El API Gateway — implementado con Nginx — actúa como portero único: valida el token JWT de cada petición antes de que llegue a cualquier servicio. Los ciudadanos no necesitan crear una cuenta nueva; usan lo que ya tienen.

**Rendimiento y concurrencia.** El control de plazas es el punto más crítico. Si dos ciudadanos intentan reservar la última plaza a la vez, el sistema debe garantizar que solo uno lo consigue. Para esto usamos Redis con operaciones atómicas — una operación que el sistema ejecuta de forma indivisible, sin posibilidad de interferencia. Sin esto, podríais tener más inscritos que plazas reales.

**Desacoplamiento — y aquí entra nuestro patrón de diseño.** Hemos implementado el **patrón Observer** con RabbitMQ. Cuando un ciudadano se inscribe, el servicio de inscripciones publica un evento y se olvida. SendGrid recoge ese evento y envía el email de confirmación de forma completamente independiente. Si mañana queréis añadir un nuevo comportamiento automático, solo añadís un nuevo consumidor sin tocar nada de lo existente.

**Escalabilidad futura.** ¿Queréis abrir el sistema a otros centros culturales del municipio? ¿Añadir una app móvil? ¿Integrar con el sistema de becas? Cada uno de estos cambios se puede hacer sin reescribir la plataforma desde cero.

---

## 4. DIAGRAMA DE CONTEXTO — NIVEL 1 (1 minuto) — Slides 6, 7, 8

*[El zoom va de la vista general → actores → sistema central → integraciones externas]*

Para entender el sistema de un vistazo, empezamos por el nivel más alto: el diagrama de contexto.

Aquí vemos el sistema como una caja negra rodeada de todo aquello con lo que interactúa. Tenemos cuatro tipos de usuarios: el **ciudadano**, que consulta y se inscribe en actividades; el **monitor**, que gestiona las actividades que imparte; el **administrador del centro**, que tiene control total sobre la plataforma; y el **responsable de sala**, que supervisa el estado y disponibilidad de su espacio.

Fuera del sistema, el centro cultural no vive aislado. Se conecta con **Stripe** para cobrar matrículas, con **SendGrid** para notificaciones por email y SMS, con el **servicio de autenticación del ayuntamiento**, con **Google Calendar** para que los ciudadanos exporten los horarios de sus actividades, y con el **sistema contable municipal** para el reporting de ingresos.

Este diagrama es importante porque define con claridad el perímetro del sistema: qué construimos nosotros y qué ya existe y simplemente integramos.

---

## 5. DIAGRAMA DE CONTENEDORES — NIVEL 2 (1 minuto 30 segundos) — Slides 9, 10, 11, 12

*[Cada elemento se ilumina en su turno]*

Si hacemos zoom dentro de ese sistema, llegamos al nivel de contenedores — las piezas tecnológicas que lo componen.

Los ciudadanos y el resto de actores acceden a través de una **aplicación web desarrollada en Nuxt**, que consume nuestra API. Nuxt nos da renderizado en servidor, lo que mejora el rendimiento inicial y el SEO — importante para una web pública del ayuntamiento.

Todas las peticiones pasan por un **API Gateway implementado con Nginx**, que actúa como portero: valida la autenticación, aplica límites de uso y enruta cada petición al microservicio correcto.

Detrás encontramos los **cinco microservicios Django/DRF**: Actividades, Usuarios, Inscripciones, Salas y Monitores. Cada uno con su propia base de datos PostgreSQL — esto es fundamental, porque garantiza que un problema en la base de datos de monitores no afecta en absoluto a las inscripciones.

Para las notificaciones usamos **Python con Celery**, que escucha el bus de eventos RabbitMQ y envía confirmaciones via **SendGrid** sin bloquear el flujo principal. Y para el control de plazas en tiempo real usamos **Redis**, que nos da esas operaciones atómicas que mencionábamos antes.

---

## 6. PATRÓN OBSERVER (1 minuto) — Slide 13

*[El flujo se construye paso a paso, bloque a bloque]*

Antes de entrar en el nivel de componentes, quiero explicaros visualmente cómo funciona el patrón Observer que hemos elegido, porque es la decisión de diseño más importante de la arquitectura.

El flujo es el siguiente: el ciudadano se inscribe. El InscripcionService hace su trabajo — valida plazas, procesa el pago con Stripe, persiste en base de datos — y entonces publica un evento llamado "InscripcionCreada" en RabbitMQ. A partir de ahí, el servicio de inscripciones se olvida completamente de lo que pase.

RabbitMQ distribuye ese evento a los consumidores suscritos. En este caso, dos: SendGrid, que manda el email de confirmación al ciudadano, y el AuditLogger, que registra la operación en MongoDB para auditoría.

El punto clave está aquí: el servicio de inscripciones no sabe quién reacciona al evento. Mañana podéis pedir que cuando se complete una actividad se avise automáticamente al monitor, o que se actualice un dashboard en tiempo real. Solo añadimos un nuevo consumidor — sin tocar una sola línea del servicio de inscripciones.

---

## 7. DIAGRAMA DE COMPONENTES — NIVEL 3 (1 minuto 30 segundos) — Slides 14, 15, 16, 17

*[Las tres capas se iluminan una a una, luego el diagrama completo]*

El nivel de componentes nos muestra el interior de cada servicio. Como veis, todos siguen la misma estructura en tres capas que se va iluminando:

La **capa de controladores** — los ViewSets de Django REST Framework — recibe las peticiones HTTP, las valida y comprueba permisos. No contiene lógica de negocio, solo orquesta y delega.

La **capa de servicios de dominio** es donde vive toda la inteligencia del sistema. El InscripcionService, por ejemplo, coordina: primero verifica plazas en Redis, luego comprueba que el ciudadano no tenga otra actividad en ese horario, procesa el pago con Stripe, persiste la inscripción y publica el evento en RabbitMQ.

La **capa de repositorios** abstrae el acceso a datos — las queries a PostgreSQL, las operaciones en Redis. Si en el futuro necesitáramos migrar de base de datos, solo tocaríamos esta capa sin tocar nada de la lógica de negocio.

Esta arquitectura interna consistente en todos los servicios no es casualidad — facilita que cualquier desarrollador pueda trabajar en cualquier servicio sin necesidad de aprender una estructura diferente cada vez.

---

## 8. DIAGRAMA DE CÓDIGO — NIVEL 4 (45 segundos) — Slides 18, 19, 20

*[Cada paso del flujo se ilumina en orden, siguiendo las flechas]*

En el nivel más detallado, hemos modelado el flujo más crítico del sistema: la inscripción de un ciudadano.

Hemos centrado este diagrama en este flujo concreto porque es donde confluyen más reglas de negocio simultáneamente. Podéis ver cómo el InscripcionViewSet recibe la petición, el Serializer valida los datos, el InscripcionService orquesta todo el proceso, los validadores comprueban plazas en Redis y conflictos de horario, el PaymentAdapter — implementando el patrón Adapter — procesa el cobro con Stripe, el Repository persiste en PostgreSQL, y finalmente el EventPublisher lanza el evento a RabbitMQ para que SendGrid envíe la confirmación.

---

## 9. BASE DE DATOS (1 minuto) — Slide 21

*[Zoom a cada entidad con su relación]*

El modelo de datos refleja exactamente las entidades que nos trasladasteis.

La entidad central es **Actividad**, que concentra toda la información de cada actividad cultural: nombre, tipo, horario, duración, plazas disponibles. Cada actividad tiene asignado un **Monitor** — relación uno a muchos — y una **Sala**, que a su vez tiene un **ResponsableSala** en relación uno a uno.

La tabla **Inscripcion** une al **UsuarioInscrito** con la Actividad guardando la fecha de inscripción. Es la clásica tabla de relación muchos a muchos con datos propios.

Lo que no veis aquí pero sí está implementado es la lógica de lista de espera: cuando las plazas se agotan, el sistema gestiona automáticamente una cola FIFO que promueve a los ciudadanos en espera cuando alguien cancela — sin ninguna intervención manual del personal del centro.

---

## 10. CIERRE Y TRANSICIÓN A DEMO (30 segundos) — Slide 22

En resumen, lo que os hemos presentado hoy es una plataforma diseñada para ser robusta, segura y preparada para crecer con vuestras necesidades. Una arquitectura que garantiza que el inicio de temporada no sea un caos, que ningún ciudadano se quede sin su plaza por un error del sistema, y que el equipo del centro pueda gestionar todo desde un único punto.

Bueno, después de enseñaros la arquitectura que vamos a construir, os mostramos una demo — un MVP realizado en Django — para que veáis cómo va a funcionar el sistema en la práctica y podáis haceros una idea real de la experiencia que tendrá el ciudadano cuando acceda a la plataforma.
