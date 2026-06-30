# Research — Estado del arte: coordinación de acceso al medio vs. nodo oculto

> Evidencia citable (papers IEEE/ACM/USENIX, RFCs, specs). Investigación 2026-06-19.
> Cada hallazgo: afirmación / fuente / confianza. Cifras textuales de las fuentes; nada inventado.

## 0. Anclas fundacionales
- (ALTA) El **terminal oculto** fue definido por **Tobagi & Kleinrock (1975)**: terminales fuera del alcance mutuo degradan el throughput de CSMA porque el carrier-sense no detecta al competidor. La solución original fue **Busy-Tone (BTMA)**, señalización fuera de banda. → IEEE Trans. Comm. 23(12):1417–1433.

## 1. Mecanismos clásicos
### 1.1 RTS/CTS y su insuficiencia en mesh multi-hop
- (ALTA) RTS/CTS = "virtual carrier sensing" vía NAV; introduce el problema del **nodo expuesto**.
- (ALTA, Xu/Gerla/Bae, GLOBECOM 2002) RTS/CTS es **ineficaz en redes ad hoc**: la potencia para corromper una recepción es mucho menor que la necesaria para entregar un paquete → el rango de interferencia (~1.78× el de tx) supera al supuesto del handshake.
- (ALTA, Ray/Carruthers/Starobinski, WCNC 2003) En multi-hop, interdependencias RTS/CTS ("false blocking") hacen que el throughput *baje* con la carga; una corrección da hasta **60%** de mejora.
> **Implicación:** en mesh comunitaria con enlaces asimétricos, RTS/CTS no resuelve el nodo oculto → hace falta coordinación explícita por software.

### 1.2 PCF y HCCA — polling centralizado (precedente directo del round-robin)
- (ALTA) **PCF**: el AP actúa de Point Coordinator; con PIFS toma el canal y sondea estaciones (CF-Poll) en un Contention-Free Period. **De facto obsoleto** (casi sin implementar, fuera de la certificación Wi-Fi Alliance).
- (ALTA, tutorial IEEE 802.11-08/1214) **HCCA** (802.11e): un Hybrid Coordinator asigna TXOP sin contención según TSPECs; "no se ha implementado a ningún nivel significativo" — el scheduler del lado AP es muy complejo, ganó EDCA.
> **Lección para el round-robin:** (a) el coordinador no conoce el estado de cola de las estaciones → conviene señalizar demanda (como Nv2/airMAX); (b) complejidad y dependencia de sync global son los puntos de falla.

### 1.3 TDMA vs CSMA/CA vs híbridos
- (ALTA) Trade-off dependiente de la carga: a **alta carga TDMA supera a CSMA/CA**; a baja/variable, CSMA/CA es más eficiente (no exige sync). TDMA da **cota de retardo determinista**; CSMA/CA no (Jaffrès-Runser, WFCS 2016).
- (ALTA) Híbridos: slots deterministas para tráfico garantizado + contención para best-effort. **PRMA** (reserva = slotted-ALOHA + TDMA) y **R-ALOHA** (reserva implícita).

### 1.4 TSCH (802.15.4e / 6TiSCH) — slotting determinista en HW barato
- (ALTA, RFC 7554) IEEE 802.15.4e (2012): TSCH = TDMA-en-tiempo + channel hopping. Timeslot típico **10 ms**; 16 canales; soluciones comerciales con **PDR >99.999%** a decenas de µA.
- (ALTA, RFC 8180/9030/8480) Configuración mínima 6TiSCH; subcapa **6top** + protocolo 6P para añadir/borrar celdas; "Tracks" deterministas.
- (ALTA) Corre en MCU baratos (OpenWSN/CC2538, Contiki-NG) con duty cycle <0.1%.
> **Por qué importa:** el slotting determinista es viable en HW barato **cuando el chipset expone control del tiempo del radio** — nativo en 802.15.4, **NO** en WiFi comercial (firmware cerrado). De ahí la necesidad de coordinar en software por encima del CSMA/CA.

### 1.5 ¿Se portó el time-slotting a WiFi?
- (ALTA) **Sí, en investigación**: **RT-WiFi** (Wei et al., RTSS 2013) = MAC TDMA sobre PHY 802.11 commodity, timing determinista hasta **6 kHz**. **Det-WiFi** (Cheng et al., WCMC 2017) extiende TDMA por software a multi-hop industrial.
- (MEDIA) **No se halló un port revisado por pares de TSCH completo sobre WiFi/ESP32** → *gap citable* que refuerza la novedad del PoC.

## 2. TDMA sobre WiFi comercial — intentos conocidos
### 2.1 Propietarios (WISP) — todos cerrados
- (ALTA, doc MikroTik) **Nv2**: TDMA propietario sobre chips Atheros; "el AP divide el tiempo en *periods* y difunde un *uplink schedule*" (**group polling**); 802.11 estándar no puede conectarse.
- (ALTA, datasheet Ubiquiti) **airMAX**: MAC TDMA que **reemplaza CSMA/CA**; el AP asigna slots → inmunidad a colisiones en PtMP exterior con estaciones que no se escuchan (nodo oculto). Cerrado (solo airOS).
- (ALTA, whitepaper) **Mimosa**: TDMA con **receptor GPS por radio** para sync, habilitando colocación/reúso de canal.
> **Patrón:** reemplazar CSMA/CA por slots del AP/master sobre chips commodity para vencer el nodo oculto exterior. Sync: master-broadcast-schedule (Nv2/airMAX) o GPS por nodo (Mimosa).

### 2.2 Open / académicos (los precedentes directos)
- (ALTA, USENIX NSDI 2007) **WiLDNet**: reemplaza CSMA/CA por MAC tipo TDMA para enlaces WiFi de larga distancia; base **SynOp** (un nodo o transmite en todos sus enlaces o recibe en todos = fases SynTx/SynRx), del MAC **2P** (Raman & Chebrolu, MobiCom 2005). Sync por **marker packets**, sin reloj global; **mejora 2–5×** el throughput.
- (ALTA, NSDR 2010) **JaldiMAC**: TDMA open para PtMP rural; **Jaldi9K** (driver open sobre Atheros commodity) + **JaldiTDMA** (scheduler en el nodo raíz). Demuestra soft-TDMA sobre la misma clase de chips que usan los propietarios.
> **Nota:** WiLDNet/2P y JaldiMAC son para PtP/PtMP de **topología conocida y estática** con antenas direccionales — el salto a mesh omnidireccional de topología parcial arbitraria es no trivial (§3.4).

### 2.3 batman-adv — por qué no coordina el medio
- (ALTA, kernel.org) batman-adv **opera solo en L2** y enruta/puentea tramas Ethernet; **no coordina el medio**. Hereda *todos* los problemas de nodo oculto del CSMA/CA subyacente.
> **Implicación para LibreMesh:** el algoritmo de coordinación opera *por debajo o en paralelo* a batman-adv (gestiona quién transmite cuándo); batman-adv sigue a cargo del ruteo.

## 3. Coordinador centralizado vs distribuido para mesh
- (ALTA) **Centralizado**: slots casi óptimos pero alto overhead de recolección de estado y recómputo ante churn. **Distribuido**: escala y se adapta, sin óptimo global.
- (ALTA) **Elección de coordinador**: Bully (mayor ID; asume conectividad completa — mal en topología parcial); **LEACH** (cluster-heads rotativos ~5%); max-degree (nodo con más vecinos).
- (ALTA) **Sync sin GPS** (linaje RBS→TPSN→FTSP): RBS ~11 µs single-hop; TPSN ~16.9 µs; **FTSP** = timestamping en MAC + flooding + regresión de skew + **elección dinámica de root** robusta. TSCH sincroniza hop-by-hop a un *time-source neighbor*.
- (ALTA) **Topología parcial (la condición que crea el nodo oculto)**: **restricción de 2 saltos** — ningún par a ≤2 saltos comparte slot; **DRAND** lo garantiza distribuido en O(δ) **sin sync**. STDMA óptimo = coloreo de grafo de conflicto (**NP-hard**) → heurísticas.
> **Síntesis de diseño:** elección de coordinador (LEACH/max-degree) → polling local con restricción de 2 saltos (precedente PCF/Nv2/airMAX) → sync FTSP-like (master difunde tiempo, robusto a fallo). Evita el coloreo NP-hard óptimo con heurística distribuida.

## 4. Métricas de evaluación
- (ALTA) **Throughput de saturación**: modelo de Bianchi (JSAC 2000) para 802.11 DCF.
- (ALTA) **Índice de Jain**: f = (Σxᵢ)² / (n·Σxᵢ²) ∈ [0,1] (1 = perfectamente justo). → Jain/Chiu/Hawe, DEC TR-301, 1984.
- (ALTA) **Performance anomaly** (Heusse, INFOCOM 2003): con tasas heterogéneas el throughput agregado cae al nivel de las estaciones lentas (reparto igual de *oportunidades*, no de *airtime*).
- (ALTA) **Airtime fairness + latencia** (Høiland-Jørgensen, ATC 2017): programar *tiempo de canal* (no oportunidades) resuelve la anomalía y acota latencia; mide fairness con índice de Jain.
> **Batería recomendada para el PoC ESP32:** (1) throughput agregado bajo nodo oculto forzado vs línea base CSMA/CA; (2) índice de Jain por flujo; (3) latencia de peor caso (cota determinista, el argumento de TDMA); (4) eficiencia de airtime / goodput.

## Caveats de verificación
PDFs no extraídos byte a byte (cifras de confianza media): Xu/Gerla/Bae (1.78×), WiLDNet (T0=1.25×T), RBS (11 µs). Mecanismos = alta confianza; cifras exactas requieren lectura directa. **Hueco citable:** sin port revisado por pares de TSCH/6TiSCH completo sobre WiFi/ESP32. Pendientes: SRMesh, token-passing inalámbrico con cita primaria fuerte.
