# 03 — Algoritmo de referencia (diseño propuesto)

> Diseño inicial para discutir y refinar con el equipo. **No es implementación; es la especificación
> de referencia** que la línea de investigación debe construir y validar. Justificación de cada decisión
> en `02-estado-del-arte.md` y el `research/`.

## 0. Qué es y qué no es este algoritmo
- **Es**: un esquema de **coordinación de acceso al medio** (familia TDMA) — concretamente **round-robin
  con polling por un coordinador**, con turnos de granularidad **milisegundos**.
- **Opera sobre**: en el banco ESP32, el sustrato real es **ESP-NOW** (el ESP32 no corre hostapd ni soporta
  4-address frames, así que **no ejecuta APuP**). En producción LibreMesh, el peering es **APuP** (plano 3).
  En ambos casos el algoritmo va por **debajo/en paralelo** de batman-adv (plano 4). No reemplaza ninguno.
  **Nunca escribir "APuP + TDMA en ESP32" como pieza integrada** (ver `05 §(A)`).
- **No es**: TDMA de microsegundos con CSMA apagado (inviable en ESP32 con API pública hoy). No es APuP
  (APuP es el sustrato de peering, no la coordinación).
- **Meta de validación**: recuperar el throughput agregado que el nodo oculto destruye, demostrando que
  la coordinación lógica de turnos basta aunque la temporización sea de ms y no de µs.

> **Actualización tras el de-risking (ver `05-viabilidad-y-asegurabilidad.md`):** el piso temporal fiable
> del ESP32 con WiFi activo es **~10 ms con guard time** (no menos); **no hay prior art** de TDMA sobre el
> MAC WiFi nativo del ESP32 resolviendo nodo oculto con múltiples Tx (es riesgo *y* novedad); y existe una
> **bifurcación de plataforma**: el radio **802.15.4** del ESP32-C6/H2 sí da TX temporizada y ACK por HW a
> nivel µs (`esp_ieee802154_transmit_at()`, TSCH) — banco honesto para validar *la lógica* con timing real,
> aunque no sea WiFi (no valida la integración con APuP). Esto ajusta los slots de §2 (usar ~10 ms, no ms
> arbitrarios) y agrega la opción 802.15.4 en §6.

## 1. Modelo del sistema y supuestos
- N nodos ESP32 (clásico o C3) en un **único canal** (ESP-NOW exige canal común; APSTA del ESP32 está atado
  a un solo canal — un solo radio, sin canal de control out-of-band real).
- Cada nodo conoce a sus vecinos directos (1 salto) por descubrimiento (beacons / probe / ESP-NOW
  broadcast). La topología puede ser **parcial** (no todos se ven) — esa es justamente la condición de
  nodo oculto.
- Existe un **canal de control** lógico por ESP-NOW broadcast (fuera del flujo de datos, mismo radio).
- Relojes locales con deriva; sincronizables a **decenas de µs** por RX-timestamp (no µs finos).

## 2. Arquitectura en componentes

> **La capa fundacional es el grafo de conflicto, no la asignación de slots (ver [`07`]).** Asignar
> turnos es *downstream* de modelar el problema: **medir → construir G(t) → detectar nodo oculto →
> colorear → adaptar**. El "vecino a 2 saltos" de abajo es solo un *proxy* del conflicto; en topología
> real el grafo es **dirigido, medido y variable en el tiempo** (`07`). 
>
> **Régimen por fase (importante, no confundir las dos cosas):**
> - **Modelar G(t) es necesario siempre que la topología no sea trivial** (paso 1-2-5 de `07`). El PoC F1
>   usa la **estrella canónica**, cuyo grafo es trivial/conocido/estático → F1 *esquiva* el modelado a
>   propósito y valida solo el mecanismo de slotting.
> - **Colorear para reúso espacial** (dejar transmitir en paralelo a nodos que no se interfieren) es
>   distinto: dentro de un clúster **el polling del coordinador ya serializa** (un grant a la vez), así
>   que el coloreo NO se necesita para evitar el nodo oculto en un clúster; aporta valor para **reúso** y
>   **bordes inter-clúster** (F3-F4).
> - **Dos paradigmas, dos fases:** *polling centralizado puro sobre grafo conocido* para **F1-F2** (lo
>   asegurable) → *STDMA distribuido con grafo medido y dinámico (DRAND + estimación)* para **F3-F4** (lo
>   abierto y el verdadero contenido de investigación).

### (A) Estimación del grafo de conflicto *(fundacional para topología real; trivial en F1)*
La versión ingenua —cada nodo difunde su lista de vecinos de 1 salto y se asume "conflicto = ≤2 saltos"
(DRAND)— es un **proxy** que el propio research desaconseja: el conflicto **no se deduce de los saltos**
(Jain MobiCom'03), es **asimétrico** (Garetto-Knightly; P_AB≠P_BA) y **acumulativo en SINR** (Gupta-Kumar;
muchos emisores débiles fuera de "rango" rompen una recepción). La versión correcta **mide** el grafo —
sondeo activo O(N) estilo Reis (SIGCOMM'06) midiendo entrega *bajo transmisión concurrente* (Niculescu),
+ inferencia pasiva por pérdidas/CCA — y lo mantiene **G(t)** re-estimándolo con histéresis. Detalle y
citas en [`07-modelado-del-grafo-de-conflicto.md`]. **Es el problema duro y lo no-asegurable (ver `05`).**

### (B) Elección de coordinador (rol, no nodo fijo)
- Por **subred de conflicto local** (clúster), se elige un coordinador con regla **max-degree + desempate
  por MAC** (determinista, sin necesidad de sync). Inspirado en LEACH/cluster-head; rotable para repartir
  carga y tolerar fallos.
- El coordinador **no** necesita ver a todos los nodos del mesh — solo a su clúster de 2 saltos. Mesh
  grandes = varios coordinadores con turnos de guarda en los bordes (problema abierto, §04).
- **Acoplamiento de supuestos a vigilar:** la regla **max-degree** y la **resolución de circularidad
  coordinador=Rx** (§C) **coinciden solo en la estrella canónica** (donde el nodo de mayor grado *es* el
  receptor común). En multi-hop general el de mayor grado no es el Rx relevante y la resolución geométrica
  deja de garantizar que todos los contendientes oigan los grants. Para F1-F2 (estrella) es válido; para
  F3-F4 hay que revisar la regla de elección.

### (C) Round-robin con polling y reserva ligera
- El tiempo se divide en **superframes** de duración T (ej. 100-200 ms; **a calibrar**), subdivididos en
  **slots de ~10 ms con guard** (el piso fiable medido; no asumir slots más finos sobre WiFi). El
  coordinador difunde un **grant** ESP-NOW: "slot k → nodo X" en orden round-robin sobre los miembros del
  clúster que tienen tráfico.
- **Solo el nodo con grant transmite datos** en su slot; los demás callan. Aquí está el anti-hidden-node:
  no dependemos del carrier-sense, dependemos del turno lógico.
- **El canal de control no es circular en el caso que importa (resolución geométrica, revisión externa):**
  en la topología canónica del nodo oculto (A y C ocultos entre sí, ambos oídos por B=Rx), **si el
  coordinador es B, entonces A y C sí oyen los grants** — el control plane se resuelve exactamente donde el
  data plane fallaba. *Supuestos que esto expone (y que hay que acotar):* (i) el coordinador debe estar en
  rango de todos los contendientes — cierto en estrella, **abierto en multi-hop general**; (ii) **cold-start**:
  antes de fijar turnos, los grants contienden como todo lo demás. El sync FTSP (decenas de µs vs guard de
  10 ms) NO es el cuello de botella; sí lo es la **entrega del grant bajo carga en topologías no-estrella**.
- **"Adaptativo" = slots ponderados por demanda, NO round-robin a secas (precisión exigida por la revisión):**
  cada nodo señaliza en su grant la ocupación de su cola; el coordinador (a) **salta** los slots de nodos sin
  tráfico (*skip*) y (b) **pondera** la cantidad/duración de slots por demanda. Round-robin puro (turnos
  fijos rotados) **no es adaptativo** — el término solo es defendible con esta asignación ponderada por cola
  (lección de PCF/HCCA, que sondeaban a ciegas). Si no se implementa la ponderación, decir "round-robin
  rotativo", no "adaptativo".
- **Best-effort residual**: un tramo del superframe queda como ventana de contención CSMA/CA para
  tráfico no coordinado y para nodos nuevos que aún no entraron al schedule (híbrido TDMA/CSMA — el
  "híbrido" que prometía la propuesta ARDC 2024).

## 3. Sincronización y presupuesto temporal
- El coordinador difunde su tiempo lógico en cada superframe (estilo **FTSP**: timestamp en el momento
  de TX, los receptores corrigen con su RX-timestamp). Precisión esperada: **decenas de µs**. **El sync NO
  es el cuello de botella** (decenas de µs « slot de ms); lo es el **jitter de entrega/ejecución** del
  ESP32 con WiFi activo.
- **Presupuesto temporal coherente con lo medido (`research/viabilidad-esp32-tdma.md §2`):** el jitter no
  es de ~1–2 ms sino que tiene **outliers de 7 ms, apilamiento UDP de 10–20 ms y bloqueos de flash de
  ~80 ms** (WiFi en core 0). Un guard fijo de 1–2 ms **no** los absorbe. Estrategia correcta: **(a)** slot
  de ~10 ms con guard de ~2–3 ms para el caso típico, **(b)** tratar el outlier no con guard gigante (que
  duplicaría el slot y mataría la eficiencia) sino con **descarte de slot + re-sync** — un slot que se
  pasa de su ventana se pierde y el nodo espera su próximo turno. El claim de 10 ms **asume** re-sync
  frecuente y tolerancia a descarte ocasional; cuantificar el % de slots descartados es trabajo de F1-F2.

## 4. Por qué cada decisión (trazabilidad al estado del arte)
| Decisión | Fundamento |
|---|---|
| Round-robin con polling (no TDMA estático) | PCF/HCCA, Nv2 group polling, airMAX; evita sync de µs |
| Señalizar demanda + skip | lección del fracaso de PCF (sondeo a ciegas) |
| Restricción de 2 saltos | DRAND/STDMA; define formalmente quién no comparte turno bajo nodo oculto |
| Coordinador por clúster max-degree, rotable | LEACH; Bully falla en topología parcial |
| Sync FTSP-like, sin GPS | RBS→TPSN→FTSP; GPS por nodo (Mimosa) es caro/innecesario para ms |
| Ventana CSMA residual (híbrido) | propuesta ARDC 2024; admite nodos nuevos y best-effort |
| Granularidad de ms, no µs | límite real del ESP32 con API pública (`research/esp32-bajo-nivel.md`) |
| ESP-NOW broadcast para grants | sin límite de receptores; sub-2 ms; canal común (`research/esp-now.md`) |
| ESP-NOW como sustrato en ESP32; APuP solo en producción LibreMesh | separación de planos (`01 §0`, `05 §A`) |

## 5. Riesgos de diseño conocidos (honestidad técnica)
- **ESP-NOW usa CSMA/CA por debajo** → el propio grant puede colisionar/retransmitir (hasta 31 veces).
  Mitigación: grants cortos, redundantes y temprano en el superframe; el coordinador es el único que
  transmite grants en su ventana.
- **Jitter de ms por WiFi en core 0** → bandas de guarda amplias; ISR de timing en core 1.
- **Bordes entre clústeres de coordinadores** → coordinación inter-clúster es problema abierto.
- **Overhead de coordinación vs ganancia** → en topologías sin nodo oculto el esquema puede *perder*
  frente a CSMA/CA (trade-off dependiente de carga, Jaffrès-Runser). Por eso debe ser **activable por
  condición de red**, no permanente.
- **No es TDMA de µs** → no captura ganancias de agregación/airtime fino; eso requiere `esp32-open-mac`
  o hardware abierto (la línea futura).

## 6. Camino de implementación incremental (cada paso valida algo)
1. **PoC 2 nodos + 1 receptor** (el escenario `hidden-node.md`, **coordinador = el receptor**): medir
   throughput agregado coordinado **vs CSMA/CA y vs RTS/CTS** (las tres curvas, ver `04 §2`). Es el
   experimento mínimo que prueba o refuta la tesis. *Polling centralizado puro, sin coloreo (Opción A).*
2. **Coordinador + N=3–5 en un clúster**: round-robin con skip + ponderación por demanda; medir Jain y
   latencia. *Sigue siendo polling puro.*
3. **Topología parcial real** (2 saltos): construir el grafo de conflicto distribuido.
4. **Multi-clúster** + bordes.
5. **Port del concepto a OpenWiFi** (Low-MAC abierta) para la versión TDMA de µs — la referencia para
   hardware moderno.

**Bifurcación de plataforma (decidir temprano):** para los pasos 2-3, ¿validar la lógica sobre **WiFi/
ESP-NOW** (slots de ~10 ms, integración realista con el problema APuP/LibreMesh, pero timing grueso) o
sobre **802.15.4** del ESP32-C6/H2 (slots y ACK por HW a nivel µs vía `transmit_at()`/TSCH, timing real,
pero no es WiFi)? **El banco load-bearing es WiFi/ESP-NOW** (es el radio del problema). El 802.15.4 va en
**cuarentena: sanity-check de correctitud algorítmica** (validar el scheduler donde el timing fino es
gratis = casi tautológico, TSCH ya hace esto), explícitamente **fuera del path del claim WiFi y droppable
si aprieta el budget**. No venderlo como parte central — un revisor lo leería como dilución del mensaje.
