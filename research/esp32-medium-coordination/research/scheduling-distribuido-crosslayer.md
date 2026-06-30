# Research — Scheduling TDMA distribuido + interacción cross-layer con el ruteo

> Para `09`. Cita: autor, venue/año, URL, "cómo encamina". **[HECHO]**/**[INFER]**, confianza marcada.

## 1. Teoría fundacional: scheduling sobre canal variable
- **Tassiulas & Ephremides, IEEE TAC 1992** — **max-weight/backpressure throughput-óptimo** (región de estabilidad ⊇ toda
  otra); Lyapunov. DOI 10.1109/9.182479 · **[HECHO/alta]** → **cita ancla**: tu scheduler es Max-Weight (conjuntos de activación =
  conjuntos independientes del grafo de conflicto; pesos = colas).
- **Tassiulas, IEEE TIT 1997** — para topología que evoluciona como Markov, usar el **estado actual** alcanza todo throughput de
  cualquier política anticipativa. DOI 10.1109/18.568722 · **[HECHO/alta]** → **respuesta directa**: Max-Weight sirve para grafo
  dinámico si se observa el conflicto vigente por época → valida **re-medir el grafo por época**.
- **Neely/Modiano/Rohrs, JSAC 2005** — backpressure con canal variable estabiliza con delay acotado. **[HECHO/alta]** → garantías
  con canal estocástico + adaptación de tasa (MCS variable, el caso WiFi).
- **Neely, M&C 2010 (drift-plus-penalty)** — estabilidad + objetivo secundario (energía/equidad) sin perder estabilidad. **[HECHO/alta]**

## 1.3 Greedy Maximal Scheduling (lo que un distribuido REALMENTE puede)
- **Dimakis & Walrand, AAP 2006** — *local pooling* ⇒ LQF/GMS throughput-óptimo. **[HECHO/alta]**
- **Joo/Lin/Shroff, ToN 2009** — región de capacidad de GMS vía **factor de local-pooling**. https://staff.ie.cuhk.edu.hk/~xjlin/paper/ton09-gmm.pdf
  · **[HECHO/alta]** → **acotar** cuánto throughput perdés: eficiencia worst-case = local-pooling factor de **tu** grafo medido.
- **Lin & Shroff, ToN 2006** — scheduler imperfecto (maximal, no maximum) → degrada de forma **controlada y conocida**. **[HECHO/alta]**
  → un TDMA distribuido (imperfecto) no colapsa, degrada acotadamente.
- **Brzezinski/Zussman/Modiano, MobiCom 2006** — **particionar** la mesh en subredes donde vale local pooling → greedy distribuido
  logra 100%. http://www.mit.edu/~modiano/papers/C107.pdf · **[HECHO/alta]** → partición como técnica para volver tratable lo distribuido.

> **Síntesis eje 1 [INFER]:** diseño *state-aware* (re-medir por época, pesos actuales); esperar throughput de GMS gobernado por el
> local-pooling **empírico** de la topología medida, no la región plena.

## 2. TDMA distribuido concreto
- **Rhee/Warrier/Min/Xu (DRAND), TMC 2009 / MobiHoc 2006** — primer RAND distribuido; **O(δ)** (vecindario 2-hop), ningún par a
  ≤2 saltos comparte slot; TinyOS/Mica2. **[HECHO/alta]** → **plantilla canónica** del scheduler.
- **Ramanathan & Lloyd, ToN 1993** — link/broadcast scheduling centralizado (coloreo); el "RAND" que DRAND distribuye. **[HECHO/alta]**
- **Moscibroda & Wattenhofer, SPAA 2005 / Dist.Comp. 2008** — coloreo distribuido en modelo radio **no estructurado** (wake-up
  asíncrono, sin CD) O(Δ log n). **[HECHO/alta]** → fundamento del **bootstrap** del schedule en frío.
- **Djukic & Valaee, ToN 2009** — *Delay Aware Link Scheduling*: min-max delay es NP-completo; orden de slots por mínimo retardo.
  https://www.comm.utoronto.ca/~valaee/Publications/09-Djukic-ToN.pdf · **[HECHO/alta]** → punto de contacto con el ruteo (el orden de slots afecta delay).
- **6TiSCH — MSF (RFC 9033) + 6P (RFC 8480) + SF0 (I-D expirado)** — SF distribuida: negocia celda al unirse, añade/borra por
  uso (>75%/<25%, MAX 100), **reubica** por PDR (RELOCATE >50%); **make-before-break** ante cambio de padre. **[HECHO/alta, fetch
  directo rfc-editor]** → **modelo de referencia de rescheduling incremental adaptativo** + asignación distribuida transaccional con
  manejo de consistencia (SeqNum, repair). *(D. Dujovne co-autor de SF0 — contexto regional.)*

## 3. Elección y mantenimiento de coordinador en grafos dinámicos
- **Gerla & Tsai, Wireless Networks 1995** — clusterheads = **coordinadores que corren scheduling TDMA**, sync de trama y reúso de
  slots; Lowest-ID / Highest-Connectivity. DOI 10.1007/BF01200845 · **[HECHO/alta]** → **el precedente exacto** de "coordinador local por cluster".
- **Ephremides/Wieselthier/Baker, Proc.IEEE 1987** — Linked Cluster Algorithm; origen de Lowest-ID. **[HECHO/alta]**
- **Basagni (DCA/DMAC), I-SPAN 1999** — clustering por peso + **mantenimiento local ante link up/down** sin recómputo global. **[HECHO/alta]**
- **Chatterjee/Das/Turgut (WCA), Cluster Computing 2002** — peso combinado (grado/distancia/movilidad/batería); re-elección
  **on-demand** para reducir churn. **[HECHO/alta]** → plegar múltiples métricas en un score + re-elección bajo demanda.
- **Younis & Fahmy (HEED), TMC 2004** — re-elección **periódica** por energía residual + secundario; conectividad casi-segura, O(1)
  iteraciones. **[HECHO/alta]** → plantilla de re-elección con garantía.
- **Malpani/Welch/Vaidya, DIALM 2000** — elección de líder **auto-estabilizante asíncrona**, exactamente un líder por componente,
  tolera cambios concurrentes. **[HECHO/alta]** → teoría primaria para coordinador por partición bajo churn/merge.
- **Bettstetter & Krausser, MobiHoc 2001** *(corrección de autoría)* — estabilidad de clusters DMAC: frecuencia de re-elección vs
  movilidad. **[HECHO/alta]** → evidencia cuantitativa para elección **consciente de estabilidad** (penalizar cambiar de coordinador).
- **Yu & Chong, IEEE COMST 2005** — survey de esquemas de clusterhead (estabilidad vs overhead). **[HECHO/alta]**
- **Akyildiz/Wang/Wang, Computer Networks 2005** — arquitectura WMN (routers/gateways vs clients; jerarquía). **[HECHO/alta]**
  *(LEACH (Heinzelman, HICSS 2000) como alternativa de rotar rol, motivación energía.)*

## 4. Interacción cross-layer con la métrica de ruteo (el riesgo a domar)
- **Kawadia & Kumar, IEEE Wireless Comm. 2005** — *"cross-layer causes adaptation loops… stability becomes a paramount issue"*,
  consecuencias no previstas, "spaghetti design". http://www.ece.ucf.edu/~yuksem/teaching/nae/reading/cross-layer-design-cautionary.pdf
  · **[HECHO/alta, verbatim]** → **la** cita de riesgo: tu scheduler inyecta airtime en la métrica → lazo cuya estabilidad hay que analizar.
- **batman-adv BATMAN V (open-mesh.org)** — ELP mide el enlace local + métrica **basada en throughput** (TQ por pérdida era
  inadecuado). **[HECHO/alta vía espejo]** → **superficie de acoplamiento exacta**: el scheduler perturba el throughput que ELP mide
  y OGMv2 propaga → si asigna pocos slots, ELP ve menos throughput → ruteo evita el enlace → menos demanda… lazo a domar.
- **De Couto et al. (ETX), MobiCom 2003** — *"ETX does not attempt to route around congested links… should not suffer from the
  oscillations that plague load-adaptive metrics"*. https://pdos.csail.mit.edu/papers/grid:mobicom03/paper.pdf · **[HECHO/alta, verbatim]**
  → **el trade-off central, dicho por los diseñadores de ETX**: volver la métrica sensible al airtime reintroduce la oscilación que
  ETX evitó. Mitigar: desacoplar (métrica con histéresis/filtro temporal fuerte) **o** co-diseñar con backpressure.
- **AREDN (grey lit.)** — flapping/loops en mesh comunitaria real por fluctuación de métrica. **[HECHO/media]** → evidencia de campo.
- **Hacerlo bien:** Neely/Modiano/Rohrs 2005 (scheduling+ruteo conjunto estable con backpressure); Georgiadis/Neely/Tassiulas
  FnT 2006; Lin/Shroff/Srikant JSAC 2006 (cross-layer = descomposición NUM). **[HECHO/alta]** → el contraste positivo al acoplamiento ad-hoc.

> **Recomendación [INFER]:** o (a) **desacoplar temporalmente** — el ruteo ve un airtime **filtrado/histéresis** que cambia mucho más
> lento que el reajuste de slots — o (b) **co-diseñar** como un único problema backpressure. Kawadia-Kumar + De Couto justifican que el
> acoplamiento ingenuo es peligroso; Georgiadis-Neely-Tassiulas, el camino principiado.

## 5. Reúso espacial (STDMA)
- **Nelson & Kleinrock, IEEE TComm 1985** — origen de Spatial TDMA (reúso por separación espacial). **[HECHO/alta]**
- **Grönkvist, MobiHoc 2000 + tesis KTH 2005** — asignación basada en **interferencia (SINR)** > graph-based para reúso.
  https://www.diva-portal.org/smash/get/diva2:12179/FULLTEXT01.pdf · **[HECHO/alta]** → argumento de fidelidad para grafo **medido**.
- **Jain et al., MobiCom 2003** — conflict graph + cotas LP/independent-set. **[HECHO/alta]** → método para reportar cotas de tu grafo medido.
- **Behzad & Rubin, GLOBECOM 2007** — STDMA min-length NP-completo = coloreo con restricciones. arXiv:cs/0701001 · **[HECHO/alta]**

## 6. Prior art integrado — VEREDICTO: hueco genuino
- **Rao & Stoica (OML), MobiSys 2005** *(corrección de venue)* — overlay TDMA-software sobre DCF 802.11; **no** interference-driven
  ni con ruteo. https://people.eecs.berkeley.edu/~istoica/papers/2005/oml.pdf · **[HECHO/alta]** → ancestro más cercano a scheduler en
  user-space sobre batman-adv, pero solo 1 de los 3 pilares.
- **Djukic & Mohapatra (Soft-TDMAC), INFOCOM 2009 / TMC 2012** — MAC TDMA software multihop sobre 802.11 sin modificar, sync ~µs.
  **[HECHO/alta]** → **factibilidad de slotting TDMA distribuido preciso en hardware commodity** (argumento de viabilidad de timing).
- **Bicket et al. (Roofnet), MobiCom 2005** — mesh comunitaria omni de bajo costo, CSMA estándar, sin TDMA. **[HECHO/alta]** → baseline de despliegue.
- **Ramachandran et al., INFOCOM 2006** — interferencia **medida** (MIC) → asignación de **canal** (no slot) en mesh real. **[HECHO/alta]**
  → análogo en frecuencia; contraste channel-assignment vs TDMA-scheduling.
- **Freifunk (LibreMesh bandwidth analysis), 2019 (grey)** — confirma substrato CSMA + batman-adv, **TDMA ausente en LibreMesh**. **[HECHO/media]**
- **Propietarios (solo estado comercial):** Ubiquiti airMAX TDMA, MikroTik Nv2 — AP-céntrico PtMP, sin grafo de conflicto ni co-diseño de ruteo publicados.

> **VEREDICTO [INFER/respaldado]:** un TDMA distribuido en LibreMesh/batman-adv **conducido por grafo de conflicto medido y
> dinámico con integración de ruteo** ocupa un **hueco genuino**: cada ingrediente está precedentado, su **integración en mesh
> comunitaria de bajo costo desplegada, no**. Argumento de novedad fuerte y honesto.

> Cautelas: páginas DRAND MobiHoc'06 (190–201, dblp); mes de Lin-Shroff-Srikant JSAC'06; autoría DMAC = **Bettstetter & Krausser**
> (no Basagni&Chlamtac); OML = **MobiSys 2005** (no NSDI); grey lit. (AREDN/Freifunk/datasheets) no peer-reviewed.
