<!--
SPDX-FileCopyrightText: 2026 AlterMundi
SPDX-License-Identifier: CC-BY-SA-4.0
-->
<!-- Supporting document for §3 (Data Collection) of the ARDC 2024 report: the distributed-sensing
     network architecture (reference design, March 2026). -->

# Arquitectura de Red Sensórica Distribuida

> Sistema nervioso distribuido para AlterMundi: fundamentos teóricos,
> decisiones arquitectónicas y diseño de infraestructura.
>
> Documento de referencia para el equipo — marzo 2026.

---

## Tabla de contenidos

1. [Visión](#1-visión)
2. [Fundamentos teóricos](#2-fundamentos-teóricos)
   - 2.1 [Actor Model](#21-actor-model)
   - 2.2 [CRDTs — Tipos de datos replicados sin conflicto](#22-crdts--tipos-de-datos-replicados-sin-conflicto)
   - 2.3 [Estigmergia y sistemas emergentes](#23-estigmergia-y-sistemas-emergentes)
   - 2.4 [Teorema CAP y consistencia eventual fuerte](#24-teorema-cap-y-consistencia-eventual-fuerte)
3. [Evaluación de tecnologías](#3-evaluación-de-tecnologías)
   - 3.1 [CouchDB: la visión original](#31-couchdb-la-visión-original)
   - 3.2 [Kafka: el log como fuente de verdad](#32-kafka-el-log-como-fuente-de-verdad)
   - 3.3 [NATS: transporte neuronal autónomo](#33-nats-transporte-neuronal-autónomo)
   - 3.4 [CRDTs en la práctica: Automerge vs Yjs](#34-crdts-en-la-práctica-automerge-vs-yjs)
   - 3.5 [Matriz de decisión](#35-matriz-de-decisión)
4. [Arquitectura del sistema](#4-arquitectura-del-sistema)
   - 4.1 [Topología de tópicos como ontología](#41-topología-de-tópicos-como-ontología)
   - 4.2 [El núcleo estable](#42-el-núcleo-estable)
   - 4.3 [Roles de los actores](#43-roles-de-los-actores)
   - 4.4 [Patrones de comunicación](#44-patrones-de-comunicación)
   - 4.5 [Propiedades emergentes](#45-propiedades-emergentes)
5. [Diseño sobre infraestructura existente](#5-diseño-sobre-infraestructura-existente)
6. [Consecuencias y trade-offs](#6-consecuencias-y-trade-offs)
7. [Camino a seguir](#7-camino-a-seguir)
8. [Referencias](#8-referencias)

---

## 1. Visión

Construir una capa de infraestructura donde sensores, agentes de IA,
servicios de software y personas coexisten como participantes equivalentes
en un bus de mensajes unificado. Cada participante publica al rate que
puede, consume lo que le interesa, y el sistema emerge de las interacciones
— sin coordinación central, sin monolitos, sin acoplamiento.

El software deja de vivir en servidores y pasa a vivir en un **espacio de
tópicos**. Un proceso puede moverse entre nodos sin que ningún otro
participante lo note, porque la interfaz es el tópico, no la dirección de red.

Lo que resulta no es una aplicación sino un **meta-software estigmérgico**:
una estructura que emerge de las interacciones locales de actores autónomos,
de la misma forma en que las termitas construyen montículos sin plano ni
coordinador.

---

## 2. Fundamentos teóricos

### 2.1 Actor Model

Formulado por Carl Hewitt en 1973, el Actor Model establece que la unidad
fundamental de computación es el **actor**: una entidad que puede:

- Recibir mensajes
- Tomar decisiones locales en respuesta
- Enviar mensajes a otros actores
- Crear nuevos actores

**No hay estado compartido.** Toda comunicación es por paso de mensajes
asíncrono. Esto elimina por construcción los problemas de concurrencia
(locks, deadlocks, race conditions) que plagan los sistemas centralizados.

En nuestro sistema:
- Un **sensor** es un actor que publica datos al rate que su conectividad permite.
- Un **agente IA** es un actor que consume de ciertos tópicos y publica inferencias.
- Un **humano** es un actor que expresa intención (atención, consultas) y consume resultados.
- Un **servicio** es un actor que transforma datos entre tópicos.

La equivalencia de todos los participantes es deliberada: no hay distinción
arquitectónica entre un sensor de temperatura y un modelo de lenguaje. Ambos
publican y consumen mensajes. La complejidad de cada uno es asunto interno.

### 2.2 CRDTs — Tipos de datos replicados sin conflicto

Un CRDT (Conflict-free Replicated Data Type) es una estructura de datos donde
el merge de cualquier par de estados es **conmutativo, asociativo e
idempotente** — es decir, forma un **join-semilattice** en el sentido de orden
parcial.

#### Definición formal

Dado un conjunto de réplicas R₁, R₂, ..., Rₙ, cada una con estado Sᵢ,
la función de merge satisface:

```
merge(a, b) = a ⊔ b    (least upper bound / supremo)

Conmutatividad:   a ⊔ b = b ⊔ a
Asociatividad:    (a ⊔ b) ⊔ c = a ⊔ (b ⊔ c)
Idempotencia:     a ⊔ a = a
```

Estas tres propiedades garantizan que **sin importar el orden de llegada
de los mensajes, ni cuántas veces se reciban, todas las réplicas convergen
al mismo estado**.

#### Las dos familias

**CvRDT (State-based):** cada réplica envía su estado completo; el receptor
hace merge. Ejemplo — G-Counter (contador grow-only):

```
Réplica A: {A:3, B:0}
Réplica B: {A:1, B:5}
merge:     {A:3, B:5}    ← max por componente
valor:     3 + 5 = 8
```

Costo de sincronización: O(n) donde n es el tamaño del estado.

**CmRDT (Operation-based):** cada réplica envía operaciones (deltas), no
estado. Requiere **causal delivery** (si op₁ happened-before op₂, op₁ se
entrega primero). Más eficiente en red pero más exigente en el transporte.

**Delta-state CRDTs** combinan ambos: envían solo el delta del estado desde
la última sincronización. Es lo que usa Automerge internamente.

#### Tipos fundamentales

| Tipo | Operaciones | Semántica | Complejidad de merge |
|---|---|---|---|
| G-Counter | increment | Grow-only counter | O(réplicas) |
| PN-Counter | inc/dec | Positive-Negative | O(réplicas) |
| G-Set | add | Grow-only set | O(elementos) |
| OR-Set | add/remove | Add wins sobre remove concurrente | O(elementos × causal dots) |
| LWW-Register | assign | Last-Writer-Wins por timestamp | O(1) |
| RGA | insert/delete | Texto/secuencias colaborativas | O(n) amortizado |
| Merkle-CRDT | any | CRDT + DAG hash (usado en IPFS) | O(profundidad del DAG) |

#### El problema del garbage collection

El secreto incómodo de los CRDTs: el **metadata crece monotónicamente**. Un
OR-Set mantiene "causal dots" (vector clocks por elemento) para distinguir
removes concurrentes de adds. Un documento de texto en Automerge mantiene una
entrada por cada carácter jamás insertado, incluyendo los borrados.

La compactación requiere que **todas las réplicas estén online
simultáneamente**, lo cual es exactamente lo que no podemos garantizar en una
red mesh. Este es el trade-off fundamental: **sync sin coordinación** a cambio
de **metadata creciente**.

#### Relevancia para nuestro sistema

Los CRDTs son ideales para el **estado que necesita sobrevivir particiones
largas**: configuración de red, identidades, registros de dispositivos. No son
adecuados para telemetría de alto volumen (ahí el log append-only es mejor).

### 2.3 Estigmergia y sistemas emergentes

El término "estigmergia" fue acuñado por Pierre-Paul Grassé en 1959 para
describir cómo las termitas coordinan la construcción de montículos sin
comunicación directa. Cada termita responde al estado local del ambiente
(feromona en el barro) y la estructura emerge de las interacciones.

En nuestro sistema:
- Las **feromonas** son los mensajes en los tópicos.
- Los **actores** responden al estado local de los tópicos que observan.
- La **estructura** (qué se procesa, qué se detecta, qué se escala) emerge de
  las interacciones, no está diseñada top-down.

La propiedad más poderosa: **escala sin coordinación**. Agregar un sensor, un
agente IA o un humano es lo mismo: suscribirse a tópicos y publicar en tópicos.
Nadie necesita "registrar" al nuevo participante — su existencia se manifiesta
por sus mensajes.

Esto contrasta con las arquitecturas de microservicios tradicionales donde
agregar un servicio requiere discovery, registration, health checks y circuit
breakers. En un sistema estigmérgico, la presencia se infiere de la actividad.

### 2.4 Teorema CAP y consistencia eventual fuerte

El teorema CAP (Brewer, 2000; Gilbert & Lynch, 2002) establece que un sistema
distribuido no puede garantizar simultáneamente:

- **C**onsistency (todos los nodos ven los mismos datos al mismo tiempo)
- **A**vailability (toda request recibe respuesta)
- **P**artition tolerance (el sistema funciona pese a pérdida de mensajes entre nodos)

En una red mesh, las particiones son la norma, no la excepción. Esto hace que
**P sea innegociable**, lo cual nos obliga a elegir entre C y A.

Shapiro et al. (2011) demostraron que los CRDTs dan **Strong Eventual
Consistency (SEC)**:

> Si dos réplicas recibieron el mismo set de operaciones (en cualquier orden),
> sus estados son idénticos.

SEC es estrictamente más fuerte que eventual consistency y se consigue **sin
coordinación** (sin consenso, sin locks, sin líder). En términos de CAP, un
sistema basado en CRDTs es **AP con SEC** — sacrifica linearizability pero no
pierde nada práctico para la mayoría de los casos de uso.

Para nuestra red sensórica: los datos de telemetría no necesitan linearizability
(no importa si un nodo ve una lectura de temperatura 5 segundos antes que otro).
Lo que importa es que **eventualmente todos converjan** y que **nada se pierda
durante una partición**.

---

## 3. Evaluación de tecnologías

### 3.1 CouchDB: la visión original

CouchDB (Damien Katz, 2005; Apache desde 2008) fue construido alrededor de
ideas radicales para su época:

- **Todo es una URL**: `/db/doc_id` devuelve el documento, `/db/doc_id/attachment`
  el adjunto. HTTP puro, REST puro.
- **MVCC (Multi-Version Concurrency Control)**: cada escritura crea una nueva
  revisión (`_rev`). Las versiones anteriores persisten hasta compactación. No
  hay locks jamás.
- **Replicación como primitiva de primer nivel**: cualquier par de nodos puede
  sincronizar bidireccionalmente. La resolución de conflictos es determinística
  (gana la revisión con hash más alto; las versiones perdedoras quedan
  accesibles como `_conflicts`).
- **Vistas como datos derivados**: MapReduce materializa las proyecciones que
  necesites del documento crudo.
- **`_changes` feed**: un stream infinito de toda modificación, que permite a
  cualquier proceso reaccionar a cambios en tiempo real.

**Lo brillante:** CouchDB demostró que **addressability matters**. Poder hacer
`GET /thing/123` y recibir el estado actual con su historial de revisiones es
poderoso. Cada vez que alguien construye una capa REST encima de otro sistema,
está reinventando lo que CouchDB ya tenía.

**Estado actual (2026):** vivo pero estancado. Última release 3.4.x (2024),
comunidad chica, desarrollo lento. IBM migró Cloudant a un fork interno.
PouchDB (el companion offline-first para browser) sigue mantenido pero con
actividad mínima. Nadie nuevo lo elige — la categoría migró hacia CRDTs.

**Legado conceptual:** la idea de que un documento tiene una URL, un historial
de revisiones, y puede replicarse sin coordinación central está viva en toda la
arquitectura que describimos en este documento.

### 3.2 Kafka: el log como fuente de verdad

Kafka (LinkedIn, 2011; Apache) invierte la metáfora: en vez de "el documento
es la verdad y el log existe para llegar a él", propone que **el log es la
verdad y el estado actual es una vista materializada del log**.

Martin Kleppmann ("Turning the Database Inside Out", 2014) formalizó esta idea:
si tenés el log completo de operaciones, podés derivar cualquier vista (SQL,
key-value, search index, grafo). Una base de datos es un cache optimizado de
un subconjunto del log.

| Concepto | CouchDB | Kafka |
|---|---|---|
| Historia inmutable | Revisiones (`_rev`) | Offsets en una partición |
| Estado actual | Última revisión | Log compaction (último valor por key) |
| Vistas derivadas | MapReduce / Mango | KStreams / ksqlDB / consumers |
| Replicación | Multi-master sync | Partition replicas + MirrorMaker |
| Direccionamiento | `/db/doc_id` (REST) | `topic/partition/offset` (protocolo binario) |

Con **log compaction** habilitado, un topic de Kafka funciona como un key-value
store donde el último mensaje por key es el "documento actual" — similar al
modelo de CouchDB.

**Estado actual (2026):** dominante en el enterprise. KRaft eliminó ZooKeeper,
simplificando la operación. Confluent sigue invirtiendo fuertemente. El
ecosistema (KStreams, ksqlDB, Connect) es maduro.

**Limitaciones para nuestro caso de uso:**

- **Topics estáticos**: hay que crearlos explícitamente y decidir particiones
  de antemano.
- **Consumer model pull-only**: no hay request-reply nativo, lo cual es
  fundamental para "orquestador pregunta, sensores responden".
- **Peso operativo**: JVM, mínimo ~1GB RAM, protocolo binario complejo
  (~2000 líneas mínimo para un cliente funcional, ~200-500 bytes overhead por
  mensaje).
- **No corre en edge**: un broker Kafka no entra en un router mesh ni en un
  microcontrolador.

**Rol en nuestra infra actual:** Kafka ya corre en yupanki CT 120 (KRaft,
single-node) para el pipeline SAI → IPFS. Para ese flujo lineal y estable,
funciona bien. No es el bus general del sistema que describimos.

### 3.3 NATS: transporte neuronal autónomo

NATS (Derek Collison, 2010; CNCF Incubating) es un message bus diseñado
alrededor de la simplicidad del wire protocol.

#### Protocolo

El protocolo es texto plano, legible por humanos:

```
PUB subject reply-to payload-size\r\n
payload\r\n

SUB subject queue-group sid\r\n

MSG subject sid payload-size\r\n
payload\r\n
```

Esto no es accidental. Collison (ex-TIBCO) argumenta que la complejidad de
protocolos como AMQP y el protocolo binario de Kafka es la fuente principal
de bugs en sistemas distribuidos. Un protocolo que cabe en ~200 líneas de
implementación puede tener clientes en Arduino y ESP32. El overhead por
mensaje es ~50 bytes.

#### Modelos de delivery

**At-most-once (NATS Core):** fire-and-forget. Si nadie escucha, el mensaje
se pierde. Correcto para telemetría, heartbeats, discovery — el 90% de los
mensajes en una red sensórica.

**At-least-once (JetStream):** agrega un log persistente. Un Stream es un log
ordenado (como un Kafka topic); un Consumer es un cursor con posición (como un
Kafka consumer group). Pero con diferencias clave:

- **Work queues nativas** — no hace falta consumer groups artificiales.
- **Key-value store built-in** — basado en JetStream (como Kafka compacted
  topics pero con API de KV directa).
- **Object store** — chunked storage de blobs sobre JetStream.

**Effectively-once (JetStream + dedup):** dedup por message-id dentro de una
ventana temporal configurable. Exactly-once es imposible en sentido estricto
(consecuencia del Two Generals' Problem), pero effectively-once es suficiente
para el 99.99% de los casos.

#### Subject-based addressing

Acá es donde NATS conecta con la pregunta original sobre URLs y CouchDB:

```
sensores.comunidad-quintana.nodo-abc.temperatura
sensores.comunidad-quintana.nodo-abc.señal
sensores.comunidad-quintana.*.temperatura       ← wildcard un nivel
sensores.comunidad-quintana.>                   ← wildcard recursivo
```

Esto es un **namespace jerárquico**, análogo a URLs REST pero para mensajes.
Un subscriber a `sensores.>` recibe todo. Uno a
`sensores.comunidad-quintana.*.temperatura` recibe solo temperaturas de
Quintana.

Con JetStream KV:
```
PUT sensores.comunidad-quintana.nodo-abc.temperatura = 23.5
GET sensores.comunidad-quintana.nodo-abc.temperatura → 23.5
WATCH sensores.comunidad-quintana.> → stream de cambios
```

Esto es el `_changes` feed de CouchDB implementado sobre un log.

#### Leaf nodes

NATS tiene **leaf nodes**: instancias livianas que se conectan a un cluster
central y reenvían solo los subjects que les interesan.

Un leaf node:
- Corre en ~10MB de RAM (binario Go estático)
- Se reconecta automáticamente
- Bufferea mensajes durante desconexiones (con JetStream)
- Solo suscribe/publica los subjects relevantes a su ubicación

Comparado con Kafka (mínimo ~1GB, JVM), esto es la diferencia entre correr
en un nodo edge y no poder.

#### Estado actual (2026)

Proyecto CNCF maduro, desarrollo muy activo por Synadia. Go puro, sin
dependencias. Corre en Linux, macOS, Windows, ARM, MIPS. El binario del
servidor pesa ~20MB.

### 3.4 CRDTs en la práctica: Automerge vs Yjs

**Automerge** (Martin Kleppmann, Cambridge): académicamente sólido, usa un RGA
con Lamport timestamps. Formato binario eficiente. Sync protocol propio sobre
cualquier transporte (incluyendo NATS). Core en Rust con bindings a
JS/Swift/Python/etc.

**Yjs** (Kevin Jahns): más pragmático, optimizado para performance. Usa YATA
algorithm, ~10x más rápido que Automerge para texto colaborativo. Menos robusto
teóricamente pero funciona mejor para documentos grandes.

**Para nuestro caso:** Automerge es preferible porque su sync protocol es más
robusto para conexiones intermitentes y no asume servidor central.

**Alternativas emergentes (2026):**

- **ElectricSQL / PowerSync**: sync bidireccional sobre PostgreSQL. Buenos para
  apps con backend Postgres pero asumen conectividad más estable que una mesh.
- **cr-sqlite**: CRDTs directamente en SQLite. Interesante para state local
  en nodos con más recursos.

### 3.5 Matriz de decisión

| Caso de uso | Tecnología elegida | Razón |
|---|---|---|
| Bus de mensajes general | **NATS** | Liviano, subject addressing, leaf nodes, request-reply |
| Persistencia de streams | **NATS JetStream** | Integrado con NATS Core, KV y Object store incluidos |
| State sync con particiones largas | **Automerge (CRDTs)** | Convergencia garantizada sin coordinación |
| Pipeline SAI→IPFS (existente) | **Kafka** (mantener) | Ya funciona, flujo lineal estable |
| State consultable (último valor) | **NATS KV** | Built-in sobre JetStream, API simple |
| Storage de blobs (grabaciones, media) | **NATS Object Store + MinIO** | Blobs grandes chunked, archival en object storage |

---

## 4. Arquitectura del sistema

### 4.1 Topología de tópicos como ontología

El namespace de tópicos **es** la ontología del sistema. No es naming
convention — es una estructura algebraica donde los wildcards definen
subconjuntos del espacio de información y una suscripción define el campo de
percepción de un actor.

```
sensores/
  {comunidad}/
    {nodo}/
      {magnitud}               → dato crudo, rate variable
      {magnitud}/embedding     → representación vectorial (procesada)

eventos/
  detección/
    {tipo-evento}/
      {timestamp}/
        contexto               → qué sensores participaron
        confianza              → score del detector
        datos                  → payload del evento

agentes/
  {nombre-agente}/
    comandos                   → órdenes que acepta
    estado                     → heartbeat, capacidad, carga
    input                      → tópicos que consume (metadata)
    output                     → tópicos donde publica (metadata)

consultas/
  {id-consulta}/
    request                    → query (quién, qué, cuándo)
    resultado                  → respuesta materializada
    estado                     → progreso

orquestación/
  atención/
    {evento-id}                → broadcast de intención
  escalamiento/
    {agente}/
      rate                     → configuración dinámica de captura
```

### 4.2 El núcleo estable

Los dos servidores públicos (inference-public y yupanki) forman el **núcleo
siempre-disponible** del sistema — los seed nodes que garantizan liveness.

```
┌─────────────────────────────────────────────────────────┐
│                  Núcleo Estable (always-on)              │
│                                                         │
│  inference-public ◄════════════════► yupanki            │
│  131.72.205.6         full mesh      138.255.88.2       │
│  (NATS node 1)        replication    (NATS node 2)      │
│                                                         │
│  Roles:                              Roles:             │
│  - NATS server                       - NATS server      │
│  - JetStream (media/pesado)          - JetStream (meta) │
│  - Agentes IA (GPU)                  - Servicios core   │
│  - IPFS cluster                      - Grafana/VM/etc   │
└────────────┬────────────────────────────────┬───────────┘
             │                                │
     ┌───────┴─────────┐            ┌─────────┴───────┐
     │  Leaf Nodes      │            │  Leaf Nodes      │
     │  (comunidades)   │            │  (comunidades)   │
     └─────────────────┘            └─────────────────┘
```

En teoría de grafos, estos dos nodos son los **puntos de articulación** del
sistema. Mientras al menos uno esté arriba, cualquier nodo que vuelva a
conectarse puede sincronizarse.

Propiedades formales del par:
- **Write quorum W=1**: cualquiera de los dos acepta escrituras.
- **Read quorum R=1**: cualquiera de los dos sirve lecturas.
- **Replicación full-mesh**: cada stream de JetStream se replica en ambos nodos.
- **Partition tolerance**: si el link entre ambos cae, cada uno sigue operando
  con su mitad de la red. Cuando el link vuelve, JetStream hace catchup
  automático.

### 4.3 Roles de los actores

#### Sensores (publishers puros)

Publican datos al rate que su conectividad permite. No consumen. No tienen
estado global. Un sensor es un actor minimal:

```
loop:
    dato = leer_sensor()
    nats.publish(f"sensores.{mi_comunidad}.{mi_id}.{magnitud}", dato)
    sleep(rate_actual)
```

Si no hay conexión, el dato se descarta (at-most-once para telemetría) o se
bufferea localmente (at-least-once vía JetStream si hay leaf node local).

#### Agentes IA (consumidores-productores)

Consumen de ciertos tópicos, procesan, publican inferencias. Forman pipelines
como dataflow graphs:

```
sensor/audio/raw
  → agente/vad (voice activity detection)
    → agente/transcripción
      → agente/clasificación-intención
        → eventos/detección/pedido-de-ayuda
          → agente/notificación → humano/operador/alertas
```

Cada flecha es un par publish/subscribe. Cada nodo del pipeline puede:
- **Escalar horizontalmente** con N instancias (NATS queue groups distribuyen).
- **Morir y revivir** sin romper el pipeline (JetStream hace replay).
- **Ser reemplazado** por una versión nueva sin downtime.

#### Agentes orquestadores

Expresan intención y coordinan atención. No procesan datos directamente:

```
orquestador detecta anomalía →
  publica en orquestación.atención.{evento-id}:
    { "rate": "máximo",
      "duración": "30min",
      "sensores": "sensores.comunidad-quintana.>",
      "prioridad": 1 }

cada sensor con leaf node recibe y decide localmente si puede cumplir →
  responde con capacidad real
```

Patrón: **scatter-gather** de NATS (publish con reply-to, recoger respuestas
durante un timeout).

#### Agentes humanos

Expresan atención (a qué eventos/datos miran) y lanzan consultas ad-hoc:

```
humano/fede/atención:
  "anomalía-térmica quintana 2026-03-20"

humano/fede/consultas:
  "correlación temperatura + actividad-red, últimas 6h, todas las comunidades"
```

La consulta es un mensaje como cualquier otro. Un agente de consultas la recibe,
ejecuta el join temporal sobre los streams relevantes, y publica el resultado
en `consultas/{id}/resultado`.

### 4.4 Patrones de comunicación

#### Publish-subscribe (1 a N)

```
sensor publica → N agentes suscritos reciben
```

Para telemetría, eventos, broadcasts. El publisher no sabe ni le importa
cuántos consumers hay.

#### Request-reply (1 a 1)

```
orquestador pregunta → sensor responde
```

NATS implementa esto nativamente con un reply-to subject auto-generado.
Fundamental para consultas directas y health checks.

#### Scatter-gather (1 a N con respuestas)

```
orquestador pregunta → N sensores responden → orquestador agrega
```

Request-reply con timeout y múltiples respuestas. Para discovery, capacidad
disponible, voting.

#### Queue groups (N a 1-de-M)

```
N sensores publican → 1 de M instancias del agente recibe cada mensaje
```

Load balancing nativo. Para escalar agentes de procesamiento sin duplicar
trabajo.

#### Chain / pipeline (stream processing)

```
tópico A → agente 1 → tópico B → agente 2 → tópico C
```

Composición funcional sobre streams. Cada agente es una transformación
pura: consume de un tópico, publica en otro.

### 4.5 Propiedades emergentes

#### Consulta multidominio como join de streams

Una consulta que cruza dominios (temperatura + actividad de red +
detecciones de audio) no requiere una base de datos central con todos los
datos indexados. Un agente puede:

1. Suscribirse a múltiples subjects con replay temporal.
2. Hacer el join por timestamp en memoria.
3. Publicar el resultado en un tópico de consulta.
4. Cualquier otro agente puede tomar ese resultado y profundizar.

**No hay bottleneck central** — el análisis fluye por el grafo de tópicos.

#### Atención como recurso asignable

"Prestar atención con todos los sensores disponibles" es un broadcast de
intención. Los sensores responden con su capacidad real. El rate de captura
se ajusta dinámicamente en función del interés declarado por los actores
del sistema.

Esto invierte el modelo clásico: en vez de configurar sensores estáticamente,
los sensores responden a la demanda del sistema. Un evento interesante atrae
recursos de observación — exactamente como la atención biológica.

#### Patrones no anticipados

En un sistema estigmérgico, los patrones emergen de las interacciones.
Ejemplos de lo que podría surgir sin ser diseñado explícitamente:

- Un agente que detecta correlaciones entre comunidades que nadie había
  conectado, porque ve todos los streams simultáneamente.
- Cascadas de atención: un evento en una comunidad dispara mayor observación
  en comunidades vecinas, como una onda.
- Auto-regulación: si demasiados agentes piden rate máximo, los sensores con
  batería limitada negocian a la baja — equilibrio emergente.

---

## 5. Diseño sobre infraestructura existente

### Mapping a yupanki + inference-public

```
┌─────────────────────────────────────────────────────────────────┐
│                        NATS Supercluster                        │
│                                                                 │
│  ┌──────────────────────┐         ┌──────────────────────────┐  │
│  │  inference-public     │         │  yupanki                 │  │
│  │  131.72.205.6         │◄═══════►│  138.255.88.2            │  │
│  │                       │  NATS   │                          │  │
│  │  NATS Server          │  route  │  NATS Server             │  │
│  │  JetStream:           │         │  JetStream:              │  │
│  │    sensores.*.media/* │         │    sensores.*.telemetría  │  │
│  │    agentes.inferencia │         │    eventos.*              │  │
│  │                       │         │    consultas.*            │  │
│  │  Agentes IA (GPU):    │         │    orquestación.*         │  │
│  │    clasificadores     │         │                          │  │
│  │    embeddings         │         │  Servicios:              │  │
│  │    transcripción      │         │    VictoriaMetrics       │  │
│  │                       │         │    Grafana               │  │
│  │  IPFS cluster         │         │    Kafka (SAI pipeline)  │  │
│  │  SAI API              │         │    Zitadel (auth)        │  │
│  └──────────┬───────────┘         └───────────┬──────────────┘  │
│             │                                  │                 │
│      ┌──────┴──────────────────────────────────┴──────┐         │
│      │              Leaf Nodes (edge)                  │         │
│      │                                                 │         │
│      │  ┌───────────┐  ┌───────────┐  ┌───────────┐  │         │
│      │  │ Quintana  │  │ Delta     │  │ Córdoba   │  │         │
│      │  │ leaf node │  │ leaf node │  │ leaf node │  │         │
│      │  │ ~10MB RAM │  │ ~10MB RAM │  │ ~10MB RAM │  │         │
│      │  └───────────┘  └───────────┘  └───────────┘  │         │
│      └─────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

### Distribución de JetStream streams por nodo

**inference-public** (RAID, más storage, GPU):
- Streams pesados: `sensores.*.*.audio.raw`, `sensores.*.*.video.raw`
- Agentes que requieren GPU
- IPFS para storage permanente

**yupanki** (más CTs, mejor conectividad, servicios core):
- Streams de metadata y telemetría
- Eventos, consultas, orquestación
- Servicios existentes (Grafana, VictoriaMetrics, Kafka para SAI)

La replicación entre ambos nodos cubre el stream completo —
la distribución es por **afinidad de procesamiento**, no por partición de datos.

### Convivencia con servicios existentes

NATS no reemplaza lo que ya funciona:

| Servicio existente | Relación con NATS |
|---|---|
| Kafka (CT 120) | Se mantiene para SAI→IPFS. Un bridge NATS↔Kafka puede conectar ambos mundos si hace falta. |
| VictoriaMetrics (CT 111) | Sigue recibiendo métricas vía Prometheus scrape. Un agente NATS puede publicar alertas basadas en métricas VM. |
| Grafana (CT 110) | Dashboards existentes no cambian. Se puede agregar un datasource NATS para streams en tiempo real. |
| Zitadel (CT 115) | Auth/identidad para agentes que requieran autenticación. |

### Implementación incremental sugerida

**Fase 1 — NATS cluster mínimo:**
- Un nodo NATS en inference-public, uno en yupanki.
- Un leaf node en un router/nodo de prueba.
- Un stream de telemetría simple (temperatura o señal WiFi).

**Fase 2 — Primer agente:**
- Un agente que consume del stream de telemetría y publica detecciones.
- Visualización en Grafana vía bridge a VictoriaMetrics.

**Fase 3 — Orquestación:**
- Agente orquestador que modifica rates de captura.
- Consultas multidominio.

**Fase 4 — CRDTs para state:**
- Automerge para configuración de red distribuida.
- Sync sobre NATS request-reply.

---

## 6. Consecuencias y trade-offs

### Lo que ganamos

- **Desacoplamiento total**: ningún actor conoce a los demás. Solo conoce
  tópicos.
- **Escalabilidad sin coordinación**: agregar actores es publicar/suscribir.
- **Resiliencia a particiones**: el edge sigue funcionando con leaf nodes
  locales. El catchup es automático.
- **Composición infinita**: pipelines de procesamiento se arman conectando
  tópicos, sin frameworks especiales.
- **Heterogeneidad**: un ESP32 y un cluster GPU participan del mismo bus.

### Lo que pagamos

- **Observabilidad**: en un monolito, un stack trace te dice qué pasó. En un
  sistema estigmérgico, una cadena de eventos cruza N actores en M nodos. Se
  necesita distributed tracing (OpenTelemetry sobre NATS).
- **Debugging**: "¿por qué no se detectó este evento?" puede requerir
  reconstruir el estado de N actores en un momento específico.
- **Consistencia**: SEC es suficiente para casi todo, pero si algún caso
  requiere linearizability (ej: billing, auth tokens), necesita un componente
  centralizado (PostgreSQL, Zitadel).
- **Complejidad conceptual**: el equipo necesita entender el modelo de actores,
  CRDTs y eventual consistency. La curva de aprendizaje no es trivial.
- **Metadata de CRDTs**: para el state distribuido (no para telemetría), el
  costo de storage crece con el historial de operaciones.

### Riesgos y mitigaciones

| Riesgo | Mitigación |
|---|---|
| NATS es un single vendor (Synadia) | Protocolo abierto, código Apache-2.0, CNCF incubating. Alternativa: migrar a cualquier pub/sub que soporte el mismo subject model. |
| Leaf nodes en hardware limitado | NATS server en Go puro, ~10MB RAM. Si es demasiado, un cliente NATS puro en C/Rust es viable (~1MB). |
| Pérdida de datos en at-most-once | Telemetría es inherently lossy. Para datos críticos, JetStream con at-least-once. |
| Explosión de tópicos | Los subjects son efímeros (no consumen recursos si nadie publica). Streams de JetStream se configuran con retención por tiempo/tamaño. |
| El equipo no conoce NATS | Protocolo simple (se aprende en una tarde), buena documentación, playground oficial para experimentar. |

---

## 7. Camino a seguir

### Preguntas abiertas

1. **¿Qué sensores concretos serían los primeros en conectarse?** Esto define
   el schema de los primeros tópicos.
2. **¿Hay nodos comunitarios con recursos suficientes para un leaf node
   (~10MB RAM, storage para buffer)?** Esto define si Fase 1 incluye edge.
3. **¿Qué agentes IA existen hoy que podrían migrar a este modelo?** SAI
   worker es candidato natural.
4. **¿El pipeline SAI→Kafka→IPFS se beneficia de migrar a NATS, o el bridge
   es suficiente?** Depende de si necesitan scatter-gather o request-reply
   en ese flujo.

### Próximos pasos concretos

1. Evaluar NATS en un CT de yupanki (minimal: 512MB RAM, 2GB disco).
2. Levantar un segundo nodo NATS en inference-public, formar cluster.
3. Conectar un leaf node desde un nodo de prueba en una comunidad.
4. Publicar telemetría real y consumirla desde Grafana.
5. Iterar.

---

## 8. Referencias

### Papers fundamentales

- **Hewitt, Bishop, Steiger** (1973). *A Universal Modular ACTOR Formalism for
  Artificial Intelligence*. IJCAI. — El paper original del Actor Model.
- **Shapiro, Preguiça, Baquero, Zawirski** (2011). *Conflict-free Replicated
  Data Types*. SSS 2011. — Formalización de CRDTs y prueba de Strong Eventual
  Consistency.
- **Grassé, Pierre-Paul** (1959). *La reconstruction du nid et les coordinations
  interindividuelles chez Bellicositermes natalensis et Cubitermes sp.*
  Insectes Sociaux. — Origen del concepto de estigmergia.
- **Gilbert, Lynch** (2002). *Brewer's Conjecture and the Feasibility of
  Consistent, Available, Partition-Tolerant Web Services*. — Prueba formal
  del teorema CAP.
- **Kleppmann, Martin** (2014). *Turning the Database Inside Out with Apache
  Samza*. Strange Loop. — El log como fuente de verdad.
- **Kleppmann, Wiggins, van Hardenberg, McGranaghan** (2019). *Local-first
  Software: You Own Your Data, in Spite of the Cloud*. Onward! 2019. —
  Manifiesto del software local-first con CRDTs.

### Tecnologías

- **NATS**: https://nats.io — Documentación oficial, tutoriales, playground.
- **Automerge**: https://automerge.org — CRDT library, sync protocol, formato binario.
- **Yjs**: https://yjs.dev — CRDT optimizado para texto colaborativo.
- **CouchDB**: https://couchdb.apache.org — La visión original de replicación REST.
- **Kafka**: https://kafka.apache.org — Event streaming platform.

### Contexto AlterMundi

- `docs/MIGRACION-YUPANKI-INFERENCE.md` — Historia de migración de servicios.
- `docs/PLAN-YUPANKI-DEPLOYMENT.md` — Plan de deployment en yupanki.
- `altermundi-infrastructure-map.md` — Mapa de infraestructura actual.
