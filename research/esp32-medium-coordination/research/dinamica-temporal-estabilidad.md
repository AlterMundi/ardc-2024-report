# Research — Dinámica temporal de G(t), cadencia de re-estimación y estabilidad

> Para `07`/`09`. Cita: autor, venue/año, URL, "cómo encamina". **[HECHO]**/**[INFER]**, confianza marcada.
> **Conclusión transversal:** un periodo de re-estimación único y fijo es el modelo equivocado → **G(t) de dos capas**.

## 1. Escalas temporales reales del canal (fija el periodo de re-estimación)
- **Aguayo et al. (Roofnet), SIGCOMM 2004** — la mayoría de enlaces tienen pérdida **estable seg-a-seg**; minoría
  bursty a 200 ms; pérdidas intermedias = **multipath**, no interferencia. **[HECHO/alta, verbatim]**
  → mesh omni: re-estimar a **~1 s**; la minoría bursty con peso probabilístico, no re-scheduling.
- **Chebrolu/Raman/Sen, MobiCom 2006** — enlaces direccionales con margen: PER **~invariante** (señal varió 1–2 dB/24 h,
  error <0.1%); clima sin efecto a 2.4 GHz; **cerca del codo SNR** los mismos 1–2 dB → error 1.5%→45% en 30 min;
  interferencia externa con patrón **diurno** (~1 am, hasta ~4 pm). https://www.cse.iitk.ac.in/users/braman/papers/2006-dgp-meas.pdf
  · **[HECHO/alta]** → aristas de conectividad cuasi-estáticas (horas/por evento); el trigger debe ser **dirigido por
  interferencia**, no por reloj; excepción peligrosa = enlace en el codo (decenas de min).
- **Sheth et al., INFOCOM 2007** — WiLD urbano: bursts cortos **<0.3 s** (Poisson, inter-arribo ~15 s) + bursts largos
  en clusters de **minutos (hasta 25–30 min)**, correlacionados; fondo 1–10%. https://cs.nyu.edu/~lakshmi/wild_character.pdf
  · **[HECHO/alta]** → aristas de interferencia: re-estimar a **pocos minutos** (clusters largos); bursts sub-s a FEC/ARQ.
- **Patra et al. (WiLDNet), NSDI 2007** — pérdidas WiLD por interferencia "not multi-path"; slot y FEC **configurables**.
  · **[HECHO/alta]** → TDMA es el marco; periodo de slot es parámetro tensionado contra la variabilidad.
- **Molina et al., CNBuB/WiMob 2015** — calidad de ruta comunitaria (FunkFeuer OLSR) **predecible** por series temporales
  (MAE ~2.4%, dependiente de hora del día). https://arco.e.ac.upc.edu/wiki/images/f/f3/Molina_cnbub15.pdf · **[HECHO/med-alta]**
  → re-estimador **predictivo** condicionado por hora del día > muestreo instantáneo.
- **ITU-R P.833-9/10** — atenuación por vegetación *"varies rapidly when the vegetation moves"*, **σ ≈ v/4** (viento);
  estacional **2 dB @900 MHz, 8.5 dB @2200 MHz**. **[HECHO/alta números; media transferencia a 2.4/5 GHz]**
  → enlace que roza follaje: corrimiento **estacional** + fading cuyo ritmo ∝ viento → **trigger consciente del viento**.
- **ITU-R P.530-16** — fading de lluvia **mucho más largo** que multipath; evento ≥10 s; **lluvia despreciable <5 GHz**.
  **[HECHO/alta]** → para WiFi 2.4/5 la lluvia es menor; para backhaul mm-wave, outages de minutos.
- **Ducting troposférico** (secundarias; primario ITU-R P.452) — eventos de **horas–día**, firma nocturna/diurna.
  **[HECHO/media]** → aristas de interferencia de largo alcance con ciclo nocturno → **grafo condicionado diurnamente**.
- **Movilidad:** mesh fijo → no aplica (la coherencia ~12.5 ms @2.4 GHz/10 m/s solo aplica a scatterers en movimiento = caso viento).

> **Bottom line [INFER]:** **G(t) de dos capas** — (1) baseline lento condicionado diurnamente (horas; día/noche, ducting,
> follaje estacional) + (2) capa reactiva rápida (min–s) para interferencia, delegando burstiness sub-s a FEC/ARQ.

## 2. Histéresis / anti-thrashing (cambio persistente vs ruido) — todas las fuentes: nunca reaccionar a una muestra
- **Villamizar et al., RFC 2439 (BGP Route Flap Damping)** — penalty fija por evento + **decaimiento exponencial**; doble
  umbral suppress/reuse. **[HECHO/alta]** → modelar "persistente vs ruido" como penalty acumulada con decay + doble umbral.
- **Mao et al., SIGCOMM 2002** — RFD **empeora** la convergencia; un retiro único puede suprimir ~1 h por flaps espurios.
  https://web.eecs.umich.edu/~zmao/Papers/sig02.pdf · **[HECHO/alta]** → contar a la penalty **solo cambios exógenos**, no los
  transitorios endógenos del propio re-plan/medición.
- **RIPE-580 / RFC 7196 (2014)** — la perilla principal es **ensanchar la banda** (suppress ~6000–12000), no la tasa de
  decay. **[HECHO/alta]** → empezar con banda de histéresis **ancha**.
- **De Couto et al. (ETX), MobiCom 2003** — `ETX = 1/(d_f·d_r)` con **media móvil** (τ=1 s, ventana w=10 s); ETX se diseñó
  para **no** ser load-adaptive y evitar oscilaciones. https://pdos.csail.mit.edu/papers/grid:mobicom03/paper.pdf · **[HECHO/alta]**
  → suavizar cada peso sobre ventana N (una muestra lo mueve 1/N).
- **Chroboczek/Schinazi, RFC 8966 (Babel)** — métrica instantánea `m` + EWMA `ms`; **conmutar solo si `m(R')<m(R)` Y
  `ms(R')<ms(R)`** (AND de dos señales); el RFC dice que la selección ingenua sobre ETX *"might lead to persistent
  oscillations"*. https://www.rfc-editor.org/rfc/rfc8966.html · **[HECHO/alta]** → **la analogía más directa**: conmutar de
  schedule solo si el alternativo gana en instantáneo **Y** suavizado.
- **Clausen/Jacquet, RFC 3626 (OLSR)** — EWMA de link-quality + banda Schmitt (HIGH=0.8, LOW=0.3). **[HECHO/med-alta]**
  → gate de doble umbral para declarar una arista de conflicto "presente".
- **TI SCEA046 (Schmitt trigger)** — dos umbrales; banda = inmunidad al ruido. **[HECHO/alta]** → justificación formal.
- **3GPP TS 36.331 (handover A3)** — histéresis de magnitud **Y** Time-to-Trigger (100–320 ms) anti ping-pong.
  **[HECHO/med-alta]** → combinar **margen Y dwell-time** (mejora persiste T muestras) antes de re-planificar.

> **Diseño [INFER]:** (1) suavizar input (EWMA/ventana τ≈3–10× muestreo) + (2) doble umbral Schmitt + (3) conmutar solo con
> **AND-de-dos-señales + dwell-time**. Caveat BGP: banda ancha inicial y excluir transitorios endógenos.

## 3. Scheduling online / re-coloreo incremental (trade-off responsividad-vs-estabilidad = TEOREMA)
- **Barba et al., WADS 2017 / Algorithmica 2019** — **no se puede minimizar a la vez nº de colores Y de recoloreos**;
  cota inferior Ω(N^{1/(2c(c−1))}) recoloreos/update. arXiv:1708.09080 · **[HECHO/alta]** → la tensión es un teorema.
- **Solomon & Wein, ESA 2018 / TALG 2020** — en grafos **esparsos**: O(1) recoloreos/update con polylog(n) colores.
  arXiv:1904.12427 · **[HECHO/alta]** → mesh suele ser esparso → **churn constante por cambio topológico** es factible.
- **Bhattacharya et al., SODA 2018** — (Δ+1)-coloreo dinámico en O(log Δ) (luego O(1)) update. arXiv:1711.04355 · **[HECHO/alta]**
- **Halldórsson & Szegedy, TCS 1994** — coloreo **online irrevocable** es Ω(n/log²n) → **el recoloreo (revocabilidad) es
  esencial**. **[HECHO/alta]** → todo buen repair TDMA debe permitir re-asignar slots.
- **DRAND (Rhee et al., MobiHoc 2006)** — coloreo distancia-2 distribuido, **O(δ)** (vecindario 2-hop), sin reloj global,
  **adapta a cambios locales sin overhead global**. **[HECHO/alta]** → baseline de local-repair.
- **USAP (Young, MILCOM 1996)** — slots de pool local + announce/confirm 2-hop + resolución de conflictos, sin livelock.
  **[HECHO/alta]** → plantilla operacional de repair bajo grafo en movimiento.
- **Herman & Tixeuil, ALGOSENSORS 2004** — TDMA **self-stabilizing** = coloreo distancia-2; convergencia local O(1).
  https://disco.ethz.ch/courses/ws0405/seminar/papers/distributedTDMA.pdf · **[HECHO/alta]** → corrección formal del repair.
- **Holme & Saramäki, Physics Reports 2012** — redes temporales: caminos **time-respecting**; el grafo agregado
  **sobreestima** la conectividad. arXiv:1108.1780 · **[HECHO/alta]** → snapshot ≠ realidad temporal.
- **Casteigts et al., IJPEDS 2012 (TVG)** — G=(V,E,T,ρ,ζ), *journey*, **footprint**. arXiv:1012.0009 · **[HECHO/alta]**
  → modelar G(t) como TVG: footprint = grafo peor-caso a colorear; ρ habilita **reúso oportunista**.
- **Ghaderi & Srikant (Spatial CSMA SIR), arXiv:1504.07825** — CSMA para SIR variable es throughput-óptimo y supera a los
  de grafo de conflicto. **[HECHO/med-alta]** → caveat: el grafo binario es aproximación; el error crece con la densidad.

## 4. Marco de control / colas sobre canal variable (el scheduler ES un controlador)
- **Tassiulas & Ephremides, IEEE TAC 1992** — **max-weight throughput-óptimo** (región de estabilidad ⊇ toda otra);
  prueba por drift de Lyapunov. DOI 10.1109/9.182479 · **[HECHO/alta]** → el benchmark de todo scheduler.
- **Neely/Modiano/Rohrs, JSAC 2005** — backpressure con estado de canal **variable** estabiliza con delay acotado si las
  tasas están en el interior de la región de capacidad (canal estacionario-ergódico, observado por slot).
  http://www.mit.edu/~modiano/papers/J28.pdf · **[HECHO/alta]** → **fundamento de que Tassiulas-Ephremides aplica a G(t)**.
- **Georgiadis/Neely/Tassiulas, FnT Networking 2006** — define **región de capacidad** + drift de Lyapunov + cross-layer.
  DOI 10.1561/1300000001 · **[HECHO/alta]** → columna vertebral formal.
- **Neely, M&C 2010** — **drift-plus-penalty**: estabilidad + objetivo a O(1/V) con O(V) backlog. **[HECHO/alta]** → la perilla V.
- **DiffQ (INFOCOM'09) / Horizon (MobiCom'08)** — backpressure en 802.11 commodity (mapeo a EDCA). **[HECHO/alta]** → realizable.
- **Ji/Joo/Shroff, INFOCOM 2011** — *last-packet problem* de backpressure por cola → **backpressure por delay** lo cura
  preservando optimalidad. arXiv:1011.5674 · **[HECHO/alta]** → estable ≠ baja latencia; peso por delay corrige.
- **Separación de escalas [INFER/media]:** las garantías Lyapunov exigen que el **lazo de control sea más rápido que la
  coherencia de canal** → las cadencias de §1 (s–min) deben ser más lentas que el paso de control. **Puente §1↔§4.**

## 5. Bootstrapping / cold-start desde medio congestionado
- **Z-MAC (Rhee et al., SenSys 2005)** — arranca CSMA, transiciona a TDMA, **"in the worst case falls back to CSMA"**;
  robusto a fallos de slot/sync; >3× B-MAC bajo contención. **[HECHO/alta]** → **el análogo más directo**: el schedule es
  capa de optimización *sobre* contención, no precondición. Degradación graciosa = propiedad objetivo.
- **DRAND (MobiHoc 2006)** — scheduling desde cero, **O(δ)** local. **[HECHO/alta]** → la mitad "agendar".
- **TRAMA (Rajendran et al., SenSys 2003)** — loop completo descubrir→intercambiar→elegir desde tráfico medido. **[HECHO/alta]**
- **Birthday protocols (McGlynn/Borbash, MobiHoc 2001)** — descubrimiento Tx/Rx/sleep probabilístico. **[HECHO/alta]**
- **Vasudevan et al., MobiCom 2009 (coupon-collector)** — descubrir n vecinos = **Θ(n ln n)** sin collision-detect, **Θ(n)**
  con feedback. https://web.cs.umass.edu/publication/docs/2009/UM-CS-2009-032.pdf · **[HECHO/alta]** → ley de tiempo del cold-start.
- **Dijkstra, CACM 1974 (self-stabilization)** — converge desde **cualquier** estado en pasos acotados. **[HECHO/alta]**
  → el criterio de corrección del bootstrap (convergencia + clausura).
- **Petig/Schiller/Tsigas, MED-HOC-NET 2014** — TDMA self-stabilizing **sin referencia externa** (sin clock/GPS/CD):
  **imposible si τ < max{2δ, χ₂}**; existe si **τ > max{4δ, χ₂}+1**. arXiv:1308.6475 · **[HECHO/alta]** → **umbral de
  factibilidad falsable**: cuánto headroom de frame sobre densidad local necesita el cold-start. El match más fuerte al caso WiFi.
- **[HECHO/ausencia]:** no hay primaria peer-reviewed de coordinación de airtime específica de **LibreMesh** (opera a nivel
  routing/auto-config, CSMA estándar) → primitivas transferibles = Z-MAC + self-stabilizing-sin-referencia.

> Banderas pre-cita: persistencia 90/10/60% de guifi.net (verificar Maccari&LoCigno 2015); dB de P.833 son a 38–42 GHz (la
> estructura temporal transfiere, los dB no); timescales de ducting (secundarias; primario P.452); wording i.i.d./ergódico de Neely-Modiano-Rohrs.
