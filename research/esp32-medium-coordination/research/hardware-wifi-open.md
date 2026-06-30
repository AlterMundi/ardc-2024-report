# Panorama de hardware WiFi open-source y drivers abiertos (2025-2026)

**Investigación para informe ARDC — AlterMundi / LibreMesh**
Fecha de corte: 2026-06-19. Metodología: búsqueda multi-fuente + verificación adversarial de las afirmaciones de mayor peso. Cada hallazgo lleva: afirmación, fuente (URL), fecha, confianza. Lo incierto/desactualizado va marcado explícitamente.

---

## 0. Veredicto sobre el argumento central

> *"TDMA y coordinación fina del medio en mesh es imposible con chipsets WiFi comerciales modernos (802.11ac/ax+) porque desde 802.11ac la MAC de bajo nivel se delegó a firmware cerrado; el último WiFi con driver 100% open es 802.11n (ath9k)."*

**El argumento se sostiene en su forma EMPÍRICA, con un matiz importante en su forma CAUSAL.**

- **Forma empírica (sólida, alta confianza):** ath9k (802.11n) es el último WiFi cuyo stack abierto llega hasta el firmware/MAC (sin blob en PCIe, o con firmware liberado como software libre en USB). Desde 802.11ac en adelante, **todos** los drivers Linux mainstream (ath10k/11k/12k, mt76, brcmfmac, iwlwifi, rtw88/89) son abiertos como *driver* pero dependen de un **blob de firmware propietario** que ejecuta la MAC de bajo nivel y el timing crítico. No tenés acceso a esa capa.
- **Forma causal (razonable pero NO documentada literalmente — confianza media):** ninguna fuente primaria (docs del kernel, IEEE) afirma textualmente "802.11ac empujó la MAC de bajo nivel a firmware cerrado". Es una **interpretación de ingeniería bien fundada**: la eficiencia de 802.11n/ac depende de agregación A-MPDU + Block-ACK en secuencias acotadas por SIFS (~10 µs), y MU-MIMO/OFDMA en 11ac/ax exigen scheduling de tiempo aún más fino — todo lo cual incentiva mover el timing fuera del host. Pero la literatura establece la *necesidad de timing ajustado*, no afirma que *por eso* los vendors cerraron el firmware. **Recomendación para el informe: presentarlo como "el patrón observable es X; la explicación técnica más plausible es Y", no como hecho citado.**

---

## 1. OpenWiFi (open-sdr / Xianjun Jiao, IDLab UGent)

**OpenWiFi es el proyecto open-source más serio que da acceso real a la Low-MAC en FPGA y permite TDMA/control de airtime de verdad — pero está limitado a 802.11a/g/n, 20 MHz y ~50 Mbps, y las funciones más avanzadas (ax, TSN avanzado) están detrás de licencia comercial.**

| # | Afirmación | Fuente | Fecha | Confianza |
|---|---|---|---|---|
| 1.1 | Repo `open-sdr/openwifi` activo; última release etiquetada **1.5.0 ("shahecheng")** el **2025-08-06**, ~1.012 commits en master. **[VERIFICADO]** | https://github.com/open-sdr/openwifi | 2025-08-06 | Alta |
| 1.2 | Repo FPGA acompañante `open-sdr/openwifi-hw` activo (~454 commits, actividad 2024-2026); fecha exacta de último commit no confirmada. | https://github.com/open-sdr/openwifi-hw | 2024-26 | Media |
| 1.3 | La versión open implementa el stack **IEEE 802.11a/g/n** (Wi-Fi 4). **802.11ax NO está en el repo open** — solo vía licencia comercial en openwifi.tech. **[VERIFICADO]** | https://github.com/open-sdr/openwifi | 2025 | Alta |
| 1.4 | Arquitectura **SoftMAC compatible con mac80211 de Linux**: High-MAC en el driver Linux, **Low-MAC + PHY en la lógica FPGA** de un SoC Xilinx Zynq. Es decir, da acceso real al MAC, no es una NIC caja-negra. | https://github.com/open-sdr/openwifi-hw ; paper VTC2020 ORCA | 2020/25 | Alta |
| 1.5 | La FPGA implementa una Low-MAC DCF (CSMA/CA) con timing SIFS de **10 µs**. | README del repo | 2025 | Alta |
| 1.6 | Expone control fino del medio vía **"time slicing based on MAC address (time gated/scheduled FPGA queues)"**, configurable en runtime con `sdrctl`. Esta es su capacidad TDMA-like / control de airtime, **y está en el código open**. **[VERIFICADO]** | README del repo | 2025 | Alta |
| 1.7 | Permite TDMA determinista vía colas FPGA time-gated y extensiones Wireless-TSN ("W-TSN"). **El límite open/pago es BORROSO**: el time-slicing básico está open; el TSN más avanzado se posiciona como feature comercial. *[INCIERTO el límite exacto]* | https://idlab.ugent.be/.../openwifi-wireless-time-sensitive-networking ; openwifi.tech/subscriptions/w-tsn | ~2024 | Media |
| 1.8 | Ancho de banda instantáneo por defecto **20 MHz** sobre sintonía 70 MHz–6 GHz (AD9361/AD9364); reconfigurable a 10 MHz (802.11p) y 2 MHz (802.11ah). | README del repo | 2025 | Alta |
| 1.9 | Throughput medido **modesto**: iperf TCP ~40-50 Mbps, UDP ~50 Mbps (mejor caso con agregación). **[VERIFICADO]** | README del repo | 2025 | Alta |
| 1.10 | **Doble licencia**: AGPLv3 para la versión open; licencia comercial separada para 802.11ax y TSN avanzado. **[VERIFICADO]** | github.com/open-sdr/openwifi | 2025 | Alta |
| 1.11 | Hardware soportado: Zynq-7000 (ZC706, ZedBoard, ZC702 + FMCOMMS2/3/4), ZynqMP (ZCU102), SOMs AD9361/64 (ADRV9364-Z7020, ADRV9361-Z7035), y placas comunitarias de menor costo: **MicroPhase AntSDR (E310v2/E200), SDRpi, NeptuneSDR, LibreSDR**. ~12 placas. | github.com/open-sdr/openwifi-hw | 2025 | Alta |
| 1.12 | **ADALM-PLUTO NO es target soportado** (Zynq-7010 monocore, sin recursos PL/PS suficientes). *[Inferencia por ausencia — INCIERTO]* | lista de placas openwifi-hw | 2025 | Media |
| 1.13 | Paper fundacional: Jiao et al., *"openwifi: a free and open-source IEEE802.11 SDR implementation on SoC"*, VTC2020-Spring, Antwerp. | orca-project.eu/.../openwifi-vtc | 2020 | Alta |
| 1.14 | Uso académico continuo hasta 2025 (p.ej. *SIMO Wi-Fi Radar for Vital Signal Sensing*, IEEE JC&S 2025; CSI fuzzer, WiSec 2021). | doc/publications.md del repo | 2021/25 | Alta |
| 1.15 | Costo es limitación real: plataformas Zynq+FMCOMMS o SOMs ADRV93xx van de cientos a varios miles de USD; AntSDR/LibreSDR bajan el costo de entrada. *[Precios exactos no verificados — INCIERTO]* | — | 2025 | Baja |

**Síntesis OpenWiFi:** vivo y mantenido hasta al menos mediados de 2025; **es la mejor opción existente para validar un algoritmo TDMA/coordinación sobre radios reales con acceso a la Low-MAC.** Pero techo en a/g/n, 20 MHz, ~50 Mbps; ax y TSN avanzado detrás de licencia comercial; requiere hardware Zynq no trivial.

---

## 2. Estado del cierre de firmware por vendor (2024-2026)

**Confirma la tesis empírica: ath9k (11n) es el último abierto hasta el firmware; de 11ac en adelante todo driver mainstream depende de un blob propietario que corre la Low-MAC.**

| # | Afirmación | Fuente | Fecha | Confianza |
|---|---|---|---|---|
| 2.1 | **ath9k** es driver SoftMAC 100% FOSS para chips Atheros 802.11n PCI/PCIe/AHB; los PCIe **NO requieren blob de firmware**. **[VERIFICADO en docs del kernel: "completely FOSS wireless driver for all Atheros 802.11n PCI/PCI-Express and AHB"]** | https://wireless.docs.kernel.org/.../ath9k.html | actual | Alta |
| 2.2 | La variante USB (`ath9k_htc`, AR9271/AR7010) **sí** carga firmware, pero QCA lo liberó como **software libre** (ClearBSD/MIT/GPLv2). Matiz: "sin blob" es estricto solo en PCIe; en USB el blob es abierto. De cualquier modo sostiene el marco "último 100% open". | https://github.com/qca/open-ath9k-htc-firmware | vigente | Alta |
| 2.3 | **ath10k** (QCA 802.11ac) **requiere blobs propietarios** (firmware-N.bin + board-2.bin); MAC de bajo nivel **offloaded a firmware**. Este es el quiebre generacional: 11n ath9k (abierto) → 11ac ath10k (firmware cerrado). | https://wireless.docs.kernel.org/.../ath10k.html ; github.com/kvalo/ath10k-firmware | actual | Alta |
| 2.4 | **ath11k** (QCA 802.11ax/WiFi6) también requiere blobs propietarios; merge en Linux 5.6 (mar-2020). | https://wireless.docs.kernel.org/.../ath11k.html | actual | Alta |
| 2.5 | **ath12k** (QCA 802.11be/WiFi7) es driver mac80211 abierto pero requiere blobs; merge ~Linux 6.3 (2023). | https://lwn.net/Articles/915152/ | 2023 | Alta |
| 2.6 | **mt76** (MediaTek) es driver mac80211 abierto, pero todas las partes ax/be (MT7915/16, MT7921/22, MT7996/7925) **requieren blobs** (linux-firmware/mediatek/). | https://wireless.docs.kernel.org/.../mediatek.html | actual | Alta |
| 2.7 | MediaTek es el vendor moderno más amigable: contribuye drivers mt76 abiertos y participa en OpenWrt; el **OpenWrt One (2024)** usa MediaTek (MT7981B Filogic, WiFi6) por esa apertura. **La apertura es a nivel driver; el firmware sigue siendo blob.** | https://www.theregister.com/2024/12/02/openwrt_one_foss_wifi_router/ | dic-2024 | Alta |
| 2.8 | **Broadcom brcmfmac** es **FullMAC**: la MAC entera vive en firmware cerrado on-chip; requiere blob propietario. Es el ejemplo canónico de FullMAC/firmware cerrado. | https://wireless.docs.kernel.org/.../brcm80211.html | actual | Alta |
| 2.9 | **Intel iwlwifi**: driver abierto en mainline, pero **todo chip Intel necesita microcódigo propietario** (iwlwifi-*.ucode). Driver abierto, firmware cerrado. | https://wireless.wiki.kernel.org/en/users/drivers/iwlwifi | actual | Alta |
| 2.10 | **Realtek rtw88** (11ac) y **rtw89** (11ax/be, incl. RTW8922AE WiFi7) son drivers abiertos pero requieren blobs. | https://wireless.docs.kernel.org/.../rtl819x.html | actual | Alta |
| 2.11 | **mac80211 es explícitamente un framework SoftMAC**: implementa la MAC superior/MLME en el host y empuja la MAC de bajo nivel timing-crítica (y opc. rate control, cifrado, fragmentación) a hardware/firmware. *NOTA: los docs NO dicen "desde 11ac el timing se movió a firmware" — esa causalidad es inferencia.* | https://docs.kernel.org/driver-api/80211/mac80211.html | actual | Alta |
| 2.12 | Eficiencia 11n/ac depende de A-MPDU + Block-ACK (muchos frames + un block-ack en secuencias SIFS-acotadas), lo que motiva mover el MAC timing-crítico fuera del host. *Las fuentes establecen el rationale de timing, NO que por eso se cerró el firmware.* | cs.uwaterloo.ca "Demystifying frame aggregation" (2021) | 2019/21 | Media |
| 2.13 | **[CAVEAT clave]** Ninguna fuente primaria afirma verbatim "802.11ac empujó la Low-MAC a firmware cerrado". Versión defendible: de 11ac en adelante **todo** driver Linux que envía requiere blob propietario que corre la Low-MAC, mientras que el ath9k PCIe (11n) no. | síntesis | 2026 | Alta (patrón) / Media (causa) |
| 2.14 | **(2024-2026)** Ningún vendor mayor liberó firmware ax/be de producción a mediados de 2026. Qualcomm no participa en OpenWrt; su soporte WiFi viene de QSDK filtrado o ingeniería inversa. El único punto brillante de firmware abierto sigue siendo el legacy ath9k_htc (11n). *[ausencia de evidencia; fuente comunitaria]* | habr.com/.../990172 | 2024-25 | Media |
| 2.15 | WCN7850 (WiFi7) ganó camino de driver ath12k abierto en OpenWrt en 2024 — la apertura *a nivel driver* avanza para chips actuales, pero siempre sobre blob cerrado. | github.com/openwrt/openwrt/pull/15945 | 2024 | Media |

---

## 3. WiFi 6/7 open y SDR — ¿hay algo más allá de OpenWiFi?

**Conclusión dura: a mediados de 2026 NO existe ninguna implementación open-source de 802.11ax (WiFi6) ni 802.11be (WiFi7), ni SDR ni FPGA. El ecosistema abierto topa en a/g/n/ac. ESP32 (incl. los ax C5/C6) es blob cerrado.**

| # | Afirmación | Fuente | Fecha | Confianza |
|---|---|---|---|---|
| 3.1 | openwifi soporta a/g (completo) y n (parcial, MCS0-7) a 20 MHz; **sin ac ni ax en el release open**. | github.com/open-sdr/openwifi | 2024-25 | Alta |
| 3.2 | 802.11ax en openwifi es solo "trabajo futuro"/comercial — **NO hay 802.11ax open en openwifi**. | github.com/open-sdr/openwifi ; openwifi.tech | 2024-25 | Media-alta |
| 3.3 | **gr-ieee802-11** (Bastian Bloessl, GNU Radio) es transceptor OFDM solo para **802.11a/g/p** — sin n/ac/ax. Paper SIGCOMM 2013. | https://github.com/bastibl/gr-ieee802-11 | 2013-19 | Alta |
| 3.4 | gr-ieee802-11 **sí es real-time** (RX/TX en vivo con USRP N210, interop con tarjeta Atheros), no solo offline. | https://www.bastibl.net/gr-ieee802-11-updates/ | ~2014-15 | Alta |
| 3.5 | Proyecto más nuevo **cloud9477/gr-ieee80211 ("GR-WiFi")** extiende a **802.11a/g/n/ac** (WiFi5), SISO y hasta 2x2 SU/MU-MIMO en USRP B210, GNU Radio 3.10 — **explícitamente SIN 802.11ax**. | https://github.com/cloud9477/gr-ieee80211 ; arXiv 2501.06176 | ene-2025 | Alta |
| 3.6 | GR-WiFi main "no estable aún"; soporta real-time (USRP) y offline. | github.com/cloud9477/gr-ieee80211 | 2025 | Media |
| 3.7 | Existe un transceptor open **802.11ah** (WiFi HaLow sub-GHz, PPDU 1 MHz) derivado de gr-ieee802-11 — adyacente, pero es HaLow, no ax. | github.com/irongiant33/gr-ieee802-11ah | ~2023-24 | Media |
| 3.8 | **w-SHARP**: proyecto FPGA académico que implementa features estilo 802.11ax (trigger frame, OFDMA multiusuario) para WiFi industrial determinista — lo más cercano a una "MAC/PHY ax" en FPGA abierta, pero es artefacto de investigación, no stack ax general. | digibuo.uniovi.es/.../wsharp.pdf | ~2020-21 | Media |
| 3.9 | Otros bloques PHY FPGA abiertos (OpenOFDM; Nuand bladeRF-wiphy con mac80211; Mango/WARP) son todos era a/g/n, **no ax/be**. WARP está mayormente dormido. | github.com/jhshi/openofdm ; nuand.com/bladerf-wiphy | 2018-23 | Alta (no-ax) |
| 3.10 | **NO se encontró ninguna implementación open de 802.11be (WiFi7)**, ni SDR ni FPGA. *[hallazgo negativo]* | búsqueda exhaustiva | 2026-06 | Media-alta |
| 3.11 | **srsRAN es solo celular** (4G/5G NR, AGPLv3, 3GPP); **NO existe un "srsWiFi"** — no toca 802.11. Importante para no confundir el precedente celular con WiFi. | github.com/srsran ; 5G-MAG/srsRAN | 2024-25 | Alta |
| 3.12 | **ESP32 WiFi MAC/PHY es cerrado**: esp_wifi son blobs binarios precompilados (libnet80211.a, libpp.a) en espressif/esp32-wifi-lib; licencia Apache-2.0 pero **binario sin fuente**. | github.com/espressif/esp32-wifi-lib | 2024-25 | Alta |
| 3.13 | **esp32-open-mac** (RE comunitario) puede TX/RX raw frames, configurar el filtro de paquetes y asociarse a APs abiertos, **pero NO controla aún rate/modulación/frecuencia/TX-power, sin WPA2, no production-ready** (init HW necesita ~53k accesos a registros reverse-engineered). | https://esp32-open-mac.be/posts/0005-the-road-ahead/ ; charla 38C3 dic-2024 | 2024 | Alta |
| 3.14 | **ESP32-C5 (WiFi6 dual-band) y C6 (WiFi6 2.4GHz)** anuncian features ax (OFDMA, MU-MIMO, TWT) pero **el stack WiFi sigue siendo el mismo blob cerrado**; esp32-open-mac apunta al ESP32 original, no a los chips ax. **No hay acceso open a la Low-MAC ax en ESP32.** | espressif.com/.../esp32-c5 , .../esp32-c6 | 2024-26 | Media-alta |
| 3.15 | **Nexmon** (parcheo de firmware Broadcom/Cypress) permite monitor mode, inyección raw y hasta jamming reactivo en chips reales — **pero los chips soportados son viejos** (BCM4339/Nexus5, BCM43430a1/Pi3, BCM4358, BCM4343x) y da control a nivel firmware, no de timing PHY arbitrario. Upstream lento; forks comunitarios activos. *[lista de chips parcialmente DESACTUALIZADA]* | github.com/seemoo-lab/nexmon | core 2017-20 | Alta (capacidad) / Media (mantenimiento) |
| 3.16 | TDMA software sobre WiFi comodín **es viable vía mac80211**: **Det-WiFi** (TDMA multihop sobre COTS 802.11) y **hMAC** (TDMA/CSMA híbrido host-side sobre cfg80211/mac80211). *Pero son era a/g/n, no ax.* | hindawi.com/.../4943691 ; tu-berlin hMAC | 2016/17 | Alta |
| 3.17 | **[Contrapunto]** *"TDMA on COTS hardware: fact and fiction revealed"* argumenta que el timing TDMA preciso sobre WiFi comodín **es difícil** por el no-determinismo del firmware/MAC — caveat relevante para cualquier claim de TDMA sobre chips comerciales. | researchgate.net/publication/272028959 | ~2014 | Media |

---

## 4. Dimensión de comunidad / política (EFF / FSF / FCC)

| # | Afirmación | Fuente | Fecha | Confianza |
|---|---|---|---|---|
| 4.1 | La **regla FCC 2016 NO obligó a bloquear el firmware del router** en bloque; solo exigió impedir la modificación de **parámetros RF** (frecuencia, modulación, potencia) en U-NII 5 GHz. | prplfoundation.org/.../fcc-open-source-router-software-is-still-legal | 2015-09-25 | Alta |
| 4.2 | La FCC confirmó que los fabricantes "*podían* elegir prohibir mods de software, pero una solución distinta que logre el mismo fin sería aceptable" — bloquear todo el firmware fue el camino *fácil* de cumplimiento, no un requisito. | (misma fuente) | 2015 | Alta |
| 4.3 | En la práctica varios fabricantes (notablemente **TP-Link**) bloquearon el router entero porque bloquear solo el módulo RF era más difícil — bloqueando de hecho OpenWrt/DD-WRT en modelos 2016. | cnx-software.com (2016-02) ; hackaday.com (2016-02-26) | 2016-02 | Alta |
| 4.4 | En un acuerdo (ago-2016) la FCC **obligó a TP-Link a SOPORTAR firmware de terceros/open-source** y pagar multa de USD 200.000. | build.slashdot.org (2016-08-01) | 2016-08 | Alta |
| 4.5 | **EFF** celebró el acuerdo pero advirtió que la ambigüedad de la FCC incentivaba a los vendors a **bloquear el firmware de bajo nivel** en vez de adoptar soluciones realmente abiertas. | https://www.eff.org/deeplinks/2016/08/fcc-settlement-requires-tp-link-support-3rd-party-firmware | 2016-08 | Alta |
| 4.6 | Campaña **"Save WiFi" (2015)**: coalición EFF + FSF + SFLC + Conservancy + OpenWrt + LibreCMC + Qualcomm contra el NPRM de la FCC; se le acredita haber forzado la aclaración de que los routers no quedarían vetados de flashear firmware de terceros. | libreplanet.org/wiki/Save_WiFi | 2015 | Alta |
| 4.7 | **FSF/ath9k**: Atheros liberó un driver wireless 100% libre (ath9k) sin blobs — el WiFi canónico avalado por FSF para distros linux-libre (Trisquel, Parabola). | https://www.fsf.org/news/ath9k | 2008 | Alta |
| 4.8 | Certificación **"Respects Your Freedom" (RYF)** al adaptador USB Atheros AR9271 de ThinkPenguin (ath9k_htc), tras que QCA liberara el firmware del dispositivo como software libre. | https://www.fsf.org/news/ryf-certification-thinkpenguin-usb-with-atheros-chip | 2013-04 | Alta |
| 4.9 | **linux-libre** (origen FSF Latinoamérica) quita blobs no-libres del kernel vía script "deblob" — por eso chips sin blob como ath9k importan para distros 100% libres. | fsfla.org/.../linux-libre | vigente | Alta |
| 4.10 | Sobre **SDR**: el análisis de la SFLC (2007) concluyó que las reglas SDR de la FCC **no restringen** a desarrolladores/distribuidores FOSS de software para SDR; la responsabilidad de mantener RF en spec recae en el hardware certificado. *[ANÁLISIS VIEJO 2007 — verificar contra la guía FCC SDR actual antes de citar con precisión]* | softwarefreedom.org/resources/2007/fcc-sdr-whitepaper.html | 2007 | Media |
| 4.11 | **(2024)** OpenWrt + Software Freedom Conservancy lanzaron el **OpenWrt One** (WiFi6, USD 89,99), que pasó compliance FCC completa para hardware **y** firmware — SFC lo enmarca como prueba de que "copyleft, derecho a reparar y requisitos FCC son todos alcanzables en un producto", refutando que las reglas FCC choquen con firmware modificable. | theregister.com (2024-12-02) ; techcrunch.com (2024-12-03) | 2024-11/12 | Alta |
| 4.12 | **(2026) "FCC router ban"**: nueva acción FCC que prohíbe **modelos nuevos de router NO fabricados en EE.UU.** por seguridad nacional — **distinto de las reglas de firmware**. SFC subraya: **"la FCC notablemente NO restringe los cambios de software hechos por los dueños de routers en EE.UU."** y no hay indicio de que las actualizaciones que la gente hace a sus propios routers violen regla alguna. El OpenWrt One (ya aprobado FCC) sigue a la venta en EE.UU. **[VERIFICADO]** | https://sfconservancy.org/blog/2026/apr/02/fcc-router-ban/ | 2026-04-02 | Alta |

**Lectura política para el informe:** la narrativa "la FCC bloqueó el firmware libre" es **inexacta y debe matizarse**. La FCC nunca exigió cerrar el firmware (solo los parámetros RF); el cierre fue una *decisión comercial* de los vendors por ser el camino de cumplimiento más barato. El OpenWrt One (2024) y la postura de SFC en 2026 demuestran que firmware libre + compliance FCC **conviven**. El argumento fuerte de AlterMundi no es "la regulación lo prohíbe" sino "**los vendors cerraron la Low-MAC en firmware por razones comerciales/de timing, y eso, no la ley, es lo que bloquea la coordinación fina del medio**".

---

## 5. ¿Por qué un algoritmo de referencia validado justifica el hardware abierto?

**El patrón "validar el algoritmo en hardware de nicho de-riesga la inversión en hardware/silicio" está bien establecido. El precedente más directo y citable es el propio OpenAirInterface/srsRAN → O-RAN en celular, y el propio origen de OpenWiFi.**

| # | Afirmación | Fuente | Fecha | Confianza |
|---|---|---|---|---|
| 5.1 | **OpenAirInterface y srsRAN** dan stacks 4G/5G open que corren sobre USRP comodín, validando algoritmos 3GPP/O-RAN end-to-end a bajo costo **antes** de comprometer hardware open RAN — sembrando el ecosistema O-RAN. *(Precedente más fuerte: software valida → hardware abierto se justifica.)* | kb.ettus.com/5G_srsRAN_End-to-End | ~2023 | Alta |
| 5.2 | srsRAN es CU/DU O-RAN-native compliant con 3GPP y O-RAN Alliance — demuestra que un stack validado en software/SDR es la rampa de entrada al ecosistema de hardware/interfaces abiertas. | researchgate.net/publication/374989956 | 2023 | Alta |
| 5.3 | **OpenWiFi se inició en 2017 dentro del proyecto EU H2020 ORCA** justamente para dar a investigadores acceso modificable a PHY/MAC que el firmware WiFi cerrado niega — ese argumento de flexibilidad-de-investigación **es la justificación explícita del esfuerzo de hardware WiFi abierto**. (Precedente directamente análogo al caso AlterMundi.) | cnx-software.com (2019-12-16) | 2019 | Alta |
| 5.4 | La arquitectura de OpenWiFi (PHY/MAC timing-crítico en Verilog FPGA, MAC superior en Linux) es precisamente la que permite modificar y validar un algoritmo de coordinación/MAC sobre radios reales — el payoff declarado de construir hardware abierto. | researchgate.net/publication/342582824 | 2020 | Alta |
| 5.5 | Se construyó un **kit Wireless-TSN y un protocolo TDMA/CSMA híbrido para tráfico de robots SOBRE OpenWiFi**, logrando sync de beacon ~10 µs y **~93% de reducción de errores de deadline perdido** vs baseline CSMA — caso concreto donde un algoritmo de coordinación se validó sobre el hardware WiFi abierto, justificándolo retroactivamente. *[la cifra 93% mapea al trabajo robot-traffic; verificar paper exacto antes de citar el número]* | researchgate.net/publication/382354713 | 2021-24 | Media |
| 5.6 | **Soft-TDMAC** (Djukic & Mohapatra) demostró una MAC TDMA con sincronización de red a nivel microsegundo como overlay de software sobre 802.11 comodín — un algoritmo probado sobre hardware existente (limitado) antes de invertir más profundo. | ieeexplore.ieee.org/document/5062104 | 2009 | Alta |
| 5.7 | **RT-WiFi** (UT Austin, Mok et al.): protocolo data-link TDMA sobre PHY 802.11 con entrega determinista hasta 6 kHz, implementado primero en COTS y validado en un sistema real de rehabilitación de marcha — validación algoritmo-primero sobre hardware disponible. | cs.utexas.edu/~cps/rt-wifi.html | ~2013 | Alta |
| 5.8 | **Det-WiFi**: MAC TDMA multihop para WiFi industrial determinista sobre 802.11 comodín — valida un esquema de coordinación TDMA sobre radios existentes como precursor del WiFi determinista a medida. | hindawi.com/.../4943691 | 2017 | Alta |
| 5.9 | **B.A.T.M.A.N./batman-adv**: la comunidad Freifunk desarrolló y validó (contra datos de red real) un algoritmo de routing/coordinación que sustentó el ecosistema de routers/firmware mesh comunitario. | en.wikipedia.org/wiki/B.A.T.M.A.N. | ~2006+ | Alta |
| 5.10 | **LibreMesh** surgió fusionando firmwares comunitarios ya validados — **AlterMesh (AlterMundi, Argentina)**, qMp (guifi.net), eigenNet (ninux) — i.e. algoritmos/firmware validados en redes reales justificaron el esfuerzo de firmware abierto consolidado. *(Precedente propio de AlterMundi.)* | old.libremesh.org ; wiki.p2pfoundation.net/Libre-Mesh | 2013-15 | Alta |
| 5.11 | **GNU Radio + USRP** es la plataforma canónica para prototipar algoritmos wireless en software antes de portar a FPGA/ASIC; RFNoC de Ettus institucionaliza el flujo "valida el algoritmo, luego comprometé hardware". | digilent.com/blog/what-is-usrp | ~2021 | Alta |
| 5.12 | El alto costo NRE de los ASIC hace de SDR/FPGA el paso estándar de de-riesgo: validar RTL/timing/algoritmo en FPGA-SoM (y MPW como Tiny Tapeout) antes del tape-out — estructura general del argumento "validación de software de-riesga inversión en hardware". | sciencedirect.com/topics/engineering/software-defined-radio | ~2023 | Media |
| 5.13 | **LimeSDR/Lime Microsystems** crowdfundeó SDRs de hardware 100% abierto (esquemáticos, layout, firmware DSP BSD) — caso donde un ecosistema de software/SDR abierto ancló hardware de radio crowdfundeado. | crowdsupply.com/lime-micro/limesdr | 2016-19 | Alta |
| 5.14 | **Meshtastic** muestra el camino inverso de mesh comunitaria: protocolo/firmware LoRa mesh open validado sobre radios baratas de estante (~USD 10-70), impulsando adopción masiva y ecosistema de nodos open sin silicio a medida. | meshtastic.org/docs/introduction | 2026 | Alta |

**Estructura del argumento para el informe (sintetizada):**
1. El cuello de botella no es teórico sino de *acceso*: la coordinación fina del medio (TDMA/airtime) requiere control del timing de la Low-MAC, que en chips comerciales 11ac/ax+ está sellado en firmware (sección 2).
2. Validar el algoritmo de coordinación primero en un platform abierto de nicho (OpenWiFi/SDR, o incluso TDMA-software estilo Det-WiFi/RT-WiFi) **demuestra que el algoritmo funciona y cuantifica la ganancia** (p.ej. el ~93% de reducción de deadlines perdidos sobre OpenWiFi, §5.5; los µs-sync de Soft-TDMAC/RT-WiFi).
3. Ese PoC validado **de-riesga y justifica** la inversión mayor en empujar hardware WiFi de última generación abierto — exactamente como srsRAN/OAI sembró O-RAN (§5.1-5.2) y como OpenWiFi mismo nació para dar el acceso que el firmware niega (§5.3-5.4).
4. AlterMundi tiene precedente propio en esta lógica: LibreMesh consolidó firmwares mesh ya validados en campo (§5.10).

---

## Advertencias de fiabilidad (resumen)

- **Causalidad "11ac → firmware cerrado" (§0, §2.12-2.13):** patrón empírico sólido; causa exacta es interpretación de ingeniería, no hecho citado. Presentar con cuidado.
- **Límite open/pago de TSN en OpenWiFi (§1.7):** borroso; el time-slicing básico está open, el TSN avanzado parece comercial.
- **Cifra "93%" de OpenWiFi TSN (§5.5):** verificar el paper exacto antes de citar el número.
- **Whitepaper SDR de SFLC (§4.10):** de 2007, predata la guía FCC SDR actual.
- **"FCC router ban" 2026 (§4.12):** acción regulatoria reciente y en movimiento; verificado que es sobre fabricación, NO firmware, pero su estado legal exacto a mediados de 2026 conviene re-chequear antes de publicar.
- **Lista de chips de Nexmon (§3.15):** parcialmente desactualizada (chips viejos; upstream lento).
- **Precios de hardware OpenWiFi (§1.15):** no verificados directamente.
- **ADALM-PLUTO no soportado (§1.12):** inferencia por ausencia.
