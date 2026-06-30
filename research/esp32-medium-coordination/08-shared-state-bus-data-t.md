# 08 — Shared-state como bus único de `data(t)`: la arquitectura unificadora

> **Origen:** observación de Fede (2026-06-19): el daemon **shared-state** reaparece como componente fundamental de
> next-gen LibreMesh — aporta orgánicamente el ecosistema de `data(t)` que la red necesita para adecuarse en tiempo real a
> cambios de performance y topología, no solo para un agente operador IA sino **para el G(t) que el TDMA de bajo nivel
> necesita**. Evidencia citable en `research/shared-state-data-t.md`. Conecta `07` (el grafo G(t)) con el entregable
> Data-Collection (entregable 3) y con el shared-state existente (`shared-state-merge-strategy.md`,
> `referencias-personas-instituciones.md`).

## 0. La tesis
**`data(t)` para el operador y `data(t)` para el scheduler son el mismo flujo, distintos consumidores.** El pipeline de
estimación de G(t) (`07`: medir → diseminar → construir → adaptar) necesita un sustrato distribuido; ese sustrato **ya
existe** (shared-state v3 Rust/`smol`, que el corpus documenta como capaz de propagar **eventos efímeros**, no solo
telemetría lenta). El mismo stream alimenta al agente operador IA. → El hilo conductor de next-gen LibreMesh es **un único
plano de observabilidad/estado distribuido** que sirve a la vez operación humana/IA **y** control autonómico de bajo nivel.
**MW-CA-RF (entregable 1) y Data-Collection (entregable 3) comparten sustrato.**

## 1. Cada mitad de la tesis tiene prior art fuerte
- **Estado eventually-consistent que alimenta control:** **Onix** (OSDI'10, dos stores: fuerte lento + DHT eventual rápido,
  la app elige) es la cita madre; **4D** (CCR'05) descompone por timescale; **Knowledge Plane** (SIGCOMM'03) es el
  antecedente filosófico ("la capa de conocimiento tolera info incompleta/inconsistente").
- **Telemetría + IA + control (la mitad operador):** **Knowledge-Defined Networking** (Mestres, CCR'17, lazo
  observe-learn-act) y **SON** (3GPP TS 32.500: self-config/optimization/healing). Tu agente operador es una instancia de KDN.
- **Medir un grafo de conflicto y reconfigurar:** **Subramanian et al.** (TMC'08) ya lo hacen — pero para *channel
  assignment*, no TDMA. La pieza "medir G(t) → reconfigurar" existe; **lo nuevo es enchufarla a un scheduler TDMA**.

## 2. La arquitectura honesta: desdoblamiento de escalas (lo que evita el error)
**No se puede correr el lazo de slots por una capa gossip eventually-consistent.** La arquitectura se parte en dos planos
(precedente directo: **4D** decision/dissemination vs data; **Kandoo** root lento vs locales rápidos; formalización de
estabilidad: **perturbaciones singulares**, Kokotović):

| Plano | Qué | Escala | Sustrato | Consistencia |
|-------|-----|--------|----------|--------------|
| **Modelo (lento)** | aristas de G(t), agregadas red-wide; modelo para el operador IA | **segundos–minutos** (la dinámica ambiental que cambia G(t) —follaje/lluvia/ducting— es lenta, ver `07`/`research/dinamica-temporal`) | **shared-state** (gossip/CRDT) | eventual, tolera staleness |
| **Ejecución (rápido)** | grant por slot (~10 ms), local al coordinador/clúster | **sub-segundo** | **ESP-NOW local** (NO shared-state) | local, autoritativa |

Shared-state alimenta el **modelo** del scheduler, no sus **decisiones por slot**.

## 3. Las tres consecuencias que esto destapa
### (a) El bug de convergencia TTL de shared-state pasa a ser load-bearing
El corpus documenta el problema (discrepancias TTL, ventana de bloqueo ~22 s, "is remote peer ill?", latencia de entrega no
contabilizada). Para telemetría es molestia; **para un G(t) que maneja un lazo de control es restricción de correctitud**:
si G(t) está stale o diverge entre nodos, los coordinadores agendan sobre grafos inconsistentes → **doble-asignación de
celdas/colisiones** (análogo formal: los *forwarding loops/black holes* de Sakic, ICC'17, cuando el control corre sobre
estado eventual divergente). → **Arreglar el merge de shared-state deja de ser higiene de Data-Collection y se vuelve
prerrequisito de MW-CA-RF.** La herramienta para cuantificar "¿es G(t) lo bastante fresco?": **Age of Information** (Kaul,
INFOCOM'12) + **Probabilistically Bounded Staleness** (Bailis, VLDB'12) — dan P(staleness ≤ t) ≥ 1−ε medido sobre el gossip,
el eslabón que falta entre la garantía CRDT (convergencia de *valor*, no de *tiempo*) y el requisito del scheduler.
*Resultado contraintuitivo aplicable (Sun, TIT'17 "Update or Wait"):* diseminar lo más rápido posible **no** da el estado
más fresco — el transporte de eventos efímeros debe optimizar AoI, no tasa de envío.

### (b) Circularidad de bootstrap (el cold-start a nivel sistema)
Shared-state se disemina sobre la misma malla cuyo acceso al medio el scheduler intenta arreglar. Cadena huevo-gallina:
*data(t) → G(t) → agendar → mejorar el medio → entregar data(t)*. **Tiene precedente formal:** **Renaissance** (ICDCS'18,
control-plane in-band **self-stabilizing** con convergencia probada) y, lo más cercano, **6TiSCH RFC 8180** (un nodo debe
seguir ya un schedule para recibir los mensajes que lo construyen → **celda compartida mínima** que carga el bootstrap sobre
el mismo medio que se agenda). Resolución: arrancar best-effort sobre la ventana CSMA residual (`03 §C`), G(t) grueso,
coordinar → mejora la entrega → afina G(t). **Ciclo virtuoso, pero requiere análisis de estabilidad** (puede oscilar; ver el
umbral de factibilidad `τ > max{4δ, χ₂}` de Petig-Schiller-Tsigas en `research/dinamica-temporal`).

### (c) Fuerza la decisión Data-Collection (F4 de GAPS)
El caso de uso de G(t) necesita **propagación efímera de baja latencia** → favorece el diseño **NATS/Automerge** (CRDT,
eventos efímeros) por sobre la telemetría lenta SensorThings. El G(t) es un buen *forcing function* para reconciliar los dos
modelos no reconciliados del entregable 3. *(Automerge: Kleppmann TPDS'17; δ-CRDTs: Almeida NETYS'15; cuidado: el conflicto
semántico —dos nodos asignando la misma celda— el CRDT NO lo resuelve, necesita política MAC determinista encima.)*

## 4. El veredicto de novedad/riesgo (honesto, para ARDC)
**No hay prior art de un mismo bus de estado distribuido sirviendo a la vez telemetría-de-operador Y control-de-MAC sobre el
medio que controla.** Los vecinos parciales se citan **juntos**, no como un precedente: **KDN/SON** (actuación out-of-band),
**6TiSCH 6P/RFC 8180** (control MAC in-band, sin telemetría de operador), **Renaissance** (control in-band L2–L4, no
scheduler-MAC). **Ese es el núcleo de la contribución.** El riesgo concreto es la **frescura diferencial**: telemetría laxa
(tolerante a staleness) y control-MAC estricto (sensible a AoI) compartiendo mecanismo. La literatura de **consistencia
adaptable** (Sakic, ICC'17) da la salida: **niveles de consistencia diferenciados por tipo de estado dentro del mismo bus**
— telemetría eventual laxa; G(t) con consistencia más fuerte **localmente** (entre vecinos del dominio de colisión, no
network-wide), que es justo donde PACELC (Abadi'12) dice que se puede pagar algo de latencia por consistencia sin violar CAP.

## 5. Implicaciones para el cierre y el desarrollo
- **Para el informe ARDC:** esto convierte a shared-state en **pieza transversal** de los entregables 1 y 3, y da una
  narrativa unificadora de "next-gen LibreMesh" — pero **declarando** que la unión es novedad no probada (riesgo), no un
  sistema existente. El arreglo del merge de shared-state se vuelve dependencia explícita de MW-CA-RF.
- **Para el desarrollo:** define dos workstreams acoplados — (i) shared-state como bus de `data(t)` con consistencia
  diferenciada y métrica de frescura (AoI/PBS), (ii) el scheduler que consume un snapshot reciente de G(t) y corre el lazo
  rápido local. Detalle de cada ángulo en `09-agenda-de-investigacion-fundamentada.md`.
