# Research — La narrativa regulatoria del "cierre del firmware WiFi" (hallazgo #3, profundizado)

> Evidencia para no escribir algo falso en el informe. Investigación 2026-06-19. Fuente/fecha/confianza.

## Tesis confirmada
La versión popular —"la FCC obligó a cerrar el firmware en 2016"— es **imprecisa**. La regla exigió
**proteger los parámetros de RF** (potencia, frecuencia, DFS) contra reprogramación no autorizada, **NO**
prohibir el firmware open source. La propia FCC lo aclaró. El bloqueo total que hicieron algunos
fabricantes (TP-Link) fue **decisión comercial** (vía más barata de cumplir), no mandato legal.

## 1. La regla FCC 2014-2016 (U-NII / 5 GHz)
- Base: *First Report and Order* **FCC 14-30** (ET Docket 13-49), adoptado 2014-03-31, vigente 2014-06-02 →
  **47 CFR §15.407(i)** (ALTA, law.cornell.edu). Texto: los fabricantes deben implementar seguridad para que
  *terceros no puedan reprogramar el equipo para operar fuera de los parámetros certificados* (frecuencia,
  potencia, máscara, **DFS**).
- Guía: **KDB 594280 D02** v01r03 (2015-11-12). Texto literal (ALTA): *"The rule is not intended to prevent
  or inhibit modification of any other software or firmware in the device, such as software modifications to
  improve performance, configure RF networks or improve cybersecurity…"*
- La confusión nació de la versión original (~mar-2015) de la guía que **nombraba a DD-WRT** ("how the device
  is protected from 'flashing'… such as DD-WRT"). Esa frase fue **eliminada** en nov-2015 (EFF, ALTA).
- **Aclaración explícita FCC:** Julius Knapp (jefe OET), blog *"Clearing the Air on Wi-Fi Software Updates"*,
  2015-11-12: no prohibían el firmware open source; el foco era solo modificaciones que sacaran al equipo de
  compliance (ALTA sustancia; MEDIA wording — la página fcc.gov se citó vía EFF/InformationWeek).

## 2. El caso TP-Link (consent decree, 2016-08-01, DA 16-850)
- Infracción: un ajuste de **country code** accesible al usuario habilitaba potencia fuera de los parámetros
  aprobados en **canales restringidos** (§15.15(b)). Multa **USD 200.000** + plan de compliance (ALTA).
- ⚠️ **Corrección de banda:** la versión popular dice "2.4 GHz"; los documentos primarios apuntan a
  **5 GHz / U-NII**. Citar "canales restringidos (U-NII / 5 GHz)", no "2.4 GHz" (MEDIA-ALTA).
- **NO obligó a bloquear firmware de terceros — lo CONTRARIO:** ¶15(a)(iv) comprometió a TP-Link a
  *investigar* soluciones que **permitieran firmware de terceros/open-source** preservando los parámetros de
  RF, y a *cooperar con desarrolladores de software libre y fabricantes de chipsets* (ALTA).
- ⚠️ La obligación vinculante es "investigar" + "cooperar", NO una garantía de soportar OSS. Titulares como
  "FCC forces TP-Link to support open source" **sobredimensionan** (LWN lo marcó). Es un settlement con una
  parte, **no rulemaking ni jurisprudencia** — peso señalizador, no precedente vinculante.

## 3. Save WiFi y la respuesta comunitaria
- Disparador formal: NPRM **ET Docket 15-170** (Federal Register 2015-08-06). Coalición "Save WiFi"
  (LibrePlanet/FSF, libreCMC, OpenWrt, EFF, SFLC, prpl, ARRL). Carta abierta de **Dave Täht + Vint Cerf**,
  ~260 firmantes (Schneier, Gettys, Reed, Torvalds, Vixie…). (ALTA)
- ⚠️ **Anacronismo a evitar:** **KRACK es de oct-2017**, posterior a la campaña de 2015. Sirve de ilustración
  *posterior*, pero es **falso** decir que la campaña de 2015 lo invocó.
- Realidad: TP-Link sí bloqueó por completo para cumplir (revelado en Battlemesh, feb-2016), luego revertido
  por el settlement de ago-2016. La FCC retrocedió (blog Knapp + guía revisada). No hubo prohibición de OSS.
- ⚠️ **No verificado / falso:** no existió demanda judicial de SFC/libreCMC contra estas reglas (eran
  advocacy, no litigantes). No atribuir cita específica a Bunnie Huang sobre este tema (no verificada).

## 4. RF-lock vs firmware-lock: el lockdown total fue COMERCIAL
- Portavoz FCC (Charles Meisch, vía prpl 2015-09-25): los fabricantes *pueden* prohibir mods de software,
  pero si tienen otra solución que logre el mismo fin (impedir mods de RF fuera de compliance) es aceptable.
  El bloqueo completo es "the easiest way to comply" (ALTA).
- **Contraejemplos (firmware libre + compliance FCC coexisten):** Linksys WRT1900AC/WRT3200ACM (parámetros
  regulatorios aislados en driver `mwlwifi`); ath9k/ath10k (regdomain en EEPROM); **OpenWrt One (2024)**, de
  SFC+OpenWrt, **certificado FCC y plenamente libre/reflasheable** ("never be locked down"). ⚠️ Matiz: el
  OpenWrt One aún tiene 1-2 blobs de radio; "totalmente abierto" aplica al SO/stack, no a la radio.

## 5. El "router ban" 2024-2026 — TEMA DISTINTO (no confundir)
- Es **seguridad nacional / origen chino**, NO firmware: campañas Volt Typhoon / Salt Typhoon / botnet
  Quad7 / implante Camaro Dragon; ~65% del mercado SOHO de EE.UU. (ALTA).
- Trayectoria: prohibición específica a TP-Link **propuesta** (oct-nov 2025) → **archivada** por la Casa
  Blanca (feb-2026, distensión comercial) → la FCC añade **todos los routers de consumo fabricados fuera de
  EE.UU.** a la Covered List (2026-03-23, orden final; bloquea nueva autorización de equipo, no retira los ya
  en uso). Barre a casi todas las marcas, no solo TP-Link.
- Clave: incluso esa orden de 2026 **"does not restrict software changes made by owners of routers"** (SFC,
  2026-04-02); el OpenWrt One sigue disponible en EE.UU.

## 6. Posturas FSF/EFF/SFC
- **FSF** (libertad): criterio RYF exige software libre salvo "procesador secundario" (firmware en ROM
  exento; blob cargado en RAM debe ser libre) → por eso favorece **ath9k** (firmware cargable *libre*).
- **EFF** (seguridad/innovación): el firmware abierto recibe parches que los fabricantes no entregan; habilita
  mesh y combate bufferbloat.
- **SFC** (right to repair): hogar fiscal de OpenWrt; co-lanzó el OpenWrt One; renovó exenciones DMCA §1201
  para firmware alternativo (2021). La más activa y actual de las tres.

## Cómo contarlo en el informe (1 párrafo, preciso)
En 2014-2016 la FCC (FCC 14-30 / 47 CFR §15.407(i) / KDB 594280) exigió asegurar **solo los parámetros de
RF** (potencia, frecuencia, DFS) para que terceros no reprogramaran el equipo fuera de su certificación —
**no prohibió el firmware open source**, y lo aclaró por escrito en nov-2015 (blog de Knapp), retirando la
mención a DD-WRT. El "cierre" que sí ocurrió en algunos productos (TP-Link, mar-2016) fue **decisión
comercial** (bloquear toda la imagen era lo más barato), no mandato legal, como prueban Linksys, ath9k/ath10k
y el OpenWrt One de 2024 (certificado FCC y libre). Evitar tres errores: (1) "la regla prohibió el OSS"
(falso); (2) "el settlement de TP-Link obligó a bloquear firmware" (lo *fomentó*; y "investigar" no es
garantía); (3) mezclar esto con el "router ban" 2024-2026 (seguridad nacional/origen chino, donde ni la orden
de mar-2026 restringe los cambios de software del dueño). Si se cita KRACK, aclarar que es de 2017.
