# 01 — Planteo del problema

> Línea de investigación abierta para el entregable **MW-CA-RF** del informe ARDC 2026.
> Alimenta la sección **§1.5 (trabajo futuro)** del `cierre/PLAN-DE-CIERRE.md`.

## 0. Desambiguación previa (leer primero): APuP ≠ TDMA

El material del equipo nombra informalmente "algoritmo APuP" al trabajo futuro de MW-CA-RF.
El término es engañoso y conviene fijar la terminología antes de seguir, porque **APuP y TDMA
viven en capas distintas y resuelven problemas distintos.**

```
┌─ Plano 4: RUTEO L2 ──────────── batman-adv, babeld, OLSR
│            ¿por qué camino va el paquete?
├─ Plano 3: PEERING / ASOCIACIÓN ─ 802.11s · ad-hoc · WDS · ► APuP ◄
│            ¿quién es vecino de quién? ¿cómo se direccionan las tramas?
├─ Plano 2: ACCESO AL MEDIO (MAC) ► CSMA/CA ◄ · ► TDMA / polling ◄
│            ¿QUIÉN TRANSMITE CUÁNDO? ¿cómo se reparte el aire?      ← aquí vive el nodo oculto
└─ Plano 1: PHY ───────────────── modulación, el radio
```

- **APuP (Access Point micro-Peering)** = **plano 3**. Parche de `hostapd` (ya en OpenWrt; es el
  PR #1134 / #1200 del informe) donde *cada nodo es AP* y se enlaza con los APs vecinos vía tramas
  de 4 direcciones, pudiendo además atender clientes. Reemplaza a 802.11s como mecanismo de
  *peering*. **Decide con quién te ves, no cuándo hablás.** Su propia documentación (Freifunk,
  2024-08-24) advierte que **no trae coordinación de medio, ni anti-colisión, ni anti-loop**:
  corre sobre el CSMA/CA del chip, intacto. Por eso APuP **no resuelve el nodo oculto** y arrastra
  la penalización de throughput de ~7–10× que mide el equipo.
- **TDMA / polling** = **plano 2**. Divide el tiempo en turnos y *sustituye* al CSMA/CA. **Decide
  quién transmite cuándo.** Es lo único que ataca el nodo oculto en su raíz.

**Son ortogonales.** El nodo oculto es un fallo del plano 2; APuP es del plano 3.

### Dónde se tocan (la idea real del equipo)
APuP hace que cada nodo sea simultáneamente *AP de sus vecinos* y *cliente de ellos*. Esa relación
AP↔cliente es estructuralmente idéntica a la que usan **PCF/HCCA** (sondeo del AP a sus estaciones)
y los TDMA propietarios de WISP (**MikroTik Nv2** "group polling", **Ubiquiti airMAX**, **Mimosa**).
El "round-robin de autenticación sobre APuP" de las notas significa, entonces: **usar la estructura
AP↔cliente que APuP ya crea como canal de señalización para repartir turnos.** APuP es el *sustrato*;
el algoritmo de coordinación es lo nuevo, y pertenece a la familia TDMA.

### Y "round-robin" tampoco es TDMA estricto
| Esquema | Reparto del aire | Sincronía | Ejemplo |
|---|---|---|---|
| CSMA/CA | contención, sin plan | ninguna | WiFi por defecto (falla en nodo oculto) |
| TDMA clásico/estático | slots fijos asignados de antemano | reloj global ajustado (µs) | Mimosa (GPS) |
| **TDMA dinámico / polling** | coordinador concede turnos según demanda | media | PCF/HCCA, Nv2, airMAX |
| **Round-robin / token** | turno rota en orden fijo | media | ← **propuesta del equipo** |
| Reserva (PRMA, R-ALOHA) | contendés una vez, luego "poseés" el slot | media | satélite/celular |

Round-robin es un caso de *polling* (TDMA dinámico), no TDMA de slot fijo. El término correcto para
esta línea es **"coordinación de acceso al medio" / "MAC programada"**, de la que round-robin y TDMA
estático son variantes.

> **Frase ancla:** *APuP decide quiénes son vecinos; el algoritmo de coordinación decide quién
> transmite cuándo. El nodo oculto solo lo arregla el segundo.*

Esto **reconcilia con la propuesta ARDC 2024**, que prometía un "híbrido TDMA/CSMA-CA": la propuesta
ya hablaba de la familia TDMA (plano 2); APuP apareció después como el mecanismo de peering (plano 3)
que habilita montar esa coordinación **sin tocar el driver cerrado**.

## 1. El problema, en una línea

En una mesh WiFi comunitaria, dos enlaces cuyos transmisores no se escuchan entre sí (**nodo oculto**)
colapsan el throughput al transmitir hacia un receptor común — muy por debajo del reparto justo del
canal. El CSMA/CA no puede evitarlo porque el carrier-sense no detecta al competidor. La solución
correcta (coordinación de acceso al medio tipo TDMA) **es imposible de implementar dentro de los
drivers WiFi comerciales modernos** porque la MAC de bajo nivel vive en firmware cerrado desde
802.11ac. **ESP32 es la única plataforma WiFi barata con control de bajo nivel en vías de apertura**,
y por eso es el banco donde validar un algoritmo de referencia.

## 2. La evidencia que ya tiene el proyecto (del corpus ingerido)

- **El problema es real y medido.** `Hidden node.md`: 3× LibreRouter, dos extremos sin verse hacia un
  receptor común; secuencial **20.8 / 21.2 Mbps**, concurrente colapsa a **1.7–4.7 Mbps** (mucho peor
  que el 50% teórico). Es la firma del nodo oculto.
- **TDMA in-driver está bloqueado.** `referencias-personas-instituciones.md` (assessment TDMA): el
  último WiFi con driver 100% open es 802.11n (ath9k); desde 802.11ac la MAC de bajo nivel se delegó a
  firmware cerrado. Veredicto por vendor: MediaTek, Qualcomm Atheros, Broadcom, Intel, Realtek,
  Espressif → todos "unviable" hoy para mesh-TDMA in-driver. (Confirmado y matizado en
  `research/hardware-wifi-open.md`: la forma empírica es sólida; la causal es interpretación de
  ingeniería bien fundada, no hecho citado.)
- **APuP es peering, no coordinación de medio — y no resuelve el nodo oculto.** `ap-up-testing.md` y
  `mt7915-apup-testing.md` midieron APuP en una **topología estrella** (todos los peers asociados a un AP
  común y en rango entre sí), que **evita el par oculto por construcción** — no porque APuP coordine el
  medio. Que dos nodos "se vean como clientes de un AP común" es alcanzabilidad L2 (plano 3); **no** hace
  que sus radios se escuchen en el carrier-sense (plano 2): si estuvieran ocultos, seguirían colisionando.
  Además APuP penaliza el throughput ~7× (MT7915: 215–250 → 33 Mbps) a ~90% (Atheros 5 GHz) por overhead
  de encapsulación VLAN 8021ad + buffer mgmt del driver. **Esa penalización es overhead, no resolución del
  nodo oculto**: confirma que APuP solo no basta.
- **Hay un sustrato de estado distribuido.** `shared-state-merge-strategy.md`: shared-state v3 en Rust
  (485 KB, runtime `smol`) ya propaga eventos efímeros de red — candidato a transportar el estado de
  coordinación. (Tiene su propio bug de convergencia TTL, documentado.)
- **Hay un testbed en preparación para iterar.** `informe-casanueva-riba.md`: banco HIL/CI (Labgrid +
  pytest + runner self-hosted, DUTs físicos y QEMU+vwifi) de becarios UNC dirigidos por javierbrk.
  **Es la infraestructura sobre la que se desarrollará y probará este algoritmo.**

## 3. Por qué ESP32 (y no otra cosa)

| Opción | Control de bajo nivel | Costo | Veredicto |
|---|---|---|---|
| Chipsets WiFi comerciales (ath10k+, mt76, brcm) | ❌ MAC en firmware cerrado | bajo | bloqueado para TDMA in-driver |
| **OpenWiFi (SDR/FPGA Zynq)** | ✅ Low-MAC abierta, ya hace time-slicing | alto (Zynq + transceiver) | mejor banco de validación "de verdad", caro |
| **ESP32 (clásico / C3)** | ⚠️ SoftMAC; `esp32-open-mac` abre la MAC (en progreso) | muy bajo | **único WiFi barato con MAC en vías de apertura** |
| ESP32-C5/C6 (WiFi6) | ❌ aún blob cerrado, sin RE abierto | bajo | no apto hoy para control fino |

ESP32 no es ideal — es el **punto de palanca**: hardware accesible, comunidad, y un esfuerzo abierto
(`esp32-open-mac`, financiado por NLnet, charla 38C3 dic-2024) que avanza hacia una MAC totalmente
libre. Validar el algoritmo aquí lo hace **reproducible por cualquiera con dos placas de USD 5**.

## 4. Alcance honesto de lo que ESP32 permite hoy (no sobrevender)

De `research/esp32-bajo-nivel.md` y `research/esp-now.md`:

- **Lo factible hoy:** coordinación a granularidad de **milisegundos** (slots/turnos de ms),
  señalización por ESP-NOW (broadcast, latencia sub-2 ms en buen enlace), sincronización por
  RX-timestamp/beacon a **decenas de µs**, inyección de tramas crudas con `esp_wifi_80211_tx()`.
- **Lo NO factible hoy por API pública:** apagar el CSMA/CA del hardware, programar TX en un instante
  exacto de µs, slots deterministas de microsegundos. El CSMA/CA corre en HW/blob; ESP-NOW *también*
  usa CSMA/CA por debajo (hasta 31 retransmisiones observadas).
- **La consecuencia de diseño clave:** el anti-hidden-node lo aporta la **lógica de turnos** (solo el
  nodo con grant transmite *datos*), no la temporización fina. Es un **TDMA aproximado / polling de ms**,
  y eso **alcanza** para demostrar que la coordinación recupera el throughput perdido por el nodo oculto.
- **La vía a TDMA "de verdad" (µs)** pasa por `esp32-open-mac` poniendo CWmin/CWmax/AIFS=0 (como
  hMAC/FreeMAC/openwifi en otras plataformas) — **aún no implementado**; es parte de la línea futura.

## 5. La pregunta de investigación

> **¿Puede un algoritmo de coordinación de acceso al medio (round-robin/polling, familia TDMA), cuya
> *lógica* se valida en ESP32 (usando ESP-NOW como sustrato), recuperar el throughput agregado que el
> nodo oculto destruye bajo CSMA/CA — y servir de especificación de referencia para cuando exista un
> driver WiFi moderno y abierto donde integrarlo con el peering APuP de LibreMesh?**

*(Nota de planos — ver §0: en el banco ESP32 el sustrato real es **ESP-NOW**, no APuP; APuP es el peering
de producción en **LibreMesh** (WiFi/hostapd). El ESP32 valida la **lógica de coordinación**, no "ejecuta
APuP". Nunca tratar "APuP + TDMA en ESP32" como una pieza integrada.)*

Y la pregunta-corolario, que es la que da valor estratégico al esfuerzo:

> **¿Un algoritmo de referencia validado reduce el riesgo percibido de invertir en hardware WiFi de
> última generación open source, y por tanto ayuda a justificar/empujar ese hardware?**

Continúa en `02-estado-del-arte.md` (qué se sabe), `03-algoritmo-de-referencia.md` (el diseño
propuesto) y `04-hoja-de-ruta-y-preguntas-abiertas.md` (cómo abrirla).
