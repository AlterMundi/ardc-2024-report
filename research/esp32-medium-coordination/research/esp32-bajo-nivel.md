# Research — Control de bajo nivel del WiFi y temporización en ESP32

> Evidencia citable para el algoritmo de coordinación. Investigación 2026-06-19.
> Convención: **OFICIAL** = doc Espressif / paper revisado · **REPO/FORO** = comunidad (requiere validación).
> Confianza marcada por hallazgo. Ninguna API ni cifra fue inventada.

## 1. Inyección de tramas crudas — `esp_wifi_80211_tx()`

- **1.1 (ALTA/OFICIAL)** La API solo transmite *beacon, probe req/resp, action frame* y *non-QoS data frame*. **No** envía QoS ni cifradas. → docs.espressif.com `.../wifi-driver/wifi-vendor-features.html` (ESP-IDF v6.0.1; idéntico en C6).
- **1.2 (ALTA/OFICIAL)** Sanity-check de Frame Control: STA→AP exige ToDS=1/FromDS=0; AP→STA al revés; PM/More-Data/Retransmission deben ser 0. Si no, `ESP_ERR_INVALID_ARG` (0x102).
- **1.3 (MEDIA/REPO)** Bug histórico #4002 (2019-09-03): en modo AP los data frames devuelven 0x102; management sí pasa.
- **1.4 (MEDIA/REPO+paper)** El símbolo `ieee80211_raw_frame_sanity_check` vive en `libnet80211` (binario cerrado); debilitarlo con objcopy "no evita la restricción en la práctica". → MDPI Sensors 2026 (Deauther32/HackHeld32, ESP32-S3).
- **1.5 (MEDIA/REPO)** Bypass comunitario `Jeija/esp32free80211` llama a `ieee80211_freedom_output` (no documentada) para enviar tramas arbitrarias; tamaño 23<len≤1400. **No** da control de CSMA/CA ni de timing.
- **1.6 (ALTA/OFICIAL)** Tasa TX por defecto 1 Mbps, ajustable `esp_wifi_config_80211_tx_rate()`; ancho `esp_wifi_set_bandwidth()`.
- **1.7 (MEDIA/REPO)** #6369: `esp_wifi_80211_tx()` deja de funcionar si se invoca en loop cerrado → NO es primitivo para TX determinista de alta cadencia.

## 2. Modo promiscuo / sniffer

- **2.1 (ALTA/OFICIAL)** `esp_wifi_set_promiscuous(true)` + callback; entrega `wifi_pkt_rx_ctrl_t` (RSSI, rate, channel). Filtro por tipo.
- **2.2 (ALTA/OFICIAL)** Campo `timestamp` = tiempo local RX en µs, "preciso solo si modem/light sleep deshabilitados".
- **2.3 (MEDIA/REPO)** Bugs: #2468 (mismo timestamp en beacons consecutivos), #1751 (`noise_floor` siempre 0).

## 3. Timing de TX, scheduler MAC, CSMA/CA y partes cerradas

- **3.1 (ALTA/OFICIAL por ausencia + paper)** **No hay API pública para deshabilitar CSMA/CA ni programar TX en instante exacto.** El acceso al medio lo arbitra HW/blob.
- **3.2 (ALTA/paper arXiv:2501.17684, 2025)** RE del driver del **ESP32-C3**: arquitectura FullMAC; MAC y PHY en HW; **CSMA/CA en hardware**; driver binario cerrado. Cifras medidas (C3): ACK wait máx **326 µs**; confirmación HW **800 ns**; WCET tarea TX **329 µs** @160 MHz.
- **3.3 (ALTA estado / REPO+38C3 dic-2024 + NLnet)** `esp32-open-mac`: el **ESP32 clásico es SoftMAC** (la mayor parte de la MAC es software). Hoy: enviar/recibir frames, ACK, cambiar canal, ajustar rate/power, filtrar MAC en HW, **sin código propietario tras la init** (el blob solo inicializa). **Aún NO** expone TDMA, scheduling de TX ni control de CSMA/CA. Solo cubre el clásico (en C3/C6 el periférico WiFi está hardcodeado en otra ubicación).
- **3.5 (ALTA para otras plataformas / papers)** Deshabilitar backoff en COTS se logra poniendo CWmin/CWmax/AIFS=0 (hMAC/FreeMAC sobre Atheros; openwifi en SDR). **No transferible directo al ESP32 cerrado** — es referencia de diseño.

> **Implicación:** la vía realista de un TDMA "de verdad" en ESP32 pasa por `esp32-open-mac`, no por la API pública de Espressif. A jun-2026 esa vía aún NO expone control de slot/backoff (trabajo en progreso). Que la API pública NO lo permita = ALTA; que open-mac lo permitirá = INCIERTO.

## 4. ESP32 clásico vs S2/S3 vs C5/C6 (WiFi 6)

- **4.1 (ALTA/OFICIAL)** La API de bajo nivel y sus restricciones son las mismas en clásico y C6; WiFi 6 NO expone nuevas primitivas MAC de slot al desarrollador.
- **4.2 (ALTA)** C5 = dual-band 2.4/5 GHz WiFi6, RISC-V ≤240 MHz; C6 = solo 2.4 GHz WiFi6. WiFi6 aporta TWT/OFDMA/MU-MIMO como *features de stack*, no como control fino de slots. BW 802.11ax limitado a 20 MHz.
- **4.3 (ALTA por ausencia)** El RE abierto está más avanzado en **clásico/C3, NO en C5/C6**. Para 5 GHz + control de bajo nivel hoy **no hay base abierta**.

## 5. Temporización y sincronización

- **5.1 (ALTA/OFICIAL)** `esp_timer`: resolución 1 µs, pero timeouts mínimos prácticos ~20 µs (one-shot) / ~50 µs (periódico) @240 MHz. Para waveform usar GPTimer/RMT.
- **5.2 (ALTA resolución / OFICIAL; jitter no especificado)** GPTimer: resolución configurable (típico 1 µs); la doc no garantiza jitter.
- **5.3 (MEDIA/FORO)** Jitter ISR peor caso ~48 µs, típico 2–9 µs. **Crítico: con WiFi activo, tareas pueden bloquearse "un par de segundos"** porque WiFi corre en core 0 (PRO_CPU). Mitigación: ISR en core 1, RMT.
- **5.4 (ALTA ausencia PTP-WiFi / MEDIA ESP1588)** No hay PTP de primera clase para WiFi en ESP-IDF (issue abierto #13423). PTP HW sub-µs aplica al MAC Ethernet, no al WiFi.
- **5.5 (MEDIA inferida)** Sync por radio (timestamp RX, §2.2) realista a **decenas de µs**; sub-10 µs NO verificado en WiFi ESP32.

## 6. Por qué "ESP32 no sirve para TDMA WiFi de verdad" — escéptico

(a) CSMA/CA en HW, no deshabilitable por API pública [ALTA]. (b) Driver MAC cerrado (libnet80211/libpp) [ALTA]. (c) WiFi en core 0 → bloqueos de segundos + jitter de decenas de µs [MEDIA]. (d) `esp_wifi_80211_tx` no es para TX determinista [MEDIA]. (e) ESP-NOW como cota inferior real: 1–10 ms RTT, 1 Mbps fijo [ALTA doc].

**Matiz honesto:** TDMA *aproximado* (slots de **ms**, sync por beacon/RX-timestamp) SÍ es factible. Lo inviable hoy es TDMA *de µs con CSMA apagado* vía API pública. Las cifras de timing del paper son del **C3**, no extrapolables sin medir al clásico/C5/C6.

## Fuentes primarias
- ESP-IDF Vendor Features / `esp_timer` / GPTimer / ESP-NOW — docs.espressif.com (OFICIAL)
- arXiv:2501.17684 (RE driver C3, CSMA/CA en HW), 2025 — PAPER
- esp32-open-mac.be + GitHub + charla 38C3 (dic-2024) + NLnet — PROYECTO ABIERTO (en progreso)
- Jeija/esp32free80211 — REPO · MDPI Sensors 2026 — PAPER · issues esp-idf #4002/#6369/#2468/#1751/#13423
