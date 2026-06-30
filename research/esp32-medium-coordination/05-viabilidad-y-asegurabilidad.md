# 05 — Viabilidad y asegurabilidad (de-risking antes de comprometer con ARDC)

> La pregunta de Fede: *"¿hasta dónde podemos asegurar de antemano que esto funciona?"*
> Esto es un **dictamen escéptico**, no un brochure. Evidencia en `research/viabilidad-esp32-tdma.md`,
> `research/causalidad-cierre-firmware.md`, `research/regulacion-fcc-firmware.md` y los research previos.

## El dictamen en cinco frases
1. El **nodo oculto es real y TDMA es la solución académicamente aceptada** (hMAC: throughput ≈0 con
   DCF → 8.8 Mbit/s con TDMA híbrido). Ciencia establecida. 🟢
2. **NO existe prior art de un TDMA "duro" sobre el MAC WiFi nativo del ESP32** (con CSMA apagado y slots
   impuestos por hardware). El único antecedente directo es un repo personal sobre ESP-NOW, 2 nodos,
   slots de 10 ms, que **ni siquiera prueba el nodo oculto**. Es un dato de riesgo, no un detalle.
3. **El piso temporal fiable del ESP32 con WiFi activo es ~10 ms con guard time.** Sub-ms es inviable en
   software; los µs son una propiedad del radio **802.15.4**, no del path WiFi del ESP32.
4. **No se puede apagar CSMA/CA ni tocar CWmin/CWmax/backoff por API pública** — está en blob cerrado.
   Confirmado por convergencia de doc oficial + paper de RE + el propio `esp32-open-mac`.
5. **`esp32-open-mac` es real y serio pero TRL ~3-4**, NO tiene en su roadmap el control de contención
   que un TDMA necesitaría, y corre **solo en ESP32 clásico**. Apuesta especulativa para colgar de ahí
   un entregable a 12-18 meses.

## Clasificación de asegurabilidad

### 🟢 ASEGURABLE de antemano (prometer con confianza)
- **Demostrar empíricamente que el nodo oculto colapsa el throughput** en un testbed mesh real (ya medido
  en `hidden-node.md`; respaldo SIGCOMM'05: caída a casi-cero y RTS/CTS lo empeora en multi-hop).
- **Implementar el *mecanismo de turnos* y MEDIR si recupera throughput** (asegurable = el mecanismo + la
  medición rigurosa; el *resultado* "recupera" NO es asegurable de antemano y vive en 🟡). Banco: slots de
  ~10 ms sobre ESP32/ESP-NOW, **topología canónica de nodo oculto (1 Rx, 2 Tx ocultos) con el coordinador
  en el rol de Rx** — el caso donde el control-plane no es circular (ver `03 §C`). Hay piso temporal
  documentado y antecedente (Sync-ESP-NOW guard 45 ms; mayankish 10 ms). **Condición (revisión externa):**
  medir contra **dos** baselines (CSMA puro **y RTS/CTS**), no solo CSMA — sin la curva de RTS/CTS no se
  firma. (En la topología canónica CSMA colapsa a 1.7 Mbps, así que la *probabilidad* de recuperar es
  alta; pero "que recupera" es outcome, no garantía.)
- **Que TDMA es la solución correcta en principio** (hMAC, Soft-TDMAC, Det-WiFi).
- **APuP como mecanismo de peering L2** (ya mergeado en OpenWrt, ago-2024, autoría AlterMundi). Real.

### 🟡 PROBABLE pero requiere validación (no prometer números de antemano)
- **Recuperación cuantificada de throughput bajo nodo oculto con coordinación de ~ms y MÚLTIPLES
  transmisores reales en ESP32** — plausible pero **sin prior art directo**; el jitter (outliers de
  7–20 ms, drift de cientos de ms en rondas largas) obliga a guard times generosos y re-sync frecuente.
- **Algoritmo adaptativo a topología** — factible como software de capa superior, pero su eficacia
  depende de la calidad de sync y del overhead de guard, que solo se conocen tras medir.
- **Scheduling sobre un grafo de conflicto CONOCIDO/medido offline** (topología estática, medida una vez)
  — probable; el coloreo/STDMA es teoría establecida (`07 §6`).
- **Portabilidad del resultado conceptual a routers OpenWrt/LibreMesh** — la *lógica* sí; pero el
  TDMA-sobre-WiFi de producción depende de **silicio específico** (Atheros AR5416+ con frame-kill y
  timers TSF de µs, ICs de Ubiquiti, RouterOS). Un resultado en ESP32 **no transfiere automáticamente**
  a chipsets WiFi arbitrarios de OpenWrt.

### 🔴 NO ASEGURABLE / especulativo (NO prometer como entregable)
- **TDMA de µs sobre el MAC WiFi del ESP32** — sin API, en blob cerrado, sin prior art. Piso = ~10 ms.
- **Apagar/controlar CSMA-CA, CWmin/CWmax o backoff en ESP32** — sin API; fuera del roadmap de
  `esp32-open-mac`; posiblemente imposible sin RE de silicio.
- **Colgar el entregable de que `esp32-open-mac` madure a tiempo** — TRL bajo, fuera de su roadmap, solo
  ESP32 clásico, bus-factor bajo.
- **TDMA de grano-µs sobre WiFi en C5/C6** — esos chips ni están soportados por el esfuerzo open-MAC.
- **Estimación de un grafo de conflicto DINÁMICO (G(t)) en campo sobre ESP32** (medido, dirigido, variable
  con clima/follaje/LoS) — **el problema duro y el verdadero contenido de investigación** (`07`). El HW
  tiene las primitivas (RSSI/paquete, CSI, noise floor) pero **NO hay prior art** que una medición de grafo
  + TDMA interference-aware en ESP32-class; el costo O(n²) del sondeo en MCU no está cuantificado. **No
  prometer como resultado**; sí como *línea de investigación* con su riesgo declarado (riesgo = novedad).

## Dos correcciones de rumbo que salieron del de-risking

### (A) "APuP + TDMA en ESP32" NO es una pieza integrada — separarlo siempre
APuP corre en `hostapd`/OpenWrt y depende de **tramas de 4 direcciones (WDS-like)**. El ESP32 **no corre
hostapd ni mac80211 y no soporta nativamente 4-address frames**; su APSTA está atado a un solo canal. Por
tanto:
- El ESP32 **valida SOLO la lógica de coordinación TDMA** (plano 2), a escala de ms. No "ejecuta APuP".
- APuP vive en LibreMesh (plano 3). La capacidad "APuP + TDMA" es de **LibreMesh**, no del ESP32.
- **Regla de redacción inobjetable:** nunca escribir "APuP + TDMA en ESP32" como sintagma único. Separar:
  *APuP (logrado, hostapd, §1.2)* vs *algoritmo de coordinación (futuro, ESP32 como banco de la lógica,
  §1.5)*. Decir "validar la lógica del algoritmo", no "validar APuP" ni "demostrar la capacidad de
  LibreMesh". Calificar siempre la granularidad (ms, no µs) y declarar el límite del banco (un radio, un
  canal, control que comparte airtime). Un revisor de ARDC con background en redes detecta el colapso de
  las dos líneas de inmediato.

### (B) Hay una bifurcación de plataforma que conviene tener sobre la mesa: 802.15.4
El de-risking destapó algo útil: el **radio 802.15.4** del ESP32-C6/H2 **sí** expone determinismo de µs
por ESP-IDF (`esp_ieee802154_transmit_at()` = TX en timestamp absoluto, `receive_at()`, ACK por hardware
con timeout en µs, y **TSCH** con slots de ~10 ms y guard de cientos de µs). Es decir: si el objetivo es
**validar la lógica del algoritmo de coordinación con timing real**, 802.15.4 da la temporización que el
WiFi del ESP32 niega — desacoplando "¿funciona el algoritmo?" de "¿puede el WiFi del ESP32 dar slots
finos?". **No es WiFi** (no valida la integración con APuP), pero es un banco honesto para la *lógica*.
Decisión de diseño abierta (ver `04 §3` y `03 §6`).

## Banderas para el informe (relevantes a `ingest/GAPS-AND-FINDINGS.md`)
1. **APuP no resuelve el nodo oculto ni hace coordinación temporal** — es peering L2 ortogonal. Si algún
   borrador insinúa lo contrario, es discrepancia técnica a corregir.
2. **Verificar el texto del grant:** la página pública del grant ("The new Wi-Fi for mesh networks",
   USD 313.970) habla de "improving wireless coordination", no de TDMA/slots/hidden node explícitos. El
   entregable técnico no debería prometer más de lo que el grant declaró. (Confirmar con Nico/Javier.)
3. **Encuadre honesto:** el valor demostrable está en el rango **ms** (no µs), sobre **ESP32
   clásico/ESP-NOW** (no WiFi-MAC abierto ni C6-WiFi), como **prueba de concepto de que la coordinación
   temporal recupera throughput** — no como un MAC TDMA de producción portable a hardware arbitrario.

## Decisión estratégica de plataforma que abrió la revisión externa (para el equipo)
El ESP32 gana en **velocidad de PoC y narrativa** (SoC barato, demo vistosa) pero tiene **cero path a
producción WiFi**. Si el objetivo real es *capacidad de LibreMesh en producción*, el camino más honesto es
**ath9k (802.11n)** — el **último MAC WiFi controlable desde el host**, con precedentes reales de soft-TDMA
(Leffler 2009 sobre Atheros, granularidad 1 µs; hMAC sobre ath9k): es WiFi y **tiene path**, aunque sea
hardware viejo y control grueso. No son excluyentes: **ath9k como banco WiFi load-bearing + ESP32 como demo
accesible** es probablemente la mejor combinación. **Esto cambia qué se le promete a ARDC** y conviene
decidirlo con Nico/Javier/Gioaccino antes de redactar §1.5. Ver `06-revision-externa.md`.

## Respuesta directa a "¿hasta dónde podemos asegurar de antemano?"
Podemos asegurar (y comprometer con ARDC) un entregable de la forma: **"diseño de referencia de un
algoritmo de coordinación de acceso al medio + prueba de concepto en ESP32 que demuestra, a granularidad
de milisegundos, que la coordinación de turnos recupera el throughput que el nodo oculto destruye, con
metodología y testbed reproducibles"** — más el **argumento estratégico de de-riesgo de hardware abierto**.
**No** podemos asegurar TDMA de µs, control de CSMA, ni portabilidad directa a producción: eso es la
*línea de investigación abierta*, no una promesa. La novedad (no hay prior art) es a la vez el **riesgo**
y la **contribución**: hay que declararla como tal, no esconderla.
