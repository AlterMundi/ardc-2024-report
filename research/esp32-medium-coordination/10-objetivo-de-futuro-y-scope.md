# 10 — Objetivo de futuro y scope: ¿avanzada o curiosidad? ¿hasta dónde?

> **Origen:** observación de Fede (2026-06-19/20): ath9k es WiFi viejo y ESP32 también; ni uno ni otro es el futuro.
> ¿Dónde un desarrollo novel tiene impacto tal que se presente como **avanzada** y no como curiosidad técnica? ¿Cuál es el
> **scope** — meramente académico, limitado a ath9k? Evidencia citable en `research/plataforma-wifi-abierta.md` y
> `research/estrategia-scope-hardware-abierto.md`. Cierra el arco de la línea (`01`→`09`).

## 0. La respuesta corta
**El scope NO es académico ni está limitado a ath9k.** ath9k y ESP32 son **peldaños de validación**, no el destino. Y hay un
giro que reordena toda la estrategia: **el alto rendimiento WiFi abierto NO está bloqueado por el silicio — el RFSoC ya tiene
el ancho de banda de sobra — sino por la ausencia de un PHY 802.11ax/be abierto.** Es un problema de **PHY-IP**, no de comprar
hardware. Eso convierte al proyecto en algo concreto y financiable: **la prueba-de-necesidad que justifica desarrollar/liberar
ese baseband abierto**, extendiendo el linaje OpenWiFi.

## 1. La escalera (plataforma de validación ≠ objetivo de impacto)
El error es buscar una plataforma a la vez barata, abierta, moderna y de alto rendimiento — no existe. Es una escalera:

| Peldaño | Rol | Aporta | Por qué no es el destino |
|---|---|---|---|
| **ESP32 (clásico/C6)** | demo ultra-barato + banco de **medición** (CSI/RSSI → G(t)) | accesibilidad, la capa de *sensing* del grafo (`07`) | sin control µs, blob cerrado |
| **ath9k** | última MAC WiFi controlable desde el host | valida la **lógica** a tasas reales (precedente soft-TDMA: hMAC, Soft-TDMAC) | 802.11n, ~150 Mbps, obsoleto |
| **OpenWiFi (FPGA/SDR)** | **la única Low-MAC abierta con radio real y timing µs** | **donde se implementa el TDMA de µs y se MIDE el impacto**; ya soportó MACs TDMA (RT-WiFi, RTAS 2022) | a/g/n, 20 MHz, ~40-50 Mbps |
| **Baseband WiFi 6/7 abierto sobre RFSoC** | **el destino** | alto rendimiento + MAC abierta | **no existe (2026)** — es lo que el esfuerzo empuja |

## 2. Dónde hay impacto medible HOY: OpenWiFi (y por qué es avanzada, no curiosidad)
OpenWiFi (open-sdr, IDLab-imec/UGent; AGPLv3; **vivo**, commits a may-2026; ~$250-400/nodo con AntSDR E310) es la única
plataforma con **Low-MAC en FPGA reprogramable**: README literal *"10us SIFS is achieved"* + *"time slicing based on MAC
address"*, con timestamping HW y sync de precisión µs. **No es especulativo:** ya se construyeron MACs TDMA encima — **RT-WiFi
(IEEE RTAS 2022)**, TDMA/CSMA híbrido (2025), OFDMA coordinado con White Rabbit (2025). Un algoritmo de coordinación de
grafo-de-conflicto de µs **es implementable ahí hoy**, y el impacto se mide en **recuperación de throughput bajo nodo oculto +
latencia/jitter/determinismo** — métricas duras, sobre un PHY WiFi real. Eso es **desarrollo de avanzada**.

**El matiz honesto sobre "alto rendimiento":** OpenWiFi es 20 MHz SISO (~50 Mbps), así que el *número absoluto* de throughput
es modesto. La ganancia de la coordinación bajo nodo oculto es **real y rate-independent en términos relativos** (recupera el
colapso que el CSMA sufre), pero su impacto *a escala WiFi 6* solo se materializa cuando exista el baseband abierto de alta
tasa. → **Hoy se demuestra el determinismo + la recuperación; el impacto de alto rendimiento es la promesa de futuro que
justifica el silicio.**

## 3. El giro que reordena la estrategia: el cuello no es el silicio
- **RFSoC resuelve el ancho de banda:** Gen3 Zynq UltraScale+ RFSoC (ADC 5 GSps / DAC 10 GSps; **RFSoC 4x2 ~$2,150**) captura
  por sampleo directo un canal de 320 MHz — **excede de sobra** los 160 (WiFi6) / 320 (WiFi7). El muro de ~56 MHz del AD9361
  (el SDR barato) **no aplica al RFSoC**.
- **Lo que NO existe es el PHY:** **ningún 802.11ax/be abierto en 2026** (OpenWiFi-ax es de pago; WARP topa en 40 MHz legacy;
  nada de 802.11be). → **La brecha es el baseband HE/EHT abierto, no el hardware.**
- **Ni siquiera MediaTek (el vendor más abierto) abre la MAC:** vive en blobs `*_wm.bin`; OpenWrt One hereda lo mismo; solo TWT
  estándar como scheduler grueso dentro del firmware cerrado. **No hay TDMA custom de µs sobre WiFi 6 comercial.**
- **GNU Radio prototipa la lógica, no ejecuta el TDMA:** su propio autor documenta que el PHY-software *"cannot respond to ACKs
  in time"* (round-trip host↔SDR de cientos de µs–ms vs SIFS de 10-16 µs) → sirve para CSI/sounding/research/lógica, no para
  MAC en tiempo real.

**Consecuencia para "empujar hardware abierto":** el esfuerzo realista NO es comprar/diseñar RF nuevo — es **financiar y
liberar un baseband HE (luego EHT) abierto y portarlo a conversores RFSoC**, extendiendo OpenWiFi. Es un trabajo de **PHY-IP,
plurianual**, donde el núcleo más creíble ya existe (imec/Ghent/OpenWiFi + el transceiver OFDMA de Aslam 2024) y donde ARDC/
NLnet ya tienen huella (OpenWiFi v1.5.0 fue financiado por NLnet).

## 4. El objetivo que justifica el esfuerzo (y por qué no es curiosidad)
> **Ser la *prueba-de-necesidad* y la *implementación de referencia* que catalizan una radio WiFi de nueva generación
> (WiFi 6+), alto rendimiento y base abierta, con MAC programable — la primera plataforma donde una red comunitaria puede
> implementar su propia coordinación de acceso al medio.**

El algoritmo demuestra que el control de bajo nivel **rinde impacto**; por tanto justifica el PHY/MAC abierto. Sin él, "queremos
WiFi 6 abierto" es un deseo; con él, es un caso técnico. Precedentes que lo respaldan:
- **OpenWiFi/ORCA (EU H2020, 2017)** nació *exactamente* para dar el acceso de bajo nivel a la MAC que el firmware niega — el
  precedente casi-idéntico, **ya ejecutado**.
- **O-RAN** = la analogía moderna: software RAN de referencia abierto (OAI/srsRAN sobre SDR) que **co-desarrolla** specs de
  hardware white-box. *(Causalidad exacta = inferencia; usar "co-desarrollo del ecosistema", no "lo causó".)*
- **"Implementación de referencia" es categoría formal con valor propio** (NIST; IETF "running code"/RFC 7942; BSD sockets,
  WebKit, CPython como pruebas de impacto desproporcionado) → **desactiva la objeción "solo académico".**
- **Precedente propio:** **LibreRouter** (AlterMundi, FRIDA + ISOC Beyond the Net) — AlterMundi ya empujó hardware libre
  comunitario. Financiamiento probado para esto: **NLnet/NGI Zero** (€21,6M hasta 2027) y **ARDC**.

## 5. Scope explícito (lo que se promete y lo que no)
- **NO es:** curiosidad académica; ni un trabajo "limitado a ath9k"; ni la construcción de un chip WiFi.
- **SÍ es:** un proyecto **escalonado de implementación de referencia + advocacy de hardware abierto**:
  1. **Especificación de referencia** del algoritmo de coordinación (lo ya hecho, `01`-`09`).
  2. **Validación de la lógica** en bancos baratos (ESP32 para sensing/ms; ath9k para la lógica a tasa real) — peldaños.
  3. **Prototipo load-bearing en OpenWiFi**: TDMA de µs sobre grafo de conflicto medido, con impacto medido (recuperación bajo
     nodo oculto + determinismo) sobre un PHY WiFi real abierto.
  4. **Posicionamiento estratégico**: ese prototipo como prueba-de-necesidad que justifica el **baseband WiFi 6/7 abierto sobre
     RFSoC** — el objetivo plurianual, a financiar (NLnet/NGI/ARDC), no entregable de este grant.
- **Framing de TRL honesto (plantilla EIC Pathfinder, que financia TRL 1-3 exigiendo separar visión + breakthrough acotado):**
  el breakthrough acotado y verificable = el algoritmo validado en OpenWiFi; la visión de largo plazo = el WiFi abierto de alta
  tasa. No colapsar los dos: lo primero es asegurable, lo segundo es la dirección.

## 6. Honestidades que el informe NO debe sobre-afirmar (calibración)
- **Silicio WiFi abierto no existe** y el RF analógico abierto está en alfa (IHP SG13G2 "no producción"; Efabless cerró
  mar-2025). El camino real = **baseband digital abierto sobre FPGA/RFSoC + RF comercial** (OpenWiFi depende del AD9361).
- **El impacto de "alto rendimiento" es promesa de futuro**, no demostración de hoy: hoy se demuestra determinismo + recuperación
  bajo nodo oculto a ~50 Mbps; el alto throughput llega con el baseband HE abierto.
- **"OpenWiFi-ax abierto" es falso** (es de pago); el ax abierto hay que construirlo.
- **No atribuir a srsRAN/OAI "causar" silicio abierto** (inferencia); ni decir que Cisco financió LibreRouter (hallazgo
  negativo, falso).

## 7. Encaje en el informe ARDC (§1.5 y la visión)
Esto da a §1.5 un **arco completo y honesto**: del problema (nodo oculto, `01`) → por qué el HW actual no alcanza (`02`,
causalidad firmware) → la lógica del algoritmo y su capa fundacional (grafo de conflicto, `03`/`07`) → el bus de `data(t)`
(`08`) → la agenda fundamentada (`09`) → **el objetivo de futuro: validar en OpenWiFi y empujar el baseband WiFi 6/7 abierto
sobre RFSoC** (este doc). El entregable comprometible es la especificación + el PoC; **la visión de alto rendimiento es la
dirección que justifica el esfuerzo y posiciona a AlterMundi como quien empuja la próxima plataforma abierta** — exactamente lo
contrario de una curiosidad técnica.
