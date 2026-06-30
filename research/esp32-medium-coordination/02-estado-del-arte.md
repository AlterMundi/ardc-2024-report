# 02 — Estado del arte

> Síntesis navegable. La evidencia detallada con fuentes y niveles de confianza está en
> `research/estado-del-arte-mac.md`, `research/esp32-bajo-nivel.md`, `research/esp-now.md` y
> `research/hardware-wifi-open.md`.

## 1. El problema y por qué CSMA/CA no lo resuelve
- El **terminal oculto** (Tobagi & Kleinrock, 1975) degrada el throughput de CSMA porque el
  carrier-sense no detecta al transmisor competidor. La solución original fue señalización fuera de
  banda (Busy-Tone) — conceptualmente, *coordinación adicional*, que es lo que hará el algoritmo.
- **RTS/CTS no alcanza en mesh multi-hop**: Xu/Gerla/Bae (GLOBECOM 2002) muestran que el rango de
  interferencia (~1.78× el de transmisión) excede el supuesto del handshake; Ray et al. (WCNC 2003)
  muestran que en multi-hop RTS/CTS puede *inducir* congestión y bajar el throughput con la carga.

## 2. Precedentes de coordinación (de dónde copiar)
- **PCF / HCCA** (802.11): polling centralizado por el AP. **Obsoletos** (complejidad del scheduler,
  dependencia de sync), pero son el **molde conceptual del round-robin**. Lección: el coordinador no
  conoce las colas de las estaciones → hay que señalizar demanda; y la sync global es el punto de falla.
- **TDMA propietario de WISP** (todos cerrados, mismo patrón): **Nv2** (MikroTik) divide el tiempo en
  *periods* y difunde un *uplink schedule* (group polling); **airMAX** (Ubiquiti) asigna slots y
  *reemplaza* CSMA/CA para PtMP exterior con nodos ocultos; **Mimosa** usa GPS por radio para la sync.
- **Open / académicos (los precedentes directos):**
  - **WiLDNet** (USENIX NSDI 2007) y su base **2P** (MobiCom 2005): fases SynTx/SynRx, sync por
    *marker packets* (sin reloj global), **mejora 2–5×**. Para enlaces de larga distancia direccionales.
  - **JaldiMAC** (NSDR 2010): driver open (Jaldi9K) + scheduler TDMA en el nodo raíz, sobre Atheros
    commodity. Demuestra **soft-TDMA sobre chips WiFi commodity**.
  - **RT-WiFi** (RTSS 2013) y **Det-WiFi** (WCMC 2017): TDMA determinista sobre PHY 802.11 estándar.
  - **TSCH / 6TiSCH** (802.15.4e; RFC 7554/8180/9030): slotting determinista en MCU baratos —
    posible **porque 802.15.4 expone el control de tiempo del radio**. En WiFi no se puede (firmware
    cerrado). *No existe port revisado por pares de TSCH completo sobre WiFi/ESP32* → hueco a llenar.
- **batman-adv opera solo en L2 y no coordina el medio** → hereda el nodo oculto del CSMA/CA. El
  algoritmo va *por debajo/en paralelo*; batman-adv sigue ruteando.

## 3. Coordinador, sincronización y topología parcial
- **Centralizado vs distribuido**: el centralizado da slots casi óptimos pero no escala ni reacciona al
  churn; el distribuido escala y se adapta sin óptimo global.
- **Elección de coordinador**: Bully (asume conectividad completa, mal en topología parcial), **LEACH**
  (cluster-heads rotativos), **max-degree** (nodo con más vecinos).
- **Sync sin GPS** (RBS→TPSN→**FTSP**): FTSP da precisión µs por flooding + regresión de skew + elección
  dinámica de root robusta a fallos — el modelo a seguir en ESP32 (limitado a decenas de µs por jitter).
- **Topología parcial = la condición que crea el nodo oculto**: **restricción de 2 saltos** (ningún par
  a ≤2 saltos comparte slot), garantizable distribuido con **DRAND** en O(δ) **sin sync**. El óptimo
  (coloreo de grafo de conflicto) es **NP-hard** → heurísticas.

## 4. Lo que ESP32 permite y lo que no (cota de realidad)
- **Sí**: inyección de tramas crudas (`esp_wifi_80211_tx`), sniffer con RX-timestamp en µs, ESP-NOW como
  canal de señalización (broadcast, sub-2 ms en buen enlace, **sin límite de receptores en broadcast**),
  coordinación a granularidad de **ms**, sync por radio a decenas de µs.
- **No (por API pública)**: apagar CSMA/CA, TX en instante exacto de µs, slots deterministas de µs.
  ESP-NOW *también* usa CSMA/CA (hasta 31 retransmisiones, delays multimodales de miles de µs).
- **La palanca**: `esp32-open-mac` (SoftMAC del clásico, financiado por NLnet) avanza hacia abrir la MAC
  completa; ahí sí se podría poner CWmin/CWmax=0. **Aún no implementado.** C5/C6 (WiFi6) siguen cerrados.

## 5. El panorama de hardware abierto (por qué importa estratégicamente)
De `research/hardware-wifi-open.md`:
- **ath9k (802.11n) es el último WiFi abierto hasta la MAC.** De 802.11ac en adelante, todo driver Linux
  mainstream (ath10k/11k/12k, mt76, brcmfmac, iwlwifi, rtw88/89) es abierto como *driver* pero depende
  de un **blob que corre la Low-MAC y el timing crítico**.
- **OpenWiFi** (open-sdr, AGPLv3, release 1.5.0 ago-2025) es **el mejor banco para validar TDMA "de
  verdad"**: da acceso real a la Low-MAC FPGA y **ya hace time-slicing por MAC**. Limitado a a/g/n,
  20 MHz, ~50 Mbps; ax y TSN avanzado detrás de licencia comercial; hardware Zynq caro.
- **WiFi 6/7 open no existe** (ni SDR ni FPGA, a 2026). Esto **refuerza** la necesidad de empujar
  hardware nuevo abierto.
- **Corrección de narrativa (importante):** la regla FCC 2016 **no** obligó a cerrar el firmware (solo
  los parámetros RF); el cierre fue decisión **comercial** de los vendors. El argumento fuerte es
  *"los vendors sellaron la Low-MAC en firmware"*, no *"la ley lo prohíbe"*.
- **El precedente "PoC justifica hardware abierto" es sólido**: srsRAN/OpenAirInterface sobre USRP
  sembraron O-RAN; OpenWiFi nació (2017, proyecto EU ORCA) justo para dar el acceso que el firmware
  niega. AlterMundi tiene precedente propio (LibreMesh consolidó firmwares mesh validados en campo).

## 6. Métricas de evaluación (qué medir)
- Throughput de saturación (Bianchi, JSAC 2000) · **índice de Jain** de fairness · **performance
  anomaly** (Heusse, INFOCOM 2003) · **airtime fairness + latencia acotada** (Høiland-Jørgensen, ATC
  2017). Batería para el PoC: (1) throughput agregado bajo nodo oculto forzado vs CSMA/CA;
  (2) Jain por flujo; (3) latencia de peor caso; (4) eficiencia de airtime / goodput.
