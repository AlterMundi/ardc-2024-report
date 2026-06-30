# 06 — Revisión externa (second-opinion) y acciones

> Revisión técnica escéptica solicitada antes de comprometer el entregable con ARDC.
> Reviewer: Claude/Opus vía ia-bridge (`/second-opinion`), 2026-06-19. Log original:
> `~/.bridge-ai/opinions/20260619-185149-Informe_ARDC_2026-claude-second-opinion.md`.
> **Veredicto global:** el esqueleto es técnicamente sólido; el riesgo no está en lo que el dictamen niega
> (🔴 correctos) sino en lo que el 🟢 y la narrativa **afirman de más**. Cerrando 3 cosas pasa de
> "atacable" a "defendible".

## Veredicto por punto
- **P1 (clasificación / circularidad del control): DE ACUERDO con matiz fuerte.** La circularidad **no es
  fatal** por una razón geométrica clave: en la topología canónica (A y C ocultos entre sí, ambos oídos por
  B=Rx), **si el coordinador es B, A y C sí oyen los grants** — el control plane se resuelve exactamente en
  el caso que importa. Pero "asegurable" es demasiado para un *número*, y el 🟢 esconde dos supuestos:
  (i) coordinador en rango de todos los contendientes (cierto en estrella, falso en multi-hop general);
  (ii) cold-start (antes de fijar turnos el control contiende). El sync FTSP (decenas de µs vs guard de
  10 ms) **no** es el problema; el problema es la **entrega del grant bajo carga en topologías no-estrella**.
- **P2 (APuP≠TDMA): DE ACUERDO — es el gap arquitectónico #1.** El TDMA del ESP32 **no porta** al stack
  donde se desplegaría LibreMesh (WiFi/ath10k+/Linux, upper-MAC en blob). "APuP+TDMA da capacidad a
  LibreMesh" es un puente que el entregable **no construye**. ESP32 = validación algorítmica, no path a
  producción. Decirlo sin eufemismos.
- **P3 (causalidad MAC): MATIZ — el framing alternativo es el correcto, no sobre-corrección.** El lower-MAC
  timing-crítico (ACK/IFS/CCA) siempre estuvo en HW; lo que ath9k exponía eran *registros de control*
  (CWmin/CWmax, AIFS, TXOP, colas) y la libertad de correr schedulers de software. ath10k movió upper-MAC +
  planificador a firmware y **dejó de exponer esos knobs**. Usar exactamente ese wording.
- **P4 (802.15.4): OBJECIÓN parcial — legítimo pero en cuarentena.** Es casi tautológico ("validamos TDMA
  en un radio diseñado para TDMA = TSCH") y **no es el radio del problema**. Va como *sanity-check de
  correctitud algorítmica*, etiquetado, fuera del path del claim WiFi, y **droppable si aprieta el budget**.
- **P5: varios agujeros** (abajo).

## Problemas priorizados (los que un revisor de ARDC tocará primero)
1. **[CRÍTICO] Falta baseline RTS/CTS.** Es la respuesta de manual al nodo oculto y cuesta **cero
   desarrollo** (`iw dev wlanX set rts <umbral>`). El primer reflejo del revisor: *"¿por qué no activan
   RTS/CTS?"*. Sin mostrar por qué RTS/CTS es insuficiente (overhead a tasas altas, virtual-CS limitado,
   sigue siendo CSMA reactivo, falla en multi-hop — Xu/Gerla/Bae), la propuesta **parece ingenua**. TDMA
   debe posicionarse *contra RTS/CTS como incumbente, con números*. → Acción en `04 §2`.
2. **[CRÍTICO] El entregable ESP32 no alcanza LibreMesh** (gap P2). Asegurás un PoC algorítmico; no asegurás
   producción WiFi. No prometer "solución para LibreMesh". → Ya en `05`; reforzado.
3. **[ALTO] "Adaptativo" sin definir.** Round-robin puro **no es adaptativo**. Definirlo (slots ponderados
   por demanda/ocupación de cola) o degradarlo a "round-robin rotativo". → Acción en `03 §C`.
4. **[ALTO] Interacción con batman-adv.** Los OGMs broadcast caen en la ventana best-effort y la métrica TQ
   asume medio CSMA broadcast; meter TDMA debajo puede degradar métrica/convergencia. Medir o acotar el
   claim al data-plane. → Ya en `04 §3`; elevado.
5. **[MEDIO] Mismo radio control+datos** (refuerza, no invalida, la circularidad) y **generalización del
   testbed** (el número sale de 2-3 nodos; multi-hop/movilidad/churn sin cuantificar). Acotar el claim.

## Alternativa estratégica que plantea el reviewer (decisión para el equipo)
Si el objetivo real es *capacidad de LibreMesh en producción*, el camino honesto **no es ESP32** sino:
- **ath9k** (802.11n) — el **último MAC WiFi controlable desde el host**, con precedentes reales de
  soft-TDMA por manipulación de colas/CW (Leffler 2009 sobre Atheros con granularidad 1 µs; hMAC sobre
  ath9k). Tradeoff: hardware viejo, control aún grueso, pero **es WiFi y tiene path**.
- **802.15.4/TSCH** como red de coordinación separada — el radio correcto para slotting, pero es *otra* red,
  no resuelve el WiFi.
- **ESP32/ESP-NOW** gana en *velocidad de PoC y narrativa* (mismo SoC barato, demo vistosa) a costa de
  **cero path a producción**. Defendible **solo** si se vende como "prueba de concepto del algoritmo", no
  como "solución para LibreMesh".

> **Pregunta abierta para el equipo (Nico/Javier/Gioaccino):** ¿el entregable ARDC apunta a (a) PoC
> algorítmico rápido y vistoso (ESP32) — narrativa de de-riesgo de hardware abierto; o (b) demostración con
> path a producción (ath9k) — más creíble pero menos "futuro"? No son excluyentes: ath9k podría ser el
> banco WiFi load-bearing y ESP32 el demo accesible. **Esto cambia qué se promete.**

## Experimentos a correr ANTES de firmar (del reviewer)
- Las **tres curvas** en el mismo testbed: CSMA puro / `iw … set rts <thr>` / TDMA. Sin la del medio, no firmar.
- **Overhead de coordinación neto**: airtime de grants+sync como % del superframe; throughput con 2 y 3 Tx.
- **Jitter de entrega del grant bajo carga** (no el sync FTSP — ese lo cubre el guard).
- **Cold-start**: tiempo a establecer el primer superframe partiendo de canal saturado.

## Lo que el reviewer confirmó como sólido
Las líneas 🔴 del dictamen (TDMA µs sobre WiFi, apagar CSMA/CWmin, depender de esp32-open-mac, portabilidad
directa) son **correctas y prudentes**. La separación de planos (P2) y la corrección del framing MAC (P3)
son de **alta confianza**. La resolución geométrica de la circularidad (coordinador=Rx) **fortalece** el 🟢
en la topología canónica.
