# Línea de investigación — Algoritmo de referencia de coordinación de acceso al medio sobre ESP32

> **Entregable ARDC:** MW-CA-RF · **Sección del informe:** §1.5 (trabajo futuro, planteado, no iniciado)
> · **Abierta:** 2026-06-19 · **Estado:** especificación + research + plan (sin implementación).
>
> **➜ Texto condensado y listo para el informe:** [`../cierre/seccion-1.5-trabajo-futuro.md`](../cierre/seccion-1.5-trabajo-futuro.md)
> — condensa `01`–`10` en la prosa de la §1.5 del entregable MW-CA-RF.

## Qué es esta carpeta
El último approach del equipo para validar la idea de coordinación de medio de MW-CA-RF: **un algoritmo
de referencia de coordinación (round-robin/polling, familia TDMA) que no puede implementarse en los
drivers WiFi comerciales cerrados, pero sí en ESP32**, que permite control de bajo nivel. La idea no es
generalizable hasta que exista un driver WiFi moderno y abierto — lo que motiva **empujar hardware WiFi
de última generación open source**, esfuerzo que un algoritmo ya validado ayuda a justificar.

## La aclaración que ordena todo: APuP ≠ TDMA
El equipo llama informalmente "algoritmo APuP" a este trabajo, pero **APuP y TDMA viven en capas
distintas**:
- **APuP** (plano de *peering*, ya en OpenWrt vía PR #1134/#1200) decide **quiénes son vecinos**. No
  coordina el medio: corre sobre el CSMA/CA del chip y por eso no resuelve el nodo oculto.
- **El algoritmo de coordinación** (plano de *acceso al medio*, familia TDMA) decide **quién transmite
  cuándo**. Es lo único que ataca el nodo oculto, y es lo que esta línea diseña.

Se tocan porque la estructura AP↔cliente que crea APuP sirve de **canal de señalización** para repartir
turnos (como hacían PCF/HCCA, Nv2, airMAX). Desarrollo completo en `01-planteo-del-problema.md §0`.

> **Ancla:** APuP decide quiénes son vecinos; el algoritmo decide quién transmite cuándo.

## Mapa de lectura
1. **[`01-planteo-del-problema.md`](01-planteo-del-problema.md)** — desambiguación APuP/TDMA, el problema,
   la evidencia del corpus, por qué ESP32, la pregunta de investigación.
2. **[`02-estado-del-arte.md`](02-estado-del-arte.md)** — síntesis: por qué falla CSMA/CA, precedentes
   (PCF/HCCA, Nv2/airMAX/Mimosa, WiLDNet/JaldiMAC, TSCH), límites de ESP32, panorama de hardware abierto.
3. **[`03-algoritmo-de-referencia.md`](03-algoritmo-de-referencia.md)** — el diseño propuesto: round-robin
   con polling (validado en ESP32 sobre ESP-NOW), coordinador rotable, sync FTSP-like, híbrido con CSMA.
   APuP es el peering de producción en LibreMesh, no parte del banco ESP32.
4. **[`07-modelado-del-grafo-de-conflicto.md`](07-modelado-del-grafo-de-conflicto.md)** — **la capa
   fundacional** (antes de asignar slots): modelar el problema como **grafo de conflicto dirigido, medido y
   variable en el tiempo** G(t) — porque el mapa de nodos ocultos real es asimétrico y cambia con
   clima/follaje/LoS. Pipeline medir→construir→detectar→colorear→adaptar. *(Leer junto con `03 §2`.)*
5. **[`08-shared-state-bus-data-t.md`](08-shared-state-bus-data-t.md)** — **la arquitectura unificadora**:
   shared-state como **bus único de `data(t)`** que sirve a la vez al operador IA (Data-Collection,
   entregable 3) y al **G(t) del scheduler TDMA** (MW-CA-RF, entregable 1). Desdoblamiento plano-modelo
   lento / plano-ejecución rápido; el bug TTL se vuelve load-bearing; frescura por AoI/PBS; novedad+riesgo.
6. **[`09-agenda-de-investigacion-fundamentada.md`](09-agenda-de-investigacion-fundamentada.md)** — **todos
   los ángulos abiertos con su bibliografía** (medición, cadencia temporal, estabilidad/anti-thrashing,
   scheduling/teoría, coordinador, cross-layer con batman-adv, cold-start) y el hilo argumental del desarrollo.
7. **[`10-objetivo-de-futuro-y-scope.md`](10-objetivo-de-futuro-y-scope.md)** — **¿avanzada o curiosidad? ¿hasta
   dónde?** La escalera de plataformas (ESP32→ath9k→**OpenWiFi**→baseband WiFi 6/7 abierto sobre RFSoC); el giro
   clave: el alto rendimiento abierto **no está bloqueado por el silicio (RFSoC sobra) sino por el PHY 802.11ax/be
   inexistente**; el objetivo que justifica el esfuerzo (prueba-de-necesidad, patrón ORCA/O-RAN); scope explícito
   (ni académico ni limitado a ath9k) y framing de TRL honesto.
8. **[`04-hoja-de-ruta-y-preguntas-abiertas.md`](04-hoja-de-ruta-y-preguntas-abiertas.md)** — fases F0–F5,
   métricas (incl. baseline RTS/CTS), preguntas abiertas, argumento estratégico, encaje en el informe.
9. **[`05-viabilidad-y-asegurabilidad.md`](05-viabilidad-y-asegurabilidad.md)** — **de-risking antes de
   comprometer con ARDC**: qué se puede asegurar 🟢 / probable 🟡 / no asegurable 🔴; las dos correcciones de
   rumbo (APuP+TDMA no es una pieza integrada; bifurcación 802.15.4); banderas para el informe. **Leer
   antes de prometer entregables.**
10. **[`06-revision-externa.md`](06-revision-externa.md)** — segunda opinión técnica (ia-bridge) antes de
   comprometer con ARDC: valida el esqueleto, pero exige (a) baseline **RTS/CTS** en toda medición, (b)
   definir "adaptativo" o degradarlo, (c) decir sin eufemismos que el PoC ESP32 **no porta a producción**.
   Abre la **decisión de plataforma ath9k vs ESP32** (path a producción vs velocidad de PoC).
11. **[`research/`](research/)** — evidencia citable con fuentes y niveles de confianza:
   - `estado-del-arte-mac.md` — coordinación de medio vs nodo oculto (papers/RFCs).
   - `esp32-bajo-nivel.md` · `esp-now.md` — qué permite y qué no el WiFi del ESP32; ESP-NOW como señalización.
   - `hardware-wifi-open.md` · `viabilidad-esp32-tdma.md` · `causalidad-cierre-firmware.md` · `regulacion-fcc-firmware.md`.
   - `grafo-de-conflicto.md` — fundamentación del modelado a grafo (Jain, Gupta-Kumar, Padhye/Reis, ITU-R…).
   - `estimacion-grafo-medicion.md` — sondeo activo O(N) / pasivo CCA / RSSI vs CSI / prior art MCU (Jia nRF52).
   - `dinamica-temporal-estabilidad.md` — escalas del canal (Roofnet/WiLD/ITU-R), histéresis (Babel/BGP), TVG, Tassiulas, cold-start.
   - `scheduling-distribuido-crosslayer.md` — backpressure, GMS, 6TiSCH 6P/MSF, clusterhead, cross-layer (Kawadia-Kumar/ETX).
   - `shared-state-data-t.md` — CRDT/gossip, Onix/4D/KDN, AoI/PBS, in-band self-stabilizing, frescura diferencial.
   - `plataforma-wifi-abierta.md` — OpenWiFi (low-MAC FPGA, SIFS 10µs, RT-WiFi), muro de banda AD9361, RFSoC, mt76 blob, GNU Radio.
   - `estrategia-scope-hardware-abierto.md` — ORCA/O-RAN, reference implementation (NIST/RFC 7942), LibreRouter, NLnet/NGI, TRL.

## Tesis en una frase
> Un esquema de coordinación de turnos a granularidad de **ms**, cuya **lógica** se valida en ESP32 (sobre
> ESP-NOW) y opera por debajo de batman-adv, debería recuperar el throughput que el nodo oculto destruye —
> y servir de **especificación de referencia** para integrarlo con el peering APuP de LibreMesh cuando
> exista una MAC WiFi abierta (OpenWiFi hoy; `esp32-open-mac` y hardware de última generación open source,
> mañana). **El PoC ESP32 valida la lógica, no porta a producción WiFi** (la MAC moderna está en blob
> cerrado); APuP y el algoritmo son dos planos en dos plataformas — nunca "APuP + TDMA en ESP32".

## Honestidad de alcance
Hoy esto es **especificación + research + plan**, no implementación. ESP32 con API pública solo permite
coordinación de **ms** (no TDMA de µs: el CSMA/CA está en HW/blob). El camino a µs pasa por abrir la MAC
(`esp32-open-mac`/OpenWiFi). El informe debe presentarlo como **§1.5 trabajo futuro**, sin afirmar
mediciones inexistentes.
