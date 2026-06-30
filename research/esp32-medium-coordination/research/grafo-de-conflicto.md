# El grafo de conflicto como objeto formal para coordinación de acceso al medio en redes mesh WiFi comunitarias

**Investigación técnica — fundamentación de la capa FORMAL de un algoritmo TDMA/polling anti-nodo-oculto (LibreMesh).**
Fecha: 2026-06-19. Metodología: fan-out de búsquedas web + fetch de fuentes primarias + verificación adversarial (las afirmaciones marcadas *CONFIRMADO verbatim* fueron contrastadas contra el PDF/texto del paper original; los números e inecuaciones se citaron literalmente).

> **Convención de niveles de confianza:** ALTA = texto verbatim extraído de la fuente primaria. MEDIA = título/venue/autores confirmados pero el dato fino viene de abstract o fuente secundaria. BAJA = aparece solo en resumen secundario, no verificado en primaria. Cada hallazgo distingue **HECHO CITADO** de **INFERENCIA**.

---

## Tesis que estos hallazgos fundamentan

1. El problema de asignar slots **debe modelarse antes** como un **grafo de conflicto** (objeto formal con semántica precisa, no una intuición de "vecinos a 2 saltos").
2. Ese grafo es **asimétrico/dirigido** en topologías reales.
3. Debe **MEDIRSE empíricamente**, no derivarse del conteo de saltos ni de un modelo de propagación idealizado.
4. Es **variable en el tiempo** (clima, follaje, vehículos, ducting), por lo que el scheduling sobre él es un problema online, no estático.

Cada uno de los cuatro pilares tiene respaldo en fuentes primarias citables que se detallan abajo.

---

## 1. El grafo de conflicto como objeto formal

### 1.1 Definición canónica (Jain et al., MobiCom 2003)

**HALLAZGO 1.1 — La definición fundacional.** *(CONFIRMADO verbatim contra el PDF del paper, §3.2.1)*
Jain, Padhye, Padmanabhan y Qiu introdujeron el grafo de conflicto como herramienta de modelado de interferencia multi-hop. Cita literal:

> *"To incorporate wireless interference into our problem formulation, we define a conflict graph, F, whose vertices correspond to the links in the connectivity graph, C. There is an edge between the vertices l_ij and l_pq in F if the links l_ij and l_pq may not be active simultaneously."*

El paper fija la terminología explícitamente: *"We use the terms 'node' and 'link' in reference to the connectivity graph while reserving the terms 'vertex' and 'edge' for the conflict graph."*

- **Vértices = ENLACES** (links de la connectivity graph), NO nodos. Este es el punto crítico de modelado.
- **Arista de conflicto = "no pueden estar activos simultáneamente"** (interfieren).
- **Fuente:** K. Jain, J. Padhye, V. N. Padmanabhan, L. Qiu, *"Impact of Interference on Multi-Hop Wireless Network Performance"*, ACM MobiCom 2003, pp. 66–80. Versión journal en *Wireless Networks* 2005. PDF: https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/mesh-interference.pdf — ACM DL: https://dl.acm.org/doi/10.1145/938985.938993
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 1.2 — Conjuntos independientes = transmisiones simultáneas viables.** *(CONFIRMADO verbatim)*
> *"Links belonging to a given independent set in conflict graph F can be scheduled simultaneously."*

Esto conecta directamente el grafo con el scheduling: un **independent set** del grafo de conflicto es un conjunto de enlaces conflict-free que pueden compartir un slot. Las **cliques** del grafo dan cotas superiores de throughput (la utilización total de los enlaces de una clique es ≤ 1).
- **Fuente:** Jain et al., MobiCom 2003, §3.2.4. **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 1.3 — El grafo se usa para cotas de throughput y el problema es NP-hard.** *(CONFIRMADO verbatim del abstract e intro)*
> *"We model such interference using a conflict graph, and present methods for computing upper and lower bounds on the optimal throughput..."* / *"...the problem of finding optimal throughput is NP-hard..."*
- **Fuente:** Jain et al., MobiCom 2003. **Confianza:** ALTA. **Tipo:** HECHO CITADO.

### 1.2 Dos convenciones sobre los vértices (precisión terminológica)

**HALLAZGO 1.4 — "Grafo de conflicto" e "interference graph" son casi sinónimos, pero la convención de qué representan los vértices varía.**
La convención dominante (linaje Jain et al.) es **vértices = enlaces**. Sin embargo, otros trabajos usan vértices = nodos o vértices = pares/flujos. Es una ambigüedad real de la literatura, no un error: hay que **declarar explícitamente** qué convención se adopta. El término "conflict/interference graph" tiene raíz histórica en la teoría de compiladores (interference graph de Chaitin para register allocation).
- **Fuente:** síntesis sobre literatura de interference graphs (arXiv:2406.08616; Zhao et al. INFOCOM 2015). **Confianza:** MEDIA. **Tipo:** INFERENCIA respaldada.

**HALLAZGO 1.5 — "Contention graph" NO es un objeto formal distinto y establecido.**
No se encontró una definición formal separada de "contention graph" en la literatura wireless; aparece como uso laxo / sinónimo de "contention relations". *Ausencia de evidencia reportada como hallazgo.* Recomendación: usar "grafo de conflicto" (Jain) o "grafo de interferencia" y evitar "contention graph" salvo que se defina.
- **Confianza:** MEDIA (ausencia en fuentes consultadas). **Tipo:** INFERENCIA.

### 1.3 El nodo oculto como PATRÓN ESTRUCTURAL del grafo

**HALLAZGO 1.6 — El nodo oculto es la configuración estructural A→B←C sin arista A–C en el grafo de sensado.** *(CONFIRMADO verbatim contra Padhye et al. IMC 2005, §2)*
> *"If the two senders, A and C, are within the carrier sense range [of B] ... only one of them can transmit at a time. Otherwise, they may both transmit ... This is known as the hidden terminal problem."*

Reformulación estructural (INFERENCIA): A y C alcanzan a B (existen aristas A–B y C–B en el grafo de conectividad/sensado), pero **A y C no se sensan mutuamente (NO existe arista A–C)**, de modo que ambos transmiten y colisionan en B. El nodo oculto es exactamente la situación donde la **relación de conflicto NO coincide con la relación de conectividad/sensado**: dos enlaces conflictúan en el receptor aunque sus transmisores no puedan sensarse. Esto es lo que el carrier sensing por sí solo no resuelve, y la razón formal por la que se necesita coordinación explícita (TDMA/polling).
- **Fuente:** Padhye et al., IMC 2005 §2 (definición textbook del problema). **Confianza:** ALTA (cita); la reformulación "arista ausente" es INFERENCIA estructural mía, consistente con el paper.

### 1.4 Fundamento de capacidad (Gupta & Kumar 2000)

**HALLAZGO 1.7 — La capacidad por nodo se desvanece como Θ(1/√(n log n)).** *(CONFIRMADO verbatim, p.388 y Main Result 3)*
> Abstract: *"...the throughput λ(n) obtainable by each node ... is Θ( W / √(n log n) ) bits per second under a noninterference protocol."*

Justifica formalmente por qué hace falta scheduling interference-aware: a medida que crece la red, la capacidad por nodo tiende a cero salvo que se explote el reúso espacial.
- **Fuente:** P. Gupta, P. R. Kumar, *"The Capacity of Wireless Networks"*, IEEE Trans. Information Theory 46(2):388–404, marzo 2000. PDF: http://people.scs.carleton.ca/~kranakis/conferences/icdcn-papers/GuptaKumar.pdf
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.

### 1.5 Linaje MAC (la respuesta histórica al patrón estructural)

**HALLAZGO 1.8 — MACA (Karn 1990) introdujo RTS/CTS para atacar hidden/exposed terminals; MACAW (Bharghavan et al., SIGCOMM 1994) lo extendió.** *(venue/autores/año confirmados)*
- MACA: P. Karn, *"MACA — a New Channel Access Method for Packet Radio"*, ARRL/CRRL Computer Networking Conference, 1990. http://www.ka9q.net/papers/
- MACAW: V. Bharghavan, A. Demers, S. Shenker, L. Zhang, *"MACAW: A Media Access Protocol for Wireless LAN's"*, ACM SIGCOMM 1994, CCR 24(4):212–225. https://dl.acm.org/doi/10.1145/190809.190334
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.
- **Relevancia para la tesis:** RTS/CTS es una *mitigación reactiva por handshake*; tu propuesta (coordinación TDMA/polling basada en grafo medido) es la alternativa *proactiva por planificación*. Citar este linaje sitúa la contribución.

---

## 2. Modelo de protocolo vs modelo físico (SINR)

### 2.1 Los dos modelos (Gupta & Kumar 2000)

**HALLAZGO 2.1 — Modelo de Protocolo: condición binaria, pairwise, por distancia.** *(CONFIRMADO verbatim, ec. (1), p.389)*
> *"this transmission is successfully received by node X_j if |X_k − X_j| ≥ (1 + Δ)|X_i − X_j| for every other node X_k simultaneously transmitting... The quantity Δ > 0 models situations where a guard zone is specified by the protocol..."*

Es exactamente una condición **binaria, par a par, basada en distancia** con zona de guarda Δ. **Esta condición es lo que se convierte en una ARISTA del grafo de conflicto.** El modelo de protocolo ES la representación natural del grafo de conflicto.
- **Confianza:** ALTA (inecuación verificada literalmente: `≥`, factor `(1+Δ)` multiplica `|X_i−X_j|`, Δ>0 zona de guarda). **Tipo:** HECHO CITADO.

**HALLAZGO 2.2 — Modelo Físico: SINR con interferencia ACUMULATIVA (suma sobre todos los emisores).** *(CONFIRMADO verbatim, ec. (2), p.389)*
> Transmisión exitosa si: ( P_i / |X_i − X_j|^α ) / ( N + Σ_{k∈T, k≠i} P_k / |X_k − X_j|^α ) ≥ β

El denominador es ruido N **más una SUMA sobre todos los demás transmisores simultáneos** (interferencia acumulativa), umbral β, decaimiento 1/r^α con α>2.
- **Confianza:** ALTA. **Tipo:** HECHO CITADO. *(Nota: el paper rotula el cociente "SIR" pero el denominador incluye N, así que funcionalmente es SINR.)*

### 2.2 Por qué el modelo binario por rango es una aproximación y dónde falla

**HALLAZGO 2.3 — La naturaleza acumulativa del SINR: muchos interferentes débiles y distantes pueden sumar por encima del umbral aunque ninguno por sí solo lo haría.**
Esto es estructural: un modelo pairwise/binario **no puede** capturarlo, porque solo evalúa cada interferente por separado. Es el modo de falla principal del modelo de protocolo / "2 saltos".
- **Fuente:** Iyer, Rosenberg & Karnik, *"What is the Right Model for Wireless Channel Interference?"*, IEEE Trans. Wireless Comm. 2009 (orig. QShine 2006). https://eceweb.uwaterloo.ca/~cath/interference.pdf
- **Confianza:** ALTA (tesis central del paper). **Tipo:** HECHO CITADO / corolario directo.

**HALLAZGO 2.4 — Los modelos de captura binarios SOBREESTIMAN el throughput y predicen comportamiento cualitativamente distinto.** *(cita directa)*
> *"not only does the capture threshold model overestimate the throughput performance, but it also predicts different qualitative behavior"*

Iyer et al. muestran que el modelo de protocolo es matemáticamente equivalente al modelo "capture threshold" de ns2 bajo path-loss isotrópico y potencia uniforme (con Δ = CpThresh^{1/η} − 1); los tres son simplificaciones binarias que **ignoran la interferencia acumulativa**. Recomiendan explícitamente usar un modelo aditivo (SINR).
- **Fuente:** Iyer, Rosenberg & Karnik, IEEE TWC 2009 (§III, contribución C3). **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 2.5 — El modelo de protocolo "captura el comportamiento de CSMA/CA" y se corresponde con un modelo de interferencia a 2 saltos.**
La suposición común de "coloreo a 2 saltos" (tipo DRAND) cae de lleno en la familia binaria/protocolo. Es decir: **DRAND y similares NO miden interferencia; asumen un grafo derivado de la relación de vecindad a 2 saltos.**
- **Fuente:** literatura de k-hop interference models; Gupta-Kumar señalan que el modelo de protocolo "captura CSMA/CA". **Confianza:** MEDIA (convención ampliamente citada). **Tipo:** HECHO CITADO.

**HALLAZGO 2.6 — Schedules factibles bajo el modelo de protocolo pueden ser INFACTIBLES bajo interferencia acumulativa real.**
"El uso ciego del modelo de protocolo puede ser engañoso." Existe una línea de trabajo dedicada: *"How to correctly use the protocol interference model for multi-hop wireless networks"*, MobiHoc 2009 (https://dl.acm.org/doi/10.1145/1530748.1530782).
- **Confianza:** MEDIA. **Tipo:** HECHO CITADO / INFERENCIA. **Implicación directa para la tesis:** un scheduler que asuma el grafo a 2 saltos puede emitir un schedule que *parece* conflict-free pero colisiona en la realidad — exactamente el riesgo que la medición empírica mitiga.

### 2.3 Costo vs precisión, y el contrapunto del SINR

**HALLAZGO 2.7 — Bajo el modelo SINR con control de potencia no uniforme, topologías arbitrarias se schedulean en O(log² n) slots — algo inalcanzable con scheduling graph-based.**
Demuestra la ventaja teórica del modelo físico (con power control), pero a costa de complejidad.
- **Fuente:** Moscibroda, Wattenhofer & Zollinger, *"Topology Control Meets SINR: The Scheduling Complexity of Arbitrary Topologies"*, ACM MobiHoc 2006. https://www.microsoft.com/en-us/research/publication/topology-control-meets-sinr-scheduling-complexity-arbitrary-topologies/
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 2.8 — La desventaja analítica del SINR acumulativo: imposible discernir la contribución individual de cada nodo a la interferencia agregada.**
Esto es precisamente por qué las abstracciones graph-based (pairwise) siguen siendo atractivas pese a ser menos precisas — el clásico **trade-off precisión vs tratabilidad**. La estrategia de diseño razonable es: **medir** para construir un grafo de conflicto *empírico* (que captura efectos reales) pero mantener la *tratabilidad* de la abstracción de grafo para el scheduler.
- **Fuente:** literatura de effective carrier sensing bajo interferencia acumulativa (arXiv:0912.2138). **Confianza:** MEDIA. **Tipo:** HECHO CITADO.

**HALLAZGO 2.9 — Comparación fundacional graph-based vs interference-based (SINR) STDMA.**
Gronkvist & Hansson, *"Comparison between graph-based and interference-based STDMA scheduling"*, ACM MobiHoc 2001, pp.255–258. Establece la comparación empírica de que los schedules graph-based difieren de los físicamente factibles (SINR).
- **Confianza:** MEDIA-ALTA (existencia/venue/autores confirmados; resultados numéricos no extraídos). **Tipo:** HECHO CITADO. *(DOI 10.1145/501449.501450 inferido de dblp — verificar antes de citar.)*

---

## 3. Estimación MEDIDA del grafo de conflicto (no asumida)

Este es el pilar empírico de la tesis. La literatura es clara, fuerte y citable.

### 3.1 La interferencia NO es binaria — hay que medirla

**HALLAZGO 3.1 — "La interferencia no es un fenómeno binario."** *(CONFIRMADO verbatim, Padhye et al. IMC 2005)*
> *"interference is not a binary phenomenon."* El Link Interference Ratio (LIR) toma valores continuos en [0,1]; de 75 pares medidos, **24 pares tienen LIR=1 (no interfieren)** y muchos tienen valores intermedios (0.5–1).

Refuta de raíz cualquier modelo que clasifique pares como simplemente "interfieren / no interfieren". El grafo real es **ponderado**, no binario.
- **Fuente:** Padhye, Agarwal, Padmanabhan, Qiu, Rao, Zill, *"Estimation of Link Interference in Static Multi-hop Wireless Networks"*, ACM IMC 2005. https://www.usenix.org/legacy/event/imc05/tech/full_papers/padhye/padhye.pdf
- **Confianza:** ALTA (verbatim). **Tipo:** HECHO CITADO.

### 3.2 Métodos de medición

**HALLAZGO 3.2 — Método LIR (sondeo activo por pares con tráfico real).**
Padhye et al. miden el throughput UDP unicast de cada enlace aislado (paquetes 1000 B, 30 s), luego el throughput agregado de los dos enlaces operando juntos, y computan el ratio; repetido 5× por par. Es el "ground truth" de interferencia medida.
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 3.3 — El truco de broadcast O(n) para reducir overhead (BIR).** *(CONFIRMADO verbatim)*
> *"The key idea is that we can estimate unicast interference using broadcast packets, if we ignore impact of ACKs."*

En vez de probar todos los pares unicast (espacio enorme), cada nodo hace broadcast por turnos (medición single-sender O(n)) y luego pares hacen broadcast simultáneo. De ahí se computa el **Broadcast Interference Ratio (BIR)** que aproxima el LIR.
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 3.4 — Reis et al.: la reducción canónica a O(N) mediciones para N² parámetros.** *(CONFIRMADO verbatim, SIGCOMM 2006)*
> *"Seeding our models requires N trials in an N node network, in which each sender transmits in turn and receivers measure RSSI values and packet counts... which requires N trials to obtain N² parameters for an N node network."*

Reescriben el SINR clásico en términos de mediciones observables (RSSI + conteos de paquetes), alimentando modelos PHY de recepción y carrier sense. Motivación explícita: los modelos abstractos de propagación RF son *"largely inaccurate"*, por eso hay que medir la entrega directamente. Usan estos modelos para **predecir el grafo de conflicto** (§5.4).
- **Fuente:** Reis, Mahajan, Rodrig, Wetherall, Zahorjan, *"Measurement-Based Models of Delivery and Interference in Static Wireless Networks"*, ACM SIGCOMM 2006. https://dl.acm.org/doi/10.1145/1159913.1159921 — PDF: https://research.cs.washington.edu/networking/wireless/bits/sigcomm2006-interference.pdf
- **Confianza:** ALTA (verbatim). **Tipo:** HECHO CITADO.

**HALLAZGO 3.5 — Precisión medida de Reis vs SINR naïve.** *(CONFIRMADO verbatim)*
Testbed de 15 PCs 802.11 a/b/g. Error RMS de predicción de throughput (dos emisores) = **11%/9%** (802.11a/b) vs **24%/31%** para un modelo que ignora interferencia. La medición empírica es ~2–3× más precisa.
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 3.6 — Niculescu: mapa de interferencia desde tres matrices (delivery, carrier-sense, hidden-terminal).** *(método extraído del PDF)*
Construye el interference map vía broadcast. Complejidad: paso 1 (cada nodo broadcast solo) O(n); paso 2 (pares broadcast simultáneo) O(n²) en tiempo, O(n³) en almacenamiento. Muestra regularidades empíricas que permiten extrapolar: la interferencia es **lineal** en la tasa de paquetes de la fuente y del interferente, y los interferentes actúan **independientemente** (delivery bajo múltiples interferentes ≈ producto de los individuales), con correlación 0.97 modelo-medición.
- **Fuente:** D. Niculescu, *"Interference map for 802.11 networks"*, ACM IMC 2007. http://conferences.sigcomm.org/imc/2007/papers/imc30.pdf
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 3.7 — Niculescu critica los métodos O(n) broadcast-only: ignoran hidden terminals remotos.**
Argumenta que Padhye y Reis, al modelar interferencia solo desde paquetes recibidos entre estaciones, **ignoran interferentes fuera del rango de carrier-sense de ambos** (hidden terminals remotos), que tienen efecto considerable incluso en redes dispersas. Además, el mapa *completo* (multi-card/multi-canal) es super-lineal y *"prohibitive for dense networks of even moderate scale."*
- **Confianza:** ALTA. **Tipo:** HECHO CITADO. **Implicación de diseño:** advertencia importante — un método de medición barato puede subestimar conflictos por nodos ocultos remotos, justo el fenómeno que se quiere atacar.

### 3.3 Inferencia pasiva (sin tráfico de sondeo dedicado)

**HALLAZGO 3.8 — CMAP: grafo de conflicto medido pasivamente desde pérdidas observadas.**
Vutukuru, Jamieson, Balakrishnan, *"Harnessing Exposed Terminals in Wireless Networks"*, USENIX NSDI 2008. Construyen un **Conflict Map (CMAP)** aprendido de pérdidas observadas durante transmisiones concurrentes (inferencia pasiva, no sondeo all-pairs); ~2× throughput sobre CSMA en testbed de 50 nodos 802.11a.
- https://www.usenix.org/legacy/events/nsdi08/tech/full_papers/vutukuru/vutukuru.pdf
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 3.9 — Inferencia pasiva del grafo de conflicto PONDERADO desde CCA busy-time (sin tráfico de sondeo).**
Abdelwedoud, Busson et al., *"A Passive Method to Infer the Weighted Conflict Graph of an IEEE 802.11 Network"*, ADHOC-NOW 2019 (Springer LNCS; Inria HAL hal-02404943) y follow-up en *Internet Technology Letters* 2021. Infieren pesos de aristas desde mediciones de "busy-time" (CCA) ya disponibles en productos WiFi comerciales, sin añadir tráfico de sondeo — **pero asumen arquitectura AP/controller**.
- https://inria.hal.science/hal-02404943v1
- **Confianza:** MEDIA (abstract; full text bloqueado). **Tipo:** HECHO CITADO.

**HALLAZGO 3.10 — Aprendizaje estadístico del grafo de interferencia desde actividad observada.**
*"Learning the Interference Graph of a Wireless Network"* (arXiv:1208.0562) — recupera el grafo solo desde actividad observada, sin sondeo activo.
- **Confianza:** MEDIA. **Tipo:** HECHO CITADO.

### 3.4 El overhead O(n²) — números duros

**HALLAZGO 3.11 — El sondeo exhaustivo por pares es O(n²) experimentos e impracticable: ">1100 horas" para UN testbed de 22 nodos.** *(CONFIRMADO verbatim)*
Padhye: 22 nodos → 152 enlaces usables → 11.476 pares posibles → **9.168 pares** tras remover los que comparten endpoint. *"Note that testing all 9168 pairs would have required more than 1100 hours."* Por eso midieron solo 75 pares aleatorios. El BIR broadcast lo reduce a *"just over 28 hours"* para la red completa.
- **Confianza:** ALTA (todos los números verificados verbatim). **Tipo:** HECHO CITADO.
- **Síntesis del arco de overhead:** medición exhaustiva pairwise O(n²) (impracticable) → seeding broadcast O(n) (Reis "N trials", Padhye BIR, Kashyap O(N)) → pero (Niculescu) el broadcast-only pierde hidden terminals remotos y el mapa completo es prohibitivo. **La inferencia pasiva (CMAP, CCA busy-time) evita el sondeo dedicado por completo.**

**HALLAZGO 3.12 — Kashyap/Das: modelo de capacidad de enlace bajo interferencia con O(N) pasos de medición.**
Kashyap, Ganguly, Das, MobiCom 2007: predice capacidades de enlace dentro del 10% en ~90% de los casos en mesh 802.11b de 12 nodos, con seeding O(N).
- **Confianza:** MEDIA-ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 3.13 — "Practical Conflict Graphs in the Wild" (ToN 2015): el sondeo exhaustivo no escala outdoor.**
Reconocen que construir grafos de conflicto exactos por medición exhaustiva de señal no escala a redes outdoor, y usan modelos de propagación calibrados por medición para interpolar, aceptando grafos conservadores (algo sobre-cautelosos) para reducir overhead.
- IEEE/ACM ToN 2015. https://escholarship.org/uc/item/9741c0pc
- **Confianza:** MEDIA-ALTA. **Tipo:** HECHO CITADO. **Directamente relevante a redes comunitarias outdoor.**

---

## 4. Asimetría y direccionalidad — por qué el grafo debe ser DIRIGIDO

**HALLAZGO 4.1 — El sensado/interferencia es fundamentalmente asimétrico: "vista asimétrica del estado del canal".** *(CONFIRMADO verbatim, Garetto/Knightly)*
> *"the root cause of the observed behavior resides in an asymmetric view of the channel state as perceived by senders"*

Un emisor puede sensar al otro mientras el reverso no es cierto.
- **Fuente:** Garetto, Salonidis, Knightly, *"Modeling Per-flow Throughput and Capturing Starvation in CSMA Multi-hop Wireless Networks"*, IEEE/ACM Trans. Networking, ago. 2008 (orig. INFOCOM 2006). https://bpb-us-e1.wpmucdn.com/blogs.rice.edu/dist/d/12661/files/2022/11/Modeling-Per-flow-Throughput-and-Capturing-Starvation-in-CSMA.pdf
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 4.2 — La Asimetría de Información (IA) causa starvation: pérdida ~100% en una dirección, 0% en la otra; throughput difiere por factor ≥10.** *(CONFIRMADO verbatim)*
> *"Flow 1 achieves significantly less throughput than flow 2 (by a factor of 10 or higher)... in some cases p(A) is close to 100% whereas flow 2 suffers no loss"*

También documentan **"Flow-in-the-Middle" (FIM)**: un flujo central sensa dos flujos externos que no se sensan entre sí, y se muere de hambre. Taxonomía exhaustiva de pérdidas: p_co (colisiones coordinadas), p_ia (asimetría de información), p_nh (near-hidden), p_fh (far-hidden) — tres de las cuatro arraigadas en geometría de sensado asimétrica.
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 4.3 — La interferencia/pérdida se mide por par ORDENADO: P_AB ≠ P_BA.** *(CONFIRMADO verbatim, Padhye)*
> *"let P_AB be the packet loss rate from A to B, and let P_BA be the loss rate from B to A... This data gives us the average packet loss rate between every ordered pair of nodes."*

Padhye trata la asimetría como el **caso por defecto**, no la excepción: mide cada **par ordenado** (de ahí el O(n²) direccional).
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 4.4 — Roofnet: "no hay umbral claro entre 'en rango' y 'fuera de rango'"; la abstracción de vecino es una mala aproximación.** *(CONFIRMADO verbatim)*
> *"there is no clear threshold separating 'in range' and 'out of range'"* / *"the neighbor abstraction is a poor approximation of reality"* / *"Signal-to-noise ratio and distance have little predictive value for loss rate."*

Y refutan explícitamente la suposición que muchos MAC/routing hacen: *"a pair of nodes will either hear each other's control packets (e.g. RTS/CTS), or will not interfere"* — falsa porque las relaciones reales son **graduadas y direccionales**.
- **Fuente:** Aguayo, Bicket, Biswas, Judd, Morris, *"Link-level Measurements from an 802.11b Mesh Network"*, ACM SIGCOMM 2004 (Roofnet, 38 nodos urbanos). https://web.stanford.edu/class/cs244/papers/roofnet-sigcomm04.pdf
- **Confianza:** ALTA. **Tipo:** HECHO CITADO. **Pieza clave: derrumba el modelo de disco simétrico subyacente al conteo de saltos.**

**HALLAZGO 4.5 — Conclusión de modelado: el grafo de conflicto debe ser DIRIGIDO.** *(INFERENCIA fuerte)*
Como una misma pareja puede tener relación de sensado/interferencia en una sola dirección (A difiere a B pero B ignora a A), la relación de conflicto es inherentemente direccional y **no se captura con una arista no dirigida**. El nodo oculto ES un caso de relación de sensado asimétrica.
- **Confianza:** ALTA (inferencia directa de 4.1–4.4). **Tipo:** INFERENCIA respaldada.

> **Nota de gap:** no se encontró en estas fuentes un estadístico tipo "X% de los enlaces fueron asimétricos". Si el informe necesita un porcentaje único cuantificado de asimetría, requiere búsqueda dedicada (p.ej. estudios de link asymmetry de Srinivasan et al. o Kim/Shin). *Ausencia reportada.*

---

## 5. Grafos de conflicto VARIABLES EN EL TIEMPO / dinámicos

### 5.1 La interferencia es time-varying, no estática

**HALLAZGO 5.1 — El nivel de interferencia varía en espacio y tiempo; por eso se necesitan grafos PONDERADOS y RSS promediado.**
Las mediciones de RSS son dinámicas y deben promediarse en periodos largos para un valor cuantizado estable.
- **Fuente:** Zhao et al., *"Quantized Conflict Graphs for Wireless Network Optimization"*, IEEE INFOCOM 2015. https://cis.temple.edu/~wu/research/publications/Publication_files/zhaoyanchao_infocom2015.pdf
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.

### 5.2 Variabilidad de enlaces outdoor / long-distance (clave comunitaria)

**HALLAZGO 5.2 — En enlaces WiFi Long Distance (WiLD), la pérdida es altamente variable: bursts (picos transitorios) + residual losses (baseline).**
Urbanos: pérdida alta, variable, asimétrica. Rurales: baja consistente. La fuente dominante de pérdida inducida por canal es **interferencia WiFi externa**; las pérdidas inducidas por protocolo vienen de timeouts y del quiebre de CSMA sobre delays largos.
- **Fuente:** Sheth, Nedevschi, Patra, Surana, Brewer, Subramanian, *"Packet Loss Characterization in WiFi-based Long Distance Networks"*, IEEE INFOCOM 2007. https://cs.nyu.edu/~lakshmi/wild_character.pdf
- **Confianza:** ALTA. **Tipo:** HECHO CITADO. **Un modelo de enlace/conflicto estático no puede predecir el estado instantáneo.**

**HALLAZGO 5.3 — WiLDNet: las tasas de pérdida altas y variables motivan MAC adaptativo (TDMA) en lugar de 802.11 con parámetros fijos.**
- **Fuente:** Patra, Nedevschi, Surana, Sheth, Subramanian, Brewer, *"WiLDNet: Design and Implementation of High Performance WiFi Based Long Distance Networks"*, USENIX NSDI 2007. **Confianza:** ALTA. **Tipo:** HECHO CITADO. **Precedente directo de TDMA sobre WiFi long-distance.**

### 5.3 Causas físicas: lluvia, follaje, ducting, multipath

**HALLAZGO 5.4 — Atenuación por vegetación es SEASONALMENTE dependiente; a ~1 GHz, árboles con hoja atenúan ~20% más (dB/m) que sin hoja.** *(CONFIRMADO verbatim contra el PDF de la Recomendación)*
> *"At frequencies of the order of 1 GHz the specific attenuation through trees in leaf appears to be about 20% greater (dB/m) than for leafless trees."*
> *"Attenuation through vegetation is seasonally dependent. Attenuation through rich vegetation during summer is higher comparing with attenuation during winter as a result of the leaves withering."*

El movimiento del follaje (viento) causa variación temporal; el ángulo de llegada disperso también es estacional.
- **Fuente:** Recomendación ITU-R P.833-10 (09/2021), *"Attenuation in vegetation"*. https://www.itu.int/dms_pubrec/itu-r/rec/p/R-REC-P.833-10-202109-I!!PDF-E.pdf
- **Confianza:** ALTA (verbatim). **Tipo:** HECHO CITADO. **Demuestra que el follaje hace el grafo estacionalmente variable.**

**HALLAZGO 5.5 — ITU-R P.530 cubre predicción de atenuación por lluvia Y desvanecimiento por multipath para enlaces terrestres LoS.** *(CONFIRMADO contra el PDF)*
> *"This Recommendation provides prediction methods for the propagation effects... in clear-air and rainfall conditions."*

Multipath fading (§2.3) es el mecanismo de clear-air dominante en bandas de microondas bajas y trayectos largos; la atenuación por lluvia (§2.4, hidrometeoros) crece con la frecuencia. La disponibilidad del enlace es inherentemente un problema de **estadística temporal**, no un valor fijo (A₀.₀₁ desde la tasa de lluvia excedida el 0.01% del tiempo).
- **Fuente:** Recomendación ITU-R P.530-19 (09/2025). https://www.itu.int/dms_pubrec/itu-r/rec/p/R-REC-P.530-19-202509-I!!PDF-E.pdf
- **Confianza:** ALTA para el alcance. **Nuance UNCERTAIN:** la frase "multipath es *el* mecanismo dominante de clear-air mientras lluvia domina a frecuencias altas" es física estándar y está reflejada estructuralmente en P.530, pero **no es una cita textual literal**; usar como caracterización, no como quote. **Tipo:** HECHO CITADO (alcance) + INFERENCIA (ranking).

**HALLAZGO 5.6 — Ducting troposférico (propagación anómala) causa interferencia transitoria inesperada, dependiente del clima.**
Gradientes verticales extremos de refractividad (típicamente inversión térmica) permiten propagación de decenas a miles de km con poca atenuación, causando interferencia inesperada entre sistemas en la misma banda — una fuente de cambio del grafo de conflicto **dirigida por el clima**, más común en zonas costeras/marítimas.
- **Fuente:** Sirkova et al., AIP Conf. Proc. 2075 (2019); *"Tropospheric Ducting: A Comprehensive Review"*, IEEE (2025). https://ieeexplore.ieee.org/document/10858737/
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 5.7 — Incluso enlaces outdoor FIJOS (ningún extremo en movimiento) tienen variación temporal por ramas/hojas al viento y objetos cruzando la LoS.**
Evidencia directa de que "fijo" no implica "estático" en el grafo.
- **Fuente:** *"Characterization of temporal fading in urban fixed wireless links"* (IEEE Comm. Letters). **Confianza:** MEDIA (venue/año no plenamente confirmados). **Tipo:** HECHO CITADO.

### 5.4 Scheduling adaptativo, re-coloreo, histéresis

**HALLAZGO 5.8 — El TDMA mesh se formula como coloreo de grafo de conflicto, y esquemas adaptativos (ATSA) reasignan slots online expandiendo/recuperando la longitud de frame según condiciones cambiantes.**
- **Fuente:** *"Adaptive TDMA slot assignment in mesh wireless networks"* (ATSA), IEEE; EURASIP JWCN 2012. **Confianza:** MEDIA. **Tipo:** HECHO CITADO.

**HALLAZGO 5.9 — TDMA dinámico (DT-MAC): ajusta número/longitud de slots por frame según conteo de nodos y tráfico actuales (slot reassignment online sobre estructura de conflicto cambiante).**
- **Fuente:** *"A Dynamic TDMA Scheduling Strategy for MANETs Based on Service Priority"*, Sensors 2020 (PMC7767107). **Confianza:** MEDIA. **Tipo:** HECHO CITADO.

**HALLAZGO 5.10 — Histéresis contra thrashing: rutas dominantes son muy efímeras por route flapping; umbrales de histéresis mejoran significativamente la persistencia.**
Argumento canónico para amortiguar la reacción a un grafo variable en el tiempo. Babel aplica histéresis de métrica modesta por exactamente esta razón.
- **Fuente:** *"Routing Stability in Static Wireless Mesh Networks"*, PAM 2007 (Springer LNCS); corroboración Babel (arXiv:1403.3488). **Confianza:** MEDIA (texto del capítulo no extraído directamente, bien corroborado). **Tipo:** HECHO CITADO. **Relevancia: justifica formalmente histéresis/estabilidad en el re-coloreo del scheduler.**

### 5.5 Formalismo de grafos temporales

**HALLAZGO 5.11 — Temporal networks: el tiempo es una dimensión explícita; representaciones por contact sequences e interval graphs. La agregación estática pierde el orden temporal de aristas → la alcanzabilidad es no transitiva, solo valen los time-respecting paths.**
Justifica formalmente tratar el grafo de conflicto como objeto variable en el tiempo.
- **Fuente:** Holme & Saramäki, *"Temporal Networks"*, Physics Reports 519(3):97–125, 2012, DOI 10.1016/j.physrep.2012.03.001. https://piccardi.faculty.polimi.it/VarieCsr/Papers/Holme2012.pdf
- **Confianza:** ALTA (cita confirmada). **Tipo:** HECHO CITADO.

**HALLAZGO 5.12 — Time-Varying Graphs (TVG): formalismo unificador con clasificación jerárquica de clases de TVG.**
La referencia estándar para tratar la red/grafo de conflicto como objeto formal variable en el tiempo.
- **Fuente:** Casteigts, Flocchini, Quattrociocchi, Santoro, *"Time-Varying Graphs and Dynamic Networks"*, Int. J. Parallel, Emergent and Distributed Systems 27(5), 2012; arXiv:1012.0009. https://arxiv.org/abs/1012.0009
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.

---

## 6. Scheduling sobre el grafo de conflicto

**HALLAZGO 6.1 — STDMA óptimo de longitud mínima es NP-completo (link y broadcast scheduling); es esencialmente un problema de coloreo.** *(CONFIRMADO verbatim, Brar/Blough/Santi)*
> *"an optimal minimum-length STDMA schedule for a general multihop ad hoc network is NP-complete for both link and broadcast scheduling... this is closely related to the problem of determining the minimum number of colors to color all the edges (or vertices) of a graph under certain adjacency constraints."*

Resultado base atribuido a Ramanathan & Lloyd 1993. Ephremides & Truong 1990 probaron NP-completitud del broadcast scheduling.
- **Fuentes:** Brar, Blough, Santi, *"On High Spatial Reuse Link Scheduling in STDMA Wireless Ad Hoc Networks"*, ACM MobiCom 2006 (arXiv cs/0701001, https://arxiv.org/pdf/cs/0701001); S. Ramanathan & E. L. Lloyd, *"Scheduling Algorithms for Multihop Radio Networks"*, IEEE/ACM Trans. Networking 1(2):166–177, 1993; A. Ephremides & T. Truong, IEEE Trans. Comm. 38(4):456–460, 1990.
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 6.2 — DRAND: garantiza conflict-free asegurando que vecinos a 2 saltos no compartan slot (modelo de interferencia a 2 saltos); O(δ) tiempo y mensajes, δ = tamaño máx. de vecindad a 2 saltos.** *(CONFIRMADO)*
DRAND es la primera implementación totalmente distribuida de RAND (coloreo aleatorizado centralizado), implementada en TinyOS.
- **Fuente:** Rhee, Warrier, Min, Xu, *"DRAND: Distributed Randomized TDMA Scheduling for Wireless Ad-hoc Networks"*, ACM MobiHoc 2006 / IEEE TMC 8(10):1384–1396, 2009. https://ieeexplore.ieee.org/document/4803842
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.
- **Punto clave para la tesis:** DRAND **asume el grafo a 2 saltos**, no lo mide. Es exactamente el modelo binario por rango que (§2) es una aproximación que falla bajo interferencia acumulativa y nodos ocultos remotos.

**HALLAZGO 6.3 — Otras heurísticas distribuidas asumen vecindad (no medición).**
- NAMA (Bao & Garcia-Luna-Aceves, MobiCom 2001): requiere IDs de toda la vecindad a 2 saltos.
- USAP (Young, MILCOM 1996): TDMA multicanal distribuido, coordina hasta 2 saltos.
- TRAMA (Rajendran, Obraczka, Garcia-Luna-Aceves, SenSys 2003): TDMA traffic-adaptive, elección distribuida, libre de colisiones.
- **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 6.4 — El acople estimación↔scheduler: los schedulers STDMA clásicos CONSUMEN un grafo asumido, no medido.** *(CONFIRMADO verbatim)*
> *"a link schedule is usually determined from a graph model of the network... graph-based scheduling algorithms assume a limited knowledge of the interference and result in low network throughput"*

**Esta es la brecha exacta que tu tesis ataca:** los schedulers clásicos (DRAND, NAMA, USAP, Ramanathan/Lloyd) toman un grafo de conflicto derivado de vecindad a 1–2 saltos en lugar de un mapa SINR/medido. La garantía conflict-free es solo tan correcta como ese grafo.
- **Fuente:** Brar/Blough/Santi (arXiv cs/0701001). **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 6.5 — Métricas: reúso espacial = paquetes recibidos exitosamente por slot; bajo el modelo físico, σ (reúso espacial) supera a C (número de colores) como proxy de throughput.** *(CONFIRMADO verbatim)*
Las dos métricas canónicas graph-based son longitud del schedule (= nº colores C, a minimizar) y la garantía conflict-free (sin conflictos de aristas primarias/secundarias).
- **Fuente:** Brar/Blough/Santi. **Confianza:** ALTA. **Tipo:** HECHO CITADO.

---

## 7. Factibilidad en hardware barato (ESP32-class) — HALLAZGO DE RIESGO

### 7.1 El ESP32 SÍ puede sensar el grafo (primitivas presentes)

**HALLAZGO 7.1 — RSSI por paquete en modo promiscuo vía `rx_ctrl.rssi`.** Confianza ALTA. Fuente: ESP-IDF Wi-Fi driver docs (Sniffer Mode). https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/wifi-driver/

**HALLAZGO 7.2 — Buffer HW de 1600 B por frame en promiscuo + channel hopping.** Confianza ALTA.

**HALLAZGO 7.3 — CSI (Channel State Information): 64 subportadoras para paquete non-HT 20 MHz, vía `esp_wifi_set_csi()` / `_set_csi_config()` / `_set_csi_rx_cb()`.** Confianza ALTA. Fuente: ESP-IDF Wi-Fi Vendor Features. https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/wifi-driver/wifi-vendor-features.html — ref impl: https://github.com/espressif/esp-csi

**HALLAZGO 7.4 — La struct CSI incluye RSSI, noise floor de RF, timestamp y antena en `rx_ctrl`** (una captura CSI da respuesta de canal + RSSI/ruido). Confianza ALTA. *(Limitación: en el ESP32 original los primeros 4 bytes de CSI pueden ser inválidos — flag `first_word_invalid`.)*

> **Conclusión de hardware:** las **primitivas de medición** (RSSI/paquete, noise floor, CSI) necesarias para estimar un grafo de enlace/interferencia están **todas presentes** en ESP32-class. El gap NO es de sensado.

### 7.2 Los stacks mesh ESP32 NO hacen scheduling interference-aware (hallazgos negativos)

**HALLAZGO 7.5 — ESP-WIFI-MESH NO hace scheduling interference-aware, NO estima grafo de conflicto, NO hace TDMA; corre sobre CSMA/CA estándar.** *(confirmado por fetch directo del spec; la doc no menciona TDMA, conflict graphs ni interference-aware scheduling)*. Usa RSSI solo como filtro de umbral en beacons para selección de padre; el cambio de canal es reactivo (CSA root-initiated), no avoidance predictivo.
- Fuente: ESP-IDF ESP-WIFI-MESH API guide. https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/esp-wifi-mesh.html. **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 7.6 — painlessMesh NO tiene TDMA ni MAC interference-aware documentado** (app-layer sobre CSMA/CA). Búsqueda dedicada devolvió cero resultados específicos. **Confianza:** MEDIA (negativo). **Tipo:** INFERENCIA.

**HALLAZGO 7.7 — ESP-NOW usa CSMA/CA 802.11; cualquier TDMA debe implementarse en user code y está limitado por clock drift.** Experimentos: slots de 80 ms funcionan, 40 ms colisionan por drift sin sync GPS (mesh de 9 nodos). Confianza ALTA/MEDIA. Fuentes: *"Synchronized ESP-NOW for Improved Energy Efficiency"*, DTU/BlackSeaCom 2023.

### 7.3 TDMA en ESP32 existe pero NO es interference/conflict-graph driven

**HALLAZGO 7.8 — Hay trabajo peer-reviewed con TDMA custom sobre ESP-MESH (WSN de calidad de aire), pero el schedule se construye por eficiencia energética/latencia sobre un árbol de ruteo, NO desde un grafo de interferencia medido.**
- Fuente: *"An Efficient WSN Based on the ESP-MESH Protocol for ... Air Quality Monitoring"*, Sustainability (MDPI) 14(24):16630, 2022. **Confianza:** MEDIA (abstract, 403 en full text). **Tipo:** HECHO CITADO / inferencia parcial.

### 7.4 El prior art de grafo de conflicto está en ROUTERS/APs, no en ESP32

**HALLAZGO 7.9 — La literatura de conflict-graph/interference-estimation apunta a APs/routers comerciales y mesh multi-radio, no a microcontroladores.** Métodos: inferencia pasiva por CCA busy-time, conflict graphs cuantizados/embebidos. **Pero asumen arquitectura AP/controller.**
- Fuentes: Abdelwedoud/Busson (ADHOC-NOW 2019, Internet Tech. Letters 2021); Zhao et al. INFOCOM 2015. **Confianza:** ALTA. **Tipo:** HECHO CITADO.

**HALLAZGO 7.10 — LibreMesh corre sobre routers OpenWrt con radios ath9k/Atheros (BATMAN-adv L2 + Babel L3); NO es plataforma ESP32 y NO tiene scheduling basado en grafo de conflicto; el acceso al medio sigue siendo CSMA/CA 802.11 estándar.**
- Fuentes: docs LibreMesh / lime-packages. http://libremesh.org/guide/packages-selection.html — https://github.com/libremesh/lime-packages. **Confianza:** ALTA. **Tipo:** HECHO CITADO.
- **Nota de plataforma importante:** existe una **brecha de plataforma** entre el target ESP32 de tu diseño y la base instalada de LibreMesh (routers OpenWrt). Conviene aclarar en el informe si el target es ESP32, OpenWrt, o ambos.

### 7.5 Prior art adyacente: TSCH/6TiSCH (otro PHY)

**HALLAZGO 7.11 — TSCH (IEEE 802.15.4e, RFC 7554) y 6TiSCH (RFC 9030) dan MAC time-slotted + channel-hopping libre de colisiones en dispositivos constrained, con resiliencia a interferencia demostrada — pero es 802.15.4 (WPAN de baja tasa), NO WiFi, y el scheduling es slotframe-based, no derivado de un grafo de conflicto medido.**
- Fuente: RFC 7554 (IETF, 2015). https://www.rfc-editor.org/rfc/rfc7554.html. **Confianza:** ALTA. **Tipo:** HECHO CITADO. **Es el prior art más fuerte de "MAC time-slotted en dispositivo constrained", pero sobre otra radio.**

### 7.6 HALLAZGO CRÍTICO DE RIESGO — vacío de prior art

**HALLAZGO 7.12 — NO se encontró prior art que combine estimación de grafo de conflicto/mapa de interferencia MEDIDO con coordinación TDMA interference-aware sobre microcontroladores WiFi ESP32-class.**

El espacio se descompone en tres cuadrantes poblados por separado que **nadie ha unido**:
1. El ESP32 **puede sensar** el grafo (RSSI/paquete, CSI, noise floor — Hallazgos 7.1–7.4), y
2. El ESP32 **puede hacer** TDMA/time-sync en user-space (Hallazgos 7.7–7.8), pero
3. La estimación de grafo de conflicto + scheduling interference-aware existe solo en **routers/APs** (802.11, Hallazgos 7.9–7.10) o sobre **otro PHY** (802.15.4 TSCH, Hallazgo 7.11).

Ninguna fuente une (1)+(2)+(3) en MCUs WiFi baratos. El método WiFi de grafo de conflicto más cercano (inferencia pasiva por CCA busy-time) asume AP/controller; el único TDMA sobre ESP-MESH schedulea por árbol de ruteo para energía, no por interferencia medida.
- **Búsqueda:** 12 queries WebSearch cubriendo todos los ángulos; fetch exitoso de spec ESP-WIFI-MESH y doc CSI. **Confianza:** ALTA en el vacío (búsqueda exhaustiva). **Tipo:** AUSENCIA DE EVIDENCIA = hallazgo válido y entregable clave.

**HALLAZGO 7.13 — Costo O(n²) del sondeo all-pairs sobre nodos constrained: NO cuantificado en la literatura.**
No se encontró ninguna fuente que cuantifique el costo de airtime/memoria/energía del sondeo all-pairs O(n²) para construcción del grafo específicamente en ESP32-class. La literatura WSN general nota que el overhead de probing degrada la utilización del canal, pero no hay presupuesto de airtime O(n²) específico de MCU. **Pregunta abierta y riesgo de factibilidad concreto.**
- **Confianza:** MEDIA (negativo). **Tipo:** INFERENCIA / ausencia de evidencia.

---

## Síntesis ejecutiva: la cadena argumental respaldada

| Pilar de la tesis | Respaldo más fuerte (confianza ALTA, verbatim) |
|---|---|
| El grafo de conflicto es el objeto formal correcto, vértices=enlaces | Jain et al., MobiCom 2003 (§3.2.1, definición literal) |
| El nodo oculto = patrón estructural (conflicto ≠ conectividad) | Padhye IMC 2005 §2; Gupta-Kumar 2000 |
| El modelo binario por rango (2 saltos / DRAND) es aproximación que FALLA | Gupta-Kumar 2000 (protocolo vs físico); Iyer/Rosenberg/Karnik 2009 (sobreestima, interferencia acumulativa); DRAND 2009 (asume 2 saltos) |
| Hay que MEDIR el grafo, no derivarlo de saltos | Padhye IMC 2005 ("no es binario", LIR); Reis SIGCOMM 2006 (N trials, 11/9% vs 24/31%); Niculescu IMC 2007 |
| El grafo es ASIMÉTRICO/DIRIGIDO | Garetto/Knightly ToN 2008 ("vista asimétrica", factor ≥10); Padhye (P_AB≠P_BA); Roofnet SIGCOMM 2004 |
| El grafo es VARIABLE EN EL TIEMPO | ITU-R P.833-10 (follaje estacional, 20%); ITU-R P.530 (lluvia/multipath); Sheth INFOCOM 2007 (WiLD bursts); Holme-Saramäki / Casteigts (formalismo temporal) |
| Scheduling = coloreo NP-hard; schedulers clásicos ASUMEN el grafo | Brar/Blough/Santi MobiCom 2006 (NP-completo, "assume limited knowledge of interference") |
| **RIESGO: sin prior art ESP32 que una medición+TDMA interference-aware** | Ausencia de evidencia (búsqueda exhaustiva, 12 queries) — entregable clave |

### Caveats globales de la investigación
- Los hallazgos marcados **ALTA/verbatim** fueron contrastados contra el texto/ecuaciones del paper primario en fase de verificación adversarial (Jain §3.2.1; Gupta-Kumar ec. 1, 2 y Main Result 3; Padhye números >1100h/28h/24-de-75; Reis N-trials y 11/9-24/31; DRAND 2-hop/O(δ); ITU-R P.833 20%). **No se inventó ningún paper, número ni resultado.**
- Gaps reportados como hallazgos válidos: (a) "contention graph" no es objeto formal distinto; (b) no se halló un % cuantificado único de asimetría de enlaces; (c) la frase "multipath domina clear-air / lluvia domina a alta frecuencia" es física estándar pero NO cita literal de P.530 — usar como caracterización, no quote; (d) costo O(n²) de sondeo en ESP32 no cuantificado en la literatura; (e) **vacío de prior art ESP32** para el sistema completo.
- Brecha de plataforma a clarificar: LibreMesh = routers OpenWrt/ath9k, no ESP32. Definir el target.
