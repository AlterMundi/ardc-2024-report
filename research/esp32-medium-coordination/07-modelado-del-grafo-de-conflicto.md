# 07 — Modelado del grafo de conflicto (la capa fundacional)

> **Origen:** observación de Fede (2026-06-19): en topologías reales multi-nodo el mapa de nodos ocultos
> es complejo y **cambiante en el tiempo** (clima, follaje, línea de vista); ¿hay un modelado a grafo
> formal *antes* de asignar slots? **Respuesta: debe haberlo, y es esta capa.** Evidencia citable en
> `research/grafo-de-conflicto.md`. Corrige el posicionamiento previo (el grafo estaba degradado a un
> componente menor de F3-F4 en `03`, y derivado del conteo de saltos — una aproximación).

## 0. La tesis de este documento
**Asignar slots es *downstream* de un problema de modelado.** El orden correcto es:

```
1. MEDIR        audibilidad/interferencia entre nodos (sondeo activo + estadística pasiva)
2. CONSTRUIR    el grafo de conflicto DIRIGIDO y PONDERADO  G(t)
3. DETECTAR     los patrones de nodo oculto (A→B←C sin arista A–C) — formalmente
4. COLOREAR     asignar slots sobre G(t) (STDMA / coloreo) — RECIÉN AQUÍ
5. ADAPTAR      re-estimar y re-colorear incrementalmente cuando G(t) cambia, con
                histéresis/confianza para no oscilar (thrashing)
```

Los documentos `03`/`04` arrancaban en el paso 4. Los pasos 1-2-5 **son** lo que significa "adaptativo e
inteligente" (más allá de la ponderación por demanda que define `03 §C`), y son el **verdadero contenido
de investigación** de la línea: el slotting es ingeniería conocida; el modelado dinámico del conflicto en
mesh comunitaria barata es donde está el aporte original.

## 1. El grafo de conflicto como objeto formal
- **Definición canónica (Jain et al., MobiCom 2003):** el *conflict graph* tiene **vértices = ENLACES**
  (no nodos) y una arista entre dos enlaces si **no pueden estar activos simultáneamente**. Los *conjuntos
  independientes* son conjuntos de transmisiones concurrentes viables; los *cliques* dan cotas de
  throughput. Hallar el schedule óptimo es **NP-hard**.
- **El nodo oculto es un patrón estructural, no un caso aislado:** A→B←C (A y C alcanzan/interfieren en el
  receptor B) **sin arista A–C** (no se sensan). Formalmente, *conflicto ≠ conectividad*: el grafo de
  conflicto no se deduce del grafo de enlaces/ruteo.
- **Consecuencia:** "vecino a 2 saltos" (lo que usa DRAND y lo que `03 §2A` escribía) es a lo sumo un
  *proxy* del conflicto, no el conflicto. En topología real hay que construir el grafo de verdad.

## 2. Modelo de protocolo vs modelo físico (SINR) — por qué el "2 saltos" falla
- **Modelo de protocolo (Gupta-Kumar 2000):** binario y por pares — una transmisión es exitosa si ningún
  interferente está dentro de un radio `(1+Δ)` de la distancia tx-rx. Es lo simple (DRAND, 2 saltos).
- **Modelo físico / SINR:** la interferencia es una **suma acumulativa** sobre *todos* los emisores; el
  éxito depende de SINR ≥ umbral. Más preciso, más caro.
- **Lo que el 2-saltos no captura (Iyer/Rosenberg/Karnik, 2009):** los modelos binarios **sobreestiman**
  el throughput, y la **interferencia acumulativa de muchos emisores débiles** (cada uno fuera del "rango"
  pairwise) puede romper una recepción aunque ningún interferente individual esté a 2 saltos. El nodo
  oculto real no respeta el conteo de saltos.

## 3. Por qué hay que MEDIRLO, no asumirlo
- **La interferencia no es binaria (Padhye et al., IMC 2005):** se mide como un *Link Interference Ratio*
  continuo. El sondeo exhaustivo de pares es caro: **>1100 horas** para caracterizar 22 nodos a fuerza
  bruta. El costo es **O(n²)** (todos los pares ordenados).
- **Reducción práctica (Reis et al., SIGCOMM 2006):** se puede estimar los **N² parámetros con O(N)
  experimentos** activos (cada nodo transmite por turnos, los demás miden entrega), con error medido de
  ~9-11% vs ~24-31% de un SINR naïve. Es la receta de sondeo activo viable.
- **Inferencia pasiva (complementaria):** CMAP (NSDI 2008) y métodos de *busy-time*/CCA (Inria 2019)
  infieren conflicto de la estadística de pérdidas/ocupación sin sondeo dedicado.
- **Cuidado (Niculescu, IMC 2007):** el sondeo solo-broadcast **pierde los hidden terminals remotos** —
  hay que medir entrega *bajo transmisiones concurrentes*, no solo audibilidad en silencio.

## 4. El grafo es DIRIGIDO (asimétrico)
- **La vista del canal es asimétrica (Garetto & Knightly, IEEE/ACM ToN 2008):** hay *starvation* con
  factores ≥10× por asimetría de carrier-sense. A puede sensar a C sin que C sense a A.
- **Padhye mide P_AB ≠ P_BA** por par ordenado: la relación de conflicto tiene dirección.
- **Roofnet (SIGCOMM 2004):** "no hay umbral claro entre en-rango y fuera-de-rango" → el modelo de disco
  simétrico es falso. **El grafo de conflicto debe ser dirigido y ponderado**, y el nodo oculto es,
  precisamente, una **asimetría** en él. Un modelo no-dirigido lo pierde.

## 5. El grafo es VARIABLE EN EL TIEMPO: G(t), no G (el punto central de Fede)
- **Follaje (ITU-R P.833-10):** atenuación por vegetación **estacional**, ~**20% mayor con hoja** a 1 GHz;
  el follaje moviéndose con viento modula la pérdida.
- **Lluvia y multipath (ITU-R P.530):** rain fade y desvanecimiento por multitrayecto varían el enlace en
  escalas de minutos a estacionales.
- **Ducting troposférico** y **bursts de pérdida en enlaces WiLD de larga distancia (Sheth et al.,
  INFOCOM 2007):** los enlaces exteriores comunitarios son intrínsecamente no-estacionarios.
- **Formalismo:** redes temporales (Holme & Saramäki, *Physics Reports* 2012; Time-Varying Graphs de
  Casteigts) — el conflicto es un grafo cuyas aristas aparecen/desaparecen. El scheduler se alimenta de
  **G(t) re-estimado**, no de un G fijo.
- **Estabilidad (anti-thrashing):** re-colorear ante cada fluctuación oscila; hace falta **histéresis /
  ventanas de confianza** (patrón conocido en métricas de ruteo, p.ej. Babel / PAM 2007) para cambiar el
  schedule solo ante cambios persistentes, no ruido.

## 6. Scheduling sobre el grafo (el paso 4, ahora bien fundado)
- **STDMA = coloreo del grafo de conflicto, NP-completo (Brar/Blough/Santi).** Los schedulers clásicos
  "asumen conocimiento limitado de la interferencia"; **DRAND asume 2 saltos, no mide**. *Esa es
  exactamente la brecha que ataca este diseño:* schedule sobre un grafo **medido** y **dinámico**, no
  sobre un proxy de saltos estático.
- Coloreo óptimo intratable → heurísticas distribuidas + re-coloreo incremental ante cambios de G(t).

## 7. Implicaciones (honestas) para el dictamen y el roadmap
- **El PoC F1 (estrella canónica) tiene un grafo de conflicto trivial, conocido y estático** — por eso es
  asegurable: **esquiva** a propósito el problema del modelado. F1 valida el *mecanismo de slotting*, no el
  problema del grafo. Hay que decirlo sin eufemismos.
- **Estimar G(t) dinámico, asimétrico, en campo y sobre ESP32 es el problema duro** — y es
  **🟡/🔴 (no asegurable)**, ver `05`. El hardware tiene las primitivas (RSSI por paquete, CSI de 64
  subportadoras, noise floor — confirmado en ESP-IDF), **pero NO existe prior art** que una *medición de
  grafo de conflicto + TDMA interference-aware* en ESP32-class (ESP-WIFI-MESH/painlessMesh corren CSMA/CA
  sin scheduling de interferencia). El costo O(n²) del sondeo en un MCU **no está cuantificado**. Vacío de
  prior art = a la vez **riesgo** y **novedad/contribución**.
- **Brecha de plataforma a clarificar (research):** el conflict-graph medido tiene prior art en routers/APs
  802.11 (ath9k) y en otro PHY (TSCH/802.15.4), **no** en ESP32. Refuerza la decisión ath9k-vs-ESP32 de
  `05`/`06`.
- **Esto es el corazón del aporte:** un revisor de ARDC verá el slotting como ingeniería conocida; el
  modelado dinámico del conflicto medido en mesh comunitaria barata es lo que justifica investigación.

## 8. Qué cambia en los demás documentos
- `03`: el grafo de conflicto deja de ser "componente A de F3-F4" y pasa a ser **capa fundacional** (pasos
  1-2-5); el coloreo (paso 4) es downstream. F1 mantiene grafo trivial explícito.
- `04`: nueva fase de **estimación del grafo de conflicto**; preguntas abiertas (sondeo activo O(N) vs
  pasivo en ESP32; SINR vs protocolo; histéresis anti-thrashing; periodo de re-estimación vs dinámica del
  canal).
- `05`: la estimación dinámica del grafo se clasifica **🟡/🔴**; el mecanismo de slotting sobre grafo
  *conocido* sigue 🟢.
