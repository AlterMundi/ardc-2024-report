# Research — Plataforma WiFi abierta de alto rendimiento (¿dónde tiene impacto el algoritmo?)

> Para `10`. Investigación 2026-06-19/20, foco 2023-2026. Cita: afirmación / fuente URL+fecha / confianza / implicación.
> **TL;DR:** la única plataforma WiFi abierta con MAC de bajo nivel reprogramable y timing µs **hoy** es **openwifi**
> (a/g/n, 20 MHz, ~40-50 Mbps). El alto rendimiento (WiFi 6/7) abierto **no está bloqueado por el silicio (RFSoC lo
> resuelve) sino por la ausencia de un PHY 802.11ax/be abierto** — es un problema de PHY-IP, no de comprar hardware.

## 1. OpenWiFi (open-sdr / Xianjun Jiao / IDLab-imec, UGent) — estado 2025-2026
- **Estándares:** 802.11a/g completo + 802.11n parcial (SISO, 20 MHz, MCS 0-7, ~72 Mbps PHY). **MIMO/40 MHz/agregación
  NO** en el core abierto; **802.11ax NO está en el master abierto** (roadmap/vía comercial). [README](https://github.com/open-sdr/openwifi)
  · **[ALTA]**
- **Throughput medido:** iperf **TCP 40-50 / UDP ~50 Mbps** (build SISO 20 MHz). **[ALTA]** → el impacto del scheduler se
  mide en **latencia/jitter/determinismo y recuperación bajo nodo oculto**, no en throughput pico.
- **HALLAZGO CLAVE — low-MAC abierta + time-slicing de µs:** README literal: *"DCF (CSMA/CA) low MAC layer in FPGA
  (**10us SIFS is achieved**)"* y *"**Time slicing based on MAC address (time gated/scheduled FPGA queues)**"*. La extensión
  **W-TSN** añade gating de colas, time-triggered, distribución de schedule OTA, **timestamping HW** y **sync de precisión µs**
  (timeslots demostrados hasta 128 µs). SIFS/DIFS/slot/CW/CCA configurables. [IDLab openwifi-TSN] · **[ALTA]**
  → **un TDMA de µs sobre grafo de conflicto es implementable HOY** aquí.
- **Prior art que ya construyó MAC TDMA encima:** **RT-WiFi on SDR** (Zelin Yun et al., **IEEE RTAS 2022**); **TDMA/CSMA
  híbrido para robótica** (arXiv:2509.06119, sep-2025, añade PTP+timestamping a la MAC Verilog); **OFDMA coordinado fine-grained
  vía openwifi + White Rabbit** (arXiv:2507.10210, jul-2025). **[ALTA]** → no es especulativo: la plataforma ya soportó MACs TDMA con resultados de determinismo.
- **Hardware/costo:** Zynq-7000/UltraScale+ + AD9361/64; entrada barata **AntSDR E310 ~$246-400**; ZCU102 = miles. Testbed
  2-3 nodos ~**$750-1200**. **[ALTA placas / MEDIA precios]**
- **Licencia:** doble — **AGPLv3** (core) + comercial (ax/TSN avanzado vía openwifi.tech, ligado a IDLab/imec). **[ALTA]**
- **Actividad:** **VIVO** — release **v1.5.0 (2025-08-06)**, commits a **2026-05-21**; openwifi-hw a 2025-09-23. v1.5.0
  **financiado por NLnet**. [GitHub API; NLnet OpenWifi-80211n] · **[ALTA]**

## 2. El muro de ancho de banda para WiFi 6/7 (160/320 MHz)
| Chip | BW inst. máx | Nota |
|---|---|---|
| **AD9361/64** (USRP B210, AntSDR, Pluto) | **56 MHz** | el SDR barato topa acá — ni 80 MHz |
| ADRV9002 | 40 MHz | más nuevo pero más angosto |
| LMS7002M (LimeSDR) | ~120 MHz analóg. / **~60 práctico** (USB3) | mismo muro |
| **ADRV9009** | **200 MHz** RX | cubre WiFi6 (160), no WiFi7 (320); ~$290/chip + FPGA JESD204B → placa miles |
| **Zynq RFSoC** (sampleo directo) | **~1.6 GHz** | resuelve el BW; miles USD |
- **Muro de ~56 MHz del AD9361 CONFIRMADO** (datasheet ADI + Ettus B210). **[ALTA]** → un SDR WiFi 6/7 *abierto y barato* de
  canal único **no es alcanzable con chips clase AD9361**. *Bandwidth stitching* (KrakenSDR/RTL coherente) probado para RX/monitoreo,
  **no resuelto para decode+TX coherente de WiFi 6/7 en open source**. **[MEDIA]**

## 3. RFSoC como habilitante — el cuello NO es el silicio
- **Gen3 Zynq UltraScale+ RFSoC:** ADC 14-bit **5 GSps** / DAC **10 GSps**; Nyquist ~2.5 GHz **excede ampliamente** 160/320 MHz
  por **sampleo directo** sin transceiver externo. [AMD UG1287] · **[ALTA]** (caveat: banda 6 GHz de WiFi7 puede requerir
  front-end/zona Nyquist cuidadosa, **[MEDIA]**). → **el BW/conversores es problema resuelto.**
- **Costo:** **RFSoC 4x2 ~$2,150** (académica); ZCU111 ~$10.8k; ZCU208 ~$13k. **[ALTA 4x2]**
- **¿Existe PHY WiFi 6/7 abierto en 2026? NO (resultado negativo honesto):** openwifi AGPLv3 = solo a/g/n (ax es de pago);
  WARP/Mango topa en 40 MHz legacy; SWiFi es análisis no PHY; **nada de 802.11be abierto.** OAI/srsRAN = solo celular.
  [openwifi.tech; WARP; SWiFi] · **[ALTA]** → **la brecha central: falta el baseband HE/EHT abierto, no el hardware.**
- Investigación 802.11ax en FPGA activa pero **PoC 20 MHz, no liberada:** Aslam et al. (Computer Communications 2024, TX MU-OFDMA,
  >87% ahorro recursos); Havinga et al. (INFOCOM 2024, EuCNC 2025, linaje openwifi). **[ALTA]** → el núcleo más creíble para crecer un PHY WiFi 6 abierto es **imec/Ghent/openwifi**.

## 4. Otros caminos
- **mt76/MediaTek (el vendor más abierto):** driver open (BSD/GPL) **pero la MAC vive en blob `*_wm.bin`** ("WiFi MAC" en el
  CONNAC on-chip); TWT/MU-MIMO/OFDMA son firmware-side. [openwrt/mt76; kernel.org] · **[ALTA]** → podés *configurar* params TWT
  (scheduler **grueso**), **no reprogramar el slot-timing**. **No hay TDMA custom de µs en mt76.**
- **OpenWrt One (Filogic MT7981B, WiFi 6, 2024-11):** router más documentado/abierto que podés comprar (datasheet abierto, FCC,
  copyleft) **pero misma MAC-blob mt76**. [SFConservancy; The Register] · **[ALTA]** → bueno para overlay TDMA host-side, inútil para slot de bajo nivel.
- **Open silicon / RISC-V:** **no existe silicio WiFi 6/7 abierto.** El RISC-V de openwifi es intención futura, no entregada. RF
  analógico abierto en alfa (IHP SG13G2, "no para producción"); ecosistema digital frágil (**Efabless cerró marzo-2025**). **[ALTA]**
- **Histórico:** **ath9k** (Atheros, ~2008, SoftMAC nivel driver, PCIe sin blob; AR9271 USB con `open-ath9k-htc-firmware`) y
  **OpenFWWF** (Broadcom b43, 802.11b/g) fueron las únicas MAC abiertas; **obsoletas en rendimiento**. La antorcha pasó a openwifi.
  **Nada moderno/de alta tasa reemplazó la MAC abierta de ath9k** — el argumento de base del informe. **[ALTA]**

## 5. GNU Radio / gr-ieee802-11 (Bastian Bloessl) — herramienta, no plataforma de impacto
- PHY OFDM **802.11a/g/p** totalmente software sobre USRP; todos los MCS, 20 MHz; mantenido (3.10). [github.com/bastibl/gr-ieee802-11] · **[ALTA]**
- **ARGUMENTO CLAVE (palabras del autor, README):** *"cannot respond to ACKs in time"* por *"architectural limitation of USRP +
  GNU Radio since Ethernet and computations on a normal CPU introduce some latency"* → *"there is no CSMA/CA mechanism"*,
  *"RTS/CTS is not working for the same reason"*. **[MUY ALTA]** Corroborado: round-trip host↔USRP **cientos de µs a ms** vs SIFS
  **10-16 µs** → déficit de 1-2 órdenes. (NxWLAN arXiv:1607.03254). → **una MAC TDMA de µs NO es implementable en PHY software.**
- **Para qué SÍ sirve:** research PHY, **CSI/channel sounding**, sniffing/decode, **prototipar la *lógica*** del grafo de conflicto,
  docencia. Los propios autores acoplan SDR + simulación (OMNeT++/Veins) para evaluar MAC *porque el SDR vivo no impone el timing MAC*. **[ALTA]**
- SIFS/slot exactos: 802.11a = **SIFS 16 / slot 9 µs**; 802.11g = **SIFS 10 / slot 9 µs**; el ACK va **exactamente un SIFS** tras el frame. **[ALTA]**

## Síntesis (tabla)
| Camino | ¿MAC abierta? | Gen | Throughput | Control slot µs | Costo/nodo |
|---|---|---|---|---|---|
| **openwifi (SDR/FPGA)** | ✅ Verilog, SIFS 10µs, time-slicing | a/g/n 20MHz SISO | ~40-50 Mbps | **Total** | ~$250-400 |
| mt76/OpenWrt One | ❌ blob `*_wm.bin` | WiFi 6/7 | alto | solo TWT estándar | ~$90-100 |
| RFSoC + PHY ax abierto | (PHY no existe) | WiFi6/7 (HW listo) | — | — falta baseband | ~$2,150 |
| gr-ieee802-11 | PHY sí, MAC RT no | a/g/p | bajo | ❌ no alcanza SIFS | USRP ~$2k |
| ath9k (histórico) | ⚠️ SoftMAC driver | n | modesto | TDMA nivel driver | commodity |

**Conclusiones:** (1) para impacto medible HOY de un TDMA de µs, **openwifi es la respuesta** (limitación: ~50 Mbps, impacto en
latencia/determinismo/recuperación bajo nodo oculto). (2) El alto rendimiento abierto **no está bloqueado por el silicio
(RFSoC sobra) sino por el PHY 802.11ax/be abierto inexistente** → el camino es **financiar/desarrollar un baseband HE abierto y
portarlo a RFSoC, extendiendo el linaje openwifi** — esfuerzo de PHY-IP plurianual, no compra de hardware. (3) MediaTek abre el
driver, no la MAC. (4) GNU Radio prototipa la lógica, no ejecuta el TDMA en tiempo real.
