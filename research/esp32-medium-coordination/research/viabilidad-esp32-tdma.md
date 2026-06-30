# Research — Viabilidad de TDMA sobre ESP32 (prior art, piso temporal, esp32-open-mac)

> Evidencia para `05-viabilidad-y-asegurabilidad.md`. Investigación 2026-06-19.
> Tipo: **OFICIAL** (Espressif/paper) · **REPO** · **FORO** (marcado). Ausencia de evidencia = hallazgo.

## 1. Prior art — ¿TDMA ranurado sobre ESP32? **NO sobre el MAC WiFi nativo**
| Proyecto | Radio | Slot | Resultado | ¿Nodo oculto? | Tipo |
|---|---|---|---|---|---|
| mayankish/5G-NR-Inspired-Custom-MAC | ESP-NOW | **10 ms** + guard 3 ms (jitter FreeRTOS σ≈420 µs) | PDR 91.4% vs ALOHA 1.5%; throughput ~14 kbps; p99 21.6 ms | **NO** (2 nodos) | repo, sin peer-review, may-2026 |
| Sync-ESP-NOW (Nazarbayev U.+DTU) | ESP-NOW | broadcast sync, **NO TDMA** (guard 45 ms) | 96-97% ahorro energía; PRR cae a -92 dBm | NO (1 Tx) | peer-review, BlackSeaCom 2023 |
| OpenWiFiSync/RBIS (DFKI/RPTU) | WiFi (beacons) | sync de **reloj** ±30 µs (σ≈5.35 µs), no MAC | — | NO | arXiv 2511.14457, nov-2025 |
| LoRaMesher | **LoRa** (no WiFi) | 200-550 ms | TDMA real, sleep fuera de slot | — | repo |
| Det-WiFi / RT-WiFi | 802.11 PHY | superframe+slots | supera 802.11s industrial | parcial | peer-review 2013/2017 — **HW Atheros COTS, NO ESP32** |

**Lectura:** lo que existe es (a) app-layer sobre ESP-NOW de 2 nodos, 10 ms, sin nodo oculto; (b) TDMA real
pero sobre radios que NO son WiFi (LoRa, 802.15.4); (c) TDMA sobre 802.11 pero en hardware Atheros que sí
deja parchear el MAC. **"TDMA sobre WiFi/ESP-NOW resolviendo nodo oculto con múltiples Tx y slots finos"
no tiene precedente publicado.** Declarar a ARDC como zona sin red de prior art (riesgo + novedad).

## 2. Piso temporal del ESP32 con WiFi activo — **~10 ms con guard**
Cotas DURAS (doc oficial Espressif, ALTA):
- WiFi task: prioridad 23, **pinneada a Core 0 (PRO_CPU)**, no removible.
- **Interrupt WDT**: bloqueo de IRQ/tick >300 ms (800 ms con SPIRAM) → reset.
- **Flash write/erase congela ambos cores + IRQs no-IRAM ~80 ms/4 KB**; WiFi lo dispara vía NVS/reconexión.
  Es el bloqueo largo real y medido.
- WiFi scan bloquea 120 ms activo / 360 ms pasivo por canal. **No hay cota superior garantizada** de
  latencia de interrupción.

Jitter de timers (oficial + medición, ALTA):
- `esp_timer` contador 1 µs, dispatch práctico ~20-50 µs; jitter por scheduling + cache-miss + co-residencia
  con WiFi en Core 0 (no por "el WiFi interrumpiendo"). Mitigable con `IRAM_ATTR` + registros directos.
- NTP real-world (blog medido): mediana 216 µs, **p99 1607 µs, p99.9 2461 µs**.
- GPTimer+IRAM ~50 µs de jitter incluso transmitiendo (FORO esp32.com, MEDIA).
- Paper TU Berlin/HPI (arXiv 2102.05559, 2021): el driver WiFi **crashea a 980 pps**; *"transmitting command
  and control over an IP network is not feasible for tasks that need hard real-time guarantees"*.

| Slot | ¿Fiable con WiFi on? |
|---|---|
| sub-ms | **NO** en software (solo GPTimer HW+IRAM ~50 µs, frágil ante flash) |
| 2 ms | riesgoso (outliers 7 ms + apilamiento UDP 10-20 ms) |
| 5 ms | marginal (bajo el worst-case de 7 ms; necesita guard) |
| **10 ms** | **frontera viable** con guard generoso |
| 35-45 ms | sí (lo que usa Sync-ESP-NOW) |

`esp_wifi_80211_tx()` **NO es bypass del MAC**: la trama pasa por cola TX, CCA y backoff del driver; el
usuario no controla el instante de salida. En bucle apretado deja de transmitir hasta reboot (#6369).

## 3. esp32-open-mac — TRL ~3-4, sin control de contención, solo clásico
- **Hoy logra** (README/blogs, ALTA): TX/RX raw 802.11 sin blob tras init, ACK, cambio canal/rate/potencia,
  filtrado MAC por HW, asociación a AP abierto + UDP/lwIP, WPA2 (2025), AP básico, AWDL experimental.
  Mesh 802.11s: solo asocia, "don't handle any data yet". Hackaday (dic-2024): "proof of concept".
- **Lo que mata el caso TDMA:** code-search en toda la org (`cwmin`, `backoff`, `contention`, `SIFS`,
  `tdma`, `slot_time`) → **cero coincidencias**. **No existe ni está planeado** el control de
  CWmin/CWmax/backoff. Exponerlo requeriría RE adicional no comenzado, posiblemente imposible sin registros
  documentados.
- **Timeline/riesgo:** grant NLnet 2023-10→2025-11; repo C original congelado (ene-2025), desarrollo migró a
  Rust (FoA/esp-wifi-hal). Bus-factor bajo. **Solo ESP32 clásico**; NO C3/C5/C6/S3 (periférico WiFi
  hardcodeado/sin documentar en los RISC-V).

## 4. Contraste decisivo: 802.15.4 (ESP32-C6/H2) SÍ da µs — pero no es WiFi
- ESP-IDF expone `esp_ieee802154_transmit_at()` (TX en timestamp absoluto), `receive_at()` (ventana RX),
  **ACK por hardware** (timeout en µs, default 1728 µs). TSCH usa slots ~10 ms, guard de cientos de µs,
  sync a nivel µs. **No hay equivalente público en el path WiFi.** El determinismo de grano-µs es propiedad
  del estándar/radio 802.15.4, no replicable a igual nivel sobre el MAC WiFi cerrado del ESP32.

## Notas de confianza
ALTA: docs Espressif (fetch directo), papers leídos (hMAC TKN-16-0004, Sync-ESP-NOW, OpenWiFiSync,
SIGCOMM'05, arXiv 2102.05559/2501.17684), commits OpenWrt, code-search esp32-open-mac.
MEDIA-BAJA (marcadas): anécdotas esp32.com (números "7000 µs", "50 µs GPTimer", "980 pps" de snippets).
Gap: no se obtuvo cita textual de un ingeniero de Espressif negando control de CSMA; lo más fuerte es la
**ausencia documentada** en la API + el paper de análisis estático del blob.
