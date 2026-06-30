# Precedentes para fundamentar el SCOPE y la ESTRATEGIA del proyecto

**Proyecto:** implementación de referencia abierta de un algoritmo de coordinación de acceso al medio (TDMA / grafo-de-conflicto), sobre `ath9k`, con el fin último de empujar el desarrollo de hardware WiFi de nueva generación, alto rendimiento y base abierta.

**Pregunta central que responde este documento:** ¿el scope es meramente académico / limitado a `ath9k`, o existe una ambición de impacto real y un camino citable para justificar el esfuerzo?

**Respuesta corta (tesis del informe):** El patrón "implementación de referencia abierta → cataliza hardware abierto" es un precedente **documentado y repetido** en redes (O-RAN, openwifi) y en silicio (RISC-V, OpenMPW). El scope del proyecto no es "académico" en el sentido peyorativo: es una **implementación de referencia de bajo TRL cuyo valor es de-riesgar y probar la necesidad** de la siguiente etapa (hardware). La honestidad exige reconocer que (a) `ath9k` es plataforma de prueba (802.11n), no producto, y (b) el silicio RF abierto todavía no existe. Pero esa misma asimetría —"probamos la necesidad sobre la única MAC abierta disponible; productizar exige nuevo hardware abierto"— **es** el argumento, y tiene precedentes de financiamiento e instrumentos formales que lo respaldan.

> **Convención:** cada hallazgo lleva **[HECHO]** (fuente primaria/citable) o **[INFERENCIA]** (síntesis interpretativa, defendible pero no citada textualmente). Confianza: alta / media / baja. URLs accedidas 2026-06-19/20.

---

## 1. El patrón "software/algoritmo de referencia abierto → justifica/cataliza hardware abierto"

Este es el corazón del argumento. Hay tres precedentes fuertes.

### 1.a — srsRAN / OpenAirInterface sobre SDR → ecosistema O-RAN

- **[HECHO · alta]** **srsRAN** (originalmente srsLTE) es una suite 4G/5G C++ open-source (AGPLv3) diseñada para correr sobre cómputo de propósito general + radio SDR de estante (p.ej. USRP B210 de NI/Ettus). Software Radio Systems (SRS, Irlanda) publicó la primera aplicación full-stack LTE UE open-source en noviembre de 2015; el proyecto se renombró srsLTE→srsRAN con la release 21.04 al pasar de 4G a 5G NR.
  - https://www.srslte.com/srslte-srsran · https://github.com/srsran/srsRAN_Project · https://kb.ettus.com/5G_srsRAN_End-to-End_Reference_Architecture_with_USRP
- **[HECHO · alta]** La **OpenAirInterface Software Alliance (OSA)** fue fundada por EURECOM en 2014 como fundación francesa sin fines de lucro para desarrollar la plataforma open-source RAN/Core OAI.
  - https://openairinterface.org/about-oai/
- **[HECHO · alta]** La **O-RAN Alliance** se fundó en **febrero de 2018** por cinco operadoras (AT&T, China Mobile, Deutsche Telekom, NTT DOCOMO, Orange), fusionando la xRAN Foundation (EE.UU.) y la C-RAN Alliance (China); incorporada como e.V. alemana en agosto de 2018. Publica **diseños de referencia de hardware "white-box"** (p.ej. *Hardware Reference Design Specification for Indoor Picocell FR1, Split Option 7-2*, cubriendo O-CU/O-DU/O-RU), atando explícitamente interfaces de software abiertas a hardware de estación base especificado abiertamente.
  - https://www.o-ran.org/about · https://www.o-ran.org/blog/20-new-o-ran-specifications-have-been-released-since-june-2020
- **[HECHO · alta]** La **O-RAN Software Community (OSC)** se creó en 2018 como esfuerzo conjunto O-RAN Alliance + Linux Foundation para producir software de **diseño de referencia** alineado con las especificaciones.
  - https://o-ran-sc.org/ · https://www.o-ran.org/software
- **[HECHO · alta]** O-RAN Alliance y OpenAirInterface **expandieron formalmente su cooperación** (anunciado 14-dic-2023). El co-chair del TSC de O-RAN declaró que "ofrecer software open-source ayuda a la industria en la comercialización de productos O-RAN" — documentando que el software RAN abierto se usa para impulsar el ecosistema de productos/hardware O-RAN.
  - https://www.o-ran.org/blog/o-ran-alliance-and-openairinterface-software-alliance-expand-cooperation-on-developing-open-software-for-the-ran
- **[INFERENCIA · media (ecosistema) / baja (atribución causal directa)]** Existe silicio "merchant"/abierto para Open RAN (chips mostrados en MWC-2022, aceleradores FPGA tipo Xilinx T1 para fronthaul/LDPC). **No hallé** fuente primaria que afirme que srsRAN/OAI *causaron* el silicio abierto. La cadena "software RAN abierto → hardware white-box → discusión de silicio abierto" debe presentarse como **inferencia** apoyada en co-desarrollo del ecosistema, no como hecho citado.
  - https://www.counterpointresearch.com/insights/open-ran-merchant-silicon/

> **Cómo lo usamos en el argumento:** O-RAN es el caso canónico moderno: un cuerpo de software de referencia open (OAI/srsRAN sobre SDR) coexiste con —y alimenta— especificaciones de **hardware de referencia abierto** y un mercado de silicio. Es la analogía directa para "implementación de referencia de MAC abierta → empuje a hardware WiFi abierto", **siempre que** seamos precisos: el vínculo software↔hardware-de-referencia está documentado; el salto a "silicio abierto" es tendencia adyacente, no causalidad probada.

### 1.b — openwifi nació del proyecto EU H2020 ORCA (la narrativa de origen más fuerte)

- **[HECHO · alta]** **openwifi** es un diseño full-stack IEEE 802.11 (baseband/FPGA sobre SDR) compatible con `mac80211` de Linux, autoría Xianjun Jiao, Michael Mehari, Wei Liu (UGent/IMEC, 2019), AGPLv3.
  - https://github.com/open-sdr/openwifi
- **[HECHO · alta]** El desarrollo interno de openwifi se remonta a **2017 dentro del proyecto EU H2020 ORCA** (grant agreement **732174**); el código se publicó abiertamente en GitHub a fines de 2019 como el "1er diseño de chip/FPGA Wi-Fi soft-mac compatible con Linux".
  - https://wiki.f-si.org/index.php?title=An_opensource_Wi-Fi_chip%2C_What%2C_Why_and_How%3F · https://www.orca-project.eu/ieee802-11-full-stack-based-on-sdr-openwifi/index.html
- **[HECHO · alta (autoría/motivación) · INFERENCIA en el fraseo exacto]** openwifi se creó en **IMEC** para dar a investigadores acceso de bajo nivel a MAC/PHY que los chips Wi-Fi comerciales niegan. Las slides del autor (F-Si) enumeran como "pain points" de los chips comerciales: fin de vida (EOL), dependencia de firmware/driver del fabricante, y chips "cada vez más cerrados". El fraseo "el firmware cerrado niega acceso a la MAC baja" es inferencia consistente con esos pain points (no es cita textual).
  - https://wiki.f-si.org/images/a/ad/Openwifi.pdf.pdf
- **[HECHO · alta]** openwifi corre el **baseband digital abierto sobre FPGA Xilinx** pero **depende de un transceptor RF comercial propietario** (Analog Devices AD9361) para el front-end analógico/RF.
  - https://github.com/open-sdr/openwifi · https://www.rtl-sdr.com/openwifi-open-source-fpga-and-sdr-based-wifi-implementation/

> **Cómo lo usamos:** openwifi es el **precedente más cercano y más fuerte**. Es exactamente nuestra tesis ya ejecutada: financiado por la UE (ORCA, 2017) **precisamente para dar el acceso de bajo nivel que el firmware cerrado niega**, produjo el primer Wi-Fi soft-MAC abierto sobre FPGA. Confirma que (i) hay financiamiento público para esto, (ii) el camino real es **baseband abierto sobre FPGA + RF propietario** (no ASIC RF abierto de golpe), y (iii) nuestro proyecto de coordinación de medio abierta es un complemento natural a esta línea (un algoritmo MAC de referencia que openwifi —o su sucesor— podría hostear).

### 1.c — Otros "software-defined-everything" que abrieron hardware

- **[HECHO · media-alta]** **Project IceStorm / Yosys / nextpnr**: una cadena de herramientas *software* open (ingeniería inversa del bitstream de las FPGA Lattice iCE40) desbloqueó hardware FPGA antes cerrado para desarrollo abierto — caso documentado de software de referencia abriendo flujos de silicio propietario.
  - https://hackaday.com/2020/03/06/mithro-runs-down-open-source-fpga-toolchains/ · https://www.yosyshq.com/open-source
- **[HECHO · alta (existencia) · INFERENCIA en fraseo causal]** El ISA abierto **RISC-V** catalizó una ola de hardware abierto (cores + EDA/FPGA open: Yosys, SymbiFlow, nextpnr), muchos consolidados bajo **CHIPS Alliance**, que mantiene herramientas open de diseño ASIC y FPGA.
  - https://www.chipsalliance.org/

---

## 2. Open silicon / RFSoC / RISC-V: ¿qué tan realista es "silicio WiFi abierto"? (la parte honesta)

Esta sección debe **no sobrevender**. La conclusión: el silicio digital abierto es real pero comercialmente frágil; RISC-V en networking es real y ya embarca; **el silicio RF abierto esencialmente todavía no existe**.

### 2.a — Fabricación de silicio abierto (digital)

- **[HECHO · alta]** El programa gratuito **OpenMPW** de Google (lanzado nov-2020) fabricó **360 diseños** open-source en 8 shuttles SKY130 + 1 corrida GF180MCU, de 600+ propuestas de 19 países, antes de transferirse a la Linux Foundation en **nov-2023**.
  - https://opensource.googleblog.com/2023/11/open-source-pdks-joining-linux-foundation-chips-alliance.html
- **[HECHO · alta]** El shuttle 100% gratis terminó en nov-2023; los PDK abiertos (SKY130, GF180MCU) pasaron a CHIPS Alliance y la fabricación gratuita se reemplazó por vías comerciales de bajo costo (Efabless ChipIgnite, Tiny Tapeout).
  - (misma URL)
- **[HECHO · alta]** **SKY130** es genuinamente abierto: cualquiera descarga manuales y modelos **sin NDA**, removiendo la barrera de confidencialidad de foundry que históricamente bloqueaba el diseño abierto.
  - https://www.skywatertechnology.com/sky130-open-source-pdk/
- **[HECHO · alta]** **Tiny Tapeout** (fundado por Matt Venn, 2022, derivado de su curso Zero-to-ASIC) baja la barrera al ASIC partiendo un chip en muchos *tiles* y compartiendo el costo de oblea: silicio real por unos cientos de dólares en vez de decenas de miles.
  - https://www.siliconimist.com/p/tiny-tapeout-matt-venn
- **[HECHO · alta] (advertencia de fragilidad)** El ecosistema **no está resuelto**: **Efabless** (que operaba ChipIgnite, del que dependía Tiny Tapeout) **cerró en marzo de 2025** por falta de financiamiento, dejando varados TT08 (~135 proyectos) y TT09 (~370, en fabricación). Recuperación parcial a fines de 2025 alrededor de tres PDK abiertos (SKY130 vía Cadence/ChipFoundry, IHP SG13G2, GF180MCU vía wafer.space); ChipFoundry adquirió la IP de Efabless en sept-2025.
  - https://www.tomshardware.com/tech-industry/semiconductors/efabless-shuts-down-fate-of-tiny-tapeout-chip-production-projects-unclear · https://www.zerotoasiccourse.com/post/excited_by_silicon/

### 2.b — RISC-V en networking

- **[HECHO · alta]** RISC-V **ya embarca en silicio comercial de networking de datacenter**: el DPU NVIDIA **BlueField-3** contiene un "DataPath Accelerator" de **16 cores RISC-V** para offload programable de paquetes/control de congestión.
  - https://arxiv.org/abs/2504.17307
- **[HECHO · alta]** **OpenTitan** es un *root-of-trust* de silicio en producción sobre el core RISC-V abierto **Ibex**, para motherboards de servidor y tarjetas de red — chip RISC-V abierto real, aunque es elemento de seguridad/cripto, **no** un datapath de procesamiento de paquetes.
  - https://lowrisc.org/news/announcing-opentitan-the-first-transparent-silicon-root-of-trust/

### 2.c — Open RF / open WiFi: la "parte difícil"

- **[HECHO · alta]** El **AMD/Xilinx Zynq UltraScale+ RFSoC** integra conversores RF de gigasamples (hasta 8 ADC a 1–5 GSa/s, 8 DAC, ~6 GHz BW analógico) con FPGA y cores ARM en un SoC — habilita SDR de un solo chip, pero es **parte comercial totalmente propietaria, no silicio abierto**.
  - https://www.xilinx.com/publications/product-briefs/rfsoc-product-brief.pdf
- **[HECHO · alta]** RFSoC sin embargo **habilita software/gateware SDR open** (p.ej. el stack libre RFSoC-PYNQ): **la apertura ocurre en la capa de gateware/software sobre hardware RF cerrado**.
  - https://www.rfsoc-pynq.io/
- **[HECHO · alta]** **"Silicio WiFi abierto" no existe hoy.** El proyecto líder, openwifi, es un **baseband digital 802.11a/g/n abierto sobre FPGA** y aun depende de un **transceptor RF comercial propietario (AD9361)** para el front-end analógico.
  - https://github.com/open-sdr/openwifi
- **[HECHO · alta (síntesis bien apoyada)]** El camino realista de RF abierto es **escalonado**: baseband digital abierto en FPGA (prototipado) + transceptor RF cerrado — **no** un ASIC RF abierto monolítico. openwifi ejemplifica exactamente esa etapa intermedia FPGA→(eventual ASIC).

### 2.d — La "brecha analógica" (por qué digital abierto ≠ RF abierto)

- **[HECHO · alta]** El silicio abierto es mucho más maduro en digital que en analógico/RF: el layout digital está totalmente automatizado, pero el analógico sigue siendo **manual y sensible a la geometría** (acoplamiento parásito, simetría), con ciclos ~2–3× más lentos aun con herramientas comerciales.
  - https://www.eetasia.com/digital-layout-is-fully-automated-why-isnt-analog-layout/
- **[HECHO · alta]** Las herramientas EDA abiertas tienen "factibilidad especialmente limitada para layout analógico"; los PDK abiertos traen formatos digital-friendly pero los *decks* de DRC/extracción carecen de formatos abiertos estándar, y SKY130 es un nodo de propósito general no optimizado para RF.
  - https://arxiv.org/pdf/2512.06122
- **[HECHO · alta]** Existe ya un PDK abierto **capaz de RF**: **IHP SG13G2**, un proceso 130nm SiGe BiCMOS (HBT hasta ~350 GHz fT) bajo Apache 2.0 — **pero IHP lo etiqueta explícitamente como preview experimental/alfa "no destinado a producción"**, subrayando que el silicio RF abierto está aún en etapa muy temprana.
  - https://github.com/IHP-GmbH/IHP-Open-PDK

> **Cómo lo usamos (honestidad calibrada):** No podemos afirmar "haremos silicio WiFi abierto". Sí podemos afirmar, con respaldo: (1) el **digital abierto es real** pero frágil; (2) **RISC-V ya está en networking**; (3) el camino RF honesto es **baseband abierto sobre FPGA + RF propietario**, exactamente la etapa que openwifi ocupa, y la que un algoritmo MAC de referencia abierto **prepara**. La ambición legítima del proyecto no es "fabricar el ASIC" sino **producir la pieza de software/algoritmo de referencia que vuelve discutible y financiable** el hardware abierto de próxima generación.

---

## 3. Redes comunitarias empujando hardware, y cómo se financia/justifica

- **[HECHO · alta]** **LibreRouter** es hardware abierto multi-radio para redes comunitarias, desarrollado por **AlterMundi** (asociación civil argentina) con el Internet Society Community Networks SIG. **Precedente propio** de hardware libre comunitario.
  - https://blog.apnic.net/2018/12/18/librerouter-powering-community-networks-with-free-and-open-hardware/
- **[HECHO · alta]** LibreRouter fue seleccionado/ganó un premio **FRIDA en 2016** (implementado por AlterMundi) para "diseñar y producir un router inalámbrico multi-radio de alto rendimiento para necesidades de Redes Comunitarias". **FRIDA** = Fondo Regional para la Innovación Digital en América Latina y el Caribe, administrado por **LACNIC** (Seed Alliance).
  - https://programafrida.net/en/archivos/project/router-libre · https://seedalliance.net/wp-content/uploads/2017/12/LibreRouter-CompiledReport.pdf
- **[HECHO · media]** LibreRouter también recibió financiamiento **Internet Society "Beyond the Net"** (incluida una "LibreRouter Phase 2", 2018); los grants Beyond the Net llegan hasta ~US$30.000. *(Montos exactos US$40k FRIDA / US$30k BtN provienen de snippets, no de página primaria fetchada — verificar antes de citar cifras.)*
  - https://www.internetsociety.org/beyond-the-net/grants/2018/librerouter-phase-2/
- **[INFERENCIA · media (hallazgo negativo)]** **No** se halló evidencia de financiamiento de **Cisco** a LibreRouter. Los financiadores documentados son FRIDA/LACNIC e Internet Society. **No usar el ángulo "Cisco" sin verificación.**
- **[HECHO · alta]** **LibreMesh** es el framework modular OpenWrt para nodos mesh inalámbricos (AGPL); el LibreRouter sale de fábrica con LibreMesh. Surgió fusionando proyectos previos de firmware comunitario — AlterMesh (AlterMundi), qMp (Catalunya), eigenNet (Italia). *(El detalle de la fusión triple es confianza media.)*
  - https://libremesh.org/
- **[HECHO · alta]** **guifi.net** es una red comunitaria de Cataluña (origen ~2004, Gurb, por Ramon Roca), hoy **>37.000 nodos activos** y 70.000+ km de enlaces, gobernada como **procomún** bajo la licencia **XOLN** (Llicència de Comuns per a la Xarxa Oberta, Lliure i Neutral): los operadores prestan servicios sobre el procomún en vez de poseer la red.
  - https://civilsociety.dev/articles/guifi-net/ · http://guifi.net/en/CommonsXOLN
- **[HECHO · alta]** La **NLnet Foundation**, vía el **NGI Zero Commons Fund**, otorga grants de I+D de **€5.000–€50.000** para FLOSS **y hardware** "de libre silicio a middleware, de infraestructura P2P a aplicaciones". Distribuirá **€21,6 M hasta 2027**, financiado por el programa **Next Generation Internet** de la Comisión Europea (DG CNECT) + SERI (Suiza). Financia rutinariamente proyectos de networking, protocolos y **hardware abierto de confianza**.
  - https://nlnet.nl/commonsfund/ · https://ngi.eu/ngi-projects/ngi-zero-commons-fund/
- **[HECHO · alta]** **ARDC** (Amateur Radio Digital Communications) es una fundación privada de California financiada por la venta (2019) de un cuarto del bloque IPv4 44/8; hace grants en Radioafición, Educación, Becas e **I+D**. Sus áreas prioritarias incluyen explícitamente **hardware abierto y sistemas open-source** (SDR, códecs abiertos), **infraestructura inalámbrica abierta** y redes comunitarias/regionales — haciendo del hardware inalámbrico abierto comunitario una **categoría financiable**.
  - https://www.ardc.net/about/ · https://www.ardc.net/apply/grants/
- **[INFERENCIA · media]** La I+D de hardware comunitario/abierto, sin retorno comercial inmediato, se financia mediante un **ecosistema en capas** de financiadores con misión: fondos de registros regionales (FRIDA/LACNIC), fundaciones de interés público de Internet (ISOC/Beyond the Net), programas públicos UE (NLnet/NGI Zero), y fundaciones de espectro/radioafición (ARDC). Se justifica como **construcción de procomún digital, soberanía tecnológica e infraestructura resiliente**, no como ganancia.

> **Cómo lo usamos:** AlterMundi **ya tiene** el precedente propio (LibreRouter) de llevar una necesidad comunitaria a hardware libre con financiamiento de misión. El presente proyecto es el **eslabón de I+D anterior al hardware**: produce el algoritmo/implementación de referencia que justifica el siguiente LibreRouter de "nueva generación". El modelo de financiamiento (ARDC + análogos NLnet/NGI/FRIDA) está probado para exactamente este tipo de esfuerzo.

---

## 4. Qué es formalmente una "implementación de referencia" y por qué su impacto puede ser desproporcionado

- **[HECHO · alta]** Una **implementación de referencia** implementa todos los requisitos de su especificación, sirviendo como **interpretación definitiva** del comportamiento "correcto" para cualquier otra implementación.
  - https://en.wikipedia.org/wiki/Reference_implementation
- **[HECHO · alta]** **NIST** la define como "la implementación de un estándar que se usa como interpretación definitiva de los requisitos de ese estándar" (NISTIR 8074 Vol. 2).
  - https://csrc.nist.gov/glossary/term/reference_implementation
- **[HECHO · alta]** **No** necesita ser de calidad de producción ni optimizada: su propósito es **corrección/definitividad, no rendimiento** (la implementación de referencia MP3 de Fraunhofer no gana en tests de escucha). Cumple tres funciones: verificar que la spec es **implementable**, validar la suite de conformidad, y ser **patrón de oro** de comparación.
  - https://en.wikipedia.org/wiki/Reference_implementation
- **[HECHO · alta]** **Cultura "running code" del IETF:** el mantra "We reject kings, presidents and voting. We believe in rough consensus and running code" (David Clark, MIT, 1992) formaliza el valor de las **implementaciones funcionando** por encima de la teoría. Codificado en la sección "Implementation Status" de los drafts (**RFC 7942**, que obsoleta RFC 6982).
  - https://www.ietf.org/runningcode/ · https://www.rfc-editor.org/rfc/rfc7942.html
- **[HECHO · alta]** Una implementación de referencia/ampliamente usada **puede volverse el estándar de facto** — p.ej. **CPython** es "la implementación de referencia de Python" y la más usada; las alternativas (PyPy, Jython) apuntan a ser compatibles con ella.
  - https://en.wikipedia.org/wiki/CPython
- **[HECHO · alta] (ejemplos de impacto desproporcionado)**
  - **Berkeley (BSD) sockets**: liberados como implementación de referencia en 4.2BSD (1983, financiado por DARPA), se volvieron la **API TCP/IP de facto** y se absorbieron en POSIX; todo SO moderno la implementa.
    https://en.wikipedia.org/wiki/Berkeley_sockets
  - **WebKit**: nació como fork del pequeño proyecto open KHTML/KJS de KDE; WebKit + su derivado Blink hoy mueven Safari, Chrome/Chromium, Edge, Opera, Brave, Vivaldi (familia WebKit ~50% de share a mediados de abril 2015). Apple liberó Safari/WebKit en 2003; Chrome (2008) se construyó sobre WebKit antes de bifurcarlo en Blink (2013).
    https://en.wikipedia.org/wiki/WebKit
- **[HECHO · media]** En hardware inalámbrico, los **diseños de referencia** de fabricantes y módulos Wi-Fi/Bluetooth pre-certificados **de-riesgan y aceleran** la adopción: eliminan semanas de optimización RF y la necesidad de certificación regulatoria/Bluetooth-SIG separada, bajando time-to-market y costo del OEM.
  - https://blog.nordicsemi.com/getconnected/using-a-bluetooth-le-wireless-module-to-accelerate-time-to-market
- **[INFERENCIA · media]** Una implementación de referencia juega tres roles útiles para un *business/investment case*: (a) **estándar de facto** que otros deben igualar, (b) **benchmark/base de comparación** de conformidad y rendimiento, y (c) **prueba de concepto que de-riesga** una especificación al probar que es implementable antes de que otros comprometan recursos. Las funciones (a)(b) están directamente fundadas; el fraseo (c) "de-riesga inversión futura" es inferencia razonable.

> **Cómo lo usamos:** Esto **redefine el scope** y desactiva la objeción "es solo académico". Una implementación de referencia **no es un producto a medias**: es una categoría con valor propio y reconocido (NIST, IETF "running code"). BSD sockets, WebKit y CPython prueban que una implementación de referencia open puede **convertirse en el estándar de facto y de-riesgar una industria entera**. Nuestro algoritmo de coordinación de medio, como implementación de referencia abierta, aspira a ese rol: ser el **patrón de oro abierto y la prueba de implementabilidad** de la coordinación TDMA/grafo-de-conflicto, que el hardware abierto de próxima generación pueda adoptar y contra el cual medirse.

---

## 5. Framing honesto de TRL y ambición (cómo presentarlo sin sobrevender)

- **[HECHO · alta]** **NASA TRL 2**: "principios básicos estudiados y aplicaciones prácticas pueden plantearse… TRL 2 es muy especulativo, con poca o ninguna prueba experimental de concepto." **TRL 3**: "comienzan investigación y diseño activos… estudios analíticos y de laboratorio." **TRL 4**: "la tecnología de prueba-de-concepto está lista… múltiples componentes probados entre sí." Un estadio de concepto/PoC es un **nivel de madurez legítimo y reconocido**, no una deficiencia.
  - https://www.nasa.gov/directorates/somd/space-communications-navigation-program/technology-readiness-levels/
- **[HECHO · alta]** **Horizonte 2020, Annex G** (texto oficial UE para auto-clasificar): TRL 1 "principios básicos observados"; TRL 2 "concepto tecnológico formulado"; TRL 3 "prueba de concepto experimental"; TRL 4 "tecnología validada en laboratorio".
  - https://ec.europa.eu/research/participants/data/ref/h2020/other/wp/2018-2020/annexes/h2020-wp1820-annex-g-trl_en.pdf
- **[HECHO · alta]** **Frascati Manual (OCDE, 2015):** *investigación básica* = adquirir conocimiento nuevo "sin ninguna aplicación o uso particular a la vista"; *investigación aplicada* = "dirigida primariamente a un objetivo práctico específico"; *desarrollo experimental* = "trabajo sistemático… dirigido a producir nuevos productos/procesos o mejorarlos". Un PoC de MAC-WiFi-abierta de bajo TRL está en la banda **investigación básica/aplicada, no desarrollo experimental**.
  - https://www.oecd.org/content/dam/oecd/en/publications/reports/2015/10/frascati-manual-2015_g1g57dcb/9789264239012-en.pdf
- **[HECHO · media]** El **"valle de la muerte"** es la brecha de financiamiento/riesgo "entre TRL 4 y 7, donde el financiamiento se seca por riesgo percibido", "extremadamente profundo" para deep-tech/hardware. El de-riesgo técnico temprano (validación a nivel componente) es estrategia reconocida para atraer inversión privada: **el trabajo de un PoC creíble es retirar riesgos técnicos específicos, no reclamar madurez de producto.**
  - https://link.springer.com/article/10.1007/s41469-023-00144-y
- **[HECHO · alta]** **`ath9k`** es "un driver inalámbrico completamente FOSS para todos los chipsets Atheros 802.11n PCI/PCIe/AHB" — confirma su carácter **full-open-source** y que es **era 802.11n** (no generación actual).
  - https://wireless.docs.kernel.org/en/latest/en/users/drivers/ath9k.html
- **[HECHO · alta]** `ath9k` usa arquitectura **SoftMAC**: "toda la gestión de tramas 802.11 la hace un módulo de software en el host, no firmware en el hardware", y la NIC "expone muchos registros de control accesibles directamente por el driver del kernel." **Esta es la razón técnica concreta por la que `ath9k` permite experimentación MAC de bajo nivel:** la lógica MAC está en software editable del host, no en un blob cerrado. (Incluso el firmware de las variantes USB AR7010/AR9271 está open-sourced por Qualcomm Atheros.)
  - https://wireless.docs.kernel.org/en/latest/en/users/drivers/ath9k.html · https://github.com/qca/open-ath9k-htc-firmware
- **[HECHO · alta] (precedente directo de TDMA sobre `ath9k`)** Investigación peer-reviewed (**hMAC, TU Berlin, 2016**) se construyó sobre "el popular driver softMAC Linux ATH9K", explotando "la funcionalidad estándar de power-saving 802.11 presente en el driver ATH9K para controlar las colas de software… [para] la asignación de slots de tiempo TDMA". **Precedente documentado** de que `ath9k` se elige específicamente porque permite **TDMA-sobre-WiFi-COTS** vía modificación a nivel driver.
  - https://arxiv.org/pdf/1611.05376 · https://ieeexplore.ieee.org/document/7053808/
- **[INFERENCIA · alta (defendible, no citada textualmente)]** La afirmación "`ath9k` es el *último/único* driver WiFi totalmente abierto con acceso MAC de bajo nivel" es un **juicio comparativo experto**, ampliamente cierto en la práctica, no un hecho citado. Es hardware era 802.11n (~2008–2012): **trabajar sobre él es necesariamente plataforma de investigación/PoC, no un camino a producto.** Esa asimetría —"probamos sobre la única MAC abierta disponible; productizar exige nuevo silicio/hardware abierto"— **es** el argumento de "prueba de necesidad". El informe debe decirlo explícitamente, no insinuar que `ath9k` escalaría a producto.
- **[HECHO · alta] (el instrumento que valida el framing)** El instrumento estrella de bajo TRL de la UE (**EIC Pathfinder**) financia trabajo en **"TRL 1–3, hasta prueba de concepto"** y *exige* combinar esa baja madurez con "una visión convincente de largo plazo para una tecnología radicalmente nueva con potencial transformador… acoplada a un avance concreto, novedoso y ambicioso de ciencia-hacia-tecnología que claramente avance el estado del arte." **Es la plantilla autorizada para combinar honestamente un PoC de bajo TRL con una visión de alto impacto:** visión y breakthrough son separados, ambos requeridos, y ninguno es madurez de producto.
  - https://eic.ec.europa.eu/eic-funding-opportunities/eic-pathfinder/eic-pathfinder-challenges-2026_en

> **Recomendaciones de framing (inferencia, destiladas de lo anterior):**
> 1. **Auto-clasificar explícito:** "esto es un PoC experimental TRL 2–3 (términos Annex G UE)". Sobre-reclamar es bandera roja para financiadores.
> 2. **Separar los dos ejes** (estructura EIC Pathfinder): (a) **visión** de largo plazo de alto impacto [hardware WiFi abierto de nueva generación]; (b) **breakthrough concreto y acotado** que este PoC demuestra [implementación de referencia de coordinación TDMA/grafo-de-conflicto sobre la única MAC abierta]. En párrafos separados, para que la visión nunca se disfrace de resultado.
> 3. **Encuadrar el PoC como retiro-de-riesgo, no producto:** su valor es de-riesgar incógnitas técnicas para volver **invertible** la siguiente etapa (hardware).
> 4. **Asumir la limitación de `ath9k` como fuerza-en-contexto:** es viejo, por eso no puede ser el producto — pero es la **única** plataforma donde la MAC es lo bastante abierta para probar el concepto. Esa es la versión creíble y no-hype del pitch.

---

## Síntesis ejecutiva — respondiendo la pregunta central

| Eje | Hallazgo | Implicación para el scope |
|---|---|---|
| Patrón sw-abierto→hw-abierto | **Documentado y repetido**: O-RAN (OAI/srsRAN→hardware white-box), openwifi (ORCA 2017→primer WiFi soft-MAC abierto), RISC-V/IceStorm | El proyecto sigue un patrón **probado**, no especulativo |
| Silicio WiFi abierto | **No existe aún**; camino real = baseband abierto sobre FPGA + RF propietario; RF analógico abierto en etapa alfa (IHP SG13G2) | La ambición legítima **no** es "fabricar el ASIC" sino **producir la pieza de referencia que lo vuelve discutible/financiable** |
| Redes comunitarias→hardware | **Precedente propio**: LibreRouter (AlterMundi, FRIDA 2016 + ISOC); modelo NLnet/NGI/ARDC probado | AlterMundi ya recorrió necesidad→hardware-libre; esto es el eslabón de I+D previo |
| Implementación de referencia | Categoría **formal con valor propio** (NIST, IETF "running code"); BSD sockets/WebKit/CPython = impacto desproporcionado | Desactiva la objeción "solo académico": es **estándar de facto + benchmark + de-riesgo** |
| Framing TRL | TRL 2–3 es **nivel legítimo**; modelo EIC Pathfinder = visión alta + breakthrough acotado, separados | Presentar como **"prueba de necesidad" de-riesgante**, con `ath9k` como única MAC abierta disponible (fuerza-en-contexto) |

**Veredicto:** El scope **no** es "meramente académico / limitado a `ath9k`". `ath9k` es el **medio** (la única MAC abierta donde el concepto se puede probar), no el **fin**. El fin —empujar hardware WiFi abierto de nueva generación— es una **ambición de impacto real con precedentes citables** (openwifi/ORCA, O-RAN, LibreRouter) y un **camino de financiamiento probado** (ARDC, NLnet/NGI, FRIDA). La honestidad obliga a (a) clasificarlo TRL 2–3, (b) no prometer silicio RF abierto, y (c) nombrar a `ath9k` como plataforma de PoC, no de producto. Hecho con esa calibración, el esfuerzo está plenamente justificado como **implementación de referencia que de-riesga y prueba la necesidad** de la siguiente generación de hardware abierto.

---

### Advertencias de fiabilidad (qué NO sobre-afirmar al citar)
- **Cisco no financió LibreRouter** según lo hallado — eliminar ese ángulo salvo verificación.
- Montos exactos (US$40k FRIDA / US$30k Beyond the Net) provienen de snippets, no de página primaria — verificar antes de citar cifras.
- "srsRAN/OAI *causaron* silicio abierto" = **inferencia**, no hecho citado. Usar "co-desarrollo del ecosistema".
- "`ath9k` es el último/único driver MAC abierto" = juicio comparativo experto, no cita.
- RFSoC propietario y "openwifi = etapa FPGA intermedia" = inferencias bien fundadas, marcadas como tales.
