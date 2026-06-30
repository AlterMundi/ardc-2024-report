# 04 — Hoja de ruta y preguntas abiertas

> Esto es lo que convierte un planteo en una **línea de investigación abierta**: qué hacer, en qué orden,
> cómo medirlo, qué no sabemos todavía, y cómo conecta con el empuje de hardware abierto.

## 1. Fases de investigación

| Fase | Objetivo | Entregable verificable | Banco |
|------|----------|------------------------|-------|
| **F0 — Encuadre** | Fijar terminología (APuP≠TDMA) y la pregunta de investigación | Este apartado (hecho) | — |
| **F1 — PoC mínimo** | ¿La coordinación recupera el throughput que el nodo oculto destruye? | 2 ESP32 + 1 receptor; throughput coordinado vs CSMA/CA en el escenario `hidden-node.md` | mesa de lab |
| **F2 — Round-robin con polling** | Coordinador + N=3–5; skip + ponderación por demanda | código + Jain + latencia | lab + QEMU/vwifi (Casanueva-Riba) |
| **F2.5 — Estimación del grafo de conflicto** | **Medir** G (dirigido) por sondeo activo O(N) (Reis) + pasivo, bajo tx concurrente (Niculescu); detectar nodo oculto formalmente | matriz de conflicto medida vs proxy de 2 saltos | HIL multi-nodo (`07`) |
| **F3 — Topología parcial + G(t) dinámico** | Scheduler sobre grafo **medido y variable en el tiempo** (no proxy de saltos); re-estimación con histéresis anti-thrashing | scheduler distribuido sobre G(t) | HIL + campo (`07 §5`) |
| **F4 — Multi-clúster** | Varios coordinadores + bordes | mesh >10 nodos | campo (MonteNet?) |
| **F5 — Port a hardware abierto** | TDMA de µs sobre Low-MAC abierta | implementación OpenWiFi y/o `esp32-open-mac` | OpenWiFi (Zynq) |

> F1 es el experimento que **prueba o refuta la tesis** y es barato. Debería ser el primer paso real
> después de validar el plan con el equipo.

## 2. Métricas y protocolo de medición
- **Tres curvas obligatorias en el mismo testbed (CRÍTICO, de la revisión externa):** CSMA puro vs
  **RTS/CTS** (`iw dev wlanX set rts <umbral>`, cero desarrollo, es la respuesta de manual al nodo oculto)
  vs nuestro TDMA. **El claim defendible es "TDMA supera *también* a RTS/CTS", no solo a CSMA crudo.** Sin
  la curva del medio, no se firma el entregable: el primer reflejo de un revisor de ARDC es "¿por qué no
  activan RTS/CTS?", y hay que responderlo con números (RTS/CTS: overhead de handshake a tasas altas,
  virtual-CS limitado, sigue siendo CSMA reactivo, falla en multi-hop — Xu/Gerla/Bae).
- **Throughput agregado** bajo nodo oculto forzado (bajar tx-power para romper la asociación entre
  extremos, como en `hidden-node.md`). Es la métrica madre.
- **Overhead de coordinación neto**: airtime de grants+sync como % del superframe; throughput con 2 y 3 Tx.
- **Jitter de entrega del grant bajo carga** (no el sync FTSP, que lo cubre el guard): si el grant llega.
- **Cold-start**: tiempo a establecer el primer superframe partiendo de canal saturado.
- **Fairness**: índice de Jain por flujo.
- **Latencia de peor caso** (el argumento de venta de TDMA: cota determinista).
- **Eficiencia de airtime / goodput** (cuánto se va en coordinación vs payload).
- **Convergencia**: tiempo de formación del schedule ante alta/baja de nodos (churn).
- Reproducibilidad: todo sobre el testbed HIL/CI de Casanueva-Riba (Labgrid + pytest + QEMU/vwifi).

## 3. Preguntas abiertas (la agenda de investigación)
1. **¿Cuánta penalización de overhead tolera el esquema antes de no compensar?** El trade-off TDMA/CSMA
   depende de la carga (Jaffrès-Runser). ¿Umbral de activación automática?
2. **Granularidad real alcanzable en ESP32** con WiFi activo en core 0: ¿slots de 5 ms? ¿2 ms? Medir
   jitter real, no asumir.
3. **¿ESP-NOW aguanta como canal de grants** con N alto y enlaces degradados (hasta 31 retransmisiones)?
   ¿O conviene un canal de control sobre tramas crudas con tasa fija?
4. **Coordinación inter-clúster**: ¿bandas de guarda? ¿negociación de offsets entre coordinadores vecinos?
5. **Interacción con batman-adv**: ¿la métrica de ruteo (TQ/throughput) debe conocer el schedule para no
   pelearse con él?
6. **¿Sirve el "round-robin de autenticación" literal** (usar el handshake de asociación 802.11 como
   turno) o es mejor desacoplar la señalización del enlace? El research no halló precedente del primero.
7. **Camino a µs**: ¿qué falta exactamente en `esp32-open-mac` para poner CWmin/CWmax=0? ¿Vale colaborar
   con ese proyecto (NLnet)?
8. **Estimación del grafo de conflicto (la agenda dura, `07`):** ¿sondeo activo O(N) (Reis) vs inferencia
   pasiva por pérdidas/CCA — cuál es viable en ESP32 y a qué costo O(n²)? ¿Modelo de protocolo (barato,
   impreciso) o aproximación SINR? ¿Cómo se mide bajo *transmisión concurrente* (Niculescu) sin romper la
   propia red? ¿Cada cuánto re-estimar vs la dinámica real del canal (follaje/lluvia, ITU-R P.833/P.530)?
   ¿Qué histéresis/ventana de confianza evita el thrashing del schedule? **No hay prior art en ESP32-class.**

## 4. El argumento estratégico: por qué esto empuja hardware abierto
La cadena lógica del informe queda así de cerrada:

1. El nodo oculto exige coordinación de medio (no CSMA/CA). *(medido, `hidden-node.md`)*
2. Esa coordinación es **imposible in-driver** en WiFi comercial moderno (Low-MAC en blob desde
   802.11ac). *(`research/hardware-wifi-open.md`)*
3. **No existe** WiFi 6/7 open (ni SDR ni FPGA) a 2026; lo más abierto topa en 802.11n/ac.
4. Luego: hace falta **empujar hardware WiFi de última generación open source**.
5. Para justificar esa inversión (cara, de nicho), **un algoritmo de coordinación ya validado** baja el
   riesgo. **Acotación honesta (revisión externa H6):** el PoC ESP32 valida la **lógica de turnos a ms**,
   NO el TDMA-µs / control-CSMA que la MAC WiFi moderna necesitaría — que es la parte dura. La analogía
   srsRAN/OpenAirInterface→O-RAN es **imperfecta**: srsRAN validó una PHY/MAC 4G/5G real sobre SDR; el
   ESP32-ms valida una *aproximación del concepto*. Frase defendible: *"valida el concepto algorítmico; el
   timing fino se de-riesga aparte con 802.15.4/OpenWiFi"* — no "valida el TDMA que correría en producción".
6. ESP32 (`esp32-open-mac`) y OpenWiFi son los dos bancos donde madurar la referencia mientras tanto.
   **El PoC ESP32 no porta a producción WiFi** (la MAC moderna está en blob cerrado): es validación
   algorítmica, no path a producción — decirlo sin eufemismos (no solo en `05`/`06`).

> El algoritmo no es solo un entregable técnico: es la **pieza de evidencia** que vuelve defendible el
> pedido de hardware WiFi abierto — **siempre que se presente como validación de la lógica, no como
> solución WiFi de producción**.

## 5. Cómo encaja en el informe ARDC (no inflar el entregable)
- Va en **§1.5 (trabajo futuro, planteado, no iniciado)** del informe — coherente con el veredicto
  "pivote ordenado" de MW-CA-RF. **No** afirmar que el algoritmo está implementado o medido.
- Lo entregable *hoy* es: (a) esta **especificación de referencia** con su fundamentación; (b) la
  **desambiguación conceptual** APuP/TDMA (valiosa por sí misma); (c) el **research citado**; (d) un
  **plan de validación** sobre un testbed que ya existe en preparación.
- Mantener la separación: APuP + fix #1200 (logrado, §1.2), shared-state Rust (logrado, §1.4),
  testbed Casanueva-Riba (en preparación, §1.1), **este algoritmo (futuro, §1.5)**.

## 6. Próximos pasos inmediatos (no de laboratorio)
- [ ] Validar la desambiguación APuP≠TDMA con G10h4ck y javierbrk (¿coinciden con el encuadre?).
- [ ] Confirmar que el "round-robin de autenticación" de las notas == polling sobre APuP (no otra cosa).
- [ ] Decidir banco de F1: mesa de lab con ESP32 propios vs QEMU/vwifi del testbed UNC.
- [ ] **Decidir plataforma WiFi load-bearing: ath9k (path a producción, soft-TDMA con precedente real) vs
      ESP32 (velocidad de PoC, cero path a producción)** — cambia qué se promete (ver `06`).
- [ ] Evaluar contacto con `esp32-open-mac` (NLnet) para el camino a µs.

### Checklist de citación pre-firma (no citar en §1.5 sin verificar — revisión externa H10)
- [ ] **"93% de reducción de deadlines"** (OpenWiFi-TSN) y **µs-sync de Soft-TDMAC**: marcados "verificar
      antes de citar" en `research/hardware-wifi-open.md` — leer el paper o NO citar la cifra.
- [ ] **Límites de peers ESP-NOW** (`research/esp-now.md`: ≤17/≤10/≤6): son version-dependent y están
      confusos — fijar la cifra contra la versión de IDF que se use.
- [ ] **Performance anomaly de Heusse** (`02 §6`): puede NO aplicar al PoC (ESP-NOW corre a 1 Mbps fijo =
      tasa homogénea) — no usarla como métrica si la tasa es uniforme.
- [ ] Cerrar el resto de cifras de confianza media del `research/` antes de citar textual.
- [ ] **Confirmar el texto del grant** ("improving wireless coordination") no prometa más de lo entregable
      (con Nico/Javier) — ver bandera 2 de `05`.
