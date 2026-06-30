# Research — Shared-state como bus único de `data(t)` (telemetría + control)

> Para `08`. Cita: autor, venue/año, URL, "cómo encamina". **[HECHO]**/**[INFER]**, confianza marcada.
> **Veredicto:** cada *mitad* de la tesis tiene prior art fuerte; la *unión* (un bus que sirve telemetría-de-operador Y
> control-de-MAC sobre el medio que controla) **no está publicada** → novedad **y** riesgo (frescura diferencial).

## 1. CRDTs y consistencia eventual
- **Shapiro/Preguiça/Baquero/Zawirski, SSS 2011** — CRDTs garantizan **Strong Eventual Consistency** (convergencia sin
  rollback). https://hal.science/hal-00932836 · informe largo **INRIA RR-7506** (catálogo: counters/sets/**graphs**/sequences).
  **[HECHO/alta]** → modelar G(t) como graph-CRDT da convergencia sin coordinación. **Qué NO da:** cota de tiempo de
  convergencia ni de staleness, ni invariante global ("ninguna celda TDMA doblemente asignada").
- **Almeida/Shoker/Baquero, NETYS 2015 / JPDC 2018 (δ-CRDTs)** — fiabilidad state-based con mensajes pequeños (deltas),
  **si** el anti-entropy respeta causalidad. arXiv:1410.2803, 1603.01529 · **[HECHO/alta]** → modelo de propagación correcto
  para eventos efímeros con BW escaso; **riesgo:** si `smol` reordena/pierde deltas sin reconciliación causal → se pierde convergencia.
- **Kleppmann & Beresford, IEEE TPDS 2017 (JSON CRDT / Automerge)** — converge sin perder updates concurrentes. arXiv:1608.03960
  · **[HECHO/alta]** → precedente de implementación si el estado es un documento; **peligro:** conflicto **semántico** (dos nodos
  asignando la misma celda) aflora como multi-valor que el CRDT NO resuelve → necesitás política MAC determinista encima.
- **Demers et al., PODC 1987 (epidemic/anti-entropy)** — gossip empuja a convergencia **probabilística**, no determinista en
  tiempo. DOI 10.1145/41840.41841 · **[HECHO/alta]** → justifica el gossip y **acota lo que NO podés prometer** (sin worst-case de frescura).

## 2. ¿Puede estado eventually-consistent alimentar un lazo de control?
- **Clark et al., SIGCOMM 2003 (Knowledge Plane)** — capa de conocimiento network-wide que tolera info **incompleta,
  inconsistente, conflictiva**. **[HECHO/alta, visión]** → antecedente filosófico directo (sin números).
- **Koponen et al. (Onix), OSDI 2010** — control-plane SDN distribuido con **DOS** stores: transaccional fuerte (lento) y
  **DHT eventually-consistent (rápido)**; la app elige el trade-off. https://static.usenix.org/event/osdi10/tech/full_papers/Koponen.pdf
  · **[HECHO/alta]** → **cita madre** de que un store eventual *puede* respaldar control; legitima la mitad (b).
- **Levin et al., HotSDN 2012** — control sobre estado **eventual** puede producir **decisiones incorrectas**; el modelo de
  consistencia determina corrección/performance. http://www.icsi.berkeley.edu/pubs/networking/logicallycentralized12.pdf
  · **[HECHO/alta]** → **desafía** la tesis: hay que mostrar que G(t)/TDMA tolera la incorrección (degradación graciosa) o acotar la divergencia.
- **Sakic et al., ICC 2017** — consistencia fuerte bloquea/infla delay; eventual → vistas divergen → decisiones incorrectas
  (forwarding loops/black holes) → **consistencia adaptable**. arXiv:1902.02573 · **[HECHO/alta]** → **partir el bus**: telemetría
  laxa vs G(t) con consistencia más fuerte **localmente** (entre vecinos del dominio de colisión).
- **Gilbert & Lynch 2002 (CAP) + Abadi 2012 (PACELC)** — sin partición: **Latencia-vs-Consistencia**. **[HECHO/alta]**
  → **PACELC** formaliza por qué un scheduler de bajo nivel no puede tener frescura/baja-latencia + consistencia fuerte network-wide a la vez.
- **Coronado et al. (5G-EmPOWER) TNSM 2019 / OpenRadio HotSDN 2012** — SDN-wireless hasta el MAC. **[HECHO/alta existencia]**
  → precedente de control hasta el MAC, **pero** ninguno cuantifica el fallo por consistencia eventual a nivel TDMA → el hueco que ocuparía el trabajo.

## 3. Desdoblamiento plano-modelo lento / plano-ejecución rápido
- **Greenberg et al. (4D), CCR 2005** — descompone por **timescale**: decision (lento) + dissemination + discovery + data
  (rápido per-packet). DOI 10.1145/1096536.1096541 · **[HECHO/alta]** → **ancla directa**: plano-modelo ≈ decision+dissemination; plano-ejecución (TDMA local) ≈ data.
- **Hassas Yeganeh & Ganjali (Kandoo), HotSDN 2012** — locales sin interconexión para eventos frecuentes (rápido) + root
  network-wide (lento). http://conferences.sigcomm.org/sigcomm/2012/paper/hotsdn/p19.pdf · **[HECHO/alta]** → patrón más limpio
  para particionar el daemon: decisiones MAC frecuentes → local; diseminación de G(t)/modelo IA → root lento.
- **Kokotović/Khalil/O'Reilly 1986/1999 (singular perturbation)** — fast inner loop / slow outer loop formal. **[HECHO/alta]**
  → aparato para argumentar **estabilidad** del desdoblamiento (lazo rápido asume G(t) "congelado"; lazo lento asume el rápido asentado).
  *(RCP/4D/RouteFlow como ejemplos concretos control-lento/forwarding-rápido.)*

## 4. Frescura como restricción de CORRECTITUD
- **Kaul/Yates/Gruteser, INFOCOM 2012 (Age of Information)** — Δ(t)=t−u(t); hay **tasa de update óptima intermedia**.
  Survey JSAC 2021. **[HECHO/alta]** → métrica para decidir si G(t) diseminado es usable: definir AoI por arista/celda y umbral de descarte.
- **Sun et al., IEEE TIT 2017 (Update or Wait)** — *zero-wait* maximiza throughput pero **NO minimiza la edad**; a veces hay
  que **esperar**. arXiv:1601.02284 · **[HECHO/alta]** → diseminar "lo más rápido posible" puede dar G(t) **más viejo**; optimizar AoI, no tasa.
- **Kadota et al., ToN 2018** — scheduling AoI-aware (servir al de mayor edad) óptimo en redes simétricas. arXiv:1801.01803
  · **[HECHO/alta]** → el scheduler debe priorizar diseminar las partes de G(t) más rancias.
- **Ayan et al., ICCPS 2019 (AoI vs VoI / control)** — el costo de control crece con la edad de la muestra. **[HECHO/media, área]**
  → puente AoI→corrección: G(t) rancio degrada cuantificablemente la decisión TDMA.
- **Bailis et al., VLDB 2012 (PBS)** — consistencia eventual con **cota probabilística de staleness** ⟨k,t⟩; en trazas, decenas
  a cientos de ms. http://pbs.cs.berkeley.edu/ · **[HECHO/alta]** → **el eslabón CRDT↔AoI**: SEC no da cota temporal, **PBS mide
  empíricamente** P(staleness ≤ t) ≥ 1−ε de tu gossip → responde "¿es G(t) lo bastante fresco?" con un número.

> **Síntesis §4 [INFER/alta]:** el scheduler es correcto mientras la **AoI de G(t) < umbral** derivado de la dinámica de red;
> CRDT da convergencia de *valor* no de *tiempo* → la cota la dan **PBS/AoI medidos**. Si las vistas divergen: **doble-asignación
> de celdas / colisiones** (análogo a forwarding loops de Sakic).

## 5. Bootstrapping / dependencia circular (control sobre el medio que controla)
- **Canini et al. (Renaissance), JCSS 2022 / ICDCS 2018** — control-plane **in-band self-stabilizing** con garantía de
  convergencia. arXiv:1712.07697 · **[HECHO/alta]** → **la cita más fuerte**: el patrón "control viaja por el medio que controla" converge.
- **Vilajosana/Pister/Watteyne, RFC 8180 (6TiSCH minimal)** — un nodo **debe seguir ya un schedule para recibir los mensajes
  que construyen el schedule** → **celda compartida mínima** carga el bootstrap. + 6P (RFC 8480), MSF (RFC 9033), arq. RFC 9030.
  **[HECHO/alta]** → **el precedente estándar más cercano**: el TDMA G(t)-driven es pariente de 6P/MSF; el bootstrap circular se
  resuelve con celda mínima/estática inicial.
- **Clausen/Jacquet, RFC 3626 (OLSR) + BATMAN-adv (open-mesh.org + Johnson IFIP 2008)** — control de topología in-band,
  calidad medida del propio tráfico. **[HECHO/alta OLSR, media BATMAN sin RFC]** → linaje LibreMesh-adyacente del daemon.

## 6. Auto-optimización por auto-medición (self-driving / SON)
- **Mestres et al., CCR 2017 (Knowledge-Defined Networking)** — telemetría + ML + control SDN en lazo **observe-learn-act**.
  https://ccronline.sigcomm.org/wp-content/uploads/2017/08/sigcomm-ccr-final92-full-letter.pdf · **[HECHO/alta]** → **cita madre
  de la mitad (a)** (el agente IA cierra el lazo). Pareja: Feamster & Rexford, ANRW 2018 (arXiv:1710.11583).
- **Aliu et al., IEEE COMST 2013 (SON) + 3GPP TS 32.500** — self-configuration/optimization/healing. **[HECHO/alta]**
  → vocabulario y precedente normativo de "la red se reconfigura desde su auto-medición".
- **Subramanian et al., IEEE TMC 2008** — mesh multi-radio construye **conflict graph medido** y reasigna canal. **[HECHO/alta]**
  → precedente directo de "medir G(t) y reconfigurar" — pero para *channel assignment*, no TDMA. Lo nuevo es enchufarlo a un scheduler TDMA vía bus distribuido.

## 7. Prior art del MISMO bus sirviendo telemetría Y control-MAC → NO EXISTE
- **[HECHO/ausencia, media-alta]:** ninguna primaria describe un **único bus de estado distribuido que sirva a la vez (a)
  telemetría para operador y (b) control para el MAC/scheduler, sobre el medio que controla.** Vecinos parciales (citar JUNTOS,
  no como un precedente): **KDN/SON** (actuación out-of-band), **6TiSCH 6P/RFC 8180** (control MAC in-band, sin telemetría de
  operador), **Renaissance** (control in-band L2–L4, no scheduler-MAC). **[INFER/alta]:** el núcleo de la contribución; el riesgo
  citable es **frescura diferencial** — telemetría laxa y control-MAC estricto comparten mecanismo → resolver con **consistencia
  diferenciada por tipo de estado** dentro del mismo bus (Sakic §2).

> Cautelas: **RR-7506** (no citar "RR-7687"); PODC'87 DOI 10.1145/41840.41841 (≠ reimpresión OSR'88); BATMAN-adv sin RFC;
> AoI↔control es un área (fijar la cita exacta); EmPOWER/OpenRadio no cuantifican fallo por consistencia eventual a nivel TDMA.
