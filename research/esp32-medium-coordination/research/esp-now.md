# Research — ESP-NOW como canal de señalización/control

> Evidencia citable. Investigación 2026-06-19. **Alta** = doc Espressif / paper revisado · **Media** = repo/medición de foro · **Baja** = foro sin método.

## 1. Qué es a nivel protocolo
- (ALTA) Protocolo Wi-Fi **connectionless** (sin asociación/handshake); transporta datos dentro de una **vendor-specific action frame** 802.11. OUI `0x18fe34`, Type=4. → ESP-IDF ESP-NOW; ESP-FAQ.
- (ALTA) **MTU**: v1.0 = **250 bytes** (`ESP_NOW_MAX_DATA_LEN`); **v2.0 = 1470 bytes** (`..._V2`, fragmenta). *Verificar versión de IDF.*
- (ALTA) **Mismo canal obligatorio**: el peer debe estar en el canal de STA/SoftAP; si el nodo está asociado a un AP de infraestructura queda atado a ese canal.
- (ALTA) Coexiste con STA/AP simultáneos pero **comparte canal y airtime** con el WiFi normal.

## 2. Determinismo y latencia
- (ALTA) **ACK a nivel MAC sí**: `ESP_NOW_SEND_SUCCESS` significa recepción confirmada por el ACK 802.11.
- (ALTA, WONS 2025) Latencia one-way en lab: **1059 µs** (1 byte), **1869 µs** (100 bytes). Jitter intrínseco bajo en buen enlace (σ≈108 µs), **pero crece fuerte con la distancia** por retransmisiones.
- (MEDIA, micropython #11757) RTT <1 ms para ~95% en buen enlace; en campo 9–63 ms, pérdidas 5.6–21%.
- (MEDIA, arXiv 2507.16594) Throughput ~15–35 Mbps monocliente.

## 3. Escalabilidad
- (ALTA) **Máx 20 peers** totales. Cifrados: ≤17 configurable (≤10 en Station, ≤6 en SoftAP). Cifrado **no** aplica a multicast/broadcast.
- (ALTA) **Broadcast no cifrado: sin límite de receptores.** → para >20 nodos, el grant round-robin debe ir por **broadcast**, no unicast emparejado.

## 4. Sincronización temporal / coordinación (precedentes)
- (MEDIA) **Sync-ESP-NOW** (2023): broadcast en intervalos predefinidos = slotting tipo TDMA sobre ESP-NOW.
- (MEDIA) **ESP-NOW + FTSP** (2023): time-sync de precisión sobre WSN de recursos limitados.
- (ALTA, WONS 2025) Nodo de sync que hace broadcast de su tiempo lógico (RBS modificado) → ESP-NOW broadcast sirve como canal de time-sync funcional.

## 5. Limitaciones para tiempo real (crítico)
- (ALTA, WONS 2025) **ESP-NOW usa el CSMA/CA por defecto de 802.11** (medido con sniffer) → NO evita la contención: el propio tráfico de control sufre colisiones y backoff.
- (ALTA, WONS 2025) **Retransmisiones automáticas, hasta 31 observadas** (default ~10), slots ~481 µs c/u → delay medio puede saltar a 3461 µs (σ 2079) y hasta **25628 µs** en enlaces degradados.
- (ALTA, WONS 2025) PDR cae de forma no monótona con la distancia (99.97% <55 m; 0% a 59 m por nulos de Fresnel). Calidad de enlace **estocástica**.

**Por qué NO sirve para slots de µs:** (1) CSMA/CA subyacente → variabilidad de cientos de µs no controlable; (2) retransmisiones MAC opacas (hasta 31) → delays multimodales de miles de µs; (3) control y datos comparten radio y canal → no es out-of-band real.

**Recomendación:** ESP-NOW es viable como **señalización/grant para round-robin/polling a escala de ms** (mitiga hidden-node a nivel de coordinación lógica), **no** garantiza exclusividad temporal de µs. El anti-hidden-node lo da la **lógica de turnos** (solo el nodo con grant transmite datos), no la temporización fina de ESP-NOW.

## 6. APuP (contexto de capa de peering — NO es coordinación de medio)
- (MEDIA, Freifunk blog 2024-08-24) APuP = sucesor simplificado de ad-hoc/WDS/802.11s/EasyMesh. Cada nodo es AP y se comunica con otros APs vía **4-address mode**; crea una interfaz por AP visible. Implementado como **patch de hostapd ya en OpenWrt** (`780-Implement-APuP...patch`), sin cambios de kernel.
- (MEDIA) **APuP NO incluye coordinación de medio, anti-colisión ni anti-loop** (el blog lo advierte explícitamente); requiere batman-adv/OLSR/Yggdrasil por encima. Conexiones AP-AP **sin cifrado**.
- **No se halló** uso de la asociación/autenticación 802.11 como scheduler de slots: la asociación controla pertenencia/enlace (plano 3), no acceso temporal al medio (plano 2).

## Fuentes
ESP-IDF ESP-NOW (oficial) · ESP-FAQ (oficial) · Becker et al. "ESP-NOW Performance in Outdoor Environments", IEEE/IFIP WONS 2025 (paper) · micropython #11757 · arXiv 2507.16594 · Sync-ESP-NOW / FTSP (ResearchGate 2023) · Freifunk "A new way to mesh — APuP" (2024-08-24) · OpenWrt git APuP patch.
