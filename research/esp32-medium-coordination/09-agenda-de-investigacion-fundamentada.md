# 09 — Agenda de investigación fundamentada (basamento para el desarrollo)

> Profundización de **todos los ángulos abiertos** de `07`/`04`, con la bibliografía que encamina el desarrollo. Cada
> ángulo: el problema, el resultado/cita que lo encamina, y la decisión de diseño. Evidencia completa con URLs y confianza
> en `research/estimacion-grafo-medicion.md`, `research/dinamica-temporal-estabilidad.md`,
> `research/scheduling-distribuido-crosslayer.md`, `research/shared-state-data-t.md`.

## A. Medición/estimación del grafo de conflicto
- **Sondeo activo:** **Reis (SIGCOMM'06)** muestra que se siembra el modelo con **O(N) experimentos** (cada nodo emite por
  turno, el resto mide) → O(N²) parámetros; **Niculescu (IMC'07)**: la interferencia es **lineal** en la tasa → extrapolar a
  O(n), pero requiere *downtime* → planificar el sondeo en ventanas ociosas. **Niculescu** además exige medir entrega *bajo
  transmisión concurrente* (broadcast-solo pierde ocultos remotos).
- **Inferencia pasiva (costo cero):** **CMAP (Vutukuru, NSDI'08)** deriva conflicto de pérdidas durante tráfico normal
  (+47%); **Abdelwedoud/Busson (AdHoc-Now'19)** pondera aristas por *busy-time*/CCA — directamente trasladable a los
  contadores del ESP32; **Rossi (AdHoc-Now'14)**: PER+airtime bastan en commodity. **Yang/Draper/Nowak (2017)** da la ley de
  escala (observación ∝ **d²·log n**) para dimensionar cuánto observar.
- **RSSI vs CSI:** **Aguayo/Roofnet (SIGCOMM'04)**: el RSSI es **mal predictor** de entrega → no construir el grafo solo con
  umbral de RSSI; **Halperin (SIGCOMM'10)**: CSI → *effective SNR* sí predice entrega. **ESP32-CSI-Tool** (Hernandez,
  WoWMoM'20) + esp-csi dan ~**64 subportadoras** sobre ESP-IDF (ojo `first_word_invalid`).
- **Modelo de protocolo vs SINR:** **Maheshwari/Jain/Das (SenSys'08, en motes)**: los modelos simples (incl. 2-saltos) son
  **inexactos** frente al SINR aditivo; **Shi/Hou (MobiHoc'09)**: ajustar el rango de interferencia hace factibles los
  horarios. **Decisión:** *physical-lite* — protocolo calibrado por medición, no SINR exacto (NP-hard, Goussevskaia'07).
- **Incremental/dirigido:** **Ahmed (CoNEXT'08, micro-probing online)** + **Zheng-Tajer (SPAWC'20, local probes binarios)** +
  **Zhou (TON'14, interpolar lo no medido)** → re-sondear solo aristas inciertas (contribución propia, no hay paper canónico).
- **Decisión de diseño:** **modo dual** — pasivo continuo (CCA/PER, costo cero) como base + sondeo activo dirigido O(N) en
  ventanas ociosas para las aristas que el pasivo deja inciertas. **Riesgo:** sin prior art en ESP32/WiFi (§F).

## B. Cadencia de re-estimación (dinámica temporal)
- **No hay un periodo único** (`research/dinamica-temporal`). Evidencia: omni **~1 s** (Roofnet); direccional con margen
  **horas/por evento** pero **cerca del codo SNR decenas de min** (Chebrolu'06); interferencia urbana en **clusters de
  minutos** (Sheth'07); comunitario **predecible por hora del día** (Molina'15); follaje con **ritmo ∝ viento** + corrimiento
  **estacional** (ITU-R P.833); ducting **nocturno (horas)**.
- **Decisión:** **G(t) de dos capas** — baseline lento condicionado diurnamente (horas–estación) + capa reactiva (min–s)
  para interferencia, con **trigger dirigido por interferencia/viento** (no por reloj fijo), delegando la burstiness sub-s a
  **FEC/ARQ** (no a re-scheduling).

## C. Estabilidad / anti-thrashing (cambio persistente vs ruido)
- Convergencia de toda la literatura: **nunca reaccionar a una muestra**. Tres mecanismos compuestos:
  (1) **suavizar el input** — EWMA/ventana (ETX τ=1s/w=10s, De Couto'03); (2) **doble umbral Schmitt** (OLSR HIGH 0.8/LOW
  0.3; TI SCEA046); (3) **conmutar solo con AND-de-dos-señales + dwell-time** — **Babel (RFC 8966)** es la analogía exacta
  (cambiar solo si el alternativo gana en instantáneo **Y** EWMA), reforzada por el **Time-to-Trigger** de handover (3GPP).
- **Caveat de oro (BGP RFD, Mao SIGCOMM'02):** el damping mal hecho **empeora** la convergencia → empezar con **banda ancha**
  (RFC 7196) y **excluir de la penalty los transitorios endógenos** del propio re-plan/medición (solo contar cambios exógenos).

## D. Scheduling: teoría, mecanismo distribuido y el régimen
- **Fundamento (¿sirve para grafo que cambia?):** **Tassiulas-Ephremides (1992)** max-weight es throughput-óptimo; **Tassiulas
  (1997)** + **Neely-Modiano-Rohrs (2005)** prueban que **sí aplica a topología/canal variable** si se observa el estado
  vigente por época (canal estacionario-ergódico) → **valida re-medir el grafo por época y ponderar slots por colas**.
- **Lo que un distribuido puede:** Max-Weight exacto es centralizado/NP-hard → **Greedy Maximal Scheduling**; **Joo-Lin-Shroff
  (ToN'09)** permite **acotar** el throughput perdido = *local-pooling factor* del grafo medido; **Brzezinski-Modiano (2006)**:
  **particionar** en clústeres donde GMS logra ~100%.
- **Mecanismo distribuido concreto:** **DRAND** (coloreo distancia-2, O(δ) local) como baseline; **6TiSCH 6P (RFC 8480) +
  MSF (RFC 9033)** como **el precedente de scheduling negociado celda-a-celda** con add/delete/relocate y **make-before-break**
  ante churn → plantilla directa. **Bootstrap** del schedule en frío: coloreo radio no estructurado (Moscibroda-Wattenhofer'05).
- **Régimen (corrección ya en `03 §2`):** dentro de un clúster **el polling serializa** (no hace falta colorear); el coloreo
  es para reúso espacial / multi-clúster (F3-F4). **Re-coloreo incremental tiene teoría de cotas:** **Barba (2017)** —no se
  minimiza colores Y churn a la vez; **Solomon-Wein (2018)** — en grafos esparsos O(1) churn/update; **Herman-Tixeuil (2004)**
  TDMA self-stabilizing = coloreo distancia-2 (corrección formal del repair). Modelar G(t) como **TVG** (Casteigts'12:
  footprint conservador + ρ para reúso oportunista).

## E. Elección/mantenimiento de coordinador
- **Precedente exacto:** **Gerla-Tsai (1995)** — el clusterhead **es** el coordinador de scheduling TDMA. Elección ponderada:
  **WCA** (Chatterjee'02, score combinado + re-elección on-demand), **HEED** (re-elección periódica con conectividad casi-segura).
- **Bajo churn/partición:** **Malpani-Welch-Vaidya (DIALM'00)** elección de líder **auto-estabilizante** (un líder por
  componente, tolera cambios concurrentes); **Bettstetter-Krausser (2001)** cuantifica re-elección vs movilidad → elección
  **consciente de estabilidad** (penalizar cambiar de coordinador, como la histéresis de §C).
- **Acoplamiento ya notado (`03 §B`):** max-degree y coordinador=Rx coinciden solo en la estrella; en multi-hop revisar la regla.

## F. Interacción cross-layer con batman-adv (el lazo a domar)
- **El riesgo, nombrado por sus autores:** **Kawadia-Kumar (2005)** "cross-layer → adaptation loops → stability paramount".
  **Superficie de acoplamiento exacta:** **BATMAN V** usa **ELP (throughput medido) + OGMv2** → el scheduler perturba el
  airtime que ELP mide → ruteo evita el enlace con pocos slots → menos demanda… lazo.
- **El trade-off, dicho por los diseñadores de ETX (De Couto'03):** ETX se diseñó para **no** ser load-adaptive y **evitar la
  oscilación** — exactamente lo que el scheduler reintroduce. **Decisión:** o (a) **desacoplar temporalmente** — el ruteo ve un
  airtime **filtrado/histéresis** mucho más lento que el reajuste de slots — o (b) **co-diseñar con backpressure**
  (Neely-Modiano-Rohrs; Georgiadis-Neely-Tassiulas: scheduling+ruteo conjunto estable). Evidencia de campo del peligro: flapping
  en AREDN.

## G. Cold-start / bootstrap (medio congestionado sin schedule)
- **Degradación graciosa:** **Z-MAC (SenSys'05)** — arranca CSMA, transiciona a TDMA, **"in the worst case falls back to
  CSMA"** → el schedule es capa de optimización *sobre* contención, **no precondición**. Es la propiedad objetivo del PoC.
- **Ley de tiempo del descubrimiento:** **coupon-collector (Vasudevan'09)** — **Θ(n ln n)** sin collision-detect, **Θ(n)** con
  feedback. **Loop completo:** TRAMA (descubrir→intercambiar→elegir). **Corrección:** self-stabilization (Dijkstra'74).
- **Umbral de factibilidad falsable:** **Petig-Schiller-Tsigas (2014)** — TDMA self-stabilizing **sin referencia externa**
  (sin clock/GPS/CD, = WiFi mesh real): **imposible si `τ < max{2δ, χ₂}`**, existe si **`τ > max{4δ, χ₂}+1`** → cuánto headroom
  de frame sobre densidad local necesita el cold-start. **La restricción de diseño más concreta y verificable.**

## H. Shared-state como bus de `data(t)` → ver `08`
Resumen: el plano-modelo (G(t) lento, red-wide) viaja por shared-state; el plano-ejecución (grant por slot rápido, local) por
ESP-NOW. El bug de convergencia TTL se vuelve load-bearing; frescura cuantificable por **AoI/PBS**; bootstrap circular con
precedente formal (Renaissance/6TiSCH); fuerza la decisión SensorThings-vs-NATS/Automerge hacia el modelo de eventos efímeros.

## Hilo argumental para el desarrollo (cómo se encadena)
**Medir G(t) (A, modo dual) → a la cadencia correcta (B, dos capas) → con estabilidad (C, Schmitt+Babel+dwell) → agendar
(D, Max-Weight/GMS sobre grafo medido, repair incremental acotado) → coordinado (E, clusterhead estable) → sin pelear con el
ruteo (F, desacoplar o co-diseñar) → arrancando del caos (G, Z-MAC + umbral τ>max{4δ,χ₂}) → todo sobre el bus de `data(t)`
(H/`08`, con frescura AoI/PBS y consistencia diferenciada).** Cada flecha tiene su fundamento citable. **El aporte de
investigación no es repartir slots (ingeniería conocida) sino la integración medida-dinámica-distribuida en hardware barato
— un hueco genuino en la literatura** (`research/scheduling-distribuido-crosslayer.md §6`).
