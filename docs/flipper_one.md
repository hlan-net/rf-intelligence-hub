# Flipper One ja suhde RF Intelligence Hub -projektiin

Tämä dokumentti kuvaa Flipper Devices -yrityksen kehittämää Flipper One -laitetta ja vertaa sitä tämän projektin (**RF Intelligence Hub**) tavoitteisiin, laitteistoon ja filosofiaan.

---

## 1. Mikä on Flipper One?

[Flipper One](https://docs.flipper.net/one) on taskukokoinen, avoimen lähdekoodin Linux-tietokone (niin sanottu "cyberdeck"), jota kehittää Flipper Zero -laitteesta tunnettu Flipper Devices. Se on suunniteltu erityisesti kyberturvallisuuden ammattilaisille, eettisille hakkereille ja tietoliikennetestaajille.

Toisin kuin Flipper Zero, joka keskittyy fyysisen maailman offline-laitteisiin ja alemman tason protokolliin (kuten NFC, RFID, sub-GHz ja infrapuna), Flipper One on suunniteltu IP-pohjaisten verkkojen ja monimutkaisempien järjestelmien analysointiin.

### Flipper One:n keskeiset ominaisuudet:
- **Suoritinarkkitehtuuri:** Perustuu Rockchip RK3576 -järjestelmäpiiriin (SoC), joka ajaa täysiveristä Linux-käyttöjärjestelmää (mainline kernel).
- **Verkkoyhteydet:** Wi-Fi 6E, dual-gigabit Ethernet ja tuki 5G-laajennusmoduuleille.
- **Laajennettavuus:** M.2-laajennusportti (tukee PCI Express, USB 3.0 ja SATA -väyliä) lisälaitteita ja omia moduuleita varten.
- **Käyttötarkoitus:** IP-verkon analyysi, langattoman verkon penetraatiotestaustoiminnot ja SDR (Software Defined Radio) -signaalien käsittely.

---

## 2. Vertailu: Flipper One vs. RF Intelligence Hub

Vaikka molemmat projektit jakavat Flipper Zero -henkisen taskukokoisen muototekijän ja avoimuuden hengen, ne on suunniteltu eri käyttötarkoituksiin ja täysin erilaisilla suunnittelufilosofioilla.

| Ominaisuus | Flipper One | RF Intelligence Hub |
| :--- | :--- | :--- |
| **Pääasiallinen radiotoiminta** | Aktiivinen ja passiivinen analyysi, SDR-signaalit, WiFi 6E, verkkopenetraatio. | **Vain passiivinen vastaanotto (Passive RX)**. BLE, Zigbee/802.15.4 ja WiFi-skannaus/lokitus. Ei aktiivista lähetystä ([device.md:L198](file:///home/larry/Projects/rf-intelligence-hub/spec/device.md#L198)). |
| **Tekoäly (AI)** | Yleiskäyttöinen Linux-alusta; ei valmista tekoäly- tai LLM-integraatiota. | **Sisäänrakennettu hybridi-tekoäly:** Paikallinen LLM (Gemma 3 1B) Hailo-10H-kiihdyttimellä ja Gemini API -pilvilinkki ([requirements.md:L34-56](file:///home/larry/Projects/rf-intelligence-hub/docs/requirements.md#L34-L56)). |
| **Pilvi-integraatio** | Ei erityistä pilviarkkitehtuuria. | **Hajautettu, käyttäjän hallitsema pilvi:** Laite tallentaa tiedot suoraan käyttäjän omaan Firebase/GCP-projektiin ilman keskitettyjä taustapalveluita ([requirements.md:L101-107](file:///home/larry/Projects/rf-intelligence-hub/docs/requirements.md#L101-L107)). |
| **Virransyöttö ja kenttäkäyttö** | Suorituskyky ja laajennettavuus edellä. | **Kenttäkelpoisuus edellä (Suomen talvi -25°C):** Modulaarinen akkukelkka, joka tukee joko 18650 Li-ion -akkua tai AA-litiumparistoja ([device.md:L108-122](file:///home/larry/Projects/rf-intelligence-hub/spec/device.md#L108-L122)). |
| **Jakelumalli** | Kaupallinen, valmiiksi koottu ja testattu kuluttajatuote. | **Täysin avoin ja itse koottava (DIY):** Vain piirustukset ja lähdekoodi julkaistaan; rakentaja on itse laitteen valmistaja ([README.md:L48-53](file:///home/larry/Projects/rf-intelligence-hub/README.md#L48-L53)). |

---

## 3. Miksi molemmille laitteille on paikkansa?

- **Flipper One** on loistava valinta, kun tarvitaan tehokasta, aktiivista verkkopenetraation työkalua ja monipuolista SDR-laitetta, jossa on valmiit rajapinnat lisäkorteille (kuten 5G). Se toimii kuten perinteinen kannettava tietokone tai "cyberdeck" pienoiskoossa.
- **RF Intelligence Hub** on erikoistunut passiiviseen tiedonkeruuseen, pitkään akunkestoon vaikeissa olosuhteissa ja tekoälyavusteiseen havaintojen analysointiin (kuten BLE- ja Zigbee-laitteiden passiivinen seuranta ja poikkeamien havaitseminen). Se ei yritä olla aktiivinen hyökkäystyökalu, vaan passiivinen RF-sensoriasema.

---

## Katso myös
- [Laitespesifikaatio (device.md)](file:///home/larry/Projects/rf-intelligence-hub/spec/device.md)
- [Projektin vaatimukset (requirements.md)](file:///home/larry/Projects/rf-intelligence-hub/docs/requirements.md)
- [Suunnitteluperiaatteet (design_principles.md)](file:///home/larry/Projects/rf-intelligence-hub/spec/design_principles.md)
