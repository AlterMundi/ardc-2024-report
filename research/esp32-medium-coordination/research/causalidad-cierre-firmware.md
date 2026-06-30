# Research — ¿Hasta dónde se sostiene "802.11ac cerró la MAC"? (hallazgo #1, profundizado)

> Evidencia para afinar la afirmación causal del informe. Investigación 2026-06-19.
> Distingue **hecho citado** de **inferencia**. Fuentes primarias priorizadas (LWN, kernel.org, specs).

## Resumen: la forma empírica es sólida; la causal fuerte tiene una grieta real
- ✅ **Empírico (ALTA):** ath9k/802.11n fue el último diseño con la MAC controlable desde el host;
  ath10k/802.11ac (2013) en adelante requiere un blob de firmware cerrado que se queda con la
  planificación del medio.
- ⚠️ **Causal fuerte imprecisa:** "802.11ac empujó la MAC de **bajo nivel** al firmware" es técnicamente
  inexacto — **el lower MAC timing-crítico (ACK, backoff, IFS) SIEMPRE estuvo en hardware, incluso en
  ath9k.** Lo que ath10k movió a firmware fue el **upper MAC + el planificador TX**, no el timing duro
  (que nunca estuvo en el host).

## Hechos citados (fuentes primarias)
- **Definición SoftMAC/FullMAC** — Linux Wireless glossary (kernel.org): SoftMAC = MLME en software del
  host (vía `mac80211`); FullMAC = MLME en HW/firmware.
- **El punto de quiebre, cita clave** — Kalle Valo (mantenedor ath10k, Qualcomm Atheros), serie de parches
  de introducción del driver, linux-wireless 2013-05-15 (LWN 550686): *"A major difference from ath9k is
  that there's now a firmware and that's why we had to implement a new driver."* (Da el CUÁNDO=2013 y el
  PORQUÉ de alto nivel = apareció un firmware. **No** dice "por el timing de 11ac" — esa conexión es tuya.)
- **Arquitectura ath10k** — kernel.org: host = glue; control por **WMI**, datos por **HTT**; el firmware es
  dueño del planificador TX. **No es FullMAC clásico** (sigue usando mac80211 vía `ath10k_ops`); la
  caracterización precisa es **"firmware-offload sobre mac80211"**.
- **Adrian Chadd** (dev driver `ath`, FreeBSD), blog 2020-07: marca la generación 11ac como la línea
  divisoria entre chips "software-driven" (pre-11ac) y "firmware based".
- **El lower MAC siempre estuvo en HW** — Wikipedia "SoftMAC" + descriptores ath9k (`ATH9K_TXDESC_NOACK`,
  registro `IGNORE_BACKOFF`): ACK/backoff los ejecutaba el HW MAC de ath9k, configurado por descriptor.
- **TDMA SÍ se hizo sobre Atheros "casi enteramente en hardware"** — Sam Leffler, "TDMA for Long Distance
  Wireless Networks", 2009: usa el temporizador de beacon (1 TU) y duración de ráfaga con **granularidad
  1 µs**. + hMAC (arXiv:1611.05376) sobre ath9k + MikroTik Nv2 sobre Atheros. (Refuerza "ath9k = último con
  control fino del medio".)
- **El cierre es comercial/regulatorio, NO necesidad técnica** — contraejemplos: ath9k PCIe sin blob (FSF
  2008), ath9k_htc USB con **firmware abierto** (github.com/qca/open-ath9k-htc-firmware), **OpenFWWF**
  (firmware MAC 802.11 open sobre Broadcom, LWN 314313). El timing justifica que la MAC esté en HW/firmware;
  NO justifica que sea **cerrada**.

## Corrección de cifras de timing (importante)
| Parámetro | 2.4 GHz | 5 GHz |
|---|---|---|
| **SIFS** | 10 µs | **16 µs** |
| Slot time | 20 µs | 9 µs |
Mi "~9-10 µs" previo confundía el **slot time de 5 GHz (9 µs)** con el **SIFS de 2.4 GHz (10 µs)**. El
deadline duro del ACK inmediato = **un SIFS** (16 µs en 5 GHz). 802.11ax OFDMA uplink: las estaciones deben
arrancar TX a un SIFS **±400 ns** del trigger frame (Rohde & Schwarz) — tolerancia sub-µs inalcanzable por
software de propósito general (justifica HW/firmware, no cierre).

## Cómo frasear en el informe — graduado por evidencia
**A — la más fuerte defendible (RECOMENDADA):**
> "La generación que llegó con 802.11ac (ath10k, 2013+) adoptó un modelo de *firmware-offload*: el
> planificador de transmisión y la agregación pasaron a un firmware on-chip cerrado, y el host dejó de
> exponer el control fino del medio que la generación anterior (ath9k, 802.11n) sí ofrecía —temporizadores
> de beacon y duración de ráfaga con granularidad de µs sobre los que se construyeron TDMA reales (Leffler
> 2009, hMAC, Nv2). Esto hace impráctica la coordinación TDMA de grano fino orquestada desde el host en los
> chipsets comerciales modernos."

**B — intermedia (causalidad como hipótesis explícita):** los deadlines de 11ac/ax (SIFS 16 µs, sounding
MU-MIMO, OFDMA ±400 ns) hacen que la planificación deba resolverse en el dispositivo; los fabricantes la
pusieron en firmware cerrado. (Marcar la cadena causal como interpretación.)

**C — la más cauta (para revisión adversarial):** solo la forma empírica (ath9k = último controlable desde
el host; desde ath10k blob obligatorio), "coincidió con" la llegada de agregación/MU-MIMO/OFDMA.

**NO escribir:** ❌ "11ac empujó la MAC de bajo nivel del host al firmware" (el lower MAC nunca estuvo en el
host). ❌ "es imposible TDMA en chips modernos" (las fuentes solo sostienen "no expuesto / muy difícil").
❌ "el cierre fue por necesidad técnica" (refutado por ath9k/ath9k_htc/OpenFWWF).
