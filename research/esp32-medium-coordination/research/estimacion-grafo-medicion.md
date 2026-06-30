# Research — Estimación del grafo de conflicto: bibliografía de MEDICIÓN

> Para `07`/`09`. Cada cita: autor, venue/año, URL, "cómo encamina". **[HECHO]** = verificado / **[INFER]** = juicio.
> Correcciones de atribución verificadas: el "General Model of Wireless Interference" (MobiCom'07) es de
> **Qiu/Zhang/Wang/Han/Mahajan** (no Maheshwari); "Learning the Interference Graph" es **Yang/Draper/Nowak**
> (pasivo); **CMAP** = Vutukuru/Jamieson/Balakrishnan NSDI'08.

## 1. Sondeo activo (O(N) experimentos)
- **Reis et al., SIGCOMM 2006** — *Measurement-Based Models of Delivery and Interference*. Siembra el modelo con
  **N trials → O(N²) parámetros**; reduce el error RMS a la mitad (11a)/un tercio (11b). DOI 10.1145/1159913.1159921.
  http://research.cs.washington.edu/networking/wireless/bits/sigcomm2006-interference.pdf · **[HECHO/alta]**
  → *cita ancla*: el barrido inicial del grafo cuesta O(N) rondas, no O(N²) pares.
- **Niculescu, IMC 2007** — *Interference Map for 802.11*. La interferencia es **lineal** en la tasa de paquetes →
  extrapolar a **O(n)** desde mediciones de pares; requiere *downtime*. http://conferences.sigcomm.org/imc/2007/papers/imc30.pdf
  · **[HECHO/alta]** → medir pares simples y extrapolar la agregada; sondear en ventanas ociosas.
- **Padhye et al., IMC 2005** — *Estimation of Link Interference…*. El método **pairwise** O(N²) que los demás
  optimizan. https://www.usenix.org/legacy/event/imc05/tech/full_papers/padhye/padhye.pdf · **[HECHO/alta]**
- **Qiu et al., MobiCom 2007** — *A General Model of Wireless Interference*. Modelo medido > propagación por
  distancia. DOI 10.1145/1287853.1287874 · **[HECHO/alta]** → calibrar con medidas reales, no teoría.

## 2. Inferencia pasiva (costo activo ~cero)
- **Vutukuru/Jamieson/Balakrishnan (CMAP), NSDI 2008** — mapa de conflicto desde pérdidas durante tráfico normal;
  **+47% throughput**, sin downtime. https://www.usenix.org/legacy/event/nsdi08/tech/full_papers/vutukuru/vutukuru.pdf
  · **[HECHO/alta]** → plantilla del modo pasivo: aristas desde la estadística de pérdida que el ESP32 ya genera.
- **Shrivastava et al. (PIE), NSDI 2011** — estimación pasiva *online* de interferencia. **[HECHO/alta]** (venue 2011).
- **Abdelwedoud/Busson et al., AdHoc-Now 2019** — grafo de conflicto **ponderado** por *busy-time*/CCA, sin tráfico.
  https://inria.hal.science/hal-02404943/document · **[HECHO/alta]** → usar contadores CCA del ESP32 para ponderar aristas.
- **Rossi/Casetti/Chiasserini, AdHoc-Now 2014** — interferencia desde **PER + airtime ocupado**, sin HW especial.
  · **[HECHO/alta]** → PER+airtime bastan en hardware commodity.
- **Yang/Draper/Nowak, IEEE TSIPN 2017** — *Learning the Interference Graph*. Observación necesaria escala
  **d²·log n**. arXiv:1208.0562 · **[HECHO/alta, teórico]** → ley de escala para dimensionar el tiempo de observación.
- **Rossi et al., GNNet'22 (CoNEXT ws)** — GCN para estimar interferencia de airtime, transferible entre redes.
  arXiv:2211.14026 · **[HECHO/media]** → ruta futura (features pasivas → predicción, costo offline).

## 3. RSSI vs CSI
- **Aguayo et al. (Roofnet), SIGCOMM 2004** — *"SNR and distance have little predictive value for loss rate"*;
  pérdidas intermedias = multipath. http://conferences.sigcomm.org/sigcomm/2004/papers/p442-aguayo1111.pdf
  · **[HECHO/alta, verbatim]** → **no** construir el grafo solo con umbral de RSSI.
- **Halperin et al., SIGCOMM 2010** — CSI → *effective SNR* predice entrega (capta fading selectivo en frecuencia).
  http://ccr.sigcomm.org/online/files/p159_0.pdf · **[HECHO/alta]** → usar la CSI del ESP32 vía effective SNR.
- **Halperin et al., CCR 2011** — *Tool Release: CSI*. La CSI da info por subportadora que el RSSI escalar no. **[HECHO/alta]**
- **Hernandez & Bulut, WoWMoM 2020** — **ESP32-CSI-Tool** (Active/Passive CSI sobre ESP-IDF). 
  https://stevenmhernandez.github.io/ESP32-CSI-Tool/ · **[HECHO/alta]** → base de SW lista (ojo: es sensing de personas).
- **ESP-IDF (CSI)** — ~**64 subportadoras** en HT20, RSSI/noise-floor en `rx_ctrl`, trampa `first_word_invalid`.
  github.com/espressif/esp-csi · **[HECHO/alta]** → confirma los observables disponibles.

## 4. Modelo de protocolo vs SINR (complejidad reducida)
- **Gupta & Kumar, IEEE TIT 2000** — Protocol Model vs Physical (SINR aditivo). DOI 10.1109/18.825799 · **[HECHO/alta]**
- **Jain/Padhye/Padmanabhan/Qiu, MobiCom 2003** — define el **conflict graph** + cotas LP. 
  https://www.cs.utexas.edu/~lili/papers/pub/mobicom2003.pdf · **[HECHO/alta]** (el objeto formal central).
- **Goussevskaia/Oswald/Wattenhofer, MobiHoc 2007** — scheduling bajo SINR geométrico es **NP-hard**. **[HECHO/alta]**
  → justifica NO implementar SINR exacto en ESP32.
- **Maheshwari/Jain/Das, SenSys 2008** — en **motes TelosB** valida PRR–SINR; los modelos simples (hop/range/protocol)
  son **notablemente inexactos** frente al SINR aditivo. DOI 10.1145/1460412.1460427 · **[HECHO/alta]** → la cita más
  pertinente a MCU: argumenta por un *physical-lite* con SINR aproximado.
- **Shi/Hou/Kompella/Sherali, MobiHoc 2009** — *reality check*: soluciones bajo protocolo pueden ser **inviables**;
  ajustar el rango de interferencia estrecha la brecha. https://kevinliu-osu.github.io/publications/hoc_shi.pdf · **[HECHO/alta]**
- **Shokri-Ghadikolaei/Fischione/Modiano, ICC 2016** — con omnidireccional hace falta SINR; con direccional basta
  protocolo. arXiv:1602.01218 · **[HECHO/media]** → elegir el modelo según el despliegue real.

## 5. Construcción incremental / sondeo dirigido
- **Ahmed/Ismail/Keshav/Papagiannaki, CoNEXT 2008** — *micro-probing*: construye el grafo **online** (con la red en
  uso). DOI 10.1145/1544012.1544016 · **[HECHO/alta]** → cita central del modo online.
- **Zheng & Tajer, SPAWC 2020** — *Learning… via Local Probes*: datos locales mínimos (binario) en vez de todos los
  pares. https://par.nsf.gov/servlets/purl/10209389 · **[HECHO/alta]** → cada ESP32 reporta señal binaria local.
- **Zhou et al. (Practical Conflict Graphs in the Wild), TON 2014 / SIGMETRICS 2013** — mantiene el grafo sin medición
  exhaustiva (propagación calibrada + interpolación). https://gangw.cs.illinois.edu/conflict-sigmetrics13.pdf · **[HECHO/alta]**
- **[INFER/riesgo]:** el *active learning que sondea solo aristas inciertas* no es un paper canónico único → posicionarlo
  como **contribución propia** apoyada en Zheng-Tajer + Yang + CCA.

## 6. Prior art en HW barato — VEREDICTO: hueco ESP32 real (novedad incremental)
- **Zhou/Stankovic (RID), INFOCOM 2005** — interference detection en motes MICA2. **[HECHO/alta]** (raíz del linaje).
- **Liu/Xing (PIM), ICNP 2010** — interferencia pasiva en motes 802.15.4. **[HECHO/alta]**.
- **Jia/Wei/Pi et al., EWSN 2023** — *Interference Graph Estimation via Concurrent Flooding* en **nRF52840 (BLE/802.15.4)**
  — el competidor conceptual más cercano. arXiv:2312.16807 + extensión TOSN 2024 (arXiv:2408.11499). **[HECHO/alta]**.
- **Falsos positivos descartados:** esp-csi / "Less is More" = *WiFi sensing de personas* (usa TDMA para EVITAR
  interferencia, no mapearla); ESP-WIFI-MESH/painlessMesh = sin medición de interferencia. **[HECHO/alta]**.
- **VEREDICTO [INFER/alta]:** **no hay prior art de estimación de grafo de interferencia en ESP32/WiFi.** Existe en
  motes 802.15.4 (RID/PIM) y BLE/nRF52 (Jia). Posicionar como *primer mapeo de grafo en ESP32/WiFi de bajo costo*,
  citando RID/PIM/Jia y argumentando diferencias de plataforma/radio/método. **Riesgo = novedad.**

> Re-verificar metadatos (confianza media): orden de autores Shi-Hou (MobiHoc'09), DOI Goussevskaia, autores Chafekar
> (INFOCOM'08), abstract literal de Ahmed (CoNEXT'08, ACM DL 403).
